# Part 54: Observability & OpenTelemetry
## Steps 1511-1540: Distributed Tracing, Metrics, Logging — The Three Pillars

---

## Step 1511: OpenTelemetry Setup in .NET 9

```csharp
// Install packages:
// OpenTelemetry.Extensions.Hosting
// OpenTelemetry.Instrumentation.AspNetCore
// OpenTelemetry.Instrumentation.Http
// OpenTelemetry.Instrumentation.EntityFrameworkCore
// OpenTelemetry.Exporter.Otlp
// OpenTelemetry.Exporter.Prometheus.AspNetCore
// OpenTelemetry.Exporter.Jaeger (or use OTLP → Jaeger)

// Program.cs
builder.Services.AddOpenTelemetry()
    .ConfigureResource(resource => resource
        .AddService(
            serviceName: "OrderService",
            serviceVersion: "2.1.0",
            serviceInstanceId: Environment.MachineName)
        .AddAttributes(new Dictionary<string, object>
        {
            ["deployment.environment"] = builder.Environment.EnvironmentName,
            ["service.region"] = "us-east-1"
        }))
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation(options =>
        {
            options.RecordException = true;
            options.Filter = ctx => !ctx.Request.Path.StartsWithSegments("/health");
        })
        .AddHttpClientInstrumentation(options =>
        {
            options.RecordException = true;
            options.FilterHttpRequestMessage = msg =>
                !msg.RequestUri?.Host.Contains("internal") ?? true;
        })
        .AddEntityFrameworkCoreInstrumentation(options =>
        {
            options.SetDbStatementForText = true;
            options.SetDbStatementForStoredProcedure = true;
        })
        .AddSource(OrderTelemetry.ActivitySourceName)  // Custom activities
        .AddOtlpExporter(options =>
        {
            options.Endpoint = new Uri(builder.Configuration["Otlp:Endpoint"]!);
            options.Protocol = OtlpExportProtocol.Grpc;
        }))
    .WithMetrics(metrics => metrics
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddRuntimeInstrumentation()
        .AddMeter(OrderTelemetry.MeterName)
        .AddOtlpExporter()
        .AddPrometheusExporter())
    .WithLogging(logging => logging
        .AddOtlpExporter());

// Expose Prometheus scrape endpoint
app.MapPrometheusScrapingEndpoint("/metrics");
```

---

## Step 1512: Custom ActivitySource (Distributed Tracing)

