# Part 70: Resilience Patterns — Polly v8, Circuit Breaker & Hedging (Steps 1749-1764)

## Steps 1749-1764: Building Fault-Tolerant .NET Applications

In distributed systems, partial failures are inevitable: a dependency slows down, a network blips, a service temporarily overloads. Resilience engineering is the discipline of designing systems that degrade gracefully rather than failing catastrophically. This part covers the full Polly v8 `ResiliencePipeline` API, all built-in strategies, and real-world patterns for HTTP clients and messaging.

---

## Step 1749: The Cost of Not Being Resilient

```
Without resilience:
  OrderApi → PaymentService (down for 2s)
  → Thread pool exhausted after 50ms timeout
  → 500 Internal Server Error cascades to all 1000 concurrent users
  → CartoonTime: 100% error rate for 10 minutes

With Circuit Breaker:
  OrderApi → PaymentService (circuit OPEN after 5 failures)
  → Fail-fast with fallback within 1ms
  → 2% error rate, 98% degraded-mode success
  → Service recovers, circuit closes automatically
```

---

## Step 1750: Polly v8 — ResiliencePipeline

```xml
<PackageReference Include="Microsoft.Extensions.Http.Resilience" Version="8.10.0" />
<PackageReference Include="Polly" Version="8.4.2" />
<PackageReference Include="Polly.Extensions" Version="8.4.2" />
```

```csharp
using Polly;
using Polly.Retry;
using Polly.CircuitBreaker;
using Polly.Timeout;
using Polly.Fallback;
using Polly.Hedging;

// Build a pipeline — strategies execute in ORDER (outermost to innermost)
var pipeline = new ResiliencePipelineBuilder<HttpResponseMessage>()
    // 1. Fallback — outermost: catches everything from inner strategies
    .AddFallback(new FallbackStrategyOptions<HttpResponseMessage>
    {
        ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
            .Handle<BrokenCircuitException>()
            .Handle<TimeoutRejectedException>(),
        FallbackAction = ctx =>
        {
            var cached = _cache.Get<HttpResponseMessage>(cacheKey);
            return Outcome.FromResultAsValueTask(cached ?? CreateDegradedResponse());
        },
        OnFallback = args =>
        {
            _logger.LogWarning("Fallback triggered. Outcome: {Outcome}", args.Outcome.Exception?.Message);
            return ValueTask.CompletedTask;
        }
    })
    // 2. Circuit Breaker — opens after too many failures
    .AddCircuitBreaker(new CircuitBreakerStrategyOptions<HttpResponseMessage>
    {
        ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
            .Handle<HttpRequestException>()
            .HandleResult(r => (int)r.StatusCode >= 500),
        FailureRatio = 0.5,                  // open when 50% of requests fail
        SamplingDuration = TimeSpan.FromSeconds(30),
        MinimumThroughput = 10,              // need at least 10 requests to evaluate
        BreakDuration = TimeSpan.FromSeconds(30),  // stay open for 30s
        OnOpened = args =>
        {
            _logger.LogError("Circuit opened for {Duration}", args.BreakDuration);
            return ValueTask.CompletedTask;
        },
        OnClosed = _ =>
        {
            _logger.LogInformation("Circuit closed — service recovered");
            return ValueTask.CompletedTask;
        }
    })
    // 3. Retry — retry transient failures
    .AddRetry(new RetryStrategyOptions<HttpResponseMessage>
    {
        ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
            .Handle<HttpRequestException>()
            .HandleResult(r => r.StatusCode is
                System.Net.HttpStatusCode.RequestTimeout or
                System.Net.HttpStatusCode.TooManyRequests or
                System.Net.HttpStatusCode.ServiceUnavailable),
        MaxRetryAttempts = 3,
        Delay = TimeSpan.FromMilliseconds(500),
        BackoffType = DelayBackoffType.Exponential,
        UseJitter = true,                    // randomize to avoid thundering herd
        OnRetry = args =>
        {
            _logger.LogWarning("Retry {Attempt}/{Max} after {Delay}",
                args.AttemptNumber, 3, args.RetryDelay);
            return ValueTask.CompletedTask;
        }
    })
    // 4. Timeout — innermost: prevents hanging indefinitely
    .AddTimeout(TimeSpan.FromSeconds(5))
    .Build();

// Execute
var response = await pipeline.ExecuteAsync(
    async ct => await _httpClient.GetAsync("/api/payment/status", ct),
    cancellationToken);
```

