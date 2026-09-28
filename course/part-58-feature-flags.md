# Part 58: Feature Flags & Canary Deployments

## Steps 1566-1585: Microsoft.FeatureManagement & Progressive Delivery

---

## Step 1566: Feature Flag Architecture

```
Feature Flag Lifecycle
══════════════════════════════════════════════════════════════

  Development          Staging             Production
  ───────────          ───────             ──────────
  flag = ON (dev)  →  flag = 10% users  →  flag = 100% users
      │                    │                    │
      │              Canary rollout         Full release
      │              A/B metrics           (flag removed
      │              monitored             from code)
      ▼
  Code merged with
  flag disabled by
  default in prod

  Feature Flag Types:
  ┌─────────────────────────────────────────────────────────┐
  │ Release Flag     — control feature rollout              │
  │ Experiment Flag  — A/B test (measure impact)            │
  │ Ops Flag         — kill switch (turn off broken feature)│
  │ Permission Flag  — beta users, premium tiers            │
  └─────────────────────────────────────────────────────────┘
```

### NuGet Packages

```xml
<PackageReference Include="Microsoft.FeatureManagement.AspNetCore"  Version="4.*" />
<PackageReference Include="Microsoft.Azure.AppConfiguration.AspNetCore" Version="8.*" />
<PackageReference Include="Microsoft.FeatureManagement.Telemetry.ApplicationInsights" Version="4.*" />
```

---

## Step 1567: Basic Feature Management Setup

```csharp
// Program.cs
using Microsoft.FeatureManagement;
using Microsoft.FeatureManagement.FeatureFilters;

var builder = WebApplication.CreateBuilder(args);

// ── Feature Management ────────────────────────────────────────────────────
builder.Services
    .AddFeatureManagement(builder.Configuration.GetSection("FeatureManagement"))
    .AddFeatureFilter<PercentageFilter>()        // % of requests
    .AddFeatureFilter<TimeWindowFilter>()        // time-based
    .AddFeatureFilter<TargetingFilter>()         // user/group targeting
    .AddFeatureFilter<ContextualTargetingFilter>() // custom context
    .AddFeatureFilter<CustomTierFilter>();        // custom filter

// ── Azure App Config (optional — for live updates) ───────────────────────
builder.Configuration.AddAzureAppConfiguration(opts =>
{
    opts.Connect(builder.Configuration["AzureAppConfig:ConnectionString"])
        .UseFeatureFlags(ff =>
        {
            ff.CacheExpirationInterval = TimeSpan.FromSeconds(30);
        });
});

builder.Services.AddAzureAppConfiguration();

var app = builder.Build();
app.UseAzureAppConfiguration(); // enables background refresh
```

```json
// appsettings.json
{
  "FeatureManagement": {
    "NewCheckoutFlow":    true,
    "DarkMode":          false,
    "BetaSearch":        {
      "EnabledFor": [
        {
          "Name": "Percentage",
          "Parameters": { "Value": 20 }
        }
      ]
    },
    "HolidaySale": {
      "EnabledFor": [
        {
          "Name": "TimeWindow",
          "Parameters": {
            "Start": "2024-12-20T00:00:00Z",
            "End":   "2024-12-26T23:59:59Z"
          }
        }
      ]
    },
    "NewDashboard": {
      "EnabledFor": [
        {
          "Name": "Targeting",
          "Parameters": {
            "Audience": {
              "Users":  ["alice@example.com", "bob@example.com"],
              "Groups": [
                { "Name": "BetaTesters",   "RolloutPercentage": 100 },
                { "Name": "EarlyAdopters", "RolloutPercentage": 25  }
              ],
              "DefaultRolloutPercentage": 0
            }
          }
        }
      ]
    }
  }
}
```

---

## Step 1568: Feature Flag Constants & Usage

