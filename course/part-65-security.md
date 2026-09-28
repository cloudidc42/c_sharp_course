# Part 65: Security Deep Dive — OWASP, Data Protection, Headers & SAST (Steps 1669-1684)

## Steps 1669-1684: Mastering Application Security in .NET

Security is not a feature — it is a fundamental property. This part covers the full security stack for production ASP.NET Core applications: OWASP Top 10 mitigations, the Data Protection API, HTTP security headers, input validation, SQL/command injection prevention, secrets management, and SAST toolchain integration.

---

## Step 1669: OWASP Top 10 Overview and .NET Mapping

| OWASP 2021 | Rank | .NET Mitigation |
|---|---|---|
| Broken Access Control | A01 | `[Authorize]`, resource-based auth, `IAuthorizationService` |
| Cryptographic Failures | A02 | Data Protection API, `AesGcm`, `Rfc2898DeriveBytes` |
| Injection | A03 | Parameterized queries, `HtmlEncoder`, `AntiXssEncoder` |
| Insecure Design | A04 | Threat modeling, defense-in-depth patterns |
| Security Misconfiguration | A05 | Security headers middleware, `UseHsts`, `UseHttpsRedirection` |
| Vulnerable Components | A06 | `dotnet list package --vulnerable`, Dependabot |
| Identification & Auth | A07 | ASP.NET Core Identity, OIDC, MFA, secure cookies |
| Software & Data Integrity | A08 | Package signing, NuGet lock files, Sigstore |
| Logging & Monitoring Failures | A09 | Structured logging, OpenTelemetry, Serilog enrichers |
| SSRF | A10 | URL validation, `AllowedHosts`, `HttpClient` restricted handlers |

---

## Step 1670: SQL Injection Prevention

```csharp
// WRONG: string concatenation → SQL injection
public async Task<User?> GetUserByNameWrong(string name)
{
    // Attacker input: name = "'; DROP TABLE Users; --"
    string sql = $"SELECT * FROM Users WHERE Name = '{name}'";
    return await _db.Users.FromSqlRaw(sql).FirstOrDefaultAsync();
}

// RIGHT 1: EF Core LINQ — always parameterized
public async Task<User?> GetUserByName(string name, CancellationToken ct)
    => await _db.Users.FirstOrDefaultAsync(u => u.Name == name, ct);

// RIGHT 2: Parameterized raw SQL with FormattableString (EF Core interpolation)
public async Task<User?> GetUserByNameRaw(string name, CancellationToken ct)
    => await _db.Users
        .FromSql($"SELECT * FROM Users WHERE Name = {name}")  // NOT FromSqlRaw — FromSql interpolates safely
        .FirstOrDefaultAsync(ct);

// RIGHT 3: Dapper with parameters
public async Task<User?> GetUserDapper(string name, CancellationToken ct)
{
    using var conn = _connectionFactory.CreateConnection();
    return await conn.QueryFirstOrDefaultAsync<User>(
        "SELECT * FROM Users WHERE Name = @Name",
        new { Name = name });
}

// WRONG: Dapper with string interpolation (same as SQL injection)
// await conn.QueryAsync($"SELECT * FROM Users WHERE Name = '{name}'");
```

---

## Step 1671: XSS Prevention

```csharp
using System.Text.Encodings.Web;

// Inject encoders from DI
public class UserProfileService(HtmlEncoder htmlEncoder, JavaScriptEncoder jsEncoder, UrlEncoder urlEncoder)
{
    // Encode user-supplied content before inserting into HTML
    public string SafeHtml(string userInput)
        => htmlEncoder.Encode(userInput);
        // "Hello <script>alert(1)</script>" → "Hello &lt;script&gt;alert(1)&lt;/script&gt;"

    // Encode for JavaScript contexts
    public string SafeJs(string userInput)
        => jsEncoder.Encode(userInput);
        // "Hello 'World'" → "Hello 'World'"

    // Encode for URL query parameters
    public string SafeUrl(string userInput)
        => urlEncoder.Encode(userInput);
}
```

```csharp
// Content Security Policy middleware — prevents injected script execution
app.Use(async (context, next) =>
{
    context.Response.Headers.Append("Content-Security-Policy",
        "default-src 'self'; " +
        "script-src 'self' 'nonce-{NONCE}' https://cdn.jsdelivr.net; " +
        "style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; " +
        "img-src 'self' data: https:; " +
        "connect-src 'self' https://api.myapp.com; " +
        "frame-ancestors 'none'; " +
        "base-uri 'self'; " +
        "form-action 'self'");

    await next();
});
```