---

## Step 1751: AddStandardResilienceHandler — Pre-Built Pipeline

Microsoft provides a standard opinionated pipeline for HTTP clients:

```csharp
// Program.cs
builder.Services.AddHttpClient<PaymentServiceClient>(client =>
{
    client.BaseAddress = new Uri("http://payment-service");
    client.DefaultRequestHeaders.Add("Accept", "application/json");
})
.AddStandardResilienceHandler(options =>
{
    // All these have sensible defaults; override only what you need
    options.Retry.MaxRetryAttempts = 3;
    options.Retry.Delay = TimeSpan.FromMilliseconds(500);
    options.Retry.BackoffType = DelayBackoffType.Exponential;
    options.Retry.UseJitter = true;

    options.CircuitBreaker.FailureRatio = 0.5;
    options.CircuitBreaker.SamplingDuration = TimeSpan.FromSeconds(30);
    options.CircuitBreaker.BreakDuration = TimeSpan.FromSeconds(30);

    options.AttemptTimeout.Timeout = TimeSpan.FromSeconds(10);
    options.TotalRequestTimeout.Timeout = TimeSpan.FromSeconds(30);
});

// The standard pipeline applies: TotalRequestTimeout → CircuitBreaker → Retry → AttemptTimeout
```

---

## Step 1752: Retry with Respect for Retry-After

```csharp
builder.Services.AddHttpClient<ApiClient>()
    .AddResilienceHandler("api-retry", (builder, context) =>
    {
        builder.AddRetry(new RetryStrategyOptions<HttpResponseMessage>
        {
            ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
                .HandleResult(r => r.StatusCode == System.Net.HttpStatusCode.TooManyRequests),
            MaxRetryAttempts = 5,
            DelayGenerator = args =>
            {
                // Respect Retry-After header from the server
                if (args.Outcome.Result?.Headers.RetryAfter is { } retryAfter)
                {
                    var delay = retryAfter.Delta
                             ?? (retryAfter.Date - DateTimeOffset.UtcNow);
                    if (delay > TimeSpan.Zero)
                        return ValueTask.FromResult<TimeSpan?>(delay);
                }
                // Default exponential backoff
                var defaultDelay = TimeSpan.FromSeconds(Math.Pow(2, args.AttemptNumber));
                return ValueTask.FromResult<TimeSpan?>(defaultDelay);
            }
        });
    });
```

---

## Step 1753: Circuit Breaker State Machine

```
           failure count exceeds threshold
Closed ──────────────────────────────────────► Open
  ▲                                              │
  │ success                                      │ wait BreakDuration
  │                                              ▼
Half-Open ◄───────────────────────────────── Open
  │
  │ one probe request allowed
  │ success → Closed
  │ failure → back to Open
```

```csharp
// Inject circuit breaker state for health checks / monitoring
public class PaymentServiceHealthCheck(
    ResiliencePipelineProvider<string> pipelineProvider) : IHealthCheck
{
    public Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken ct = default)
    {
        var pipeline = pipelineProvider.GetPipeline<HttpResponseMessage>("payment-service");
        var cbState = pipeline.GetCircuitBreakerState();  // via Polly.Extensions

        return Task.FromResult(cbState switch
        {
            CircuitState.Closed => HealthCheckResult.Healthy("Circuit closed"),
            CircuitState.HalfOpen => HealthCheckResult.Degraded("Circuit half-open (probing)"),
            CircuitState.Open => HealthCheckResult.Unhealthy("Circuit open — payment service unreachable"),
            _ => HealthCheckResult.Degraded("Unknown circuit state")
        });
    }
}
```

---

## Step 1754: Timeout Strategy

```csharp
var timeoutPipeline = new ResiliencePipelineBuilder()
    .AddTimeout(new TimeoutStrategyOptions
    {
        Timeout = TimeSpan.FromSeconds(5),
        OnTimeout = args =>
        {
            _logger.LogWarning(
                "Operation timed out after {Timeout}. OperationKey={Key}",
                args.Timeout,
                args.Context.OperationKey);
            return ValueTask.CompletedTask;
        }
    })
    .Build();

// The timeout cancels the inner CancellationToken
await timeoutPipeline.ExecuteAsync(async ct =>
{
    // This CancellationToken is cancelled after 5 seconds
    var result = await _httpClient.GetAsync("/api/slow-endpoint", ct);
    return result.Content.ReadAsStringAsync(ct);
}, cancellationToken);
// Throws TimeoutRejectedException if exceeded
```

