# Part 33: Security in .NET Applications

## Steps 916-950: Defense-in-Depth Security

---

## Step 916: Security Fundamentals

```
Defense in Depth:
Layer 1: Infrastructure (Firewalls, WAF, DDoS protection)
Layer 2: Network (TLS/HTTPS, VPN, Network policies)
Layer 3: Application (Auth, AuthZ, Input validation)
Layer 4: Data (Encryption at rest, masking, audit)
Layer 5: Monitoring (Logging, alerting, SIEM)
```

### OWASP Top 10 (2021)
1. Broken Access Control
2. Cryptographic Failures
3. Injection
4. Insecure Design
5. Security Misconfiguration
6. Vulnerable Components
7. Authentication Failures
8. Software/Data Integrity Failures
9. Security Logging Failures
10. Server-Side Request Forgery (SSRF)

---

## Step 917: Input Validation และ Sanitization

```csharp
// FluentValidation
using FluentValidation;

public class CreateUserRequest
{
    public string Username { get; set; } = string.Empty;
    public string Email { get; set; } = string.Empty;
    public string Password { get; set; } = string.Empty;
    public int Age { get; set; }
}

public class CreateUserRequestValidator : AbstractValidator<CreateUserRequest>
{
    public CreateUserRequestValidator()
    {
        RuleFor(x => x.Username)
            .NotEmpty()
            .Length(3, 50)
            .Matches(@"^[a-zA-Z0-9_-]+$")
            .WithMessage("Username can only contain letters, numbers, underscores and hyphens");
        
        RuleFor(x => x.Email)
            .NotEmpty()
            .EmailAddress()
            .MaximumLength(256);
        
        RuleFor(x => x.Password)
            .NotEmpty()
            .MinimumLength(8)
            .MaximumLength(128)
            .Matches(@"[A-Z]").WithMessage("Password must have at least one uppercase letter")
            .Matches(@"[a-z]").WithMessage("Password must have at least one lowercase letter")
            .Matches(@"[0-9]").WithMessage("Password must have at least one number")
            .Matches(@"[!@#$%^&*(),.?\"":{}|<>]")
            .WithMessage("Password must have at least one special character");
        
        RuleFor(x => x.Age)
            .InclusiveBetween(13, 120)
            .WithMessage("Age must be between 13 and 120");
    }
}

// Register FluentValidation
builder.Services.AddFluentValidationAutoValidation();
builder.Services.AddValidatorsFromAssemblyContaining<CreateUserRequestValidator>();
```

---

## Step 918: SQL Injection Prevention

```csharp
// ไม่ดี - SQL Injection ได้
public async Task<User?> GetUserBad(string username)
{
    // NEVER DO THIS
    var sql = $"SELECT * FROM Users WHERE Username = '{username}'";
    return await _db.Users.FromSqlRaw(sql).FirstOrDefaultAsync();
}

// ดี - Parameterized query
public async Task<User?> GetUserGood(string username, CancellationToken ct)
{
    // Using EF Core (safe by default)
    return await _db.Users
        .FirstOrDefaultAsync(u => u.Username == username, ct);
}

// ดี - Parameterized Raw SQL
public async Task<User?> GetUserRawSafe(string username, CancellationToken ct)
{
    return await _db.Users
        .FromSqlInterpolated($"SELECT * FROM Users WHERE Username = {username}")
        .FirstOrDefaultAsync(ct);
}

// ดี - สำหรับ Dapper
public async Task<User?> GetUserDapper(string username, IDbConnection conn)
{
    return await conn.QueryFirstOrDefaultAsync<User>(
        "SELECT * FROM Users WHERE Username = @Username",
        new { Username = username }); // parameterized
}
```

---

## Step 919: XSS Prevention

