# Part 45: Advanced Security & Zero Trust (Steps 1281-1320)

## Step 1281: Security Architecture Overview

### Zero Trust Model

```
Traditional Perimeter Security:
─────────────────────────────────
Outside │  Firewall │  Inside (trusted)
Internet │──────────│  Everything trusted
         │          │  once inside

Zero Trust Architecture:
─────────────────────────────────
"Never Trust, Always Verify"
  ┌───────────────────────────────┐
  │   Every request:              │
  │   ✓ Authenticate Identity    │
  │   ✓ Authorize Access         │
  │   ✓ Verify Device            │
  │   ✓ Check Context            │
  │   ✓ Audit Everything         │
  └───────────────────────────────┘
```

### Security Layers for .NET Applications

```
┌─────────────────────────────────────────────────────────────┐
│                    Security Layers                           │
├─────────────────────────────────────────────────────────────┤
│  L1: Network    │ TLS 1.3, mTLS, Private endpoints         │
│  L2: Identity   │ OAuth 2.0, OIDC, JWT, MFA                │
│  L3: API        │ Rate limiting, WAF, Input validation      │
│  L4: Application│ AuthZ policies, RBAC/ABAC                 │
│  L5: Data       │ Encryption at rest/transit, masking       │
│  L6: Audit      │ Structured logs, SIEM integration         │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 1282: OAuth 2.0 and OIDC Implementation

### Identity Server (Duende) Setup

```csharp
// IdentityServer/Program.cs
using Duende.IdentityServer;
using Duende.IdentityServer.Models;

var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddIdentityServer(options =>
    {
        options.Events.RaiseErrorEvents = true;
        options.Events.RaiseInformationEvents = true;
        options.Events.RaiseFailureEvents = true;
        options.Events.RaiseSuccessEvents = true;
        options.EmitStaticAudienceClaim = true;
    })
    .AddInMemoryIdentityResources(Config.IdentityResources)
    .AddInMemoryApiScopes(Config.ApiScopes)
    .AddInMemoryApiResources(Config.ApiResources)
    .AddInMemoryClients(Config.Clients)
    .AddAspNetIdentity<ApplicationUser>();

var app = builder.Build();
app.UseIdentityServer();
app.Run();

public static class Config
{
    public static IEnumerable<IdentityResource> IdentityResources =>
    [
        new IdentityResources.OpenId(),
        new IdentityResources.Profile(),
        new IdentityResources.Email(),
        new IdentityResource("roles", "User Roles", ["role"]),
    ];

    public static IEnumerable<ApiScope> ApiScopes =>
    [
        new ApiScope("orders.read", "Read orders"),
        new ApiScope("orders.write", "Write orders"),
        new ApiScope("admin", "Admin access"),
    ];

    public static IEnumerable<ApiResource> ApiResources =>
    [
        new ApiResource("orders-api", "Orders API")
        {
            Scopes = ["orders.read", "orders.write"],
            UserClaims = ["role", "sub"],
        },
    ];

    public static IEnumerable<Client> Clients =>
    [
        // Machine-to-machine
        new Client
        {
            ClientId = "orders-worker",
            ClientName = "Orders Background Worker",
            ClientSecrets = [new Secret("worker-secret".Sha256())],
            AllowedGrantTypes = GrantTypes.ClientCredentials,
            AllowedScopes = ["orders.read", "orders.write"],
        },

        // SPA (React/Angular)
        new Client
        {
            ClientId = "orders-spa",
            ClientName = "Orders SPA",
            AllowedGrantTypes = GrantTypes.Code,
            RequirePkce = true,
            RequireClientSecret = false,
            RedirectUris = ["https://myapp.com/callback"],
            PostLogoutRedirectUris = ["https://myapp.com"],
            AllowedCorsOrigins = ["https://myapp.com"],
            AllowedScopes =
            [
                IdentityServerConstants.StandardScopes.OpenId,
                IdentityServerConstants.StandardScopes.Profile,
                "orders.read",
                "orders.write",
            ],
            AccessTokenLifetime = 3600,
            RefreshTokenUsage = TokenUsage.ReUse,
            AllowOfflineAccess = true,
        },
    ];
}
```

---

## Step 1283: JWT Authentication in ASP.NET Core

### Complete JWT Setup

```csharp
// Program.cs - API with JWT auth
builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = builder.Configuration["Auth:Authority"];
        options.Audience = "orders-api";

        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ClockSkew = TimeSpan.FromSeconds(30),
        };

        // Validate JWT signing key
        options.TokenValidationParameters.IssuerSigningKeyResolver =
            (token, securityToken, kid, parameters) =>
            {
                var client = new HttpClient();
                var json = client.GetStringAsync(
                    $"{options.Authority}/.well-known/openid-configuration/jwks").Result;
                var keys = new JsonWebKeySet(json);
                return keys.GetSigningKeys();
            };

        // Custom events
        options.Events = new JwtBearerEvents
        {
            OnAuthenticationFailed = ctx =>
            {
                ctx.Response.Headers["WWW-Authenticate"] = "Bearer error=\"invalid_token\"";
                return Task.CompletedTask;
            },
            OnChallenge = ctx =>
            {
                ctx.HandleResponse();
                ctx.Response.StatusCode = 401;
                ctx.Response.ContentType = "application/json";
                return ctx.Response.WriteAsJsonAsync(new
                {
                    error = "Unauthorized",
                    detail = "Valid JWT token required"
                });
            },
        };
    });

// Authorization policies
builder.Services.AddAuthorizationBuilder()
    .AddPolicy("ReadOrders", policy =>
    {
        policy.RequireAuthenticatedUser();
        policy.RequireClaim("scope", "orders.read");
    })
    .AddPolicy("WriteOrders", policy =>
    {
        policy.RequireAuthenticatedUser();
        policy.RequireClaim("scope", "orders.write");
    })
    .AddPolicy("AdminOnly", policy =>
    {
        policy.RequireAuthenticatedUser();
        policy.RequireRole("admin");
        policy.RequireClaim("scope", "admin");
    })
    .AddPolicy("SameCustomer", policy =>
    {
        policy.RequireAuthenticatedUser();
        policy.AddRequirements(new SameCustomerRequirement());
    });

// Custom authorization requirement
public class SameCustomerRequirement : IAuthorizationRequirement { }

public class SameCustomerHandler : AuthorizationHandler<SameCustomerRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        SameCustomerRequirement requirement)
    {
        if (context.Resource is HttpContext httpContext)
        {
            var userId = context.User.FindFirst("sub")?.Value;
            var requestedCustomerId = httpContext.GetRouteValue("customerId")?.ToString();

            if (userId == requestedCustomerId ||
                context.User.HasClaim("role", "admin"))
            {
                context.Succeed(requirement);
            }
        }

        return Task.CompletedTask;
    }
}
```

---

## Step 1284: Role-Based and Policy-Based Authorization

### RBAC Implementation

```csharp
// Domain roles and permissions
public enum Permission
{
    ReadOrders,
    CreateOrder,
    CancelOrder,
    RefundOrder,
    ViewAllOrders,   // Admin
    DeleteOrder,     // Admin
    ManageUsers,     // Admin
}

