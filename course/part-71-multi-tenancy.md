# Part 71: Multi-Tenancy Architecture — Row-Level Security, Schema-Per-Tenant & API Key Management

## Steps 1765–1780 | World-Class SaaS Tenancy Patterns

---

## Step 1765: Multi-Tenancy Models — Trade-Offs

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Multi-Tenancy Models                              │
│                                                                     │
│  Shared Database / Shared Schema (Row-Level Security)               │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  DB: SaaSPlatform                                            │  │
│  │  Table: Orders (TenantId column on every row)                │  │
│  │  Table: Products (TenantId column on every row)              │  │
│  │  RLS Policy: CURRENT_SETTING('app.tenant_id') = TenantId    │  │
│  └──────────────────────────────────────────────────────────────┘  │
│  + Cheapest to operate, simplest schema migrations                 │
│  - Cross-tenant data leak risk, noisy neighbor                     │
│                                                                     │
│  Shared Database / Schema-Per-Tenant                                │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  DB: SaaSPlatform                                            │  │
│  │  Schema: tenant_acme   → Orders, Products                    │  │
│  │  Schema: tenant_beta   → Orders, Products                    │  │
│  └──────────────────────────────────────────────────────────────┘  │
│  + Strong isolation, easy per-tenant backup                        │
│  - Schema migrations must run per-tenant                           │
│                                                                     │
│  Database-Per-Tenant                                                │
│  + Maximum isolation, dedicated resources                          │
│  - Most expensive, complex connection management                   │
└─────────────────────────────────────────────────────────────────────┘
```

Choosing the right model depends on compliance, cost, and scale.

---

## Step 1766: ITenantContext — Resolving the Current Tenant

```csharp
// Domain/Tenancy/ITenantContext.cs
public interface ITenantContext
{
    Guid TenantId { get; }
    string TenantSlug { get; }
    TenantPlan Plan { get; }
}

public enum TenantPlan { Free, Pro, Enterprise }

// Infrastructure/Tenancy/TenantContext.cs
public sealed class TenantContext : ITenantContext
{
    public Guid TenantId { get; init; }
    public string TenantSlug { get; init; } = string.Empty;
    public TenantPlan Plan { get; init; }
}

// The mutable holder scoped per request
public sealed class TenantContextAccessor
{
    private static readonly AsyncLocal<TenantContext?> _current = new();

    public TenantContext? Current
    {
        get => _current.Value;
        set => _current.Value = value;
    }
}
```

---

## Step 1767: Tenant Resolution Middleware

```csharp
// Infrastructure/Tenancy/TenantResolutionMiddleware.cs
public sealed class TenantResolutionMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ITenantResolver _resolver;
    private readonly TenantContextAccessor _accessor;
    private readonly ILogger<TenantResolutionMiddleware> _logger;

    public TenantResolutionMiddleware(
        RequestDelegate next,
        ITenantResolver resolver,
        TenantContextAccessor accessor,
        ILogger<TenantResolutionMiddleware> logger)
    {
        _next = next;
        _resolver = resolver;
        _accessor = accessor;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var tenant = await _resolver.ResolveAsync(context);

        if (tenant is null)
        {
            context.Response.StatusCode = StatusCodes.Status401Unauthorized;
            await context.Response.WriteAsJsonAsync(new { error = "Tenant could not be resolved." });
            return;
        }

        _accessor.Current = tenant;

        using var logScope = _logger.BeginScope(new Dictionary<string, object>
        {
            ["TenantId"] = tenant.TenantId,
            ["TenantSlug"] = tenant.TenantSlug
        });

        await _next(context);
    }
}

// Extension
public static class TenantMiddlewareExtensions
{
    public static IApplicationBuilder UseTenantResolution(this IApplicationBuilder app)
        => app.UseMiddleware<TenantResolutionMiddleware>();
}
```

---

## Step 1768: Multiple Tenant Resolution Strategies

```csharp
// Infrastructure/Tenancy/ITenantResolver.cs
public interface ITenantResolver
{
    Task<TenantContext?> ResolveAsync(HttpContext context);
}

// Strategy 1: Subdomain-based (acme.saas.io)
public sealed class SubdomainTenantResolver : ITenantResolver
{
    private readonly ITenantRepository _repo;

    public SubdomainTenantResolver(ITenantRepository repo) => _repo = repo;

    public async Task<TenantContext?> ResolveAsync(HttpContext context)
    {
        var host = context.Request.Host.Host; // acme.saas.io
        var parts = host.Split('.');
        if (parts.Length < 3) return null;

        var slug = parts[0]; // "acme"
        return await _repo.FindBySlugAsync(slug);
    }
}

// Strategy 2: Header-based (X-Tenant-ID)
public sealed class HeaderTenantResolver : ITenantResolver
{
    private readonly ITenantRepository _repo;

    public HeaderTenantResolver(ITenantRepository repo) => _repo = repo;

    public async Task<TenantContext?> ResolveAsync(HttpContext context)
    {
        if (!context.Request.Headers.TryGetValue("X-Tenant-ID", out var tenantIdStr))
            return null;

        if (!Guid.TryParse(tenantIdStr, out var tenantId))
            return null;

        return await _repo.FindByIdAsync(tenantId);
    }
}

