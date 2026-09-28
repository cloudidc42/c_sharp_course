# Part 69: Observability Deep Dive — OpenTelemetry, Tracing & Structured Logging (Steps 1733-1748)

## Steps 1733-1748: Production Observability for .NET Distributed Systems

Observability is the ability to understand what your system is doing from the outside. The three pillars are Traces (what happened), Metrics (how much / how fast), and Logs (what was said). This part covers the full .NET observability stack: OpenTelemetry SDK, distributed tracing with Jaeger/Grafana Tempo, structured logging with Serilog, and the correlation IDs that tie everything together.

---

## Step 1733: The Three Pillars of Observability

```
┌─────────────────────────────────────────────────────────────┐
│  TRACES — request lifecycle across services                  │
│  Span: PlaceOrder [3.2ms]                                    │
│    └─ Span: ValidateOrder [0.3ms]                            │
│    └─ Span: DB: INSERT Orders [1.1ms]                        │
│    └─ Span: Publish: OrderPlaced [0.8ms]                     │
│                                                               │
│  METRICS — aggregated numerical measurements                  │
│  http_requests_total{status="200"} 45234                     │
│  order_processing_duration_p99 0.342s                        │
│                                                               │
│  LOGS — timestamped text records                             │
│  2026-09-28T12:01:33Z INF Order placed OrderId=abc CustomerId=xyz│
└─────────────────────────────────────────────────────────────┘
                        │ correlated by
                        ▼
              TraceId + SpanId + CorrelationId
```

---

## Step 1734: OpenTelemetry SDK Setup

```xml
<PackageReference Include="OpenTelemetry.Extensions.Hosting" Version="1.9.0" />
<PackageReference Include="OpenTelemetry.Instrumentation.AspNetCore" Version="1.9.0" />
<PackageReference Include="OpenTelemetry.Instrumentation.Http" Version="1.9.0" />
<PackageReference Include="OpenTelemetry.Instrumentation.Runtime" Version="1.9.0" />
<PackageReference Include="OpenTelemetry.Instrumentation.EntityFrameworkCore" Version="1.0.0-beta.12" />
<PackageReference Include="OpenTelemetry.Exporter.Otlp" Version="1.9.0" />
<PackageReference Include="OpenTelemetry.Exporter.Prometheus.AspNetCore" Version="1.9.0-beta.2" />
<PackageReference Include="OpenTelemetry.Exporter.Console" Version="1.9.0" />
```

```csharp
using OpenTelemetry;
using OpenTelemetry.Resources;
using OpenTelemetry.Trace;
using OpenTelemetry.Metrics;
using OpenTelemetry.Logs;

var resourceBuilder = ResourceBuilder.CreateDefault()
    .AddService(
        serviceName: "order-api",
        serviceVersion: "1.2.3",
        serviceInstanceId: Environment.MachineName)
    .AddAttributes(new Dictionary<string, object>
    {
        ["deployment.environment"] = builder.Environment.EnvironmentName,
        ["host.name"] = Environment.MachineName,
    });

builder.Services.AddOpenTelemetry()
    .WithTracing(tracing =>
    {
        tracing
            .SetResourceBuilder(resourceBuilder)
            .AddAspNetCoreInstrumentation(opts =>
            {
                opts.RecordException = true;
                opts.Filter = ctx =>
                    !ctx.Request.Path.StartsWithSegments("/health") &&
                    !ctx.Request.Path.StartsWithSegments("/metrics");
            })
            .AddHttpClientInstrumentation(opts =>
            {
                opts.RecordException = true;
                opts.FilterHttpRequestMessage = req =>
                    !req.RequestUri?.Host.Contains("health") == true;
            })
            .AddEntityFrameworkCoreInstrumentation(opts =>
            {
                opts.SetDbStatementForText = true;        // include SQL text
                opts.SetDbStatementForStoredProcedure = true;
            })
            .AddSource("MassTransit")                     // messaging spans
            .AddSource("MyApp.*")                         // custom ActivitySources
            .AddOtlpExporter(opts =>
                opts.Endpoint = new Uri("http://otel-collector:4317"));

        if (builder.Environment.IsDevelopment())
            tracing.AddConsoleExporter();
    })
    .WithMetrics(metrics =>
    {
        metrics
            .SetResourceBuilder(resourceBuilder)
            .AddAspNetCoreInstrumentation()
            .AddHttpClientInstrumentation()
            .AddRuntimeInstrumentation()
            .AddMeter("MyApp.Orders")
            .AddMeter("MyApp.Payments")
            .AddOtlpExporter()
            .AddPrometheusExporter();
    });

// Logs: redirect ILogger to OpenTelemetry
builder.Logging.AddOpenTelemetry(logging =>
{
    logging.SetResourceBuilder(resourceBuilder);
    logging.AddOtlpExporter();
    logging.IncludeFormattedMessage = true;
    logging.IncludeScopes = true;
    logging.ParseStateValues = true;
});

app.MapPrometheusScrapingEndpoint("/metrics");
```