```csharp
// ใน Razor/Blazor - auto-encoded by default
// @user.Name  → encoded automatically
// @Html.Raw(user.Name)  → NOT safe, avoid unless necessary

// HTML Encoding manually
using System.Text.Encodings.Web;

public class XssPrevention
{
    public string SafeOutput(string userInput)
    {
        // Encode for HTML context
        return HtmlEncoder.Default.Encode(userInput);
    }
    
    public string SafeJavaScript(string userInput)
    {
        // Encode for JavaScript context  
        return JavaScriptEncoder.Default.Encode(userInput);
    }
    
    public string SafeUrl(string userInput)
    {
        // Encode for URL context
        return UrlEncoder.Default.Encode(userInput);
    }
}

// Content Security Policy (CSP)
app.Use(async (context, next) =>
{
    context.Response.Headers.Append("Content-Security-Policy",
        "default-src 'self'; " +
        "script-src 'self' 'nonce-{nonce}'; " +
        "style-src 'self' fonts.googleapis.com; " +
        "font-src fonts.gstatic.com; " +
        "img-src 'self' data: https:; " +
        "connect-src 'self'; " +
        "frame-ancestors 'none'");
    
    context.Response.Headers.Append("X-Frame-Options", "DENY");
    context.Response.Headers.Append("X-Content-Type-Options", "nosniff");
    context.Response.Headers.Append("Referrer-Policy", "strict-origin-when-cross-origin");
    context.Response.Headers.Append("Permissions-Policy", 
        "geolocation=(), camera=(), microphone=()");
    
    await next();
});
```

---

## Step 920: Authentication - JWT Best Practices

```csharp
// JWT configuration
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(opts =>
    {
        opts.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(
                Convert.FromBase64String(
                    builder.Configuration["Jwt:Secret"]!)),
            ValidateIssuer = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidateAudience = true,
            ValidAudience = builder.Configuration["Jwt:Audience"],
            ValidateLifetime = true,
            ClockSkew = TimeSpan.FromMinutes(5),
            RequireExpirationTime = true
        };
        
        opts.Events = new JwtBearerEvents
        {
            OnAuthenticationFailed = ctx =>
            {
                if (ctx.Exception is SecurityTokenExpiredException)
                    ctx.Response.Headers.Append(
                        "Token-Expired", "true");
                return Task.CompletedTask;
            }
        };
    });

// Token generation service
public class TokenService(IConfiguration config)
{
    public string GenerateAccessToken(User user)
    {
        var key = new SymmetricSecurityKey(
            Convert.FromBase64String(config["Jwt:Secret"]!));
        var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);
        
        var claims = new List<Claim>
        {
            new(JwtRegisteredClaimNames.Sub, user.Id),
            new(JwtRegisteredClaimNames.Email, user.Email),
            new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
            new(JwtRegisteredClaimNames.Iat, 
                DateTimeOffset.UtcNow.ToUnixTimeSeconds().ToString(),
                ClaimValueTypes.Integer64)
        };
        
        // Add roles
        foreach (var role in user.Roles)
            claims.Add(new Claim(ClaimTypes.Role, role));
        
        var token = new JwtSecurityToken(
            issuer: config["Jwt:Issuer"],
            audience: config["Jwt:Audience"],
            claims: claims,
            notBefore: DateTime.UtcNow,
            expires: DateTime.UtcNow.AddMinutes(15), // Short lived!
            signingCredentials: creds);
        
        return new JwtSecurityTokenHandler().WriteToken(token);
    }
    
    public string GenerateRefreshToken()
    {
        var randomBytes = new byte[64];
        using var rng = RandomNumberGenerator.Create();
        rng.GetBytes(randomBytes);
        return Convert.ToBase64String(randomBytes);
    }
}
```

---

## Step 921: Password Security

```csharp
// ใช้ bcrypt/Argon2 - ไม่ใช้ MD5, SHA1
using BCrypt.Net;

public class PasswordService
{
    private const int WorkFactor = 12; // ~300ms on modern hardware
    
    public string Hash(string password)
        => BCrypt.HashPassword(password, workFactor: WorkFactor);
    
    public bool Verify(string password, string hash)
        => BCrypt.Verify(password, hash);
    
    // ASP.NET Core Identity hasher (Argon2 via PBKDF2)
    public string HashWithIdentity(string password)
    {
        var hasher = new PasswordHasher<object>();
        return hasher.HashPassword(null!, password);
    }
}

// Password strength checking
public static class PasswordStrength
{
    public static (bool IsStrong, List<string> Errors) Check(string password)
    {
        var errors = new List<string>();
        
        if (password.Length < 12)
            errors.Add("At least 12 characters");
        if (!password.Any(char.IsUpper))
            errors.Add("At least one uppercase letter");
        if (!password.Any(char.IsLower))
            errors.Add("At least one lowercase letter");
        if (!password.Any(char.IsDigit))
            errors.Add("At least one digit");
        if (!password.Any(c => "!@#$%^&*()_+-=[]{}|;':\",./<>?".Contains(c)))
            errors.Add("At least one special character");
        
        // Check against common passwords
        var commonPasswords = new HashSet<string>(File.ReadAllLines("common-passwords.txt"));
        if (commonPasswords.Contains(password.ToLower()))
            errors.Add("Too common password");
        
        return (errors.Count == 0, errors);
    }
    
    // Rate limiting for brute force protection
    private static readonly MemoryCache _loginAttempts = 
        new(new MemoryCacheOptions());
    
    public static bool IsRateLimited(string ipOrUsername)
    {
        var key = $"login:{ipOrUsername}";
        var attempts = _loginAttempts.GetOrCreate(key, entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(15);
            return 0;
        });
        
        if (attempts >= 5)
            return true;
        
        _loginAttempts.Set(key, attempts + 1, TimeSpan.FromMinutes(15));
        return false;
    }
}
```