public static class Roles
{
    public const string Customer = "customer";
    public const string SupportAgent = "support";
    public const string Manager = "manager";
    public const string Admin = "admin";

    private static readonly Dictionary<string, IReadOnlySet<Permission>> RolePermissions =
        new()
        {
            [Customer] = new HashSet<Permission>
            {
                Permission.ReadOrders,
                Permission.CreateOrder,
                Permission.CancelOrder,
            },
            [SupportAgent] = new HashSet<Permission>
            {
                Permission.ReadOrders,
                Permission.ViewAllOrders,
                Permission.CancelOrder,
            },
            [Manager] = new HashSet<Permission>
            {
                Permission.ReadOrders,
                Permission.ViewAllOrders,
                Permission.CancelOrder,
                Permission.RefundOrder,
            },
            [Admin] = new HashSet<Permission>(Enum.GetValues<Permission>()),
        };

    public static bool HasPermission(string role, Permission permission)
        => RolePermissions.TryGetValue(role, out var perms) && perms.Contains(permission);
}

// Permission attribute
[AttributeUsage(AttributeTargets.Method | AttributeTargets.Class)]
public class RequirePermissionAttribute(Permission permission) : Attribute
{
    public Permission Permission { get; } = permission;
}

// Authorization handler
public class PermissionAuthorizationHandler
    : AuthorizationHandler<PermissionRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        PermissionRequirement requirement)
    {
        var userRoles = context.User.Claims
            .Where(c => c.Type == ClaimTypes.Role)
            .Select(c => c.Value);

        var hasPermission = userRoles
            .Any(role => Roles.HasPermission(role, requirement.Permission));

        if (hasPermission)
            context.Succeed(requirement);

        return Task.CompletedTask;
    }
}

public class PermissionRequirement(Permission permission) : IAuthorizationRequirement
{
    public Permission Permission { get; } = permission;
}
```

### ABAC (Attribute-Based Access Control)

```csharp
public class ResourceAccessPolicy : IAuthorizationRequirement
{
    public string ResourceType { get; init; } = string.Empty;
    public string Action { get; init; } = string.Empty;
}

public class ResourceAccessHandler
    : AuthorizationHandler<ResourceAccessPolicy, IResourceWithOwner>
{
    private readonly IUserContextService _userContext;

    public ResourceAccessHandler(IUserContextService userContext)
    {
        _userContext = userContext;
    }

    protected override async Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        ResourceAccessPolicy requirement,
        IResourceWithOwner resource)
    {
        var userId = context.User.FindFirst("sub")?.Value;
        if (userId is null) return;

        var userContext = await _userContext.GetUserContextAsync(userId);

        // Evaluate access based on multiple attributes
        var hasAccess = EvaluateAccess(
            user: userContext,
            resource: resource,
            action: requirement.Action);

        if (hasAccess)
            context.Succeed(requirement);
    }

    private static bool EvaluateAccess(
        UserContext user,
        IResourceWithOwner resource,
        string action)
    {
        // Owner can do anything
        if (resource.OwnerId == user.UserId) return true;

        // Admin can do anything
        if (user.Roles.Contains("admin")) return true;

        // Support can read
        if (user.Roles.Contains("support") && action == "read") return true;

        // Same organization members can read
        if (resource.OrganizationId == user.OrganizationId && action == "read")
            return true;

        return false;
    }
}

public interface IResourceWithOwner
{
    string OwnerId { get; }
    string OrganizationId { get; }
}

public class UserContext
{
    public required string UserId { get; init; }
    public required string OrganizationId { get; init; }
    public required IReadOnlyList<string> Roles { get; init; }
    public required Dictionary<string, string> Attributes { get; init; }
}
```

---

## Step 1285: Encryption and Key Management

### Data Encryption Service

```csharp
using System.Security.Cryptography;
using Microsoft.AspNetCore.DataProtection;

public class DataEncryptionService
{
    private readonly IDataProtectionProvider _dataProtection;

    public DataEncryptionService(IDataProtectionProvider dataProtection)
    {
        _dataProtection = dataProtection;
    }

    // For sensitive user data (reversible, purpose-isolated)
    public string ProtectSensitiveData(string plaintext, string purpose)
    {
        var protector = _dataProtection.CreateProtector(purpose);
        return protector.Protect(plaintext);
    }

    public string UnprotectSensitiveData(string ciphertext, string purpose)
    {
        var protector = _dataProtection.CreateProtector(purpose);
        return protector.Unprotect(ciphertext);
    }

    // AES-GCM for symmetric encryption
    public (byte[] Ciphertext, byte[] Nonce, byte[] Tag) EncryptAesGcm(
        byte[] plaintext,
        byte[] key)
    {
        var nonce = new byte[AesGcm.NonceByteSizes.MaxSize];
        RandomNumberGenerator.Fill(nonce);

        var ciphertext = new byte[plaintext.Length];
        var tag = new byte[AesGcm.TagByteSizes.MaxSize];

        using var aesGcm = new AesGcm(key, AesGcm.TagByteSizes.MaxSize);
        aesGcm.Encrypt(nonce, plaintext, ciphertext, tag);

        return (ciphertext, nonce, tag);
    }

    public byte[] DecryptAesGcm(
        byte[] ciphertext,
        byte[] key,
        byte[] nonce,
        byte[] tag)
    {
        var plaintext = new byte[ciphertext.Length];
        using var aesGcm = new AesGcm(key, AesGcm.TagByteSizes.MaxSize);
        aesGcm.Decrypt(nonce, ciphertext, tag, plaintext);
        return plaintext;
    }

    // RSA for asymmetric encryption (e.g., encrypting DEK)
    public byte[] EncryptRsa(byte[] data, RSAParameters publicKey)
    {
        using var rsa = RSA.Create();
        rsa.ImportParameters(publicKey);
        return rsa.Encrypt(data, RSAEncryptionPadding.OaepSHA256);
    }

    public byte[] DecryptRsa(byte[] data, RSAParameters privateKey)
    {
        using var rsa = RSA.Create();
        rsa.ImportParameters(privateKey);
        return rsa.Decrypt(data, RSAEncryptionPadding.OaepSHA256);
    }
}
```

### Envelope Encryption (DEK + KEK Pattern)

```csharp
public class EnvelopeEncryptionService
{
    private readonly SecretManager _secretManager;

    public EnvelopeEncryptionService(SecretManager secretManager)
    {
        _secretManager = secretManager;
    }

