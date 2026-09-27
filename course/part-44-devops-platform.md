# Part 44: Advanced DevOps & Platform Engineering (Steps 1261-1300)

## Step 1261: Platform Engineering Overview

### Platform Engineering vs DevOps

```
Traditional DevOps (Old Model)
─────────────────────────────
Developer → writes code
Dev → sets up CI/CD
Dev → configures K8s
Dev → manages secrets
Dev → monitors application
(Every team reinvents the wheel)

Platform Engineering (New Model)
─────────────────────────────────
Platform Team → builds Internal Developer Platform (IDP)
                → Golden paths, self-service portals
                → Paved roads for teams to follow

Application Teams → focus on business logic
                  → use platform capabilities
                  → deploy via golden path
```

### Internal Developer Platform Components

```
┌──────────────────────────────────────────────────────────────┐
│                Internal Developer Platform                    │
├──────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │   Developer  │  │   Service    │  │  Infrastructure  │  │
│  │    Portal    │  │   Catalog    │  │    Provisioning  │  │
│  │  (Backstage) │  │ (Backstage)  │  │  (Terraform/    │  │
│  │              │  │              │  │   Pulumi)        │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │  CI/CD       │  │  Secrets     │  │  Observability   │  │
│  │  Pipelines   │  │  Management  │  │  (Grafana Stack) │  │
│  │  (GitHub     │  │  (Vault/     │  │                  │  │
│  │   Actions)   │  │   KV)        │  │                  │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

---

## Step 1262: Infrastructure as Code with Pulumi (.NET)

### ทำไมต้องใช้ Pulumi?

| Feature | Terraform | Pulumi |
|---------|-----------|--------|
| Language | HCL | C#, TypeScript, Python, Go |
| State | Remote (S3/TF Cloud) | Pulumi Cloud / S3 |
| Loops/Conditions | Limited | Full language features |
| Testing | terratest | xUnit/NUnit |
| IDE Support | Basic | Full IntelliSense |
| .NET Integration | External | Native |

### Pulumi .NET - Azure Setup

```xml
<!-- Pulumi.Infrastructure.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <Nullable>enable</Nullable>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Pulumi" Version="3.64.0" />
    <PackageReference Include="Pulumi.AzureNative" Version="2.46.0" />
    <PackageReference Include="Pulumi.Random" Version="4.16.0" />
  </ItemGroup>
</Project>
```

### Complete Azure Infrastructure

```csharp
using Pulumi;
using Pulumi.AzureNative.Resources;
using Pulumi.AzureNative.ContainerRegistry;
using Pulumi.AzureNative.OperationalInsights;
using Pulumi.AzureNative.App;
using Pulumi.AzureNative.KeyVault;
using Pulumi.AzureNative.Sql;
using Pulumi.AzureNative.Cache;

return await Pulumi.Deployment.RunAsync<MyAppStack>();

public class MyAppStack : Stack
{
    [Output]
    public Output<string> ContainerAppUrl { get; set; } = default!;

    [Output]
    public Output<string> ContainerRegistryServer { get; set; } = default!;

    public MyAppStack()
    {
        var config = new Config();
        var env = config.Require("environment");
        var location = config.Get("location") ?? "southeastasia";

        // Resource Group
        var rg = new ResourceGroup($"myapp-{env}-rg", new ResourceGroupArgs
        {
            Location = location,
            Tags = new InputMap<string>
            {
                ["environment"] = env,
                ["managedBy"] = "pulumi",
            }
        });

        // Container Registry
        var acr = new Registry($"myapp{env}acr", new RegistryArgs
        {
            ResourceGroupName = rg.Name,
            Location = rg.Location,
            Sku = new SkuArgs { Name = "Basic" },
            AdminUserEnabled = true,
        });

        // Log Analytics
        var logAnalytics = new Workspace($"myapp-{env}-logs", new WorkspaceArgs
        {
            ResourceGroupName = rg.Name,
            Location = rg.Location,
            Sku = new WorkspaceSkuArgs { Name = "PerGB2018" },
            RetentionInDays = 30,
        });

        // Container Apps Environment
        var caEnv = new ManagedEnvironment($"myapp-{env}-env", new ManagedEnvironmentArgs
        {
            ResourceGroupName = rg.Name,
            Location = rg.Location,
            AppLogsConfiguration = new AppLogsConfigurationArgs
            {
                Destination = "log-analytics",
                LogAnalyticsConfiguration = new LogAnalyticsConfigurationArgs
                {
                    CustomerId = logAnalytics.CustomerId,
                    SharedKey = logAnalytics.GetSharedKeys().Apply(k => k.PrimarySharedKey!),
                }
            },
        });

        // SQL Server
        var sqlServer = new Server($"myapp-{env}-sql", new ServerArgs
        {
            ResourceGroupName = rg.Name,
            Location = rg.Location,
            AdministratorLogin = "sqladmin",
            AdministratorLoginPassword = config.RequireSecret("sqlPassword"),
            MinimalTlsVersion = "1.2",
        });

        var sqlDb = new Database($"myapp-{env}-db", new DatabaseArgs
        {
            ResourceGroupName = rg.Name,
            ServerName = sqlServer.Name,
            Sku = new Pulumi.AzureNative.Sql.Inputs.SkuArgs
            {
                Name = env == "prod" ? "GP_S_Gen5_2" : "Basic",
                Tier = env == "prod" ? "GeneralPurpose" : "Basic",
            },
        });

        // Key Vault
        var kv = new Vault($"myapp-{env}-kv", new VaultArgs
        {
            ResourceGroupName = rg.Name,
            Location = rg.Location,
            Properties = new VaultPropertiesArgs
            {
                TenantId = config.Require("tenantId"),
                Sku = new Pulumi.AzureNative.KeyVault.Inputs.SkuArgs
                {
                    Family = "A",
                    Name = SkuName.Standard,
                },
                EnableRbacAuthorization = true,
            },
        });

        // Container App
        var containerApp = new ContainerApp($"myapp-{env}-api", new ContainerAppArgs
        {
            ResourceGroupName = rg.Name,
            Location = rg.Location,
            ManagedEnvironmentId = caEnv.Id,
            Configuration = new ConfigurationArgs
            {
                Ingress = new IngressArgs
                {
                    External = true,
                    TargetPort = 8080,
                    Transport = IngressTransportMethod.Http,
                },
                Registries = [
                    new RegistryCredentialsArgs
                    {
                        Server = acr.LoginServer,
                        Username = acr.Name,
                        PasswordSecretRef = "acr-password",
                    }
                ],
                Secrets = [
                    new SecretArgs
                    {
                        Name = "acr-password",
                        Value = acr.ListRegistryCredentials().Apply(
                            c => c.Passwords![0].Value!),
                    }
                ],
            },
            Template = new TemplateArgs
            {
                Containers = [
                    new ContainerArgs
                    {
                        Name = "api",
                        Image = Output.Format($"{acr.LoginServer}/myapp-api:latest"),
                        Resources = new ContainerResourcesArgs
                        {
                            Cpu = 0.5,
                            Memory = "1.0Gi",
                        },
                        Env = [
                            new EnvironmentVarArgs
                            {
                                Name = "ASPNETCORE_ENVIRONMENT",
                                Value = env == "prod" ? "Production" : "Staging",
                            },
                        ],
                    }
                ],
                Scale = new ScaleArgs
                {
                    MinReplicas = env == "prod" ? 2 : 1,
                    MaxReplicas = env == "prod" ? 20 : 5,
                },
            },
        });

        ContainerAppUrl = containerApp.LatestRevisionFqdn
            .Apply(fqdn => $"https://{fqdn}");
        ContainerRegistryServer = acr.LoginServer;
    }
}
```

---

## Step 1263: Terraform with .NET Projects

### Terraform Module for .NET App

```hcl
# modules/dotnet-app/main.tf
variable "app_name" {}
variable "environment" {}
variable "docker_image" {}
variable "resource_group_name" {}
variable "location" { default = "southeastasia" }
variable "min_replicas" { default = 1 }
variable "max_replicas" { default = 10 }

