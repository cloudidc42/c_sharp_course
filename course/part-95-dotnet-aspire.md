# Part 95: .NET Aspire - Cloud-Native Development Orchestration

## Steps 2149-2164

.NET Aspire คือ framework สำหรับสร้าง cloud-native distributed applications ที่รวม service orchestration, observability, และ service discovery มาให้พร้อมใช้งาน

---

## Step 2149: .NET Aspire Overview และ Architecture

```
.NET Aspire Stack
================

┌─────────────────────────────────────────────┐
│              Aspire App Host                │
│  (Orchestration & Service Discovery)        │
├─────────────────────────────────────────────┤
│         Service Defaults Project            │
│  (OpenTelemetry, Health Checks, Resilience) │
├──────────────┬──────────────────────────────┤
│  API Service │  Worker Service  │  Frontend │
│   (.NET 9)   │    (.NET 9)      │  (Blazor) │
├──────────────┴──────────────────────────────┤
│         Infrastructure Components           │
│  PostgreSQL │ Redis │ RabbitMQ │ Azure      │
└─────────────────────────────────────────────┘

Key Benefits:
- Local development orchestration
- Automatic service discovery via environment variables
- Built-in OpenTelemetry integration
- Health checks dashboard
- Testable distributed systems
```

---

## Step 2150: สร้าง Aspire Solution

```bash
# Install Aspire workload
dotnet workload install aspire

# Create new Aspire solution
dotnet new aspire -n ECommerceAspire
cd ECommerceAspire

# Structure created:
# ECommerceAspire.sln
# ECommerceAspire.AppHost/     (orchestration)
# ECommerceAspire.ServiceDefaults/  (shared config)
# ECommerceAspire.ApiService/  (sample API)
# ECommerceAspire.Web/         (Blazor frontend)

# Add more services
dotnet new webapi -n ECommerceAspire.OrderService
dotnet new worker -n ECommerceAspire.OrderProcessor
dotnet sln add ECommerceAspire.OrderService
dotnet sln add ECommerceAspire.OrderProcessor
```

**AppHost project file:**

```xml
<!-- ECommerceAspire.AppHost.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net9.0</TargetFramework>
    <IsAspireHost>true</IsAspireHost>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Aspire.Hosting.AppHost" Version="9.0.0" />
    <PackageReference Include="Aspire.Hosting.PostgreSQL" Version="9.0.0" />
    <PackageReference Include="Aspire.Hosting.Redis" Version="9.0.0" />
    <PackageReference Include="Aspire.Hosting.RabbitMQ" Version="9.0.0" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\ECommerceAspire.ApiService\ECommerceAspire.ApiService.csproj" />
    <ProjectReference Include="..\ECommerceAspire.OrderService\ECommerceAspire.OrderService.csproj" />
    <ProjectReference Include="..\ECommerceAspire.OrderProcessor\ECommerceAspire.OrderProcessor.csproj" />
    <ProjectReference Include="..\ECommerceAspire.Web\ECommerceAspire.Web.csproj" />
  </ItemGroup>
</Project>
```

---

## Step 2151: AppHost - Orchestration Configuration

```csharp
// ECommerceAspire.AppHost/Program.cs
var builder = DistributedApplication.CreateBuilder(args);

// Infrastructure resources
var postgres = builder.AddPostgres("postgres")
    .WithDataVolume("postgres-data")
    .WithPgAdmin();  // adds pgAdmin UI

var orderDb = postgres.AddDatabase("orderdb");
var catalogDb = postgres.AddDatabase("catalogdb");

var redis = builder.AddRedis("redis")
    .WithRedisInsight();  // adds RedisInsight UI

var rabbitMq = builder.AddRabbitMQ("messaging")
    .WithManagementPlugin();  // adds RabbitMQ management UI

// Application services
var catalogApi = builder.AddProject<Projects.ECommerceAspire_ApiService>("catalog-api")
    .WithReference(catalogDb)
    .WithReference(redis)
    .WithHttpHealthCheck("/health")
    .WithReplicas(2);

var orderService = builder.AddProject<Projects.ECommerceAspire_OrderService>("order-service")
    .WithReference(orderDb)
    .WithReference(rabbitMq)
    .WithReference(catalogApi)  // injects connection string as env var
    .WithHttpHealthCheck("/health");

var orderProcessor = builder.AddProject<Projects.ECommerceAspire_OrderProcessor>("order-processor")
    .WithReference(orderDb)
    .WithReference(rabbitMq);

var webFrontend = builder.AddProject<Projects.ECommerceAspire_Web>("web-frontend")
    .WithReference(catalogApi)
    .WithReference(orderService)
    .WithExternalHttpEndpoints();

builder.Build().Run();
```