// Strategy 3: JWT claims-based
public sealed class JwtClaimsTenantResolver : ITenantResolver
{
    public Task<TenantContext?> ResolveAsync(HttpContext context)
    {
        var user = context.User;
        if (!user.Identity?.IsAuthenticated ?? true) return Task.FromResult<TenantContext?>(null);

        var tenantIdClaim = user.FindFirst("tenant_id")?.Value;
        var tenantSlugClaim = user.FindFirst("tenant_slug")?.Value;
        var planClaim = user.FindFirst("tenant_plan")?.Value;

        if (tenantIdClaim is null || !Guid.TryParse(tenantIdClaim, out var tenantId))
            return Task.FromResult<TenantContext?>(null);

        Enum.TryParse<TenantPlan>(planClaim, out var plan);

        return Task.FromResult<TenantContext?>(new TenantContext
        {
            TenantId = tenantId,
            TenantSlug = tenantSlugClaim ?? string.Empty,
            Plan = plan
        });
    }
}

// Composite: try strategies in order
public sealed class CompositeTenantResolver : ITenantResolver
{
    private readonly IEnumerable<ITenantResolver> _resolvers;

    public CompositeTenantResolver(IEnumerable<ITenantResolver> resolvers)
        => _resolvers = resolvers;

    public async Task<TenantContext?> ResolveAsync(HttpContext context)
    {
        foreach (var resolver in _resolvers)
        {
            var tenant = await resolver.ResolveAsync(context);
            if (tenant is not null) return tenant;
        }
        return null;
    }
}
```

---

## Step 1769: Row-Level Security with EF Core

```csharp
// Infrastructure/Data/MultiTenantDbContext.cs
public abstract class MultiTenantDbContext : DbContext
{
    private readonly ITenantContext _tenantContext;

    protected MultiTenantDbContext(
        DbContextOptions options,
        ITenantContext tenantContext)
        : base(options)
    {
        _tenantContext = tenantContext;
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
        ApplyTenantFilters(modelBuilder);
    }

    private void ApplyTenantFilters(ModelBuilder modelBuilder)
    {
        foreach (var entityType in modelBuilder.Model.GetEntityTypes())
        {
            if (!typeof(ITenantEntity).IsAssignableFrom(entityType.ClrType))
                continue;

            var method = typeof(MultiTenantDbContext)
                .GetMethod(nameof(ApplyTenantFilter), BindingFlags.NonPublic | BindingFlags.Static)!
                .MakeGenericMethod(entityType.ClrType);

            method.Invoke(null, [modelBuilder, _tenantContext]);
        }
    }

    private static void ApplyTenantFilter<TEntity>(
        ModelBuilder modelBuilder,
        ITenantContext tenantContext)
        where TEntity : class, ITenantEntity
    {
        modelBuilder.Entity<TEntity>()
            .HasQueryFilter(e => e.TenantId == tenantContext.TenantId);
    }

    public override Task<int> SaveChangesAsync(CancellationToken ct = default)
    {
        StampTenantId();
        return base.SaveChangesAsync(ct);
    }

    private void StampTenantId()
    {
        foreach (var entry in ChangeTracker.Entries<ITenantEntity>()
            .Where(e => e.State == EntityState.Added))
        {
            entry.Entity.TenantId = _tenantContext.TenantId;
        }
    }
}

// Marker interface
public interface ITenantEntity
{
    Guid TenantId { get; set; }
}

// Example entity
public class Order : ITenantEntity
{
    public Guid Id { get; init; } = Guid.NewGuid();
    public Guid TenantId { get; set; }
    public string CustomerEmail { get; set; } = string.Empty;
    public decimal Total { get; set; }
    public OrderStatus Status { get; set; }
    public DateTime CreatedAt { get; init; } = DateTime.UtcNow;
}
```

---

## Step 1770: PostgreSQL Row-Level Security (Database-Level Enforcement)

```sql
-- migrations/V1__add_rls_policies.sql

-- Enable RLS on every multi-tenant table
ALTER TABLE "Orders" ENABLE ROW LEVEL SECURITY;
ALTER TABLE "Products" ENABLE ROW LEVEL SECURITY;
ALTER TABLE "Invoices" ENABLE ROW LEVEL SECURITY;

-- Policy: only rows matching the current tenant setting
CREATE POLICY tenant_isolation_policy ON "Orders"
    USING ("TenantId" = current_setting('app.tenant_id')::uuid);

CREATE POLICY tenant_isolation_policy ON "Products"
    USING ("TenantId" = current_setting('app.tenant_id')::uuid);

CREATE POLICY tenant_isolation_policy ON "Invoices"
    USING ("TenantId" = current_setting('app.tenant_id')::uuid);

-- Grant usage to the app role (not superuser — superuser bypasses RLS)
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_user;
```

```csharp
// Infrastructure/Data/RlsTenantDbContext.cs
// Interceptor that sets PostgreSQL session variable before each query
public sealed class TenantSettingInterceptor : DbConnectionInterceptor
{
    private readonly TenantContextAccessor _accessor;