# Container Apps Environment
resource "azurerm_container_app_environment" "main" {
  name                       = "${var.app_name}-${var.environment}-env"
  location                   = var.location
  resource_group_name        = var.resource_group_name
  log_analytics_workspace_id = azurerm_log_analytics_workspace.main.id
}

resource "azurerm_container_app" "api" {
  name                         = "${var.app_name}-${var.environment}-api"
  container_app_environment_id = azurerm_container_app_environment.main.id
  resource_group_name          = var.resource_group_name
  revision_mode                = "Single"

  identity {
    type = "SystemAssigned"
  }

  template {
    container {
      name   = "api"
      image  = var.docker_image
      cpu    = 0.5
      memory = "1.0Gi"

      env {
        name  = "ASPNETCORE_ENVIRONMENT"
        value = var.environment == "prod" ? "Production" : "Staging"
      }

      readiness_probe {
        transport = "HTTP"
        path      = "/health/ready"
        port      = 8080
      }

      liveness_probe {
        transport = "HTTP"
        path      = "/health/live"
        port      = 8080
      }
    }

    min_replicas = var.min_replicas
    max_replicas = var.max_replicas

    http_scale_rule {
      name                = "http-rule"
      concurrent_requests = 100
    }
  }

  ingress {
    external_enabled = true
    target_port      = 8080
    traffic_weight {
      percentage      = 100
      latest_revision = true
    }
  }
}

output "app_url" {
  value = "https://${azurerm_container_app.api.latest_revision_fqdn}"
}
```

---

## Step 1264: SRE Practices - SLOs and Error Budgets

### SLO Definition and Tracking

```csharp
public class SloConfiguration
{
    public string ServiceName { get; init; } = string.Empty;
    public required IReadOnlyList<ServiceLevelObjective> Objectives { get; init; }
}

public class ServiceLevelObjective
{
    public string Name { get; init; } = string.Empty;
    public string Description { get; init; } = string.Empty;
    public double Target { get; init; }       // e.g., 0.999 = 99.9%
    public TimeSpan Window { get; init; }     // e.g., 30 days
    public SloMetricType MetricType { get; init; }
}

public enum SloMetricType
{
    Availability,   // % requests that return 2xx
    Latency,        // % requests under threshold
    Throughput,     // requests per second
    ErrorRate,      // % requests that return 5xx
}

// SLO Tracking Service
public class SloTracker
{
    private readonly IMetricsClient _metricsClient;
    private readonly ILogger<SloTracker> _logger;

    public SloTracker(IMetricsClient metricsClient, ILogger<SloTracker> logger)
    {
        _metricsClient = metricsClient;
        _logger = logger;
    }

    public async Task<SloStatus> GetStatusAsync(
        ServiceLevelObjective slo,
        CancellationToken ct = default)
    {
        var (goodEvents, totalEvents) = await _metricsClient.GetEventsAsync(
            slo.MetricType, slo.Window, ct);

        var currentRate = totalEvents > 0
            ? (double)goodEvents / totalEvents
            : 1.0;

        var errorBudget = CalculateErrorBudget(slo, currentRate);

        return new SloStatus
        {
            Slo = slo,
            CurrentRate = currentRate,
            IsMet = currentRate >= slo.Target,
            ErrorBudgetRemaining = errorBudget,
            ErrorBudgetBurnRate = CalculateBurnRate(errorBudget, slo.Window),
        };
    }

    private static ErrorBudget CalculateErrorBudget(
        ServiceLevelObjective slo,
        double currentRate)
    {
        var allowed = 1.0 - slo.Target;  // e.g., 0.001 for 99.9%
        var consumed = 1.0 - currentRate; // actual error rate
        var remaining = allowed - consumed;
        var remainingPercent = allowed > 0 ? remaining / allowed : 1.0;

        return new ErrorBudget
        {
            AllowedErrorRate = allowed,
            ConsumedErrorRate = consumed,
            RemainingPercent = Math.Max(0, remainingPercent),
        };
    }

    private static double CalculateBurnRate(ErrorBudget budget, TimeSpan window)
    {
        // Burn rate > 1 means budget will be exhausted before window ends
        if (budget.AllowedErrorRate <= 0) return 0;
        return budget.ConsumedErrorRate / budget.AllowedErrorRate;
    }
}

public class SloStatus
{
    public required ServiceLevelObjective Slo { get; init; }
    public double CurrentRate { get; init; }
    public bool IsMet { get; init; }
    public required ErrorBudget ErrorBudgetRemaining { get; init; }
    public double ErrorBudgetBurnRate { get; init; }
    public string StatusEmoji => IsMet ? "✅" : "🚨";
}

public class ErrorBudget
{
    public double AllowedErrorRate { get; init; }
    public double ConsumedErrorRate { get; init; }
    public double RemainingPercent { get; init; }
}
```

### Alert on Error Budget Burn Rate

```csharp
public class ErrorBudgetBurnRateAlert : BackgroundService
{
    private readonly SloTracker _sloTracker;
    private readonly IAlertingService _alerting;
    private readonly ILogger<ErrorBudgetBurnRateAlert> _logger;

    // Alert thresholds (Google SRE book recommendations)
    private static readonly (double BurnRate, string Severity, TimeSpan Lookback)[] Thresholds =
    [
        (14.4, "Critical", TimeSpan.FromHours(1)),   // 2% budget in 1 hour
        (6.0,  "High",     TimeSpan.FromHours(6)),   // 5% budget in 6 hours
        (1.0,  "Warning",  TimeSpan.FromDays(3)),    // consuming entire budget
    ];

    public ErrorBudgetBurnRateAlert(
        SloTracker sloTracker,
        IAlertingService alerting,
        ILogger<ErrorBudgetBurnRateAlert> logger)
    {
        _sloTracker = sloTracker;
        _alerting = alerting;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await CheckBurnRatesAsync(stoppingToken);
            await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken);
        }
    }

    private async Task CheckBurnRatesAsync(CancellationToken ct)
    {
        var slos = await _sloTracker.GetAllSlosAsync(ct);

        foreach (var sloStatus in slos)
        {
            foreach (var (burnRateThreshold, severity, _) in Thresholds)
            {
                if (sloStatus.ErrorBudgetBurnRate >= burnRateThreshold)
                {
                    await _alerting.SendAlertAsync(new Alert
                    {
                        Severity = severity,
                        Title = $"High error budget burn rate: {sloStatus.Slo.Name}",
                        Description =
                            $"Burn rate: {sloStatus.ErrorBudgetBurnRate:F1}x " +
                            $"(threshold: {burnRateThreshold}x). " +
                            $"Budget remaining: {sloStatus.ErrorBudgetRemaining.RemainingPercent:P1}",
                    }, ct);
                    break; // Only send most severe alert
                }
            }
        }
    }
}
```

---

## Step 1265: Chaos Engineering

### Chaos Testing with .NET

```csharp
// Chaos engineering middleware for testing resilience
public class ChaosMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ChaosOptions _options;
    private readonly ILogger<ChaosMiddleware> _logger;
    private readonly Random _random = new();

    public ChaosMiddleware(
        RequestDelegate next,
        IOptions<ChaosOptions> options,
        ILogger<ChaosMiddleware> logger)
    {
        _next = next;
        _options = options.Value;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        if (!_options.Enabled)
        {
            await _next(context);
            return;
        }

        // Random latency injection
        if (_options.LatencyInjectionEnabled &&
            _random.NextDouble() < _options.LatencyProbability)
        {
            var delay = _random.Next(
                _options.MinLatencyMs,
                _options.MaxLatencyMs);
            _logger.LogWarning("Chaos: Injecting {Delay}ms latency", delay);
            await Task.Delay(delay, context.RequestAborted);
        }

        // Random error injection
        if (_options.ErrorInjectionEnabled &&
            _random.NextDouble() < _options.ErrorProbability)
        {
            var statusCode = _options.ErrorStatusCodes[
                _random.Next(_options.ErrorStatusCodes.Length)];
            _logger.LogWarning("Chaos: Injecting HTTP {StatusCode} error", statusCode);
            context.Response.StatusCode = statusCode;
            await context.Response.WriteAsJsonAsync(new
            {
                error = "Chaos engineering: intentional error injection"
            });
            return;
        }

        await _next(context);
    }
}