---

## Step 2152: Service Defaults - Shared Configuration

```csharp
// ECommerceAspire.ServiceDefaults/Extensions.cs
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Diagnostics.HealthChecks;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Diagnostics.HealthChecks;
using Microsoft.Extensions.Logging;
using OpenTelemetry;
using OpenTelemetry.Metrics;
using OpenTelemetry.Trace;

namespace Microsoft.Extensions.Hosting;

public static class Extensions
{
    public static IHostApplicationBuilder AddServiceDefaults(this IHostApplicationBuilder builder)
    {
        builder.ConfigureOpenTelemetry();
        builder.AddDefaultHealthChecks();
        builder.Services.AddServiceDiscovery();
        builder.Services.ConfigureHttpClientDefaults(http =>
        {
            http.AddStandardResilienceHandler();
            http.AddServiceDiscovery();
        });
        return builder;
    }

    public static IHostApplicationBuilder ConfigureOpenTelemetry(this IHostApplicationBuilder builder)
    {
        builder.Logging.AddOpenTelemetry(logging =>
        {
            logging.IncludeFormattedMessage = true;
            logging.IncludeScopes = true;
        });

        builder.Services.AddOpenTelemetry()
            .WithMetrics(metrics =>
            {
                metrics.AddAspNetCoreInstrumentation()
                       .AddHttpClientInstrumentation()
                       .AddRuntimeInstrumentation();
            })
            .WithTracing(tracing =>
            {
                tracing.AddAspNetCoreInstrumentation()
                       .AddHttpClientInstrumentation()
                       .AddEntityFrameworkCoreInstrumentation()
                       .AddSource("ECommerce.*");
            });

        builder.AddOpenTelemetryExporters();
        return builder;
    }

    private static IHostApplicationBuilder AddOpenTelemetryExporters(this IHostApplicationBuilder builder)
    {
        var useOtlpExporter = !string.IsNullOrWhiteSpace(
            builder.Configuration["OTEL_EXPORTER_OTLP_ENDPOINT"]);

        if (useOtlpExporter)
        {
            builder.Services.AddOpenTelemetry().UseOtlpExporter();
        }

        return builder;
    }

    public static IHostApplicationBuilder AddDefaultHealthChecks(this IHostApplicationBuilder builder)
    {
        builder.Services.AddHealthChecks()
            .AddCheck("self", () => HealthCheckResult.Healthy(), ["live"]);
        return builder;
    }

    public static WebApplication MapDefaultEndpoints(this WebApplication app)
    {
        if (app.Environment.IsDevelopment())
        {
            app.MapHealthChecks("/health");
            app.MapHealthChecks("/alive", new HealthCheckOptions
            {
                Predicate = r => r.Tags.Contains("live")
            });
        }
        return app;
    }
}
```

---

## Step 2153: Service Discovery ใน Client Services

```csharp
// ECommerceAspire.OrderService/Program.cs
var builder = WebApplication.CreateBuilder(args);

// Add Aspire service defaults (OpenTelemetry, health checks, service discovery)
builder.AddServiceDefaults();

// Register HttpClient with service discovery
// "catalog-api" maps to the service name in AppHost
builder.Services.AddHttpClient<ICatalogClient, CatalogClient>(
    client => client.BaseAddress = new Uri("https+http://catalog-api"));

// Add database from Aspire connection string
builder.AddNpgsqlDbContext<OrderDbContext>("orderdb");

// Add Redis from Aspire connection string
builder.AddRedisDistributedCache("redis");

var app = builder.Build();
app.MapDefaultEndpoints();
app.MapOrderEndpoints();
app.Run();

// CatalogClient.cs
public class CatalogClient(HttpClient httpClient) : ICatalogClient
{
    public async Task<ProductDto?> GetProductAsync(Guid productId, CancellationToken ct = default)
    {
        return await httpClient.GetFromJsonAsync<ProductDto>(
            $"/api/products/{productId}", ct);
    }

    public async Task<bool> CheckStockAsync(Guid productId, int quantity, CancellationToken ct = default)
    {
        var response = await httpClient.PostAsJsonAsync(
            "/api/inventory/check", new { ProductId = productId, Quantity = quantity }, ct);
        return response.IsSuccessStatusCode;
    }
}
```