    public TenantSettingInterceptor(TenantContextAccessor accessor)
        => _accessor = accessor;

    public override async Task ConnectionOpenedAsync(
        DbConnection connection,
        ConnectionEndEventData eventData,
        CancellationToken ct)
    {
        var tenant = _accessor.Current;
        if (tenant is null) return;

        await using var cmd = connection.CreateCommand();
        cmd.CommandText = $"SELECT set_config('app.tenant_id', '{tenant.TenantId}', false)";
        await cmd.ExecuteNonQueryAsync(ct);
    }
}

// Registration
builder.Services.AddDbContext<AppDbContext>((sp, options) =>
{
    options.UseNpgsql(connectionString);
    options.AddInterceptors(sp.GetRequiredService<TenantSettingInterceptor>());
});

builder.Services.AddScoped<TenantSettingInterceptor>();
```

---

## Step 1771: Schema-Per-Tenant with EF Core

```csharp
// Infrastructure/Data/SchemaPerTenantDbContext.cs
public sealed class AppDbContext : DbContext
{
    private readonly string _schema;

    public AppDbContext(DbContextOptions<AppDbContext> options, ITenantContext tenant)
        : base(options)
    {
        _schema = $"tenant_{tenant.TenantSlug}";
    }

    public DbSet<Order> Orders => Set<Order>();
    public DbSet<Product> Products => Set<Product>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.HasDefaultSchema(_schema);
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
    }
}

// Dynamic connection string per tenant
public sealed class TenantDbContextFactory
{
    private readonly ITenantRepository _tenantRepo;
    private readonly IServiceProvider _sp;

    public TenantDbContextFactory(ITenantRepository tenantRepo, IServiceProvider sp)
    {
        _tenantRepo = tenantRepo;
        _sp = sp;
    }

    public async Task<AppDbContext> CreateForTenantAsync(Guid tenantId)
    {
        var tenant = await _tenantRepo.FindByIdAsync(tenantId)
            ?? throw new TenantNotFoundException(tenantId);

        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseNpgsql(tenant.ConnectionString)
            .Options;

        return new AppDbContext(options, tenant);
    }
}
```

---

## Step 1772: Schema Migration Per Tenant

```csharp
// Infrastructure/Tenancy/TenantMigrationService.cs
public sealed class TenantMigrationService
{
    private readonly ITenantRepository _tenantRepo;
    private readonly ILogger<TenantMigrationService> _logger;

    public TenantMigrationService(
        ITenantRepository tenantRepo,
        ILogger<TenantMigrationService> logger)
    {
        _tenantRepo = tenantRepo;
        _logger = logger;
    }

    public async Task MigrateAllTenantsAsync(CancellationToken ct)
    {
        var tenants = await _tenantRepo.GetAllActiveAsync(ct);

        // Process tenants in parallel with bounded concurrency
        var semaphore = new SemaphoreSlim(5); // max 5 concurrent migrations
        var tasks = tenants.Select(async tenant =>
        {
            await semaphore.WaitAsync(ct);
            try
            {
                await MigrateTenantAsync(tenant, ct);
            }
            finally
            {
                semaphore.Release();
            }
        });

        await Task.WhenAll(tasks);
    }

    private async Task MigrateTenantAsync(TenantContext tenant, CancellationToken ct)
    {
        _logger.LogInformation("Migrating tenant {TenantSlug}", tenant.TenantSlug);

        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseNpgsql(
                /* tenant.ConnectionString */ "",
                npgsql => npgsql.MigrationsHistoryTable(
                    "__EFMigrationsHistory",
                    $"tenant_{tenant.TenantSlug}"))
            .Options;

        await using var db = new AppDbContext(options, tenant);

        // Create schema if not exists
        await db.Database.ExecuteSqlRawAsync(
            $"CREATE SCHEMA IF NOT EXISTS tenant_{tenant.TenantSlug}", ct);

        await db.Database.MigrateAsync(ct);

        _logger.LogInformation("Tenant {TenantSlug} migrated successfully", tenant.TenantSlug);
    }
}

// Hosted service to migrate on startup
public sealed class MigrationHostedService : IHostedService
{
    private readonly IServiceScopeFactory _scopeFactory;

    public MigrationHostedService(IServiceScopeFactory scopeFactory)
        => _scopeFactory = scopeFactory;

    public async Task StartAsync(CancellationToken ct)
    {
        await using var scope = _scopeFactory.CreateAsyncScope();
        var migrationService = scope.ServiceProvider
            .GetRequiredService<TenantMigrationService>();
        await migrationService.MigrateAllTenantsAsync(ct);
    }

