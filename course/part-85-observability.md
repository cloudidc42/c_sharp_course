# Part 85: Advanced Observability — OpenTelemetry, Grafana & Alerting

## Steps 1989–2004

---

## Step 1989: The Three Pillars of Observability

```
Logs     — what happened (structured events with context)
Metrics  — how the system behaves over time (counters, histograms, gauges)
Traces   — why it happened (distributed request flow across services)

OpenTelemetry unifies all three with a vendor-neutral SDK.
```

```bash
dotnet add package OpenTelemetry.Extensions.Hosting
dotnet add package OpenTelemetry.Instrumentation.AspNetCore
dotnet add package OpenTelemetry.Instrumentation.Http
dotnet add package OpenTelemetry.Instrumentation.Runtime
dotnet add package OpenTelemetry.Instrumentation.Process
dotnet add package OpenTelemetry.Exporter.Otlp
dotnet add package OpenTelemetry.Exporter.Prometheus.AspNetCore
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Sinks.OpenTelemetry
dotnet add package Serilog.Enrichers.Span
```

---

## Step 1990: Structured Logging with Serilog

```csharp
// Program.cs — configure Serilog
using Serilog;
using Serilog.Events;

Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Information()
    .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
    .MinimumLevel.Override("Microsoft.Hosting.Lifetime", LogEventLevel.Information)
    .Enrich.FromLogContext()
    .Enrich.WithMachineName()
    .Enrich.WithEnvironmentName()
    .Enrich.WithSpan() // adds trace_id, span_id from OTel activity
    .CreateBootstrapLogger();

try
{
    var builder = WebApplication.CreateBuilder(args);

    builder.Host.UseSerilog((ctx, services, config) => config
        .ReadFrom.Configuration(ctx.Configuration)
        .ReadFrom.Services(services)
        .Enrich.FromLogContext()
        .Enrich.WithMachineName()
        .Enrich.WithEnvironmentName()
        .Enrich.WithSpan()
        .WriteTo.Console(new JsonFormatter())
        .WriteTo.OpenTelemetry(opts =>
        {
            opts.Endpoint = ctx.Configuration["OTEL_EXPORTER_OTLP_ENDPOINT"]
                ?? "http://localhost:4317";
            opts.Protocol = OtlpProtocol.Grpc;
            opts.ResourceAttributes = new Dictionary<string, object>
            {
                ["service.name"] = "orders-api",
                ["service.version"] = typeof(Program).Assembly.GetName().Version?.ToString() ?? "1.0.0"
            };
        }));

    // ... app setup
    app.Run();
}
catch (Exception ex)
{
    Log.Fatal(ex, "Application startup failed");
    return 1;
}
finally
{
    Log.CloseAndFlush();
}

return 0;
```

```json
// appsettings.json — Serilog configuration
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning",
        "Microsoft.EntityFrameworkCore": "Warning",
        "System": "Warning"
      }
    },
    "Destructure": [
      {
        "Name": "ToMaximumDepth",
        "Args": { "maximumDestructuringDepth": 4 }
      },
      {
        "Name": "ToMaximumStringLength",
        "Args": { "maximumStringLength": 1000 }
      }
    ],
    "Properties": {
      "Application": "OrdersApi"
    }
  }
}
```

---

## Step 1991: Log Enrichment & Context

```csharp
// Add per-request context
app.Use(async (context, next) =>
{
    var correlationId = context.Request.Headers["X-Correlation-ID"]
        .FirstOrDefault() ?? Guid.NewGuid().ToString();

    using (LogContext.PushProperty("CorrelationId", correlationId))
    using (LogContext.PushProperty("UserId", context.User?.FindFirst("sub")?.Value))
    using (LogContext.PushProperty("RequestPath", context.Request.Path))
    {
        context.Response.Headers["X-Correlation-ID"] = correlationId;
        await next();
    }
});

// Structured logging — use message templates, not string interpolation
// WRONG — prevents log aggregation by pattern
_logger.LogInformation($"Order {orderId} placed by customer {customerId}");

// RIGHT — structured
_logger.LogInformation(
    "Order {OrderId} placed by customer {CustomerId} with {ItemCount} items totaling {Total:F2}",
    order.Id, order.CustomerId, order.Items.Count, order.TotalAmount);

// Destructure complex objects
_logger.LogInformation(
    "Processing {@Order}", // @ = destructure the object
    new { order.Id, order.Status, ItemCount = order.Items.Count });
```