---

## Step 1735: Custom Spans with ActivitySource

```csharp
using System.Diagnostics;

// Define ActivitySource per logical component — best practice: one per assembly
public static class Telemetry
{
    public static readonly ActivitySource OrderSource =
        new("MyApp.Orders", "1.2.3");

    public static readonly ActivitySource PaymentSource =
        new("MyApp.Payments", "1.2.3");
}

public class OrderService
{
    public async Task<Order> PlaceOrderAsync(PlaceOrderCommand cmd, CancellationToken ct)
    {
        // Start a span
        using var activity = Telemetry.OrderSource.StartActivity(
            "PlaceOrder",
            ActivityKind.Internal);

        // Add attributes (key-value pairs on the span)
        activity?.SetTag("order.customer_id", cmd.CustomerId);
        activity?.SetTag("order.item_count", cmd.Items.Count);
        activity?.SetTag("order.source", cmd.Source);

        try
        {
            var order = await ValidateAndCreateAsync(cmd, ct);

            activity?.SetTag("order.id", order.Id.ToString());
            activity?.SetTag("order.total_amount", order.TotalAmount);
            activity?.SetStatus(ActivityStatusCode.Ok);

            return order;
        }
        catch (ValidationException ex)
        {
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            activity?.RecordException(ex);  // records exception as span event
            throw;
        }
    }

    private async Task<Order> ValidateAndCreateAsync(PlaceOrderCommand cmd, CancellationToken ct)
    {
        // Nested span
        using var validateActivity = Telemetry.OrderSource.StartActivity("ValidateOrder");
        validateActivity?.SetTag("order.item_count", cmd.Items.Count);

        foreach (var item in cmd.Items)
        {
            var inStock = await _inventory.CheckStockAsync(item.Sku, item.Quantity, ct);
            if (!inStock)
            {
                validateActivity?.AddEvent(new ActivityEvent("StockInsufficient",
                    tags: new ActivityTagsCollection
                    {
                        ["sku"] = item.Sku,
                        ["requested_quantity"] = item.Quantity
                    }));
                throw new ValidationException($"Insufficient stock for {item.Sku}");
            }
        }

        return Order.Create(cmd);
    }
}
```

---

## Step 1736: Distributed Tracing — Propagating Context

```csharp
// Context propagation is automatic for HTTP (via W3C TraceContext headers)
// For message queues, you need to propagate manually

// Publish with trace context (MassTransit does this automatically)
// For manual Kafka/RabbitMQ:

public class TracedMessagePublisher(IMessageBroker broker)
{
    public async Task PublishAsync<T>(T message, CancellationToken ct) where T : class
    {
        var activity = Activity.Current;

        // Inject the current trace context into message headers
        var headers = new Dictionary<string, string>();
        DistributedContextPropagator.Current.Inject(
            activity,
            headers,
            static (carrier, key, value) => ((Dictionary<string, string>)carrier!)[key] = value);

        await broker.PublishAsync(message, headers, ct);
    }
}

// Consumer: extract trace context from message headers
public class TracedMessageConsumer<T>(Func<T, CancellationToken, Task> handler)
{
    public async Task ConsumeAsync(T message, IDictionary<string, string> headers, CancellationToken ct)
    {
        // Extract propagated trace context
        var propagatedContext = DistributedContextPropagator.Current.ExtractBaggage(
            headers,
            static (carrier, key) =>
            {
                ((IDictionary<string, string>)carrier!).TryGetValue(key, out var value);
                return value;
            });

        // Start a span linked to the producer's trace
        using var activity = Telemetry.OrderSource.StartActivity(
            $"Consume {typeof(T).Name}",
            ActivityKind.Consumer,
            parentContext: default);  // propagation is via current context

        await handler(message, ct);
    }
}
```