### Anti-Forgery (CSRF) Protection

```csharp
// Program.cs — CSRF protection is on by default for Razor Pages
// For minimal APIs, add explicitly:
builder.Services.AddAntiforgery();
app.UseAntiforgery();

// Minimal API endpoint requiring CSRF token
app.MapPost("/orders", [ValidateAntiForgeryToken] async (
    PlaceOrderCommand cmd,
    IOrderService svc,
    CancellationToken ct) => await svc.PlaceAsync(cmd, ct));

// For API (non-browser) clients, use SameSite=Strict cookies instead
builder.Services.ConfigureApplicationCookie(options =>
{
    options.Cookie.SameSite = SameSiteMode.Strict;
    options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
    options.Cookie.HttpOnly = true;
});
```

---

## Step 1672: Security Headers Middleware

```csharp
// Install: NWebSec or configure headers manually
app.Use(async (context, next) =>
{
    var headers = context.Response.Headers;

    // Prevent clickjacking
    headers.Append("X-Frame-Options", "DENY");

    // Prevent MIME type sniffing
    headers.Append("X-Content-Type-Options", "nosniff");

    // Control referrer information
    headers.Append("Referrer-Policy", "strict-origin-when-cross-origin");

    // Block old IE compatibility mode
    headers.Append("X-UA-Compatible", "IE=edge");

    // Permissions policy — disable unused browser features
    headers.Append("Permissions-Policy",
        "camera=(), microphone=(), geolocation=(), payment=()");

    // Remove server identification
    headers.Remove("Server");
    headers.Remove("X-Powered-By");

    await next();
});

// HSTS — force HTTPS for 1 year, include subdomains, preload
app.UseHsts();  // adds Strict-Transport-Security header

// In production, configure with longer maxAge:
builder.Services.AddHsts(options =>
{
    options.Preload = true;
    options.IncludeSubDomains = true;
    options.MaxAge = TimeSpan.FromDays(365);
    options.ExcludedHosts.Add("localhost");
});
```

---

## Step 1673: Data Protection API

The ASP.NET Core Data Protection API provides a simple, purpose-bound crypto system for protecting sensitive data (cookies, tokens, PII at rest).

```csharp
using Microsoft.AspNetCore.DataProtection;

// Setup — critical: keys must be persisted and protected
builder.Services.AddDataProtection()
    .PersistKeysToFileSystem(new DirectoryInfo("/app/keys"))  // persist keys
    .ProtectKeysWithAzureKeyVault(
        new Uri("https://myvault.vault.azure.net/keys/data-protection"),
        new DefaultAzureCredential())                          // encrypt keys
    .SetApplicationName("MyApp")                               // isolate from other apps
    .SetDefaultKeyLifetime(TimeSpan.FromDays(90));
```

```csharp
// Usage — purpose string isolates protectors
public class TokenService(IDataProtectionProvider provider)
{
    // Purpose string must be unique to this use case
    private readonly IDataProtector _protector =
        provider.CreateProtector("MyApp.Tokens.PasswordReset.v1");

    public string Protect(string userId) => _protector.Protect(userId);

    public string? Unprotect(string token)
    {
        try { return _protector.Unprotect(token); }
        catch (CryptographicException) { return null; } // tampered or expired
    }
}

// Time-limited tokens (expire after 15 minutes)
public class TimeLimitedTokenService(IDataProtectionProvider provider)
{
    private readonly ITimeLimitedDataProtector _protector =
        provider.CreateProtector("MyApp.Tokens.EmailVerification.v1")
                .ToTimeLimitedDataProtector();

    public string CreateToken(string email)
        => _protector.Protect(email, lifetime: TimeSpan.FromMinutes(15));

    public string? ValidateToken(string token)
    {
        try { return _protector.Unprotect(token); }
        catch (SecurityTokenExpiredException) { return null; }   // expired
        catch (CryptographicException) { return null; }           // invalid
    }
}
```

---

## Step 1674: Password Hashing with ASP.NET Core Identity