public class ChaosOptions
{
    public bool Enabled { get; set; }
    public bool LatencyInjectionEnabled { get; set; }
    public double LatencyProbability { get; set; } = 0.1;
    public int MinLatencyMs { get; set; } = 100;
    public int MaxLatencyMs { get; set; } = 2000;
    public bool ErrorInjectionEnabled { get; set; }
    public double ErrorProbability { get; set; } = 0.05;
    public int[] ErrorStatusCodes { get; set; } = [500, 502, 503];
}
```

### Resilience Testing

```csharp
public class ResilienceTestRunner
{
    private readonly HttpClient _httpClient;
    private readonly ILogger<ResilienceTestRunner> _logger;

    public ResilienceTestRunner(IHttpClientFactory factory, ILogger<ResilienceTestRunner> logger)
    {
        _httpClient = factory.CreateClient();
        _logger = logger;
    }

    public async Task<ResilienceTestResult> RunTestAsync(
        string endpoint,
        int requestCount = 1000,
        int maxConcurrency = 10,
        CancellationToken ct = default)
    {
        var semaphore = new SemaphoreSlim(maxConcurrency);
        var results = new ConcurrentBag<RequestResult>();
        var sw = Stopwatch.StartNew();

        var tasks = Enumerable.Range(0, requestCount)
            .Select(async _ =>
            {
                await semaphore.WaitAsync(ct);
                try
                {
                    var requestSw = Stopwatch.StartNew();
                    try
                    {
                        var response = await _httpClient.GetAsync(endpoint, ct);
                        results.Add(new RequestResult
                        {
                            StatusCode = (int)response.StatusCode,
                            DurationMs = requestSw.ElapsedMilliseconds,
                            Succeeded = response.IsSuccessStatusCode,
                        });
                    }
                    catch (Exception ex)
                    {
                        results.Add(new RequestResult
                        {
                            StatusCode = 0,
                            DurationMs = requestSw.ElapsedMilliseconds,
                            Succeeded = false,
                            Error = ex.Message,
                        });
                    }
                }
                finally
                {
                    semaphore.Release();
                }
            });

        await Task.WhenAll(tasks);
        sw.Stop();

        var resultList = results.ToList();
        var successCount = resultList.Count(r => r.Succeeded);
        var durations = resultList.Select(r => r.DurationMs).OrderBy(d => d).ToArray();

        return new ResilienceTestResult
        {
            TotalRequests = requestCount,
            SuccessCount = successCount,
            FailureCount = requestCount - successCount,
            SuccessRate = (double)successCount / requestCount,
            TotalDurationMs = sw.ElapsedMilliseconds,
            Throughput = requestCount / sw.Elapsed.TotalSeconds,
            P50LatencyMs = durations[durations.Length / 2],
            P95LatencyMs = durations[(int)(durations.Length * 0.95)],
            P99LatencyMs = durations[(int)(durations.Length * 0.99)],
            MaxLatencyMs = durations[^1],
            ErrorsByStatus = resultList
                .Where(r => !r.Succeeded)
                .GroupBy(r => r.StatusCode)
                .ToDictionary(g => g.Key, g => g.Count()),
        };
    }
}

public record RequestResult
{
    public int StatusCode { get; init; }
    public long DurationMs { get; init; }
    public bool Succeeded { get; init; }
    public string? Error { get; init; }
}

public class ResilienceTestResult
{
    public int TotalRequests { get; init; }
    public int SuccessCount { get; init; }
    public int FailureCount { get; init; }
    public double SuccessRate { get; init; }
    public long TotalDurationMs { get; init; }
    public double Throughput { get; init; }
    public long P50LatencyMs { get; init; }
    public long P95LatencyMs { get; init; }
    public long P99LatencyMs { get; init; }
    public long MaxLatencyMs { get; init; }
    public required Dictionary<int, int> ErrorsByStatus { get; init; }
}
```

---

## Step 1266: Advanced Pipeline Patterns

### GitHub Actions - Matrix Build

```yaml
# .github/workflows/matrix-build.yml
name: Matrix Build

on:
  push:
    branches: [main, develop]

jobs:
  build:
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest]
        dotnet: ['8.0.x', '9.0.x']
        exclude:
          - os: windows-latest
            dotnet: '8.0.x'
    
    runs-on: ${{ matrix.os }}
    
    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET ${{ matrix.dotnet }}
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ matrix.dotnet }}

      - name: Restore
        run: dotnet restore

      - name: Build
        run: dotnet build -c Release --no-restore

      - name: Test
        run: dotnet test -c Release --no-build \
          --logger "github" \
          --collect:"XPlat Code Coverage"

  analyze:
    needs: build
    runs-on: ubuntu-latest
    permissions:
      security-events: write
    steps:
      - uses: actions/checkout@v4
      
      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: csharp

      - name: Build for CodeQL
        run: dotnet build

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3
```

### Reusable Workflow

```yaml
# .github/workflows/dotnet-build.yml (reusable)
name: .NET Build and Test

on:
  workflow_call:
    inputs:
      dotnet-version:
        type: string
        default: '9.0.x'
      run-integration-tests:
        type: boolean
        default: false
    secrets:
      CODECOV_TOKEN:
        required: true

jobs:
  build-test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 5s
          --health-timeout 3s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ inputs.dotnet-version }}

      - name: Cache NuGet packages
        uses: actions/cache@v4
        with:
          path: ~/.nuget/packages
          key: ${{ runner.os }}-nuget-${{ hashFiles('**/*.csproj') }}
          restore-keys: ${{ runner.os }}-nuget-

      - name: Restore
        run: dotnet restore

      - name: Build
        run: dotnet build -c Release --no-restore

      - name: Unit Tests
        run: |
          dotnet test tests/UnitTests -c Release --no-build \
            --logger "trx" \
            --collect:"XPlat Code Coverage"

      - name: Integration Tests
        if: inputs.run-integration-tests
        env:
          ConnectionStrings__Default: "Host=localhost;Database=testdb;Username=test;Password=test"
        run: |
          dotnet test tests/IntegrationTests -c Release --no-build \
            --logger "trx"

      - name: Upload Coverage
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
```

---

## Step 1267: Security Scanning in CI/CD

### SAST and Dependency Scanning

```yaml
# .github/workflows/security.yml
name: Security Scanning

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 6 * * 1'  # Monday at 6 AM UTC

jobs:
  dependency-review:
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    steps:
      - uses: actions/checkout@v4
      - uses: actions/dependency-review-action@v4
        with:
          fail-on-severity: high

  sast:
    runs-on: ubuntu-latest
    permissions:
      security-events: write
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Semgrep
        uses: semgrep/semgrep-action@v1
        with:
          config: >-
            p/csharp
            p/owasp-top-ten
            p/sql-injection

  container-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'myapp:${{ github.sha }}'
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'

      - name: Upload Trivy scan results
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: 'trivy-results.sarif'

  secrets-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full history for git-leaks

      - name: Run Gitleaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## Step 1268: Advanced Health Checks

