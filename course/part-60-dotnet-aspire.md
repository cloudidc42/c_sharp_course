# Part 60: .NET Aspire — Cloud-Native Application Orchestration

## Steps 1598-1618: Building Production-Ready Distributed Applications

---

## Step 1598: .NET Aspire Architecture

```
.NET Aspire Stack
══════════════════════════════════════════════════════════════

  Developer Machine                      Production (Azure/k8s)
  ─────────────────                      ──────────────────────
  
  AppHost (orchestrator)                 Manifests → Bicep / Helm
  ┌─────────────────────────────────┐    ┌──────────────────────────────┐
  │  .AddProject("api")             │ →  │  ContainerApp / Deployment   │
  │  .AddProject("worker")          │    │  ContainerApp / Deployment   │
  │  .AddRedis("cache")             │    │  Azure Cache for Redis       │
  │  .AddPostgres("db")             │    │  Azure Database for PgSQL    │
  │  .AddRabbitMQ("mq")             │    │  Azure Service Bus           │
  │  .AddAzureServiceBus("bus")     │    │  Azure Service Bus           │
  └─────────────────────────────────┘    └──────────────────────────────┘
         │                                        │
         ▼                                        ▼
  Dashboard (localhost:18888)            OTel Collector → Grafana
  ┌─────────────────────────────────┐
  │  Traces / Metrics / Logs        │
  │  Resource health / env vars     │
  │  Structured log viewer          │
  └─────────────────────────────────┘

  Service Discovery: https+http://{service-name}
  Config injection: ConnectionStrings, OTEL_*, SERVICE_* env vars
```

### NuGet Packages

```xml
<!-- AppHost project -->
<PackageReference Include="Aspire.Hosting.AppHost"          Version="9.*" />
<PackageReference Include="Aspire.Hosting.Azure.Storage"    Version="9.*" />
<PackageReference Include="Aspire.Hosting.Azure.ServiceBus" Version="9.*" />
<PackageReference Include="Aspire.Hosting.Azure.KeyVault"   Version="9.*" />
<PackageReference Include="Aspire.Hosting.Redis"            Version="9.*" />
<PackageReference Include="Aspire.Hosting.PostgreSQL"       Version="9.*" />
<PackageReference Include="Aspire.Hosting.RabbitMQ"         Version="9.*" />
<PackageReference Include="Aspire.Hosting.Kafka"            Version="9.*" />

<!-- Service project -->
<PackageReference Include="Aspire.StackExchange.Redis"              Version="9.*" />
<PackageReference Include="Aspire.Npgsql.EntityFrameworkCore.PostgreSQL" Version="9.*" />
<PackageReference Include="Aspire.RabbitMQ.Client"                  Version="9.*" />
<PackageReference Include="Microsoft.Extensions.ServiceDiscovery"   Version="9.*" />
```

---

## Step 1599: AppHost — Orchestration