```csharp
// Centralized telemetry definitions
public static class OrderTelemetry
{
    public const string ActivitySourceName = "OrderService";
    public const string MeterName = "OrderService";

    private static readonly ActivitySource Source = new(ActivitySourceName, "1.0.0");
    private static readonly Meter Meter = new(MeterName, "1.0.0");

    // Counters
    public static readonly Counter<long> OrdersCreated =
        Meter.CreateCounter<long>("orders.created", "orders", "Total orders created");
    public static readonly Counter<long> OrdersFailed =
        Meter.CreateCounter<long>("orders.failed", "orders", "Total order failures");
    public static readonly Counter<long> PaymentsProcessed =
        Meter.CreateCounter<long>("payments.processed", "payments", "Total payments processed");

    // Histograms
    public static readonly Histogram<double> OrderProcessingDuration =
        Meter.CreateHistogram<double>("orders.processing.duration", "ms", "Order processing time");
    public static readonly Histogram<double> OrderTotal =
        Meter.CreateHistogram<double>("orders.total", "USD", "Order total amounts");

    // UpDownCounters (for gauges — can go up and down)
    public static readonly UpDownCounter<long> ActiveOrders =
        Meter.CreateUpDownCounter<long>("orders.active", "orders", "Currently active orders");
    public static readonly UpDownCounter<long> QueueDepth =
        Meter.CreateUpDownCounter<long>("queue.depth", "messages", "Message queue depth");

    // ObservableGauges (polled values)
    private static int _connectionCount;
    public static readonly ObservableGauge<int> DatabaseConnections =
        Meter.CreateObservableGauge<int>("db.connections", () => _connectionCount, "connections");

    public static void SetConnectionCount(int count) => _connectionCount = count;

    // Activity helpers
    public static Activity? StartCreateOrder() =>
        Source.StartActivity("CreateOrder", ActivityKind.Internal);

    public static Activity? StartProcessPayment(string provider) =>
        Source.StartActivity("ProcessPayment", ActivityKind.Client,
            tags: [new("payment.provider", provider)]);

    public static Activity? StartInventoryCheck() =>
        Source.StartActivity("CheckInventory", ActivityKind.Client);
}

// Usage in service
public class OrderApplicationService(
    IOrderRepository repo,
    IPaymentService payment,
    IInventoryService inventory,
    ILogger<OrderApplicationService> logger)
{
    public async Task<Order> CreateOrderAsync(CreateOrderRequest request, CancellationToken ct = default)
    {
        using var activity = OrderTelemetry.StartCreateOrder();
        var sw = Stopwatch.StartNew();

        // Add custom tags to the trace span
        activity?.SetTag("order.customer_id", request.CustomerId);
        activity?.SetTag("order.item_count", request.Items.Count);

        try
        {
            // Check inventory — creates child span
            using (var inventoryActivity = OrderTelemetry.StartInventoryCheck())
            {
                foreach (var item in request.Items)
                {
                    var available = await inventory.CheckAsync(item.ProductId, item.Quantity, ct);
                    if (!available)
                    {
                        inventoryActivity?.SetStatus(ActivityStatusCode.Error, "Insufficient stock");
                        throw new InsufficientStockException(item.ProductId);
                    }
                }
            }

            // Process payment — creates child span
            using (var paymentActivity = OrderTelemetry.StartProcessPayment("stripe"))
            {
                var charge = await payment.ChargeAsync(request.Total, request.Payment, ct);
                paymentActivity?.SetTag("payment.transaction_id", charge.TransactionId);
            }

            var order = await repo.CreateAsync(request.ToOrder(), ct);

            // Record metrics
            sw.Stop();
            OrderTelemetry.OrdersCreated.Add(1,
                new("customer.tier", request.CustomerTier.ToString()),
                new("payment.method", request.Payment.Type));
            OrderTelemetry.OrderProcessingDuration.Record(sw.Elapsed.TotalMilliseconds);
            OrderTelemetry.OrderTotal.Record((double)order.Total,
                new("currency", "USD"));
            OrderTelemetry.ActiveOrders.Add(1);

            activity?.SetTag("order.id", order.Id);
            activity?.SetStatus(ActivityStatusCode.Ok);

            return order;
        }
        catch (Exception ex)
        {
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            activity?.RecordException(ex);

            OrderTelemetry.OrdersFailed.Add(1,
                new("error.type", ex.GetType().Name));

            throw;
        }
    }
}
```

---

## Step 1513: Structured Logging with Serilog

```csharp
// Install: Serilog.AspNetCore, Serilog.Sinks.Console, Serilog.Sinks.Seq,
//          Serilog.Sinks.Elasticsearch, Serilog.Enrichers.Environment

// Program.cs
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Debug()
    .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
    .MinimumLevel.Override("Microsoft.Hosting.Lifetime", LogEventLevel.Information)
    .MinimumLevel.Override("System", LogEventLevel.Warning)
    .Enrich.FromLogContext()
    .Enrich.WithMachineName()
    .Enrich.WithEnvironmentName()
    .Enrich.WithProperty("Application", "OrderService")
    .Enrich.WithProperty("Version", "2.1.0")
    .WriteTo.Console(new JsonFormatter())
    .WriteTo.Seq("http://seq:5341")
    .WriteTo.Elasticsearch(new ElasticsearchSinkOptions(new Uri("http://elasticsearch:9200"))
    {
        IndexFormat = "orders-{0:yyyy.MM}",
        AutoRegisterTemplate = true
    })
    .CreateLogger();

builder.Host.UseSerilog();

// Structured log properties (searchable in Seq/Elasticsearch)
public class OrderEventLogger(ILogger<OrderEventLogger> logger)
{
    [LoggerMessage(
        Level = LogLevel.Information,
        Message = "Order {OrderId} created for customer {CustomerId} with {ItemCount} items, total {Total:C}")]
    private partial void LogOrderCreated(Guid orderId, Guid customerId, int itemCount, decimal total);

    [LoggerMessage(
        Level = LogLevel.Warning,
        Message = "Order {OrderId} payment failed: {DeclineReason}")]
    private partial void LogPaymentFailed(Guid orderId, string declineReason);

    [LoggerMessage(
        Level = LogLevel.Error,
        Message = "Order {OrderId} processing failed after {RetryCount} retries")]
    private partial void LogOrderFailed(Guid orderId, int retryCount, Exception ex);

    public void OrderCreated(Order order)
    {
        LogOrderCreated(order.Id, order.CustomerId, order.Items.Count, order.Total);

        // Enrich with correlation context
        using (LogContext.PushProperty("OrderId", order.Id))
        using (LogContext.PushProperty("CustomerId", order.CustomerId))
        {
            // All logs within this scope include OrderId and CustomerId
            LogOrderCreated(order.Id, order.CustomerId, order.Items.Count, order.Total);
        }
    }
}
```