```csharp
// Sensitive data scrubbing
public class SensitiveDataDestructuringPolicy : IDestructuringPolicy
{
    private static readonly HashSet<string> SensitiveFields = new(StringComparer.OrdinalIgnoreCase)
    {
        "password", "token", "secret", "creditcard", "cvv", "ssn"
    };

    public bool TryDestructure(object value, ILogEventPropertyValueFactory propertyValueFactory,
        out LogEventPropertyValue? result)
    {
        // Intercept specific types and redact fields
        if (value is not IDictionary<string, object> dict)
        {
            result = null;
            return false;
        }

        var sanitized = dict.ToDictionary(
            kv => kv.Key,
            kv => SensitiveFields.Any(f => kv.Key.Contains(f, StringComparison.OrdinalIgnoreCase))
                ? "[REDACTED]" : kv.Value.ToString() ?? "");

        result = propertyValueFactory.CreatePropertyValue(sanitized);
        return true;
    }
}
```

---

## Step 1992: OpenTelemetry Tracing

```csharp
// Shared.Observability/ObservabilityExtensions.cs
using OpenTelemetry.Resources;
using OpenTelemetry.Trace;

public static class ObservabilityExtensions
{
    public static IServiceCollection AddObservability(
        this IServiceCollection services,
        IConfiguration config,
        string serviceName,
        Action<TracerProviderBuilder>? configureTracing = null,
        Action<MeterProviderBuilder>? configureMetrics = null)
    {
        var serviceVersion = typeof(ObservabilityExtensions).Assembly
            .GetName().Version?.ToString() ?? "1.0.0";

        services.AddOpenTelemetry()
            .ConfigureResource(r => r
                .AddService(serviceName, serviceVersion: serviceVersion)
                .AddTelemetrySdk()
                .AddAttributes(new Dictionary<string, object>
                {
                    ["deployment.environment"] = config["ASPNETCORE_ENVIRONMENT"] ?? "production",
                    ["host.name"] = Environment.MachineName
                }))
            .WithTracing(tracing =>
            {
                tracing
                    .AddAspNetCoreInstrumentation(opts =>
                    {
                        opts.RecordException = true;
                        opts.EnrichWithHttpRequest = (activity, request) =>
                        {
                            var correlationId = request.Headers["X-Correlation-ID"].FirstOrDefault();
                            if (correlationId is not null)
                                activity.SetTag("correlation.id", correlationId);
                        };
                        opts.Filter = ctx =>
                            !ctx.Request.Path.StartsWithSegments("/health") &&
                            !ctx.Request.Path.StartsWithSegments("/metrics");
                    })
                    .AddHttpClientInstrumentation(opts =>
                    {
                        opts.RecordException = true;
                        opts.EnrichWithHttpResponseMessage = (activity, response) =>
                        {
                            activity.SetTag("http.response_content_type",
                                response.Content.Headers.ContentType?.ToString());
                        };
                    })
                    .AddEntityFrameworkCoreInstrumentation(opts =>
                    {
                        opts.SetDbStatementForText = config["ASPNETCORE_ENVIRONMENT"] == "Development";
                    })
                    .AddSource("MassTransit");

                configureTracing?.Invoke(tracing);

                var otlpEndpoint = config["OTEL_EXPORTER_OTLP_ENDPOINT"];
                if (!string.IsNullOrEmpty(otlpEndpoint))
                    tracing.AddOtlpExporter(o => o.Endpoint = new Uri(otlpEndpoint));
                else
                    tracing.AddConsoleExporter();
            })
            .WithMetrics(metrics =>
            {
                metrics
                    .AddAspNetCoreInstrumentation()
                    .AddHttpClientInstrumentation()
                    .AddRuntimeInstrumentation()
                    .AddProcessInstrumentation();

                configureMetrics?.Invoke(metrics);

                var otlpEndpoint = config["OTEL_EXPORTER_OTLP_ENDPOINT"];
                if (!string.IsNullOrEmpty(otlpEndpoint))
                    metrics.AddOtlpExporter(o => o.Endpoint = new Uri(otlpEndpoint));

                metrics.AddPrometheusExporter();
            });

        return services;
    }
}
```

---