    public Task StopAsync(CancellationToken ct) => Task.CompletedTask;
}
```

---

## Step 1773: API Key Management

```csharp
// Domain/ApiKeys/ApiKey.cs
public sealed class ApiKey
{
    public Guid Id { get; init; } = Guid.NewGuid();
    public Guid TenantId { get; init; }
    public string Name { get; set; } = string.Empty;
    public string KeyHash { get; private set; } = string.Empty;
    public string KeyPrefix { get; private set; } = string.Empty; // "sk_live_abc123..." shown to user
    public ApiKeyScope Scopes { get; set; }
    public DateTime? ExpiresAt { get; set; }
    public DateTime CreatedAt { get; init; } = DateTime.UtcNow;
    public DateTime? LastUsedAt { get; set; }
    public bool IsRevoked { get; private set; }

    public static (ApiKey Key, string PlainTextKey) Create(
        Guid tenantId,
        string name,
        ApiKeyScope scopes,
        DateTime? expiresAt = null)
    {
        // Generate cryptographically secure random key
        var rawBytes = RandomNumberGenerator.GetBytes(32);
        var rawKey = Convert.ToBase64String(rawBytes)
            .Replace("+", "-").Replace("/", "_").TrimEnd('=');

        var prefix = $"sk_live_{rawKey[..8]}";
        var plainText = $"{prefix}_{rawKey}"; // Full key shown ONCE

        var keyHash = HashApiKey(plainText);

        var apiKey = new ApiKey
        {
            TenantId = tenantId,
            Name = name,
            KeyHash = keyHash,
            KeyPrefix = prefix,
            Scopes = scopes,
            ExpiresAt = expiresAt
        };

        return (apiKey, plainText);
    }

    public static string HashApiKey(string plainTextKey)
    {
        var bytes = SHA256.HashData(Encoding.UTF8.GetBytes(plainTextKey));
        return Convert.ToHexString(bytes);
    }

    public bool IsValid(string plainTextKey)
    {
        if (IsRevoked) return false;
        if (ExpiresAt.HasValue && ExpiresAt.Value < DateTime.UtcNow) return false;

        var hash = HashApiKey(plainTextKey);
        return CryptographicOperations.FixedTimeEquals(
            Encoding.UTF8.GetBytes(hash),
            Encoding.UTF8.GetBytes(KeyHash));
    }

    public void Revoke() => IsRevoked = true;
    public void RecordUsage() => LastUsedAt = DateTime.UtcNow;
}

[Flags]
public enum ApiKeyScope
{
    None = 0,
    ReadOrders = 1 << 0,
    WriteOrders = 1 << 1,
    ReadProducts = 1 << 2,
    WriteProducts = 1 << 3,
    Webhooks = 1 << 4,
    Admin = 1 << 10,
    FullAccess = ReadOrders | WriteOrders | ReadProducts | WriteProducts | Webhooks
}
```

---

## Step 1774: API Key Authentication Handler

```csharp
// Infrastructure/Auth/ApiKeyAuthenticationHandler.cs
public sealed class ApiKeyAuthenticationHandler
    : AuthenticationHandler<AuthenticationSchemeOptions>
{
    private const string ApiKeyHeaderName = "X-Api-Key";
    private const string ApiKeyQueryParam = "api_key";
    private readonly IApiKeyRepository _apiKeyRepo;
    private readonly TenantContextAccessor _tenantAccessor;

    public ApiKeyAuthenticationHandler(
        IOptionsMonitor<AuthenticationSchemeOptions> options,
        ILoggerFactory logger,
        UrlEncoder encoder,
        IApiKeyRepository apiKeyRepo,
        TenantContextAccessor tenantAccessor)
        : base(options, logger, encoder)
    {
        _apiKeyRepo = apiKeyRepo;
        _tenantAccessor = tenantAccessor;
    }

    protected override async Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        string? plainTextKey = null;

        // Try header first, then query string
        if (Request.Headers.TryGetValue(ApiKeyHeaderName, out var headerValue))
            plainTextKey = headerValue.ToString();
        else if (Request.Query.TryGetValue(ApiKeyQueryParam, out var queryValue))
            plainTextKey = queryValue.ToString();

        if (string.IsNullOrEmpty(plainTextKey))
            return AuthenticateResult.NoResult();

        // Extract prefix for fast lookup
        var prefix = ExtractPrefix(plainTextKey);
        if (prefix is null)
            return AuthenticateResult.Fail("Invalid API key format.");

        var apiKey = await _apiKeyRepo.FindByPrefixAsync(prefix);
        if (apiKey is null || !apiKey.IsValid(plainTextKey))
            return AuthenticateResult.Fail("Invalid or expired API key.");

        // Set tenant context
        var tenant = await _apiKeyRepo.GetTenantForKeyAsync(apiKey.TenantId);
        _tenantAccessor.Current = tenant;

        // Record usage asynchronously (fire-and-forget, non-critical)
        _ = _apiKeyRepo.RecordUsageAsync(apiKey.Id);

        var claims = new[]
        {
            new Claim(ClaimTypes.NameIdentifier, apiKey.Id.ToString()),
            new Claim("tenant_id", apiKey.TenantId.ToString()),
            new Claim("api_key_scopes", ((int)apiKey.Scopes).ToString()),
            new Claim("auth_method", "api_key")
        };

        var identity = new ClaimsIdentity(claims, Scheme.Name);
        var principal = new ClaimsPrincipal(identity);
        var ticket = new AuthenticationTicket(principal, Scheme.Name);

        return AuthenticateResult.Success(ticket);
    }

    private static string? ExtractPrefix(string key)
    {
        // Format: sk_live_XXXXXXXX_...
        var parts = key.Split('_');
        if (parts.Length < 4) return null;
        return $"{parts[0]}_{parts[1]}_{parts[2]}";
    }
}