---

## Step 1755: Hedging — Parallel Redundant Requests

Hedging sends a second (redundant) request if the first hasn't completed within a threshold. Returns whichever responds first.

```csharp
var hedgingPipeline = new ResiliencePipelineBuilder<HttpResponseMessage>()
    .AddHedging(new HedgingStrategyOptions<HttpResponseMessage>
    {
        // Send a second request if the first doesn't respond in 1s
        Delay = TimeSpan.FromSeconds(1),
        MaxHedgedAttempts = 2,      // up to 2 parallel hedged requests
        ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
            .Handle<HttpRequestException>()
            .HandleResult(r => r.StatusCode >= System.Net.HttpStatusCode.InternalServerError),
        ActionGenerator = args =>
        {
            // The hedged action can target a different endpoint
            return () => args.Callback(args.ActionContext);
        },
        OnHedging = args =>
        {
            _logger.LogInformation(
                "Hedging: sending attempt {Attempt} after {Delay}",
                args.AttemptNumber,
                args.Delay);
            return ValueTask.CompletedTask;
        }
    })
    .Build();

// Use case: read from nearest replica; hedge to secondary after 100ms
var result = await hedgingPipeline.ExecuteAsync(async ct =>
    await _primaryClient.GetAsync("/data", ct));
// If primary doesn't reply in 1s, also fires against secondary replica
// Returns the first 2xx response
```

---

## Step 1756: Bulkhead — Limiting Concurrency

```csharp
// Prevent one slow dependency from consuming all thread pool threads
var bulkheadPipeline = new ResiliencePipelineBuilder()
    .AddConcurrencyLimiter(new ConcurrencyLimiterStrategyOptions
    {
        PermitLimit = 50,              // max 50 concurrent calls
        QueuedTasksLimit = 100,        // queue up to 100 waiting tasks
        OnRejected = args =>
        {
            _logger.LogWarning("Bulkhead full — rejecting request. Key={Key}",
                args.Context.OperationKey);
            return ValueTask.CompletedTask;
        }
    })
    .Build();

// Wrap expensive downstream calls
try
{
    var result = await bulkheadPipeline.ExecuteAsync(
        async ct => await _expensiveService.CallAsync(ct),
        cancellationToken);
}
catch (BulkheadRejectedException)
{
    return Results.StatusCode(503); // Service Unavailable
}
```

---

## Step 1757: Rate Limiter Strategy

```csharp
using System.Threading.RateLimiting;

var rateLimitedPipeline = new ResiliencePipelineBuilder<HttpResponseMessage>()
    .AddRateLimiter(new SlidingWindowRateLimiter(
        new SlidingWindowRateLimiterOptions
        {
            PermitLimit = 100,
            Window = TimeSpan.FromSeconds(10),
            SegmentsPerWindow = 5,
            QueueProcessingOrder = QueueProcessingOrder.OldestFirst,
            QueueLimit = 50
        }))
    .Build();
```

---

## Step 1758: Registering Pipelines with DI

