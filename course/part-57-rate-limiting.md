# Part 57: Rate Limiting & API Gateway Patterns

## Steps 1551-1570: Production-Grade Rate Limiting with ASP.NET Core & YARP

---

## Step 1551: Rate Limiting Architecture Overview

```
Rate Limiting Strategies
══════════════════════════════════════════════════════════════

  Fixed Window          Sliding Window         Token Bucket
  ────────────          ──────────────         ────────────
  |‾‾‾‾‾‾‾‾‾‾|         [────────────]         ◉ ◉ ◉ ◉ ◉
  | 100 req  |         rolling 60s            tokens refill
  | per min  |         100 requests           continuously
  |__________|         exact window
  Reset at :00         smooth traffic         bursting allowed

  Concurrency Limit     Chained Policies
  ─────────────────     ────────────────
  ≤ 10 concurrent       Per-IP → Per-User → Global
  requests active       first match wins
  at same time          or all must pass
```

### NuGet Packages

```xml
<PackageReference Include="Microsoft.AspNetCore.RateLimiting"              Version="8.*" />
<PackageReference Include="Yarp.ReverseProxy"                              Version="2.*" />
<PackageReference Include="StackExchange.Redis"                            Version="2.*" />
<PackageReference Include="Microsoft.Extensions.Caching.StackExchangeRedis" Version="8.*" />
```

---

## Step 1552: Built-In Rate Limiting — All Algorithms

```csharp
// Program.cs
using Microsoft.AspNetCore.RateLimiting;
using System.Threading.RateLimiting;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddRateLimiter(options =>
{
    // ── 1. Fixed Window ──────────────────────────────────────────────────
    options.AddFixedWindowLimiter("fixed", o =>
    {
        o.PermitLimit         = 100;
        o.Window              = TimeSpan.FromMinutes(1);
        o.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        o.QueueLimit          = 10;  // queue up to 10 when limit reached
    });

    // ── 2. Sliding Window ────────────────────────────────────────────────
    options.AddSlidingWindowLimiter("sliding", o =>
    {
        o.PermitLimit            = 100;
        o.Window                 = TimeSpan.FromMinutes(1);
        o.SegmentsPerWindow      = 6;  // 10-second sub-windows
        o.QueueProcessingOrder   = QueueProcessingOrder.OldestFirst;
        o.QueueLimit             = 5;
    });

    // ── 3. Token Bucket ──────────────────────────────────────────────────
    options.AddTokenBucketLimiter("token", o =>
    {
        o.TokenLimit             = 200;   // max burst
        o.ReplenishmentPeriod    = TimeSpan.FromSeconds(10);
        o.TokensPerPeriod        = 50;    // refill rate
        o.AutoReplenishment      = true;  // background replenishment
        o.QueueProcessingOrder   = QueueProcessingOrder.OldestFirst;
        o.QueueLimit             = 20;
    });

    // ── 4. Concurrency Limiter ───────────────────────────────────────────
    options.AddConcurrencyLimiter("concurrency", o =>
    {
        o.PermitLimit            = 50;   // max simultaneous requests
        o.QueueProcessingOrder   = QueueProcessingOrder.OldestFirst;
        o.QueueLimit             = 10;
    });

    // ── 5. Per-User Partition ────────────────────────────────────────────
    options.AddPolicy("per-user", httpContext =>
    {
        // Partition by user ID if authenticated, else by IP
        var userId = httpContext.User.FindFirst("sub")?.Value;
        if (userId is not null)
        {
            return RateLimitPartition.GetSlidingWindowLimiter(userId, key =>
                new SlidingWindowRateLimiterOptions
                {
                    PermitLimit       = 500,
                    Window            = TimeSpan.FromMinutes(1),
                    SegmentsPerWindow = 6
                });
        }

        var ip = httpContext.Connection.RemoteIpAddress?.ToString() ?? "anonymous";
        return RateLimitPartition.GetFixedWindowLimiter(ip, key =>
            new FixedWindowRateLimiterOptions
            {
                PermitLimit = 20,
                Window      = TimeSpan.FromMinutes(1)
            });
    });

    // ── 6. Per-Tier Partition (Free/Pro/Enterprise) ──────────────────────
    options.AddPolicy("per-tier", httpContext =>
    {
        var tier = httpContext.User.FindFirst("tier")?.Value ?? "free";
        return tier switch
        {
            "enterprise" => RateLimitPartition.GetNoLimiter("enterprise"),
            "pro"        => RateLimitPartition.GetTokenBucketLimiter("pro:" + httpContext.User.FindFirst("sub")!.Value,
                key => new TokenBucketRateLimiterOptions
                {
                    TokenLimit          = 10_000,
                    ReplenishmentPeriod = TimeSpan.FromHours(1),
                    TokensPerPeriod     = 5_000
                }),
            _ => RateLimitPartition.GetFixedWindowLimiter("free:" + httpContext.Connection.RemoteIpAddress,
                key => new FixedWindowRateLimiterOptions
                {
                    PermitLimit = 100,
                    Window      = TimeSpan.FromHours(1)
                })
        };
    });

    // ── Rejection Response ────────────────────────────────────────────────
    options.OnRejected = async (context, ct) =>
    {
        context.HttpContext.Response.StatusCode  = 429;
        context.HttpContext.Response.ContentType = "application/json";

        // Tell client when to retry
        if (context.Lease.TryGetMetadata(MetadataName.RetryAfter, out var retryAfter))
        {
            context.HttpContext.Response.Headers["Retry-After"] =
                ((int)retryAfter.TotalSeconds).ToString();
        }

        await context.HttpContext.Response.WriteAsJsonAsync(new
        {
            type    = "https://tools.ietf.org/html/rfc6585#section-4",
            title   = "Too Many Requests",
            status  = 429,
            detail  = "Rate limit exceeded. Please retry after the indicated period.",
            retryAfter = retryAfter.TotalSeconds
        }, ct);
    };
});
```