---

## Step 2154: Aspire Components - Database Integration

```csharp
// Add Aspire PostgreSQL component
// ECommerceAspire.OrderService/Program.cs

// Package: Aspire.Npgsql.EntityFrameworkCore.PostgreSQL
builder.AddNpgsqlDbContext<OrderDbContext>(
    connectionName: "orderdb",
    configureDbContextOptions: options =>
    {
        options.UseNpgsql(npgsql => npgsql
            .MigrationsAssembly(typeof(OrderDbContext).Assembly.FullName)
            .EnableRetryOnFailure(5));
    });

// This automatically:
// 1. Reads connection string from ASPIRE_CONNECTIONSTRING_orderdb env var
// 2. Configures health checks
// 3. Sets up OpenTelemetry tracing for EF Core
// 4. Handles retry logic

// appsettings.Development.json (for local without Aspire)
{
  "ConnectionStrings": {
    "orderdb": "Host=localhost;Database=orderdb;Username=postgres;Password=postgres"
  }
}
```

**Redis component:**

```csharp
// Package: Aspire.StackExchange.Redis
builder.AddRedisClient("redis");

// Or for IDistributedCache:
// Package: Aspire.StackExchange.Redis.DistributedCaching
builder.AddRedisDistributedCache("redis");

// Or for OutputCache:
// Package: Aspire.StackExchange.Redis.OutputCaching
builder.AddRedisOutputCache("redis");
```

**RabbitMQ component:**

```csharp
// Package: Aspire.RabbitMQ.Client
builder.AddRabbitMQClient("messaging");

// Or with MassTransit:
// Package: Aspire.MassTransit.RabbitMQ
builder.AddMassTransitRabbitMq("messaging", configurator =>
{
    configurator.AddConsumer<OrderCreatedConsumer>();
    configurator.AddSagaStateMachine<OrderFulfillmentSaga, OrderFulfillmentState>();
});
```

---

## Step 2155: Observability - Custom Metrics และ Tracing

```csharp
// ECommerceAspire.OrderService/Telemetry/OrderMetrics.cs
using System.Diagnostics;
using System.Diagnostics.Metrics;

public sealed class OrderMetrics : IDisposable
{
    private readonly Meter _meter;
    private readonly Counter<long> _ordersCreated;
    private readonly Histogram<double> _orderProcessingTime;
    private readonly ObservableGauge<int> _pendingOrders;
    private int _pendingOrderCount;

    public static readonly ActivitySource ActivitySource = new("ECommerce.OrderService");

    public OrderMetrics(IMeterFactory meterFactory)
    {
        _meter = meterFactory.Create("ECommerce.OrderService");

        _ordersCreated = _meter.CreateCounter<long>(
            "ecommerce.orders.created",
            unit: "{order}",
            description: "Total number of orders created");

        _orderProcessingTime = _meter.CreateHistogram<double>(
            "ecommerce.orders.processing_duration",
            unit: "ms",
            description: "Order processing duration in milliseconds");

        _pendingOrders = _meter.CreateObservableGauge<int>(
            "ecommerce.orders.pending",
            () => _pendingOrderCount,
            unit: "{order}",
            description: "Current number of pending orders");
    }

    public void RecordOrderCreated(string customerId, decimal amount)
    {
        _ordersCreated.Add(1,
            new TagList
            {
                { "customer.tier", GetCustomerTier(amount) },
                { "order.currency", "THB" }
            });
    }

    public void RecordProcessingTime(double milliseconds, bool success)
    {
        _orderProcessingTime.Record(milliseconds,
            new TagList { { "processing.success", success } });
    }

    public void SetPendingOrderCount(int count) => _pendingOrderCount = count;

    private static string GetCustomerTier(decimal amount) => amount switch
    {
        >= 10000 => "platinum",
        >= 5000 => "gold",
        >= 1000 => "silver",
        _ => "standard"
    };

    public void Dispose() => _meter.Dispose();
}

// Register in DI
builder.Services.AddSingleton<OrderMetrics>();

// Use in service
public class OrderService(OrderMetrics metrics, ILogger<OrderService> logger)
{
    public async Task<Order> CreateOrderAsync(CreateOrderCommand command, CancellationToken ct)
    {
        using var activity = OrderMetrics.ActivitySource.StartActivity("CreateOrder");
        activity?.SetTag("order.customer_id", command.CustomerId);

        var sw = Stopwatch.StartNew();
        try
        {
            var order = await ProcessOrderAsync(command, ct);
            
            activity?.SetTag("order.id", order.Id);
            activity?.SetTag("order.total", order.Total.Amount);
            activity?.SetStatus(ActivityStatusCode.Ok);
            
            metrics.RecordOrderCreated(command.CustomerId.ToString(), order.Total.Amount);
            return order;
        }
        catch (Exception ex)
        {
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            activity?.RecordException(ex);
            throw;
        }
        finally
        {
            sw.Stop();
            metrics.RecordProcessingTime(sw.Elapsed.TotalMilliseconds, true);
        }
    }
}
```