---

## Step 922: CSRF Protection

```csharp
// ASP.NET Core CSRF protection (for form-based apps)
builder.Services.AddAntiforgery(opts =>
{
    opts.HeaderName = "X-CSRF-TOKEN";
    opts.Cookie.Name = "XSRF-TOKEN";
    opts.Cookie.SameSite = SameSiteMode.Strict;
    opts.Cookie.SecurePolicy = CookieSecurePolicy.Always;
});

// ใน controller
[HttpPost]
[ValidateAntiForgeryToken]
public IActionResult Submit(FormModel model) { }

// สำหรับ API (SPA) ใช้ SameSite cookies + Origin header validation
app.Use(async (context, next) =>
{
    if (context.Request.Method is "POST" or "PUT" or "DELETE" or "PATCH")
    {
        var origin = context.Request.Headers.Origin.FirstOrDefault();
        var allowedOrigins = new[] { "https://app.example.com" };
        
        if (origin != null && !allowedOrigins.Contains(origin))
        {
            context.Response.StatusCode = 403;
            return;
        }
    }
    
    await next();
});
```

---

## Step 923: CORS Configuration

```csharp
builder.Services.AddCors(opts =>
{
    opts.AddPolicy("ProductionCors", policy =>
    {
        policy
            .WithOrigins(
                "https://app.example.com",
                "https://admin.example.com")
            .WithMethods("GET", "POST", "PUT", "DELETE")
            .WithHeaders("Content-Type", "Authorization", "X-Request-ID")
            .AllowCredentials()
            .SetPreflightMaxAge(TimeSpan.FromMinutes(10));
    });
    
    opts.AddPolicy("DevelopmentCors", policy =>
    {
        policy
            .AllowAnyOrigin()
            .AllowAnyMethod()
            .AllowAnyHeader();
    });
});

app.UseCors(builder.Environment.IsDevelopment() ? "DevelopmentCors" : "ProductionCors");
```

---

## Step 924: Data Encryption

```csharp
using System.Security.Cryptography;

public class EncryptionService
{
    // AES-256-GCM (authenticated encryption)
    public byte[] Encrypt(string plaintext, byte[] key)
    {
        var plaintextBytes = Encoding.UTF8.GetBytes(plaintext);
        var nonce = new byte[AesGcm.NonceByteSizes.MaxSize];
        var ciphertext = new byte[plaintextBytes.Length];
        var tag = new byte[AesGcm.TagByteSizes.MaxSize];
        
        RandomNumberGenerator.Fill(nonce);
        
        using var aes = new AesGcm(key, AesGcm.TagByteSizes.MaxSize);
        aes.Encrypt(nonce, plaintextBytes, ciphertext, tag);
        
        // Format: [nonce][tag][ciphertext]
        var result = new byte[nonce.Length + tag.Length + ciphertext.Length];
        Buffer.BlockCopy(nonce, 0, result, 0, nonce.Length);
        Buffer.BlockCopy(tag, 0, result, nonce.Length, tag.Length);
        Buffer.BlockCopy(ciphertext, 0, result, nonce.Length + tag.Length, ciphertext.Length);
        
        return result;
    }
    
    public string Decrypt(byte[] data, byte[] key)
    {
        var nonceSize = AesGcm.NonceByteSizes.MaxSize;
        var tagSize = AesGcm.TagByteSizes.MaxSize;
        var ciphertextSize = data.Length - nonceSize - tagSize;
        
        var nonce = data[..nonceSize];
        var tag = data[nonceSize..(nonceSize + tagSize)];
        var ciphertext = data[(nonceSize + tagSize)..];
        var plaintext = new byte[ciphertextSize];
        
        using var aes = new AesGcm(key, tagSize);
        aes.Decrypt(nonce, ciphertext, tag, plaintext);
        
        return Encoding.UTF8.GetString(plaintext);
    }
    
    // Key derivation
    public byte[] DeriveKey(string password, byte[] salt, int keyLength = 32)
    {
        using var rfc2898 = new Rfc2898DeriveBytes(
            password, salt,
            iterations: 310000,         // OWASP recommended
            HashAlgorithmName.SHA256);
        
        return rfc2898.GetBytes(keyLength);
    }
    
    // Data Protection API (ASP.NET Core built-in)
    public class DataProtectionExample(IDataProtector protector)
    {
        public string Protect(string plaintext)
            => protector.Protect(plaintext);
        
        public string Unprotect(string ciphertext)
            => protector.Unprotect(ciphertext);
    }
}
```

