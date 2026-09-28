# Part 87: Advanced Authentication — OAuth 2.0, PKCE & OpenID Connect

## Steps 2021–2036

---

## Step 2021: OAuth 2.0 Fundamentals & Grant Types

OAuth 2.0 is an authorization framework, not an authentication protocol. Understanding its grant types is critical.

```
OAuth 2.0 Roles:
┌─────────────────────────────────────────────────────────┐
│  Resource Owner  │  The user who owns the data          │
│  Client          │  The application requesting access   │
│  Authorization   │  Issues tokens (IdentityServer,      │
│  Server          │  Duende, Keycloak, Auth0, etc.)       │
│  Resource Server │  The API being protected             │
└─────────────────────────────────────────────────────────┘

Grant Types:
1. Authorization Code + PKCE  → Web apps, SPAs, mobile (RECOMMENDED)
2. Client Credentials         → Service-to-service (M2M)
3. Refresh Token              → Extends sessions without re-auth
4. Device Authorization       → CLI tools, smart TVs
5. Implicit                   → DEPRECATED
6. ROPC                       → DEPRECATED (avoid)
```

```csharp
// NuGet packages for an ASP.NET Core API (Resource Server)
// Microsoft.AspNetCore.Authentication.JwtBearer
// System.IdentityModel.Tokens.Jwt

// Program.cs — minimal JWT bearer setup
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = "https://identity.example.com";
        options.Audience  = "orders-api";

        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer           = true,
            ValidateAudience         = true,
            ValidateLifetime         = true,
            ValidateIssuerSigningKey = true,
            ClockSkew                = TimeSpan.FromSeconds(30), // tighten from default 5 min
        };

        // Support SignalR WebSocket token (from query string)
        options.Events = new JwtBearerEvents
        {
            OnMessageReceived = ctx =>
            {
                var accessToken = ctx.Request.Query["access_token"];
                var path = ctx.HttpContext.Request.Path;
                if (!string.IsNullOrEmpty(accessToken) && path.StartsWithSegments("/hubs"))
                    ctx.Token = accessToken;
                return Task.CompletedTask;
            }
        };
    });

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("orders:read",  p => p.RequireClaim("scope", "orders:read"));
    options.AddPolicy("orders:write", p => p.RequireClaim("scope", "orders:write"));
    options.AddPolicy("admin",        p => p.RequireRole("admin"));

    // Default policy requires authenticated user
    options.DefaultPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();
    options.FallbackPolicy = options.DefaultPolicy;
});
```

---

## Step 2022: Authorization Code Flow with PKCE

PKCE (Proof Key for Code Exchange) prevents authorization code interception attacks. It's mandatory for public clients (SPAs, mobile) and recommended for all clients.

```
PKCE Flow:
┌──────┐          ┌──────────┐          ┌───────────────┐
│Client│          │Auth Srv  │          │Resource Server│
└──┬───┘          └────┬─────┘          └───────┬───────┘
   │                   │                        │
   │ 1. Generate       │                        │
   │ code_verifier     │                        │
   │ code_challenge=   │                        │
   │  SHA256(verifier) │                        │
   │                   │                        │
   │ 2. /authorize?    │                        │
   │  code_challenge=..│                        │
   │  code_challenge_  │                        │
   │  method=S256      │                        │
   │ ─────────────────>│                        │
   │                   │                        │
   │ 3. User login     │                        │
   │    consent        │                        │
   │ <─────────────────│                        │
   │                   │                        │
   │ 4. Redirect with  │                        │
   │    ?code=xxx      │                        │
   │ <─────────────────│                        │
   │                   │                        │
   │ 5. /token with    │                        │
   │    code_verifier  │                        │
   │ ─────────────────>│                        │
   │                   │                        │
   │ 6. access_token + │                        │
   │    id_token +     │                        │
   │    refresh_token  │                        │
   │ <─────────────────│                        │
   │                   │                        │
   │ 7. Bearer token   │                        │
   │ ─────────────────────────────────────────>│
   └───────────────────────────────────────────┘
```

```csharp
// PKCE implementation for a web client
public static class PkceHelper
{
    public static (string Verifier, string Challenge) GeneratePkce()
    {
        // code_verifier: 43–128 chars, URL-safe base64
        var bytes = RandomNumberGenerator.GetBytes(64);
        var verifier = Base64UrlEncoder.Encode(bytes);

        // code_challenge = BASE64URL(SHA256(verifier))
        var challengeBytes = SHA256.HashData(Encoding.ASCII.GetBytes(verifier));
        var challenge = Base64UrlEncoder.Encode(challengeBytes);

        return (verifier, challenge);
    }
}

// ASP.NET Core web app — OIDC client
builder.Services.AddAuthentication(options =>
{
    options.DefaultScheme          = CookieAuthenticationDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = OpenIdConnectDefaults.AuthenticationScheme;
})
.AddCookie(options =>
{
    options.Cookie.HttpOnly  = true;
    options.Cookie.SameSite  = SameSiteMode.Lax;
    options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
    options.SlidingExpiration = true;
    options.ExpireTimeSpan   = TimeSpan.FromHours(8);
})
.AddOpenIdConnect(options =>
{
    options.Authority            = "https://identity.example.com";
    options.ClientId             = "web-client";
    options.ClientSecret         = builder.Configuration["Oidc:ClientSecret"];
    options.ResponseType         = "code"; // Authorization Code
    options.UsePkce              = true;   // Enable PKCE
    options.SaveTokens           = true;   // Persist tokens in cookie

    options.Scope.Clear();
    options.Scope.Add("openid");
    options.Scope.Add("profile");
    options.Scope.Add("email");
    options.Scope.Add("orders:read");
    options.Scope.Add("orders:write");
    options.Scope.Add("offline_access"); // refresh tokens

    options.GetClaimsFromUserInfoEndpoint = true;
    options.MapInboundClaims = false; // use raw OIDC claim names

    options.TokenValidationParameters = new TokenValidationParameters
    {
        NameClaimType  = "name",
        RoleClaimType  = "role",
        ClockSkew      = TimeSpan.FromSeconds(30)
    };

    options.Events = new OpenIdConnectEvents
    {
        OnRedirectToIdentityProvider = ctx =>
        {
            // Add custom parameters (e.g., login_hint, ui_locales)
            if (ctx.HttpContext.Items.TryGetValue("login_hint", out var hint))
                ctx.ProtocolMessage.LoginHint = hint?.ToString();
            return Task.CompletedTask;
        },
        OnTokenValidated = ctx =>
        {
            // Additional claim mapping or validation
            var email = ctx.Principal?.FindFirstValue("email");
            if (email?.EndsWith("@blocked.com") == true)
                ctx.Fail("Domain not allowed");
            return Task.CompletedTask;
        }
    };
});
```

---

## Step 2023: Client Credentials — Machine-to-Machine Authentication

