# Part 83: Security Hardening & Secure Coding Practices

## Steps 1957–1972

---

## Step 1957: Security Principles

Security is not a feature — it's a cross-cutting concern that must be built in from the start.

```
OWASP Top 10 (2021) in .NET context:
1. Broken Access Control      → Authorization policies, resource-based auth
2. Cryptographic Failures     → HTTPS, AES-256, BCrypt, no MD5/SHA1 for passwords
3. Injection                  → Parameterized queries, no string SQL, no eval
4. Insecure Design            → Threat modeling, defense in depth
5. Security Misconfiguration  → Remove defaults, least privilege, secure headers
6. Vulnerable Components      → NuGet audit, dependabot
7. Auth & Session Failures    → JWT, secure cookies, PKCE
8. Software Integrity         → Signed packages, SBOM, CI/CD security
9. Logging Failures           → Audit logging, no PII in logs
10. SSRF                      → Validate URLs, allowlist outbound hosts
```

---

## Step 1958: SQL Injection Prevention

```csharp
// VULNERABLE — never do this
var userId = Request.Query["id"];
var sql = $"SELECT * FROM Users WHERE Id = {userId}"; // SQL injection!
await _db.Database.ExecuteSqlRawAsync(sql);

// SAFE — parameterized query
var id = Guid.Parse(Request.Query["id"]);
var user = await _db.Users
    .Where(u => u.Id == id) // EF Core generates parameterized SQL
    .FirstOrDefaultAsync(ct);

// SAFE — raw SQL with parameters
var users = await _db.Users
    .FromSqlInterpolated($"SELECT * FROM \"Users\" WHERE \"Email\" = {email}")
    .ToListAsync(ct);

// SAFE — FormattableString parameters (EF Core uses SqlParameter internally)
await _db.Database.ExecuteSqlInterpolatedAsync(
    $"UPDATE \"Products\" SET \"Stock\" = {newStock} WHERE \"Id\" = {productId}", ct);
```

---

## Step 1959: XSS Prevention

```csharp
// API responses — JSON encoding handles this automatically
// No manual escaping needed when returning JSON

// Blazor — automatic escaping
<div>@user.Name</div>  // Safe: Blazor escapes this

// Razor Pages — @Html.Raw is DANGEROUS
@Html.Raw(userInput)  // DO NOT USE with user input

// Safe
@userInput  // Auto-escaped

// Content Security Policy header
app.Use(async (context, next) =>
{
    context.Response.Headers.Add("Content-Security-Policy",
        "default-src 'self'; " +
        "script-src 'self' 'nonce-{nonce}'; " +
        "style-src 'self' https://fonts.googleapis.com; " +
        "img-src 'self' data: https:; " +
        "font-src 'self' https://fonts.gstatic.com; " +
        "connect-src 'self' wss:; " +
        "frame-ancestors 'none'; " +
        "base-uri 'self'; " +
        "form-action 'self'");
    await next();
});
```

---

## Step 1960: Security Headers Middleware

```csharp
// Middleware/SecurityHeadersMiddleware.cs
namespace CleanApp.Api.Middleware;

public class SecurityHeadersMiddleware
{
    private readonly RequestDelegate _next;

    public SecurityHeadersMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var response = context.Response;

        // Prevent clickjacking
        response.Headers["X-Frame-Options"] = "DENY";

        // Prevent MIME type sniffing
        response.Headers["X-Content-Type-Options"] = "nosniff";

        // Referrer policy
        response.Headers["Referrer-Policy"] = "strict-origin-when-cross-origin";

        // Permissions policy — disable unused browser features
        response.Headers["Permissions-Policy"] =
            "camera=(), microphone=(), geolocation=(), payment=()";

        // HTTPS only (set by HTTPS middleware, but be explicit)
        response.Headers["Strict-Transport-Security"] =
            "max-age=31536000; includeSubDomains; preload";

        // Remove server header
        response.Headers.Remove("Server");
        response.Headers.Remove("X-Powered-By");

        await _next(context);
    }
}
```

```csharp
// Program.cs
app.UseMiddleware<SecurityHeadersMiddleware>();
app.UseHsts(); // Strict-Transport-Security
app.UseHttpsRedirection();
```