```csharp
// ASP.NET Core Data Protection setup
builder.Services.AddDataProtection()
    .PersistKeysToAzureBlobStorage(
        builder.Configuration["Azure:StorageConnectionString"],
        "data-protection",
        "keys.xml")
    .ProtectKeysWithAzureKeyVault(
        new Uri(builder.Configuration["Azure:KeyVaultKeyUri"]!),
        new DefaultAzureCredential())
    .SetApplicationName("MyApp")
    .SetDefaultKeyLifetime(TimeSpan.FromDays(90));
```

---

## Step 925: Secret Management

```csharp
// Azure Key Vault integration
builder.Configuration.AddAzureKeyVault(
    new Uri(builder.Configuration["Azure:KeyVaultUri"]!),
    new DefaultAzureCredential());

// HashiCorp Vault
builder.Configuration.AddVaultConfiguration(opts =>
{
    opts.VaultServerUriWithPort = "https://vault.example.com:8200";
    opts.Token = Environment.GetEnvironmentVariable("VAULT_TOKEN");
    opts.MountPoint = "secret";
    opts.VaultSecretMappings.Add(new VaultSecretMapping
    {
        VaultSecretPath = "myapp/database",
        ConfigurationKey = "ConnectionStrings:Default"
    });
});

// User Secrets (Development only)
// dotnet user-secrets set "Jwt:Secret" "my-secret-key"
// Stored in: ~/.microsoft/usersecrets/<project-id>/secrets.json
if (builder.Environment.IsDevelopment())
    builder.Configuration.AddUserSecrets<Program>();
```

---

## Step 926: Authorization Policies

```csharp
// Policy-based authorization
builder.Services.AddAuthorization(opts =>
{
    // Require claim
    opts.AddPolicy("AdminOnly", policy =>
        policy.RequireRole("Admin"));
    
    // Require claim value
    opts.AddPolicy("PremiumUser", policy =>
        policy.RequireClaim("subscription", "premium", "enterprise"));
    
    // Custom requirement
    opts.AddPolicy("MinAge18", policy =>
        policy.Requirements.Add(new MinimumAgeRequirement(18)));
    
    // Multi-factor
    opts.AddPolicy("SensitiveOperation", policy =>
    {
        policy.RequireAuthenticatedUser();
        policy.RequireClaim("mfa", "true");
        policy.RequireRole("Admin", "Manager");
    });
    
    // Resource-based (handled in handler, not here)
    opts.AddPolicy("CanEditProduct", policy =>
        policy.Requirements.Add(new ResourceOwnerRequirement()));
});

// Custom requirement + handler
public class MinimumAgeRequirement(int minimumAge) : IAuthorizationRequirement
{
    public int MinimumAge { get; } = minimumAge;
}

public class MinimumAgeHandler : AuthorizationHandler<MinimumAgeRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        MinimumAgeRequirement requirement)
    {
        var dateOfBirthClaim = context.User.FindFirst("date_of_birth");
        
        if (dateOfBirthClaim == null ||
            !DateOnly.TryParse(dateOfBirthClaim.Value, out var dob))
        {
            return Task.CompletedTask; // fail silently
        }
        
        var age = DateTime.Today.Year - dob.Year;
        if (DateTime.Today < dob.AddYears(age)) age--;
        
        if (age >= requirement.MinimumAge)
            context.Succeed(requirement);
        
        return Task.CompletedTask;
    }
}

builder.Services.AddSingleton<IAuthorizationHandler, MinimumAgeHandler>();
```