```csharp
// Register named pipelines with the DI container
builder.Services.AddResiliencePipeline<string, HttpResponseMessage>(
    "payment-service",
    (builder, context) =>
    {
        // Access DI services inside the builder
        var logger = context.ServiceProvider.GetRequiredService<ILogger<Program>>();
        var meter = context.ServiceProvider.GetRequiredService<IMeterFactory>()
            .Create("Polly.Pipelines");

        builder
            .AddRetry(new RetryStrategyOptions<HttpResponseMessage>
            {
                MaxRetryAttempts = 3,
                Delay = TimeSpan.FromMilliseconds(500),
                BackoffType = DelayBackoffType.Exponential,
                UseJitter = true,
                OnRetry = args =>
                {
                    logger.LogWarning("Retry {Attempt}", args.AttemptNumber);
                    return ValueTask.CompletedTask;
                }
            })
            .AddCircuitBreaker(new CircuitBreakerStrategyOptions<HttpResponseMessage>
            {
                FailureRatio = 0.5,
                MinimumThroughput = 10,
                BreakDuration = TimeSpan.FromSeconds(30)
            })
            .AddTimeout(TimeSpan.FromSeconds(10));
    });

// Inject and use
public class PaymentService(ResiliencePipelineProvider<string> pipelineProvider)
{
    private readonly ResiliencePipeline<HttpResponseMessage> _pipeline =
        pipelineProvider.GetPipeline<HttpResponseMessage>("payment-service");

    public async Task<PaymentResult?> ProcessAsync(PaymentRequest req, CancellationToken ct)
    {
        try
        {
            var response = await _pipeline.ExecuteAsync(
                async token => await _client.PostAsJsonAsync("/process", req, token),
                ct);

            response.EnsureSuccessStatusCode();
            return await response.Content.ReadFromJsonAsync<PaymentResult>(ct);
        }
        catch (BrokenCircuitException)
        {
            return null; // circuit open — fast fail
        }
    }
}
```

---

## Step 1759: Resilience with gRPC Clients

```csharp
// gRPC + Polly — retry on transient gRPC status codes
builder.Services.AddGrpcClient<OrderService.OrderServiceClient>(opts =>
{
    opts.Address = new Uri("http://order-grpc-service");
})
.AddResilienceHandler("grpc-retry", builder =>
{
    builder.AddRetry(new RetryStrategyOptions<HttpResponseMessage>
    {
        MaxRetryAttempts = 3,
        ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
            .Handle<Grpc.Core.RpcException>(ex =>
                ex.StatusCode is
                    Grpc.Core.StatusCode.Unavailable or
                    Grpc.Core.StatusCode.DeadlineExceeded or
                    Grpc.Core.StatusCode.ResourceExhausted),
        Delay = TimeSpan.FromMilliseconds(200),
        BackoffType = DelayBackoffType.Exponential,
        UseJitter = true
    })
    .AddTimeout(TimeSpan.FromSeconds(5));
});
```

---

## Step 1760: Context — Passing Data Through the Pipeline

```csharp
// Pass context data to strategy callbacks
var context = ResilienceContextPool.Shared.Get(operationKey: "place-order");
context.Properties.Set(new ResiliencePropertyKey<string>("order.id"), orderId.ToString());
context.Properties.Set(new ResiliencePropertyKey<string>("customer.tier"), tier);

try
{
    await pipeline.ExecuteAsync(async (ctx, ct) =>
    {
        // Access context inside the execution
        var orderId = ctx.Properties.GetValue(
            new ResiliencePropertyKey<string>("order.id"), "unknown");
        return await _client.PostAsync("/orders", content, ct);
    }, context, cancellationToken);
}
finally
{
    ResilienceContextPool.Shared.Return(context);
}

// Access in strategy callbacks
.AddRetry(new RetryStrategyOptions<HttpResponseMessage>
{
    OnRetry = args =>
    {
        var orderId = args.Context.Properties.GetValue(
            new ResiliencePropertyKey<string>("order.id"), "unknown");
        _logger.LogWarning("Retrying order {OrderId} attempt {Attempt}", orderId, args.AttemptNumber);
        return ValueTask.CompletedTask;
    }
})
```

---

## Step 1761: Testing Resilience Pipelines