---

## Step 1961: Password Hashing

```csharp
// WRONG — MD5, SHA1, SHA256 are not password hashing algorithms
var hash = Convert.ToHexString(MD5.HashData(Encoding.UTF8.GetBytes(password))); // NEVER

// WRONG — SHA256 without salt or iterations
var hash = Convert.ToHexString(SHA256.HashData(Encoding.UTF8.GetBytes(password)));

// RIGHT — BCrypt (recommended)
using BCrypt.Net;
var hash = BCrypt.HashPassword(password, workFactor: 12); // ~250ms at factor 12
var isValid = BCrypt.Verify(password, hash);

// RIGHT — ASP.NET Core built-in (PBKDF2)
using Microsoft.AspNetCore.Identity;
var hasher = new PasswordHasher<ApplicationUser>();
var hashed = hasher.HashPassword(user, plainPassword);
var result = hasher.VerifyHashedPassword(user, hashed, inputPassword);
// result: PasswordVerificationResult.Success / Failed / SuccessRehashNeeded
```

```csharp
// Argon2id — current best practice
using Konscious.Security.Cryptography;

public static class Argon2Hasher
{
    private const int MemorySize = 65536; // 64 MB
    private const int Iterations = 3;
    private const int Parallelism = 4;
    private const int HashLength = 32;

    public static string HashPassword(string password)
    {
        var salt = RandomNumberGenerator.GetBytes(16);
        var hash = Hash(password, salt);

        return $"argon2id${Convert.ToBase64String(salt)}${Convert.ToBase64String(hash)}";
    }

    public static bool VerifyPassword(string password, string storedHash)
    {
        var parts = storedHash.Split('$');
        if (parts.Length != 3 || parts[0] != "argon2id")
            return false;

        var salt = Convert.FromBase64String(parts[1]);
        var expectedHash = Convert.FromBase64String(parts[2]);
        var actualHash = Hash(password, salt);

        return CryptographicOperations.FixedTimeEquals(actualHash, expectedHash);
    }

    private static byte[] Hash(string password, byte[] salt)
    {
        using var argon2 = new Argon2id(Encoding.UTF8.GetBytes(password))
        {
            Salt = salt,
            MemorySize = MemorySize,
            Iterations = Iterations,
            DegreeOfParallelism = Parallelism
        };

        return argon2.GetBytes(HashLength);
    }
}
```

---

## Step 1962: JWT Security Best Practices

```csharp
// Program.cs — secure JWT configuration
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(opts =>
    {
        opts.TokenValidationParameters = new TokenValidationParameters
        {
            // MUST validate all of these:
            ValidateIssuer = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],

            ValidateAudience = true,
            ValidAudience = builder.Configuration["Jwt:Audience"],

            ValidateLifetime = true,
            ClockSkew = TimeSpan.FromMinutes(1), // reduce from default 5 min

            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(
                Convert.FromBase64String(builder.Configuration["Jwt:SecretBase64"]!)),

            // Prevent algorithm confusion attacks
            ValidAlgorithms = [SecurityAlgorithms.HmacSha256],

            NameClaimType = JwtRegisteredClaimNames.Sub,
            RoleClaimType = "roles",

            // Require exp claim
            RequireExpirationTime = true
        };

        opts.MapInboundClaims = false; // don't remap claim names
    });
```

```csharp
// Token generation — secure settings
public class JwtTokenService
{
    private readonly IConfiguration _config;
    private readonly IDateTime _dateTime;

    public string GenerateAccessToken(ApplicationUser user, IList<string> roles)
    {
        var keyBytes = Convert.FromBase64String(_config["Jwt:SecretBase64"]!);
        var key = new SymmetricSecurityKey(keyBytes);
        var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

        var claims = new List<Claim>
        {
            new(JwtRegisteredClaimNames.Sub, user.Id),
            new(JwtRegisteredClaimNames.Email, user.Email!),
            new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()), // unique token ID
        };

        foreach (var role in roles)
            claims.Add(new Claim("roles", role));

        var now = _dateTime.UtcNow;
        var token = new JwtSecurityToken(
            issuer: _config["Jwt:Issuer"],
            audience: _config["Jwt:Audience"],
            claims: claims,
            notBefore: now.UtcDateTime,
            expires: now.AddMinutes(15).UtcDateTime, // short-lived: 15 minutes
            signingCredentials: creds);

        return new JwtSecurityTokenHandler().WriteToken(token);
    }

    public string GenerateRefreshToken()
    {
        return Convert.ToBase64String(RandomNumberGenerator.GetBytes(64));
    }
}
```