---

## Step 1514: Health Checks

```csharp
// Comprehensive health check setup

builder.Services
    .AddHealthChecks()
    // Database
    .AddNpgsql(connectionString, name: "postgres",
        failureStatus: HealthStatus.Unhealthy,
        tags: ["db", "critical"])
    // Redis
    .AddRedis(redisConnectionString, name: "redis",
        failureStatus: HealthStatus.Degraded,
        tags: ["cache"])
    // RabbitMQ / Kafka
    .AddRabbitMQ(rabbitmqConnectionString, name: "rabbitmq", tags: ["messaging"])
    // Elasticsearch
    .AddElasticsearch(elasticUrl, name: "elasticsearch", tags: ["search"])
    // External HTTP APIs
    .AddUrlGroup(new Uri("https://api.stripe.com/v1/"), name: "stripe",
        failureStatus: HealthStatus.Degraded)
    // Custom check
    .AddCheck<OrderQueueHealthCheck>("order-queue", tags: ["queue"])
    .AddCheck<DiskSpaceHealthCheck>("disk-space");

// Custom health check
public class OrderQueueHealthCheck(IConnection rabbitmq) : IHealthCheck
{
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken ct = default)
    {
        try
        {
            using var channel = await rabbitmq.CreateChannelAsync(ct: ct);
            var queueInfo = await channel.QueueDeclarePassiveAsync("orders", ct);

            var messageCount = queueInfo.MessageCount;
            var consumerCount = queueInfo.ConsumerCount;

            var data = new Dictionary<string, object>
            {
                ["queue.message_count"] = messageCount,
                ["queue.consumer_count"] = consumerCount,
                ["queue.name"] = "orders"
            };

            if (messageCount > 10000)
                return HealthCheckResult.Degraded("Queue backlog is high", data: data);
            if (consumerCount == 0)
                return HealthCheckResult.Unhealthy("No consumers on queue", data: data);

            return HealthCheckResult.Healthy("Queue is healthy", data);
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("Cannot connect to RabbitMQ", ex);
        }
    }
}

// Health check endpoints with detailed response
app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("critical"),
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false,  // Just checks the app is running
});

// Health Checks UI
builder.Services.AddHealthChecksUI(options =>
{
    options.SetEvaluationTimeInSeconds(15);
    options.MaximumHistoryEntriesPerEndpoint(50);
    options.AddHealthCheckEndpoint("Order Service", "/health");
}).AddInMemoryStorage();

app.MapHealthChecksUI("/health-ui");
```

---

## Step 1515: Distributed Tracing — Baggage and Context Propagation