```csharp
// AspireApp.AppHost/Program.cs
using Aspire.Hosting;
using Aspire.Hosting.Azure;

var builder = DistributedApplication.CreateBuilder(args);

// ── Infrastructure ──────────────────────────────────────────────────────────

var postgres = builder.AddPostgres("postgres")
    .WithDataVolume("aspire-pg-data")
    .WithPgAdmin();   // pgAdmin UI on http://localhost:5050

var ordersDb    = postgres.AddDatabase("ordersdb");
var inventoryDb = postgres.AddDatabase("inventorydb");

var redis = builder.AddRedis("redis")
    .WithRedisCommander()  // Redis Commander UI
    .WithDataVolume("aspire-redis-data");

var rabbitmq = builder.AddRabbitMQ("rabbitmq")
    .WithManagementPlugin()  // Management UI on http://localhost:15672
    .WithDataVolume("aspire-rabbit-data");

var kafka = builder.AddKafka("kafka")
    .WithKafkaUI();

// ── Azure Resources (use emulators in dev) ──────────────────────────────────

var storage = builder.AddAzureStorage("storage")
    .RunAsEmulator(config =>
    {
        config.WithDataVolume("aspire-storage-data");
    });

var blobs   = storage.AddBlobs("blobs");
var queues  = storage.AddQueues("queues");
var tables  = storage.AddTables("tables");

var serviceBus = builder.AddAzureServiceBus("servicebus")
    .RunAsEmulator()
    .AddQueue("orders")
    .AddTopic("order-events")
    .AddSubscription("order-events", "analytics-sub");

var keyVault = builder.AddAzureKeyVault("keyvault")
    .RunAsEmulator();

// ── Application Projects ─────────────────────────────────────────────────────

var orderApi = builder.AddProject<Projects.OrderApi>("order-api")
    .WithReference(ordersDb)
    .WithReference(redis)
    .WithReference(rabbitmq)
    .WithReference(serviceBus)
    .WithReference(blobs)
    .WithEnvironment("ASPIRE_ENVIRONMENT", "Development")
    .WithReplicas(2);  // run 2 instances

var inventoryApi = builder.AddProject<Projects.InventoryApi>("inventory-api")
    .WithReference(inventoryDb)
    .WithReference(redis)
    .WithReference(rabbitmq);

var worker = builder.AddProject<Projects.OrderWorker>("order-worker")
    .WithReference(ordersDb)
    .WithReference(rabbitmq)
    .WithReference(serviceBus)
    .WithReference(blobs);

// API gateway referencing both services
var gateway = builder.AddProject<Projects.Gateway>("gateway")
    .WithReference(orderApi)
    .WithReference(inventoryApi)
    .WithExternalHttpEndpoints();

// Frontend
var frontend = builder.AddNpmApp("frontend", "../AspireApp.Frontend")
    .WithReference(gateway)
    .WithEnvironment("VITE_API_URL", gateway.GetEndpoint("https"));

// ── Observability ─────────────────────────────────────────────────────────────

// Aspire's built-in dashboard is the default; add external if needed:
// builder.AddDashboard();

builder.Build().Run();
```

---

## Step 1600: Service Discovery & Configuration

```csharp
// Service projects automatically get:
// 1. Connection strings injected via environment variables
// 2. Service discovery via HTTP endpoint names
// 3. OpenTelemetry configured automatically

// OrderApi/Program.cs
var builder = WebApplication.CreateBuilder(args);

// ── Aspire Service Defaults ─────────────────────────────────────────────────
builder.AddServiceDefaults(); // registers OTel, health checks, service discovery

// ── Database ────────────────────────────────────────────────────────────────
// Connection string injected by Aspire as "ConnectionStrings:ordersdb"
builder.AddNpgsqlDbContext<OrderDbContext>("ordersdb", config =>
{
    config.DisableRetry   = false;
    config.MaxRetryCount  = 3;
    config.CommandTimeout = 30;
});

// ── Redis ────────────────────────────────────────────────────────────────────
// Connection string injected as "ConnectionStrings:redis"
builder.AddRedisClient("redis");
builder.AddRedisDistributedCache("redis");

// ── RabbitMQ ─────────────────────────────────────────────────────────────────
builder.AddRabbitMQClient("rabbitmq", configure =>
{
    configure.ConnectionFactory.RequestedHeartbeat = TimeSpan.FromSeconds(60);
});

// ── Blob Storage ─────────────────────────────────────────────────────────────
builder.AddAzureBlobClient("blobs");

// ── Service Discovery ─────────────────────────────────────────────────────────
// Resolve "inventory-api" by name; Aspire handles DNS/port mapping
builder.Services.AddHttpClient<IInventoryClient, InventoryClient>(c =>
{
    c.BaseAddress = new Uri("https+http://inventory-api");  // service name
})
.AddServiceDiscovery();

var app = builder.Build();
app.MapDefaultEndpoints();  // /health, /alive, /ready
app.Run();
```