// Scope-based authorization policy
public static class ApiKeyPolicies
{
    public const string ReadOrders = "ApiKey:ReadOrders";
    public const string WriteOrders = "ApiKey:WriteOrders";

    public static void AddApiKeyPolicies(this IServiceCollection services)
    {
        services.AddAuthorization(options =>
        {
            options.AddPolicy(ReadOrders, policy =>
                policy.RequireClaim("api_key_scopes")
                      .RequireAssertion(ctx =>
                      {
                          var scopesClaim = ctx.User.FindFirst("api_key_scopes")?.Value;
                          if (!int.TryParse(scopesClaim, out var scopesBit)) return false;
                          var scopes = (ApiKeyScope)scopesBit;
                          return scopes.HasFlag(ApiKeyScope.ReadOrders);
                      }));

            options.AddPolicy(WriteOrders, policy =>
                policy.RequireClaim("api_key_scopes")
                      .RequireAssertion(ctx =>
                      {
                          var scopesClaim = ctx.User.FindFirst("api_key_scopes")?.Value;
                          if (!int.TryParse(scopesClaim, out var scopesBit)) return false;
                          var scopes = (ApiKeyScope)scopesBit;
                          return scopes.HasFlag(ApiKeyScope.WriteOrders) ||
                                 scopes.HasFlag(ApiKeyScope.Admin);
                      }));
        });
    }
}
```

---

## Step 1775: Tenant-Aware Caching

```csharp
// Infrastructure/Caching/TenantAwareCacheService.cs
public sealed class TenantAwareCacheService
{
    private readonly IDistributedCache _cache;
    private readonly ITenantContext _tenantContext;
    private readonly JsonSerializerOptions _jsonOptions = new(JsonSerializerDefaults.Web);

    public TenantAwareCacheService(IDistributedCache cache, ITenantContext tenantContext)
    {
        _cache = cache;
        _tenantContext = tenantContext;
    }

    // All keys are prefixed with tenant ID — no cross-tenant leakage
    private string BuildKey(string key) => $"t:{_tenantContext.TenantId}:{key}";

    public async Task<T?> GetAsync<T>(string key, CancellationToken ct = default)
    {
        var bytes = await _cache.GetAsync(BuildKey(key), ct);
        if (bytes is null) return default;
        return JsonSerializer.Deserialize<T>(bytes, _jsonOptions);
    }

    public async Task SetAsync<T>(
        string key,
        T value,
        TimeSpan? absoluteExpiry = null,
        CancellationToken ct = default)
    {
        var bytes = JsonSerializer.SerializeToUtf8Bytes(value, _jsonOptions);
        var options = new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = absoluteExpiry ?? TimeSpan.FromMinutes(5)
        };
        await _cache.SetAsync(BuildKey(key), bytes, options, ct);
    }

    public async Task RemoveAsync(string key, CancellationToken ct = default)
        => await _cache.RemoveAsync(BuildKey(key), ct);

    public async Task<T> GetOrSetAsync<T>(
        string key,
        Func<CancellationToken, Task<T>> factory,
        TimeSpan? absoluteExpiry = null,
        CancellationToken ct = default)
    {
        var cached = await GetAsync<T>(key, ct);
        if (cached is not null) return cached;

        var value = await factory(ct);
        await SetAsync(key, value, absoluteExpiry, ct);
        return value;
    }

    // Invalidate all cache entries for current tenant
    public async Task InvalidateTenantCacheAsync(CancellationToken ct = default)
    {
        // With Redis: SCAN + DEL pattern
        // With in-memory: tag-based eviction
        var endpoint = $"t:{_tenantContext.TenantId}:*";
        // Implementation depends on cache provider
        await Task.CompletedTask;
    }
}
```

---

## Step 1776: Tenant Plan Enforcement — Feature Flags

```csharp
// Domain/Tenancy/TenantFeatures.cs
public static class TenantFeatures
{
    public static class Limits
    {
        public static int MaxProducts(TenantPlan plan) => plan switch
        {
            TenantPlan.Free => 10,
            TenantPlan.Pro => 1_000,
            TenantPlan.Enterprise => int.MaxValue,
            _ => 0
        };

        public static int MaxApiKeysPerTenant(TenantPlan plan) => plan switch
        {
            TenantPlan.Free => 1,
            TenantPlan.Pro => 10,
            TenantPlan.Enterprise => 100,
            _ => 0
        };

        public static TimeSpan DataRetentionPeriod(TenantPlan plan) => plan switch
        {
            TenantPlan.Free => TimeSpan.FromDays(30),
            TenantPlan.Pro => TimeSpan.FromDays(365),
            TenantPlan.Enterprise => TimeSpan.FromDays(365 * 7),
            _ => TimeSpan.Zero
        };
    }