```csharp
// Propagating context across service boundaries
// W3C TraceContext standard — built into .NET

public class OrderProcessingService(HttpClient httpClient)
{
    public async Task<InventoryResult> CheckInventoryAsync(Guid productId, int quantity)
    {
        // Activity context (TraceId, SpanId) automatically propagated via HTTP headers
        // traceparent: 00-{traceId}-{spanId}-{flags}
        // tracestate: vendor-specific data

        var response = await httpClient.GetAsync(
            $"http://inventory-service/api/inventory/{productId}?qty={quantity}");
        return await response.Content.ReadFromJsonAsync<InventoryResult>();
    }
}

// Baggage — key/value pairs propagated across services
public class OrderRequestEnricher
{
    public static void EnrichWithOrderContext(Order order)
    {
        // Set baggage — travels with all downstream spans
        Baggage.SetBaggage("order.id", order.Id.ToString());
        Baggage.SetBaggage("customer.tier", order.CustomerTier.ToString());

        // Downstream services can read these
        var orderId = Baggage.GetBaggage("order.id");
    }
}

// Custom propagator for non-HTTP transport (e.g., Kafka)
public class KafkaTracePropagator
{
    private static readonly TextMapPropagator Propagator = Propagators.DefaultTextMapPropagator;

    public void Inject(Message<string, string> message, Activity? activity)
    {
        Propagator.Inject(
            new PropagationContext(activity?.Context ?? default, Baggage.Current),
            message,
            (msg, key, value) => msg.Headers ??= new Headers(),
            (msg, key, value) => msg.Headers!.Add(new Header(key, Encoding.UTF8.GetBytes(value)))
        );
    }

    public PropagationContext Extract(Message<string, string> message)
    {
        return Propagator.Extract(
            default,
            message,
            (msg, key) =>
            {
                var header = msg.Headers?.FirstOrDefault(h => h.Key == key);
                return header is null ? [] : [Encoding.UTF8.GetString(header.GetValueBytes())];
            }
        );
    }
}

// Using extracted context when consuming Kafka messages
public async Task ConsumeOrderAsync(ConsumeResult<string, string> result, CancellationToken ct)
{
    var propagator = new KafkaTracePropagator();
    var parentContext = propagator.Extract(result.Message);

    // Create child span linked to the producer's trace
    using var activity = OrderTelemetry.Source.StartActivity(
        "ConsumeOrder",
        ActivityKind.Consumer,
        parentContext.ActivityContext
    );

    activity?.SetTag("messaging.system", "kafka");
    activity?.SetTag("messaging.destination", result.Topic);

    await ProcessOrderMessageAsync(result.Message.Value, ct);
}
```

---

## Step 1516: Metrics — Histograms and Exemplars

```csharp
// Exemplars — link metric observations to trace spans

public class PaymentService(IPaymentGateway gateway)
{
    private static readonly Histogram<double> PaymentDuration =
        OrderTelemetry.Meter.CreateHistogram<double>(
            "payment.duration",
            "ms",
            "Payment processing duration");

    public async Task<ChargeResult> ChargeAsync(decimal amount, PaymentMethod method, CancellationToken ct)
    {
        var sw = Stopwatch.StartNew();
        using var activity = Activity.Current;

        try
        {
            var result = await gateway.ChargeAsync(amount, method, ct);
            sw.Stop();

            // Record with exemplar — links this metric observation to the current trace
            PaymentDuration.Record(sw.Elapsed.TotalMilliseconds,
                new("payment.provider", method.ProviderName),
                new("payment.status", "success"),
                new("payment.currency", "USD")
            );

            return result;
        }
        catch (Exception ex)
        {
            sw.Stop();
            PaymentDuration.Record(sw.Elapsed.TotalMilliseconds,
                new("payment.provider", method.ProviderName),
                new("payment.status", "failure"),
                new("error.type", ex.GetType().Name)
            );
            throw;
        }
    }
}

// Rate counter (custom)
public class RateCounter
{
    private readonly Counter<long> _counter;
    private long _lastCount;
    private DateTimeOffset _lastReset = DateTimeOffset.UtcNow;

    public RateCounter(string name, string unit, string description)
    {
        _counter = OrderTelemetry.Meter.CreateCounter<long>(name, unit, description);
    }

    public void Increment(params KeyValuePair<string, object?>[] tags)
    {
        _counter.Add(1, tags);
        Interlocked.Increment(ref _lastCount);
    }

    public double GetRatePerSecond()
    {
        var elapsed = (DateTimeOffset.UtcNow - _lastReset).TotalSeconds;
        return elapsed > 0 ? Interlocked.Read(ref _lastCount) / elapsed : 0;
    }
}
```

---

## Step 1517: Grafana Dashboard Configuration