---

## Step 1963: Authorization Policies

```csharp
// Resource-based authorization — check ownership
public class OrderAuthorizationHandler
    : AuthorizationHandler<OrderOperationRequirement, Order>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        OrderOperationRequirement requirement,
        Order resource)
    {
        var userId = context.User.FindFirstValue(JwtRegisteredClaimNames.Sub);

        // Admins can do anything
        if (context.User.IsInRole("Admin"))
        {
            context.Succeed(requirement);
            return Task.CompletedTask;
        }

        // Users can only access their own orders
        if (requirement == OrderOperations.Read ||
            requirement == OrderOperations.Cancel)
        {
            if (resource.CustomerId.ToString() == userId)
                context.Succeed(requirement);
        }

        return Task.CompletedTask;
    }
}

public static class OrderOperations
{
    public static readonly OrderOperationRequirement Read = new("Read");
    public static readonly OrderOperationRequirement Cancel = new("Cancel");
    public static readonly OrderOperationRequirement Refund = new("Refund");
}

public record OrderOperationRequirement(string Name) : IAuthorizationRequirement;
```

```csharp
// Registration
builder.Services.AddAuthorization(opts =>
{
    opts.AddPolicy("RequireAdmin", policy =>
        policy.RequireRole("Admin"));

    opts.AddPolicy("RequireActiveUser", policy =>
        policy.RequireAuthenticatedUser()
              .RequireClaim("account_status", "active"));

    opts.FallbackPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build(); // every endpoint requires auth by default
});

builder.Services.AddSingleton<IAuthorizationHandler, OrderAuthorizationHandler>();
```

```csharp
// Usage in controller
[HttpDelete("{id:guid}")]
public async Task<IActionResult> CancelOrder(Guid id, CancellationToken ct)
{
    var order = await _orders.GetByIdAsync(id, ct);
    if (order is null) return NotFound();

    var authResult = await _authService.AuthorizeAsync(
        User, order, OrderOperations.Cancel);

    if (!authResult.Succeeded)
        return Forbid();

    order.Cancel("User requested cancellation");
    await _uow.CommitAsync(ct);
    return NoContent();
}
```

---

## Step 1964: Input Validation & Sanitization

```csharp
// FluentValidation — comprehensive validation
public class CreateProductValidator : AbstractValidator<CreateProductCommand>
{
    public CreateProductValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty()
            .MaximumLength(200)
            .Matches(@"^[\w\s\-\.,'""()&+]+$") // whitelist characters
            .WithMessage("Product name contains invalid characters");

        RuleFor(x => x.Price)
            .GreaterThan(0)
            .LessThanOrEqualTo(999_999.99m)
            .PrecisionScale(8, 2, false);

        RuleFor(x => x.Sku)
            .NotEmpty()
            .Matches(@"^[A-Z0-9\-]{3,20}$")
            .WithMessage("SKU must be 3-20 uppercase alphanumeric characters with hyphens");

        RuleFor(x => x.Description)
            .MaximumLength(5000)
            .When(x => x.Description is not null);

        RuleFor(x => x.ImageUrl)
            .Must(BeAValidHttpsUrl).When(x => x.ImageUrl is not null)
            .WithMessage("Image URL must be a valid HTTPS URL");
    }

    private static bool BeAValidHttpsUrl(string? url)
    {
        if (url is null) return true;
        return Uri.TryCreate(url, UriKind.Absolute, out var uri) &&
               uri.Scheme == Uri.UriSchemeHttps;
    }
}
```