---

## Step 1553: Applying Rate Limits to Endpoints

```csharp
var app = builder.Build();
app.UseRateLimiter();

// ── Apply to route groups ────────────────────────────────────────────────

// Public API: 20 req/min per IP
var publicApi = app.MapGroup("/api/public")
    .RequireRateLimiting("fixed");

publicApi.MapGet("/products", GetProducts);
publicApi.MapGet("/categories", GetCategories);

// Auth'd API: per-user limits
var authApi = app.MapGroup("/api")
    .RequireAuthorization()
    .RequireRateLimiting("per-user");

authApi.MapPost("/orders", CreateOrder);
authApi.MapGet("/orders/{id}", GetOrder);

// Per-tier: premium endpoints
var premiumApi = app.MapGroup("/api/v2")
    .RequireAuthorization()
    .RequireRateLimiting("per-tier");

premiumApi.MapGet("/analytics", GetAnalytics);
premiumApi.MapPost("/bulk-import", BulkImport);

// Specific endpoint: token bucket for write operations
app.MapPost("/api/webhooks/ingest", IngestWebhook)
    .RequireRateLimiting("token");

// Disable rate limiting on health checks
app.MapHealthChecks("/health").DisableRateLimiting();

// MVC Controllers with attribute
// [EnableRateLimiting("per-user")] on controller
// [DisableRateLimiting] on specific action
```

```csharp
// Apply rate limiting to MVC controllers
[ApiController]
[Route("api/[controller]")]
[EnableRateLimiting("per-user")]
public class OrdersController : ControllerBase
{
    // Inherits per-user limit

    [HttpPost]
    public async Task<IActionResult> CreateOrder([FromBody] CreateOrderRequest req)
        => Ok(await _service.CreateAsync(req));

    // Override with stricter limit for expensive operation
    [HttpPost("batch")]
    [EnableRateLimiting("concurrency")]
    public async Task<IActionResult> BatchCreate([FromBody] BatchCreateRequest req)
        => Ok(await _service.BatchCreateAsync(req));

    // Disable for internal health-check style endpoint
    [HttpGet("ping")]
    [DisableRateLimiting]
    public IActionResult Ping() => Ok("pong");
}
```

---

## Step 1554: Redis-Backed Distributed Rate Limiting

```csharp
// RedisRateLimiter.cs — for multi-instance deployments
using StackExchange.Redis;
using System.Threading.RateLimiting;

namespace RateLimitingDemo;

public class RedisFixedWindowRateLimiter : RateLimiter
{
    private readonly IConnectionMultiplexer _redis;
    private readonly RedisFixedWindowOptions _options;
    private readonly string _key;

    public RedisFixedWindowRateLimiter(
        string partitionKey,
        RedisFixedWindowOptions options,
        IConnectionMultiplexer redis)
    {
        _key     = $"ratelimit:{partitionKey}";
        _options = options;
        _redis   = redis;
    }

    public override RateLimiterStatistics? GetStatistics()
        => null; // Redis-backed — statistics not tracked locally

    public override TimeSpan? IdleDuration => null;

    protected override async ValueTask<RateLimitLease> AcquireAsyncCore(
        int permitCount,
        CancellationToken cancellationToken)
    {
        var db     = _redis.GetDatabase();
        var now    = DateTimeOffset.UtcNow.ToUnixTimeSeconds();
        var window = (long)_options.Window.TotalSeconds;

        // Lua script: atomic increment + TTL set
        const string lua = @"
            local key = KEYS[1]
            local limit = tonumber(ARGV[1])
            local window = tonumber(ARGV[2])
            local count = redis.call('INCR', key)
            if count == 1 then
                redis.call('EXPIRE', key, window)
            end
            if count <= limit then
                return {1, count, limit - count}
            else
                return {0, count, 0}
            end";

        var result = (RedisValue[])await db.ScriptEvaluateAsync(
            lua,
            keys:   new RedisKey[]  { _key },
            values: new RedisValue[] { _options.PermitLimit, window });

        bool allowed    = (long)result[0] == 1;
        long remaining  = (long)result[2];

        if (allowed)
            return new RedisRateLimitLease(true, remaining, _options.Window);

        // Get TTL for Retry-After
        var ttl = await db.KeyTimeToLiveAsync(_key);
        return new RedisRateLimitLease(false, 0, ttl ?? _options.Window);
    }

    protected override RateLimitLease AttemptAcquireCore(int permitCount)
    {
        // Synchronous path — not ideal for Redis; throw to force async
        throw new NotSupportedException("Use AcquireAsync for Redis-backed limiter");
    }
}

public sealed class RedisRateLimitLease : RateLimitLease
{
    private readonly bool _isAcquired;
    private readonly long _remaining;
    private readonly TimeSpan _retryAfter;

    public RedisRateLimitLease(bool isAcquired, long remaining, TimeSpan retryAfter)
    {
        _isAcquired = isAcquired;
        _remaining  = remaining;
        _retryAfter = retryAfter;
    }

    public override bool IsAcquired => _isAcquired;

    public override IEnumerable<MetadataName> MetadataNames =>
        _isAcquired
            ? [MetadataName.ReasonPhrase]
            : [MetadataName.RetryAfter, MetadataName.ReasonPhrase];

    public override bool TryGetMetadata(MetadataName metadataName, out object? metadata)
    {
        if (metadataName == MetadataName.RetryAfter)
        {
            metadata = _retryAfter;
            return true;
        }
        if (metadataName == MetadataName.ReasonPhrase)
        {
            metadata = _isAcquired ? $"{_remaining} requests remaining" : "Rate limit exceeded";
            return true;
        }
        metadata = null;
        return false;
    }

    protected override void Dispose(bool disposing) { }
}

public record RedisFixedWindowOptions(int PermitLimit, TimeSpan Window);
```