```csharp
// Service-to-service using client credentials
// NuGet: Duende.AccessTokenManagement (handles token caching + refresh)

builder.Services.AddClientCredentialsTokenManagement()
    .AddClient("inventory-api", client =>
    {
        client.TokenEndpoint = "https://identity.example.com/connect/token";
        client.ClientId      = "orders-service";
        client.ClientSecret  = builder.Configuration["Services:ClientSecret"];
        client.Scope         = "inventory:read inventory:write";
    });

builder.Services.AddClientCredentialsHttpClient<IInventoryClient, InventoryClient>(
    "inventory-api",
    client => client.BaseAddress = new Uri("https://inventory.internal/"));

// The HTTP client automatically acquires and caches the token
public class InventoryClient : IInventoryClient
{
    private readonly HttpClient _http;

    public InventoryClient(HttpClient http) => _http = http;

    public async Task<StockLevel> GetStockAsync(Guid productId, CancellationToken ct)
    {
        var response = await _http.GetAsync($"api/inventory/{productId}", ct);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<StockLevel>(ct)
               ?? throw new InvalidOperationException("Null response");
    }
}
```

```csharp
// Manual client credentials (without Duende.AccessTokenManagement)
public class TokenCacheService : ITokenCacheService
{
    private readonly IMemoryCache _cache;
    private readonly IHttpClientFactory _factory;
    private readonly IConfiguration _config;

    public TokenCacheService(IMemoryCache cache, IHttpClientFactory factory, IConfiguration config)
    {
        _cache   = cache;
        _factory = factory;
        _config  = config;
    }

    public async Task<string> GetAccessTokenAsync(string clientName, CancellationToken ct = default)
    {
        var cacheKey = $"token:{clientName}";

        if (_cache.TryGetValue(cacheKey, out string? cached))
            return cached!;

        var token = await FetchTokenAsync(clientName, ct);

        // Cache with 30s buffer before actual expiry
        var expiry = DateTimeOffset.UtcNow.AddSeconds(token.ExpiresIn - 30);
        _cache.Set(cacheKey, token.AccessToken, expiry);

        return token.AccessToken;
    }

    private async Task<TokenResponse> FetchTokenAsync(string clientName, CancellationToken ct)
    {
        var http = _factory.CreateClient();
        var section = _config.GetRequiredSection($"Services:{clientName}");

        var request = new HttpRequestMessage(HttpMethod.Post, section["TokenEndpoint"])
        {
            Content = new FormUrlEncodedContent(new Dictionary<string, string>
            {
                ["grant_type"]    = "client_credentials",
                ["client_id"]     = section["ClientId"]!,
                ["client_secret"] = section["ClientSecret"]!,
                ["scope"]         = section["Scope"]!
            })
        };

        var response = await http.SendAsync(request, ct);
        response.EnsureSuccessStatusCode();

        return await response.Content.ReadFromJsonAsync<TokenResponse>(ct)
               ?? throw new InvalidOperationException();
    }
}
```

---

## Step 2024: OpenIddict — Open Source Identity Server

OpenIddict is a flexible, developer-friendly OIDC server that integrates with ASP.NET Core Identity.

```bash
dotnet new web -n IdentityServer
dotnet add package OpenIddict.AspNetCore
dotnet add package OpenIddict.EntityFrameworkCore
dotnet add package Microsoft.AspNetCore.Identity.EntityFrameworkCore
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL
```

```csharp
// Program.cs — OpenIddict setup
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddDbContext<ApplicationDbContext>(options =>
{
    options.UseNpgsql(builder.Configuration.GetConnectionString("Default"));
    options.UseOpenIddict(); // adds OpenIddict tables
});

builder.Services
    .AddIdentity<ApplicationUser, IdentityRole>(options =>
    {
        options.Password.RequiredLength         = 12;
        options.Password.RequireNonAlphanumeric = true;
        options.Lockout.MaxFailedAccessAttempts = 5;
        options.Lockout.DefaultLockoutTimeSpan  = TimeSpan.FromMinutes(15);
    })
    .AddEntityFrameworkStores<ApplicationDbContext>()
    .AddDefaultTokenProviders();

builder.Services.AddOpenIddict()
    .AddCore(options =>
    {
        options.UseEntityFrameworkCore()
               .UseDbContext<ApplicationDbContext>();
    })
    .AddServer(options =>
    {
        // Endpoints
        options.SetAuthorizationEndpointUris("/connect/authorize")
               .SetTokenEndpointUris("/connect/token")
               .SetUserInfoEndpointUris("/connect/userinfo")
               .SetIntrospectionEndpointUris("/connect/introspect")
               .SetRevocationEndpointUris("/connect/revoke")
               .SetLogoutEndpointUris("/connect/logout");

        // Flows
        options.AllowAuthorizationCodeFlow()
               .RequireProofKeyForCodeExchange()
               .AllowClientCredentialsFlow()
               .AllowRefreshTokenFlow();

        // Signing & encryption keys
        if (builder.Environment.IsDevelopment())
        {
            options.AddDevelopmentEncryptionCertificate()
                   .AddDevelopmentSigningCertificate();
        }
        else
        {
            // Load from certificate store or Key Vault
            options.AddEncryptionCertificate(LoadCertificate("encryption"))
                   .AddSigningCertificate(LoadCertificate("signing"));
        }

        options.UseAspNetCore()
               .EnableAuthorizationEndpointPassthrough()
               .EnableTokenEndpointPassthrough()
               .EnableUserInfoEndpointPassthrough()
               .EnableLogoutEndpointPassthrough();

        // Token lifetimes
        options.SetAccessTokenLifetime(TimeSpan.FromMinutes(15));
        options.SetRefreshTokenLifetime(TimeSpan.FromDays(30));
        options.SetAuthorizationCodeLifetime(TimeSpan.FromMinutes(5));
    })
    .AddValidation(options =>
    {
        options.UseLocalServer();
        options.UseAspNetCore();
    });

builder.Services.AddAuthentication(options =>
{
    options.DefaultScheme          = CookieAuthenticationDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = OpenIdConnectDefaults.AuthenticationScheme;
})
.AddCookie();
```

---

## Step 2025: OpenIddict — Authorization Endpoint