---

## Step 2156: Aspire Dashboard

```
Aspire Dashboard (localhost:18888)
===================================

Services Panel:
┌─────────────────┬──────────┬──────────┬────────────────┐
│ Service         │ State    │ Health   │ URL            │
├─────────────────┼──────────┼──────────┼────────────────┤
│ catalog-api     │ Running  │ Healthy  │ https://...    │
│ catalog-api (2) │ Running  │ Healthy  │ https://...    │
│ order-service   │ Running  │ Healthy  │ https://...    │
│ order-processor │ Running  │ Healthy  │ https://...    │
│ web-frontend    │ Running  │ Healthy  │ https://...    │
│ postgres        │ Running  │ -        │ -              │
│ redis           │ Running  │ -        │ -              │
│ messaging       │ Running  │ -        │ -              │
└─────────────────┴──────────┴──────────┴────────────────┘

Traces Panel (OpenTelemetry):
- HTTP requests with full trace context propagation
- Database queries with timing
- Cross-service call graphs
- Error highlighting

Metrics Panel:
- Request rate, latency percentiles
- Custom business metrics
- Resource utilization

Logs Panel:
- Structured logs from all services
- Correlated by trace ID
- Severity filtering
```

---

## Step 2157: Testing Aspire Applications

```csharp
// ECommerceAspire.Tests/IntegrationTests/OrderServiceTests.cs
using Aspire.Hosting.Testing;

public class OrderServiceIntegrationTests : IAsyncLifetime
{
    private DistributedApplication _app = null!;

    public async Task InitializeAsync()
    {
        var appHost = await DistributedApplicationTestingBuilder
            .CreateAsync<Projects.ECommerceAspire_AppHost>();

        appHost.Services.ConfigureHttpClientDefaults(clientBuilder =>
        {
            clientBuilder.AddStandardHedgingHandler();
        });

        _app = await appHost.BuildAsync();
        await _app.StartAsync();
    }

    [Fact]
    public async Task CreateOrder_ShouldReturnOrderId()
    {
        // Wait for services to be healthy
        var catalogApi = _app.CreateHttpClient("catalog-api");
        await _app.WaitForHealthyAsync("catalog-api");
        await _app.WaitForHealthyAsync("order-service");

        var orderService = _app.CreateHttpClient("order-service");

        var command = new CreateOrderRequest
        {
            CustomerId = Guid.NewGuid(),
            Items = [new OrderItemRequest { ProductId = Guid.NewGuid(), Quantity = 2 }]
        };

        var response = await orderService.PostAsJsonAsync("/api/orders", command);
        response.EnsureSuccessStatusCode();

        var orderId = await response.Content.ReadFromJsonAsync<Guid>();
        orderId.Should().NotBeEmpty();
    }

    [Fact]
    public async Task ServiceDiscovery_CatalogApiShouldBeReachable()
    {
        var resourceNotificationService = _app.Services
            .GetRequiredService<ResourceNotificationService>();

        await resourceNotificationService
            .WaitForResourceAsync("catalog-api", KnownResourceStates.Running)
            .WaitAsync(TimeSpan.FromSeconds(30));

        var httpClient = _app.CreateHttpClient("catalog-api");
        var response = await httpClient.GetAsync("/health");
        response.IsSuccessStatusCode.Should().BeTrue();
    }

    public async Task DisposeAsync() => await _app.DisposeAsync();
}
```