---

## Step 1601: ServiceDefaults Extension

```csharp
// ServiceDefaults/Extensions.cs (shared project)
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Diagnostics.HealthChecks;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Diagnostics.HealthChecks;
using OpenTelemetry;
using OpenTelemetry.Metrics;
using OpenTelemetry.Trace;

namespace Microsoft.Extensions.Hosting;

public static class ServiceDefaultsExtensions
{
    public static IHostApplicationBuilder AddServiceDefaults(
        this IHostApplicationBuilder builder)
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

    public static IHostApplicationBuilder ConfigureOpenTelemetry(
        this IHostApplicationBuilder builder)
    {
        builder.Logging.AddOpenTelemetry(log =>
        {
            log.IncludeFormattedMessage = true;
            log.IncludeScopes          = true;
        });

        builder.Services.AddOpenTelemetry()
            .WithMetrics(metrics =>
            {
                metrics
                    .AddAspNetCoreInstrumentation()
                    .AddHttpClientInstrumentation()
                    .AddRuntimeInstrumentation();
            })
            .WithTracing(tracing =>
            {
                tracing
                    .AddAspNetCoreInstrumentation()
                    .AddHttpClientInstrumentation()
                    .AddEntityFrameworkCoreInstrumentation();
            });

        builder.AddOpenTelemetryExporters();
        return builder;
    }

    private static IHostApplicationBuilder AddOpenTelemetryExporters(
        this IHostApplicationBuilder builder)
    {
        var useOtlpExporter = !string.IsNullOrWhiteSpace(
            builder.Configuration["OTEL_EXPORTER_OTLP_ENDPOINT"]);

        if (useOtlpExporter)
        {
            builder.Services.AddOpenTelemetry().UseOtlpExporter();
        }

        return builder;
    }

    public static IHostApplicationBuilder AddDefaultHealthChecks(
        this IHostApplicationBuilder builder)
    {
        builder.Services.AddHealthChecks()
            .AddCheck("self", () => HealthCheckResult.Healthy(), ["live"]);
        return builder;
    }

    public static WebApplication MapDefaultEndpoints(this WebApplication app)
    {
        app.MapHealthChecks("/health");

        app.MapHealthChecks("/alive", new HealthCheckOptions
        {
            Predicate = r => r.Tags.Contains("live")
        });

        app.MapHealthChecks("/ready", new HealthCheckOptions
        {
            Predicate = r => r.Tags.Contains("ready")
        });

        return app;
    }
}
```

---

## Step 1602: Resilient HTTP Client

```csharp
// Service-to-service HTTP with built-in resilience
// (AddStandardResilienceHandler in AddServiceDefaults gives retry + circuit breaker)

// Custom override for specific requirements
builder.Services.AddHttpClient<IOrderApiClient, OrderApiClient>(c =>
{
    c.BaseAddress = new Uri("https+http://order-api");
    c.Timeout     = TimeSpan.FromSeconds(30);
})
.AddServiceDiscovery()
.AddResilienceHandler("order-client", pipeline =>
{
    pipeline.AddTimeout(TimeSpan.FromSeconds(10));

    pipeline.AddRetry(new Polly.Retry.RetryStrategyOptions<HttpResponseMessage>
    {
        MaxRetryAttempts = 3,
        BackoffType      = DelayBackoffType.Exponential,
        UseJitter        = true,
        Delay            = TimeSpan.FromMilliseconds(500),
        ShouldHandle     = new PredicateBuilder<HttpResponseMessage>()
            .HandleResult(r => r.StatusCode == System.Net.HttpStatusCode.ServiceUnavailable)
            .Handle<HttpRequestException>()
    });

    pipeline.AddCircuitBreaker(new Polly.CircuitBreaker.CircuitBreakerStrategyOptions<HttpResponseMessage>
    {
        FailureRatio      = 0.5,
        SamplingDuration  = TimeSpan.FromSeconds(30),
        MinimumThroughput = 10,
        BreakDuration     = TimeSpan.FromSeconds(30),
        ShouldHandle      = new PredicateBuilder<HttpResponseMessage>()
            .HandleResult(r => (int)r.StatusCode >= 500)
    });
});
```