```json
// grafana-dashboard.json (key panels for Order Service)
{
  "title": "Order Service - SLO Dashboard",
  "panels": [
    {
      "title": "Request Rate",
      "type": "timeseries",
      "targets": [{
        "expr": "sum(rate(http_server_request_duration_count{service='order-service'}[5m])) by (http_route)",
        "legendFormat": "{{http_route}}"
      }]
    },
    {
      "title": "Error Rate",
      "type": "stat",
      "targets": [{
        "expr": "sum(rate(http_server_request_duration_count{service='order-service',http_response_status_code=~'5..'}[5m])) / sum(rate(http_server_request_duration_count{service='order-service'}[5m]))",
        "legendFormat": "Error Rate"
      }],
      "thresholds": [
        { "value": 0.01, "color": "yellow" },
        { "value": 0.05, "color": "red" }
      ]
    },
    {
      "title": "P99 Latency",
      "type": "timeseries",
      "targets": [{
        "expr": "histogram_quantile(0.99, sum(rate(http_server_request_duration_bucket{service='order-service'}[5m])) by (le, http_route))",
        "legendFormat": "{{http_route}} p99"
      }]
    },
    {
      "title": "Orders Created per Minute",
      "type": "timeseries",
      "targets": [{
        "expr": "sum(rate(orders_created_total{service='order-service'}[1m])) * 60",
        "legendFormat": "Orders/min"
      }]
    },
    {
      "title": "Active Database Connections",
      "type": "gauge",
      "targets": [{
        "expr": "db_connections{service='order-service'}"
      }],
      "min": 0, "max": 100,
      "thresholds": [{ "value": 80, "color": "red" }]
    },
    {
      "title": "Payment Success Rate",
      "type": "stat",
      "targets": [{
        "expr": "sum(rate(payments_processed_total{service='order-service',payment_status='success'}[5m])) / sum(rate(payments_processed_total{service='order-service'}[5m]))"
      }]
    }
  ]
}
```

---

## Step 1518: Logging Best Practices

```csharp
// Logging guidelines for production .NET applications

public class OrderService(ILogger<OrderService> logger)
{
    // ✅ Use structured logging — fields are searchable
    public async Task<Order> GetOrderAsync(Guid orderId)
    {
        logger.LogInformation("Retrieving order {OrderId}", orderId);
        // Produces: {"Message": "Retrieving order abc123", "OrderId": "abc123"}

        var order = await _repo.GetByIdAsync(orderId);

        if (order is null)
        {
            logger.LogWarning("Order {OrderId} not found", orderId);
            return null!;
        }

        return order;
    }

    // ✅ Log exceptions as exceptions, not as strings
    public async Task ProcessPaymentAsync(Order order, PaymentMethod method)
    {
        try
        {
            await _payment.ChargeAsync(order.Total, method);
        }
        catch (PaymentException ex)
        {
            // ✅ Exception goes as structured data — stack trace, type, etc.
            logger.LogError(ex, "Payment failed for order {OrderId} with provider {Provider}",
                order.Id, method.ProviderName);
            throw;
        }
    }

    // ✅ Use log scopes for correlated operations
    public async Task ProcessOrderBatchAsync(IEnumerable<Guid> orderIds)
    {
        var batchId = Guid.NewGuid();
        using var scope = logger.BeginScope(new Dictionary<string, object>
        {
            ["BatchId"] = batchId,
            ["BatchSize"] = orderIds.Count()
        });

        foreach (var orderId in orderIds)
        {
            logger.LogInformation("Processing order {OrderId} in batch", orderId);
            await ProcessOrderAsync(orderId);
        }

        logger.LogInformation("Batch {BatchId} complete", batchId);
    }

    // ✅ Check log level before expensive string construction
    public void LogOrderDetails(Order order)
    {
        if (logger.IsEnabled(LogLevel.Debug))
        {
            // Expensive serialization only when DEBUG is enabled
            logger.LogDebug("Order details: {Order}", JsonSerializer.Serialize(order));
        }
    }

    // ✅ Source-generated logging — zero allocation at runtime
    [LoggerMessage(Level = LogLevel.Information, Message = "Order {OrderId} confirmed by {UserId}")]
    private partial void LogOrderConfirmed(Guid orderId, Guid userId);

    [LoggerMessage(Level = LogLevel.Warning, Message = "Order {OrderId} payment retry #{RetryCount}")]
    private partial void LogPaymentRetry(Guid orderId, int retryCount);

    // ❌ Don't: string concatenation in log calls
    // logger.LogInformation("Order " + order.Id + " created"); // Allocates even if not logged

    // ❌ Don't: log sensitive data
    // logger.LogDebug("User password hash: {Hash}", user.PasswordHash);
    // logger.LogDebug("Credit card: {CardNumber}", payment.CardNumber);
}

// Redaction — mask sensitive fields
public class SensitiveDataRedactingLogger(ILogger inner) : ILogger
{
    private static readonly Regex CardNumberPattern = new(@"\b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b");

    public IDisposable? BeginScope<TState>(TState state) where TState : notnull =>
        inner.BeginScope(state);

    public bool IsEnabled(LogLevel logLevel) => inner.IsEnabled(logLevel);

    public void Log<TState>(LogLevel logLevel, EventId eventId, TState state,
        Exception? exception, Func<TState, Exception?, string> formatter)
    {
        var message = formatter(state, exception);
        var redacted = CardNumberPattern.Replace(message, "****-****-****-****");
        inner.Log(logLevel, eventId, redacted, exception, (m, _) => m.ToString()!);
    }
}
```