```csharp
// AuthorizationController.cs
[ApiController, Route("connect")]
public class AuthorizationController : Controller
{
    private readonly IOpenIddictApplicationManager _applicationManager;
    private readonly IOpenIddictAuthorizationManager _authorizationManager;
    private readonly IOpenIddictScopeManager _scopeManager;
    private readonly SignInManager<ApplicationUser> _signInManager;
    private readonly UserManager<ApplicationUser> _userManager;

    public AuthorizationController(
        IOpenIddictApplicationManager applicationManager,
        IOpenIddictAuthorizationManager authorizationManager,
        IOpenIddictScopeManager scopeManager,
        SignInManager<ApplicationUser> signInManager,
        UserManager<ApplicationUser> userManager)
    {
        _applicationManager   = applicationManager;
        _authorizationManager = authorizationManager;
        _scopeManager         = scopeManager;
        _signInManager        = signInManager;
        _userManager          = userManager;
    }

    [HttpGet("authorize"), HttpPost("authorize")]
    public async Task<IActionResult> Authorize()
    {
        var request = HttpContext.GetOpenIddictServerRequest()
                      ?? throw new InvalidOperationException("No OIDC request.");

        // If user is not logged in, redirect to login
        var result = await HttpContext.AuthenticateAsync(CookieAuthenticationDefaults.AuthenticationScheme);
        if (!result.Succeeded)
        {
            return Challenge(new AuthenticationProperties
            {
                RedirectUri = Request.PathBase + Request.Path + QueryString.Create(
                    Request.HasFormContentType ? Request.Form.ToList() : Request.Query.ToList())
            }, CookieAuthenticationDefaults.AuthenticationScheme);
        }

        var user = await _userManager.GetUserAsync(result.Principal)
                   ?? throw new InvalidOperationException("User not found.");

        var application = await _applicationManager.FindByClientIdAsync(request.ClientId!)
                          ?? throw new InvalidOperationException("Client not found.");

        // Check existing authorizations (skip consent if already granted)
        var authorizations = await _authorizationManager.FindAsync(
            subject: await _userManager.GetUserIdAsync(user),
            client:  await _applicationManager.GetIdAsync(application) ?? string.Empty,
            status:  Statuses.Valid,
            type:    AuthorizationTypes.Permanent,
            scopes:  request.GetScopes()).ToListAsync();

        var consentType = await _applicationManager.GetConsentTypeAsync(application);

        if (consentType == ConsentTypes.Implicit || authorizations.Count > 0)
        {
            return await CreateSignInResultAsync(user, request, application, authorizations);
        }

        // Show consent page
        return View("Consent", new ConsentViewModel
        {
            ApplicationName = await _applicationManager.GetLocalizedDisplayNameAsync(application) ?? string.Empty,
            Scopes          = request.GetScopes().ToList()
        });
    }

    [HttpPost("authorize"), ActionName("Authorize"), FormValueRequired("submit.Accept")]
    public async Task<IActionResult> Accept()
    {
        var request = HttpContext.GetOpenIddictServerRequest()
                      ?? throw new InvalidOperationException("No OIDC request.");

        var user = await _userManager.GetUserAsync(User)
                   ?? throw new InvalidOperationException("User not found.");

        var application = await _applicationManager.FindByClientIdAsync(request.ClientId!)
                          ?? throw new InvalidOperationException("Client not found.");

        var authorizations = await _authorizationManager.FindAsync(
            subject: await _userManager.GetUserIdAsync(user),
            client:  await _applicationManager.GetIdAsync(application) ?? string.Empty,
            status:  Statuses.Valid,
            type:    AuthorizationTypes.Permanent,
            scopes:  request.GetScopes()).ToListAsync();

        return await CreateSignInResultAsync(user, request, application, authorizations);
    }

    private async Task<IActionResult> CreateSignInResultAsync(
        ApplicationUser user,
        OpenIddictRequest request,
        object application,
        List<object> existingAuthorizations)
    {
        var identity = new ClaimsIdentity(
            authenticationType: TokenValidationParameters.DefaultAuthenticationType,
            nameType:  Claims.Name,
            roleType:  Claims.Role);

        // Standard OIDC claims
        identity.SetClaim(Claims.Subject, await _userManager.GetUserIdAsync(user))
                .SetClaim(Claims.Email,   await _userManager.GetEmailAsync(user))
                .SetClaim(Claims.Name,    await _userManager.GetUserNameAsync(user));

        // Roles as claims
        var roles = await _userManager.GetRolesAsync(user);
        foreach (var role in roles)
            identity.AddClaim(new Claim(Claims.Role, role));

        // Custom claims
        var customClaims = await _userManager.GetClaimsAsync(user);
        foreach (var claim in customClaims)
            identity.AddClaim(claim);

        identity.SetScopes(request.GetScopes());

        // Determine destinations (which token types get which claims)
        identity.SetDestinations(GetDestinations);

        // Create or reuse authorization
        var authorization = existingAuthorizations.LastOrDefault()
            ?? await _authorizationManager.CreateAsync(
                identity:    new ClaimsPrincipal(identity),
                subject:     await _userManager.GetUserIdAsync(user),
                client:      await _applicationManager.GetIdAsync(application) ?? string.Empty,
                type:        AuthorizationTypes.Permanent,
                scopes:      identity.GetScopes());

        identity.SetAuthorizationId(await _authorizationManager.GetIdAsync(authorization));

        return SignIn(new ClaimsPrincipal(identity), OpenIddictServerAspNetCoreDefaults.AuthenticationScheme);
    }

    private static IEnumerable<string> GetDestinations(Claim claim)
    {
        switch (claim.Type)
        {
            case Claims.Name:
            case Claims.PreferredUsername:
                yield return Destinations.AccessToken;
                if (claim.Subject!.HasScope(Scopes.Profile))
                    yield return Destinations.IdentityToken;
                yield break;

            case Claims.Email:
                yield return Destinations.AccessToken;
                if (claim.Subject!.HasScope(Scopes.Email))
                    yield return Destinations.IdentityToken;
                yield break;

            case Claims.Role:
                yield return Destinations.AccessToken;
                if (claim.Subject!.HasScope(Scopes.Roles))
                    yield return Destinations.IdentityToken;
                yield break;

            default:
                yield return Destinations.AccessToken;
                yield break;
        }
    }
}
```

---

## Step 2026: OpenIddict — Token Endpoint (Client Credentials & Refresh)