---

## Step 927: Rate Limiting

```csharp
using Microsoft.AspNetCore.RateLimiting;
using System.Threading.RateLimiting;

builder.Services.AddRateLimiter(opts =>
{
    // Fixed window - limit per time window
    opts.AddFixedWindowLimiter("api-fixed", limiterOpts =>
    {
        limiterOpts.Window = TimeSpan.FromMinutes(1);
        limiterOpts.PermitLimit = 60;
        limiterOpts.QueueLimit = 0;
        limiterOpts.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
    });
    
    // Sliding window - smoother distribution
    opts.AddSlidingWindowLimiter("api-sliding", limiterOpts =>
    {
        limiterOpts.Window = TimeSpan.FromMinutes(1);
        limiterOpts.SegmentsPerWindow = 6; // 10-second segments
        limiterOpts.PermitLimit = 60;
    });
    
    // Token bucket - handle burst traffic
    opts.AddTokenBucketLimiter("api-burst", limiterOpts =>
    {
        limiterOpts.TokenLimit = 100;
        limiterOpts.QueueLimit = 20;
        limiterOpts.ReplenishmentPeriod = TimeSpan.FromSeconds(10);
        limiterOpts.TokensPerPeriod = 20;
        limiterOpts.AutoReplenishment = true;
    });
    
    // Concurrency limiter
    opts.AddConcurrencyLimiter("concurrent", limiterOpts =>
    {
        limiterOpts.PermitLimit = 10;
        limiterOpts.QueueLimit = 100;
    });
    
    // Per-user rate limiting
    opts.AddPolicy("per-user", context =>
    {
        var userId = context.User.FindFirst(ClaimTypes.NameIdentifier)?.Value
            ?? context.Connection.RemoteIpAddress?.ToString()
            ?? "anonymous";
        
        return RateLimitPartition.GetFixedWindowLimiter(userId, _ =>
            new FixedWindowRateLimiterOptions
            {
                Window = TimeSpan.FromMinutes(1),
                PermitLimit = 100
            });
    });
    
    opts.OnRejected = async (context, ct) =>
    {
        context.HttpContext.Response.StatusCode = StatusCodes.Status429TooManyRequests;
        
        if (context.Lease.TryGetMetadata(MetadataName.RetryAfter, out var retryAfter))
        {
            context.HttpContext.Response.Headers.RetryAfter = 
                ((int)retryAfter.TotalSeconds).ToString();
        }
        
        await context.HttpContext.Response.WriteAsync(
            "Too many requests. Please try again later.", ct);
    };
});

app.UseRateLimiter();

// Apply to routes
app.MapGet("/api/products", GetProducts)
    .RequireRateLimiting("per-user");

app.MapPost("/api/auth/login", Login)
    .RequireRateLimiting("api-fixed");
```

---

## Step 928: Security Headers

```csharp
// Comprehensive security headers middleware
public class SecurityHeadersMiddleware(RequestDelegate next)
{
    private static readonly string[] _removeHeaders = 
        ["Server", "X-Powered-By", "X-AspNet-Version"];
    
    public async Task InvokeAsync(HttpContext context)
    {
        // Remove information disclosure headers
        foreach (var header in _removeHeaders)
            context.Response.Headers.Remove(header);
        
        // Add security headers
        context.Response.Headers["X-Content-Type-Options"] = "nosniff";
        context.Response.Headers["X-Frame-Options"] = "DENY";
        context.Response.Headers["X-XSS-Protection"] = "0"; // deprecated but safe
        context.Response.Headers["Referrer-Policy"] = "strict-origin-when-cross-origin";
        context.Response.Headers["Permissions-Policy"] = 
            "accelerometer=(), camera=(), geolocation=(), gyroscope=(), magnetometer=(), microphone=(), payment=(), usb=()";
        
        if (context.Request.IsHttps)
        {
            context.Response.Headers["Strict-Transport-Security"] = 
                "max-age=31536000; includeSubDomains; preload";
        }
        
        await next(context);
    }
}

app.UseMiddleware<SecurityHeadersMiddleware>();
```