```csharp
using Microsoft.AspNetCore.Identity;

// Inject IPasswordHasher<T> directly (no full Identity setup needed)
builder.Services.AddScoped<IPasswordHasher<User>, PasswordHasher<User>>();

// Configure PBKDF2 iteration count (default is 310,000 in .NET 8)
builder.Services.Configure<PasswordHasherOptions>(options =>
{
    options.IterationCount = 600_000; // OWASP 2023 recommendation
});

public class AuthService(IPasswordHasher<User> hasher)
{
    public User Register(string email, string plainPassword)
    {
        var user = new User { Email = email };
        user.PasswordHash = hasher.HashPassword(user, plainPassword);
        return user;
    }

    public bool VerifyPassword(User user, string plainPassword)
    {
        var result = hasher.VerifyHashedPassword(user, user.PasswordHash!, plainPassword);
        return result switch
        {
            PasswordVerificationResult.Success => true,
            PasswordVerificationResult.SuccessRehashNeeded => true, // update hash in DB
            _ => false
        };
    }
}
```

```csharp
// For non-Identity scenarios: PBKDF2 directly
using System.Security.Cryptography;

public static string HashPassword(string password)
{
    byte[] salt = RandomNumberGenerator.GetBytes(32);
    byte[] hash = Rfc2898DeriveBytes.Pbkdf2(
        password,
        salt,
        iterations: 600_000,
        hashAlgorithm: HashAlgorithmName.SHA256,
        outputLength: 32);

    return $"v1:{Convert.ToBase64String(salt)}:{Convert.ToBase64String(hash)}";
}

public static bool VerifyPassword(string password, string storedHash)
{
    var parts = storedHash.Split(':');
    if (parts.Length != 3 || parts[0] != "v1") return false;

    byte[] salt = Convert.FromBase64String(parts[1]);
    byte[] storedHashBytes = Convert.FromBase64String(parts[2]);
    byte[] hash = Rfc2898DeriveBytes.Pbkdf2(
        password, salt, 600_000, HashAlgorithmName.SHA256, 32);

    return CryptographicOperations.FixedTimeEquals(hash, storedHashBytes);
    // FixedTimeEquals prevents timing attacks
}
```

---

## Step 1675: Secrets Management

```csharp
// Development: User Secrets (never check into source control)
// dotnet user-secrets set "ConnectionStrings:Database" "Server=..."
// Stored in: ~/.microsoft/usersecrets/<app-guid>/secrets.json

// Program.cs — User Secrets are auto-loaded in Development environment
var builder = WebApplication.CreateBuilder(args);
// builder.Configuration already includes user secrets in Development

// Production: Azure Key Vault
builder.Configuration.AddAzureKeyVault(
    new Uri("https://myvault.vault.azure.net/"),
    new DefaultAzureCredential()); // uses managed identity in production

// Production: Kubernetes secrets via environment variables
// Secret "db-password" → env var DB_PASSWORD
// Configuration.GetValue<string>("DB_PASSWORD") works automatically
```

```csharp
// Strongly-typed secrets with options validation
public class DatabaseOptions
{
    public const string SectionName = "Database";

    [Required]
    public string ConnectionString { get; init; } = string.Empty;

    [Required, MinLength(32)]
    public string EncryptionKey { get; init; } = string.Empty;

    [Range(1, 100)]
    public int MaxConnections { get; init; } = 20;
}

builder.Services
    .AddOptions<DatabaseOptions>()
    .BindConfiguration(DatabaseOptions.SectionName)
    .ValidateDataAnnotations()
    .ValidateOnStart(); // fail fast if misconfigured
```

---

## Step 1676: Authorization — Resource-Based and Policy-Based

```csharp
// Define requirements
public class OrderOwnerRequirement : IAuthorizationRequirement { }

public class OrderOwnerHandler(IOrderRepository orders)
    : AuthorizationHandler<OrderOwnerRequirement, Order>
{
    protected override async Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        OrderOwnerRequirement requirement,
        Order order)
    {
        var userId = context.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        if (userId is null) return; // not authenticated

        if (order.CustomerId == userId)
        {
            context.Succeed(requirement);
            return;
        }

        // Admins can access any order
        if (context.User.IsInRole("Admin"))
        {
            context.Succeed(requirement);
            return;
        }
    }
}

// Register
builder.Services.AddAuthorizationBuilder()
    .AddPolicy("CanEditOrder", policy =>
        policy.Requirements.Add(new OrderOwnerRequirement()))
    .AddPolicy("SeniorStaff", policy =>
    {
        policy.RequireRole("Manager", "Admin");
        policy.RequireClaim("department", "Operations");
        policy.RequireAssertion(ctx =>
            ctx.User.HasClaim("employment_years", v => int.Parse(v) >= 3));
    });

builder.Services.AddScoped<IAuthorizationHandler, OrderOwnerHandler>();
```