```csharp
// TokenController.cs
[ApiController, Route("connect")]
public class TokenController : Controller
{
    private readonly UserManager<ApplicationUser> _userManager;
    private readonly SignInManager<ApplicationUser> _signInManager;

    public TokenController(UserManager<ApplicationUser> userManager, SignInManager<ApplicationUser> signInManager)
    {
        _userManager   = userManager;
        _signInManager = signInManager;
    }

    [HttpPost("token")]
    public async Task<IActionResult> Exchange()
    {
        var request = HttpContext.GetOpenIddictServerRequest()
                      ?? throw new InvalidOperationException("No OIDC request.");

        if (request.IsAuthorizationCodeGrantType() || request.IsDeviceCodeGrantType())
        {
            // The authorization code has already been validated by OpenIddict.
            // Retrieve the ClaimsPrincipal stored in the authorization code/device code.
            var result = await HttpContext.AuthenticateAsync(OpenIddictServerAspNetCoreDefaults.AuthenticationScheme);

            var user = await _userManager.FindByIdAsync(result.Principal!.GetClaim(Claims.Subject)!);
            if (user is null)
                return Forbid(OpenIddictServerAspNetCoreDefaults.AuthenticationScheme);

            // Verify account is still active
            if (!await _signInManager.CanSignInAsync(user))
                return Forbid(OpenIddictServerAspNetCoreDefaults.AuthenticationScheme);

            var identity = result.Principal!.Identity as ClaimsIdentity;
            identity!.SetDestinations(GetDestinations);

            return SignIn(result.Principal!, OpenIddictServerAspNetCoreDefaults.AuthenticationScheme);
        }

        if (request.IsClientCredentialsGrantType())
        {
            // Service-to-service: no user context
            var identity = new ClaimsIdentity(
                authenticationType: TokenValidationParameters.DefaultAuthenticationType,
                nameType:  Claims.Name,
                roleType:  Claims.Role);

            identity.SetClaim(Claims.Subject, request.ClientId!)
                    .SetClaim(Claims.ClientId, request.ClientId!);

            identity.SetScopes(request.GetScopes());
            identity.SetDestinations(_ => [Destinations.AccessToken]);

            return SignIn(new ClaimsPrincipal(identity), OpenIddictServerAspNetCoreDefaults.AuthenticationScheme);
        }

        if (request.IsRefreshTokenGrantType())
        {
            var result = await HttpContext.AuthenticateAsync(OpenIddictServerAspNetCoreDefaults.AuthenticationScheme);

            var user = await _userManager.FindByIdAsync(result.Principal!.GetClaim(Claims.Subject)!);
            if (user is null)
                return Forbid(OpenIddictServerAspNetCoreDefaults.AuthenticationScheme);

            // Re-validate user can still sign in
            if (!await _signInManager.CanSignInAsync(user))
                return Forbid(OpenIddictServerAspNetCoreDefaults.AuthenticationScheme);

            // Refresh claims in case user data changed
            var identity = result.Principal!.Identity as ClaimsIdentity;
            identity!.SetClaim(Claims.Email, await _userManager.GetEmailAsync(user));

            var roles = await _userManager.GetRolesAsync(user);
            identity.RemoveClaims(Claims.Role);
            foreach (var role in roles)
                identity.AddClaim(new Claim(Claims.Role, role));

            identity.SetDestinations(GetDestinations);

            return SignIn(result.Principal!, OpenIddictServerAspNetCoreDefaults.AuthenticationScheme);
        }

        throw new InvalidOperationException($"Unsupported grant type: {request.GrantType}");
    }

    private static IEnumerable<string> GetDestinations(Claim claim) =>
        claim.Type switch
        {
            Claims.Name or Claims.Email => [Destinations.AccessToken, Destinations.IdentityToken],
            Claims.Role                 => [Destinations.AccessToken, Destinations.IdentityToken],
            _                           => [Destinations.AccessToken]
        };
}
```

---

## Step 2027: OpenIddict — Registering Applications (Clients)

```csharp
// Worker that seeds OpenIddict clients on startup
public class OpenIddictSeeder : IHostedService
{
    private readonly IServiceProvider _sp;

    public OpenIddictSeeder(IServiceProvider sp) => _sp = sp;

    public async Task StartAsync(CancellationToken ct)
    {
        await using var scope = _sp.CreateAsyncScope();
        var manager = scope.ServiceProvider.GetRequiredService<IOpenIddictApplicationManager>();

        // Web application (Authorization Code + PKCE)
        if (await manager.FindByClientIdAsync("web-client", ct) is null)
        {
            await manager.CreateAsync(new OpenIddictApplicationDescriptor
            {
                ClientId          = "web-client",
                ClientSecret      = "super-secret-web",
                ConsentType       = ConsentTypes.Implicit,
                DisplayName       = "Web Application",
                RedirectUris      = { new Uri("https://app.example.com/signin-oidc") },
                PostLogoutRedirectUris = { new Uri("https://app.example.com/signout-callback-oidc") },
                Permissions =
                {
                    Permissions.Endpoints.Authorization,
                    Permissions.Endpoints.Token,
                    Permissions.Endpoints.Logout,
                    Permissions.GrantTypes.AuthorizationCode,
                    Permissions.GrantTypes.RefreshToken,
                    Permissions.ResponseTypes.Code,
                    Permissions.Scopes.Email,
                    Permissions.Scopes.Profile,
                    Permissions.Scopes.Roles,
                    Permissions.Prefixes.Scope + "orders:read",
                    Permissions.Prefixes.Scope + "orders:write",
                }
            }, ct);
        }

        // SPA (Authorization Code + PKCE, no secret)
        if (await manager.FindByClientIdAsync("spa-client", ct) is null)
        {
            await manager.CreateAsync(new OpenIddictApplicationDescriptor
            {
                ClientId    = "spa-client",
                ClientType  = ClientTypes.Public, // no secret
                ConsentType = ConsentTypes.Implicit,
                DisplayName = "Single Page Application",
                RedirectUris =
                {
                    new Uri("http://localhost:5173/callback"),
                    new Uri("https://app.example.com/callback")
                },
                Permissions =
                {
                    Permissions.Endpoints.Authorization,
                    Permissions.Endpoints.Token,
                    Permissions.GrantTypes.AuthorizationCode,
                    Permissions.ResponseTypes.Code,
                    Permissions.Scopes.Email,
                    Permissions.Scopes.Profile,
                    Permissions.Prefixes.Scope + "orders:read",
                }
            }, ct);
        }

        // Service-to-service (Client Credentials)
        if (await manager.FindByClientIdAsync("orders-service", ct) is null)
        {
            await manager.CreateAsync(new OpenIddictApplicationDescriptor
            {
                ClientId     = "orders-service",
                ClientSecret = "orders-service-secret",
                DisplayName  = "Orders Microservice",
                Permissions  =
                {
                    Permissions.Endpoints.Token,
                    Permissions.GrantTypes.ClientCredentials,
                    Permissions.Prefixes.Scope + "inventory:read",
                    Permissions.Prefixes.Scope + "inventory:write",
                }
            }, ct);
        }
    }

    public Task StopAsync(CancellationToken ct) => Task.CompletedTask;
}

// Registering custom scopes
public class ScopeSeeder : IHostedService
{
    private readonly IServiceProvider _sp;

    public ScopeSeeder(IServiceProvider sp) => _sp = sp;

    public async Task StartAsync(CancellationToken ct)
    {
        await using var scope = _sp.CreateAsyncScope();
        var manager = scope.ServiceProvider.GetRequiredService<IOpenIddictScopeManager>();

        foreach (var (name, description) in new[]
        {
            ("orders:read",    "Read access to orders"),
            ("orders:write",   "Write access to orders"),
            ("inventory:read", "Read access to inventory"),
            ("inventory:write","Write access to inventory"),
        })
        {
            if (await manager.FindByNameAsync(name, ct) is null)
            {
                await manager.CreateAsync(new OpenIddictScopeDescriptor
                {
                    Name        = name,
                    Description = description,
                    Resources   = { "orders-api" }
                }, ct);
            }
        }
    }

    public Task StopAsync(CancellationToken ct) => Task.CompletedTask;
}
```

---

## Step 2028: Token Introspection & Validation