---

## Step 1737: Structured Logging with Serilog

```xml
<PackageReference Include="Serilog.AspNetCore" Version="8.0.2" />
<PackageReference Include="Serilog.Sinks.Console" Version="6.0.0" />
<PackageReference Include="Serilog.Sinks.Seq" Version="8.0.0" />
<PackageReference Include="Serilog.Sinks.OpenTelemetry" Version="4.0.0" />
<PackageReference Include="Serilog.Enrichers.Environment" Version="3.0.0" />
<PackageReference Include="Serilog.Enrichers.Thread" Version="4.0.0" />
<PackageReference Include="Serilog.Enrichers.Span" Version="3.1.0" />
```

```csharp
using Serilog;
using Serilog.Events;

// Configure Serilog before anything else (bootstraps early)
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Information()
    .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
    .MinimumLevel.Override("Microsoft.EntityFrameworkCore.Database.Command", LogEventLevel.Warning)
    .MinimumLevel.Override("System.Net.Http", LogEventLevel.Warning)
    .Enrich.FromLogContext()            // from LogContext.PushProperty(...)
    .Enrich.WithEnvironmentName()
    .Enrich.WithMachineName()
    .Enrich.WithThreadId()
    .Enrich.WithSpan()                 // adds TraceId, SpanId from OpenTelemetry
    .WriteTo.Console(new Serilog.Formatting.Compact.CompactJsonFormatter())
    .WriteTo.Seq("http://seq:5341")    // structured log server
    .WriteTo.OpenTelemetry(opts =>
    {
        opts.Endpoint = "http://otel-collector:4317";
        opts.Protocol = OtlpProtocol.Grpc;
    })
    .CreateLogger();

builder.Host.UseSerilog();

// Request logging middleware
app.UseSerilogRequestLogging(opts =>
{
    opts.MessageTemplate = "HTTP {RequestMethod} {RequestPath} responded {StatusCode} in {Elapsed:0.0000} ms";
    opts.EnrichDiagnosticContext = (diagnosticContext, httpContext) =>
    {
        diagnosticContext.Set("RequestHost", httpContext.Request.Host.Value);
        diagnosticContext.Set("UserAgent", httpContext.Request.Headers["User-Agent"]);
        diagnosticContext.Set("UserId",
            httpContext.User.FindFirst("sub")?.Value ?? "anonymous");
    };
});
```

---

## Step 1738: Correlation IDs — Linking Logs, Traces, and Requests

```csharp
// Middleware to ensure every request has a correlation ID
public class CorrelationIdMiddleware(RequestDelegate next)
{
    private const string Header = "X-Correlation-ID";

    public async Task InvokeAsync(HttpContext context)
    {
        // Use incoming correlation ID or generate a new one
        var correlationId = context.Request.Headers[Header].FirstOrDefault()
                         ?? Activity.Current?.TraceId.ToString()
                         ?? Guid.NewGuid().ToString("N");

        context.Items["CorrelationId"] = correlationId;

        // Expose on response so clients can correlate
        context.Response.OnStarting(() =>
        {
            context.Response.Headers.TryAdd(Header, correlationId);
            return Task.CompletedTask;
        });

        // Push to Serilog context
        using (LogContext.PushProperty("CorrelationId", correlationId))
        // Push to Activity (OpenTelemetry)
        using (Activity.Current?.SetBaggage("correlation.id", correlationId) is not null
               ? (IDisposable)Disposable.Empty : Disposable.Empty)
        {
            await next(context);
        }
    }
}

app.UseMiddleware<CorrelationIdMiddleware>();
app.UseAuthentication();
app.UseAuthorization();
```