```csharp
// Register Redis rate limiter
builder.Services.AddSingleton<IConnectionMultiplexer>(
    ConnectionMultiplexer.Connect(builder.Configuration["Redis:ConnectionString"]!));

builder.Services.AddRateLimiter(options =>
{
    options.AddPolicy("redis-per-user", context =>
    {
        var userId = context.User.FindFirst("sub")?.Value
                  ?? context.Connection.RemoteIpAddress?.ToString()
                  ?? "anon";

        var redis = context.RequestServices.GetRequiredService<IConnectionMultiplexer>();

        return RateLimitPartition.Get(userId, key =>
            new RedisFixedWindowRateLimiter(key,
                new RedisFixedWindowOptions(100, TimeSpan.FromMinutes(1)),
                redis));
    });
});
```

---

## Step 1555: Rate Limit Headers & Client Transparency

```csharp
// RateLimitHeaderMiddleware.cs
public class RateLimitHeaderMiddleware
{
    private readonly RequestDelegate _next;

    public RateLimitHeaderMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(HttpContext context)
    {
        await _next(context);

        // Add standard rate limit headers after response is created
        // These align with the draft IETF spec (draft-ietf-httpapi-ratelimit-headers)
        if (context.Items.TryGetValue("RateLimitRemaining", out var remaining))
        {
            context.Response.Headers["X-RateLimit-Limit"]     = context.Items["RateLimitLimit"]?.ToString();
            context.Response.Headers["X-RateLimit-Remaining"] = remaining?.ToString();
            context.Response.Headers["X-RateLimit-Reset"]     = context.Items["RateLimitReset"]?.ToString();
        }
    }
}

// Custom rate limit policy that enriches context
builder.Services.AddRateLimiter(options =>
{
    options.AddPolicy("transparent", context =>
    {
        var ip = context.Connection.RemoteIpAddress?.ToString() ?? "unknown";

        return RateLimitPartition.GetFixedWindowLimiter(ip, key =>
            new FixedWindowRateLimiterOptions
            {
                PermitLimit = 100,
                Window      = TimeSpan.FromMinutes(1)
            });
    });

    // Hook into lease acquisition to set headers
    options.OnRejected = (ctx, ct) =>
    {
        ctx.HttpContext.Response.StatusCode = 429;

        if (ctx.Lease.TryGetMetadata(MetadataName.RetryAfter, out var retryAfter))
            ctx.HttpContext.Response.Headers["Retry-After"] = ((int)retryAfter.TotalSeconds).ToString();

        // Standard rate limit headers
        ctx.HttpContext.Response.Headers["X-RateLimit-Limit"]     = "100";
        ctx.HttpContext.Response.Headers["X-RateLimit-Remaining"] = "0";
        ctx.HttpContext.Response.Headers["X-RateLimit-Reset"]     =
            DateTimeOffset.UtcNow.AddMinutes(1).ToUnixTimeSeconds().ToString();

        return ctx.HttpContext.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Status = 429,
            Title  = "Too Many Requests",
            Detail = "Rate limit exceeded"
        }, ct);
    };
});
```

---

## Step 1556: YARP — Yet Another Reverse Proxy

```
YARP Architecture
══════════════════════════════════════════════════════════════

  Client
    │
    ▼
  ┌──────────────────────────────────────────────────────────┐
  │                    YARP Reverse Proxy                     │
  │  ┌──────────┐  ┌────────────────┐  ┌──────────────────┐  │
  │  │  Route   │  │   Load Balance │  │  Health Check    │  │
  │  │  Matching│→ │   (RoundRobin, │→ │  (passive/active)│  │
  │  │          │  │   LeastReq,    │  │                  │  │
  │  │          │  │   PowerOfTwo)  │  │                  │  │
  │  └──────────┘  └────────────────┘  └──────────────────┘  │
  │       │                │                    │             │
  │       ▼                ▼                    ▼             │
  │  ┌──────────────────────────────────────────────────────┐ │
  │  │              Middleware Pipeline                       │ │
  │  │  Rate Limit → Auth → Transform → Retry → Forward     │ │
  │  └──────────────────────────────────────────────────────┘ │
  └──────────────────────────────────────────────────────────┘
         │               │               │
    ┌────┴───┐      ┌─────┴──┐      ┌────┴───┐
    │Service │      │Service │      │Service │
    │  v1    │      │  v2    │      │  v3    │
    └────────┘      └────────┘      └────────┘
```