### Comprehensive Health Check System

```csharp
using Microsoft.Extensions.Diagnostics.HealthChecks;

// Program.cs
builder.Services
    .AddHealthChecks()
    // Database
    .AddDbContextCheck<AppDbContext>("database", tags: ["db", "ready"])
    // Cache
    .AddRedis(
        redisConnectionString,
        name: "redis",
        tags: ["cache", "ready"])
    // External dependencies
    .AddUrlGroup(
        new Uri("https://api.stripe.com/v1"),
        name: "payment-api",
        tags: ["external"])
    // Custom: queue depth
    .AddCheck<QueueDepthHealthCheck>("queue-depth", tags: ["queue"])
    // Custom: disk space
    .AddCheck<DiskSpaceHealthCheck>("disk-space", tags: ["infra"])
    // Custom: memory
    .AddCheck<MemoryHealthCheck>("memory", tags: ["infra"]);

// Map health endpoints
app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse,
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready"),
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse,
});

app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false, // Just check if app is running
});

// Custom health checks
public class QueueDepthHealthCheck : IHealthCheck
{
    private readonly IMessageQueueMonitor _queueMonitor;

    public QueueDepthHealthCheck(IMessageQueueMonitor queueMonitor)
    {
        _queueMonitor = queueMonitor;
    }

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        var depth = await _queueMonitor.GetQueueDepthAsync(cancellationToken);

        return depth switch
        {
            < 1000 => HealthCheckResult.Healthy($"Queue depth: {depth}"),
            < 5000 => HealthCheckResult.Degraded(
                $"Queue depth high: {depth}",
                data: new Dictionary<string, object> { ["depth"] = depth }),
            _ => HealthCheckResult.Unhealthy(
                $"Queue depth critical: {depth}",
                data: new Dictionary<string, object> { ["depth"] = depth }),
        };
    }
}

public class MemoryHealthCheck : IHealthCheck
{
    private const long MaxAllowedMemoryBytes = 500 * 1024 * 1024; // 500 MB

    public Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        var allocated = GC.GetTotalMemory(forceFullCollection: false);
        var gcInfo = GC.GetGCMemoryInfo();

        var data = new Dictionary<string, object>
        {
            ["allocated_bytes"] = allocated,
            ["heap_size_bytes"] = gcInfo.HeapSizeBytes,
            ["fragmented_bytes"] = gcInfo.FragmentedBytes,
        };

        return Task.FromResult(allocated < MaxAllowedMemoryBytes
            ? HealthCheckResult.Healthy($"Memory: {allocated / 1024 / 1024}MB", data)
            : HealthCheckResult.Degraded($"High memory: {allocated / 1024 / 1024}MB", data: data));
    }
}
```

---

## Step 1269: Database DevOps

### EF Core Migrations in CI/CD

```csharp
// EF Core Migration script generation
// Program.cs - Migration runner
public class MigrationRunner
{
    public static async Task<int> RunAsync(string[] args)
    {
        if (args.Contains("--migrate"))
        {
            return await ApplyMigrationsAsync();
        }

        if (args.Contains("--generate-script"))
        {
            return await GenerateMigrationScriptAsync();
        }

        return 0;
    }

    private static async Task<int> ApplyMigrationsAsync()
    {
        var host = CreateMigrationHost();

        using var scope = host.Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();

        Console.WriteLine("Checking pending migrations...");
        var pending = await db.Database.GetPendingMigrationsAsync();

        if (!pending.Any())
        {
            Console.WriteLine("No pending migrations.");
            return 0;
        }

        Console.WriteLine($"Applying {pending.Count()} migration(s)...");
        foreach (var migration in pending)
            Console.WriteLine($"  - {migration}");

        await db.Database.MigrateAsync();
        Console.WriteLine("Migrations applied successfully.");
        return 0;
    }

    private static async Task<int> GenerateMigrationScriptAsync()
    {
        var host = CreateMigrationHost();

        using var scope = host.Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();

        var script = db.Database.GenerateMigrationScript(
            fromMigration: "0",
            toMigration: null,
            idempotent: true);

        await File.WriteAllTextAsync("migration.sql", script);
        Console.WriteLine("Migration script saved to migration.sql");
        return 0;
    }

    private static IHost CreateMigrationHost()
    {
        return Host.CreateDefaultBuilder()
            .ConfigureAppConfiguration(config =>
            {
                config.AddEnvironmentVariables();
            })
            .ConfigureServices((ctx, services) =>
            {
                services.AddDbContext<AppDbContext>(options =>
                    options.UseNpgsql(
                        ctx.Configuration.GetConnectionString("Default")));
            })
            .Build();
    }
}
```

### Database Migration GitHub Action

```yaml
# .github/workflows/db-migrate.yml
name: Database Migration

on:
  workflow_dispatch:
    inputs:
      environment:
        type: choice
        options: [staging, production]
        required: true
      action:
        type: choice
        options: [migrate, generate-script]
        required: true

jobs:
  migrate:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '9.0.x'

      - name: Build migration tool
        run: dotnet build src/MyApp.MigrationTool -c Release

      - name: Run migration
        env:
          ConnectionStrings__Default: ${{ secrets.DB_CONNECTION_STRING }}
        run: |
          dotnet run --project src/MyApp.MigrationTool \
            -c Release -- \
            --${{ inputs.action }}
```

---

## Step 1270: Advanced Logging and Alerting

### Structured Logging Patterns

```csharp
using Serilog;
using Serilog.Enrichers.Span;
using Serilog.Formatting.Compact;

// Program.cs - Production-grade Serilog setup
builder.Host.UseSerilog((context, services, config) =>
{
    config
        .ReadFrom.Configuration(context.Configuration)
        .ReadFrom.Services(services)
        .Enrich.FromLogContext()
        .Enrich.WithMachineName()
        .Enrich.WithEnvironmentName()
        .Enrich.WithSpan()  // OpenTelemetry trace correlation
        .Enrich.With<TenantEnricher>()
        .WriteTo.Console(new CompactJsonFormatter())
        .WriteTo.Seq(context.Configuration["Seq:ServerUrl"]!)
        .WriteTo.OpenTelemetry(o =>
        {
            o.Endpoint = context.Configuration["OTEL_EXPORTER_OTLP_ENDPOINT"]!;
        });

    if (context.HostingEnvironment.IsDevelopment())
    {
        config.WriteTo.Console();
    }
});

// Structured log messages
public class OrderService(ILogger<OrderService> logger)
{
    public async Task<Order> CreateOrderAsync(CreateOrderCommand cmd)
    {
        using var activity = AppMetrics.ActivitySource
            .StartActivity("CreateOrder");
        activity?.SetTag("customer.id", cmd.CustomerId.ToString());

        logger.LogInformation(
            "Creating order for customer {CustomerId} with {ItemCount} items",
            cmd.CustomerId,
            cmd.Items.Count);

        try
        {
            var order = new Order(/* ... */);

            logger.LogInformation(
                "Order {OrderId} created successfully. Total: {Amount} {Currency}",
                order.Id,
                order.TotalAmount.Amount,
                order.TotalAmount.Currency);

            return order;
        }
        catch (Exception ex)
        {
            logger.LogError(ex,
                "Failed to create order for customer {CustomerId}",
                cmd.CustomerId);
            throw;
        }
    }
}

// Custom enricher
public class TenantEnricher : ILogEventEnricher
{
    private readonly IHttpContextAccessor _httpContextAccessor;

    public TenantEnricher(IHttpContextAccessor httpContextAccessor)
    {
        _httpContextAccessor = httpContextAccessor;
    }

    public void Enrich(LogEvent logEvent, ILogEventPropertyFactory propertyFactory)
    {
        var tenantId = _httpContextAccessor.HttpContext?
            .User.FindFirst("tenant_id")?.Value;

        if (!string.IsNullOrEmpty(tenantId))
        {
            logEvent.AddPropertyIfAbsent(
                propertyFactory.CreateProperty("TenantId", tenantId));
        }
    }
}
```