    // Encrypt data using envelope encryption
    public async Task<EncryptedData> EncryptAsync(byte[] plaintext)
    {
        // 1. Generate Data Encryption Key (DEK)
        var dek = new byte[32];
        RandomNumberGenerator.Fill(dek);

        // 2. Encrypt plaintext with DEK (AES-GCM)
        var nonce = new byte[12];
        RandomNumberGenerator.Fill(nonce);

        var ciphertext = new byte[plaintext.Length];
        var tag = new byte[16];

        using var aesGcm = new AesGcm(dek, 16);
        aesGcm.Encrypt(nonce, plaintext, ciphertext, tag);

        // 3. Encrypt DEK with KEK (from Key Vault)
        var kek = await GetKeyEncryptionKeyAsync();
        var encryptedDek = EncryptDek(dek, kek);

        CryptographicOperations.ZeroMemory(dek); // Zero out DEK

        return new EncryptedData
        {
            Ciphertext = ciphertext,
            Nonce = nonce,
            Tag = tag,
            EncryptedDek = encryptedDek,
            KeyVersion = kek.Version,
        };
    }

    public async Task<byte[]> DecryptAsync(EncryptedData encryptedData)
    {
        // 1. Get KEK from Key Vault
        var kek = await GetKeyEncryptionKeyByVersionAsync(encryptedData.KeyVersion);

        // 2. Decrypt DEK
        var dek = DecryptDek(encryptedData.EncryptedDek, kek);

        try
        {
            // 3. Decrypt data with DEK
            var plaintext = new byte[encryptedData.Ciphertext.Length];
            using var aesGcm = new AesGcm(dek, 16);
            aesGcm.Decrypt(
                encryptedData.Nonce,
                encryptedData.Ciphertext,
                encryptedData.Tag,
                plaintext);

            return plaintext;
        }
        finally
        {
            CryptographicOperations.ZeroMemory(dek);
        }
    }

    private Task<KeyEncryptionKey> GetKeyEncryptionKeyAsync() =>
        throw new NotImplementedException("Get from Azure Key Vault / HashiCorp Vault");

    private Task<KeyEncryptionKey> GetKeyEncryptionKeyByVersionAsync(string version) =>
        throw new NotImplementedException("Get specific version from Key Vault");

    private static byte[] EncryptDek(byte[] dek, KeyEncryptionKey kek) =>
        throw new NotImplementedException("RSA or AES-KW wrapping");

    private static byte[] DecryptDek(byte[] encryptedDek, KeyEncryptionKey kek) =>
        throw new NotImplementedException("RSA or AES-KW unwrapping");
}

public class EncryptedData
{
    public required byte[] Ciphertext { get; init; }
    public required byte[] Nonce { get; init; }
    public required byte[] Tag { get; init; }
    public required byte[] EncryptedDek { get; init; }
    public required string KeyVersion { get; init; }
}

public class KeyEncryptionKey
{
    public required string Version { get; init; }
    public required RSAParameters PublicKey { get; init; }
    public RSAParameters? PrivateKey { get; init; }
}
```

---

## Step 1286: Input Validation and Security

### Model Validation with FluentValidation

```csharp
using FluentValidation;

public class CreateOrderRequest
{
    public Guid CustomerId { get; init; }
    public IReadOnlyList<OrderLineRequest> Items { get; init; } = [];
    public string? SpecialInstructions { get; init; }
    public string DeliveryAddress { get; init; } = string.Empty;
}

public class CreateOrderRequestValidator : AbstractValidator<CreateOrderRequest>
{
    private static readonly Regex SafeTextRegex =
        new(@"^[a-zA-Z0-9\s\-\.,!?]+$", RegexOptions.Compiled);

    public CreateOrderRequestValidator()
    {
        RuleFor(x => x.CustomerId)
            .NotEmpty()
            .WithMessage("Customer ID is required");

        RuleFor(x => x.Items)
            .NotEmpty()
            .WithMessage("Order must have at least one item")
            .Must(items => items.Count <= 100)
            .WithMessage("Order cannot have more than 100 items");

        RuleForEach(x => x.Items)
            .SetValidator(new OrderLineRequestValidator());

        RuleFor(x => x.SpecialInstructions)
            .MaximumLength(500)
            .Matches(SafeTextRegex)
            .When(x => x.SpecialInstructions is not null)
            .WithMessage("Special instructions contain invalid characters");

        RuleFor(x => x.DeliveryAddress)
            .NotEmpty()
            .MaximumLength(500)
            .Must(addr => !addr.Contains('<') && !addr.Contains('>'))
            .WithMessage("Address contains invalid characters");
    }
}

public class OrderLineRequestValidator : AbstractValidator<OrderLineRequest>
{
    public OrderLineRequestValidator()
    {
        RuleFor(x => x.ProductId)
            .NotEmpty()
            .Must(id => Guid.TryParse(id, out _))
            .WithMessage("Invalid product ID format");

        RuleFor(x => x.Quantity)
            .GreaterThan(0)
            .LessThanOrEqualTo(1000)
            .WithMessage("Quantity must be between 1 and 1000");
    }
}

public record OrderLineRequest(string ProductId, int Quantity);
```

### SQL Injection Prevention

```csharp
// Bad: SQL injection vulnerable
public async Task<List<Order>> FindByStatusBadAsync(string status)
{
    // NEVER DO THIS
    var query = $"SELECT * FROM Orders WHERE Status = '{status}'";
    return await _context.Orders.FromSqlRaw(query).ToListAsync();
}

// Good: Parameterized query with EF Core
public async Task<List<Order>> FindByStatusGoodAsync(string status)
{
    // Option 1: LINQ (safest, auto-parameterized)
    return await _context.Orders
        .Where(o => o.Status == status)
        .ToListAsync();
}

public async Task<List<Order>> FindByStatusWithRawSqlAsync(string status)
{
    // Option 2: Parameterized raw SQL
    return await _context.Orders
        .FromSql($"SELECT * FROM Orders WHERE Status = {status}")
        .ToListAsync();
}
```

### XSS Prevention

```csharp
using System.Web;

public class HtmlSanitizationService
{
    // Encode user input before rendering to HTML
    public string SanitizeHtml(string input)
    {
        return HttpUtility.HtmlEncode(input);
    }

    // For rich text: use a proper HTML sanitizer library
    // Install: dotnet add package HtmlSanitizer
    public string SanitizeRichText(string html)
    {
        var sanitizer = new HtmlSanitizer();
        sanitizer.AllowedTags.Remove("script");
        sanitizer.AllowedTags.Remove("style");
        sanitizer.AllowedAttributes.Remove("onclick");
        sanitizer.AllowedAttributes.Remove("onerror");

        return sanitizer.Sanitize(html);
    }
}