```csharp
// Use in endpoint
app.MapPut("/orders/{id:guid}", async (
    Guid id,
    UpdateOrderCommand cmd,
    IAuthorizationService auth,
    IOrderRepository repo,
    HttpContext ctx,
    CancellationToken ct) =>
{
    var order = await repo.GetByIdAsync(id, ct);
    if (order is null) return Results.NotFound();

    var authResult = await auth.AuthorizeAsync(ctx.User, order, "CanEditOrder");
    if (!authResult.Succeeded) return Results.Forbid();

    await repo.UpdateAsync(order.Apply(cmd), ct);
    return Results.NoContent();
});
```

---

## Step 1677: Preventing SSRF (Server-Side Request Forgery)

```csharp
using System;
using System.Net;
using System.Net.Http;

// Custom HttpMessageHandler that blocks private/loopback addresses
public class SsrfProtectedHandler : DelegatingHandler
{
    private static readonly IPAddress[] _blocked = new[]
    {
        IPAddress.Loopback,   // 127.0.0.1
        IPAddress.IPv6Loopback,
        IPAddress.Any,        // 0.0.0.0
    };

    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request,
        CancellationToken cancellationToken)
    {
        if (request.RequestUri is null)
            throw new InvalidOperationException("Request URI is null.");

        // Only allow HTTPS
        if (request.RequestUri.Scheme != Uri.UriSchemeHttps)
            throw new SecurityException($"Only HTTPS is allowed. Got: {request.RequestUri.Scheme}");

        // Resolve the host and check it's not private
        var addresses = await Dns.GetHostAddressesAsync(
            request.RequestUri.Host, cancellationToken);

        foreach (var addr in addresses)
        {
            if (IsPrivateAddress(addr))
                throw new SecurityException(
                    $"Request to private/loopback address {addr} is not allowed.");
        }

        return await base.SendAsync(request, cancellationToken);
    }

    private static bool IsPrivateAddress(IPAddress addr)
    {
        if (addr.Equals(IPAddress.Loopback) || addr.Equals(IPAddress.IPv6Loopback)) return true;

        byte[] bytes = addr.GetAddressBytes();
        return addr.AddressFamily == System.Net.Sockets.AddressFamily.InterNetwork && (
            bytes[0] == 10 ||                              // 10.0.0.0/8
            (bytes[0] == 172 && bytes[1] >= 16 && bytes[1] <= 31) || // 172.16.0.0/12
            (bytes[0] == 192 && bytes[1] == 168) ||       // 192.168.0.0/16
            bytes[0] == 169 && bytes[1] == 254);          // 169.254.0.0/16 link-local
    }
}

// Register
builder.Services.AddHttpClient("external")
    .AddHttpMessageHandler<SsrfProtectedHandler>();
```

---

## Step 1678: Cryptography Best Practices

```csharp
using System.Security.Cryptography;

// Symmetric encryption with AES-GCM (authenticated encryption)
public class AesGcmEncryption
{
    private const int NonceSize = 12;  // 96-bit nonce for GCM
    private const int TagSize = 16;    // 128-bit authentication tag
    private const int KeySize = 32;    // 256-bit key

    public static byte[] Encrypt(ReadOnlySpan<byte> plaintext, ReadOnlySpan<byte> key)
    {
        byte[] nonce = RandomNumberGenerator.GetBytes(NonceSize);
        byte[] ciphertext = new byte[plaintext.Length];
        byte[] tag = new byte[TagSize];

        using var aes = new AesGcm(key, TagSize);
        aes.Encrypt(nonce, plaintext, ciphertext, tag);

        // Layout: [nonce (12)] + [tag (16)] + [ciphertext]
        var result = new byte[NonceSize + TagSize + ciphertext.Length];
        nonce.CopyTo(result.AsSpan(0, NonceSize));
        tag.CopyTo(result.AsSpan(NonceSize, TagSize));
        ciphertext.CopyTo(result.AsSpan(NonceSize + TagSize));
        return result;
    }

    public static byte[] Decrypt(ReadOnlySpan<byte> encrypted, ReadOnlySpan<byte> key)
    {
        if (encrypted.Length < NonceSize + TagSize)
            throw new CryptographicException("Invalid encrypted data length.");

        var nonce = encrypted.Slice(0, NonceSize);
        var tag = encrypted.Slice(NonceSize, TagSize);
        var ciphertext = encrypted.Slice(NonceSize + TagSize);
        var plaintext = new byte[ciphertext.Length];

        using var aes = new AesGcm(key, TagSize);
        aes.Decrypt(nonce, ciphertext, tag, plaintext);
        return plaintext;
    }
}
```