```csharp
// Resource server validating tokens via introspection
// (for opaque tokens; JWT validation is done locally)
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = "https://identity.example.com";
        options.Audience  = "orders-api";

        // Use introspection for token validation (calls /connect/introspect)
        options.Events = new JwtBearerEvents
        {
            OnTokenValidated = async ctx =>
            {
                // Additional validation: check if token is revoked
                var introspector = ctx.HttpContext.RequestServices
                    .GetRequiredService<ITokenIntrospector>();

                var token = ctx.SecurityToken as JwtSecurityToken;
                var isValid = await introspector.IsActiveAsync(token?.RawData ?? string.Empty);

                if (!isValid)
                    ctx.Fail("Token has been revoked");
            }
        };
    });

// Custom introspection for opaque tokens
public interface ITokenIntrospector
{
    Task<bool> IsActiveAsync(string token, CancellationToken ct = default);
}

public class OpenIddictTokenIntrospector : ITokenIntrospector
{
    private readonly HttpClient _http;
    private readonly IConfiguration _config;

    public OpenIddictTokenIntrospector(HttpClient http, IConfiguration config)
    {
        _http   = http;
        _config = config;
    }

    public async Task<bool> IsActiveAsync(string token, CancellationToken ct = default)
    {
        var request = new HttpRequestMessage(HttpMethod.Post, "/connect/introspect")
        {
            Content = new FormUrlEncodedContent(new Dictionary<string, string>
            {
                ["token"]         = token,
                ["token_type_hint"] = "access_token"
            })
        };

        // Authenticate with the introspection endpoint
        var clientId     = _config["Introspection:ClientId"]!;
        var clientSecret = _config["Introspection:ClientSecret"]!;
        var credentials  = Convert.ToBase64String(Encoding.UTF8.GetBytes($"{clientId}:{clientSecret}"));
        request.Headers.Authorization = new AuthenticationHeaderValue("Basic", credentials);

        var response = await _http.SendAsync(request, ct);
        if (!response.IsSuccessStatusCode) return false;

        var json = await response.Content.ReadFromJsonAsync<IntrospectionResponse>(ct);
        return json?.Active == true;
    }
}

public record IntrospectionResponse([property: JsonPropertyName("active")] bool Active);
```

---

## Step 2029: Refresh Token Rotation

```csharp
// Token service with refresh token rotation
public class TokenService : ITokenService
{
    private readonly IHttpClientFactory _factory;
    private readonly IOptionsMonitor<AuthOptions> _options;
    private readonly IMemoryCache _cache;

    public TokenService(IHttpClientFactory factory, IOptionsMonitor<AuthOptions> options, IMemoryCache cache)
    {
        _factory = factory;
        _options = options;
        _cache   = cache;
    }

    public async Task<TokenPair> RefreshAsync(string refreshToken, CancellationToken ct = default)
    {
        var opts = _options.CurrentValue;
        var http = _factory.CreateClient("identity");

        var response = await http.PostAsync("/connect/token",
            new FormUrlEncodedContent(new Dictionary<string, string>
            {
                ["grant_type"]    = "refresh_token",
                ["refresh_token"] = refreshToken,
                ["client_id"]     = opts.ClientId,
                ["client_secret"] = opts.ClientSecret
            }), ct);

        if (!response.IsSuccessStatusCode)
        {
            // Refresh token is invalid/expired
            var error = await response.Content.ReadFromJsonAsync<OAuthError>(ct);
            throw new TokenRefreshException(error?.ErrorDescription ?? "Refresh failed");
        }

        var tokenResponse = await response.Content.ReadFromJsonAsync<TokenResponse>(ct)
                            ?? throw new InvalidOperationException();

        // Revoke old refresh token (rotation)
        await RevokeAsync(refreshToken, ct);

        return new TokenPair(tokenResponse.AccessToken, tokenResponse.RefreshToken!);
    }

    private async Task RevokeAsync(string token, CancellationToken ct)
    {
        var opts = _options.CurrentValue;
        var http = _factory.CreateClient("identity");

        await http.PostAsync("/connect/revoke",
            new FormUrlEncodedContent(new Dictionary<string, string>
            {
                ["token"]         = token,
                ["token_type_hint"] = "refresh_token",
                ["client_id"]     = opts.ClientId,
                ["client_secret"] = opts.ClientSecret
            }), ct);
    }
}

public record TokenPair(string AccessToken, string RefreshToken);
public record TokenResponse(
    [property: JsonPropertyName("access_token")]  string AccessToken,
    [property: JsonPropertyName("refresh_token")] string? RefreshToken,
    [property: JsonPropertyName("expires_in")]    int ExpiresIn,
    [property: JsonPropertyName("token_type")]    string TokenType);

// Middleware to auto-refresh expired tokens in a web app
public class AutoTokenRefreshMiddleware
{
    private readonly RequestDelegate _next;

    public AutoTokenRefreshMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(HttpContext ctx, ITokenService tokenService)
    {
        var accessToken  = await ctx.GetTokenAsync("access_token");
        var refreshToken = await ctx.GetTokenAsync("refresh_token");
        var expiresAtStr = await ctx.GetTokenAsync("expires_at");

        if (accessToken is not null && refreshToken is not null && expiresAtStr is not null)
        {
            var expiresAt = DateTimeOffset.Parse(expiresAtStr);

            if (expiresAt < DateTimeOffset.UtcNow.AddMinutes(2))
            {
                try
                {
                    var newTokens = await tokenService.RefreshAsync(refreshToken);

                    // Update the tokens in the cookie
                    var authResult = await ctx.AuthenticateAsync();
                    var props = authResult.Properties!;
                    props.UpdateTokenValue("access_token",  newTokens.AccessToken);
                    props.UpdateTokenValue("refresh_token", newTokens.RefreshToken);
                    props.UpdateTokenValue("expires_at",
                        DateTimeOffset.UtcNow.AddMinutes(15).ToString("o"));

                    await ctx.SignInAsync(authResult.Principal!, props);
                }
                catch (TokenRefreshException)
                {
                    await ctx.SignOutAsync();
                    ctx.Response.Redirect("/");
                    return;
                }
            }
        }

        await _next(ctx);
    }
}
```

---

## Step 2030: Multi-Factor Authentication