---

## Step 1603: Custom Resource Extensions

```csharp
// AppHost/Extensions/CustomExtensions.cs
// Add a custom external service (e.g., Stripe, SendGrid)
public static class ExternalServiceExtensions
{
    public static IResourceBuilder<ExternalServiceResource> AddStripe(
        this IDistributedApplicationBuilder builder,
        string name,
        string? apiKey = null)
    {
        var resource = new ExternalServiceResource(name, "https://api.stripe.com");
        return builder.AddResource(resource)
            .WithEnvironment("Stripe__ApiKey", apiKey ?? builder.Configuration["Stripe:ApiKey"]!);
    }
}

public class ExternalServiceResource : Resource, IResourceWithConnectionString
{
    private readonly string _baseUrl;

    public ExternalServiceResource(string name, string baseUrl) : base(name)
        => _baseUrl = baseUrl;

    public ReferenceExpression ConnectionStringExpression
        => ReferenceExpression.Create($"{_baseUrl}");
}

// Container extension
public static IResourceBuilder<ContainerResource> AddOpenSearchContainer(
    this IDistributedApplicationBuilder builder,
    string name)
{
    return builder.AddContainer(name, "opensearchproject/opensearch", "2.12")
        .WithEnvironment("discovery.type", "single-node")
        .WithEnvironment("OPENSEARCH_INITIAL_ADMIN_PASSWORD", "Admin@1234!")
        .WithBindMount("./opensearch-data", "/usr/share/opensearch/data")
        .WithHttpEndpoint(targetPort: 9200, name: "http")
        .WithHttpEndpoint(targetPort: 9600, name: "performance-analyzer");
}
```

---

## Step 1604: AppHost Integration Tests

```csharp
// Tests/AspireIntegrationTests.cs
using Aspire.Hosting.Testing;
using Microsoft.Extensions.DependencyInjection;
using System.Net.Http.Json;

namespace AspireApp.Tests;

public class AspireIntegrationTests : IAsyncLifetime
{
    private DistributedApplication _app = null!;

    public async Task InitializeAsync()
    {
        var appHost = await DistributedApplicationTestingBuilder
            .CreateAsync<Projects.AspireApp_AppHost>();

        // Override for tests
        appHost.Services.ConfigureHttpClientDefaults(c => c.AddStandardResilienceHandler());

        _app = await appHost.BuildAsync();
        await _app.StartAsync();
    }

    public async Task DisposeAsync()
    {
        await _app.DisposeAsync();
    }

    [Fact]
    public async Task OrderApi_HealthCheck_ReturnsHealthy()
    {
        var http = _app.CreateHttpClient("order-api");

        var response = await http.GetAsync("/health");

        response.EnsureSuccessStatusCode();
        var body = await response.Content.ReadAsStringAsync();
        Assert.Contains("Healthy", body);
    }

    [Fact]
    public async Task CreateOrder_EndToEnd_ReturnsCreated()
    {
        var http = _app.CreateHttpClient("order-api");

        var response = await http.PostAsJsonAsync("/api/orders", new
        {
            customerId      = "aspire-test-customer",
            items           = new[] { new { productId = "p1", name = "Widget", quantity = 2 } },
            shippingAddress = "123 Test St"
        });

        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
        var created = await response.Content.ReadFromJsonAsync<CreatedOrderResponse>();
        Assert.NotNull(created?.OrderId);
    }

    [Fact]
    public async Task InventoryApi_CheckStock_ReturnsAvailability()
    {
        var http     = _app.CreateHttpClient("inventory-api");
        var response = await http.GetAsync("/api/inventory/p1/stock?quantity=1");

        response.EnsureSuccessStatusCode();
        var result = await response.Content.ReadFromJsonAsync<StockCheckResult>();
        Assert.NotNull(result);
    }

    [Fact]
    public async Task ServiceDiscovery_GatewayRoutesTo_OrderApi()
    {
        var http = _app.CreateHttpClient("gateway");

        // Gateway routes /api/orders → order-api via service discovery
        var response = await http.GetAsync("/api/orders/health");
        response.EnsureSuccessStatusCode();
    }

    [Fact]
    public async Task Redis_CachingWorks()
    {
        var http = _app.CreateHttpClient("order-api");

        // First call: cache miss
        var sw1 = System.Diagnostics.Stopwatch.StartNew();
        await http.GetAsync("/api/products/p1");
        sw1.Stop();

        // Second call: cache hit (should be faster)
        var sw2 = System.Diagnostics.Stopwatch.StartNew();
        await http.GetAsync("/api/products/p1");
        sw2.Stop();

        // Cache hit should be significantly faster
        Assert.True(sw2.ElapsedMilliseconds < sw1.ElapsedMilliseconds * 0.5 || sw2.ElapsedMilliseconds < 50);
    }
}
```