```csharp
// Digital signatures with RSA
public class DigitalSignatureService
{
    private readonly RSA _rsa = RSA.Create(4096);

    public byte[] Sign(byte[] data)
    {
        return _rsa.SignData(data, HashAlgorithmName.SHA256, RSASignaturePadding.Pss);
    }

    public bool Verify(byte[] data, byte[] signature)
    {
        return _rsa.VerifyData(data, signature, HashAlgorithmName.SHA256, RSASignaturePadding.Pss);
    }

    // Export public key for distribution
    public string GetPublicKeyPem() => _rsa.ExportSubjectPublicKeyInfoPem();

    // Import public key for verification
    public static bool VerifyWithPublicKey(byte[] data, byte[] signature, string publicKeyPem)
    {
        using var rsa = RSA.Create();
        rsa.ImportFromPem(publicKeyPem);
        return rsa.VerifyData(data, signature, HashAlgorithmName.SHA256, RSASignaturePadding.Pss);
    }
}
```

```csharp
// Key derivation — HKDF (HMAC-based Key Derivation Function)
using System.Security.Cryptography;

byte[] masterKey = RandomNumberGenerator.GetBytes(32);
byte[] salt = RandomNumberGenerator.GetBytes(32);
string info = "MyApp.DatabaseEncryption.v1";

byte[] derivedKey = HKDF.DeriveKey(
    hashAlgorithmName: HashAlgorithmName.SHA256,
    ikm: masterKey,           // input key material
    outputLength: 32,
    salt: salt,
    info: System.Text.Encoding.UTF8.GetBytes(info));
// Different 'info' strings produce different independent keys from the same master
```

---

## Step 1679: Input Validation

```csharp
using System.ComponentModel.DataAnnotations;
using FluentValidation;

// FluentValidation for rich business validation
public class PlaceOrderValidator : AbstractValidator<PlaceOrderCommand>
{
    public PlaceOrderValidator()
    {
        RuleFor(x => x.CustomerId)
            .NotEmpty()
            .MaximumLength(50)
            .Matches(@"^CUST-\d+$").WithMessage("CustomerID must match CUST-{number}");

        RuleFor(x => x.Items)
            .NotEmpty().WithMessage("At least one item required")
            .Must(items => items.Count <= 100).WithMessage("Maximum 100 items per order");

        RuleForEach(x => x.Items).SetValidator(new OrderItemValidator());

        RuleFor(x => x.ShippingAddress.PostalCode)
            .NotEmpty()
            .Matches(@"^\d{5}(-\d{4})?$").WithMessage("Invalid US postal code");
    }
}

public class OrderItemValidator : AbstractValidator<OrderItem>
{
    public OrderItemValidator()
    {
        RuleFor(x => x.Sku)
            .NotEmpty()
            .MaximumLength(50)
            .Matches(@"^[A-Z0-9\-]+$").WithMessage("SKU must be uppercase alphanumeric");

        RuleFor(x => x.Quantity)
            .GreaterThan(0)
            .LessThanOrEqualTo(9999);

        RuleFor(x => x.UnitPrice)
            .GreaterThan(0)
            .LessThanOrEqualTo(999_999.99m);
    }
}

// Register in DI
builder.Services.AddValidatorsFromAssemblyContaining<PlaceOrderValidator>();
builder.Services.AddFluentValidationAutoValidation();
```