    public static class Features
    {
        public static bool AdvancedAnalytics(TenantPlan plan) =>
            plan is TenantPlan.Pro or TenantPlan.Enterprise;

        public static bool CustomDomain(TenantPlan plan) =>
            plan is TenantPlan.Enterprise;

        public static bool SsoIntegration(TenantPlan plan) =>
            plan is TenantPlan.Enterprise;

        public static bool AuditLog(TenantPlan plan) =>
            plan is TenantPlan.Pro or TenantPlan.Enterprise;

        public static bool Webhooks(TenantPlan plan) =>
            plan is TenantPlan.Pro or TenantPlan.Enterprise;
    }
}

// Usage enforcement in application service
public sealed class ProductService
{
    private readonly AppDbContext _db;
    private readonly ITenantContext _tenant;

    public ProductService(AppDbContext db, ITenantContext tenant)
    {
        _db = db;
        _tenant = tenant;
    }

    public async Task<Product> CreateProductAsync(CreateProductCommand cmd, CancellationToken ct)
    {
        // Enforce plan limits
        var currentCount = await _db.Products.CountAsync(ct);
        var maxAllowed = TenantFeatures.Limits.MaxProducts(_tenant.Plan);

        if (currentCount >= maxAllowed)
            throw new PlanLimitExceededException(
                $"Your {_tenant.Plan} plan allows up to {maxAllowed} products. " +
                "Please upgrade to add more.");

        var product = new Product
        {
            TenantId = _tenant.TenantId,
            Name = cmd.Name,
            Price = cmd.Price
        };

        _db.Products.Add(product);
        await _db.SaveChangesAsync(ct);
        return product;
    }
}
```

---

## Step 1777: Tenant Provisioning Flow

```csharp
// Application/Tenancy/ProvisionTenantCommandHandler.cs
public record ProvisionTenantCommand(
    string CompanyName,
    string AdminEmail,
    string Slug,
    TenantPlan Plan);

public record ProvisionTenantResult(Guid TenantId, string AdminApiKey);

public sealed class ProvisionTenantCommandHandler
{
    private readonly AppDbContext _db;
    private readonly IApiKeyRepository _apiKeyRepo;
    private readonly TenantMigrationService _migrationService;
    private readonly IEmailService _emailService;
    private readonly ILogger<ProvisionTenantCommandHandler> _logger;

    public ProvisionTenantCommandHandler(
        AppDbContext db,
        IApiKeyRepository apiKeyRepo,
        TenantMigrationService migrationService,
        IEmailService emailService,
        ILogger<ProvisionTenantCommandHandler> logger)
    {
        _db = db;
        _apiKeyRepo = apiKeyRepo;
        _migrationService = migrationService;
        _emailService = emailService;
        _logger = logger;
    }