```csharp
// HTML sanitization for rich text fields
using Ganss.Xss;

public static class HtmlSanitizer
{
    private static readonly HtmlSanitizer Sanitizer = new();

    static HtmlSanitizer()
    {
        // Whitelist only safe HTML elements
        Sanitizer.AllowedTags.Clear();
        Sanitizer.AllowedTags.Add("p");
        Sanitizer.AllowedTags.Add("br");
        Sanitizer.AllowedTags.Add("strong");
        Sanitizer.AllowedTags.Add("em");
        Sanitizer.AllowedTags.Add("ul");
        Sanitizer.AllowedTags.Add("ol");
        Sanitizer.AllowedTags.Add("li");

        Sanitizer.AllowedAttributes.Clear();
        Sanitizer.AllowedSchemes.Clear();
        Sanitizer.AllowedSchemes.Add("https"); // no javascript: or data: URIs
    }

    public static string Sanitize(string? html) =>
        string.IsNullOrEmpty(html) ? string.Empty : Sanitizer.Sanitize(html);
}
```

---

## Step 1965: Secrets Management

```csharp
// WRONG — secrets in appsettings.json / source control
{
  "ConnectionStrings": {
    "Postgres": "Host=prod-db;Password=SuperSecret123!"
  }
}

// RIGHT — environment variables (twelve-factor app)
{
  "ConnectionStrings": {
    "Postgres": "" // empty — loaded from env
  }
}
// Set: ConnectionStrings__Postgres = "Host=prod-db;..."

// RIGHT — Azure Key Vault
builder.Configuration.AddAzureKeyVault(
    new Uri($"https://{vaultName}.vault.azure.net/"),
    new DefaultAzureCredential());

// RIGHT — AWS Secrets Manager
builder.Configuration.AddSecretsManager(region: RegionEndpoint.USEast1, opts =>
{
    opts.SecretFilter = entry => entry.Name.StartsWith("myapp/");
    opts.KeyGenerator = (entry, key) => key
        .Replace("myapp/", "")
        .Replace("__", ":");
    opts.PollingInterval = TimeSpan.FromHours(1);
});
```

```csharp
// User Secrets — development only (never committed to Git)
// dotnet user-secrets set "Jwt:SecretBase64" "base64-encoded-secret"

// .gitignore must include:
// appsettings.*.json (environment-specific)
// .env files
// secrets.json
```

---

## Step 1966: CSRF Protection

```csharp
// API-only apps: use JWT in Authorization header — immune to CSRF
// Cookie-based apps: require anti-forgery tokens

// For Minimal APIs
builder.Services.AddAntiforgery(opts =>
{
    opts.HeaderName = "X-XSRF-TOKEN";
    opts.Cookie.HttpOnly = false; // JS must be able to read this cookie
    opts.Cookie.SameSite = SameSiteMode.Strict;
    opts.Cookie.SecurePolicy = CookieSecurePolicy.Always;
});

app.UseAntiforgery();

// Endpoint that generates token
app.MapGet("/api/antiforgery/token", (IAntiforgery antiforgery, HttpContext ctx) =>
{
    var tokens = antiforgery.GetAndStoreTokens(ctx);
    return Results.Ok(new { token = tokens.RequestToken });
});

// Protected endpoint
app.MapPost("/api/sensitive-action", [ValidateAntiForgeryToken] async (
    SensitiveRequest request, ...) => { ... });
```

---

## Step 1967: Secure Cookie Configuration

```csharp
builder.Services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
    .AddCookie(opts =>
    {
        opts.Cookie.HttpOnly = true;           // JS cannot read
        opts.Cookie.Secure = true;             // HTTPS only
        opts.Cookie.SameSite = SameSiteMode.Strict; // no cross-site sending
        opts.Cookie.Name = "__Host-session";   // __Host- prefix: enforces Secure + no domain
        opts.ExpireTimeSpan = TimeSpan.FromHours(2);
        opts.SlidingExpiration = true;

        // Custom paths
        opts.LoginPath = "/auth/login";
        opts.AccessDeniedPath = "/auth/forbidden";

        // Prevent session fixation
        opts.SessionStore = null; // use cookie, not server-side session
    });

// Session cookie — additional hardening
builder.Services.AddSession(opts =>
{
    opts.Cookie.HttpOnly = true;
    opts.Cookie.Secure = true;
    opts.Cookie.SameSite = SameSiteMode.Strict;
    opts.IdleTimeout = TimeSpan.FromMinutes(30);
});
```