### PagerDuty/OpsGenie Alert Integration

```csharp
public interface IAlertingService
{
    Task SendAlertAsync(Alert alert, CancellationToken ct = default);
    Task ResolveAlertAsync(string alertId, CancellationToken ct = default);
}

public class PagerDutyAlertingService : IAlertingService
{
    private readonly HttpClient _httpClient;
    private readonly string _routingKey;

    public PagerDutyAlertingService(
        IHttpClientFactory factory,
        IConfiguration config)
    {
        _httpClient = factory.CreateClient("pagerduty");
        _httpClient.BaseAddress = new Uri("https://events.pagerduty.com");
        _routingKey = config["PagerDuty:RoutingKey"]!;
    }

    public async Task SendAlertAsync(Alert alert, CancellationToken ct = default)
    {
        var payload = new
        {
            routing_key = _routingKey,
            event_action = "trigger",
            dedup_key = alert.DedupKey ?? Guid.NewGuid().ToString(),
            payload = new
            {
                summary = alert.Title,
                severity = MapSeverity(alert.Severity),
                source = alert.Source ?? "myapp-api",
                timestamp = DateTime.UtcNow.ToString("O"),
                custom_details = new
                {
                    description = alert.Description,
                    alert.Environment,
                    alert.Service,
                }
            }
        };

        var response = await _httpClient.PostAsJsonAsync(
            "/v2/enqueue", payload, ct);

        response.EnsureSuccessStatusCode();
    }

    public async Task ResolveAlertAsync(string alertId, CancellationToken ct = default)
    {
        var payload = new
        {
            routing_key = _routingKey,
            event_action = "resolve",
            dedup_key = alertId,
        };

        await _httpClient.PostAsJsonAsync("/v2/enqueue", payload, ct);
    }

    private static string MapSeverity(string severity) => severity switch
    {
        "Critical" => "critical",
        "High" => "error",
        "Medium" => "warning",
        _ => "info",
    };
}

public class Alert
{
    public required string Title { get; init; }
    public required string Description { get; init; }
    public required string Severity { get; init; }
    public string? DedupKey { get; init; }
    public string? Source { get; init; }
    public string? Environment { get; init; }
    public string? Service { get; init; }
}
```

---

## Step 1271: Canary Deployments with .NET

### Feature Flag-Based Canary

```csharp
// Canary deployment using header-based routing
public class CanaryMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IFeatureManager _featureManager;

    public CanaryMiddleware(RequestDelegate next, IFeatureManager featureManager)
    {
        _next = next;
        _featureManager = featureManager;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // Check if canary enabled for this user
        var userId = context.User.FindFirst("sub")?.Value;
        var isCanaryUser = userId is not null &&
            await _featureManager.IsEnabledForUserAsync("CanaryV2", userId);

        if (isCanaryUser)
        {
            context.Items["version"] = "v2";
            context.Response.Headers["X-App-Version"] = "v2-canary";
        }

        await _next(context);
    }
}

// Progressive rollout with percentage
public class ProgressiveRolloutFilter : IFeatureFilter
{
    public Task<bool> EvaluateAsync(FeatureFilterEvaluationContext context)
    {
        var percentage = context.Parameters.GetSection("Percentage")
            .Get<double>();

        // Use hash of user ID for stable assignment
        var userId = context.HttpContext?.User.FindFirst("sub")?.Value;
        if (userId is null) return Task.FromResult(false);

        var hash = Math.Abs(HashCode.Combine(userId, context.FeatureName));
        var bucket = (hash % 100) + 1;

        return Task.FromResult(bucket <= percentage);
    }

    public string Name => "ProgressiveRollout";
}
```

---

## Step 1272: Load Testing

### NBomber Load Testing

```csharp
// Install: dotnet add package NBomber

using NBomber.CSharp;
using NBomber.Http.CSharp;

public class ApiLoadTest
{
    public static void Run()
    {
        using var httpClient = new HttpClient();

        // Scenario 1: High-volume read
        var readScenario = Scenario.Create("read_orders", async context =>
        {
            var orderId = Guid.NewGuid();
            var response = await httpClient.GetAsync(
                $"http://localhost:8080/api/orders/{orderId}");

            return response.IsSuccessStatusCode
                ? Response.Ok()
                : Response.Fail();
        })
        .WithWarmUpDuration(TimeSpan.FromSeconds(10))
        .WithLoadSimulations(
            // Ramp up to 100 users over 30s
            Simulation.RampingInject(rate: 100, interval: TimeSpan.FromSeconds(1),
                during: TimeSpan.FromSeconds(30)),
            // Hold 100 users for 2 minutes
            Simulation.Inject(rate: 100, interval: TimeSpan.FromSeconds(1),
                during: TimeSpan.FromMinutes(2)),
            // Ramp down
            Simulation.RampingInject(rate: 0, interval: TimeSpan.FromSeconds(1),
                during: TimeSpan.FromSeconds(30)));

        // Scenario 2: Write-heavy
        var writeScenario = Scenario.Create("create_orders", async context =>
        {
            var order = new { CustomerId = Guid.NewGuid(), Items = new[] { new { ProductId = "P1", Qty = 1 } } };
            var response = await httpClient.PostAsJsonAsync(
                "http://localhost:8080/api/orders", order);

            return response.IsSuccessStatusCode
                ? Response.Ok()
                : Response.Fail();
        })
        .WithWarmUpDuration(TimeSpan.FromSeconds(10))
        .WithLoadSimulations(
            Simulation.Inject(rate: 20, interval: TimeSpan.FromSeconds(1),
                during: TimeSpan.FromMinutes(2)));

        // Run
        NBomberRunner
            .RegisterScenarios(readScenario, writeScenario)
            .WithReportFolder("load-test-results")
            .WithReportFormats(ReportFormat.Html, ReportFormat.Md)
            .Run();
    }
}
```

### k6 Load Test Script (from .NET)

```javascript
// k6-test.js - generate from .NET configuration
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

const errorRate = new Rate('errors');

export const options = {
  stages: [
    { duration: '30s', target: 20 },
    { duration: '1m', target: 100 },
    { duration: '2m', target: 100 },
    { duration: '30s', target: 0 },
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],  // 95% under 500ms
    errors: ['rate<0.01'],             // Error rate under 1%
  },
};

export default function () {
  const BASE_URL = 'http://localhost:8080/api';
  
  const response = http.get(`${BASE_URL}/orders`);
  
  const success = check(response, {
    'status is 200': (r) => r.status === 200,
    'response time OK': (r) => r.timings.duration < 500,
  });
  
  errorRate.add(!success);
  sleep(1);
}
```

---

## Step 1273: Distributed Tracing Deep Dive

### Custom Spans and Baggage