---

## Step 1605: Publishing to Azure Container Apps

```csharp
// AppHost/Program.cs (production configuration)
var builder = DistributedApplication.CreateBuilder(args);

// Azure infrastructure
var postgres = builder.AddAzurePostgresFlexibleServer("postgres")
    .WithPasswordAuthentication();  // use managed identity in prod

var ordersDb = postgres.AddDatabase("ordersdb");

var redis = builder.AddAzureRedis("redis");

var serviceBus = builder.AddAzureServiceBus("servicebus")
    .AddQueue("orders")
    .AddTopic("order-events");

var keyVault = builder.AddAzureKeyVault("keyvault");

// Application projects
var orderApi = builder.AddProject<Projects.OrderApi>("order-api")
    .WithReference(ordersDb)
    .WithReference(redis)
    .WithReference(serviceBus)
    .WithReference(keyVault)
    .WithExternalHttpEndpoints();

builder.Build().Run();
```

```bash
# Deploy to Azure Container Apps via Aspire CLI
# 1. Generate manifests
dotnet run --project AspireApp.AppHost -- --publisher manifest --output-path ./manifests

# 2. Provision + deploy with azd
azd init --template aspire
azd up

# 3. Or use Aspire's Azure deployment
dotnet run --project AspireApp.AppHost -- --publisher azure --output-path ./azure-infra
```

---

## Step 1606: Kubernetes Deployment via Aspire

```yaml
# Generated manifests/aspire-manifest.json → k8s yamls via aspirate tool
# aspirate generate
# aspirate apply --env Production

# Typical generated Kubernetes deployment for order-api:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-api
  labels:
    app: order-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: order-api
  template:
    metadata:
      labels:
        app: order-api
      annotations:
        # Scrape metrics
        prometheus.io/scrape: "true"
        prometheus.io/port:   "8080"
        prometheus.io/path:   "/metrics"
    spec:
      containers:
        - name: order-api
          image: registry.example.com/order-api:latest
          ports:
            - containerPort: 8080
              name: http
            - containerPort: 8443
              name: https
          env:
            - name: ASPNETCORE_ENVIRONMENT
              value: Production
            - name: ConnectionStrings__ordersdb
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: connection-string
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: "http://otel-collector:4317"
            - name: OTEL_RESOURCE_ATTRIBUTES
              value: "service.name=order-api,service.version=1.0.0"
          resources:
            requests:
              cpu:    "100m"
              memory: "256Mi"
            limits:
              cpu:    "500m"
              memory: "512Mi"
          livenessProbe:
            httpGet:
              path: /alive
              port: 8080
            initialDelaySeconds: 5
            periodSeconds:       10
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds:       5
          startupProbe:
            httpGet:
              path: /health
              port: 8080
            failureThreshold: 30
            periodSeconds:    10
---
apiVersion: v1
kind: Service
metadata:
  name: order-api
spec:
  selector:
    app: order-api
  ports:
    - name: http
      port: 80
      targetPort: 8080
    - name: https
      port: 443
      targetPort: 8443
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-api
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "1000"
```