    public async Task<ProvisionTenantResult> HandleAsync(
        ProvisionTenantCommand cmd,
        CancellationToken ct)
    {
        // Validate slug uniqueness
        var slugExists = await _db.Tenants.AnyAsync(t => t.Slug == cmd.Slug, ct);
        if (slugExists)
            throw new ValidationException($"Slug '{cmd.Slug}' is already taken.");

        await using var tx = await _db.Database.BeginTransactionAsync(ct);
        try
        {
            // 1. Create tenant record
            var tenant = new TenantRecord
            {
                Id = Guid.NewGuid(),
                CompanyName = cmd.CompanyName,
                Slug = cmd.Slug,
                Plan = cmd.Plan,
                CreatedAt = DateTime.UtcNow,
                IsActive = true
            };
            _db.Tenants.Add(tenant);
            await _db.SaveChangesAsync(ct);

            // 2. Create initial admin API key
            var (apiKey, plainTextKey) = ApiKey.Create(
                tenant.Id,
                "Default Admin Key",
                ApiKeyScope.FullAccess | ApiKeyScope.Admin);
            _db.ApiKeys.Add(apiKey);
            await _db.SaveChangesAsync(ct);

            await tx.CommitAsync(ct);

            // 3. Run schema migration for this tenant
            // (outside TX — migrations are DDL, can't be rolled back in PostgreSQL)
            await _migrationService.MigrateTenantAsync(
                new TenantContext
                {
                    TenantId = tenant.Id,
                    TenantSlug = tenant.Slug,
                    Plan = tenant.Plan
                }, ct);

            // 4. Send welcome email
            await _emailService.SendWelcomeEmailAsync(cmd.AdminEmail, tenant.Slug, ct);

            _logger.LogInformation(
                "Tenant {TenantSlug} provisioned successfully. TenantId={TenantId}",
                tenant.Slug, tenant.Id);

            // plainTextKey shown only once — store in response, never in DB
            return new ProvisionTenantResult(tenant.Id, plainTextKey);
        }
        catch
        {
            await tx.RollbackAsync(ct);
            throw;
        }
    }
}
```

---

## Step 1778: Tenant Isolation Testing

```csharp
// Tests/Tenancy/TenantIsolationTests.cs
public sealed class TenantIsolationTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;

    public TenantIsolationTests(WebApplicationFactory<Program> factory)
        => _factory = factory;

    [Fact]
    public async Task TenantA_CannotSee_TenantB_Orders()
    {
        var tenantA = Guid.NewGuid();
        var tenantB = Guid.NewGuid();

        // Arrange: seed orders for both tenants directly in DB
        using var scope = _factory.Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();

        db.Orders.AddRange(
            new Order { TenantId = tenantA, CustomerEmail = "a@acme.com", Total = 100 },
            new Order { TenantId = tenantB, CustomerEmail = "b@beta.com", Total = 200 }
        );
        await db.SaveChangesAsync();

        // Act: query as Tenant A
        var clientA = _factory.WithWebHostBuilder(builder =>
            builder.ConfigureServices(services =>
                services.AddScoped<ITenantContext>(_ =>
                    new TenantContext { TenantId = tenantA, TenantSlug = "acme", Plan = TenantPlan.Pro })))
            .CreateClient();

        var response = await clientA.GetFromJsonAsync<List<Order>>("/api/orders");

        // Assert: only Tenant A's orders visible
        Assert.NotNull(response);
        Assert.All(response, order => Assert.Equal(tenantA, order.TenantId));
        Assert.DoesNotContain(response, o => o.CustomerEmail == "b@beta.com");
    }

    [Fact]
    public async Task ApiKey_With_ReadScope_Cannot_WriteOrders()
    {
        var client = _factory.CreateClient();
        var (_, readOnlyKey) = ApiKey.Create(
            Guid.NewGuid(), "Read Only", ApiKeyScope.ReadOrders);

        client.DefaultRequestHeaders.Add("X-Api-Key", readOnlyKey);

        var response = await client.PostAsJsonAsync("/api/orders",
            new { CustomerEmail = "test@example.com", Total = 50.0m });

        Assert.Equal(HttpStatusCode.Forbidden, response.StatusCode);
    }

    [Fact]
    public async Task PlanLimit_Enforced_OnProductCreation()
    {
        using var scope = _factory.Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        var freeTenantId = Guid.NewGuid();

        // Seed 10 products (Free plan limit)
        for (int i = 0; i < 10; i++)
        {
            db.Products.Add(new Product
            {
                TenantId = freeTenantId,
                Name = $"Product {i}",
                Price = 9.99m
            });
        }
        await db.SaveChangesAsync();

        var client = _factory.WithWebHostBuilder(builder =>
            builder.ConfigureServices(services =>
                services.AddScoped<ITenantContext>(_ =>
                    new TenantContext
                    {
                        TenantId = freeTenantId,
                        TenantSlug = "free-tenant",
                        Plan = TenantPlan.Free
                    })))
            .CreateClient();

        var response = await client.PostAsJsonAsync("/api/products",
            new { Name = "Over Limit", Price = 1.0m });

        Assert.Equal(HttpStatusCode.PaymentRequired, response.StatusCode);
    }
}
```

---

## Step 1779: Multi-Tenant Service Registration

```csharp
// Program.cs — Full multi-tenancy wiring
var builder = WebApplication.CreateBuilder(args);

// Tenant context — scoped per request
builder.Services.AddScoped<TenantContextAccessor>();
builder.Services.AddScoped<ITenantContext>(sp =>
{
    var accessor = sp.GetRequiredService<TenantContextAccessor>();
    return accessor.Current
        ?? throw new InvalidOperationException(
            "ITenantContext accessed outside of a tenant-resolved request.");
});

// Tenant resolvers (composite: JWT → header → subdomain)
builder.Services.AddScoped<JwtClaimsTenantResolver>();
builder.Services.AddScoped<HeaderTenantResolver>();
builder.Services.AddScoped<SubdomainTenantResolver>();
builder.Services.AddScoped<ITenantResolver>(sp =>
    new CompositeTenantResolver(new ITenantResolver[]
    {
        sp.GetRequiredService<JwtClaimsTenantResolver>(),
        sp.GetRequiredService<HeaderTenantResolver>(),
        sp.GetRequiredService<SubdomainTenantResolver>()
    }));

// Database with RLS interceptor
builder.Services.AddScoped<TenantSettingInterceptor>();
builder.Services.AddDbContext<AppDbContext>((sp, options) =>
{
    options.UseNpgsql(builder.Configuration.GetConnectionString("DefaultConnection"));
    options.AddInterceptors(sp.GetRequiredService<TenantSettingInterceptor>());
});

// Authentication — support both JWT Bearer and API Key
builder.Services
    .AddAuthentication(options =>
    {
        options.DefaultAuthenticateScheme = "Smart";
        options.DefaultChallengeScheme = "Smart";
    })
    .AddPolicyScheme("Smart", "JWT or API Key", options =>
    {
        options.ForwardDefaultSelector = ctx =>
            ctx.Request.Headers.ContainsKey("X-Api-Key") ||
            ctx.Request.Query.ContainsKey("api_key")
                ? "ApiKey"
                : JwtBearerDefaults.AuthenticationScheme;
    })
    .AddJwtBearer(options =>
    {
        options.Authority = builder.Configuration["Auth:Authority"];
        options.Audience = builder.Configuration["Auth:Audience"];
    })
    .AddScheme<AuthenticationSchemeOptions, ApiKeyAuthenticationHandler>(
        "ApiKey", _ => { });