---

## Step 1519: Alerting Rules (Prometheus/AlertManager)

```yaml
# alerts.yaml
groups:
  - name: order-service
    interval: 30s
    rules:
      # Error rate > 5% for 5 minutes
      - alert: HighErrorRate
        expr: |
          sum(rate(http_server_request_duration_count{service="order-service",http_response_status_code=~"5.."}[5m]))
          /
          sum(rate(http_server_request_duration_count{service="order-service"}[5m]))
          > 0.05
        for: 5m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "High error rate on Order Service"
          description: "Error rate is {{ humanizePercentage $value }} (threshold: 5%)"
          runbook_url: "https://wiki.example.com/runbooks/high-error-rate"

      # P99 latency > 2s for 10 minutes
      - alert: HighLatency
        expr: |
          histogram_quantile(0.99, sum(rate(http_server_request_duration_bucket{service="order-service"}[5m])) by (le))
          > 2
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "P99 latency above 2s"
          description: "P99 latency is {{ $value | humanizeDuration }}"

      # Payment failures
      - alert: PaymentFailures
        expr: |
          sum(rate(payments_processed_total{service="order-service",payment_status="failure"}[5m]))
          > 0.1
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High payment failure rate"

      # Order queue backlog
      - alert: QueueBacklog
        expr: queue_depth{service="order-service"} > 1000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Order processing queue backlog: {{ $value }} messages"

      # Service down
      - alert: ServiceDown
        expr: up{job="order-service"} == 0
        for: 1m
        labels:
          severity: critical
          page: "true"
        annotations:
          summary: "Order Service is down!"
```

---

## Step 1520: Observability Stack with Docker Compose

```yaml
# docker-compose.observability.yml
services:
  # Metrics collection and storage
  prometheus:
    image: prom/prometheus:v2.54.1
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=30d'
      - '--web.enable-lifecycle'
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./alerts.yaml:/etc/prometheus/alerts.yaml:ro
      - prometheus-data:/prometheus
    ports:
      - "9090:9090"

  # Alerting
  alertmanager:
    image: prom/alertmanager:v0.27.0
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
    ports:
      - "9093:9093"

  # Metrics visualization
  grafana:
    image: grafana/grafana:11.2.0
    environment:
      GF_AUTH_ANONYMOUS_ENABLED: "true"
      GF_AUTH_ANONYMOUS_ORG_ROLE: "Admin"
    volumes:
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards:ro
      - ./grafana/datasources:/etc/grafana/provisioning/datasources:ro
      - grafana-data:/var/lib/grafana
    ports:
      - "3000:3000"

  # Distributed tracing
  jaeger:
    image: jaegertracing/all-in-one:1.61.0
    environment:
      COLLECTOR_OTLP_ENABLED: "true"
    ports:
      - "16686:16686"  # Jaeger UI
      - "4317:4317"    # OTLP gRPC
      - "4318:4318"    # OTLP HTTP

  # Log aggregation
  loki:
    image: grafana/loki:3.2.0
    volumes:
      - ./loki.yaml:/etc/loki/config.yaml:ro
      - loki-data:/loki
    ports:
      - "3100:3100"

  # Log shipping
  promtail:
    image: grafana/promtail:3.2.0
    volumes:
      - /var/log:/var/log:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - ./promtail.yaml:/etc/promtail/config.yaml:ro

  # OpenTelemetry Collector (receives all telemetry, routes to backends)
  otel-collector:
    image: otel/opentelemetry-collector-contrib:0.111.0
    command: ["--config=/etc/otel-collector-config.yaml"]
    volumes:
      - ./otel-collector-config.yaml:/etc/otel-collector-config.yaml:ro
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
      - "8888:8888"   # Prometheus metrics of the collector itself
      - "8889:8889"   # Prometheus exporter

  # Log search and analysis
  seq:
    image: datalust/seq:2024.3
    environment:
      ACCEPT_EULA: Y
    volumes:
      - seq-data:/data
    ports:
      - "5341:80"

volumes:
  prometheus-data:
  grafana-data:
  loki-data:
  seq-data:
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
    timeout: 1s
    send_batch_size: 1024
  memory_limiter:
    limit_mib: 512
    spike_limit_mib: 128
  resource:
    attributes:
      - key: environment
        value: production
        action: upsert

exporters:
  jaeger:
    endpoint: jaeger:14250
    tls:
      insecure: true
  prometheus:
    endpoint: "0.0.0.0:8889"
  loki:
    endpoint: http://loki:3100/loki/api/v1/push

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
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

## Step 1521: Application Insights (Azure)

```csharp
// Azure Application Insights via OpenTelemetry exporter

builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddAzureMonitorTraceExporter(options =>
        {
            options.ConnectionString = builder.Configuration["ApplicationInsights:ConnectionString"];
        }))
    .WithMetrics(metrics => metrics
        .AddAzureMonitorMetricExporter(options =>
        {
            options.ConnectionString = builder.Configuration["ApplicationInsights:ConnectionString"];
        }))
    .WithLogging(logging => logging
        .AddAzureMonitorLogExporter(options =>
        {
            options.ConnectionString = builder.Configuration["ApplicationInsights:ConnectionString"];
        }));

// KQL queries for Application Insights (Log Analytics)
// Request failures in the last hour
/*
requests
| where timestamp > ago(1h)
| where success == false
| summarize count() by resultCode, name
| order by count_ desc
*/

// P99 latency per endpoint
/*
requests
| where timestamp > ago(1h)
| summarize percentile(duration, 99) by name
| order by percentile_duration_99 desc
| render barchart
*/

// Trace correlation — find all operations for an order
/*
let orderId = "12345678-...";
union *
| where customDimensions["order.id"] == orderId
| order by timestamp asc
| project timestamp, itemType, name, message = coalesce(message, tostring(customDimensions))
*/
```

---

## Summary: Three Pillars of Observability

```
╔══════════════════════════════════════════════════════════════╗
║              Three Pillars of Observability                  ║
╠══════════════╦══════════════╦══════════════════════════════╣
║   Traces     ║   Metrics    ║           Logs               ║
╠══════════════╬══════════════╬══════════════════════════════╣
║ WHY it's slow║ HOW it's    ║ WHAT happened                ║
║              ║ behaving     ║                              ║
╠══════════════╬══════════════╬══════════════════════════════╣
║ ActivitySource│ Meter.Create║ ILogger<T>                   ║
║ StartActivity ║ Counter<T>  ║ LoggerMessage                ║
║ SetTag()      ║ Histogram<T>║ BeginScope()                 ║
║ SetStatus()   ║ Gauge<T>    ║ Serilog enrichers            ║
╠══════════════╬══════════════╬══════════════════════════════╣
║ Jaeger        ║ Prometheus  ║ Elasticsearch / Loki / Seq   ║
║ Zipkin        ║ Grafana     ║ Azure Monitor                ║
║ Azure Monitor ║ DataDog     ║ DataDog                      ║
╠══════════════╬══════════════╬══════════════════════════════╣
║ TraceId links ║ SLO alerts  ║ Error context                ║
║ all spans     ║ SLI dashbds ║ Correlation IDs              ║
╚══════════════╩══════════════╩══════════════════════════════╝

OpenTelemetry Collector:
  Receives all three → Routes to multiple backends
  Buffers, batches, retries → No vendor lock-in
```

---

*Part 54 complete — Steps 1511-1521. Topics covered: OpenTelemetry SDK setup (tracing, metrics, logging), custom ActivitySource with tags and status, Meter with Counter/Histogram/UpDownCounter/ObservableGauge, structured logging with Serilog (enrichers, sinks: Seq/Elasticsearch/console), health checks (built-in + custom + UI), distributed context propagation (W3C TraceContext, Baggage), Kafka trace propagation, metric exemplars linking to traces, Grafana dashboard JSON, Prometheus alerting rules, full observability stack with Docker Compose (Prometheus, Grafana, Jaeger, Loki, Seq, OTel Collector), OTel Collector config for fan-out to multiple backends, Application Insights via Azure Monitor exporter.*