---

## Step 1607: Observability Dashboard Configuration

```yaml
# docker-compose.observability.yml — full Aspire-compatible observability stack
version: '3.9'

services:
  # OpenTelemetry Collector
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    command: ["--config=/etc/otel-collector-config.yaml"]
    volumes:
      - ./otel-collector-config.yaml:/etc/otel-collector-config.yaml
    ports:
      - "4317:4317"   # gRPC OTLP
      - "4318:4318"   # HTTP OTLP
      - "8889:8889"   # Prometheus scrape

  # Prometheus
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  # Grafana
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning
      - grafana-data:/var/lib/grafana

  # Loki (logs)
  loki:
    image: grafana/loki:latest
    ports:
      - "3100:3100"

  # Jaeger (traces)
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"   # UI
      - "14317:4317"    # OTLP gRPC

volumes:
  grafana-data:
```

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 5s
    send_batch_size: 1024

  memory_limiter:
    check_interval: 1s
    limit_mib: 512

  resource:
    attributes:
      - key: deployment.environment
        value: development
        action: insert

exporters:
  prometheus:
    endpoint: 0.0.0.0:8889

  loki:
    endpoint: http://loki:3100/loki/api/v1/push
    labels:
      attributes:
        service.name: ""
        service.version: ""

  jaeger:
    endpoint: http://jaeger:14317
    tls:
      insecure: true

service:
  pipelines:
    traces:
      receivers:  [otlp]
      processors: [memory_limiter, batch]
      exporters:  [jaeger]
    metrics:
      receivers:  [otlp]
      processors: [memory_limiter, batch]
      exporters:  [prometheus]
    logs:
      receivers:  [otlp]
      processors: [memory_limiter, resource, batch]
      exporters:  [loki]
```

---

## Step 1608: Worker Service Integration

```csharp
// OrderWorker/Program.cs
var builder = Host.CreateApplicationBuilder(args);

// Aspire service defaults
builder.AddServiceDefaults();

// Resources
builder.AddNpgsqlDbContext<OrderDbContext>("ordersdb");
builder.AddRabbitMQClient("rabbitmq");
builder.AddAzureServiceBusClient("servicebus");
builder.AddAzureBlobServiceClient("blobs");

// Worker
builder.Services.AddHostedService<OrderProcessingWorker>();
builder.Services.AddHostedService<OrderEventPublisherWorker>();

var host = builder.Build();
host.Run();
```

```csharp
// OrderWorker/Workers/OrderProcessingWorker.cs
public class OrderProcessingWorker : BackgroundService
{
    private readonly IServiceScopeFactory _scope;
    private readonly ILogger<OrderProcessingWorker> _log;

    public OrderProcessingWorker(
        IServiceScopeFactory scope,
        ILogger<OrderProcessingWorker> log)
    {
        _scope = scope;
        _log   = log;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _log.LogInformation("Order processing worker started");

        await using var scope   = _scope.CreateAsyncScope();
        var rabbit = scope.ServiceProvider.GetRequiredService<IConnection>();

        using var channel = await rabbit.CreateChannelAsync();
        await channel.QueueDeclareAsync("order-processing",
            durable: true, exclusive: false, autoDelete: false);

        await channel.BasicQosAsync(0, 10, false); // prefetch 10

        var consumer = new AsyncEventingBasicConsumer(channel);
        consumer.ReceivedAsync += async (_, args) =>
        {
            var messageId = args.BasicProperties.MessageId;
            _log.LogInformation("Processing order message {MessageId}", messageId);

            try
            {
                var json  = System.Text.Encoding.UTF8.GetString(args.Body.Span);
                var msg   = System.Text.Json.JsonSerializer.Deserialize<OrderMessage>(json);

                await ProcessOrderAsync(msg!, stoppingToken);
                await channel.BasicAckAsync(args.DeliveryTag, false);
            }
            catch (Exception ex)
            {
                _log.LogError(ex, "Failed to process message {MessageId}", messageId);
                // Requeue for retry (with DLX for poison messages)
                await channel.BasicNackAsync(args.DeliveryTag, false, requeue: true);
            }
        };

        await channel.BasicConsumeAsync("order-processing", autoAck: false, consumer);

        // Keep alive
        await Task.Delay(Timeout.Infinite, stoppingToken);
    }