```csharp
using System.Diagnostics;

public class TraceEnrichedOrderService
{
    private static readonly ActivitySource ActivitySource =
        new ActivitySource("MyApp.Orders");

    private readonly IOrderRepository _repository;
    private readonly ILogger<TraceEnrichedOrderService> _logger;

    public TraceEnrichedOrderService(
        IOrderRepository repository,
        ILogger<TraceEnrichedOrderService> logger)
    {
        _repository = repository;
        _logger = logger;
    }

    public async Task<Order> CreateOrderAsync(CreateOrderCommand cmd)
    {
        using var activity = ActivitySource.StartActivity(
            "CreateOrder",
            ActivityKind.Server);

        // Add trace attributes
        activity?.SetTag("order.customer_id", cmd.CustomerId.ToString());
        activity?.SetTag("order.item_count", cmd.Items.Count);

        // Baggage propagates across service boundaries
        Activity.Current?.SetBaggage("customer.tier",
            await GetCustomerTierAsync(cmd.CustomerId));

        try
        {
            // Child span for validation
            using var validationSpan = ActivitySource.StartActivity("ValidateOrder");
            await ValidateOrderAsync(cmd);
            validationSpan?.SetStatus(ActivityStatusCode.Ok);

            // Child span for persistence
            using var persistSpan = ActivitySource.StartActivity("PersistOrder");
            var order = Order.Create(cmd);
            await _repository.SaveAsync(order);
            persistSpan?.SetTag("order.id", order.Id.ToString());
            persistSpan?.SetStatus(ActivityStatusCode.Ok);

            activity?.SetTag("order.id", order.Id.ToString());
            activity?.SetStatus(ActivityStatusCode.Ok);

            return order;
        }
        catch (Exception ex)
        {
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            activity?.RecordException(ex);
            throw;
        }
    }

    private Task ValidateOrderAsync(CreateOrderCommand cmd) => Task.CompletedTask;

    private Task<string> GetCustomerTierAsync(CustomerId customerId) =>
        Task.FromResult("standard");
}
```

---

## Step 1274: Platform Engineering - Backstage

### Backstage API Catalog Descriptor

```yaml
# catalog-info.yaml (in each .NET service repo)
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: order-service
  description: Order management microservice
  tags:
    - dotnet
    - csharp
    - microservice
  annotations:
    github.com/project-slug: myorg/order-service
    backstage.io/techdocs-ref: dir:.
    sonarqube.org/project-key: myorg_order-service
spec:
  type: service
  lifecycle: production
  owner: team-payments
  system: ecommerce-platform
  dependsOn:
    - component:product-catalog
    - component:customer-service
    - resource:orders-postgres
    - resource:orders-redis
  providesApis:
    - order-api
  consumesApis:
    - product-catalog-api
    - payment-api

---
apiVersion: backstage.io/v1alpha1
kind: API
metadata:
  name: order-api
  description: Order management REST API
spec:
  type: openapi
  lifecycle: production
  owner: team-payments
  definition:
    $text: ./openapi.yaml
```

### OpenAPI Spec Generation from .NET

```csharp
// Scalar/Swashbuckle for OpenAPI
builder.Services.AddOpenApi(options =>
{
    options.AddDocumentTransformer<SecuritySchemeTransformer>();
    options.AddOperationTransformer<TagOperationTransformer>();
});

// Generate and save spec to file
app.MapGet("/api-spec", async (IDocumentProvider documentProvider) =>
{
    var document = await documentProvider.GetDocumentAsync();
    return Results.Text(document, "application/json");
}).ExcludeFromDescription();

// Security scheme transformer
public class SecuritySchemeTransformer : IOpenApiDocumentTransformer
{
    public Task TransformAsync(
        OpenApiDocument document,
        OpenApiDocumentTransformerContext context,
        CancellationToken ct)
    {
        document.Components ??= new OpenApiComponents();
        document.Components.SecuritySchemes ??= new Dictionary<string, OpenApiSecurityScheme>();

        document.Components.SecuritySchemes["Bearer"] = new OpenApiSecurityScheme
        {
            Type = SecuritySchemeType.Http,
            Scheme = "bearer",
            BearerFormat = "JWT",
            Description = "Enter JWT token",
        };

        // Apply to all operations
        foreach (var path in document.Paths.Values)
        {
            foreach (var operation in path.Operations.Values)
            {
                operation.Security.Add(new OpenApiSecurityRequirement
                {
                    [new OpenApiSecurityScheme
                    {
                        Reference = new OpenApiReference
                        {
                            Type = ReferenceType.SecurityScheme,
                            Id = "Bearer"
                        }
                    }] = []
                });
            }
        }

        return Task.CompletedTask;
    }
}
```

---

## Step 1275: Event-Driven Architecture with Kafka

### Confluent Kafka .NET Producer

```csharp
using Confluent.Kafka;
using System.Text.Json;

public class KafkaOrderProducer : IAsyncDisposable
{
    private readonly IProducer<string, string> _producer;
    private readonly ILogger<KafkaOrderProducer> _logger;

    public KafkaOrderProducer(IConfiguration config, ILogger<KafkaOrderProducer> logger)
    {
        _logger = logger;

        var producerConfig = new ProducerConfig
        {
            BootstrapServers = config["Kafka:BootstrapServers"],
            // Idempotent producer: exactly-once semantics
            EnableIdempotence = true,
            Acks = Acks.All,
            MessageSendMaxRetries = 3,
            RetryBackoffMs = 1000,
            // Performance
            BatchSize = 32768,
            LingerMs = 5,
            CompressionType = CompressionType.Snappy,
        };

        _producer = new ProducerBuilder<string, string>(producerConfig)
            .SetErrorHandler((_, error) =>
                _logger.LogError("Kafka error: {Error}", error))
            .Build();
    }

    public async Task PublishOrderCreatedAsync(OrderCreatedEvent orderEvent)
    {
        var topic = "orders.created";
        var key = orderEvent.OrderId.ToString();
        var value = JsonSerializer.Serialize(orderEvent);

        var deliveryResult = await _producer.ProduceAsync(
            topic,
            new Message<string, string>
            {
                Key = key,
                Value = value,
                Headers = [
                    new Header("event-type", "OrderCreated"u8.ToArray()),
                    new Header("correlation-id",
                        System.Text.Encoding.UTF8.GetBytes(
                            Activity.Current?.TraceId.ToString() ?? Guid.NewGuid().ToString())),
                ]
            });

        _logger.LogInformation(
            "Published OrderCreated {OrderId} to {Topic}[{Partition}]@{Offset}",
            orderEvent.OrderId,
            topic,
            deliveryResult.Partition.Value,
            deliveryResult.Offset.Value);
    }

    public async ValueTask DisposeAsync()
    {
        _producer.Flush(TimeSpan.FromSeconds(10));
        _producer.Dispose();
        await Task.CompletedTask;
    }
}
```

### Kafka Consumer (BackgroundService)

```csharp
public class KafkaOrderConsumer : BackgroundService
{
    private readonly IServiceProvider _serviceProvider;
    private readonly IConfiguration _config;
    private readonly ILogger<KafkaOrderConsumer> _logger;

    public KafkaOrderConsumer(
        IServiceProvider serviceProvider,
        IConfiguration config,
        ILogger<KafkaOrderConsumer> logger)
    {
        _serviceProvider = serviceProvider;
        _config = config;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        var consumerConfig = new ConsumerConfig
        {
            BootstrapServers = _config["Kafka:BootstrapServers"],
            GroupId = "order-processor",
            AutoOffsetReset = AutoOffsetReset.Earliest,
            EnableAutoCommit = false,  // Manual commit for at-least-once
            MaxPollIntervalMs = 300_000,
            SessionTimeoutMs = 30_000,
        };

        using var consumer = new ConsumerBuilder<string, string>(consumerConfig)
            .SetErrorHandler((_, e) => _logger.LogError("Kafka error: {Error}", e))
            .SetPartitionsAssignedHandler((c, partitions) =>
                _logger.LogInformation("Assigned partitions: {Partitions}",
                    string.Join(",", partitions)))
            .SetPartitionsRevokedHandler((c, partitions) =>
                _logger.LogInformation("Revoked partitions: {Partitions}",
                    string.Join(",", partitions)))
            .Build();

        consumer.Subscribe(["orders.created", "orders.cancelled"]);

        try
        {
            while (!stoppingToken.IsCancellationRequested)
            {
                ConsumeResult<string, string>? result = null;
                try
                {
                    result = consumer.Consume(TimeSpan.FromMilliseconds(100));
                    if (result is null) continue;

                    await ProcessMessageAsync(result, stoppingToken);
                    consumer.Commit(result);
                }
                catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested)
                {
                    break;
                }
                catch (Exception ex)
                {
                    _logger.LogError(ex,
                        "Error processing Kafka message at {Topic}[{Partition}]@{Offset}",
                        result?.Topic, result?.Partition, result?.Offset);

                    // Dead letter queue
                    if (result is not null)
                        await SendToDeadLetterAsync(result, ex.Message);
                }
            }
        }
        finally
        {
            consumer.Close();
        }
    }

    private async Task ProcessMessageAsync(
        ConsumeResult<string, string> result,
        CancellationToken ct)
    {
        var eventType = result.Message.Headers
            .FirstOrDefault(h => h.Key == "event-type")?
            .GetValueBytes();

        var eventTypeName = eventType is not null
            ? System.Text.Encoding.UTF8.GetString(eventType)
            : "Unknown";

        using var scope = _serviceProvider.CreateScope();

        switch (eventTypeName)
        {
            case "OrderCreated":
                var orderEvent = JsonSerializer.Deserialize<OrderCreatedEvent>(result.Message.Value)!;
                var handler = scope.ServiceProvider.GetRequiredService<IOrderCreatedHandler>();
                await handler.HandleAsync(orderEvent, ct);
                break;

            default:
                _logger.LogWarning("Unknown event type: {EventType}", eventTypeName);
                break;
        }
    }

    private Task SendToDeadLetterAsync(
        ConsumeResult<string, string> result, string error) =>
        Task.CompletedTask; // Implement dead letter queue logic
}
```