```csharp
// TOTP-based 2FA with ASP.NET Core Identity
public class MfaService : IMfaService
{
    private readonly UserManager<ApplicationUser> _userManager;
    private readonly SignInManager<ApplicationUser> _signInManager;

    public MfaService(UserManager<ApplicationUser> userManager, SignInManager<ApplicationUser> signInManager)
    {
        _userManager   = userManager;
        _signInManager = signInManager;
    }

    public async Task<MfaSetupResult> SetupTotpAsync(ApplicationUser user)
    {
        // Reset any existing key
        await _userManager.ResetAuthenticatorKeyAsync(user);
        var key = await _userManager.GetAuthenticatorKeyAsync(user);

        var email         = await _userManager.GetEmailAsync(user) ?? user.UserName!;
        var issuer        = "MyApp";
        var totpUri       = GenerateTotpUri(issuer, email, key!);

        return new MfaSetupResult(key!, totpUri);
    }

    public async Task<bool> EnableTotpAsync(ApplicationUser user, string code)
    {
        var isValid = await _userManager.VerifyTwoFactorTokenAsync(
            user,
            _userManager.Options.Tokens.AuthenticatorTokenProvider,
            code);

        if (!isValid) return false;

        await _userManager.SetTwoFactorEnabledAsync(user, true);

        // Generate recovery codes
        var recoveryCodes = await _userManager.GenerateNewTwoFactorRecoveryCodesAsync(user, 10);

        // Store recovery codes securely (show only once)
        return true;
    }

    public async Task<SignInResult> VerifyTwoFactorAsync(string userId, string code, bool rememberDevice)
    {
        return await _signInManager.TwoFactorAuthenticatorSignInAsync(code, isPersistent: false, rememberClient: rememberDevice);
    }

    private static string GenerateTotpUri(string issuer, string account, string key)
    {
        var encodedIssuer  = Uri.EscapeDataString(issuer);
        var encodedAccount = Uri.EscapeDataString(account);
        return $"otpauth://totp/{encodedIssuer}:{encodedAccount}?secret={key}&issuer={encodedIssuer}&algorithm=SHA1&digits=6&period=30";
    }
}

// API endpoints for 2FA setup
app.MapPost("/api/mfa/setup", async (IMfaService mfa, HttpContext ctx, UserManager<ApplicationUser> um) =>
{
    var user = await um.GetUserAsync(ctx.User) ?? throw new InvalidOperationException();
    var result = await mfa.SetupTotpAsync(user);

    // Return setup info (QR code URI is used by an authenticator app)
    return Results.Ok(new { result.SharedKey, QrCodeUri = result.TotpUri });
}).RequireAuthorization();

app.MapPost("/api/mfa/enable", async (
    [FromBody] EnableMfaRequest req,
    IMfaService mfa, HttpContext ctx, UserManager<ApplicationUser> um) =>
{
    var user = await um.GetUserAsync(ctx.User) ?? throw new InvalidOperationException();
    var success = await mfa.EnableTotpAsync(user, req.Code);

    return success
        ? Results.Ok(new { Message = "2FA enabled successfully" })
        : Results.BadRequest(new { Error = "Invalid verification code" });
}).RequireAuthorization();
```

---

## Step 2031: Role-Based & Claims-Based Authorization Policies

```csharp
// Complex authorization policies
builder.Services.AddAuthorization(options =>
{
    // Role-based
    options.AddPolicy("AdminOnly",        p => p.RequireRole("admin"));
    options.AddPolicy("SupportOrAdmin",   p => p.RequireRole("admin", "support"));

    // Claims-based
    options.AddPolicy("VerifiedEmail",    p => p.RequireClaim("email_verified", "true"));
    options.AddPolicy("PremiumUser",      p => p.RequireClaim("subscription", "premium", "enterprise"));

    // Scope-based (API protection)
    options.AddPolicy("orders:read",  p => p.RequireClaim("scope", "orders:read"));
    options.AddPolicy("orders:write", p => p.RequireClaim("scope", "orders:write"));

    // Combined
    options.AddPolicy("SeniorEmployee",   p =>
        p.RequireAuthenticatedUser()
         .RequireRole("employee")
         .RequireClaim("tenure_years")
         .RequireAssertion(ctx =>
         {
             var tenureClaim = ctx.User.FindFirst("tenure_years")?.Value;
             return int.TryParse(tenureClaim, out var years) && years >= 3;
         }));

    // Custom requirement
    options.AddPolicy("OfficeHoursOnly", p => p.Requirements.Add(new OfficeHoursRequirement()));
});

// Custom requirement
public class OfficeHoursRequirement : IAuthorizationRequirement { }

public class OfficeHoursHandler : AuthorizationHandler<OfficeHoursRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext ctx,
        OfficeHoursRequirement requirement)
    {
        var now = DateTimeOffset.UtcNow;
        var isWeekday    = now.DayOfWeek is not DayOfWeek.Saturday and not DayOfWeek.Sunday;
        var isOfficeHour = now.Hour >= 8 && now.Hour < 18;

        if (isWeekday && isOfficeHour)
            ctx.Succeed(requirement);
        else
            ctx.Fail(new AuthorizationFailureReason(this, "Outside office hours"));

        return Task.CompletedTask;
    }
}

builder.Services.AddSingleton<IAuthorizationHandler, OfficeHoursHandler>();
```

---

## Step 2032: Resource-Based Authorization

```csharp
// Resource-based: check if user CAN act on a specific resource
public class OrderAuthorizationHandler : AuthorizationHandler<OrderOperationRequirement, Order>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext ctx,
        OrderOperationRequirement requirement,
        Order order)
    {
        var userId = ctx.User.FindFirstValue(ClaimTypes.NameIdentifier);

        if (requirement == OrderOperations.Read)
        {
            // Owner can read their own orders; support and admin can read any
            if (order.CustomerId.ToString() == userId ||
                ctx.User.IsInRole("admin") ||
                ctx.User.IsInRole("support"))
            {
                ctx.Succeed(requirement);
            }
        }
        else if (requirement == OrderOperations.Cancel)
        {
            // Only owner can cancel; admins can cancel anything
            if ((order.CustomerId.ToString() == userId && order.Status == OrderStatus.Pending) ||
                ctx.User.IsInRole("admin"))
            {
                ctx.Succeed(requirement);
            }
        }
        else if (requirement == OrderOperations.Refund)
        {
            // Only support/admin can issue refunds
            if (ctx.User.IsInRole("admin") || ctx.User.IsInRole("support"))
                ctx.Succeed(requirement);
        }

        return Task.CompletedTask;
    }
}

public class OrderOperationRequirement : IAuthorizationRequirement
{
    private OrderOperationRequirement(string name) => Name = name;
    public string Name { get; }
    public static readonly OrderOperationRequirement Read   = new("Read");
    public static readonly OrderOperationRequirement Cancel = new("Cancel");
    public static readonly OrderOperationRequirement Refund = new("Refund");
}

// Static class for syntactic sugar
public static class OrderOperations
{
    public static readonly OrderOperationRequirement Read   = OrderOperationRequirement.Read;
    public static readonly OrderOperationRequirement Cancel = OrderOperationRequirement.Cancel;
    public static readonly OrderOperationRequirement Refund = OrderOperationRequirement.Refund;
}

// Usage in an endpoint
app.MapGet("/api/orders/{id}", async (
    Guid id, IOrderRepository repo,
    IAuthorizationService authz, HttpContext ctx) =>
{
    var order = await repo.GetByIdAsync(id);
    if (order is null) return Results.NotFound();

    var result = await authz.AuthorizeAsync(ctx.User, order, OrderOperations.Read);
    if (!result.Succeeded) return Results.Forbid();

    return Results.Ok(order);
}).RequireAuthorization();

app.MapPost("/api/orders/{id}/cancel", async (
    Guid id, IOrderRepository repo,
    IAuthorizationService authz, HttpContext ctx,
    IMediator mediator) =>
{
    var order = await repo.GetByIdAsync(id);
    if (order is null) return Results.NotFound();

    var result = await authz.AuthorizeAsync(ctx.User, order, OrderOperations.Cancel);
    if (!result.Succeeded) return Results.Forbid();

    await mediator.Send(new CancelOrderCommand(id), ctx.RequestAborted);
    return Results.NoContent();
}).RequireAuthorization();
```