---

## Step 1968: Rate Limiting & Brute-Force Protection

```csharp
// Auth endpoints need aggressive rate limiting
builder.Services.AddRateLimiter(opts =>
{
    // Login — 5 attempts per IP per minute
    opts.AddFixedWindowLimiter("login", o =>
    {
        o.Window = TimeSpan.FromMinutes(1);
        o.PermitLimit = 5;
        o.QueueLimit = 0;
    });

    // Password reset — 3 per hour per IP
    opts.AddFixedWindowLimiter("password-reset", o =>
    {
        o.Window = TimeSpan.FromHours(1);
        o.PermitLimit = 3;
        o.QueueLimit = 0;
    });

    opts.RejectionStatusCode = 429;
    opts.OnRejected = async (ctx, ct) =>
    {
        var ip = ctx.HttpContext.Connection.RemoteIpAddress?.ToString() ?? "unknown";
        // Log the blocked attempt
        ctx.HttpContext.RequestServices
            .GetRequiredService<ILogger<Program>>()
            .LogWarning("Rate limit exceeded for IP {IP}", ip);

        ctx.HttpContext.Response.Headers.RetryAfter = "60";
        await ctx.HttpContext.Response.WriteAsJsonAsync(
            new { error = "too_many_requests" }, ct);
    };
});

app.MapPost("/api/auth/login", LoginHandler)
    .RequireRateLimiting("login")
    .AllowAnonymous();
```

```csharp
// Account lockout (ASP.NET Core Identity)
builder.Services.AddIdentityCore<ApplicationUser>(opts =>
{
    opts.Lockout.MaxFailedAccessAttempts = 5;
    opts.Lockout.DefaultLockoutTimeSpan = TimeSpan.FromMinutes(15);
    opts.Lockout.AllowedForNewUsers = true;

    opts.Password.RequireDigit = true;
    opts.Password.RequiredLength = 12;
    opts.Password.RequireNonAlphanumeric = true;
    opts.Password.RequireUppercase = true;
});
```

---

## Step 1969: Audit Logging

```csharp
// Domain/Interfaces/IAuditLog.cs
public interface IAuditLogger
{
    Task LogAsync(AuditEntry entry, CancellationToken ct = default);
}

public record AuditEntry(
    string Action,
    string ResourceType,
    string? ResourceId,
    string UserId,
    string? UserEmail,
    string IpAddress,
    bool Success,
    string? FailureReason = null,
    DateTimeOffset? OccurredAt = null);
```

```csharp
// Infrastructure/Logging/DatabaseAuditLogger.cs
public class DatabaseAuditLogger : IAuditLogger
{
    private readonly AuditDbContext _db;

    public async Task LogAsync(AuditEntry entry, CancellationToken ct = default)
    {
        // Ensure no PII leaks into audit logs (sanitize as needed)
        _db.AuditLogs.Add(new AuditLogRecord
        {
            Id = Guid.NewGuid(),
            Action = entry.Action,
            ResourceType = entry.ResourceType,
            ResourceId = entry.ResourceId,
            UserId = entry.UserId,
            // Hash email for privacy — still queryable but not plaintext
            UserEmailHash = entry.UserEmail is null
                ? null
                : Convert.ToHexString(SHA256.HashData(
                    Encoding.UTF8.GetBytes(entry.UserEmail.ToLower()))),
            IpAddress = entry.IpAddress,
            Success = entry.Success,
            FailureReason = entry.FailureReason,
            OccurredAt = entry.OccurredAt ?? DateTimeOffset.UtcNow
        });

        await _db.SaveChangesAsync(ct);
    }
}
```