---

## Step 1276: Advanced Caching Strategies

### Multi-Level Cache

```csharp
public class MultiLevelCache<TKey, TValue> where TKey : notnull
{
    private readonly IMemoryCache _l1;           // In-process
    private readonly IDistributedCache _l2;      // Redis
    private readonly TimeSpan _l1Duration;
    private readonly TimeSpan _l2Duration;

    public MultiLevelCache(
        IMemoryCache l1Cache,
        IDistributedCache l2Cache,
        TimeSpan l1Duration,
        TimeSpan l2Duration)
    {
        _l1 = l1Cache;
        _l2 = l2Cache;
        _l1Duration = l1Duration;
        _l2Duration = l2Duration;
    }

    public async Task<TValue?> GetOrSetAsync(
        TKey key,
        Func<CancellationToken, Task<TValue?>> factory,
        CancellationToken ct = default)
    {
        var cacheKey = key.ToString()!;

        // L1: Memory cache
        if (_l1.TryGetValue(cacheKey, out TValue? l1Value))
            return l1Value;

        // L2: Distributed cache
        var l2Bytes = await _l2.GetAsync(cacheKey, ct);
        if (l2Bytes is not null)
        {
            var l2Value = JsonSerializer.Deserialize<TValue>(l2Bytes);
            _l1.Set(cacheKey, l2Value, _l1Duration);
            return l2Value;
        }

        // Cache miss: fetch from source
        var value = await factory(ct);
        if (value is not null)
        {
            _l1.Set(cacheKey, value, _l1Duration);
            await _l2.SetAsync(
                cacheKey,
                JsonSerializer.SerializeToUtf8Bytes(value),
                new DistributedCacheEntryOptions
                {
                    AbsoluteExpirationRelativeToNow = _l2Duration,
                },
                ct);
        }

        return value;
    }

    public async Task InvalidateAsync(TKey key, CancellationToken ct = default)
    {
        var cacheKey = key.ToString()!;
        _l1.Remove(cacheKey);
        await _l2.RemoveAsync(cacheKey, ct);
    }
}

// Cache invalidation via events
public class ProductCacheInvalidator
    : INotificationHandler<ProductUpdatedEvent>
{
    private readonly MultiLevelCache<Guid, Product> _cache;

    public ProductCacheInvalidator(MultiLevelCache<Guid, Product> cache)
    {
        _cache = cache;
    }

    public async Task Handle(
        ProductUpdatedEvent notification,
        CancellationToken ct)
    {
        await _cache.InvalidateAsync(notification.ProductId, ct);
    }
}
```

---

## Step 1277: Rate Limiting Strategies

### Advanced Rate Limiting

```csharp
using Microsoft.AspNetCore.RateLimiting;
using System.Threading.RateLimiting;

// Program.cs
builder.Services.AddRateLimiter(options =>
{
    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;

    // Global: Fixed window
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(ctx =>
    {
        var apiKey = ctx.Request.Headers["X-Api-Key"].ToString();
        if (!string.IsNullOrEmpty(apiKey))
        {
            // API key users get higher limits
            return RateLimitPartition.GetFixedWindowLimiter(
                partitionKey: $"api:{apiKey}",
                factory: _ => new FixedWindowRateLimiterOptions
                {
                    Window = TimeSpan.FromMinutes(1),
                    PermitLimit = 1000,
                    QueueLimit = 0,
                });
        }

        // Anonymous users: lower limits by IP
        var ip = ctx.Connection.RemoteIpAddress?.ToString() ?? "unknown";
        return RateLimitPartition.GetSlidingWindowLimiter(
            partitionKey: $"ip:{ip}",
            factory: _ => new SlidingWindowRateLimiterOptions
            {
                Window = TimeSpan.FromMinutes(1),
                PermitLimit = 60,
                SegmentsPerWindow = 6,
                QueueLimit = 0,
            });
    });

    // Named policy: AI endpoints (expensive)
    options.AddPolicy("ai-api", ctx =>
    {
        var userId = ctx.User.FindFirst("sub")?.Value ?? "anonymous";
        return RateLimitPartition.GetTokenBucketLimiter(
            partitionKey: $"ai:{userId}",
            factory: _ => new TokenBucketRateLimiterOptions
            {
                TokenLimit = 10,
                ReplenishmentPeriod = TimeSpan.FromMinutes(1),
                TokensPerPeriod = 10,
                AutoReplenishment = true,
                QueueLimit = 0,
            });
    });

    options.OnRejected = async (ctx, ct) =>
    {
        ctx.HttpContext.Response.Headers.RetryAfter = "60";
        await ctx.HttpContext.Response.WriteAsJsonAsync(new
        {
            error = "Rate limit exceeded",
            retryAfter = 60
        }, ct);
    };
});

// Apply to AI endpoints
app.MapPost("/api/chat", ChatHandler)
    .RequireRateLimiting("ai-api");
```

---

## Step 1278: Distributed Caching with Redis

### Redis Patterns