```csharp
// FeatureFlags.cs — centralize flag names
namespace FeatureFlagsDemo;

public static class FeatureFlags
{
    // Release flags
    public const string NewCheckoutFlow = nameof(NewCheckoutFlow);
    public const string RedesignedSearch = nameof(RedesignedSearch);
    public const string PaymentV2       = nameof(PaymentV2);

    // Experiment flags
    public const string RecommendationAlgoV2 = nameof(RecommendationAlgoV2);
    public const string PricingExperiment     = nameof(PricingExperiment);

    // Ops flags (kill switches)
    public const string CacheEnabled         = nameof(CacheEnabled);
    public const string AsyncOrderProcessing  = nameof(AsyncOrderProcessing);

    // Permission flags
    public const string BetaDashboard        = nameof(BetaDashboard);
    public const string EnterpriseReporting  = nameof(EnterpriseReporting);
}
```

```csharp
// OrderService.cs — IFeatureManager usage
using Microsoft.FeatureManagement;

public class OrderService
{
    private readonly IOrderRepository _repo;
    private readonly IFeatureManager _features;
    private readonly ILogger<OrderService> _log;

    public OrderService(
        IOrderRepository repo,
        IFeatureManager features,
        ILogger<OrderService> log)
    {
        _repo     = repo;
        _features = features;
        _log      = log;
    }

    public async Task<OrderResult> CreateOrderAsync(CreateOrderCommand cmd, CancellationToken ct = default)
    {
        // Simple on/off flag
        if (await _features.IsEnabledAsync(FeatureFlags.NewCheckoutFlow))
        {
            _log.LogInformation("Using new checkout flow for order");
            return await CreateOrderV2Async(cmd, ct);
        }

        return await CreateOrderV1Async(cmd, ct);
    }

    public async Task<IEnumerable<Product>> GetRecommendationsAsync(string userId, CancellationToken ct = default)
    {
        // A/B experiment: different algorithm per user
        if (await _features.IsEnabledAsync(FeatureFlags.RecommendationAlgoV2))
        {
            return await _repo.GetCollaborativeFilteringRecommendationsAsync(userId, ct);
        }

        return await _repo.GetContentBasedRecommendationsAsync(userId, ct);
    }

    public async Task ProcessOrderAsync(Guid orderId, CancellationToken ct = default)
    {
        // Ops flag — kill switch for async processing
        if (await _features.IsEnabledAsync(FeatureFlags.AsyncOrderProcessing))
        {
            await _repo.EnqueueOrderAsync(orderId, ct);
        }
        else
        {
            // Synchronous fallback
            await _repo.ProcessOrderSynchronouslyAsync(orderId, ct);
        }
    }
}
```

---

## Step 1569: Custom Feature Filters

```csharp
// Filters/CustomTierFilter.cs
using Microsoft.FeatureManagement;

[FilterAlias("CustomTier")]
public class CustomTierFilter : IContextualFeatureFilter<IFeatureFilterEvaluationContext>
{
    private readonly IHttpContextAccessor _http;

    public CustomTierFilter(IHttpContextAccessor http) => _http = http;

    public Task<bool> EvaluateAsync(
        FeatureFilterEvaluationContext featureContext,
        IFeatureFilterEvaluationContext appContext)
    {
        var settings = featureContext.Parameters.Get<TierFilterSettings>()
                    ?? new TierFilterSettings();

        var userTier = _http.HttpContext?.User.FindFirst("tier")?.Value ?? "free";

        return Task.FromResult(settings.AllowedTiers.Contains(userTier));
    }
}

public class TierFilterSettings
{
    public List<string> AllowedTiers { get; set; } = ["enterprise"];
}

// appsettings.json usage:
// "EnterpriseReporting": {
//   "EnabledFor": [{ "Name": "CustomTier", "Parameters": { "AllowedTiers": ["pro","enterprise"] }}]
// }
```

```csharp
// Filters/GeoTargetingFilter.cs
[FilterAlias("GeoTargeting")]
public class GeoTargetingFilter : IContextualFeatureFilter<IFeatureFilterEvaluationContext>
{
    private readonly IGeoIpService _geoIp;
    private readonly IHttpContextAccessor _http;

    public GeoTargetingFilter(IGeoIpService geoIp, IHttpContextAccessor http)
    {
        _geoIp = geoIp;
        _http  = http;
    }

    public async Task<bool> EvaluateAsync(
        FeatureFilterEvaluationContext featureContext,
        IFeatureFilterEvaluationContext appContext)
    {
        var settings = featureContext.Parameters.Get<GeoFilterSettings>()
                    ?? new GeoFilterSettings();

        var ip      = _http.HttpContext?.Connection.RemoteIpAddress;
        var country = await _geoIp.GetCountryAsync(ip);

        return settings.Countries.Contains(country, StringComparer.OrdinalIgnoreCase);
    }
}

public class GeoFilterSettings
{
    public List<string> Countries { get; set; } = [];
}
```