```csharp
// Use correlation ID in services
public class OrderService(ILogger<OrderService> logger, IHttpContextAccessor httpContext)
{
    public async Task PlaceOrderAsync(PlaceOrderCommand cmd, CancellationToken ct)
    {
        var correlationId = httpContext.HttpContext?.Items["CorrelationId"]?.ToString()
                         ?? Activity.Current?.TraceId.ToString();

        // Serilog enriches all logs in this scope with CorrelationId
        using var _ = LogContext.PushProperty("CorrelationId", correlationId);

        logger.LogInformation("Placing order for {CustomerId}", cmd.CustomerId);
        // Log output: { "CorrelationId": "abc123", "CustomerId": "CUST-001", ... }

        var order = await ProcessAsync(cmd, ct);

        logger.LogInformation("Order placed successfully {OrderId} {TotalAmount}",
            order.Id, order.TotalAmount);
    }
}
```

---

## Step 1739: Log Levels and Structured Properties

```csharp
public class PaymentService(ILogger<PaymentService> logger)
{
    public async Task<PaymentResult> ProcessAsync(PaymentRequest req, CancellationToken ct)
    {
        // Trace: highly detailed, only for diagnosing specific issues
        logger.LogTrace("Payment request details: {@PaymentRequest}", req);

        // Debug: useful in development, off in production
        logger.LogDebug("Processing payment for order {OrderId}", req.OrderId);

        // Information: normal operation milestones
        logger.LogInformation("Payment authorized. OrderId={OrderId} Amount={Amount} PaymentRef={Ref}",
            req.OrderId, req.Amount, result.Reference);

        // Warning: unusual but not breaking
        logger.LogWarning("Payment gateway slow. Duration={Duration}ms OrderId={OrderId}",
            duration.TotalMilliseconds, req.OrderId);

        // Error: operation failed, handled
        logger.LogError(ex, "Payment failed. OrderId={OrderId} ErrorCode={Code}",
            req.OrderId, ex.Code);

        // Critical: system is in a broken state
        logger.LogCritical(ex, "Payment infrastructure down. All payments failing.");
    }
}
```

```csharp
// High-performance logging with LoggerMessage source generator
public static partial class Log
{
    [LoggerMessage(
        EventId = 1001,
        Level = LogLevel.Information,
        Message = "Order placed. OrderId={OrderId} CustomerId={CustomerId} Total={TotalAmount}")]
    public static partial void OrderPlaced(
        ILogger logger,
        Guid orderId,
        string customerId,
        decimal totalAmount);

    [LoggerMessage(
        EventId = 1002,
        Level = LogLevel.Warning,
        Message = "Order payment retry {Attempt}/{MaxAttempts}. OrderId={OrderId}")]
    public static partial void PaymentRetry(
        ILogger logger,
        int attempt,
        int maxAttempts,
        Guid orderId);
}

// Usage — zero allocation, no boxing
Log.OrderPlaced(_logger, order.Id, order.CustomerId, order.TotalAmount);
Log.PaymentRetry(_logger, attempt, maxAttempts, orderId);
```

---

## Step 1740: OpenTelemetry Collector Configuration

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
    timeout: 1s
    send_batch_size: 1024
  memory_limiter:
    check_interval: 5s
    limit_mib: 512
  resource:
    attributes:
      - key: deployment.environment
        value: production
        action: upsert
  # Drop health check spans
  filter/drop_health:
    spans:
      exclude:
        match_type: strict
        attributes:
          - key: http.target
            value: "/health"

exporters:
  # Traces → Grafana Tempo
  otlp/tempo:
    endpoint: tempo:4317
    tls:
      insecure: true
  # Metrics → Prometheus
  prometheus:
    endpoint: 0.0.0.0:8889
    namespace: myapp
  # Logs → Loki
  loki:
    endpoint: http://loki:3100/loki/api/v1/push

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, filter/drop_health, batch]
      exporters: [otlp/tempo]
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

## Step 1741: Grafana Dashboard Configuration