## Step 1993: Custom Activity Sources

```csharp
// Application/Tracing/AppActivitySource.cs
using System.Diagnostics;

namespace Orders.Application.Tracing;

public static class AppActivitySource
{
    public static readonly ActivitySource Orders = new("Orders.Application");
    public static readonly ActivitySource Infrastructure = new("Orders.Infrastructure");

    public static Activity? StartPlacingOrder(Guid customerId, int itemCount)
    {
        var activity = Orders.StartActivity("PlaceOrder");
        activity?.SetTag("order.customer_id", customerId.ToString());
        activity?.SetTag("order.item_count", itemCount);
        return activity;
    }

    public static Activity? StartPaymentProcessing(Guid orderId, decimal amount, string currency)
    {
        var activity = Orders.StartActivity("ProcessPayment");
        activity?.SetTag("payment.order_id", orderId.ToString());
        activity?.SetTag("payment.amount", amount.ToString("F2"));
        activity?.SetTag("payment.currency", currency);
        return activity;
    }
}
```

```csharp
// Usage in command handler
public async Task<PlaceOrderResult> Handle(PlaceOrderCommand cmd, CancellationToken ct)
{
    using var activity = AppActivitySource.StartPlacingOrder(
        cmd.CustomerId, cmd.Items.Count);

    try
    {
        var order = Order.Create(cmd.CustomerId, cmd.ShippingAddress);

        foreach (var item in cmd.Items)
            order.AddItem(item.ProductId, item.Sku, item.Quantity, item.UnitPrice);

        order.Confirm();

        await _orders.SaveAsync(order, ct);

        activity?.SetTag("order.id", order.Id.ToString());
        activity?.SetTag("order.total", order.TotalAmount.ToString("F2"));
        activity?.SetStatus(ActivityStatusCode.Ok);

        return new PlaceOrderResult(order.Id, order.Status.ToString());
    }
    catch (Exception ex)
    {
        activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
        activity?.RecordException(ex);
        throw;
    }
}
```

---

## Step 1994: Custom Metrics with System.Diagnostics.Metrics

```csharp
// Application/Metrics/OrderMetrics.cs
using System.Diagnostics.Metrics;

namespace Orders.Application.Metrics;

public sealed class OrderMetrics : IDisposable
{
    private readonly Meter _meter;
    private readonly Counter<long> _ordersPlaced;
    private readonly Counter<long> _ordersFailed;
    private readonly Histogram<double> _orderValue;
    private readonly Histogram<double> _processingDuration;
    private readonly ObservableGauge<int> _activeOrders;
    private int _activeOrderCount;

    public OrderMetrics(IMeterFactory meterFactory)
    {
        _meter = meterFactory.Create("Orders.Application");

        _ordersPlaced = _meter.CreateCounter<long>(
            "orders.placed",
            unit: "{orders}",
            description: "Number of orders placed");

        _ordersFailed = _meter.CreateCounter<long>(
            "orders.failed",
            unit: "{orders}",
            description: "Number of orders that failed to place");

        _orderValue = _meter.CreateHistogram<double>(
            "orders.value",
            unit: "USD",
            description: "Distribution of order values");

        _processingDuration = _meter.CreateHistogram<double>(
            "orders.processing_duration",
            unit: "s",
            description: "Time to process an order");

        _activeOrders = _meter.CreateObservableGauge(
            "orders.active",
            () => _activeOrderCount,
            unit: "{orders}",
            description: "Currently processing orders");
    }

    public void RecordOrderPlaced(decimal amount, string customerTier, string channel)
    {
        var tags = new TagList
        {
            { "customer.tier", customerTier },
            { "order.channel", channel }
        };

        _ordersPlaced.Add(1, tags);
        _orderValue.Record((double)amount, tags);
        Interlocked.Increment(ref _activeOrderCount);
    }

    public void RecordOrderFailed(string reason)
    {
        _ordersFailed.Add(1, new TagList { { "failure.reason", reason } });
    }

    public IDisposable TrackProcessingDuration()
    {
        return new DurationTracker(_processingDuration,
            () => Interlocked.Decrement(ref _activeOrderCount));
    }

    public void Dispose() => _meter.Dispose();

    private sealed class DurationTracker : IDisposable
    {
        private readonly Histogram<double> _histogram;
        private readonly Action _onDispose;
        private readonly long _startedAt = Stopwatch.GetTimestamp();

        public DurationTracker(Histogram<double> histogram, Action onDispose)
        {
            _histogram = histogram;
            _onDispose = onDispose;
        }

        public void Dispose()
        {
            var duration = Stopwatch.GetElapsedTime(_startedAt).TotalSeconds;
            _histogram.Record(duration);
            _onDispose();
        }
    }
}
```