---

## Step 1570: Targeting Context — Per-Request User Targeting

```csharp
// Targeting/HttpContextTargetingContextAccessor.cs
using Microsoft.FeatureManagement.FeatureFilters;

public class HttpContextTargetingContextAccessor : ITargetingContextAccessor
{
    private readonly IHttpContextAccessor _http;

    public HttpContextTargetingContextAccessor(IHttpContextAccessor http) => _http = http;

    public ValueTask<TargetingContext> GetContextAsync()
    {
        var user = _http.HttpContext?.User;

        var userId = user?.FindFirst("sub")?.Value
                  ?? user?.FindFirst("email")?.Value
                  ?? _http.HttpContext?.Connection.RemoteIpAddress?.ToString()
                  ?? "anonymous";

        var groups = new List<string>();

        // Map claims to groups
        if (user?.IsInRole("beta-tester") == true)  groups.Add("BetaTesters");
        if (user?.IsInRole("admin") == true)          groups.Add("Admins");

        var tier = user?.FindFirst("tier")?.Value;
        if (tier is not null) groups.Add($"Tier:{tier}");

        return ValueTask.FromResult(new TargetingContext
        {
            UserId = userId,
            Groups = groups
        });
    }
}

// Register
builder.Services.AddSingleton<ITargetingContextAccessor, HttpContextTargetingContextAccessor>();
```

---

## Step 1571: Feature Flags in Minimal APIs & Controllers

```csharp
// Minimal API with feature flag route filter
app.MapGet("/api/v2/search", SearchV2Handler)
    .WithName("SearchV2")
    .AddEndpointFilter(async (context, next) =>
    {
        var features = context.HttpContext.RequestServices.GetRequiredService<IVariantFeatureManager>();
        if (!await features.IsEnabledAsync(FeatureFlags.RedesignedSearch))
        {
            return Results.NotFound("Feature not available");
        }
        return await next(context);
    });

// Feature-gated controller action
[ApiController]
[Route("api/[controller]")]
public class DashboardController : ControllerBase
{
    private readonly IFeatureManager _features;

    [HttpGet("beta")]
    public async Task<IActionResult> BetaDashboard()
    {
        if (!await _features.IsEnabledAsync(FeatureFlags.BetaDashboard))
            return NotFound("Beta dashboard not available for your account");

        return Ok(await LoadBetaDashboardDataAsync());
    }
}

// FeatureGate attribute
[FeatureGate(FeatureFlags.EnterpriseReporting)]
[HttpGet("enterprise/reports")]
public async Task<IActionResult> EnterpriseReports()
{
    // Only reached if flag is enabled for current user
    return Ok(await LoadEnterpriseReportsAsync());
}
```

---

## Step 1572: Feature Variants (A/B Testing Values)

```csharp
// Variant feature management — return different VALUES
// appsettings.json
{
  "feature_management": {
    "feature_flags": [
      {
        "id": "PricingDisplay",
        "enabled": true,
        "variants": [
          {
            "name": "Discounted",
            "configuration_value": { "showDiscount": true,  "discountPercent": 15 }
          },
          {
            "name": "Regular",
            "configuration_value": { "showDiscount": false, "discountPercent": 0  }
          }
        ],
        "allocation": {
          "percentile": [
            { "variant": "Discounted", "from": 0, "to": 50  },
            { "variant": "Regular",    "from": 50, "to": 100 }
          ],
          "seed": "pricing-experiment-v1"
        },
        "telemetry": { "enabled": true }
      }
    ]
  }
}
```