```csharp
using StackExchange.Redis;

public class RedisService
{
    private readonly IConnectionMultiplexer _redis;
    private readonly IDatabase _db;

    public RedisService(IConnectionMultiplexer redis)
    {
        _redis = redis;
        _db = redis.GetDatabase();
    }

    // Distributed lock (Redlock algorithm)
    public async Task<IDisposable?> AcquireLockAsync(
        string resource,
        TimeSpan expiry,
        CancellationToken ct = default)
    {
        var lockKey = $"lock:{resource}";
        var lockValue = Guid.NewGuid().ToString();

        var acquired = await _db.StringSetAsync(
            lockKey, lockValue, expiry, When.NotExists);

        if (!acquired) return null;

        return new RedisLock(_db, lockKey, lockValue);
    }

    // Atomic increment with expiry
    public async Task<long> IncrementWithExpiryAsync(
        string key,
        TimeSpan expiry)
    {
        var tran = _db.CreateTransaction();

        var incrementTask = tran.StringIncrementAsync(key);
        _ = tran.KeyExpireAsync(key, expiry, CommandFlags.None);

        await tran.ExecuteAsync();
        return await incrementTask;
    }

    // Pub/Sub
    public async Task PublishAsync<T>(string channel, T message)
    {
        var subscriber = _redis.GetSubscriber();
        var json = JsonSerializer.Serialize(message);
        await subscriber.PublishAsync(
            RedisChannel.Literal(channel), json);
    }

    public async Task SubscribeAsync<T>(
        string channel,
        Func<T, Task> handler)
    {
        var subscriber = _redis.GetSubscriber();
        await subscriber.SubscribeAsync(
            RedisChannel.Literal(channel),
            async (_, value) =>
            {
                if (value.HasValue)
                {
                    var message = JsonSerializer.Deserialize<T>(value!)!;
                    await handler(message);
                }
            });
    }

    // Leaderboard with sorted sets
    public async Task UpdateScoreAsync(string leaderboard, string userId, double score)
    {
        await _db.SortedSetAddAsync(leaderboard, userId, score);
        await _db.KeyExpireAsync(leaderboard, TimeSpan.FromDays(30));
    }

    public async Task<IReadOnlyList<(string UserId, double Score)>> GetTopAsync(
        string leaderboard,
        int count = 10)
    {
        var entries = await _db.SortedSetRangeByRankWithScoresAsync(
            leaderboard, 0, count - 1, Order.Descending);

        return entries
            .Select(e => (e.Element.ToString(), e.Score))
            .ToList();
    }
}

public class RedisLock : IDisposable
{
    private readonly IDatabase _db;
    private readonly string _key;
    private readonly string _value;

    public RedisLock(IDatabase db, string key, string value)
    {
        _db = db;
        _key = key;
        _value = value;
    }

    public void Dispose()
    {
        // Release lock only if we still own it (Lua script for atomicity)
        var script = @"
            if redis.call('get', KEYS[1]) == ARGV[1] then
                return redis.call('del', KEYS[1])
            else
                return 0
            end";

        _db.ScriptEvaluate(script,
            new RedisKey[] { _key },
            new RedisValue[] { _value });
    }
}
```

---

## Step 1279: Production Incident Management

### Runbook as Code

```csharp
public class IncidentRunbook
{
    private readonly ILogger<IncidentRunbook> _logger;
    private readonly IDiagnosticsService _diagnostics;
    private readonly IAlertingService _alerting;

    public IncidentRunbook(
        ILogger<IncidentRunbook> logger,
        IDiagnosticsService diagnostics,
        IAlertingService alerting)
    {
        _logger = logger;
        _diagnostics = diagnostics;
        _alerting = alerting;
    }

    // Automated incident response for high error rate
    public async Task HandleHighErrorRateAsync(
        string service,
        double errorRate,
        CancellationToken ct = default)
    {
        _logger.LogWarning(
            "Starting incident runbook for high error rate: {Service} {ErrorRate:P2}",
            service, errorRate);

        var incidentId = Guid.NewGuid().ToString()[..8];
        var steps = new List<RunbookStepResult>();

        // Step 1: Check recent deployments
        var deployments = await _diagnostics.GetRecentDeploymentsAsync(
            service, TimeSpan.FromHours(1), ct);

        steps.Add(new RunbookStepResult(
            "Check recent deployments",
            deployments.Any()
                ? $"Found {deployments.Count} deployment(s) in last hour: " +
                  string.Join(", ", deployments.Select(d => d.Version))
                : "No recent deployments",
            deployments.Any() ? RunbookStatus.Warning : RunbookStatus.Ok));

        // Step 2: Check database health
        var dbHealth = await _diagnostics.CheckDatabaseHealthAsync(ct);
        steps.Add(new RunbookStepResult(
            "Database health",
            dbHealth.IsHealthy ? "Healthy" : $"Unhealthy: {dbHealth.Error}",
            dbHealth.IsHealthy ? RunbookStatus.Ok : RunbookStatus.Critical));

        // Step 3: Check memory/CPU
        var resources = await _diagnostics.GetResourceUsageAsync(ct);
        steps.Add(new RunbookStepResult(
            "Resource usage",
            $"CPU: {resources.CpuPercent:F1}%, Memory: {resources.MemoryMb}MB",
            resources.CpuPercent > 90 ? RunbookStatus.Critical : RunbookStatus.Ok));

        // Step 4: Check external dependencies
        var deps = await _diagnostics.CheckDependenciesAsync(service, ct);
        foreach (var dep in deps)
        {
            steps.Add(new RunbookStepResult(
                $"Dependency: {dep.Name}",
                dep.IsAvailable ? "Available" : $"Unavailable: {dep.Error}",
                dep.IsAvailable ? RunbookStatus.Ok : RunbookStatus.Critical));
        }

        // Generate incident summary
        var summary = GenerateIncidentSummary(incidentId, service, errorRate, steps);
        await _alerting.SendAlertAsync(summary, ct);

        _logger.LogInformation("Runbook completed for incident {IncidentId}", incidentId);
    }

    private Alert GenerateIncidentSummary(
        string incidentId,
        string service,
        double errorRate,
        List<RunbookStepResult> steps)
    {
        var criticalSteps = steps.Where(s => s.Status == RunbookStatus.Critical).ToList();

        return new Alert
        {
            Title = $"[{incidentId}] High error rate in {service}: {errorRate:P2}",
            Description = $"Automated diagnostics completed. " +
                $"{criticalSteps.Count} critical finding(s):\n" +
                string.Join("\n", criticalSteps.Select(s => $"❌ {s.Name}: {s.Message}")),
            Severity = criticalSteps.Any() ? "Critical" : "High",
            Service = service,
        };
    }
}

public record RunbookStepResult(string Name, string Message, RunbookStatus Status);
public enum RunbookStatus { Ok, Warning, Critical }
```

---

## Step 1280: Summary - DevOps & Platform Engineering

### ภาพรวมที่เรียนในบทนี้

```
DevOps & Platform Engineering
│
├── Infrastructure as Code
│   ├── Pulumi (C# native IaC)
│   └── Terraform modules
│
├── SRE Practices
│   ├── SLO/SLA/SLI definitions
│   ├── Error budget tracking
│   └── Burn rate alerting
│
├── CI/CD Advanced
│   ├── Matrix builds
│   ├── Reusable workflows
│   ├── Security scanning (SAST, Trivy, Gitleaks)
│   └── Database migrations in pipelines
│
├── Observability
│   ├── Serilog structured logging
│   ├── Custom distributed spans
│   └── PagerDuty alerting
│
├── Resilience Testing
│   ├── Chaos middleware
│   ├── NBomber load testing
│   └── k6 performance tests
│
├── Kafka Event Streaming
│   ├── Idempotent producer
│   ├── Consumer group management
│   └── Dead letter queue
│
├── Caching
│   ├── Multi-level cache (L1/L2)
│   ├── Redis patterns (pub/sub, sorted sets)
│   └── Distributed lock (Redlock)
│
├── Rate Limiting
│   ├── Sliding/Fixed/Token bucket
│   └── Per-user partitioning
│
└── Platform Tools
    ├── Backstage developer portal
    ├── Feature flags (progressive rollout)
    ├── Canary deployments
    └── Incident runbooks as code
```

### Key Practices Summary

| Practice | Tool | Benefit |
|----------|------|---------|
| IaC | Pulumi (.NET) | Type-safe, testable infra |
| SLO tracking | Custom + Prometheus | Data-driven reliability |
| Chaos testing | Custom middleware | Find weaknesses early |
| Load testing | NBomber | Performance validation |
| Security scanning | Trivy + Semgrep | Shift security left |
| Feature flags | Azure App Config | Safe deployments |
| Multi-level cache | Redis + IMemoryCache | Fast and consistent |
| Distributed tracing | OpenTelemetry | Debug across services |

ในบทถัดไปจะเรียนเรื่อง **Advanced Security & Zero Trust (Steps 1281-1320)** ซึ่งครอบคลุม OAuth 2.0/OIDC, Zero Trust Architecture, API security, encryption best practices และ security testing!