```yaml
# docker-compose.yml — complete observability stack
version: '3.8'
services:
  order-api:
    image: myregistry.azurecr.io/order-api:latest
    environment:
      OTEL_EXPORTER_OTLP_ENDPOINT: http://otel-collector:4317
      OTEL_SERVICE_NAME: order-api
      OTEL_RESOURCE_ATTRIBUTES: "deployment.environment=development"

  otel-collector:
    image: otel/opentelemetry-collector-contrib:0.102.0
    volumes:
      - ./otel-collector-config.yaml:/etc/otel-collector-config.yaml
    command: --config=/etc/otel-collector-config.yaml
    ports:
      - "4317:4317"
      - "4318:4318"
      - "8889:8889"

  tempo:
    image: grafana/tempo:2.4.1
    command: ["-config.file=/etc/tempo.yaml"]
    volumes:
      - ./tempo.yaml:/etc/tempo.yaml
    ports:
      - "3200:3200"

  loki:
    image: grafana/loki:3.0.0
    command: -config.file=/etc/loki/local-config.yaml
    ports:
      - "3100:3100"

  prometheus:
    image: prom/prometheus:v2.51.0
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:10.4.0
    environment:
      GF_AUTH_ANONYMOUS_ENABLED: "true"
      GF_AUTH_ANONYMOUS_ORG_ROLE: Admin
    volumes:
      - ./grafana/datasources:/etc/grafana/provisioning/datasources
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards
    ports:
      - "3000:3000"

  seq:
    image: datalust/seq:2024.1
    environment:
      ACCEPT_EULA: "Y"
    ports:
      - "5341:5341"
      - "80:80"
```

```yaml
# grafana/datasources/datasources.yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    url: http://prometheus:9090
    isDefault: true
  - name: Tempo
    type: tempo
    url: http://tempo:3200
    jsonData:
      tracesToLogsV2:
        datasourceUid: loki
        filterByTraceID: true
        filterBySpanID: true
  - name: Loki
    type: loki
    url: http://loki:3100
    jsonData:
      derivedFields:
        - matcherRegex: '"TraceId":"([a-f0-9]{32})"'
          name: TraceID
          url: '$${__value.raw}'
          datasourceUid: Tempo
```

---

## Step 1742: Custom Metrics — RED Pattern

RED = Rate, Errors, Duration — the three metrics every service must expose.

```csharp
using System.Diagnostics.Metrics;

public class OrderMetrics : IDisposable
{
    private readonly Meter _meter;

    // Rate — requests per second
    private readonly Counter<long> _ordersPlaced;
    private readonly Counter<long> _ordersRejected;

    // Errors — error rate
    private readonly Counter<long> _paymentFailures;

    // Duration — latency percentiles
    private readonly Histogram<double> _orderDuration;
    private readonly Histogram<double> _paymentDuration;

    // Business metrics
    private readonly Histogram<double> _orderValue;
    private readonly UpDownCounter<int> _pendingOrders;

    public OrderMetrics(IMeterFactory factory)
    {
        _meter = factory.Create("MyApp.Orders");

        _ordersPlaced = _meter.CreateCounter<long>(
            "orders_placed_total",
            description: "Total orders placed");

        _ordersRejected = _meter.CreateCounter<long>(
            "orders_rejected_total",
            description: "Orders rejected (validation, stock, etc.)");

        _paymentFailures = _meter.CreateCounter<long>(
            "payment_failures_total",
            description: "Payment processing failures");

        _orderDuration = _meter.CreateHistogram<double>(
            "order_placement_duration_seconds",
            unit: "s",
            description: "Time to place an order");

        _paymentDuration = _meter.CreateHistogram<double>(
            "payment_duration_seconds",
            unit: "s");

        _orderValue = _meter.CreateHistogram<double>(
            "order_value_usd",
            unit: "USD",
            description: "Distribution of order values");

        _pendingOrders = _meter.CreateUpDownCounter<int>(
            "orders_pending",
            description: "Current count of pending orders");
    }

    public void RecordOrderPlaced(string tier, string region, decimal value)
    {
        _ordersPlaced.Add(1, new("customer.tier", tier), new("region", region));
        _orderValue.Record((double)value, new("region", region));
        _pendingOrders.Add(1, new("region", region));
    }

    public void RecordOrderCompleted(string tier, string region, double durationSeconds)
    {
        _orderDuration.Record(durationSeconds, new("tier", tier));
        _pendingOrders.Add(-1, new("region", region));
    }

    public void RecordOrderRejected(string reason)
        => _ordersRejected.Add(1, new("reason", reason));

    public void RecordPaymentFailure(string gateway, string errorCode)
        => _paymentFailures.Add(1, new("gateway", gateway), new("error_code", errorCode));

    public void Dispose() => _meter.Dispose();
}
```