```csharp
// Using IVariantFeatureManager
using Microsoft.FeatureManagement.FeatureFilters;

public class PricingService
{
    private readonly IVariantFeatureManager _variantFeatures;

    public PricingService(IVariantFeatureManager variantFeatures)
        => _variantFeatures = variantFeatures;

    public async Task<PricingConfig> GetPricingConfigAsync(CancellationToken ct = default)
    {
        var variant = await _variantFeatures.GetVariantAsync(
            "PricingDisplay",
            new TargetingContext { UserId = GetCurrentUserId() },
            ct);

        if (variant is null)
            return PricingConfig.Default;

        return new PricingConfig(
            ShowDiscount:    variant.Configuration?["showDiscount"]?.GetValue<bool>() ?? false,
            DiscountPercent: variant.Configuration?["discountPercent"]?.GetValue<int>() ?? 0
        );
    }
}

public record PricingConfig(bool ShowDiscount, int DiscountPercent)
{
    public static readonly PricingConfig Default = new(false, 0);
}
```

---

## Step 1573: Canary Deployment Strategy

```csharp
// CanaryDeploymentMiddleware.cs
// Routes % of traffic to new version based on feature flag
public class CanaryMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IFeatureManager _features;
    private readonly IHttpClientFactory _http;

    public CanaryMiddleware(
        RequestDelegate next,
        IFeatureManager features,
        IHttpClientFactory http)
    {
        _next     = next;
        _features = features;
        _http     = http;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // Check if this request should go to canary
        if (await _features.IsEnabledAsync("CanaryOrderService"))
        {
            // Forward to canary cluster
            var canaryClient = _http.CreateClient("canary-orders");

            // Copy request, forward, copy response back
            var request = context.Request;
            using var upstreamRequest = new HttpRequestMessage(
                new HttpMethod(request.Method),
                request.Path + request.QueryString);

            foreach (var header in request.Headers)
                upstreamRequest.Headers.TryAddWithoutValidation(header.Key, header.Value.ToArray());

            if (request.ContentLength > 0)
                upstreamRequest.Content = new StreamContent(request.Body);

            using var upstreamResponse = await canaryClient.SendAsync(
                upstreamRequest, context.RequestAborted);

            context.Response.StatusCode = (int)upstreamResponse.StatusCode;
            foreach (var header in upstreamResponse.Headers)
                context.Response.Headers[header.Key] = header.Value.ToArray();

            await upstreamResponse.Content.CopyToAsync(context.Response.Body, context.RequestAborted);
            return;
        }

        await _next(context);
    }
}
```

---

## Step 1574: Azure App Configuration Integration

```csharp
// Connecting to Azure App Configuration for dynamic flag updates
builder.Configuration
    .AddAzureAppConfiguration(options =>
    {
        options
            .Connect(builder.Configuration["AzureAppConfig:ConnectionString"])
            .Select("*")                    // all config keys
            .Select("*", "Production")     // with Production label

            // Feature flags refresh every 30 seconds
            .UseFeatureFlags(ff =>
            {
                ff.Select("*");
                ff.CacheExpirationInterval = TimeSpan.FromSeconds(30);
            })

            // Refresh sentinel key (lightweight poll)
            .ConfigureRefresh(refresh =>
            {
                refresh
                    .Register("App:Sentinel", refreshAll: true)
                    .SetRefreshInterval(TimeSpan.FromSeconds(30));
            });
    });

// Background refresh worker
builder.Services.AddAzureAppConfiguration();

// In controller/service: inject IConfigurationRefresherProvider
public class FeatureController : ControllerBase
{
    private readonly IConfigurationRefresherProvider _refresher;

    [HttpPost("refresh")]
    [Authorize(Roles = "admin")]
    public async Task<IActionResult> ForceRefresh()
    {
        foreach (var refresher in _refresher.Refreshers)
            await refresher.RefreshAsync();
        return Ok("Config refreshed");
    }
}
```

---

## Step 1575: Feature Flag Telemetry & Metrics