```csharp
// Registration
builder.Services.AddMetrics();
builder.Services.AddSingleton<OrderMetrics>();

// Usage
public class PlaceOrderCommandHandler : IRequestHandler<PlaceOrderCommand, PlaceOrderResult>
{
    private readonly OrderMetrics _metrics;

    public async Task<PlaceOrderResult> Handle(PlaceOrderCommand cmd, CancellationToken ct)
    {
        using var _ = _metrics.TrackProcessingDuration();

        try
        {
            var order = Order.Create(cmd.CustomerId, cmd.ShippingAddress);
            // ...
            _metrics.RecordOrderPlaced(order.TotalAmount, "standard", "web");
            return new PlaceOrderResult(order.Id, order.Status.ToString());
        }
        catch (Exception ex)
        {
            _metrics.RecordOrderFailed(ex.GetType().Name);
            throw;
        }
    }
}
```

---

## Step 1995: OpenTelemetry Collector Configuration

```yaml
# otel-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 1s
    send_batch_size: 1024
  memory_limiter:
    check_interval: 1s
    limit_mib: 1024
  resourcedetection:
    detectors: [env, system]
    timeout: 5s
  attributes:
    actions:
      - key: db.statement  # remove in production for security
        action: delete
        value: ""

exporters:
  jaeger:
    endpoint: jaeger:14250
    tls:
      insecure: true

  prometheus:
    endpoint: 0.0.0.0:8889

  loki:
    endpoint: http://loki:3100/loki/api/v1/push
    tls:
      insecure: true

  debug:
    verbosity: detailed

extensions:
  health_check:
    endpoint: 0.0.0.0:13133
  zpages:
    endpoint: 0.0.0.0:55679

service:
  extensions: [health_check, zpages]
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch, resourcedetection]
      exporters: [jaeger]
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [prometheus]
    logs:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [loki]
```

---

## Step 1996: Grafana Dashboard as Code

```json
// grafana/dashboards/orders-api.json (excerpt)
{
  "title": "Orders API",
  "uid": "orders-api",
  "panels": [
    {
      "title": "Request Rate",
      "type": "stat",
      "targets": [
        {
          "expr": "sum(rate(http_server_duration_milliseconds_count{service_name='orders-api'}[5m]))",
          "legendFormat": "req/s"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "reqps",
          "thresholds": {
            "steps": [
              { "color": "green", "value": 0 },
              { "color": "yellow", "value": 500 },
              { "color": "red", "value": 1000 }
            ]
          }
        }
      }
    },
    {
      "title": "P99 Latency",
      "type": "timeseries",
      "targets": [
        {
          "expr": "histogram_quantile(0.99, sum(rate(http_server_duration_milliseconds_bucket{service_name='orders-api'}[5m])) by (le, http_route))",
          "legendFormat": "{{ http_route }}"
        }
      ]
    },
    {
      "title": "Error Rate",
      "type": "timeseries",
      "targets": [
        {
          "expr": "sum(rate(http_server_duration_milliseconds_count{service_name='orders-api',http_status_code=~'5..'}[5m])) / sum(rate(http_server_duration_milliseconds_count{service_name='orders-api'}[5m]))",
          "legendFormat": "error rate"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "percentunit",
          "thresholds": {
            "steps": [
              { "color": "green", "value": 0 },
              { "color": "red", "value": 0.01 }
            ]
          }
        }
      }
    },
    {
      "title": "Orders Per Minute",
      "type": "stat",
      "targets": [
        {
          "expr": "sum(rate(orders_placed_total[1m])) * 60"
        }
      ]
    }
  ]
}
```

---

## Step 1997: Prometheus Alerting Rules