// Configure CSP headers
app.Use(async (context, next) =>
{
    context.Response.Headers.Add("Content-Security-Policy",
        "default-src 'self'; " +
        "script-src 'self' https://cdn.jsdelivr.net; " +
        "style-src 'self' 'unsafe-inline'; " +
        "img-src 'self' data:; " +
        "frame-ancestors 'none'");

    context.Response.Headers.Add("X-Frame-Options", "DENY");
    context.Response.Headers.Add("X-Content-Type-Options", "nosniff");
    context.Response.Headers.Add("Referrer-Policy", "strict-origin-when-cross-origin");
    context.Response.Headers.Add("Permissions-Policy",
        "camera=(), microphone=(), geolocation=()");

    await next();
});
```

---

## Step 1287: API Security Patterns

### API Key Management

```csharp
public class ApiKeyMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IApiKeyService _apiKeyService;

    public ApiKeyMiddleware(RequestDelegate next, IApiKeyService apiKeyService)
    {
        _next = next;
        _apiKeyService = apiKeyService;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        if (!context.Request.Headers.TryGetValue("X-Api-Key", out var apiKey))
        {
            context.Response.StatusCode = 401;
            await context.Response.WriteAsJsonAsync(new
            {
                error = "API key is required"
            });
            return;
        }

        var principal = await _apiKeyService.ValidateKeyAsync(apiKey.ToString());
        if (principal is null)
        {
            context.Response.StatusCode = 401;
            await context.Response.WriteAsJsonAsync(new
            {
                error = "Invalid API key"
            });
            return;
        }

        context.User = principal;
        await _next(context);
    }
}

public class ApiKeyService : IApiKeyService
{
    private readonly IDistributedCache _cache;
    private readonly IApiKeyRepository _repository;

    public ApiKeyService(IDistributedCache cache, IApiKeyRepository repository)
    {
        _cache = cache;
        _repository = repository;
    }

    public async Task<ClaimsPrincipal?> ValidateKeyAsync(string rawKey)
    {
        // Hash the key before lookup
        var keyHash = ComputeHash(rawKey);
        var cacheKey = $"apikey:{keyHash}";

        // Check cache first
        var cachedJson = await _cache.GetStringAsync(cacheKey);
        if (cachedJson is not null)
        {
            var cached = JsonSerializer.Deserialize<ApiKeyInfo>(cachedJson)!;
            return cached.IsActive ? BuildPrincipal(cached) : null;
        }

        // Check database
        var keyInfo = await _repository.FindByHashAsync(keyHash);
        if (keyInfo is null) return null;

        // Cache for 5 minutes
        await _cache.SetStringAsync(
            cacheKey,
            JsonSerializer.Serialize(keyInfo),
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5),
            });

        return keyInfo.IsActive ? BuildPrincipal(keyInfo) : null;
    }

    public async Task<ApiKeyInfo> CreateKeyAsync(
        string clientId,
        IReadOnlyList<string> scopes)
    {
        // Generate cryptographically secure key
        var rawKey = GenerateSecureKey();
        var keyHash = ComputeHash(rawKey);

        var keyInfo = new ApiKeyInfo
        {
            Id = Guid.NewGuid(),
            ClientId = clientId,
            KeyHash = keyHash,
            Scopes = scopes,
            CreatedAt = DateTime.UtcNow,
            IsActive = true,
        };

        await _repository.SaveAsync(keyInfo);

        return keyInfo with { RawKey = rawKey }; // Only returned once
    }

    private static string GenerateSecureKey()
    {
        var bytes = new byte[32];
        RandomNumberGenerator.Fill(bytes);
        return $"sk_{Convert.ToBase64String(bytes).TrimEnd('=').Replace('+', '-').Replace('/', '_')}";
    }

    private static string ComputeHash(string key)
    {
        var bytes = SHA256.HashData(Encoding.UTF8.GetBytes(key));
        return Convert.ToHexString(bytes);
    }

    private static ClaimsPrincipal BuildPrincipal(ApiKeyInfo keyInfo)
    {
        var claims = new List<Claim>
        {
            new(ClaimTypes.NameIdentifier, keyInfo.ClientId),
            new("client_id", keyInfo.ClientId),
        };

        claims.AddRange(keyInfo.Scopes.Select(s => new Claim("scope", s)));

        return new ClaimsPrincipal(
            new ClaimsIdentity(claims, "ApiKey"));
    }
}

public class ApiKeyInfo
{
    public Guid Id { get; init; }
    public required string ClientId { get; init; }
    public required string KeyHash { get; init; }
    public required IReadOnlyList<string> Scopes { get; init; }
    public DateTime CreatedAt { get; init; }
    public DateTime? ExpiresAt { get; init; }
    public bool IsActive { get; init; }
    public string? RawKey { get; init; } // Only set on creation
}
```

---

## Step 1288: Audit Logging

### Comprehensive Audit Trail

```csharp
public class AuditLog
{
    public Guid Id { get; init; } = Guid.NewGuid();
    public required string UserId { get; init; }
    public required string UserEmail { get; init; }
    public required string Action { get; init; }
    public required string ResourceType { get; init; }
    public string? ResourceId { get; init; }
    public required string IpAddress { get; init; }
    public required string UserAgent { get; init; }
    public string? SessionId { get; init; }
    public bool Succeeded { get; init; }
    public string? FailureReason { get; init; }
    public required DateTime Timestamp { get; init; }
    public Dictionary<string, object> Metadata { get; init; } = [];
}

public class AuditService
{
    private readonly ILogger<AuditService> _logger;
    private readonly IAuditRepository _auditRepository;

    public AuditService(
        ILogger<AuditService> logger,
        IAuditRepository auditRepository)
    {
        _logger = logger;
        _auditRepository = auditRepository;
    }

    public async Task LogAsync(
        AuditContext context,
        string action,
        string resourceType,
        bool succeeded,
        string? resourceId = null,
        string? failureReason = null,
        Dictionary<string, object>? metadata = null)
    {
        var log = new AuditLog
        {
            UserId = context.UserId,
            UserEmail = context.UserEmail,
            Action = action,
            ResourceType = resourceType,
            ResourceId = resourceId,
            IpAddress = context.IpAddress,
            UserAgent = context.UserAgent,
            SessionId = context.SessionId,
            Succeeded = succeeded,
            FailureReason = failureReason,
            Timestamp = DateTime.UtcNow,
            Metadata = metadata ?? [],
        };

        // Write to persistent storage
        await _auditRepository.SaveAsync(log);

        // Also emit structured log for SIEM
        _logger.LogInformation(
            "[AUDIT] {Action} {ResourceType} {ResourceId} by {UserId} from {IpAddress}: {Status}",
            action, resourceType, resourceId ?? "N/A",
            context.UserId, context.IpAddress,
            succeeded ? "SUCCESS" : $"FAILED ({failureReason})");
    }
}

public record AuditContext(
    string UserId,
    string UserEmail,
    string IpAddress,
    string UserAgent,
    string? SessionId);

// Audit filter attribute
[AttributeUsage(AttributeTargets.Method)]
public class AuditAttribute : Attribute
{
    public string Action { get; init; }
    public string ResourceType { get; init; }