---

## Step 2033: JWT Best Practices & Hardening

```csharp
// JWT validation hardening
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = "https://identity.example.com";
        options.Audience  = "orders-api";

        options.TokenValidationParameters = new TokenValidationParameters
        {
            // Algorithm allowlist — CRITICAL: prevents alg:none attack
            ValidAlgorithms           = ["RS256", "ES256"],
            ValidateIssuer            = true,
            ValidIssuer               = "https://identity.example.com",
            ValidateAudience          = true,
            ValidAudience             = "orders-api",
            ValidateLifetime          = true,
            ValidateIssuerSigningKey  = true,
            ClockSkew                 = TimeSpan.FromSeconds(30), // tighten from default 5min
            RequireExpirationTime     = true,
            RequireSignedTokens       = true,
        };

        // Reject access tokens used as ID tokens
        options.Events = new JwtBearerEvents
        {
            OnTokenValidated = ctx =>
            {
                var token = ctx.SecurityToken as JwtSecurityToken;

                // Verify 'typ' header is 'at+jwt' for access tokens
                if (token?.Header.TryGetValue("typ", out var typ) == true &&
                    typ?.ToString() != "at+JWT")
                {
                    ctx.Fail("Invalid token type");
                }

                // Verify required claims exist
                if (ctx.Principal?.FindFirst("sub") is null)
                    ctx.Fail("Missing 'sub' claim");

                return Task.CompletedTask;
            },
            OnAuthenticationFailed = ctx =>
            {
                // Log authentication failures without exposing details to client
                var logger = ctx.HttpContext.RequestServices
                    .GetRequiredService<ILogger<Program>>();
                logger.LogWarning("JWT auth failed: {Error}", ctx.Exception.GetType().Name);
                return Task.CompletedTask;
            }
        };
    });

// Short-lived access tokens with automatic JWKS refresh
builder.Services.AddAuthentication()
    .AddJwtBearer(o =>
    {
        // OpenID Connect discovery automatically retrieves JWKS
        o.Authority                   = "https://identity.example.com";
        o.RefreshOnIssuerKeyNotFound  = true; // auto-refresh signing keys
        o.MetadataAddress             = "https://identity.example.com/.well-known/openid-configuration";
    });
```

```csharp
// JWT generation (if building your own token endpoint without OpenIddict)
public class JwtTokenFactory : IJwtTokenFactory
{
    private readonly IOptionsMonitor<JwtOptions> _options;

    public JwtTokenFactory(IOptionsMonitor<JwtOptions> options) => _options = options;

    public string CreateAccessToken(ClaimsIdentity identity)
    {
        var opts    = _options.CurrentValue;
        var handler = new JsonWebTokenHandler();

        // Use RSA private key from config (loaded from Key Vault in production)
        var key = new RsaSecurityKey(LoadRsaKey(opts.SigningKeyPem))
        {
            KeyId = opts.KeyId
        };

        var descriptor = new SecurityTokenDescriptor
        {
            Subject            = identity,
            Issuer             = opts.Issuer,
            Audience           = opts.Audience,
            IssuedAt           = DateTime.UtcNow,
            NotBefore          = DateTime.UtcNow,
            Expires            = DateTime.UtcNow.AddMinutes(opts.AccessTokenExpiryMinutes),
            SigningCredentials = new SigningCredentials(key, SecurityAlgorithms.RsaSha256),
            AdditionalHeaderClaims = new Dictionary<string, object>
            {
                ["typ"] = "at+JWT" // RFC 9068
            }
        };

        return handler.CreateToken(descriptor);
    }

    private static RSA LoadRsaKey(string pem)
    {
        var rsa = RSA.Create();
        rsa.ImportFromPem(pem.AsSpan());
        return rsa;
    }
}
```

---

## Step 2034: API Key Authentication

```csharp
// API key authentication for programmatic access
public class ApiKeyAuthenticationHandler : AuthenticationHandler<ApiKeyAuthenticationOptions>
{
    private readonly IApiKeyRepository _apiKeyRepo;

    public ApiKeyAuthenticationHandler(
        IApiKeyRepository apiKeyRepo,
        IOptionsMonitor<ApiKeyAuthenticationOptions> options,
        ILoggerFactory logger,
        UrlEncoder encoder)
        : base(options, logger, encoder)
    {
        _apiKeyRepo = apiKeyRepo;
    }

    protected override async Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        // Extract API key from header
        if (!Request.Headers.TryGetValue("X-Api-Key", out var apiKeyValues))
            return AuthenticateResult.NoResult();

        var apiKey = apiKeyValues.FirstOrDefault();
        if (string.IsNullOrWhiteSpace(apiKey))
            return AuthenticateResult.NoResult();

        // Hash the key and look it up (never store raw keys)
        var keyHash = HashApiKey(apiKey);
        var record  = await _apiKeyRepo.FindByHashAsync(keyHash);

        if (record is null)
            return AuthenticateResult.Fail("Invalid API key");

        if (record.ExpiresAt < DateTimeOffset.UtcNow)
            return AuthenticateResult.Fail("API key expired");

        // Update last-used timestamp
        await _apiKeyRepo.UpdateLastUsedAsync(record.Id);

        var claims = new List<Claim>
        {
            new(ClaimTypes.NameIdentifier, record.OwnerId.ToString()),
            new(ClaimTypes.Name,           record.Name),
            new("api_key_id",              record.Id.ToString()),
            new("api_key_scopes",          string.Join(" ", record.Scopes)),
        };

        foreach (var scope in record.Scopes)
            claims.Add(new Claim("scope", scope));

        var identity  = new ClaimsIdentity(claims, Scheme.Name);
        var principal = new ClaimsPrincipal(identity);
        var ticket    = new AuthenticationTicket(principal, Scheme.Name);

        return AuthenticateResult.Success(ticket);
    }

    private static string HashApiKey(string apiKey)
    {
        var bytes  = Encoding.UTF8.GetBytes(apiKey);
        var hash   = SHA256.HashData(bytes);
        return Convert.ToBase64String(hash);
    }
}

public class ApiKeyAuthenticationOptions : AuthenticationSchemeOptions { }

// Registration
builder.Services.AddAuthentication()
    .AddScheme<ApiKeyAuthenticationOptions, ApiKeyAuthenticationHandler>("ApiKey", null);

// Support both JWT and API key
builder.Services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = "Smart";
    options.DefaultChallengeScheme    = "Smart";
})
.AddPolicyScheme("Smart", "JWT or API Key", options =>
{
    options.ForwardDefaultSelector = ctx =>
    {
        if (ctx.Request.Headers.ContainsKey("X-Api-Key"))
            return "ApiKey";
        return JwtBearerDefaults.AuthenticationScheme;
    };
})
.AddJwtBearer(options => { /* ... */ })
.AddScheme<ApiKeyAuthenticationOptions, ApiKeyAuthenticationHandler>("ApiKey", null);
```