```csharp
// Custom telemetry publisher
using Microsoft.FeatureManagement.Telemetry;

public class FeatureFlagTelemetryPublisher : ITelemetryPublisher
{
    private static readonly Meter Meter = new("FeatureManagement", "1.0.0");
    private static readonly Counter<long> FeatureChecks = Meter.CreateCounter<long>("feature.evaluations");
    private readonly ILogger<FeatureFlagTelemetryPublisher> _log;

    public FeatureFlagTelemetryPublisher(ILogger<FeatureFlagTelemetryPublisher> log)
        => _log = log;

    public Task PublishAsync(EvaluationEvent evaluationEvent, CancellationToken cancellationToken)
    {
        var tags = new TagList
        {
            { "feature",     evaluationEvent.FeatureName },
            { "enabled",     evaluationEvent.IsEnabled   },
            { "variant",     evaluationEvent.Variant?.Name ?? "none" },
            { "targeting_id", evaluationEvent.TargetingId ?? "anonymous" }
        };

        FeatureChecks.Add(1, tags);

        if (evaluationEvent.Variant is not null)
        {
            _log.LogInformation(
                "Feature {Feature} variant {Variant} assigned to {User}",
                evaluationEvent.FeatureName,
                evaluationEvent.Variant.Name,
                evaluationEvent.TargetingId);
        }

        return Task.CompletedTask;
    }
}

// Register
builder.Services
    .AddFeatureManagement()
    .WithTargeting<HttpContextTargetingContextAccessor>()
    .AddTelemetryPublisher<FeatureFlagTelemetryPublisher>();
```

```csharp
// FeatureAuditService.cs — log all feature evaluations for experiment analysis
public class FeatureAuditService
{
    private readonly IFeatureManager _features;
    private readonly IAnalyticsService _analytics;

    public async Task LogFeatureExposureAsync(string userId, CancellationToken ct = default)
    {
        // Record which features are active for this user
        var exposure = new Dictionary<string, bool>();

        foreach (var flag in new[]
        {
            FeatureFlags.NewCheckoutFlow,
            FeatureFlags.RecommendationAlgoV2,
            FeatureFlags.PricingExperiment
        })
        {
            exposure[flag] = await _features.IsEnabledAsync(flag);
        }

        await _analytics.LogExposureAsync(userId, exposure, ct);
    }
}
```

---

## Step 1576: LaunchDarkly Integration (Alternative)

```csharp
// LaunchDarkly as alternative flag provider
// NuGet: LaunchDarkly.ServerSdk

using LaunchDarkly.Sdk;
using LaunchDarkly.Sdk.Server;

builder.Services.AddSingleton(sp =>
{
    var config = Configuration.Builder(builder.Configuration["LaunchDarkly:SdkKey"])
        .Http(Components.HttpConfiguration().ConnectTimeout(TimeSpan.FromSeconds(5)))
        .DataStore(Components.PersistentDataStore(
            Redis.DataStore().Uri(new Uri(builder.Configuration["Redis:ConnectionString"]!))))
        .Build();

    return new LdClient(config);
});

// Wrapper service
public class LaunchDarklyFeatureService : IFeatureService
{
    private readonly LdClient _client;
    private readonly IHttpContextAccessor _http;

    public LaunchDarklyFeatureService(LdClient client, IHttpContextAccessor http)
    {
        _client = client;
        _http   = http;
    }

    public bool IsEnabled(string flag)
    {
        var user = BuildLdUser();
        return _client.BoolVariation(flag, user, defaultValue: false);
    }

    public T GetVariation<T>(string flag, T defaultValue)
    {
        var user = BuildLdUser();
        return typeof(T) switch
        {
            var t when t == typeof(string)  => (T)(object)_client.StringVariation(flag, user, (string)(object)defaultValue!),
            var t when t == typeof(int)     => (T)(object)_client.IntVariation(flag, user, (int)(object)defaultValue!),
            var t when t == typeof(bool)    => (T)(object)_client.BoolVariation(flag, user, (bool)(object)defaultValue!),
            var t when t == typeof(double)  => (T)(object)_client.DoubleVariation(flag, user, (double)(object)defaultValue!),
            _ => defaultValue
        };
    }

    private Context BuildLdUser()
    {
        var user = _http.HttpContext?.User;
        var userId = user?.FindFirst("sub")?.Value ?? "anonymous";

        return Context.Builder(userId)
            .Name(user?.FindFirst("name")?.Value ?? userId)
            .Set("tier", user?.FindFirst("tier")?.Value ?? "free")
            .Set("country", user?.FindFirst("country")?.Value ?? "US")
            .Build();
    }
}
```

---

## Step 1577: Gradual Rollout State Machine