---

## Step 2158: Custom Aspire Resources

```csharp
// AppHost/Resources/ElasticsearchResource.cs
public class ElasticsearchResource(string name) : ContainerResource(name), IResourceWithConnectionString
{
    internal const string DefaultContainerName = "elasticsearch";
    private const int DefaultPort = 9200;

    private EndpointReference? _endpoint;

    public EndpointReference Endpoint =>
        _endpoint ??= new EndpointReference(this, "http");

    public ReferenceExpression ConnectionStringExpression =>
        ReferenceExpression.Create($"http://{Endpoint.Property(EndpointProperty.Host)}:{Endpoint.Property(EndpointProperty.Port)}");
}

// Extension methods
public static class ElasticsearchExtensions
{
    public static IResourceBuilder<ElasticsearchResource> AddElasticsearch(
        this IDistributedApplicationBuilder builder,
        string name,
        int? port = null)
    {
        var resource = new ElasticsearchResource(name);

        return builder.AddResource(resource)
            .WithImage("docker.elastic.co/elasticsearch/elasticsearch")
            .WithImageTag("8.11.0")
            .WithHttpEndpoint(targetPort: 9200, port: port, name: "http")
            .WithEnvironment("discovery.type", "single-node")
            .WithEnvironment("xpack.security.enabled", "false")
            .WithEnvironment("ES_JAVA_OPTS", "-Xms512m -Xmx512m")
            .WithDataVolume("elasticsearch-data");
    }

    public static IResourceBuilder<TDestination> WithReference<TDestination>(
        this IResourceBuilder<TDestination> builder,
        IResourceBuilder<ElasticsearchResource> source)
        where TDestination : IResourceWithEnvironment
    {
        return builder.WithEnvironment(context =>
        {
            context.EnvironmentVariables["ELASTICSEARCH__URL"] =
                source.Resource.ConnectionStringExpression;
        });
    }
}

// Usage in AppHost
var elasticsearch = builder.AddElasticsearch("search");
var orderService = builder.AddProject<Projects.OrderService>("order-service")
    .WithReference(elasticsearch);
```

---

## Step 2159: Aspire Manifest และ Deployment

```bash
# Generate deployment manifest
dotnet run --project ECommerceAspire.AppHost -- \
  --publisher manifest \
  --output-path aspire-manifest.json

# Deploy to Azure Container Apps
dotnet tool install -g azd
azd init --from-code
azd up

# Deploy to Kubernetes
dotnet run --project ECommerceAspire.AppHost -- \
  --publisher kubernetes \
  --output-path ./k8s-manifests
```

**aspire-manifest.json (partial):**

```json
{
  "$schema": "https://json.schemastore.org/aspire-8.0.json",
  "resources": {
    "postgres": {
      "type": "container.v0",
      "image": "docker.io/library/postgres:16.2",
      "env": {
        "POSTGRES_HOST_AUTH_METHOD": "scram-sha-256",
        "POSTGRES_INITDB_ARGS": "--auth-host=scram-sha-256 --auth-local=scram-sha-256"
      },
      "bindings": {
        "tcp": {
          "scheme": "tcp",
          "protocol": "tcp",
          "transport": "tcp",
          "targetPort": 5432
        }
      }
    },
    "order-service": {
      "type": "project.v0",
      "path": "../ECommerceAspire.OrderService/ECommerceAspire.OrderService.csproj",
      "env": {
        "OTEL_DOTNET_EXPERIMENTAL_OTLP_EMIT_EVENT_LOG_NOTIFICATIONS": "true",
        "ConnectionStrings__orderdb": "{orderdb.connectionString}",
        "ConnectionStrings__redis": "{redis.connectionString}",
        "ConnectionStrings__messaging": "{messaging.connectionString}",
        "services__catalog-api__https__0": "{catalog-api.bindings.https.url}"
      },
      "bindings": {
        "http": { "scheme": "http", "protocol": "tcp", "transport": "http" },
        "https": { "scheme": "https", "protocol": "tcp", "transport": "http" }
      }
    }
  }
}
```