```csharp
// YARP Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"))
    .AddConfigFilter<CustomConfigFilter>()
    .AddTransforms<CustomTransformProvider>()
    .ConfigureHttpClient((context, handler) =>
    {
        handler.MaxConnectionsPerServer = 100;
        handler.PooledConnectionLifetime = TimeSpan.FromMinutes(10);
    });

// Add rate limiting to the proxy
builder.Services.AddRateLimiter(options =>
{
    options.AddSlidingWindowLimiter("proxy-limit", o =>
    {
        o.PermitLimit       = 1000;
        o.Window            = TimeSpan.FromMinutes(1);
        o.SegmentsPerWindow = 6;
    });
});

var app = builder.Build();
app.UseRateLimiter();
app.MapReverseProxy().RequireRateLimiting("proxy-limit");
app.Run();
```

```json
// appsettings.json — YARP configuration
{
  "ReverseProxy": {
    "Routes": {
      "orders-route": {
        "ClusterId": "orders-cluster",
        "Match": {
          "Path": "/api/orders/{**catch-all}"
        },
        "Transforms": [
          { "PathRemovePrefix": "/api" },
          { "RequestHeader": "X-Forwarded-Host", "Append": "{Host}" },
          { "RequestHeader": "X-Correlation-Id", "Append": "{RequestId}" }
        ],
        "Metadata": {
          "RateLimitPolicy": "per-user"
        }
      },

      "inventory-route": {
        "ClusterId": "inventory-cluster",
        "Match": {
          "Path": "/api/inventory/{**catch-all}",
          "Headers": [
            { "Name": "X-Api-Version", "Values": ["v2"], "Mode": "ExactHeader" }
          ]
        },
        "Transforms": [
          { "PathRemovePrefix": "/api" }
        ]
      },

      "websocket-route": {
        "ClusterId": "realtime-cluster",
        "Match": {
          "Path": "/ws/{**catch-all}"
        }
      }
    },

    "Clusters": {
      "orders-cluster": {
        "LoadBalancingPolicy": "RoundRobin",
        "HealthCheck": {
          "Passive": {
            "Enabled": true,
            "ReactivationPeriod": "00:00:30"
          },
          "Active": {
            "Enabled": true,
            "Interval": "00:00:10",
            "Timeout": "00:00:05",
            "Policy": "ConsecutiveFailures",
            "Path": "/health"
          }
        },
        "HttpClient": {
          "SslProtocols": "Tls12,Tls13",
          "DangerousAcceptAnyServerCertificate": false,
          "MaxConnectionsPerServer": 50
        },
        "Destinations": {
          "orders-1": { "Address": "https://orders-service-1:5001" },
          "orders-2": { "Address": "https://orders-service-2:5001" },
          "orders-3": { "Address": "https://orders-service-3:5001" }
        }
      },

      "inventory-cluster": {
        "LoadBalancingPolicy": "LeastRequests",
        "Destinations": {
          "inv-1": { "Address": "http://inventory-service:5002" },
          "inv-2": { "Address": "http://inventory-service-replica:5002" }
        }
      },

      "realtime-cluster": {
        "LoadBalancingPolicy": "PowerOfTwoChoices",
        "SessionAffinity": {
          "Enabled": true,
          "Policy": "Cookie",
          "AffinityKeyName": "YARP_Affinity",
          "Cookie": {
            "SameSite": "Strict",
            "Secure": true,
            "MaxAge": "01:00:00"
          }
        },
        "Destinations": {
          "rt-1": { "Address": "http://realtime-service-1:5003" },
          "rt-2": { "Address": "http://realtime-service-2:5003" }
        }
      }
    }
  }
}
```

---

## Step 1557: YARP Custom Transforms

```csharp
// CustomTransformProvider.cs
using Yarp.ReverseProxy.Transforms;
using Yarp.ReverseProxy.Transforms.Builder;

namespace RateLimitingDemo;

public class CustomTransformProvider : ITransformProvider
{
    public void ValidateRoute(TransformRouteValidationContext context) { }
    public void ValidateCluster(TransformClusterValidationContext context) { }

    public void Apply(TransformBuilderContext context)
    {
        // Add correlation ID to every proxied request
        context.AddRequestTransform(transformContext =>
        {
            transformContext.ProxyRequest.Headers.TryAddWithoutValidation(
                "X-Correlation-Id",
                transformContext.HttpContext.TraceIdentifier);
            return ValueTask.CompletedTask;
        });

        // Remove internal headers from upstream response
        context.AddResponseTransform(transformContext =>
        {
            transformContext.HttpContext.Response.Headers.Remove("X-Internal-Service-Id");
            transformContext.HttpContext.Response.Headers.Remove("Server");
            return ValueTask.CompletedTask;
        });

        // Add rate limit headers from lease
        context.AddResponseTransform(transformContext =>
        {
            if (transformContext.HttpContext.Items.TryGetValue("RateLimitRemaining", out var rem))
            {
                transformContext.HttpContext.Response.Headers["X-RateLimit-Remaining"] = rem?.ToString();
            }
            return ValueTask.CompletedTask;
        });
    }
}

// Programmatic route + cluster manipulation
public class DynamicRouteConfigProvider : IProxyConfigProvider
{
    private volatile DynamicRouteConfig _config;

    public DynamicRouteConfigProvider()
    {
        _config = Build(GetInitialRoutes(), GetInitialClusters());
    }

    public IProxyConfig GetConfig() => _config;

    public void Update(IReadOnlyList<RouteConfig> routes, IReadOnlyList<ClusterConfig> clusters)
    {
        var oldConfig = _config;
        _config = Build(routes, clusters);
        oldConfig.SignalChange();
    }

    private static DynamicRouteConfig Build(
        IReadOnlyList<RouteConfig> routes,
        IReadOnlyList<ClusterConfig> clusters)
        => new(routes, clusters, CancellationToken.None);
}
```