    public AuditAttribute(string action, string resourceType)
    {
        Action = action;
        ResourceType = resourceType;
    }
}

// Action filter to apply audit logging
public class AuditActionFilter : IAsyncActionFilter
{
    private readonly AuditService _auditService;
    private readonly IHttpContextAccessor _httpContextAccessor;

    public AuditActionFilter(
        AuditService auditService,
        IHttpContextAccessor httpContextAccessor)
    {
        _auditService = auditService;
        _httpContextAccessor = httpContextAccessor;
    }

    public async Task OnActionExecutionAsync(
        ActionExecutingContext context,
        ActionExecutionDelegate next)
    {
        var auditAttr = context.ActionDescriptor.EndpointMetadata
            .OfType<AuditAttribute>()
            .FirstOrDefault();

        if (auditAttr is null)
        {
            await next();
            return;
        }

        var httpContext = _httpContextAccessor.HttpContext!;
        var userId = httpContext.User.FindFirst("sub")?.Value ?? "anonymous";
        var userEmail = httpContext.User.FindFirst("email")?.Value ?? "unknown";
        var ip = httpContext.Connection.RemoteIpAddress?.ToString() ?? "unknown";
        var userAgent = httpContext.Request.Headers.UserAgent.ToString();

        var auditCtx = new AuditContext(userId, userEmail, ip, userAgent, null);

        var executed = await next();
        var succeeded = executed.Exception is null &&
            (executed.Result is not ObjectResult objectResult ||
             objectResult.StatusCode is null or < 400);

        await _auditService.LogAsync(
            auditCtx,
            auditAttr.Action,
            auditAttr.ResourceType,
            succeeded,
            failureReason: executed.Exception?.Message);
    }
}
```

---

## Step 1289: OWASP Top 10 Prevention

### Security Headers Middleware

```csharp
public class SecurityHeadersMiddleware
{
    private readonly RequestDelegate _next;

    public SecurityHeadersMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(HttpContext context)
    {
        // HSTS: Enforce HTTPS
        if (context.Request.IsHttps)
        {
            context.Response.Headers.StrictTransportSecurity =
                "max-age=31536000; includeSubDomains; preload";
        }

        // Prevent MIME sniffing
        context.Response.Headers.XContentTypeOptions = "nosniff";

        // Prevent clickjacking
        context.Response.Headers.XFrameOptions = "DENY";

        // XSS protection (legacy but still useful)
        context.Response.Headers["X-XSS-Protection"] = "1; mode=block";

        // Remove server header
        context.Response.Headers.Remove("Server");
        context.Response.Headers.Remove("X-Powered-By");
        context.Response.Headers.Remove("X-AspNet-Version");
        context.Response.Headers.Remove("X-AspNetMvc-Version");

        // Referrer policy
        context.Response.Headers["Referrer-Policy"] = "strict-origin-when-cross-origin";

        // Permissions policy
        context.Response.Headers["Permissions-Policy"] =
            "camera=(), microphone=(), geolocation=(), payment=()";

        await _next(context);
    }
}
```

### CSRF Protection

```csharp
// For APIs using JWT - validate Origin/Referer
public class ApiCsrfMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IReadOnlyList<string> _allowedOrigins;

    public ApiCsrfMiddleware(RequestDelegate next, IConfiguration config)
    {
        _next = next;
        _allowedOrigins = config.GetSection("Security:AllowedOrigins")
            .Get<string[]>() ?? [];
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // Skip for GET, HEAD, OPTIONS
        if (context.Request.Method is "GET" or "HEAD" or "OPTIONS")
        {
            await _next(context);
            return;
        }

        // Skip if JWT Bearer auth (CSRF not needed for API key/JWT)
        if (context.Request.Headers.Authorization.ToString().StartsWith("Bearer "))
        {
            await _next(context);
            return;
        }

        // Check Origin header
        var origin = context.Request.Headers.Origin.ToString();
        if (!string.IsNullOrEmpty(origin) &&
            !_allowedOrigins.Contains(origin))
        {
            context.Response.StatusCode = 403;
            await context.Response.WriteAsJsonAsync(new
            {
                error = "CSRF validation failed"
            });
            return;
        }

        await _next(context);
    }
}
```

---

## Step 1290: mTLS - Mutual TLS

### mTLS for Service-to-Service Communication

```csharp
// Service A calling Service B with mTLS
public class MtlsHttpClient
{
    private readonly HttpClient _client;

    public MtlsHttpClient(IConfiguration config)
    {
        var certPath = config["Mtls:CertPath"]!;
        var certPassword = config["Mtls:CertPassword"]!;

        var cert = new X509Certificate2(certPath, certPassword);

        var handler = new HttpClientHandler
        {
            ClientCertificates = { cert },
            // Validate server certificate
            ServerCertificateCustomValidationCallback = (_, serverCert, chain, errors) =>
            {
                if (errors != SslPolicyErrors.None) return false;

                // Validate server cert against known CA
                var expectedThumbprint = config["Mtls:ServerCertThumbprint"]!;
                return serverCert?.Thumbprint == expectedThumbprint;
            },
        };

        _client = new HttpClient(handler)
        {
            BaseAddress = new Uri(config["Services:OrdersApi"]!),
        };
    }

    public async Task<T?> GetAsync<T>(string path)
    {
        var response = await _client.GetAsync(path);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<T>();
    }
}

// Service B: validate client certificate
builder.Services
    .AddAuthentication(CertificateAuthenticationDefaults.AuthenticationScheme)
    .AddCertificate(options =>
    {
        options.AllowedCertificateTypes = CertificateTypes.SelfSigned;
        options.RevocationMode = X509RevocationMode.NoCheck; // Use OCSP in production

        options.Events = new CertificateAuthenticationEvents
        {
            OnCertificateValidated = ctx =>
            {
                // Validate thumbprint
                var allowedThumbprints = new[]
                {
                    "A1B2C3D4...", // orders-worker
                    "E5F6G7H8...", // payments-api
                };

                if (!allowedThumbprints.Contains(ctx.ClientCertificate.Thumbprint))
                {
                    ctx.Fail("Certificate not authorized");
                    return Task.CompletedTask;
                }

                var claims = new List<Claim>
                {
                    new(ClaimTypes.NameIdentifier, ctx.ClientCertificate.Subject),
                    new("thumbprint", ctx.ClientCertificate.Thumbprint),
                };

                ctx.Principal = new ClaimsPrincipal(
                    new ClaimsIdentity(claims, ctx.Scheme.Name));
                ctx.Success();

                return Task.CompletedTask;
            }
        };
    });