```csharp
using Polly.Testing;

public class PaymentServiceTests
{
    [Fact]
    public async Task CircuitBreaker_Opens_After_5_Failures()
    {
        var pipeline = new ResiliencePipelineBuilder<HttpResponseMessage>()
            .AddCircuitBreaker(new CircuitBreakerStrategyOptions<HttpResponseMessage>
            {
                FailureRatio = 1.0,       // open on 100% failure
                MinimumThroughput = 3,    // after 3 attempts
                BreakDuration = TimeSpan.FromSeconds(10),
                ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
                    .Handle<HttpRequestException>()
            })
            .Build();

        // Trigger failures
        for (int i = 0; i < 3; i++)
        {
            await Assert.ThrowsAsync<HttpRequestException>(() =>
                pipeline.ExecuteAsync(_ =>
                    ValueTask.FromException<HttpResponseMessage>(
                        new HttpRequestException("Service down")),
                CancellationToken.None).AsTask());
        }

        // Circuit should now be open
        await Assert.ThrowsAsync<BrokenCircuitException>(() =>
            pipeline.ExecuteAsync(_ =>
                ValueTask.FromResult(new HttpResponseMessage()),
            CancellationToken.None).AsTask());
    }

    [Fact]
    public async Task Retry_Succeeds_On_Third_Attempt()
    {
        int callCount = 0;

        var pipeline = new ResiliencePipelineBuilder<string>()
            .AddRetry(new RetryStrategyOptions<string>
            {
                MaxRetryAttempts = 3,
                Delay = TimeSpan.Zero,
                ShouldHandle = new PredicateBuilder<string>()
                    .Handle<InvalidOperationException>()
            })
            .Build();

        var result = await pipeline.ExecuteAsync(_ =>
        {
            callCount++;
            if (callCount < 3)
                throw new InvalidOperationException("Not yet");
            return ValueTask.FromResult("success");
        }, CancellationToken.None);

        Assert.Equal("success", result);
        Assert.Equal(3, callCount);
    }

    [Fact]
    public async Task Hedging_Returns_Faster_Response()
    {
        var pipeline = new ResiliencePipelineBuilder<string>()
            .AddHedging(new HedgingStrategyOptions<string>
            {
                Delay = TimeSpan.FromMilliseconds(100),
                MaxHedgedAttempts = 2
            })
            .Build();

        int callCount = 0;

        var result = await pipeline.ExecuteAsync(async (ctx, ct) =>
        {
            int attempt = Interlocked.Increment(ref callCount);
            if (attempt == 1)
                await Task.Delay(500, ct);  // primary is slow
            return $"response-{attempt}";
        }, CancellationToken.None);

        // Hedged request responds first
        Assert.Equal("response-2", result);
    }
}
```

---

## Step 1762: Chaos Engineering with Simmy

```csharp
// Install: Polly.Simmy (chaos testing library)
// Inject faults in test/staging to verify resilience

var chaosPipeline = new ResiliencePipelineBuilder<HttpResponseMessage>()
    // Your normal pipeline
    .AddRetry(new RetryStrategyOptions<HttpResponseMessage> { MaxRetryAttempts = 3 })
    .AddCircuitBreaker(new CircuitBreakerStrategyOptions<HttpResponseMessage>())
    // Inject chaos (only in non-production)
    .AddChaosLatency(new ChaosLatencyStrategyOptions
    {
        EnabledGenerator = args => ValueTask.FromResult(
            Environment.GetEnvironmentVariable("CHAOS_ENABLED") == "true"),
        InjectionRate = 0.1,                    // inject in 10% of requests
        Latency = TimeSpan.FromSeconds(3)       // add 3s latency
    })
    .AddChaosFault(new ChaosFaultStrategyOptions
    {
        EnabledGenerator = args => ValueTask.FromResult(
            Environment.GetEnvironmentVariable("CHAOS_ENABLED") == "true"),
        InjectionRate = 0.05,
        FaultGenerator = args => ValueTask.FromResult<Exception?>(
            new HttpRequestException("Chaos: injected fault"))
    })
    .AddChaosOutcome(new ChaosOutcomeStrategyOptions<HttpResponseMessage>
    {
        InjectionRate = 0.03,
        OutcomeGenerator = args => ValueTask.FromResult<Outcome<HttpResponseMessage>?>(
            Outcome.FromResult(new HttpResponseMessage(System.Net.HttpStatusCode.ServiceUnavailable)))
    })
    .Build();
```

---

## Step 1763: Real-World Resilience Configuration