---

## Step 1743: Alerting — SLO-Based Alerts

```yaml
# prometheus/rules/slo.yaml
groups:
  - name: order-api-slo
    rules:
      # SLO: 99.9% of requests succeed (error budget = 0.1%)
      - alert: ErrorBudgetBurning
        expr: |
          (
            rate(http_requests_total{service="order-api",status=~"5.."}[1h])
            / rate(http_requests_total{service="order-api"}[1h])
          ) > 0.001
        for: 5m
        labels:
          severity: critical
          slo: availability
        annotations:
          summary: "Error budget burning fast"
          description: "Error rate {{ $value | humanizePercentage }} exceeds SLO of 0.1%"

      # SLO: p99 latency < 500ms
      - alert: LatencySloBreached
        expr: |
          histogram_quantile(0.99,
            rate(order_placement_duration_seconds_bucket{service="order-api"}[5m])
          ) > 0.5
        for: 2m
        labels:
          severity: warning
          slo: latency
        annotations:
          summary: "p99 order placement latency > 500ms"
          description: "p99 latency is {{ $value | humanizeDuration }}"

      # Anomaly: sudden drop in order rate (possible outage)
      - alert: OrderRateDrop
        expr: |
          rate(orders_placed_total[5m])
          < 0.5 * avg_over_time(rate(orders_placed_total[5m])[1h:5m])
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Order rate dropped by >50% compared to 1h average"
```

---

## Step 1744: Exception Tracking with Sentry Integration

```csharp
// Install: Sentry.AspNetCore
builder.WebHost.UseSentry(opts =>
{
    opts.Dsn = builder.Configuration["Sentry:Dsn"];
    opts.Environment = builder.Environment.EnvironmentName;
    opts.Release = "order-api@1.2.3";
    opts.TracesSampleRate = 0.1;    // 10% of transactions traced in Sentry
    opts.ProfilesSampleRate = 0.01; // 1% profiled
    opts.SendDefaultPii = false;    // don't send PII
    opts.AttachStacktrace = true;

    // Filter sensitive data
    opts.BeforeSend = evt =>
    {
        // Remove request bodies that might contain sensitive data
        evt.Request.Data = "[filtered]";
        return evt;
    };
});

// Manually capture exceptions with context
try { /* ... */ }
catch (Exception ex)
{
    SentrySdk.CaptureException(ex, scope =>
    {
        scope.SetTag("order.id", orderId.ToString());
        scope.SetTag("customer.tier", customerTier);
        scope.SetExtra("order_items", cmd.Items.Count);
    });
    throw;
}
```

---

## Step 1745: Log Aggregation with Seq

```csharp
// Query Serilog output in Seq with CLEF (Compact Log Event Format)
// http://localhost:80/#/events

// Useful Seq queries:
// Find all errors for a specific order:
// @Level = 'Error' and OrderId = 'abc-123'

// Slow requests over 1 second:
// @Properties['Elapsed'] > 1000 and RequestPath like '/api/orders%'

// All events in a trace:
// TraceId = '0af7651916cd43dd8448eb211c80319c'

// High error rate in last 5 minutes:
// @Level in ['Error', 'Fatal'] | count() > 10
```

---

## Step 1746: Health Check Dashboard