```csharp
// MediatR behavior — audit all commands automatically
public class AuditBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : ICommand
{
    private readonly IAuditLogger _audit;
    private readonly ICurrentUser _user;
    private readonly IHttpContextAccessor _http;

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken ct)
    {
        var commandName = typeof(TRequest).Name;
        var ipAddress = _http.HttpContext?.Connection.RemoteIpAddress?.ToString() ?? "unknown";

        try
        {
            var response = await next();

            await _audit.LogAsync(new AuditEntry(
                Action: commandName,
                ResourceType: ExtractResourceType(commandName),
                ResourceId: ExtractResourceId(request),
                UserId: _user.UserId ?? "anonymous",
                UserEmail: _user.Email,
                IpAddress: ipAddress,
                Success: true), ct);

            return response;
        }
        catch (Exception ex)
        {
            await _audit.LogAsync(new AuditEntry(
                Action: commandName,
                ResourceType: ExtractResourceType(commandName),
                ResourceId: ExtractResourceId(request),
                UserId: _user.UserId ?? "anonymous",
                UserEmail: _user.Email,
                IpAddress: ipAddress,
                Success: false,
                FailureReason: ex.GetType().Name), ct);

            throw;
        }
    }

    private static string ExtractResourceType(string commandName) =>
        commandName.Replace("Command", "").Replace("Query", "");

    private static string? ExtractResourceId(TRequest request) =>
        request.GetType().GetProperty("Id")?.GetValue(request)?.ToString();
}
```

---

## Step 1970: SSRF Prevention

```csharp
// Server-Side Request Forgery — attacker makes server fetch internal resources

// VULNERABLE
app.MapPost("/api/preview-url", async (UrlPreviewRequest req) =>
{
    var html = await httpClient.GetStringAsync(req.Url); // SSRF!
    return Results.Ok(ParseTitle(html));
});

// SAFE
public class SafeHttpClient
{
    private readonly HttpClient _http;
    private static readonly HashSet<string> AllowedHosts = new(StringComparer.OrdinalIgnoreCase)
    {
        "api.example.com",
        "cdn.example.com"
    };

    public async Task<string> FetchAllowedAsync(string url, CancellationToken ct = default)
    {
        if (!Uri.TryCreate(url, UriKind.Absolute, out var uri))
            throw new ArgumentException("Invalid URL");

        // HTTPS only
        if (uri.Scheme != Uri.UriSchemeHttps)
            throw new SecurityException("Only HTTPS URLs are allowed");

        // Allowlist hosts
        if (!AllowedHosts.Contains(uri.Host))
            throw new SecurityException($"Host '{uri.Host}' is not allowed");

        // Block private/loopback addresses
        var addresses = await Dns.GetHostAddressesAsync(uri.Host, ct);
        foreach (var address in addresses)
        {
            if (IPAddress.IsLoopback(address) || IsPrivateNetwork(address))
                throw new SecurityException("Internal network addresses are not allowed");
        }

        return await _http.GetStringAsync(url, ct);
    }

    private static bool IsPrivateNetwork(IPAddress address)
    {
        var bytes = address.GetAddressBytes();
        return bytes switch
        {
            [10, ..] => true,                           // 10.0.0.0/8
            [172, >= 16 and <= 31, ..] => true,         // 172.16.0.0/12
            [192, 168, ..] => true,                     // 192.168.0.0/16
            [169, 254, ..] => true,                     // 169.254.0.0/16 (link-local)
            _ => false
        };
    }
}
```

---

## Step 1971: Dependency Scanning

```xml
<!-- Directory.Build.props — enable NuGet audit -->
<Project>
  <PropertyGroup>
    <NuGetAudit>true</NuGetAudit>
    <NuGetAuditMode>all</NuGetAuditMode>
    <NuGetAuditLevel>low</NuGetAuditLevel>  <!-- fail on low+ severity -->
  </PropertyGroup>
</Project>
```

```yaml
# .github/workflows/security.yml
name: Security Scan
on:
  push:
    branches: [main]
  schedule:
    - cron: '0 6 * * 1'  # Monday 6am UTC

jobs:
  dotnet-audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '9.x'

      - name: Restore
        run: dotnet restore

      - name: Audit packages
        run: dotnet list package --vulnerable --include-transitive

      - name: OWASP Dependency Check
        uses: dependency-check/Dependency-Check_Action@main
        with:
          project: 'MyApp'
          path: '.'
          format: 'HTML'
          args: --failOnCVSS 7

  codeql:
    runs-on: ubuntu-latest
    permissions:
      security-events: write
    steps:
      - uses: actions/checkout@v4
      - uses: github/codeql-action/init@v3
        with:
          languages: csharp
      - uses: github/codeql-action/autobuild@v3
      - uses: github/codeql-action/analyze@v3
```