```csharp
// Guard clauses for service layer validation
using Ardalis.GuardClauses;

public async Task<Order> PlaceOrderAsync(PlaceOrderCommand cmd, CancellationToken ct)
{
    Guard.Against.NullOrEmpty(cmd.CustomerId, nameof(cmd.CustomerId));
    Guard.Against.NullOrEmpty(cmd.Items, nameof(cmd.Items));
    Guard.Against.OutOfRange(cmd.Items.Count, nameof(cmd.Items), 1, 100);

    // Domain validation
    foreach (var item in cmd.Items)
    {
        Guard.Against.OutOfRange(item.Quantity, $"Items[{item.Sku}].Quantity", 1, 9999);
        Guard.Against.NegativeOrZero(item.UnitPrice, $"Items[{item.Sku}].UnitPrice");
    }

    var order = Order.Place(
        orderId: Guid.NewGuid(),
        customerId: cmd.CustomerId,
        items: cmd.Items.Select(i => new OrderLineItem(i.Sku, i.Quantity, i.UnitPrice)).ToList(),
        shippingAddress: cmd.ShippingAddress);

    await _repository.SaveAsync(order, ct);
    return order;
}
```

---

## Step 1680: JWT Security

```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;

builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = "https://auth.myapp.com";  // OIDC discovery endpoint
        options.Audience = "myapp-api";

        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ClockSkew = TimeSpan.FromSeconds(30),  // small skew only
            RequireSignedTokens = true,
            RequireExpirationTime = true,
        };

        // Reject tokens that don't use RS256 or ES256 (reject HS256 from untrusted issuers)
        options.TokenValidationParameters.ValidAlgorithms = new[] { "RS256", "ES256" };

        // For debugging only — log token validation failures
        options.Events = new JwtBearerEvents
        {
            OnAuthenticationFailed = ctx =>
            {
                var logger = ctx.HttpContext.RequestServices
                    .GetRequiredService<ILogger<Program>>();
                logger.LogWarning(ctx.Exception, "JWT authentication failed");
                return Task.CompletedTask;
            }
        };
    });
```

```csharp
// Token generation (in the auth server)
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using Microsoft.IdentityModel.Tokens;

public class JwtTokenService(IOptions<JwtOptions> opts)
{
    private readonly JwtOptions _opts = opts.Value;

    public string GenerateToken(User user, IEnumerable<string> roles)
    {
        var claims = new List<Claim>
        {
            new(JwtRegisteredClaimNames.Sub, user.Id.ToString()),
            new(JwtRegisteredClaimNames.Email, user.Email),
            new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
            new(JwtRegisteredClaimNames.Iat,
                DateTimeOffset.UtcNow.ToUnixTimeSeconds().ToString(),
                ClaimValueTypes.Integer64),
        };

        claims.AddRange(roles.Select(r => new Claim(ClaimTypes.Role, r)));

        var key = new SymmetricSecurityKey(
            System.Text.Encoding.UTF8.GetBytes(_opts.SigningKey));
        // In production use RSA or ECDSA, not HMAC:
        // var key = new RsaSecurityKey(RSA.Create());

        var token = new JwtSecurityToken(
            issuer: _opts.Issuer,
            audience: _opts.Audience,
            claims: claims,
            notBefore: DateTime.UtcNow,
            expires: DateTime.UtcNow.AddMinutes(15), // short-lived
            signingCredentials: new SigningCredentials(key, SecurityAlgorithms.HmacSha256));

        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}
```

---

## Step 1681: Secure File Upload