```csharp
// Complete production configuration for an order service
// that calls: PaymentService, InventoryService, NotificationService

builder.Services.AddHttpClient<IPaymentService, PaymentService>(client =>
{
    client.BaseAddress = new Uri(builder.Configuration["Services:Payment"]!);
    client.Timeout = Timeout.InfiniteTimeSpan; // Polly manages timeout
})
.AddResilienceHandler("payment", (pipeline, context) =>
{
    pipeline
        .AddFallback(new FallbackStrategyOptions<HttpResponseMessage>
        {
            ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
                .Handle<BrokenCircuitException>(),
            FallbackAction = _ => Outcome.FromResultAsValueTask(
                new HttpResponseMessage(System.Net.HttpStatusCode.ServiceUnavailable))
        })
        .AddCircuitBreaker(new CircuitBreakerStrategyOptions<HttpResponseMessage>
        {
            FailureRatio = 0.5,
            MinimumThroughput = 20,
            SamplingDuration = TimeSpan.FromSeconds(30),
            BreakDuration = TimeSpan.FromSeconds(30)
        })
        .AddRetry(new RetryStrategyOptions<HttpResponseMessage>
        {
            MaxRetryAttempts = 3,
            Delay = TimeSpan.FromMilliseconds(300),
            BackoffType = DelayBackoffType.Exponential,
            UseJitter = true,
            ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
                .Handle<HttpRequestException>()
                .HandleResult(r => (int)r.StatusCode >= 500)
        })
        .AddTimeout(new TimeoutStrategyOptions
        {
            Timeout = TimeSpan.FromSeconds(8)
        });
});

// Inventory — read-heavy, can use hedging for low latency
builder.Services.AddHttpClient<IInventoryService, InventoryService>()
.AddResilienceHandler("inventory", pipeline =>
{
    pipeline
        .AddHedging(new HedgingStrategyOptions<HttpResponseMessage>
        {
            Delay = TimeSpan.FromMilliseconds(500),
            MaxHedgedAttempts = 1
        })
        .AddRetry(new RetryStrategyOptions<HttpResponseMessage>
        {
            MaxRetryAttempts = 2,
            Delay = TimeSpan.FromMilliseconds(200)
        })
        .AddTimeout(TimeSpan.FromSeconds(3));
});

// Notifications — fire-and-forget, don't fail the order if notifications fail
builder.Services.AddHttpClient<INotificationService, NotificationService>()
.AddResilienceHandler("notifications", pipeline =>
{
    pipeline
        .AddRetry(new RetryStrategyOptions<HttpResponseMessage>
        {
            MaxRetryAttempts = 5,
            Delay = TimeSpan.FromSeconds(1),
            BackoffType = DelayBackoffType.Exponential
        })
        .AddTimeout(TimeSpan.FromSeconds(2));
});
```

---

## Step 1764: Resilience Patterns Decision Matrix

```
Request type           → Recommended strategies
─────────────────────────────────────────────────────────
Synchronous HTTP call  → Timeout + Retry + Circuit Breaker
Read-heavy query       → Hedging + Timeout + Retry
Write operation        → Timeout + Retry (idempotent writes only)
Background job         → Retry + Circuit Breaker (no fallback)
Payment/financial      → Timeout only — NEVER retry non-idempotent payments
Notifications          → Retry with high max attempts, long delays
Database queries       → Timeout + Retry (SQL transient errors only)
Message consumers      → Retry + DLQ (MassTransit/Wolverine handles this)
```

```csharp
// Golden rule: NEVER retry non-idempotent operations without checking first
// For payments: use idempotency keys instead of retry

public class PaymentService
{
    public async Task<PaymentResult> ProcessAsync(PaymentRequest req, CancellationToken ct)
    {
        // Idempotency key prevents double-charging even if retried
        var idempotencyKey = $"order:{req.OrderId}:payment";

        var response = await _client.PostAsJsonAsync("/charge", new
        {
            Amount = req.Amount,
            CustomerId = req.CustomerId,
            IdempotencyKey = idempotencyKey  // server deduplicates on this
        }, ct);

        // Now retry is safe — server will return the original result
        return await response.Content.ReadFromJsonAsync<PaymentResult>(ct) ?? throw new();
    }
}
```

| Strategy | Use When |
|---|---|
| Retry | Transient failures, idempotent operations |
| Circuit Breaker | Protecting from cascading failures |
| Timeout | Preventing thread pool exhaustion |
| Fallback | Providing degraded mode / cached response |
| Hedging | Low-latency critical reads with multiple replicas |
| Bulkhead | Isolating a slow dependency from affecting others |
| Rate Limiter | Protecting downstream from bursts |

### What Next?

- **Part 71**: Multi-Tenancy Architecture — Row-level security, schema-per-tenant, API key tenancy
- **Part 72**: Domain-Driven Design Advanced — Aggregates, Bounded Contexts, Anti-Corruption Layer
- **Part 73**: Testing Strategies — Contract testing with Pact, mutation testing, load testing with k6