```yaml
# prometheus/rules/orders-api.yaml
groups:
  - name: orders-api
    interval: 30s
    rules:
      # High error rate
      - alert: HighErrorRate
        expr: |
          sum(rate(http_server_duration_milliseconds_count{
            service_name="orders-api",
            http_status_code=~"5.."
          }[5m])) /
          sum(rate(http_server_duration_milliseconds_count{
            service_name="orders-api"
          }[5m])) > 0.05
        for: 2m
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "Orders API error rate > 5%"
          description: "Error rate is {{ $value | humanizePercentage }} for the last 2 minutes"
          runbook: "https://wiki.example.com/runbooks/orders-api-errors"

      # High latency
      - alert: HighLatencyP99
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_server_duration_milliseconds_bucket{
              service_name="orders-api"
            }[5m])) by (le)
          ) > 2000
        for: 5m
        labels:
          severity: warning
          team: backend
        annotations:
          summary: "Orders API P99 latency > 2s"
          description: "P99 latency is {{ $value | humanizeDuration }}ms"

      # Pod count low
      - alert: LowReplicaCount
        expr: |
          kube_deployment_status_replicas_available{
            deployment="orders-api",
            namespace="production"
          } < 2
        for: 1m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "Orders API has fewer than 2 available replicas"

      # Memory pressure
      - alert: HighMemoryUsage
        expr: |
          container_memory_usage_bytes{
            container="orders-api",
            namespace="production"
          } /
          container_spec_memory_limit_bytes{
            container="orders-api",
            namespace="production"
          } > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Orders API memory usage > 85%"

      # GC pressure — dotnet specific
      - alert: HighGCPressure
        expr: |
          rate(process_runtime_dotnet_gc_collections_count_total{
            service_name="orders-api",
            generation="gen2"
          }[5m]) > 1
        for: 5m
        labels:
          severity: warning
          team: backend
        annotations:
          summary: "Gen2 GC rate > 1/min — possible memory leak"
```

---

## Step 1998: SLI/SLO Definitions

```yaml
# SLI — what we measure
# SLO — the target we commit to
# Error budget — 1 - SLO availability

# orders-api SLOs
# Availability SLO: 99.9% over 30 days (43.2 min downtime budget)
# Latency SLO: 95% of requests < 500ms, 99% < 2s

# Prometheus SLI recording rules
groups:
  - name: orders-api-sli
    interval: 30s
    rules:
      # Success rate over 5m (for burn rate calculations)
      - record: job:orders_api_requests:success_rate5m
        expr: |
          sum(rate(http_server_duration_milliseconds_count{
            service_name="orders-api",
            http_status_code!~"5.."
          }[5m])) /
          sum(rate(http_server_duration_milliseconds_count{
            service_name="orders-api"
          }[5m]))

      # Latency SLI — fraction meeting < 500ms target
      - record: job:orders_api_latency:sli5m
        expr: |
          sum(rate(http_server_duration_milliseconds_bucket{
            service_name="orders-api",
            le="500"
          }[5m])) /
          sum(rate(http_server_duration_milliseconds_count{
            service_name="orders-api"
          }[5m]))

  # Multi-window burn rate alerts (Google SRE Workbook)
  - name: orders-api-slo-alerts
    rules:
      - alert: OrdersApiErrorBudgetBurnFast
        expr: |
          (
            job:orders_api_requests:success_rate5m < 0.999 * 0.86
          ) and (
            job:orders_api_requests:success_rate5m < 0.999 * 0.86
          )
        labels:
          severity: critical
          burn_rate: "14x"
        annotations:
          summary: "Orders API burning error budget at 14x rate"
          description: "At this rate, 30-day error budget exhausted in ~2 hours"
```

---

## Step 1999: Distributed Tracing Across Services

```csharp
// Propagation — automatic with OTel SDK
// W3C TraceContext is the default propagator

// Headers automatically injected by HttpClient instrumentation:
// traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
// tracestate: (vendor-specific)

// Manual context propagation if needed
public class OrdersApiClient
{
    private readonly HttpClient _http;

    public async Task<OrderDto?> GetOrderAsync(Guid id, CancellationToken ct = default)
    {
        // Activity.Current propagated automatically by HttpClient instrumentation
        using var activity = AppActivitySource.Orders.StartActivity("GetOrder");
        activity?.SetTag("order.id", id.ToString());

        return await _http.GetFromJsonAsync<OrderDto>($"/orders/{id}", ct);
    }
}

// Baggage — key-value pairs propagated across the entire trace
Activity.Current?.SetBaggage("tenant.id", currentTenant.Id);
// Downstream services can read:
var tenantId = Activity.Current?.GetBaggageItem("tenant.id");
```