```csharp
// Transform via route metadata
public class MetadataRateLimitTransform : ITransformProvider
{
    private readonly IRateLimiterPolicy<string> _policy;

    public MetadataRateLimitTransform(IRateLimiterPolicy<string> policy) => _policy = policy;

    public void Apply(TransformBuilderContext context)
    {
        if (!context.Route.Metadata?.TryGetValue("RateLimitPolicy", out var policyName) == true)
            return;

        context.AddRequestTransform(async transformContext =>
        {
            var httpContext = transformContext.HttpContext;
            // Apply the named rate limit policy per route
            using var lease = await _policy.AcquireAsync(httpContext, cancellationToken: httpContext.RequestAborted);

            if (!lease.IsAcquired)
            {
                httpContext.Response.StatusCode = 429;
                httpContext.Items["RateLimitRejected"] = true;

                if (lease.TryGetMetadata(MetadataName.RetryAfter, out var retryAfter))
                    httpContext.Response.Headers["Retry-After"] = ((int)retryAfter.TotalSeconds).ToString();

                transformContext.HttpContext.Abort();
            }
        });
    }

    public void ValidateRoute(TransformRouteValidationContext context) { }
    public void ValidateCluster(TransformClusterValidationContext context) { }
}
```

---

## Step 1558: YARP Dynamic Configuration & A/B Testing

```csharp
// DynamicConfigService.cs — update routes at runtime (e.g., from feature flags)
public class DynamicConfigService : BackgroundService
{
    private readonly IProxyStateLookup _proxy;
    private readonly IFeatureManager _features;
    private readonly DynamicRouteConfigProvider _configProvider;
    private readonly ILogger<DynamicConfigService> _log;

    public DynamicConfigService(
        IProxyStateLookup proxy,
        IFeatureManager features,
        DynamicRouteConfigProvider configProvider,
        ILogger<DynamicConfigService> log)
    {
        _proxy          = proxy;
        _features       = features;
        _configProvider = configProvider;
        _log            = log;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await Task.Delay(TimeSpan.FromSeconds(30), stoppingToken);
            await UpdateRoutesFromFeatureFlagsAsync(stoppingToken);
        }
    }

    private async Task UpdateRoutesFromFeatureFlagsAsync(CancellationToken ct)
    {
        var routes   = new List<RouteConfig>();
        var clusters = new List<ClusterConfig>();

        // Route v2 traffic based on feature flag
        if (await _features.IsEnabledAsync("OrderServiceV2"))
        {
            routes.Add(new RouteConfig
            {
                RouteId   = "orders-v2",
                ClusterId = "orders-v2-cluster",
                Match     = new RouteMatch { Path = "/api/orders/{**rest}" }
            });

            clusters.Add(new ClusterConfig
            {
                ClusterId = "orders-v2-cluster",
                Destinations = new Dictionary<string, DestinationConfig>
                {
                    ["v2-1"] = new() { Address = "http://orders-v2-1:5001" }
                }
            });
        }
        else
        {
            routes.Add(new RouteConfig
            {
                RouteId   = "orders-v1",
                ClusterId = "orders-v1-cluster",
                Match     = new RouteMatch { Path = "/api/orders/{**rest}" }
            });
        }

        _configProvider.Update(routes, clusters);
        _log.LogInformation("YARP routes updated. V2 enabled: {V2}",
            await _features.IsEnabledAsync("OrderServiceV2"));
    }
}

// A/B routing via custom load balancing policy
public class CanaryLoadBalancingPolicy : ILoadBalancingPolicy
{
    private readonly IFeatureManager _features;

    public CanaryLoadBalancingPolicy(IFeatureManager features) => _features = features;

    public string Name => "Canary";

    public async ValueTask<DestinationState?> PickDestinationAsync(
        HttpContext context,
        ClusterState cluster,
        IReadOnlyList<DestinationState> availableDestinations,
        CancellationToken cancellationToken)
    {
        if (availableDestinations.Count == 0) return null;

        // Route 10% of traffic to canary
        var isCanary = await _features.IsEnabledAsync("CanaryDeployment", context);
        if (isCanary)
        {
            var canary = availableDestinations
                .FirstOrDefault(d => d.Model.Config.Metadata?.ContainsKey("canary") == true);
            if (canary is not null) return canary;
        }

        // Default: round-robin stable
        var stable = availableDestinations
            .Where(d => d.Model.Config.Metadata?.ContainsKey("canary") != true)
            .ToList();

        var idx = (int)(context.Connection.Id.GetHashCode() % stable.Count);
        return stable[Math.Abs(idx)];
    }
}
```

---

## Step 1559: Circuit Breaking at the Gateway