---

## Step 2035: External Identity Providers (Google, GitHub, Microsoft)

```csharp
// Social login integration
// NuGet: Microsoft.AspNetCore.Authentication.Google
//        Microsoft.AspNetCore.Authentication.GitHub (AspNet.Security.OAuth.GitHub)
//        Microsoft.AspNetCore.Authentication.MicrosoftAccount

builder.Services.AddAuthentication()
    .AddGoogle(options =>
    {
        options.ClientId     = builder.Configuration["Auth:Google:ClientId"]!;
        options.ClientSecret = builder.Configuration["Auth:Google:ClientSecret"]!;

        options.Scope.Add("profile");
        options.Scope.Add("email");

        options.Events.OnCreatingTicket = async ctx =>
        {
            // Map Google claims to app claims
            var googleId = ctx.Principal?.FindFirstValue(ClaimTypes.NameIdentifier);
            var email    = ctx.Principal?.FindFirstValue(ClaimTypes.Email);

            // Link or create user account
            var userService = ctx.HttpContext.RequestServices.GetRequiredService<IExternalAuthService>();
            var user        = await userService.GetOrCreateFromExternalAsync("Google", googleId!, email!);

            if (!user.IsActive)
                ctx.Fail("Account is disabled");
        };
    })
    .AddGitHub(options =>
    {
        options.ClientId     = builder.Configuration["Auth:GitHub:ClientId"]!;
        options.ClientSecret = builder.Configuration["Auth:GitHub:ClientSecret"]!;
        options.Scope.Add("user:email");
    })
    .AddMicrosoftAccount(options =>
    {
        options.ClientId     = builder.Configuration["Auth:Microsoft:ClientId"]!;
        options.ClientSecret = builder.Configuration["Auth:Microsoft:ClientSecret"]!;
    });

// External auth service
public class ExternalAuthService : IExternalAuthService
{
    private readonly UserManager<ApplicationUser> _userManager;
    private readonly IUserStore<ApplicationUser> _userStore;

    public ExternalAuthService(UserManager<ApplicationUser> userManager, IUserStore<ApplicationUser> userStore)
    {
        _userManager = userManager;
        _userStore   = userStore;
    }

    public async Task<ApplicationUser> GetOrCreateFromExternalAsync(
        string provider, string externalId, string email)
    {
        // Find user by external login
        var user = await _userManager.FindByLoginAsync(provider, externalId);
        if (user is not null) return user;

        // Find by email (link existing account)
        user = await _userManager.FindByEmailAsync(email);
        if (user is not null)
        {
            await _userManager.AddLoginAsync(user, new UserLoginInfo(provider, externalId, provider));
            return user;
        }

        // Create new user
        user = new ApplicationUser
        {
            UserName       = email,
            Email          = email,
            EmailConfirmed = true, // external provider already verified
            IsActive       = true
        };

        var result = await _userManager.CreateAsync(user);
        if (!result.Succeeded)
            throw new InvalidOperationException($"Failed to create user: {string.Join(", ", result.Errors.Select(e => e.Description))}");

        await _userManager.AddLoginAsync(user, new UserLoginInfo(provider, externalId, provider));
        await _userManager.AddToRoleAsync(user, "user");

        return user;
    }
}
```

---

## Step 2036: Security Headers, CORS & Complete Auth Configuration

```csharp
// Complete auth configuration with security headers
var builder = WebApplication.CreateBuilder(args);

// Authentication
builder.Services.AddAuthentication(...)     // as shown above
builder.Services.AddAuthorization(...);     // as shown above

// CORS — restrictive by default
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowFrontend", policy =>
        policy
            .WithOrigins(
                "https://app.example.com",
                builder.Environment.IsDevelopment() ? "http://localhost:5173" : null!)
            .AllowAnyMethod()
            .AllowAnyHeader()
            .AllowCredentials()
            .SetPreflightMaxAge(TimeSpan.FromHours(1))
    );

    options.AddPolicy("AllowNone", _ => { }); // explicitly no CORS for admin APIs
});

// Rate limiting on auth endpoints
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("login", config =>
    {
        config.Window           = TimeSpan.FromMinutes(15);
        config.PermitLimit      = 5;
        config.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        config.QueueLimit       = 0;
    });
});

var app = builder.Build();

// Security headers middleware (run before auth)
app.Use(async (ctx, next) =>
{
    var response = ctx.Response;
    response.Headers["X-Frame-Options"]           = "DENY";
    response.Headers["X-Content-Type-Options"]    = "nosniff";
    response.Headers["X-XSS-Protection"]          = "0"; // disabled; use CSP instead
    response.Headers["Referrer-Policy"]           = "strict-origin-when-cross-origin";
    response.Headers["Permissions-Policy"]        = "camera=(), microphone=(), geolocation=()";

    if (!ctx.Request.IsHttps && !builder.Environment.IsDevelopment())
        response.Headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains; preload";

    response.Headers["Content-Security-Policy"]   =
        "default-src 'self'; " +
        "script-src 'self'; " +
        "style-src 'self' 'unsafe-inline'; " +  // 'unsafe-inline' only if needed
        "img-src 'self' data: https:; " +
        "font-src 'self'; " +
        "connect-src 'self' https://identity.example.com; " +
        "frame-ancestors 'none'; " +
        "form-action 'self'; " +
        "upgrade-insecure-requests;";

    await next();
});

app.UseHttpsRedirection();
app.UseCors("AllowFrontend");
app.UseRateLimiter();
app.UseAuthentication();
app.UseAuthorization();

// Apply rate limit to login endpoint
app.MapPost("/api/auth/login", HandleLogin)
   .RequireRateLimiting("login")
   .AllowAnonymous();

app.MapControllers();
app.Run();
```

```json
// appsettings.json — auth configuration
{
  "Authentication": {
    "Authority": "https://identity.example.com",
    "Audience": "orders-api",
    "AccessTokenExpiryMinutes": 15,
    "RefreshTokenExpiryDays": 30,
    "KeyId": "rsa-2024-v1"
  },
  "Oidc": {
    "ClientId": "web-client",
    "ClientSecret": "YOUR-SECRET-FROM-KEY-VAULT"
  },
  "Auth": {
    "Google": {
      "ClientId": "xxx.apps.googleusercontent.com",
      "ClientSecret": "YOUR-SECRET"
    },
    "GitHub": {
      "ClientId": "Iv1.xxx",
      "ClientSecret": "YOUR-SECRET"
    }
  },
  "Introspection": {
    "ClientId": "orders-api",
    "ClientSecret": "YOUR-INTROSPECTION-SECRET"
  }
}
```

**Summary**: Part 87 covers OAuth 2.0 fundamentals, PKCE authorization code flow, client credentials for M2M, OpenIddict setup with authorization/token endpoints, client registration, token introspection, refresh token rotation, TOTP-based MFA, RBAC/ABAC policies, resource-based authorization, JWT hardening (algorithm allowlist, typ header), API key authentication, social login federation, and complete security headers + CORS configuration. Steps 2021–2036 complete.