---

## Step 2000: Step 2000 — Course Milestone 🎯

This is Step 2000. The course has covered:

```
Steps 1–100:    C# Fundamentals (variables, control flow, methods, OOP)
Steps 101–200:  .NET Core, ASP.NET, EF Core basics
Steps 201–300:  Web API design, authentication, authorization
Steps 301–400:  Testing, SOLID principles, design patterns
Steps 401–500:  Advanced C# (LINQ, async/await, reflection, generics)
Steps 501–600:  Architecture patterns, DDD basics
Steps 601–700:  Messaging, caching, resilience
Steps 701–800:  Cloud-native, containers, microservices
Steps 801–900:  Advanced patterns (Event Sourcing, CQRS, Sagas)
Steps 901–1000: Performance, security, observability
Steps 1001–1200: World-class production patterns
Steps 1201–1400: DevOps, Kubernetes, CI/CD
Steps 1401–1600: Advanced topics (GraphQL, SignalR, CLI tools)
Steps 1601–1800: Expert-level (Multi-tenancy, DDD Advanced, Testing)
Steps 1801–2000: World-class complete curriculum
```

---

## Step 2001: Log-Based Alerting with Loki

```yaml
# loki-rules.yaml — alert on log patterns
groups:
  - name: orders-api-logs
    rules:
      - alert: ExceptionSpike
        expr: |
          sum(count_over_time({service_name="orders-api", level="Error"}[5m])) > 50
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "More than 50 errors in 5 minutes in Orders API"

      - alert: DatabaseConnectionFailure
        expr: |
          count_over_time({
            service_name="orders-api"
          } |= "Failed to connect" |= "Postgres"[5m]) > 5
        for: 30s
        labels:
          severity: critical
```

```csharp
// Loki log query examples
// Find all errors for a specific order
{service_name="orders-api"} |= "OrderId" | json | OrderId="550e8400-e29b-41d4-a716-446655440000" | level = "Error"

// Slow requests
{service_name="orders-api"} | json | ElapsedMs > 1000 | line_format "{{.RequestPath}} took {{.ElapsedMs}}ms"

// Correlation ID trace
{service_name=~"orders-api|products-api|payment-api"} | json | CorrelationId="abc123"
```

---

## Step 2002: OpenTelemetry Demo Stack (docker-compose)

```yaml
# docker-compose.observability.yml
version: '3.8'

services:
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    volumes:
      - ./otel-config.yaml:/etc/otelcol-contrib/config.yaml
    ports:
      - "4317:4317"
      - "4318:4318"
      - "8889:8889"  # Prometheus metrics
    depends_on:
      - jaeger
      - prometheus
      - loki

  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"  # Jaeger UI
      - "14250:14250"  # gRPC collector
    environment:
      COLLECTOR_OTLP_ENABLED: "true"

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus/rules:/etc/prometheus/rules
    ports:
      - "9090:9090"
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--rule-files=/etc/prometheus/rules/*.yaml'
      - '--web.enable-remote-write-receiver'

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
      GF_DASHBOARDS_DEFAULT_HOME_DASHBOARD_PATH: /var/lib/grafana/dashboards/orders-api.json
    volumes:
      - ./grafana/dashboards:/var/lib/grafana/dashboards
      - ./grafana/provisioning:/etc/grafana/provisioning
    depends_on:
      - prometheus
      - jaeger
      - loki

  loki:
    image: grafana/loki:latest
    ports:
      - "3100:3100"
    volumes:
      - ./loki-config.yaml:/etc/loki/local-config.yaml

  alertmanager:
    image: prom/alertmanager:latest
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml
    ports:
      - "9093:9093"
```

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m
  slack_api_url: 'YOUR_SLACK_WEBHOOK_URL'

route:
  group_by: ['alertname', 'severity']
  group_wait: 10s
  group_interval: 10m
  repeat_interval: 12h
  receiver: 'slack-notifications'
  routes:
    - match:
        severity: critical
      receiver: 'pagerduty'
      continue: true