```csharp
public class SecureFileUploadService(IWebHostEnvironment env)
{
    private static readonly string[] _allowedExtensions = [".jpg", ".jpeg", ".png", ".pdf"];
    private static readonly string[] _allowedMimeTypes =
        ["image/jpeg", "image/png", "application/pdf"];
    private const long MaxFileSizeBytes = 10 * 1024 * 1024; // 10 MB

    public async Task<string> UploadAsync(IFormFile file, CancellationToken ct)
    {
        // 1. Validate size
        if (file.Length > MaxFileSizeBytes)
            throw new ValidationException($"File too large. Max {MaxFileSizeBytes / 1024 / 1024} MB.");

        if (file.Length == 0)
            throw new ValidationException("File is empty.");

        // 2. Validate content type (from Content-Type header)
        if (!_allowedMimeTypes.Contains(file.ContentType, StringComparer.OrdinalIgnoreCase))
            throw new ValidationException($"File type '{file.ContentType}' is not allowed.");

        // 3. Validate extension (from filename)
        var extension = Path.GetExtension(file.FileName).ToLowerInvariant();
        if (!_allowedExtensions.Contains(extension))
            throw new ValidationException($"Extension '{extension}' is not allowed.");

        // 4. Validate magic bytes (actual file content, not just MIME/extension)
        await using var stream = file.OpenReadStream();
        if (!await IsValidFileContentAsync(stream, extension, ct))
            throw new ValidationException("File content does not match declared type.");

        stream.Position = 0;

        // 5. Generate a safe, random filename — NEVER use user-supplied filename
        var safeFileName = $"{Guid.NewGuid()}{extension}";

        // 6. Store OUTSIDE web root so files cannot be directly executed
        var uploadPath = Path.Combine(env.ContentRootPath, "uploads", safeFileName);
        Directory.CreateDirectory(Path.GetDirectoryName(uploadPath)!);

        await using var dest = File.Create(uploadPath);
        await stream.CopyToAsync(dest, ct);

        return safeFileName;
    }

    private static async Task<bool> IsValidFileContentAsync(
        Stream stream,
        string extension,
        CancellationToken ct)
    {
        byte[] magic = new byte[8];
        int read = await stream.ReadAsync(magic, ct);

        return extension switch
        {
            ".jpg" or ".jpeg" => read >= 3 && magic[0] == 0xFF && magic[1] == 0xD8 && magic[2] == 0xFF,
            ".png" => read >= 8 &&
                      magic[0] == 0x89 && magic[1] == 0x50 && magic[2] == 0x4E && magic[3] == 0x47,
            ".pdf" => read >= 4 &&
                      magic[0] == 0x25 && magic[1] == 0x50 && magic[2] == 0x44 && magic[3] == 0x46,
            _ => false
        };
    }
}
```

---

## Step 1682: Logging Security-Sensitive Information

```csharp
using Microsoft.Extensions.Logging;

// WRONG: logging sensitive data
_logger.LogInformation("User {Email} logged in with password {Password}", email, password);
_logger.LogDebug("JWT token: {Token}", jwtToken);
_logger.LogInformation("Processing payment for card {CardNumber}", cardNumber);

// RIGHT: log only what's needed for diagnostics
_logger.LogInformation("User authenticated. UserId={UserId}", userId);
_logger.LogDebug("JWT issued. Subject={Subject} ExpiresAt={ExpiresAt}", subject, expiresAt);
_logger.LogInformation("Payment processed. Last4={Last4} Amount={Amount}", last4, amount);

// Structured log redaction middleware
builder.Services.AddHttpLogging(opts =>
{
    opts.LoggingFields = HttpLoggingFields.RequestPath |
                         HttpLoggingFields.RequestMethod |
                         HttpLoggingFields.ResponseStatusCode;
    // DO NOT include: RequestBody, ResponseBody, RequestHeaders (may contain Authorization)
});
```

```csharp
// Serilog destructuring policies to redact sensitive fields
using Serilog.Core;
using Serilog.Events;

public class SensitiveDataDestructuringPolicy : IDestructuringPolicy
{
    private static readonly HashSet<string> _sensitiveFields =
        new(StringComparer.OrdinalIgnoreCase)
        { "password", "token", "secret", "key", "authorization", "cookie" };

    public bool TryDestructureInstance(object value, ILogEventPropertyValueFactory factory,
        out LogEventPropertyValue? result)
    {
        if (value is not IDictionary<string, object> dict)
        {
            result = null;
            return false;
        }

        var safe = dict
            .Select(kv => new KeyValuePair<ScalarValue, LogEventPropertyValue>(
                new ScalarValue(kv.Key),
                _sensitiveFields.Any(s => kv.Key.Contains(s))
                    ? new ScalarValue("[REDACTED]")
                    : factory.CreatePropertyValue(kv.Value)))
            .ToDictionary(kv => kv.Key, kv => kv.Value);

        result = new DictionaryValue(safe);
        return true;
    }
}
```

---

## Step 1683: SAST Integration — dotnet-security-scan and Semgrep

```bash
# Install security audit tools
dotnet tool install -g security-scan
dotnet tool install -g snyk

# Scan for vulnerable NuGet packages
dotnet list package --vulnerable --include-transitive

# Run security-scan (detects OWASP issues in source code)
dotnet security-scan MyApp.sln

# Snyk full scan (SCA + SAST)
snyk test --file=MyApp.sln
snyk code test
```