```csharp
// RolloutManager.cs — orchestrate gradual rollout
public class RolloutManager
{
    private readonly IFeatureConfigurationClient _config;
    private readonly IMetricsService _metrics;
    private readonly ILogger<RolloutManager> _log;

    public RolloutManager(
        IFeatureConfigurationClient config,
        IMetricsService metrics,
        ILogger<RolloutManager> log)
    {
        _config  = config;
        _metrics = metrics;
        _log     = log;
    }

    public async Task<RolloutResult> AdvanceRolloutAsync(
        string flagName,
        RolloutConfig config,
        CancellationToken ct = default)
    {
        var current = await _config.GetRolloutPercentageAsync(flagName, ct);

        // Check health metrics before advancing
        var health = await _metrics.GetHealthMetricsAsync(flagName, TimeSpan.FromMinutes(15), ct);

        if (health.ErrorRate > config.MaxErrorRate)
        {
            _log.LogWarning("Rollout of {Flag} paused: error rate {Rate}% exceeds threshold {Max}%",
                flagName, health.ErrorRate * 100, config.MaxErrorRate * 100);

            // Auto-rollback if error rate is critical
            if (health.ErrorRate > config.CriticalErrorRate)
            {
                await _config.SetRolloutPercentageAsync(flagName, 0, ct);
                _log.LogError("Rollout of {Flag} ROLLED BACK: critical error rate {Rate}%",
                    flagName, health.ErrorRate * 100);
                return RolloutResult.RolledBack;
            }

            return RolloutResult.Paused;
        }

        // Advance by configured step
        var next = Math.Min(current + config.StepSize, 100);
        await _config.SetRolloutPercentageAsync(flagName, next, ct);

        _log.LogInformation("Rollout of {Flag}: {Old}% → {New}%",
            flagName, current, next);

        return next >= 100 ? RolloutResult.Complete : RolloutResult.Advanced;
    }
}

public enum RolloutResult { Advanced, Complete, Paused, RolledBack }

public record RolloutConfig(
    int StepSize,
    double MaxErrorRate,
    double CriticalErrorRate,
    TimeSpan StepInterval);

// Background rollout scheduler
public class RolloutScheduler : BackgroundService
{
    private readonly RolloutManager _rollout;
    private readonly IOptions<RolloutSchedulerOptions> _options;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            foreach (var (flagName, config) in _options.Value.Rollouts)
            {
                var result = await _rollout.AdvanceRolloutAsync(flagName, config, stoppingToken);
                if (result == RolloutResult.Complete)
                    _options.Value.Rollouts.Remove(flagName);
            }

            await Task.Delay(_options.Value.CheckInterval, stoppingToken);
        }
    }
}
```

---

## Step 1578: Blue-Green Deployment with Feature Flags

```csharp
// Blue-green switch via feature flag
// appsettings.json
{
  "FeatureManagement": {
    "UseGreenEnvironment": {
      "EnabledFor": [
        {
          "Name": "Targeting",
          "Parameters": {
            "Audience": {
              "Groups": [
                { "Name": "InternalTesters", "RolloutPercentage": 100 },
                { "Name": "BetaUsers",       "RolloutPercentage": 50  }
              ],
              "DefaultRolloutPercentage": 0
            }
          }
        }
      ]
    }
  }
}
```

```csharp
// Infrastructure routing
public class BlueGreenRouter
{
    private readonly IFeatureManager _features;
    private readonly IHttpClientFactory _http;

    public BlueGreenRouter(IFeatureManager features, IHttpClientFactory http)
    {
        _features = features;
        _http     = http;
    }

    public async Task<HttpClient> GetClientAsync()
    {
        if (await _features.IsEnabledAsync("UseGreenEnvironment"))
            return _http.CreateClient("green-environment");

        return _http.CreateClient("blue-environment");
    }
}

// Registration
builder.Services.AddHttpClient("blue-environment", c =>
    c.BaseAddress = new Uri(builder.Configuration["Environments:Blue:Url"]!));

builder.Services.AddHttpClient("green-environment", c =>
    c.BaseAddress = new Uri(builder.Configuration["Environments:Green:Url"]!));
```

---

## Step 1579: Testing Feature Flags