---

## Step 2160: Aspire + Azure Integrations

```csharp
// AppHost - Azure resources
// Package: Aspire.Hosting.Azure

var builder = DistributedApplication.CreateBuilder(args);

if (builder.ExecutionContext.IsPublishMode)
{
    // Use Azure services in production/publish mode
    var keyVault = builder.AddAzureKeyVault("secrets");
    
    var serviceBus = builder.AddAzureServiceBus("messaging")
        .AddQueue("orders")
        .AddTopic("order-events");

    var cosmosDb = builder.AddAzureCosmosDB("cosmos")
        .AddDatabase("ecommerce");

    var signalR = builder.AddAzureSignalR("signalr");

    var openAi = builder.AddAzureOpenAI("openai")
        .AddDeployment(new AzureOpenAIDeployment("gpt-4o", "gpt-4o", "2024-08-06"));

    var orderService = builder.AddProject<Projects.OrderService>("order-service")
        .WithReference(serviceBus)
        .WithReference(cosmosDb)
        .WithReference(keyVault)
        .WithReference(openAi);
}
else
{
    // Use local emulators in development
    var rabbitMq = builder.AddRabbitMQ("messaging");
    var postgres = builder.AddPostgres("postgres");

    var orderService = builder.AddProject<Projects.OrderService>("order-service")
        .WithReference(rabbitMq)
        .WithReference(postgres.AddDatabase("orderdb"));
}

builder.Build().Run();
```

---

## Step 2161: Health Checks Advanced Configuration

```csharp
// ServiceDefaults or individual service
builder.Services.AddHealthChecks()
    .AddNpgSql(
        connectionString: builder.Configuration.GetConnectionString("orderdb")!,
        healthQuery: "SELECT 1",
        name: "postgresql",
        failureStatus: HealthStatus.Unhealthy,
        tags: ["db", "postgresql", "ready"])
    .AddRedis(
        redisConnectionString: builder.Configuration.GetConnectionString("redis")!,
        name: "redis",
        failureStatus: HealthStatus.Degraded,
        tags: ["cache", "redis", "ready"])
    .AddRabbitMQ(
        rabbitConnectionString: builder.Configuration.GetConnectionString("messaging")!,
        name: "rabbitmq",
        tags: ["messaging", "ready"])
    .AddCheck<ExternalServiceHealthCheck>(
        "external-api",
        failureStatus: HealthStatus.Degraded,
        tags: ["external"]);

// Custom health check
public class ExternalServiceHealthCheck(HttpClient httpClient) : IHealthCheck
{
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken ct = default)
    {
        try
        {
            var response = await httpClient.GetAsync("/ping", ct);
            return response.IsSuccessStatusCode
                ? HealthCheckResult.Healthy("External service is responsive")
                : HealthCheckResult.Degraded($"External service returned {response.StatusCode}");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("External service is unreachable", ex);
        }
    }
}

// Map health check endpoints
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready"),
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("live")
});
```

---

## Step 2162: Resilience Pipelines ใน Aspire

```csharp
// ServiceDefaults adds standard resilience, but you can customize:
builder.Services.ConfigureHttpClientDefaults(http =>
{
    http.AddResilienceHandler("custom", pipeline =>
    {
        // Retry with exponential backoff
        pipeline.AddRetry(new HttpRetryStrategyOptions
        {
            MaxRetryAttempts = 3,
            BackoffType = DelayBackoffType.Exponential,
            UseJitter = true,
            Delay = TimeSpan.FromSeconds(1),
            ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
                .HandleResult(r => r.StatusCode >= HttpStatusCode.InternalServerError)
                .Handle<HttpRequestException>()
        });

        // Circuit breaker
        pipeline.AddCircuitBreaker(new HttpCircuitBreakerStrategyOptions
        {
            SamplingDuration = TimeSpan.FromSeconds(30),
            MinimumThroughput = 5,
            FailureRatio = 0.5,
            BreakDuration = TimeSpan.FromSeconds(30),
            OnOpened = args =>
            {
                logger.LogWarning("Circuit breaker opened for {Duration}", args.BreakDuration);
                return ValueTask.CompletedTask;
            }
        });

        // Timeout
        pipeline.AddTimeout(TimeSpan.FromSeconds(5));
    });
});

// Per-client resilience
builder.Services.AddHttpClient<ICatalogClient, CatalogClient>(
    client => client.BaseAddress = new Uri("https+http://catalog-api"))
    .AddStandardResilienceHandler(options =>
    {
        options.AttemptTimeout.Timeout = TimeSpan.FromSeconds(5);
        options.TotalRequestTimeout.Timeout = TimeSpan.FromSeconds(30);
        options.Retry.MaxRetryAttempts = 3;
    });
```