```csharp
// Polly v8 integration with YARP
using Polly;
using Polly.CircuitBreaker;

builder.Services.AddReverseProxy()
    .ConfigureHttpClient((serviceProvider, context, handler) =>
    {
        // Nothing here — configure via AddResilienceHandler below
    })
    .AddResilienceHandler("gateway-pipeline", (builder, context) =>
    {
        builder
            .AddTimeout(TimeSpan.FromSeconds(10))
            .AddCircuitBreaker(new CircuitBreakerStrategyOptions
            {
                FailureRatio            = 0.5,   // 50% failures
                SamplingDuration        = TimeSpan.FromSeconds(30),
                MinimumThroughput       = 10,    // need 10 requests to trip
                BreakDuration           = TimeSpan.FromSeconds(30),
                ShouldHandle            = new PredicateBuilder<HttpResponseMessage>()
                    .HandleResult(r => (int)r.StatusCode >= 500)
                    .Handle<HttpRequestException>(),
                OnOpened = args =>
                {
                    context.ServiceProvider
                        .GetService<ILogger<Program>>()
                        ?.LogWarning("Circuit breaker OPEN for cluster {ClusterId}",
                            context.Cluster?.Config.ClusterId);
                    return ValueTask.CompletedTask;
                },
                OnClosed = args =>
                {
                    context.ServiceProvider
                        .GetService<ILogger<Program>>()
                        ?.LogInformation("Circuit breaker CLOSED for cluster {ClusterId}",
                            context.Cluster?.Config.ClusterId);
                    return ValueTask.CompletedTask;
                }
            })
            .AddRetry(new Polly.Retry.RetryStrategyOptions<HttpResponseMessage>
            {
                MaxRetryAttempts = 3,
                BackoffType      = DelayBackoffType.Exponential,
                Delay            = TimeSpan.FromMilliseconds(250),
                ShouldHandle     = new PredicateBuilder<HttpResponseMessage>()
                    .HandleResult(r => r.StatusCode == System.Net.HttpStatusCode.ServiceUnavailable)
                    .Handle<HttpRequestException>()
            });
    });
```

---

## Step 1560: Rate Limit Middleware for Custom Scenarios

```csharp
// ThrottleMiddleware.cs — custom per-API-key throttling
public class ApiKeyThrottleMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IConnectionMultiplexer _redis;
    private readonly ILogger<ApiKeyThrottleMiddleware> _log;

    public ApiKeyThrottleMiddleware(
        RequestDelegate next,
        IConnectionMultiplexer redis,
        ILogger<ApiKeyThrottleMiddleware> log)
    {
        _next  = next;
        _redis = redis;
        _log   = log;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var apiKey = context.Request.Headers["X-Api-Key"].FirstOrDefault();
        if (apiKey is null)
        {
            await _next(context);
            return;
        }

        var limit    = await GetLimitForKeyAsync(apiKey);
        var allowed  = await CheckAndIncrementAsync(apiKey, limit);

        if (!allowed.IsAllowed)
        {
            _log.LogWarning("API key {Key} rate limited. Count: {Count}", apiKey[..8] + "...", allowed.Count);

            context.Response.StatusCode = 429;
            context.Response.Headers["X-RateLimit-Limit"]     = limit.RequestsPerMinute.ToString();
            context.Response.Headers["X-RateLimit-Remaining"] = "0";
            context.Response.Headers["Retry-After"]           = "60";

            await context.Response.WriteAsJsonAsync(new
            {
                error = "rate_limit_exceeded",
                limit = limit.RequestsPerMinute,
                window = "60s"
            });

            return;
        }

        context.Response.Headers["X-RateLimit-Limit"]     = limit.RequestsPerMinute.ToString();
        context.Response.Headers["X-RateLimit-Remaining"] = (limit.RequestsPerMinute - allowed.Count).ToString();

        await _next(context);
    }

    private async Task<ApiKeyLimit> GetLimitForKeyAsync(string apiKey)
    {
        // In production: load from database/cache
        return apiKey.StartsWith("ent_")
            ? new ApiKeyLimit(10_000)
            : apiKey.StartsWith("pro_")
                ? new ApiKeyLimit(1_000)
                : new ApiKeyLimit(100);
    }

    private async Task<(bool IsAllowed, long Count)> CheckAndIncrementAsync(
        string apiKey,
        ApiKeyLimit limit)
    {
        var db  = _redis.GetDatabase();
        var key = $"apikey:{apiKey}:window";

        const string lua = @"
            local key = KEYS[1]
            local limit = tonumber(ARGV[1])
            local count = redis.call('INCR', key)
            if count == 1 then
                redis.call('EXPIRE', key, 60)
            end
            return count";

        var count   = (long)await db.ScriptEvaluateAsync(lua,
            new RedisKey[]  { key },
            new RedisValue[] { limit.RequestsPerMinute });

        return (count <= limit.RequestsPerMinute, count);
    }
}

public record ApiKeyLimit(int RequestsPerMinute);

// Register
app.UseMiddleware<ApiKeyThrottleMiddleware>();
```

---

## Step 1561: Testing Rate Limiting