    private async Task ProcessOrderAsync(OrderMessage msg, CancellationToken ct)
    {
        await using var scope  = _scope.CreateAsyncScope();
        var handler = scope.ServiceProvider.GetRequiredService<OrderCommandHandler>();
        await handler.HandleAsync(new ConfirmOrderCommand(msg.OrderId, "system", "warehouse-main"), ct);
    }
}
```

---

## Step 1609: Configuration Management with Key Vault

```csharp
// Production secrets via Azure Key Vault
// AppHost auto-wires key vault references

// In service project (auto-configured by Aspire)
builder.Configuration.AddAzureKeyVaultSecrets("keyvault"); // injected by Aspire

// Or manually for specific secrets:
builder.Services.AddAzureClients(clients =>
{
    clients.AddSecretClient(
        new Uri(builder.Configuration["KeyVault:Uri"]!));
});

// Access secrets
public class PaymentService
{
    private readonly SecretClient _secrets;

    public PaymentService(SecretClient secrets) => _secrets = secrets;

    public async Task<string> GetStripeKeyAsync(CancellationToken ct = default)
    {
        var secret = await _secrets.GetSecretAsync("stripe-api-key", cancellationToken: ct);
        return secret.Value.Value;
    }
}
```

---

## Step 1610: Aspire Dashboard Deep Dive

```csharp
// Custom structured logs for Aspire dashboard
using Microsoft.Extensions.Logging;

public class OrderService
{
    private readonly ILogger<OrderService> _log;

    public async Task CreateOrderAsync(CreateOrderCommand cmd)
    {
        // Structured log — shows in Aspire dashboard with searchable fields
        using var activity = ActivitySource.StartActivity("order.create");
        activity?.SetTag("order.customer_id", cmd.CustomerId);
        activity?.SetTag("order.items_count", cmd.Items.Count);

        _log.LogInformation(
            "Creating order for customer {CustomerId} with {ItemCount} items",
            cmd.CustomerId,
            cmd.Items.Count);

        try
        {
            var orderId = Guid.NewGuid();
            // ... create order ...

            activity?.SetTag("order.id", orderId);
            _log.LogInformation("Order {OrderId} created successfully", orderId);

            return orderId;
        }
        catch (Exception ex)
        {
            activity?.RecordException(ex);
            _log.LogError(ex, "Failed to create order for {CustomerId}", cmd.CustomerId);
            throw;
        }
    }
}

// Environment variables Aspire injects for OTel:
// OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:18889
// OTEL_SERVICE_NAME=order-api
// OTEL_RESOURCE_ATTRIBUTES=service.instance.id=...,service.version=...
```

---

## Step 1611: Complete Project Structure

```
AspireApp/
├── AspireApp.AppHost/           ← Orchestrator
│   ├── Program.cs
│   └── Extensions/
│       └── CustomExtensions.cs
│
├── AspireApp.ServiceDefaults/   ← Shared config
│   └── Extensions.cs
│
├── AspireApp.OrderApi/          ← Order microservice
│   ├── Program.cs
│   ├── Endpoints/
│   ├── Services/
│   ├── Data/
│   └── appsettings.json
│
├── AspireApp.InventoryApi/      ← Inventory microservice
│   ├── Program.cs
│   └── ...
│
├── AspireApp.OrderWorker/       ← Background worker
│   ├── Program.cs
│   └── Workers/
│
├── AspireApp.Gateway/           ← YARP API gateway
│   ├── Program.cs
│   └── yarp.json
│
├── AspireApp.Frontend/          ← React/Vue (npm)
│   ├── package.json
│   └── src/
│
└── AspireApp.Tests/             ← Integration tests
    └── AspireIntegrationTests.cs