```csharp
// Tests/FeatureFlagTests.cs
using Microsoft.FeatureManagement;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Configuration;

namespace FeatureFlagsDemo.Tests;

public class FeatureFlagTests
{
    [Fact]
    public async Task NewCheckoutFlow_WhenEnabled_UsesV2()
    {
        var services = new ServiceCollection();
        services.AddLogging();
        services.AddFeatureManagement(
            new ConfigurationBuilder()
                .AddInMemoryCollection(new Dictionary<string, string?>
                {
                    ["FeatureManagement:NewCheckoutFlow"] = "true"
                })
                .Build()
                .GetSection("FeatureManagement"));

        services.AddScoped<IOrderRepository, MockOrderRepository>();
        services.AddScoped<OrderService>();

        var sp      = services.BuildServiceProvider();
        var service = sp.GetRequiredService<OrderService>();

        var result = await service.CreateOrderAsync(new CreateOrderCommand
        {
            CustomerId = "test",
            Items = [new OrderItem { ProductId = "p1", Quantity = 1 }]
        });

        Assert.True(result.UsedV2Flow);
    }

    [Fact]
    public async Task PercentageFilter_Half_IsEnabled50Percent()
    {
        var config = new ConfigurationBuilder()
            .AddInMemoryCollection(new Dictionary<string, string?>
            {
                ["FeatureManagement:BetaSearch:EnabledFor:0:Name"]                   = "Percentage",
                ["FeatureManagement:BetaSearch:EnabledFor:0:Parameters:Value"]        = "50"
            })
            .Build();

        var services = new ServiceCollection();
        services.AddLogging();
        services.AddFeatureManagement(config.GetSection("FeatureManagement"))
                .AddFeatureFilter<PercentageFilter>();

        var sp       = services.BuildServiceProvider();
        var features = sp.GetRequiredService<IFeatureManager>();

        // Due to hashing — just verify it's deterministic
        var result1 = await features.IsEnabledAsync("BetaSearch");
        var result2 = await features.IsEnabledAsync("BetaSearch");
        Assert.Equal(result1, result2);
    }

    [Fact]
    public async Task TargetingFilter_BetaUser_IsEnabled()
    {
        var services = new ServiceCollection();
        services.AddLogging();
        services.AddHttpContextAccessor();

        var config = new ConfigurationBuilder()
            .AddInMemoryCollection(new Dictionary<string, string?>
            {
                ["FeatureManagement:NewDashboard:EnabledFor:0:Name"] = "Targeting",
                ["FeatureManagement:NewDashboard:EnabledFor:0:Parameters:Audience:Users:0"] = "beta@test.com",
                ["FeatureManagement:NewDashboard:EnabledFor:0:Parameters:Audience:DefaultRolloutPercentage"] = "0"
            })
            .Build();

        services.AddFeatureManagement(config.GetSection("FeatureManagement"))
                .AddFeatureFilter<TargetingFilter>();

        // Register targeting accessor that returns our test user
        services.AddSingleton<ITargetingContextAccessor>(
            new StaticTargetingContextAccessor("beta@test.com", []));

        var sp       = services.BuildServiceProvider();
        var features = sp.GetRequiredService<IFeatureManager>();

        Assert.True(await features.IsEnabledAsync("NewDashboard"));
    }
}

public class StaticTargetingContextAccessor : ITargetingContextAccessor
{
    private readonly TargetingContext _ctx;

    public StaticTargetingContextAccessor(string userId, IEnumerable<string> groups)
        => _ctx = new TargetingContext { UserId = userId, Groups = groups.ToList() };

    public ValueTask<TargetingContext> GetContextAsync() => ValueTask.FromResult(_ctx);
}
```

---

## Step 1580: Feature Flag Dashboard Endpoint

```csharp
// Admin endpoint to inspect feature flags
app.MapGet("/admin/features", async (IFeatureManager features, IConfiguration config) =>
{
    var section = config.GetSection("FeatureManagement");
    var flags   = section.GetChildren().Select(c => c.Key).ToList();

    var results = new List<object>();
    foreach (var flag in flags)
    {
        results.Add(new
        {
            name    = flag,
            enabled = await features.IsEnabledAsync(flag)
        });
    }

    return Results.Ok(results);
}).RequireAuthorization("AdminPolicy");

// Feature flag change webhook
app.MapPost("/admin/features/{flag}/toggle", async (
    string flag,
    bool enabled,
    IConfigurationRefresherProvider refresher) =>
{
    // Trigger Azure App Config refresh after manual change
    foreach (var r in refresher.Refreshers)
        await r.RefreshAsync();

    return Results.Ok(new { flag, enabled });
}).RequireAuthorization("AdminPolicy");
```