```csharp
// Tests/RateLimitingTests.cs
using Microsoft.AspNetCore.Mvc.Testing;
using Microsoft.AspNetCore.RateLimiting;

namespace RateLimitingDemo.Tests;

public class RateLimitingTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;

    public RateLimitingTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                // Override with tight limits for testing
                services.PostConfigure<RateLimiterOptions>(options =>
                {
                    options.AddFixedWindowLimiter("test-limit", o =>
                    {
                        o.PermitLimit = 3;
                        o.Window      = TimeSpan.FromSeconds(10);
                    });
                });
            });
        });
    }

    [Fact]
    public async Task WithinLimit_RequestsSucceed()
    {
        var client = _factory.CreateClient();

        for (int i = 0; i < 3; i++)
        {
            var response = await client.GetAsync("/api/public/products");
            Assert.Equal(HttpStatusCode.OK, response.StatusCode);
        }
    }

    [Fact]
    public async Task ExceedingLimit_Returns429()
    {
        var client = _factory.CreateClient();

        HttpResponseMessage? lastResponse = null;
        for (int i = 0; i < 10; i++)
        {
            lastResponse = await client.GetAsync("/api/public/products");
        }

        Assert.Equal(HttpStatusCode.TooManyRequests, lastResponse!.StatusCode);
        Assert.True(lastResponse.Headers.Contains("Retry-After"));
    }

    [Fact]
    public async Task RejectedResponse_ContainsExpectedHeaders()
    {
        var client = _factory.CreateClient();

        // Exhaust the limit
        for (int i = 0; i < 10; i++)
            await client.GetAsync("/api/public/products");

        var response = await client.GetAsync("/api/public/products");

        Assert.Equal(429, (int)response.StatusCode);
        Assert.True(response.Headers.Contains("Retry-After"));
        Assert.Equal("application/json", response.Content.Headers.ContentType?.MediaType);

        var body = await response.Content.ReadFromJsonAsync<Dictionary<string, object>>();
        Assert.NotNull(body);
        Assert.Equal(429, (int)(body["status"] as System.Text.Json.JsonElement?)!.Value.GetInt32());
    }

    [Fact]
    public async Task PerUserPartition_IsolatesLimits()
    {
        var user1 = _factory.CreateClient();
        var user2 = _factory.CreateClient();

        user1.DefaultRequestHeaders.Authorization =
            new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", CreateJwt("user-1"));
        user2.DefaultRequestHeaders.Authorization =
            new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", CreateJwt("user-2"));

        // Exhaust user1's limit
        for (int i = 0; i < 10; i++)
            await user1.GetAsync("/api/orders");

        // user2 should still succeed
        var response = await user2.GetAsync("/api/orders");
        Assert.Equal(HttpStatusCode.OK, response.StatusCode);
    }

    private static string CreateJwt(string userId) =>
        // Generate test JWT — in practice use Microsoft.IdentityModel.Tokens
        Convert.ToBase64String(System.Text.Encoding.UTF8.GetBytes(
            $"{{\"sub\":\"{userId}\",\"exp\":{DateTimeOffset.UtcNow.AddHours(1).ToUnixTimeSeconds()}}}"));
}
```

---

## Step 1562: Rate Limiting with Polly (Client-Side)

```csharp
// Client-side rate limiting via Polly v8 RateLimiter strategy
using Polly;
using Polly.RateLimiting;
using System.Threading.RateLimiting;

// Register HttpClient with Polly rate limit + retry
builder.Services
    .AddHttpClient<IOrderApiClient, OrderApiClient>(c =>
    {
        c.BaseAddress = new Uri(builder.Configuration["OrderApi:BaseUrl"]!);
    })
    .AddResilienceHandler("order-client", pipeline =>
    {
        // Client-side rate limit (prevent overwhelming the server)
        pipeline.AddRateLimiter(new SlidingWindowRateLimiter(
            new SlidingWindowRateLimiterOptions
            {
                PermitLimit       = 100,
                Window            = TimeSpan.FromMinutes(1),
                SegmentsPerWindow = 6
            }));

        // Timeout per request
        pipeline.AddTimeout(TimeSpan.FromSeconds(15));

        // Retry on 429: respect Retry-After header
        pipeline.AddRetry(new Polly.Retry.RetryStrategyOptions<HttpResponseMessage>
        {
            MaxRetryAttempts = 3,
            BackoffType      = DelayBackoffType.Exponential,
            UseJitter        = true,
            ShouldHandle     = new PredicateBuilder<HttpResponseMessage>()
                .HandleResult(r => r.StatusCode == System.Net.HttpStatusCode.TooManyRequests)
                .HandleResult(r => r.StatusCode == System.Net.HttpStatusCode.ServiceUnavailable),
            DelayGenerator = static args =>
            {
                if (args.Outcome.Result?.Headers.RetryAfter?.Delta is { } delta)
                    return new ValueTask<TimeSpan?>(delta);
                if (args.Outcome.Result?.Headers.RetryAfter?.Date is { } date)
                    return new ValueTask<TimeSpan?>(date - DateTimeOffset.UtcNow);
                return new ValueTask<TimeSpan?>((TimeSpan?)null);  // use default backoff
            }
        });

        // Circuit breaker
        pipeline.AddCircuitBreaker(new Polly.CircuitBreaker.CircuitBreakerStrategyOptions<HttpResponseMessage>
        {
            FailureRatio      = 0.5,
            SamplingDuration  = TimeSpan.FromSeconds(30),
            MinimumThroughput = 20,
            BreakDuration     = TimeSpan.FromSeconds(30),
            ShouldHandle      = new PredicateBuilder<HttpResponseMessage>()
                .HandleResult(r => (int)r.StatusCode >= 500)
        });
    });
```