---

## Step 929: Audit Logging

```csharp
// Audit trail
public class AuditEntry
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public string UserId { get; set; } = string.Empty;
    public string UserEmail { get; set; } = string.Empty;
    public string Action { get; set; } = string.Empty;
    public string EntityType { get; set; } = string.Empty;
    public string EntityId { get; set; } = string.Empty;
    public string? OldValues { get; set; }
    public string? NewValues { get; set; }
    public string IpAddress { get; set; } = string.Empty;
    public string UserAgent { get; set; } = string.Empty;
    public DateTime Timestamp { get; set; } = DateTime.UtcNow;
    public bool Success { get; set; }
}

// EF Core Interceptor for automatic auditing
public class AuditSaveChangesInterceptor(
    ICurrentUserService currentUser,
    ILogger<AuditSaveChangesInterceptor> logger)
    : SaveChangesInterceptor
{
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData,
        InterceptionResult<int> result,
        CancellationToken ct)
    {
        var db = eventData.Context;
        if (db == null) return ValueTask.FromResult(result);
        
        var entries = db.ChangeTracker.Entries()
            .Where(e => e.State is EntityState.Added or 
                              EntityState.Modified or 
                              EntityState.Deleted)
            .ToList();
        
        foreach (var entry in entries)
        {
            var audit = new AuditEntry
            {
                UserId = currentUser.UserId ?? "system",
                UserEmail = currentUser.UserEmail ?? "system",
                Action = entry.State.ToString(),
                EntityType = entry.Entity.GetType().Name,
                EntityId = GetEntityId(entry),
                OldValues = entry.State == EntityState.Modified
                    ? JsonSerializer.Serialize(
                        entry.OriginalValues.Properties
                            .ToDictionary(p => p.Name, p => entry.OriginalValues[p]))
                    : null,
                NewValues = entry.State != EntityState.Deleted
                    ? JsonSerializer.Serialize(
                        entry.CurrentValues.Properties
                            .ToDictionary(p => p.Name, p => entry.CurrentValues[p]))
                    : null
            };
            
            logger.LogInformation(
                "Audit: {Action} {EntityType} {EntityId} by {UserId}",
                audit.Action, audit.EntityType, audit.EntityId, audit.UserId);
            
            db.Set<AuditEntry>().Add(audit);
        }
        
        return ValueTask.FromResult(result);
    }
    
    private static string GetEntityId(EntityEntry entry)
    {
        var keyValues = entry.Metadata.FindPrimaryKey()?.Properties
            .Select(p => entry.Property(p.Name).CurrentValue?.ToString())
            .Where(v => v != null);
        
        return string.Join(",", keyValues ?? []);
    }
}
```

---

## Step 930: HTTPS and TLS

```csharp
// Force HTTPS
builder.Services.AddHttpsRedirection(opts =>
{
    opts.RedirectStatusCode = StatusCodes.Status308PermanentRedirect;
    opts.HttpsPort = 443;
});

// HSTS
builder.Services.AddHsts(opts =>
{
    opts.Preload = true;
    opts.IncludeSubDomains = true;
    opts.MaxAge = TimeSpan.FromDays(365);
    opts.ExcludedHosts.Add("localhost");
});

if (!app.Environment.IsDevelopment())
{
    app.UseHsts();
}

app.UseHttpsRedirection();

// Certificate validation (for mutual TLS)
builder.Services.AddAuthentication()
    .AddCertificate(opts =>
    {
        opts.AllowedCertificateTypes = CertificateTypes.All;
        opts.Events = new CertificateAuthenticationEvents
        {
            OnCertificateValidated = context =>
            {
                var subject = context.ClientCertificate.Subject;
                // Validate certificate subject/issuer
                if (subject.Contains("CN=trusted-client"))
                {
                    context.Success();
                }
                else
                {
                    context.Fail("Untrusted certificate");
                }
                return Task.CompletedTask;
            }
        };
    });
```

---

## Step 931: Dependency Security