builder.Services.AddApiKeyPolicies();

// Caching — tenant-aware
builder.Services.AddStackExchangeRedisCache(options =>
    options.Configuration = builder.Configuration.GetConnectionString("Redis"));
builder.Services.AddScoped<TenantAwareCacheService>();

// Tenant services
builder.Services.AddScoped<TenantMigrationService>();
builder.Services.AddHostedService<MigrationHostedService>();

var app = builder.Build();

app.UseAuthentication();
app.UseTenantResolution(); // AFTER authentication (JWT resolver needs Claims)
app.UseAuthorization();

app.MapControllers();
app.Run();
```

---

## Step 1780: API Controller with Tenant Context

```csharp
// Api/Controllers/OrdersController.cs
[ApiController]
[Route("api/orders")]
[Authorize]
public sealed class OrdersController : ControllerBase
{
    private readonly AppDbContext _db;
    private readonly ITenantContext _tenant;
    private readonly TenantAwareCacheService _cache;

    public OrdersController(
        AppDbContext db,
        ITenantContext tenant,
        TenantAwareCacheService cache)
    {
        _db = db;
        _tenant = tenant;
        _cache = cache;
    }

    [HttpGet]
    [Authorize(Policy = ApiKeyPolicies.ReadOrders)]
    public async Task<IActionResult> GetOrders(
        [FromQuery] int page = 1,
        [FromQuery] int pageSize = 20,
        CancellationToken ct = default)
    {
        // Query filter automatically applies TenantId == _tenant.TenantId
        var cacheKey = $"orders:page:{page}:size:{pageSize}";
        var orders = await _cache.GetOrSetAsync(
            cacheKey,
            async _ => await _db.Orders
                .AsNoTracking()
                .OrderByDescending(o => o.CreatedAt)
                .Skip((page - 1) * pageSize)
                .Take(pageSize)
                .Select(o => new OrderDto(o.Id, o.CustomerEmail, o.Total, o.Status))
                .ToListAsync(ct),
            TimeSpan.FromSeconds(30),
            ct);

        return Ok(new
        {
            TenantId = _tenant.TenantId,
            Plan = _tenant.Plan.ToString(),
            Data = orders,
            Page = page
        });
    }

    [HttpPost]
    [Authorize(Policy = ApiKeyPolicies.WriteOrders)]
    public async Task<IActionResult> CreateOrder(
        [FromBody] CreateOrderRequest request,
        CancellationToken ct)
    {
        if (!TenantFeatures.Features.Webhooks(_tenant.Plan) &&
            request.NotifyViaWebhook)
        {
            return StatusCode(
                StatusCodes.Status402PaymentRequired,
                new { error = "Webhooks require Pro or Enterprise plan." });
        }

        var order = new Order
        {
            // TenantId stamped automatically by SaveChanges override
            CustomerEmail = request.CustomerEmail,
            Total = request.Total,
            Status = OrderStatus.Pending
        };

        _db.Orders.Add(order);
        await _db.SaveChangesAsync(ct);

        // Invalidate cache for this tenant
        await _cache.RemoveAsync($"orders:page:1:size:20", ct);

        return CreatedAtAction(nameof(GetOrders), new { }, order);
    }
}

public record CreateOrderRequest(
    string CustomerEmail,
    decimal Total,
    bool NotifyViaWebhook = false);

public record OrderDto(Guid Id, string CustomerEmail, decimal Total, OrderStatus Status);
```

---

## Summary: Multi-Tenancy Architecture Patterns

| Pattern | Isolation | Cost | Migration Complexity | Best For |
|---|---|---|---|---|
| Shared DB + RLS | Medium (DB-enforced) | Lowest | Simple | High-volume SaaS |
| Shared DB + EF Filters | Medium (App-enforced) | Low | Simple | Standard SaaS |
| Schema-Per-Tenant | High | Medium | Per-tenant | Compliance-heavy |
| DB-Per-Tenant | Highest | High | Per-tenant | Enterprise / Financial |

**Defense-in-depth**: Use **both** EF query filters (catches accidental queries) **and** PostgreSQL RLS (catches raw SQL / ORM bypass). Neither alone is sufficient for high-compliance requirements.

**Key rules**:
1. Never expose `TenantId` as a user-settable parameter — always derive from auth context
2. Stamp `TenantId` on `SaveChanges`, not in application code, to avoid forgetting
3. Test tenant isolation explicitly — integration tests that seed both tenants and verify leakage doesn't occur
4. API keys should store only a hash — return plain text exactly once at creation
5. Plan enforcement at application layer + rate limiting at infrastructure layer

---

*Next: Part 72 — Domain-Driven Design Advanced: Aggregates, Bounded Contexts & Anti-Corruption Layer*