receivers:
  - name: 'slack-notifications'
    slack_configs:
      - channel: '#alerts-backend'
        title: '{{ template "slack.default.title" . }}'
        text: '{{ template "slack.default.text" . }}'
        send_resolved: true

  - name: 'pagerduty'
    pagerduty_configs:
      - service_key: 'YOUR_PAGERDUTY_KEY'
        description: '{{ .GroupLabels.alertname }}'
```

---

## Step 2003: Health Check Dashboard

```csharp
// Health check UI
builder.Services
    .AddHealthChecks()
    .AddNpgSql(config.GetConnectionString("Postgres")!, name: "postgres")
    .AddRedis(config.GetConnectionString("Redis")!, name: "redis")
    .AddUrlGroup(new Uri(config["Services:Products:HttpUrl"] + "/health"), "products-api");

builder.Services
    .AddHealthChecksUI(opts =>
    {
        opts.SetEvaluationTimeInSeconds(10);
        opts.MaximumHistoryEntriesPerEndpoint(50);
        opts.AddHealthCheckEndpoint("Orders API", "/health");
    })
    .AddInMemoryStorage();

// Expose endpoints
app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

app.MapHealthChecksUI(opts => opts.UIPath = "/health-ui");
```

---

## Step 2004: Complete Observability Registration

```csharp
// Program.cs — full observability stack
using Serilog;
using Shared.Observability;

// Bootstrap logger for startup errors
Log.Logger = new LoggerConfiguration()
    .WriteTo.Console()
    .CreateBootstrapLogger();

var builder = WebApplication.CreateBuilder(args);

// Serilog
builder.Host.UseSerilog((ctx, services, cfg) => cfg
    .ReadFrom.Configuration(ctx.Configuration)
    .ReadFrom.Services(services)
    .Enrich.FromLogContext()
    .Enrich.WithSpan()
    .WriteTo.Console(new JsonFormatter())
    .WriteTo.OpenTelemetry(opts =>
    {
        opts.Endpoint = ctx.Configuration["OTEL_EXPORTER_OTLP_ENDPOINT"]!;
        opts.Protocol = OtlpProtocol.Grpc;
    }));

// OpenTelemetry — traces + metrics
builder.Services.AddObservability(
    builder.Configuration,
    "orders-api",
    configureTracing: tracing =>
    {
        tracing.AddSource("Orders.Application", "Orders.Infrastructure");
    },
    configureMetrics: metrics =>
    {
        metrics.AddMeter("Orders.Application");
    });

// Custom metrics
builder.Services.AddSingleton<OrderMetrics>();

// Health checks + UI
builder.Services.AddHealthChecks()
    .AddNpgSql(builder.Configuration.GetConnectionString("Postgres")!, tags: ["ready"])
    .AddRedis(builder.Configuration.GetConnectionString("Redis")!, tags: ["ready"])
    .AddCheck("self", () => HealthCheckResult.Healthy(), tags: ["live"]);

builder.Services.AddHealthChecksUI().AddInMemoryStorage();

var app = builder.Build();

// Request logging
app.UseSerilogRequestLogging(opts =>
{
    opts.MessageTemplate = "HTTP {RequestMethod} {RequestPath} responded {StatusCode} in {Elapsed:0.0000} ms";
    opts.EnrichDiagnosticContext = (diagnosticContext, httpContext) =>
    {
        diagnosticContext.Set("RequestHost", httpContext.Request.Host.Value);
        diagnosticContext.Set("UserAgent", httpContext.Request.Headers["User-Agent"].ToString());
    };
});

app.MapMetrics("/metrics");
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = c => c.Tags.Contains("live")
});
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = c => c.Tags.Contains("ready"),
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});
app.MapHealthChecksUI(o => o.UIPath = "/health-ui");

app.Run();
```

---

## Summary

| Pillar | Tool | Purpose |
|--------|------|---------|
| Logs | Serilog + Loki | Structured, searchable event log |
| Traces | OpenTelemetry + Jaeger | Cross-service request flows |
| Metrics | OTel + Prometheus | Time-series performance data |
| Dashboards | Grafana | Visualization + alerting |
| Alerts | Alertmanager | PagerDuty / Slack routing |
| Health checks | ASP.NET Core + UI | K8s probes + dashboard |
| SLOs | Prometheus recording rules | Error budget tracking |
| Correlation | `X-Correlation-ID` + baggage | Log-trace-metric correlation |

**Next**: Part 86 — Advanced EF Core: Interceptors, Query Filters & Bulk Operations