```

---

## Step 1291: Penetration Testing Helpers

### Security Test Suite

```csharp
[Collection("Security Tests")]
public class ApiSecurityTests(WebApplicationFactory<Program> factory)
    : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client = factory.CreateClient();

    [Theory]
    [InlineData("' OR '1'='1")]
    [InlineData("'; DROP TABLE Orders;--")]
    [InlineData("' UNION SELECT * FROM users--")]
    public async Task CreateOrder_WithSqlInjectionAttempt_ReturnsBadRequest(string injection)
    {
        var request = new { CustomerId = injection, Items = new[] { new { ProductId = "p1", Quantity = 1 } } };
        var response = await _client.PostAsJsonAsync("/api/orders", request);
        response.StatusCode.Should().Be(HttpStatusCode.BadRequest);
    }

    [Theory]
    [InlineData("<script>alert('xss')</script>")]
    [InlineData("<img src=x onerror=alert(1)>")]
    [InlineData("javascript:alert(1)")]
    public async Task CreateOrder_WithXssAttempt_ReturnsBadRequest(string xss)
    {
        var request = new { CustomerId = Guid.NewGuid(), SpecialInstructions = xss };
        var response = await _client.PostAsJsonAsync("/api/orders", request);
        response.StatusCode.Should().Be(HttpStatusCode.BadRequest);
    }

    [Fact]
    public async Task GetOrder_WithoutAuth_ReturnsUnauthorized()
    {
        var response = await _client.GetAsync($"/api/orders/{Guid.NewGuid()}");
        response.StatusCode.Should().Be(HttpStatusCode.Unauthorized);
    }

    [Fact]
    public async Task GetOrder_WithDifferentUserToken_ReturnsForbidden()
    {
        // Arrange: create order for user1, try to access as user2
        var user1Token = GenerateTestToken("user1");
        var user2Token = GenerateTestToken("user2");

        _client.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", user1Token);

        var createResp = await _client.PostAsJsonAsync("/api/orders",
            new { CustomerId = "user1", Items = Array.Empty<object>() });
        var order = await createResp.Content.ReadFromJsonAsync<OrderResponse>();

        // Act: user2 tries to access user1's order
        _client.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", user2Token);

        var getResp = await _client.GetAsync($"/api/orders/{order!.OrderId}");

        // Assert
        getResp.StatusCode.Should().Be(HttpStatusCode.Forbidden);
    }

    [Fact]
    public async Task Api_HasSecurityHeaders()
    {
        var response = await _client.GetAsync("/health");

        response.Headers.Should().ContainKey("X-Content-Type-Options");
        response.Headers.Should().ContainKey("X-Frame-Options");
        response.Headers.Should().NotContainKey("Server");
    }

    private static string GenerateTestToken(string userId)
    {
        var key = new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes("test-signing-key-at-least-32-chars!!"));

        var token = new JwtSecurityToken(
            issuer: "test",
            audience: "orders-api",
            claims: [new Claim("sub", userId), new Claim("scope", "orders.read")],
            expires: DateTime.UtcNow.AddHours(1),
            signingCredentials: new SigningCredentials(key, SecurityAlgorithms.HmacSha256));

        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}
```

---

## Step 1292: Secrets Management Best Practices

### Secret Rotation

```csharp
public class SecretRotationService : BackgroundService
{
    private readonly SecretManager _secretManager;
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<SecretRotationService> _logger;

    public SecretRotationService(
        SecretManager secretManager,
        IServiceProvider serviceProvider,
        ILogger<SecretRotationService> logger)
    {
        _secretManager = secretManager;
        _serviceProvider = serviceProvider;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        // Check for expiring secrets every hour
        while (!stoppingToken.IsCancellationRequested)
        {
            await RotateExpiringSecretsAsync(stoppingToken);
            await Task.Delay(TimeSpan.FromHours(1), stoppingToken);
        }
    }

    private async Task RotateExpiringSecretsAsync(CancellationToken ct)
    {
        var secretsToRotate = await _secretManager
            .GetSecretsExpiringWithinAsync(TimeSpan.FromDays(7), ct);

        foreach (var secretName in secretsToRotate)
        {
            try
            {
                await RotateSecretAsync(secretName, ct);
                _logger.LogInformation("Rotated secret {SecretName}", secretName);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to rotate secret {SecretName}", secretName);
            }
        }
    }

    private async Task RotateSecretAsync(string secretName, CancellationToken ct)
    {
        var newValue = GenerateNewSecretValue(secretName);
        await _secretManager.SetSecretAsync(secretName, newValue);
        // Update dependent services if needed
    }

    private static string GenerateNewSecretValue(string secretName)
    {
        var bytes = new byte[32];
        RandomNumberGenerator.Fill(bytes);
        return Convert.ToBase64String(bytes);
    }
}
```

---

## Step 1293: Security Scanning and Dependency Updates

### Automated Dependency Updates with Dependabot

```yaml
# .github/dependabot.yml
version: 2
updates:
  # .NET NuGet packages
  - package-ecosystem: "nuget"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
    open-pull-requests-limit: 10
    reviewers:
      - "security-team"
    groups:
      aspnetcore:
        patterns:
          - "Microsoft.AspNetCore.*"
      entityframework:
        patterns:
          - "Microsoft.EntityFrameworkCore.*"
    ignore:
      - dependency-name: "SomeLegacyPackage"
        versions: ["2.x"]

  # Docker base images
  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "weekly"
    reviewers:
      - "devops-team"

  # GitHub Actions
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

### NuGet Security Audit

```bash
# Check for known vulnerabilities
dotnet list package --vulnerable

# Check for outdated packages
dotnet list package --outdated

# Audit all transitive dependencies
dotnet list package --vulnerable --include-transitive

# In CI/CD
dotnet restore
dotnet list package --vulnerable --include-transitive 2>&1 | \
  tee vulnerability-report.txt | \
  grep -c "has known vulnerability" && exit 1 || exit 0
```

---

## Step 1294: Security Monitoring

### Security Event Detection