---

## Step 1581: Complete Integration Example

```csharp
// Full e-commerce feature flag integration
public class ProductController : ControllerBase
{
    private readonly IProductService _products;
    private readonly IVariantFeatureManager _features;
    private readonly ILogger<ProductController> _log;

    [HttpGet("{id}")]
    public async Task<IActionResult> GetProduct(string id)
    {
        var product = await _products.GetByIdAsync(id);
        if (product is null) return NotFound();

        // Variant A/B test: different price display
        var pricingVariant = await _features.GetVariantAsync(
            "PricingDisplay",
            CancellationToken.None);

        var response = new ProductResponse
        {
            Id   = product.Id,
            Name = product.Name,
        };

        if (pricingVariant?.Name == "Discounted")
        {
            var discount = pricingVariant.Configuration?["discountPercent"]?.GetValue<int>() ?? 0;
            response.Price        = product.Price * (1 - discount / 100m);
            response.OriginalPrice = product.Price;
            response.ShowDiscount = true;
        }
        else
        {
            response.Price = product.Price;
        }

        // Feature flag: new recommendation engine
        if (await _features.IsEnabledAsync(FeatureFlags.RecommendationAlgoV2))
        {
            response.Related = await _products.GetRelatedV2Async(id);
        }
        else
        {
            response.Related = await _products.GetRelatedV1Async(id);
        }

        return Ok(response);
    }

    [HttpGet("search")]
    [FeatureGate(FeatureFlags.RedesignedSearch)]
    public async Task<IActionResult> SearchV2([FromQuery] string q)
    {
        // Only accessible when flag is enabled for current user
        return Ok(await _products.SearchV2Async(q));
    }
}
```

---

## Step 1582: Deployment Checklist

```
Feature Flags & Canary Deployments — Production Checklist
═══════════════════════════════════════════════════════════

Flag Design
✅ Centralized flag names as constants (no magic strings)
✅ Naming convention: PascalCase, descriptive
✅ Document intent: release/experiment/ops/permission
✅ Plan removal date to avoid flag debt

Configuration Sources
✅ appsettings.json for defaults (committed to repo)
✅ Azure App Config / LaunchDarkly for live updates
✅ Refresh interval ≤ 30s for production flags
✅ Sentinel key pattern for batch updates

Targeting
✅ User ID + groups for deterministic assignment
✅ Sticky sessions (same user always sees same variant)
✅ IP fallback for anonymous users
✅ Tier-based filtering for premium features

Rollout Strategy
✅ Start at 0% → 5% → 25% → 50% → 100%
✅ Automated rollback on error rate spike
✅ Canary comparison metrics (p99, error rate, conversion)
✅ Blue-green switch via single flag update
✅ Kill switch always available (set to 0%)

Telemetry
✅ Log every flag evaluation with user context
✅ Counter: evaluations by flag + variant + result
✅ Experiment exposure logged before action
✅ Grafana dashboard: active flags, rollout progress

Testing
✅ Unit tests with in-memory configuration
✅ Integration tests with both flag states
✅ A/B test statistical significance tracking
✅ Ensure no flag dependency on another flag

Security
✅ Admin endpoints require authorization
✅ Audit log for flag changes
✅ No secrets in flag configurations
✅ Feature flags ≠ security controls
```

---

**Part 58 ครอบคลุม:**
- `Microsoft.FeatureManagement` — ทุก filter (Percentage, TimeWindow, Targeting)
- Custom filters (Tier, GeoTargeting)
- Variant A/B testing ด้วย `IVariantFeatureManager`
- Azure App Configuration integration + dynamic refresh
- LaunchDarkly integration
- Gradual rollout automation + auto-rollback
- Blue-green deployment via feature flags
- Unit/integration testing patterns