---

## Step 2163: Configuration Management ใน Aspire

```csharp
// AppHost - inject configuration to services
var apiKey = builder.AddParameter("catalog-api-key", secret: true);
var environment = builder.AddParameter("environment", "development");

var catalogApi = builder.AddProject<Projects.CatalogApi>("catalog-api")
    .WithEnvironment("ApiKey", apiKey)
    .WithEnvironment("Environment", environment);

// WithEnvironment with computed values
var orderService = builder.AddProject<Projects.OrderService>("order-service")
    .WithEnvironment("CATALOG_API_URL", catalogApi.GetEndpoint("https"))
    .WithEnvironment(ctx =>
    {
        ctx.EnvironmentVariables["FeatureFlags__NewCheckout"] = "true";
        ctx.EnvironmentVariables["MaxConcurrentOrders"] = "100";
    });
```

**secrets.json (AppHost - local secrets):**

```json
{
  "Parameters:catalog-api-key": "my-secret-api-key-12345"
}
```

```bash
# Set parameter value
dotnet user-secrets set "Parameters:catalog-api-key" "my-secret-key" \
  --project ECommerceAspire.AppHost
```

---

## Step 2164: Full Aspire Solution Structure

```
ECommerceAspire/
├── ECommerceAspire.sln
├── ECommerceAspire.AppHost/
│   ├── Program.cs                    # Orchestration
│   ├── Resources/
│   │   └── ElasticsearchResource.cs  # Custom resources
│   └── ECommerceAspire.AppHost.csproj
├── ECommerceAspire.ServiceDefaults/
│   ├── Extensions.cs                 # OpenTelemetry, health checks
│   └── ECommerceAspire.ServiceDefaults.csproj
├── ECommerceAspire.ApiService/       # Catalog API
│   ├── Program.cs
│   ├── Endpoints/
│   ├── Data/
│   └── ECommerceAspire.ApiService.csproj
├── ECommerceAspire.OrderService/     # Order API
│   ├── Program.cs
│   ├── Commands/
│   ├── Queries/
│   ├── Domain/
│   ├── Data/
│   ├── Telemetry/
│   │   └── OrderMetrics.cs
│   └── ECommerceAspire.OrderService.csproj
├── ECommerceAspire.OrderProcessor/   # Background worker
│   ├── Program.cs
│   ├── Consumers/
│   └── ECommerceAspire.OrderProcessor.csproj
├── ECommerceAspire.Web/              # Blazor frontend
│   ├── Program.cs
│   ├── Components/
│   └── ECommerceAspire.Web.csproj
└── ECommerceAspire.Tests/
    ├── Integration/
    │   └── OrderServiceTests.cs
    └── ECommerceAspire.Tests.csproj

Key Aspire Principles:
======================
1. AppHost = single source of truth for topology
2. ServiceDefaults = consistent cross-cutting concerns
3. Service discovery via environment variables (automatic)
4. OpenTelemetry built-in for all services
5. Local development mirrors production topology
6. Testable with DistributedApplicationTestingBuilder
7. Deploy to ACA/K8s via manifest generation
```

---

## สรุป Part 95

.NET Aspire เปลี่ยนวิธีพัฒนา distributed applications:
- **AppHost** orchestrates ทุก service และ infrastructure
- **ServiceDefaults** รวม observability และ resilience patterns
- **Service Discovery** อัตโนมัติผ่าน environment variables
- **Components** เชื่อม databases/caches ด้วย code เดียว
- **Dashboard** แสดง traces, metrics, logs จากทุก service
- **Testing** ด้วย `DistributedApplicationTestingBuilder` ทดสอบ end-to-end จริง