```yaml
# .github/workflows/security.yml
name: Security Scan

on: [push, pull_request]

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: 9.x

      - name: Restore packages
        run: dotnet restore

      - name: Check vulnerable packages
        run: dotnet list package --vulnerable --include-transitive

      - name: Run Semgrep SAST
        uses: semgrep/semgrep-action@v1
        with:
          config: >-
            p/csharp
            p/owasp-top-ten
            p/dotnet-security

      - name: Run Trivy (container scan)
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          exit-code: '1'
          severity: 'CRITICAL,HIGH'
```

```yaml
# .semgrepignore — exclude generated code and tests
**/obj/**
**/bin/**
**/*.g.cs
**/*Tests/**
```

---

## Step 1684: Security Checklist — Production Readiness

```csharp
// Security middleware pipeline order (matters!)
var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/error");   // 1. generic error page, no stack traces
    app.UseHsts();                        // 2. HTTPS enforcement
}

app.UseHttpsRedirection();               // 3. redirect HTTP → HTTPS
app.UseStaticFiles();                    // 4. static files before auth
app.UseRouting();                        // 5. route matching

app.UseCors("AllowMyOrigins");           // 6. CORS before auth
app.UseAuthentication();                 // 7. who are you?
app.UseAuthorization();                  // 8. what can you do?
app.UseAntiforgery();                    // 9. CSRF protection
app.UseRateLimiter();                    // 10. throttle requests

app.MapControllers().RequireAuthorization(); // 11. endpoints, secure by default
```

```csharp
// Secure defaults: ProblemDetails without leaking internals
builder.Services.AddProblemDetails(options =>
{
    options.CustomizeProblemDetails = ctx =>
    {
        // Remove stack trace, exception type, and other internal details
        ctx.ProblemDetails.Extensions.Remove("exception");
        ctx.ProblemDetails.Extensions.Remove("traceId");
        // Only include trace ID in Development
        if (ctx.HttpContext.RequestServices
                .GetRequiredService<IWebHostEnvironment>().IsDevelopment())
        {
            ctx.ProblemDetails.Extensions["traceId"] =
                System.Diagnostics.Activity.Current?.Id ?? ctx.HttpContext.TraceIdentifier;
        }
    };
});

// Override the default exception handler to prevent detail leakage
app.Map("/error", (HttpContext ctx) =>
{
    var feature = ctx.Features.Get<IExceptionHandlerFeature>();
    var logger = ctx.RequestServices.GetRequiredService<ILogger<Program>>();
    logger.LogError(feature?.Error, "Unhandled exception");
    // Return generic 500 — never return exception details in production
    return Results.Problem(title: "An error occurred.", statusCode: 500);
});
```

| Security Concern | .NET Solution |
|---|---|
| SQL Injection | EF Core LINQ, `FromSql()`, parameterized Dapper |
| XSS | `HtmlEncoder`, CSP headers, `[ValidateAntiForgeryToken]` |
| CSRF | `UseAntiforgery()`, `SameSite=Strict` cookies |
| Password storage | `IPasswordHasher<T>`, PBKDF2 600k iterations |
| Secret storage | User Secrets (dev), Azure Key Vault (prod) |
| Transport security | HSTS, `UseHttpsRedirection`, TLS 1.2+ |
| Authentication | ASP.NET Core Identity, OIDC, JWT validation |
| Authorization | `[Authorize]`, `IAuthorizationService`, resource-based |
| Data encryption | AES-GCM, Data Protection API |
| File uploads | Extension + magic byte validation, random names |
| SSRF | `SsrfProtectedHandler`, restrict `HttpClient` outbound hosts |
| Vulnerable packages | `dotnet list package --vulnerable`, Dependabot, Snyk |
| Logging | Never log PII, tokens, passwords; use Serilog redaction |

### What Next?

- **Part 66**: Kubernetes & Container Orchestration — Helm, KEDA, Dapr, probes, resource limits
- **Part 67**: Messaging Patterns — MassTransit sagas, Wolverine, Outbox, exactly-once semantics
- **Part 68**: Database Performance — EF Core compiled queries, Dapper multi-mapping, indexing strategy