```csharp
public class SecurityEventDetector
{
    private readonly ILogger<SecurityEventDetector> _logger;
    private readonly IDistributedCache _cache;
    private readonly IAlertingService _alerting;

    public SecurityEventDetector(
        ILogger<SecurityEventDetector> logger,
        IDistributedCache cache,
        IAlertingService alerting)
    {
        _logger = logger;
        _cache = cache;
        _alerting = alerting;
    }

    // Detect brute force attacks
    public async Task RecordFailedLoginAsync(
        string username,
        string ipAddress,
        CancellationToken ct = default)
    {
        // Per-user tracking
        var userKey = $"failed-login:user:{username}";
        var userCount = await IncrementCounterAsync(userKey, TimeSpan.FromMinutes(15), ct);

        // Per-IP tracking
        var ipKey = $"failed-login:ip:{ipAddress}";
        var ipCount = await IncrementCounterAsync(ipKey, TimeSpan.FromMinutes(15), ct);

        if (userCount >= 5)
        {
            _logger.LogWarning(
                "Brute force attack detected for user {Username} from {IpAddress}",
                username, ipAddress);

            await _alerting.SendAlertAsync(new Alert
            {
                Title = $"Brute force attack detected: {username}",
                Description = $"{userCount} failed logins in 15 minutes from {ipAddress}",
                Severity = "High",
            }, ct);
        }

        if (ipCount >= 20)
        {
            _logger.LogWarning(
                "Suspicious IP activity: {IpAddress} with {Count} failed logins",
                ipAddress, ipCount);

            // Auto-block IP
            await BlockIpAsync(ipAddress, TimeSpan.FromHours(1), ct);
        }
    }

    // Detect account enumeration
    public async Task RecordUserEnumerationAttemptAsync(
        string ipAddress,
        CancellationToken ct = default)
    {
        var key = $"enum-attempt:ip:{ipAddress}";
        var count = await IncrementCounterAsync(key, TimeSpan.FromMinutes(5), ct);

        if (count >= 10)
        {
            await BlockIpAsync(ipAddress, TimeSpan.FromHours(24), ct);

            _logger.LogWarning(
                "Account enumeration detected from {IpAddress}, blocking for 24h",
                ipAddress);
        }
    }

    public async Task<bool> IsIpBlockedAsync(string ipAddress, CancellationToken ct = default)
    {
        var blocked = await _cache.GetStringAsync($"blocked:ip:{ipAddress}", ct);
        return blocked is not null;
    }

    private async Task<long> IncrementCounterAsync(
        string key,
        TimeSpan expiry,
        CancellationToken ct)
    {
        var current = await _cache.GetStringAsync(key, ct);
        var count = current is null ? 1 : long.Parse(current) + 1;

        await _cache.SetStringAsync(
            key,
            count.ToString(),
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = expiry,
            },
            ct);

        return count;
    }

    private Task BlockIpAsync(string ipAddress, TimeSpan duration, CancellationToken ct)
    {
        return _cache.SetStringAsync(
            $"blocked:ip:{ipAddress}",
            DateTime.UtcNow.ToString("O"),
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = duration,
            },
            ct);
    }
}
```

---

## Step 1295: Data Privacy and GDPR

### Data Masking

```csharp
public static class DataMasking
{
    // Email: j***@example.com
    public static string MaskEmail(string email)
    {
        var parts = email.Split('@');
        if (parts.Length != 2) return "***@***.***";

        var localPart = parts[0];
        var maskedLocal = localPart.Length <= 2
            ? new string('*', localPart.Length)
            : $"{localPart[0]}{new string('*', localPart.Length - 2)}{localPart[^1]}";

        return $"{maskedLocal}@{parts[1]}";
    }

    // Phone: +66***-***-99
    public static string MaskPhone(string phone)
    {
        if (phone.Length < 4) return new string('*', phone.Length);
        return $"{phone[..2]}{new string('*', phone.Length - 4)}{phone[^2..]}";
    }

    // Credit card: **** **** **** 1234
    public static string MaskCreditCard(string number)
    {
        var digits = number.Replace(" ", "").Replace("-", "");
        return $"**** **** **** {digits[^4..]}";
    }

    // National ID: keep first 2 and last 2
    public static string MaskNationalId(string id)
    {
        if (id.Length < 5) return new string('*', id.Length);
        return $"{id[..2]}{new string('*', id.Length - 4)}{id[^2..]}";
    }
}

// GDPR: Data export
public class GdprDataExportService
{
    private readonly AppDbContext _context;

    public GdprDataExportService(AppDbContext context)
    {
        _context = context;
    }

    public async Task<UserDataExport> ExportUserDataAsync(Guid userId)
    {
        var user = await _context.Users.FindAsync(userId);
        var orders = await _context.Orders
            .Where(o => o.CustomerId == userId)
            .ToListAsync();

        return new UserDataExport
        {
            ExportedAt = DateTime.UtcNow,
            UserId = userId,
            Profile = user is not null ? new UserProfile
            {
                Email = user.Email,
                Name = user.FullName,
                CreatedAt = user.CreatedAt,
            } : null,
            Orders = orders.Select(o => new OrderSummary
            {
                OrderId = o.Id,
                Status = o.Status.ToString(),
                TotalAmount = o.TotalAmount.Amount,
                CreatedAt = o.CreatedAt,
            }).ToList(),
        };
    }

    public async Task DeleteUserDataAsync(Guid userId)
    {
        // Pseudonymize instead of delete (for audit trail)
        var user = await _context.Users.FindAsync(userId);
        if (user is null) return;

        user.Email = $"deleted_{userId}@deleted.invalid";
        user.FullName = "[Deleted User]";
        user.IsDeleted = true;
        user.DeletedAt = DateTime.UtcNow;

        await _context.SaveChangesAsync();
    }
}

public class UserDataExport
{
    public required DateTime ExportedAt { get; init; }
    public required Guid UserId { get; init; }
    public UserProfile? Profile { get; init; }
    public required List<OrderSummary> Orders { get; init; }
}

public class UserProfile
{
    public required string Email { get; init; }
    public required string Name { get; init; }
    public required DateTime CreatedAt { get; init; }
}

public class OrderSummary
{
    public required Guid OrderId { get; init; }
    public required string Status { get; init; }
    public required decimal TotalAmount { get; init; }
    public required DateTime CreatedAt { get; init; }
}
```

---

## Step 1296: Secure Configuration Management

### appsettings Security Checklist

```json
{
  "ConnectionStrings": {
    "Default": "${DB_CONNECTION_STRING}"
  },
  "JWT": {
    "Secret": "${JWT_SECRET}",
    "Issuer": "https://auth.myapp.com",
    "Audience": "myapp-api",
    "ExpirationMinutes": 60
  },
  "Security": {
    "AllowedOrigins": ["https://myapp.com"],
    "ContentSecurityPolicy": "default-src 'self'",
    "RequireHttps": true,
    "HstsMaxAgeSeconds": 31536000
  }
}
```

### Secure Defaults in Program.cs

```csharp
// Secure defaults
if (app.Environment.IsProduction())
{
    // Force HTTPS
    app.UseHttpsRedirection();
    app.UseHsts();
}

// Remove sensitive headers
app.Use(async (ctx, next) =>
{
    ctx.Response.OnStarting(() =>
    {
        ctx.Response.Headers.Remove("Server");
        ctx.Response.Headers.Remove("X-Powered-By");
        ctx.Response.Headers.Remove("X-SourceFiles");
        return Task.CompletedTask;
    });
    await next();
});

// Security headers
app.UseMiddleware<SecurityHeadersMiddleware>();

// Block Tor/VPN/proxy (if needed)
// app.UseMiddleware<IpReputationMiddleware>();

// Authentication & Authorization
app.UseAuthentication();
app.UseAuthorization();

// Audit logging
app.UseMiddleware<AuditMiddleware>();
```

---

## Step 1297: Security Testing with OWASP ZAP

### Automated Security Scan in CI