```

---

## Step 1612: CI/CD Pipeline with Aspire

```yaml
# .github/workflows/aspire-deploy.yml
name: Build and Deploy Aspire App

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_PREFIX: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '9.x'

      - name: Restore
        run: dotnet restore

      - name: Build
        run: dotnet build --no-restore

      - name: Test (Aspire integration)
        run: dotnet test --no-build --verbosity normal
        env:
          ASPIRE_ALLOW_UNSECURED_TRANSPORT: true

  deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v4

      - name: Login to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push images
        run: |
          dotnet publish AspireApp.OrderApi --os linux --arch x64 \
            -p PublishProfile=DefaultContainer \
            -p ContainerRegistry=${{ env.REGISTRY }} \
            -p ContainerImageName=${{ env.IMAGE_PREFIX }}/order-api \
            -p ContainerImageTag=${{ github.sha }}

      - name: Deploy to Azure Container Apps
        uses: azure/container-apps-deploy-action@v2
        with:
          acrName: myregistry
          containerAppName: order-api
          resourceGroup: aspire-prod
          imageToDeploy: ${{ env.REGISTRY }}/${{ env.IMAGE_PREFIX }}/order-api:${{ github.sha }}
```

---

## Step 1613: .NET Aspire Production Checklist

```
.NET Aspire Production Readiness Checklist
═══════════════════════════════════════════════════════════

AppHost Design
✅ Use Azure-backed resources in production (not emulators)
✅ Separate AppHost configs per environment (dev/staging/prod)
✅ WithExternalHttpEndpoints() only for public-facing services
✅ WithReplicas(n) for services requiring HA
✅ Resource health dependencies (WithHealthCheck)

ServiceDefaults
✅ AddServiceDefaults() in every service project
✅ MapDefaultEndpoints() in every WebApplication
✅ Standard resilience handler on all outbound HTTP
✅ Service discovery on all inter-service HTTP clients
✅ Structured logging with correlation IDs

Observability
✅ OTLP endpoint configured (Aspire dashboard or external)
✅ All services emit traces, metrics, and logs
✅ Custom spans for business operations
✅ Health checks: /alive (liveness) /ready (readiness)
✅ Prometheus scrape annotations on k8s pods

Secrets
✅ Key Vault for all secrets (no secrets in env vars)
✅ Managed Identity for Azure resource access
✅ No passwords in connection strings (use Azure AD auth)

Deployment
✅ azd up for Azure Container Apps
✅ aspirate for Kubernetes
✅ Container image built via dotnet publish (not docker build)
✅ HPA configured with CPU + request rate metrics
✅ Rolling updates with 0-downtime health checks

Testing
✅ DistributedApplicationTestingBuilder for integration tests
✅ Real infrastructure via testcontainers (not mocks)
✅ Load tests against the full Aspire app
```

---

**Part 60 ครอบคลุม:**
- `.NET Aspire` AppHost — orchestrate DB, Redis, RabbitMQ, Kafka, Azure services
- Service discovery ด้วย `https+http://service-name` URI scheme
- `ServiceDefaults` — OTel + health checks + resilience ทุก service
- `DistributedApplicationTestingBuilder` — integration tests แบบ end-to-end
- Deploy ไป Azure Container Apps + Kubernetes
- OTel Collector → Prometheus + Loki + Jaeger pipeline
- CI/CD ด้วย GitHub Actions