```csharp
// Install: AspNetCore.HealthChecks.UI
builder.Services.AddHealthChecksUI(opts =>
{
    opts.SetEvaluationTimeInSeconds(30);
    opts.MaximumHistoryEntriesPerEndpoint(50);
    opts.SetApiMaxActiveRequests(1);
    opts.AddHealthCheckEndpoint("order-api", "/health");
    opts.AddHealthCheckEndpoint("payment-api", "http://payment-api/health");
    opts.AddHealthCheckEndpoint("inventory-api", "http://inventory-api/health");
})
.AddInMemoryStorage();

app.UseHealthChecksUI(opts =>
{
    opts.UIPath = "/health-ui";          // dashboard at /health-ui
    opts.ApiPath = "/health-ui-api";
});

// Webhook notifications on status change
builder.Services.AddHealthChecksUI(opts =>
{
    opts.AddWebhookNotification("slack",
        uri: "https://hooks.slack.com/services/...",
        payload: """
            {
              "text": "[[LIVENESS]] ⚠️ [[FAILURE]] - [[DESCRIPTIONS]]",
              "attachments": [{
                "color": "#FF0000",
                "fields": [{"title": "Details", "value": "[[DESCRIPTIONS]]"}]
              }]
            }
            """,
        restorePayload: """{"text": "[[LIVENESS]] ✅ Healthy"}""");
});
```

---

## Step 1747: Distributed Tracing — Baggage Propagation

```csharp
// Baggage: key-value pairs propagated through the entire trace
// Use for: tenant ID, user tier, experiment flags

// Set baggage at the entry point (e.g., in middleware)
public class TenantBaggageMiddleware(RequestDelegate next)
{
    public async Task InvokeAsync(HttpContext context)
    {
        var tenantId = context.User.FindFirst("tenant_id")?.Value;
        if (tenantId is not null)
        {
            // Propagates to all downstream services via W3C baggage header
            Activity.Current?.SetBaggage("tenant.id", tenantId);
            Activity.Current?.SetBaggage("customer.tier",
                context.User.FindFirst("tier")?.Value ?? "standard");
        }
        await next(context);
    }
}

// Read baggage in any downstream service (same process or remote)
public class OrderRepository
{
    public async Task<List<Order>> GetByTenantAsync(CancellationToken ct)
    {
        // Retrieve from propagated baggage
        var tenantId = Activity.Current?.GetBaggageItem("tenant.id")
                    ?? throw new InvalidOperationException("No tenant context");

        return await _db.Orders
            .Where(o => o.TenantId == tenantId)
            .ToListAsync(ct);
    }
}
```

---

## Step 1748: Observability Checklist

```csharp
// Complete observability setup — summary checklist

// 1. RESOURCES: tag every service with name, version, environment, host
var resource = ResourceBuilder.CreateDefault()
    .AddService("order-api", "1.2.3")
    .AddAttributes(new Dictionary<string, object> {
        ["deployment.environment"] = "production",
        ["host.name"] = Environment.MachineName });

// 2. TRACES: instrument every significant operation
using var span = ActivitySource.StartActivity("ProcessOrder", ActivityKind.Internal);
span?.SetTag("order.id", orderId);

// 3. METRICS: expose RED metrics (Rate, Errors, Duration)
_requestCounter.Add(1, new("status", "success"));
_durationHistogram.Record(sw.Elapsed.TotalSeconds);

// 4. LOGS: structured, enriched, correlated with trace ID
logger.LogInformation("Order processed {OrderId} in {Duration:F2}s", orderId, elapsed);

// 5. CORRELATION: TraceId + SpanId in every log entry
// Serilog.Enrichers.Span does this automatically

// 6. HEALTH: liveness + readiness probes with meaningful checks
app.MapHealthChecks("/health/ready", new() { Predicate = c => c.Tags.Contains("ready") });
app.MapHealthChecks("/health/live", new() { Predicate = _ => false });

// 7. ALERTING: SLO-based, not just "service is up"
// Error budget, latency p99, business metric anomalies

// 8. DASHBOARDS: RED per service + business metrics
// Grafana → Prometheus + Tempo (linked traces in logs)
```

| Signal | Tool | Retention |
|---|---|---|
| Traces | Grafana Tempo | 7–30 days |
| Metrics | Prometheus + Thanos | 90 days |
| Logs | Grafana Loki / Seq | 30 days |
| Errors | Sentry | 90 days |
| Profiling | Pyroscope | 7 days |

### What Next?

- **Part 70**: Resilience Patterns — Polly v8, Circuit Breaker, Bulkhead, Hedging, Timeout, Fallback
- **Part 71**: Multi-Tenancy Architecture — Row-level security, schema-per-tenant, API key tenancy
- **Part 72**: Domain-Driven Design Advanced — Aggregates, Bounded Contexts, Anti-Corruption Layer