```yaml
# .github/workflows/security-scan.yml
name: OWASP Security Scan

on:
  schedule:
    - cron: '0 2 * * 0'  # Weekly Sunday at 2 AM

jobs:
  zap-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Start application
        run: |
          docker compose up -d api
          sleep 30  # wait for startup

      - name: Run OWASP ZAP API Scan
        uses: zaproxy/action-api-scan@v0.7.0
        with:
          target: 'http://localhost:8080/openapi.json'
          format: openapi
          fail_action: false
          cmd_options: '-a -j -l WARN'

      - name: Upload ZAP Report
        uses: actions/upload-artifact@v4
        with:
          name: zap-report
          path: report_html.html

      - name: Check for critical vulnerabilities
        run: |
          if grep -q "HIGH\|CRITICAL" zap_scan.json; then
            echo "Critical vulnerabilities found!"
            cat zap_scan.json
            exit 1
          fi
```

---

## Step 1298: Complete Security Middleware Pipeline

### Ordered Security Middleware

```csharp
// Program.cs - Complete security middleware pipeline
var app = builder.Build();

// 1. Exception handling (must be first)
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
}
else
{
    app.UseExceptionHandler("/error");
    app.UseHsts();
}

// 2. HTTPS redirection
app.UseHttpsRedirection();

// 3. Security headers
app.UseMiddleware<SecurityHeadersMiddleware>();

// 4. Request size limits
app.Use(async (context, next) =>
{
    context.Request.EnableBuffering();

    var contentLength = context.Request.ContentLength;
    if (contentLength > 10 * 1024 * 1024) // 10 MB
    {
        context.Response.StatusCode = 413;
        await context.Response.WriteAsJsonAsync(new { error = "Payload too large" });
        return;
    }

    await next();
});

// 5. Rate limiting
app.UseRateLimiter();

// 6. IP blocking check
app.UseMiddleware<IpBlockingMiddleware>();

// 7. CORS
app.UseCors("production-policy");

// 8. Authentication
app.UseAuthentication();

// 9. Authorization
app.UseAuthorization();

// 10. Audit logging
app.UseMiddleware<AuditMiddleware>();

// 11. Anti-CSRF (for form-based flows)
// app.UseAntiforgery();

// Map endpoints
app.MapControllers();
```

---

## Step 1299: Vulnerability Management

### CVE Scanning and Response

```csharp
// Automated vulnerability notification service
public class VulnerabilityNotificationService : BackgroundService
{
    private readonly HttpClient _nvdClient;  // NVD API
    private readonly IAlertingService _alerting;
    private readonly ILogger<VulnerabilityNotificationService> _logger;

    public VulnerabilityNotificationService(
        IHttpClientFactory factory,
        IAlertingService alerting,
        ILogger<VulnerabilityNotificationService> logger)
    {
        _nvdClient = factory.CreateClient("nvd");
        _nvdClient.BaseAddress = new Uri("https://services.nvd.nist.gov/rest/json/cves/2.0");
        _alerting = alerting;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await CheckForNewVulnerabilitiesAsync(stoppingToken);
            await Task.Delay(TimeSpan.FromDays(1), stoppingToken);
        }
    }

    private async Task CheckForNewVulnerabilitiesAsync(CancellationToken ct)
    {
        // Check for .NET and ASP.NET Core CVEs
        var keywords = new[] { "dotnet", "aspnetcore", "nuget" };

        foreach (var keyword in keywords)
        {
            try
            {
                var response = await _nvdClient.GetFromJsonAsync<NvdResponse>(
                    $"?keywordSearch={keyword}&lastModStartDate={DateTime.UtcNow.AddDays(-7):O}" +
                    $"&cvssV3Severity=HIGH",
                    ct);

                if (response?.Vulnerabilities?.Any() == true)
                {
                    foreach (var vuln in response.Vulnerabilities)
                    {
                        await _alerting.SendAlertAsync(new Alert
                        {
                            Title = $"New vulnerability affecting .NET: {vuln.Cve.Id}",
                            Description = vuln.Cve.Descriptions
                                .FirstOrDefault(d => d.Lang == "en")?.Value ?? "No description",
                            Severity = "High",
                        }, ct);
                    }
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to check NVD for {Keyword}", keyword);
            }
        }
    }
}

public record NvdResponse(IReadOnlyList<NvdVulnerability>? Vulnerabilities);
public record NvdVulnerability(NvdCve Cve);
public record NvdCve(string Id, IReadOnlyList<CveDescription> Descriptions);
public record CveDescription(string Lang, string Value);
```

---

## Step 1300: Summary - Security & Zero Trust

### ภาพรวมที่เรียนในบทนี้

```
Advanced Security & Zero Trust
│
├── Authentication
│   ├── OAuth 2.0 / OIDC (Duende Identity Server)
│   ├── JWT validation
│   ├── API Key management
│   └── mTLS (service-to-service)
│
├── Authorization
│   ├── JWT Bearer policies
│   ├── RBAC (Role-Based)
│   ├── ABAC (Attribute-Based)
│   └── Resource ownership checks
│
├── Cryptography
│   ├── AES-GCM encryption
│   ├── RSA for key wrapping
│   ├── Envelope encryption (DEK/KEK)
│   └── Data Protection API
│
├── Input Security
│   ├── FluentValidation
│   ├── SQL injection prevention
│   ├── XSS prevention (HTML sanitization)
│   └── Content Security Policy
│
├── API Security
│   ├── Security headers middleware
│   ├── CSRF protection
│   ├── Rate limiting (per user/IP)
│   └── Request size limits
│
├── Audit & Compliance
│   ├── Structured audit logs
│   ├── GDPR data export/deletion
│   └── Data masking/pseudonymization
│
├── Security Testing
│   ├── Unit tests for auth/authz
│   ├── SQL injection test cases
│   └── OWASP ZAP scanning
│
└── Monitoring
    ├── Brute force detection
    ├── Account enumeration detection
    ├── IP blocking
    └── CVE monitoring
```

### OWASP Top 10 Coverage

| Risk | Mitigation in this Part |
|------|------------------------|
| A01: Broken Access Control | RBAC, ABAC, resource ownership |
| A02: Cryptographic Failures | AES-GCM, envelope encryption |
| A03: Injection | Parameterized queries, input validation |
| A04: Insecure Design | Zero Trust architecture |
| A05: Security Misconfiguration | Security headers, defaults |
| A06: Vulnerable Components | Dependabot, `dotnet list package --vulnerable` |
| A07: Auth Failures | JWT, rate limiting, brute force detection |
| A08: Integrity Failures | Code signing, SBOM, Trivy |
| A09: Logging Failures | Audit logs, SIEM integration |
| A10: SSRF | URL validation, allow-list |

ในบทถัดไปจะเรียนเรื่อง **Microservices Patterns & Service Communication (Steps 1301-1340)** ซึ่งครอบคลุม API Gateway, BFF pattern, gRPC, GraphQL, Service Mesh และ microservice design patterns!