```xml
<!-- Directory.Packages.props - centralized package management -->
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>
  
  <ItemGroup>
    <!-- Pin specific versions, review regularly -->
    <PackageVersion Include="Microsoft.EntityFrameworkCore" Version="9.0.0" />
    <PackageVersion Include="System.Text.Json" Version="9.0.0" />
  </ItemGroup>
</Project>
```

```yaml
# .github/workflows/security.yml
- name: Scan for vulnerable packages
  run: |
    dotnet list package --vulnerable --include-transitive 2>&1 | tee vulnerabilities.txt
    if grep -q "has the following vulnerable packages" vulnerabilities.txt; then
      echo "Found vulnerable packages!"
      cat vulnerabilities.txt
      exit 1
    fi

# Dependabot
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: nuget
    directory: "/"
    schedule:
      interval: weekly
    open-pull-requests-limit: 10
    ignore:
      - dependency-name: "*"
        update-types: ["version-update:semver-major"]
```

---

## Step 932: Sensitive Data Handling

```csharp
// Never log sensitive data
public class SafeLogger(ILogger<SafeLogger> logger)
{
    public void LogUserAction(string userId, string action, string? email = null)
    {
        // OK: log user ID and action
        logger.LogInformation("User {UserId} performed {Action}", userId, action);
        
        // NOT OK: log email, password, card numbers
        // logger.LogInformation("User {Email} logged in", email);
    }
}

// Mask sensitive data in responses
public class MaskedData
{
    [JsonConverter(typeof(MaskedStringJsonConverter))]
    public string? CreditCard { get; set; }
    
    [JsonConverter(typeof(MaskedEmailJsonConverter))]
    public string? Email { get; set; }
}

public class MaskedStringJsonConverter : JsonConverter<string>
{
    public override string? Read(ref Utf8JsonReader reader, Type typeToConvert, 
        JsonSerializerOptions options)
        => reader.GetString();
    
    public override void Write(Utf8JsonWriter writer, string? value, 
        JsonSerializerOptions options)
    {
        if (value == null)
        {
            writer.WriteNullValue();
            return;
        }
        
        // Show last 4 chars: ****1234
        var masked = value.Length > 4
            ? $"{"*".PadLeft(value.Length - 4, '*')}{value[^4..]}"
            : "****";
        
        writer.WriteStringValue(masked);
    }
}

// PII data - mark fields for GDPR compliance
[AttributeUsage(AttributeTargets.Property)]
public class PersonalDataAttribute : Attribute
{
    public string DataCategory { get; }
    
    public PersonalDataAttribute(string category) => DataCategory = category;
}

public class UserProfile
{
    public string Id { get; set; } = string.Empty;
    
    [PersonalData("name")]
    public string FullName { get; set; } = string.Empty;
    
    [PersonalData("contact")]
    public string Email { get; set; } = string.Empty;
    
    [PersonalData("location")]
    public string Address { get; set; } = string.Empty;
}
```

---

## Step 933: OpenID Connect / OAuth 2.0

```csharp
// ASP.NET Core Identity + OIDC
builder.Services.AddAuthentication()
    .AddOpenIdConnect("oidc", opts =>
    {
        opts.Authority = "https://identity.example.com";
        opts.ClientId = builder.Configuration["Oidc:ClientId"];
        opts.ClientSecret = builder.Configuration["Oidc:ClientSecret"];
        opts.ResponseType = OpenIdConnectResponseType.Code; // PKCE
        opts.UsePkce = true;
        
        opts.Scope.Add("openid");
        opts.Scope.Add("profile");
        opts.Scope.Add("email");
        opts.Scope.Add("api");
        
        opts.CallbackPath = "/signin-oidc";
        opts.SaveTokens = true;
        
        opts.TokenValidationParameters = new TokenValidationParameters
        {
            NameClaimType = "name",
            RoleClaimType = "role"
        };
    });

// Resource server - validate tokens from auth server
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(opts =>
    {
        opts.Authority = "https://identity.example.com";
        opts.Audience = "myapi";
        opts.TokenValidationParameters = new TokenValidationParameters
        {
            ValidTypes = new[] { "at+jwt" }
        };
    });
```

---

## Step 934: Security Testing