---

## Step 1972: Security Testing Checklist

```csharp
// Security test examples
public class SecurityTests : IClassFixture<WebApplicationFactory<Program>>
{
    [Fact]
    public async Task UnauthenticatedRequest_Returns401()
    {
        var client = _factory.CreateClient();
        var response = await client.GetAsync("/api/orders");
        response.StatusCode.Should().Be(HttpStatusCode.Unauthorized);
    }

    [Fact]
    public async Task AccessOtherUsersOrder_Returns403()
    {
        var user1Token = await LoginAsync("user1@test.com");
        var user2Order = await CreateOrderAsUserAsync("user2@test.com");

        var client = CreateClientWithToken(user1Token);
        var response = await client.GetAsync($"/api/orders/{user2Order.Id}");

        response.StatusCode.Should().Be(HttpStatusCode.Forbidden);
    }

    [Theory]
    [InlineData("' OR '1'='1")]
    [InlineData("; DROP TABLE Users;--")]
    [InlineData("<script>alert(1)</script>")]
    public async Task InjectionPayloads_AreRejectedOrSanitized(string maliciousInput)
    {
        var client = CreateAuthenticatedClient();
        var response = await client.PostAsJsonAsync("/api/products", new
        {
            Name = maliciousInput,
            Price = 9.99
        });

        // Should either be validated out (400) or sanitized — never stored/reflected
        response.StatusCode.Should().BeOneOf(
            HttpStatusCode.BadRequest,
            HttpStatusCode.UnprocessableEntity);
    }

    [Fact]
    public async Task SecurityHeaders_ArePresent()
    {
        var client = _factory.CreateClient();
        var response = await client.GetAsync("/api/health");

        response.Headers.Should().ContainKey("X-Content-Type-Options");
        response.Headers.Should().ContainKey("X-Frame-Options");
        response.Headers.GetValues("X-Content-Type-Options").First()
            .Should().Be("nosniff");
    }

    [Fact]
    public async Task LoginRateLimit_BlocksExcessiveAttempts()
    {
        var client = _factory.CreateClient();
        for (var i = 0; i < 5; i++)
        {
            await client.PostAsJsonAsync("/api/auth/login", new
            {
                Email = "test@test.com",
                Password = "wrong"
            });
        }

        // 6th attempt should be rate-limited
        var response = await client.PostAsJsonAsync("/api/auth/login", new
        {
            Email = "test@test.com",
            Password = "wrong"
        });

        response.StatusCode.Should().Be(HttpStatusCode.TooManyRequests);
    }
}
```

---

## Summary

| Threat | Defense | Implementation |
|--------|---------|----------------|
| SQL Injection | Parameterized queries | EF Core LINQ, `FromSqlInterpolated` |
| XSS | CSP headers, output encoding | `@` in Razor, CSP middleware |
| CSRF | SameSite cookies, anti-forgery tokens | `IAntiforgery`, `SameSiteMode.Strict` |
| Broken Auth | Short JWT TTL, refresh rotation | 15min access token, 7-day refresh |
| Broken Access | Resource-based auth | `IAuthorizationHandler<TResource>` |
| Password storage | Argon2id / BCrypt | `workFactor: 12`, FixedTimeEquals |
| Secrets | Key Vault / env vars | `AddAzureKeyVault`, never in code |
| Rate limiting | IP + user partitioned | `PartitionedRateLimiter` |
| SSRF | Allowlist outbound | `IsPrivateNetwork`, `AllowedHosts` |
| Dependencies | NuGet audit + CodeQL | `NuGetAudit=true`, GitHub Actions |
| Audit trail | Immutable log per command | `AuditBehavior<T>` |

**Next**: Part 84 — Kubernetes & Cloud Deployment