---

## Step 1563: Observability — Rate Limit Metrics

```csharp
// RateLimitMetricsMiddleware.cs
public class RateLimitMetricsMiddleware
{
    private static readonly Meter   Meter        = new("RateLimitingDemo", "1.0.0");
    private static readonly Counter<long>    RequestsAllowed  = Meter.CreateCounter<long>("ratelimit.requests.allowed");
    private static readonly Counter<long>    RequestsRejected = Meter.CreateCounter<long>("ratelimit.requests.rejected");
    private static readonly Histogram<double> QueueWaitTime   = Meter.CreateHistogram<double>("ratelimit.queue.wait_ms", "ms");

    private readonly RequestDelegate _next;

    public RateLimitMetricsMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(HttpContext context)
    {
        var sw = System.Diagnostics.Stopwatch.StartNew();

        await _next(context);

        sw.Stop();
        var tags = new TagList
        {
            { "route", context.GetEndpoint()?.DisplayName ?? "unknown" },
            { "status", context.Response.StatusCode }
        };

        if (context.Response.StatusCode == 429)
            RequestsRejected.Add(1, tags);
        else
            RequestsAllowed.Add(1, tags);

        if (context.Items.ContainsKey("RateLimitQueued"))
            QueueWaitTime.Record(sw.Elapsed.TotalMilliseconds, tags);
    }
}

// Grafana dashboard query examples:
// Rate of 429s: rate(ratelimit_requests_rejected_total[1m])
// Rejection ratio: rate(rejected[1m]) / (rate(allowed[1m]) + rate(rejected[1m]))
// p99 queue wait: histogram_quantile(0.99, rate(ratelimit_queue_wait_ms_bucket[5m]))
```

---

## Step 1564: Complete appsettings Configuration

```json
{
  "RateLimiting": {
    "DefaultPolicy": "per-user",
    "Policies": {
      "public":     { "Type": "FixedWindow",   "PermitLimit": 60,   "Window": "00:01:00" },
      "auth":       { "Type": "SlidingWindow",  "PermitLimit": 300,  "Window": "00:01:00" },
      "write":      { "Type": "TokenBucket",    "TokenLimit": 50,    "RefillRate": 10,     "Period": "00:00:10" },
      "streaming":  { "Type": "Concurrency",    "PermitLimit": 20 }
    },
    "Tiers": {
      "free":       { "PerMinute": 60,   "PerDay": 1000  },
      "pro":        { "PerMinute": 600,  "PerDay": 10000 },
      "enterprise": { "PerMinute": 6000, "PerDay": 100000 }
    },
    "Redis": {
      "Enabled": true,
      "KeyPrefix": "rl:",
      "ConnectionString": "localhost:6379"
    }
  },

  "ReverseProxy": {
    "Routes": {
      "orders": {
        "ClusterId": "orders",
        "Match": { "Path": "/api/orders/{**rest}" }
      }
    },
    "Clusters": {
      "orders": {
        "LoadBalancingPolicy": "RoundRobin",
        "Destinations": {
          "primary":  { "Address": "http://orders-1:5001" },
          "secondary": { "Address": "http://orders-2:5001" }
        }
      }
    }
  }
}
```

---

## Step 1565: Summary & Production Checklist

```
Rate Limiting & API Gateway — Production Checklist
═══════════════════════════════════════════════════════════

Rate Limiting
✅ Algorithm choice:
   - Fixed Window: simple quotas (hourly API limits)
   - Sliding Window: smooth traffic (per-minute APIs)
   - Token Bucket: burst-friendly with steady refill
   - Concurrency: protect expensive operations
✅ Partitioning strategy:
   - Per-IP for anonymous (unauthenticated)
   - Per-user/API-key for authenticated
   - Per-tier for subscription-based limits
✅ Redis backing for multi-instance deployments
✅ 429 response with Retry-After header
✅ X-RateLimit-Limit / Remaining / Reset headers
✅ Exclude health checks from rate limiting
✅ Client-side rate limiting (Polly) to protect downstream

YARP Gateway
✅ Load balancing: RoundRobin / LeastRequests / PowerOfTwo
✅ Session affinity for stateful services (WebSockets)
✅ Active + passive health checks
✅ Circuit breaker per cluster (Polly)
✅ Request/response transforms (headers, path rewrites)
✅ gRPC + WebSocket proxying
✅ TLS termination + certificate validation
✅ Dynamic route updates without restart
✅ Canary / A/B routing via metadata

Observability
✅ Counter: requests allowed vs rejected by policy
✅ Histogram: queue wait time
✅ Tracing: span per proxied request
✅ Grafana dashboard: rejection rate, queue depth

Security
✅ Rate limit on auth endpoints (brute force protection)
✅ Per-API-key quotas in Redis
✅ IP allowlist/denylist via middleware
✅ DDoS mitigation at load balancer level
```

---

**Part 57 ครอบคลุม:**
- `System.Threading.RateLimiting` ทั้ง 4 algorithms + partitioned policies
- Redis-backed distributed limiter ด้วย Lua atomic scripts
- YARP Reverse Proxy — routing, load balancing, transforms, session affinity
- Circuit breaking ผ่าน Polly v8 `AddResilienceHandler`
- Client-side rate limiting ด้วย Polly + Retry-After header respect
- การ test rate limits ด้วย `WebApplicationFactory`