```csharp
// Security unit tests
[Fact]
public void CreateUser_WithWeakPassword_ShouldFail()
{
    var validator = new CreateUserRequestValidator();
    var request = new CreateUserRequest
    {
        Username = "testuser",
        Email = "test@example.com",
        Password = "password123" // weak - no special chars, uppercase
    };
    
    var result = validator.Validate(request);
    
    Assert.False(result.IsValid);
    Assert.Contains(result.Errors, e => e.PropertyName == "Password");
}

[Fact]
public void SqlParameter_ShouldPreventInjection()
{
    var maliciousInput = "'; DROP TABLE Users; --";
    var param = new SqlParameter("@Username", maliciousInput);
    
    // Verify parameter value is treated as literal string
    Assert.Equal(maliciousInput, param.Value as string);
}

[Fact]
public async Task AdminEndpoint_WithoutRole_Returns403()
{
    var client = _factory.CreateClient();
    
    // Authenticate as regular user
    var token = GenerateTestToken(roles: new[] { "User" });
    client.DefaultRequestHeaders.Authorization = 
        new AuthenticationHeaderValue("Bearer", token);
    
    var response = await client.GetAsync("/api/admin/users");
    
    Assert.Equal(HttpStatusCode.Forbidden, response.StatusCode);
}

[Theory]
[InlineData("<script>alert('xss')</script>")]
[InlineData("javascript:alert('xss')")]
[InlineData("<img src=x onerror=alert('xss')>")]
public void UserInput_ShouldBeEncoded(string maliciousInput)
{
    var encoded = HtmlEncoder.Default.Encode(maliciousInput);
    
    Assert.DoesNotContain("<script>", encoded);
    Assert.DoesNotContain("javascript:", encoded);
    Assert.True(encoded != maliciousInput);
}
```

---

## Step 935: Security Misconfiguration Prevention

```csharp
// appsettings.Production.json - never commit sensitive values
{
  "Jwt": {
    "Secret": "${JWT_SECRET}",  // From environment variable
    "Issuer": "https://api.example.com"
  },
  "ConnectionStrings": {
    "Default": "${DB_CONNECTION_STRING}"  // From secret manager
  }
}

// Production security configuration
if (!app.Environment.IsDevelopment())
{
    // Disable detailed errors
    app.UseExceptionHandler("/error");
    
    // Disable swagger in production
    // (or require auth)
}

// app.UseSwagger() protected
app.MapSwagger().RequireAuthorization("AdminOnly");

// Development ONLY endpoints
if (app.Environment.IsDevelopment())
{
    app.MapGet("/debug/config", (IConfiguration config) =>
        config.GetChildren().ToDictionary(s => s.Key, s => s.Value));
    
    app.MapGet("/debug/routes", (IEnumerable<EndpointDataSource> endpointSources) =>
        endpointSources.SelectMany(e => e.Endpoints).Select(e => e.DisplayName));
}
```

---

## Step 936-950: สรุป Security Checklist

### Security Checklist

#### Authentication & Authorization
- [ ] Strong password policy (12+ chars, complexity)
- [ ] Password hashing with Argon2/bcrypt (not MD5/SHA)
- [ ] JWT short expiry (15 min) + refresh tokens
- [ ] MFA support
- [ ] Rate limiting on login endpoints
- [ ] Account lockout after failed attempts

#### Input/Output
- [ ] Validate all inputs with FluentValidation
- [ ] Parameterized queries (no string interpolation in SQL)
- [ ] HTML encoding for output (Razor does this by default)
- [ ] File upload validation (type, size, malware scan)

#### Network Security
- [ ] HTTPS everywhere
- [ ] HSTS enabled
- [ ] CORS properly configured
- [ ] Security headers set
- [ ] Rate limiting

#### Data Protection
- [ ] Sensitive data encrypted at rest (AES-256)
- [ ] Secrets in secret manager (not in code/config)
- [ ] PII fields identified and protected
- [ ] Audit logging for sensitive operations

#### Dependencies
- [ ] `dotnet list package --vulnerable` in CI
- [ ] Dependabot configured
- [ ] No outdated packages with CVEs
- [ ] Supply chain security (lock files)

#### Monitoring
- [ ] Security events logged (login, access denied, errors)
- [ ] Alerting on suspicious patterns
- [ ] SIEM integration
- [ ] Regular penetration testing

---

*จบ Part 33: Security in .NET Applications*
*ต่อไป Part 34: Source Generators & Compile-Time Code Generation*
