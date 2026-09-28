# Part 99: Final Architecture Patterns และ System Design at Scale

## Steps 2213-2228

Architecture patterns สำหรับ production systems ขนาดใหญ่: Clean Architecture, CQRS, Event Sourcing, Saga Pattern, Strangler Fig, และ system design considerations

---

## Step 2213: Production Architecture Overview

```
World-Class .NET Architecture
================================

┌─────────────────────────────────────────────────────────────────┐
│                    API Gateway / BFF                             │
│           (Rate Limiting, Auth, Routing, Aggregation)           │
└────────────────────────────┬────────────────────────────────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│  Order Service│   │Catalog Service│   │ User Service  │
│   CQRS+ES    │   │   Read Model  │   │  Auth/Identity│
│  PostgreSQL   │   │ Elasticsearch │   │  PostgreSQL   │
└───────┬───────┘   └───────────────┘   └───────────────┘
        │
        │ Domain Events → Message Bus
        ▼
┌─────────────────────────────────────────────────────────┐
│              Message Bus (RabbitMQ / Service Bus)        │
│         Pub/Sub, Dead Letter, Retry, Ordering           │
└─────────────────┬────────────────────────────────────────┘
                  │
      ┌───────────┼───────────┐
      │           │           │
      ▼           ▼           ▼
┌──────────┐ ┌──────────┐ ┌──────────┐
│Inventory │ │Payment   │ │Notif.    │
│Processor │ │Processor │ │Processor │
└──────────┘ └──────────┘ └──────────┘

Cross-cutting:
- .NET Aspire (orchestration + observability)
- OpenTelemetry (traces, metrics, logs)
- Azure Key Vault / Secrets Management
- Feature Flags (Azure App Configuration)
```

---

## Step 2214: Clean Architecture - Complete Implementation

```csharp
/*
Solution Structure:
ECommerce/
├── src/
│   ├── ECommerce.Domain/           ← Core business rules, no dependencies
│   │   ├── Entities/
│   │   ├── ValueObjects/
│   │   ├── Events/
│   │   ├── Exceptions/
│   │   └── Specifications/
│   │
│   ├── ECommerce.Application/      ← Use cases, orchestrates domain
│   │   ├── Commands/
│   │   ├── Queries/
│   │   ├── Handlers/
│   │   ├── Behaviors/              ← MediatR pipeline behaviors
│   │   ├── Interfaces/             ← Abstractions for infrastructure
│   │   └── DTOs/
│   │
│   ├── ECommerce.Infrastructure/   ← External concerns
│   │   ├── Persistence/
│   │   │   ├── DbContext/
│   │   │   ├── Repositories/
│   │   │   ├── Migrations/
│   │   │   └── Configurations/
│   │   ├── Messaging/
│   │   ├── Caching/
│   │   ├── ExternalServices/
│   │   └── DependencyInjection.cs
│   │
│   └── ECommerce.Api/              ← Presentation layer
│       ├── Endpoints/
│       ├── Filters/
│       ├── Middleware/
│       └── Program.cs
│
└── tests/
    ├── ECommerce.Domain.Tests/
    ├── ECommerce.Application.Tests/
    └── ECommerce.Integration.Tests/
*/

// Dependency Rule: inner layers know nothing about outer layers
// Domain ← Application ← Infrastructure
//                     ← API
```

---

## Step 2215: CQRS with MediatR - Complete Pattern

```csharp
// Command: mutates state
public record PlaceOrderCommand(
    Guid CustomerId,
    List<OrderItemRequest> Items,
    AddressDto ShippingAddress) : ICommand<PlaceOrderResult>;

public record PlaceOrderResult(Guid OrderId, decimal Total);

// Command handler
public class PlaceOrderCommandHandler(
    IOrderRepository repo,
    IUnitOfWork uow,
    IProductService productService,
    IPublisher publisher,
    ILogger<PlaceOrderCommandHandler> logger)
    : ICommandHandler<PlaceOrderCommand, PlaceOrderResult>
{
    public async Task<PlaceOrderResult> Handle(
        PlaceOrderCommand command, CancellationToken ct)
    {
        // 1. Validate products exist and have stock
        var productIds = command.Items.Select(i => i.ProductId).ToList();
        var products = await productService.GetByIdsAsync(productIds, ct);

        if (products.Count != productIds.Count)
            throw new NotFoundException("One or more products not found");

        // 2. Create aggregate
        var items = command.Items.Select(i =>
        {
            var product = products.First(p => p.Id == i.ProductId);
            return new OrderItemRequest(i.ProductId, product.Name, i.Quantity, product.Price);
        }).ToList();

        var shippingAddress = new Address(
            command.ShippingAddress.Street,
            command.ShippingAddress.City,
            command.ShippingAddress.PostalCode,
            command.ShippingAddress.Country);

        var order = Order.Place(new CustomerId(command.CustomerId), items, shippingAddress);

        // 3. Persist
        await repo.AddAsync(order, ct);
        await uow.SaveChangesAsync(ct);

        // 4. Domain events automatically published via post-save dispatcher

        logger.LogInformation("Order {OrderId} placed for customer {CustomerId}",
            order.Id, command.CustomerId);

        return new PlaceOrderResult(order.Id.Value, order.Total.Amount);
    }
}

// Query: reads state, no mutation
public record GetOrderByIdQuery(Guid OrderId) : IQuery<OrderDto?>;

public class GetOrderByIdQueryHandler(IReadDbContext db)
    : IQueryHandler<GetOrderByIdQuery, OrderDto?>
{
    public async Task<OrderDto?> Handle(GetOrderByIdQuery query, CancellationToken ct)
    {
        return await db.Orders
            .AsNoTracking()
            .Where(o => o.Id == query.OrderId)
            .Select(o => new OrderDto(
                o.Id,
                o.CustomerId,
                o.Status,
                o.Total.Amount,
                o.Total.Currency,
                o.Items.Select(i => new OrderItemDto(i.ProductName, i.Quantity, i.UnitPrice)).ToList(),
                o.CreatedAt))
            .FirstOrDefaultAsync(ct);
    }
}

// MediatR Pipeline Behaviors
public class ValidationBehavior<TRequest, TResponse>(
    IEnumerable<IValidator<TRequest>> validators)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    public async Task<TResponse> Handle(
        TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        if (!validators.Any()) return await next();

        var context = new ValidationContext<TRequest>(request);
        var results = await Task.WhenAll(validators.Select(v => v.ValidateAsync(context, ct)));
        var failures = results.SelectMany(r => r.Errors).Where(f => f is not null).ToList();

        if (failures.Count != 0) throw new ValidationException(failures);

        return await next();
    }
}

public class LoggingBehavior<TRequest, TResponse>(
    ILogger<LoggingBehavior<TRequest, TResponse>> logger)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    public async Task<TResponse> Handle(
        TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        var requestName = typeof(TRequest).Name;
        logger.LogInformation("Handling {Request}", requestName);
        var sw = Stopwatch.StartNew();

        try
        {
            var response = await next();
            logger.LogInformation("{Request} handled in {Duration}ms", requestName, sw.ElapsedMilliseconds);
            return response;
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "{Request} failed after {Duration}ms", requestName, sw.ElapsedMilliseconds);
            throw;
        }
    }
}
```

---

## Step 2216: Event Sourcing Pattern

```csharp
// Event-sourced aggregate
public abstract class EventSourcedAggregate
{
    private readonly List<IDomainEvent> _uncommittedEvents = [];
    public int Version { get; private set; }

    public IReadOnlyList<IDomainEvent> UncommittedEvents => _uncommittedEvents;

    protected void RaiseEvent(IDomainEvent @event)
    {
        Apply(@event);
        _uncommittedEvents.Add(@event);
        Version++;
    }

    public void LoadFromHistory(IEnumerable<IDomainEvent> history)
    {
        foreach (var @event in history)
        {
            Apply(@event);
            Version++;
        }
    }

    protected abstract void Apply(IDomainEvent @event);

    public void MarkEventsAsCommitted() => _uncommittedEvents.Clear();
}

// Event-sourced Order
public class Order : EventSourcedAggregate
{
    public Guid Id { get; private set; }
    public Guid CustomerId { get; private set; }
    public string Status { get; private set; } = "Draft";
    public List<OrderItem> Items { get; private set; } = [];
    public decimal Total { get; private set; }

    private Order() { }

    public static Order Place(Guid customerId, List<OrderItemRequest> items)
    {
        var order = new Order();
        order.RaiseEvent(new OrderPlacedEvent(
            Guid.NewGuid(), customerId, items,
            items.Sum(i => i.Quantity * i.UnitPrice),
            DateTimeOffset.UtcNow));
        return order;
    }

    public void Confirm()
    {
        if (Status != "Pending")
            throw new InvalidOperationException($"Cannot confirm order in {Status} status");
        RaiseEvent(new OrderConfirmedEvent(Id, DateTimeOffset.UtcNow));
    }

    public void Cancel(string reason)
    {
        if (Status is "Shipped" or "Delivered")
            throw new InvalidOperationException($"Cannot cancel order in {Status} status");
        RaiseEvent(new OrderCancelledEvent(Id, reason, DateTimeOffset.UtcNow));
    }

    protected override void Apply(IDomainEvent @event)
    {
        switch (@event)
        {
            case OrderPlacedEvent e:
                Id = e.OrderId;
                CustomerId = e.CustomerId;
                Status = "Pending";
                Items = e.Items.Select(i => new OrderItem(i.ProductId, i.Quantity, i.UnitPrice)).ToList();
                Total = e.Total;
                break;
            case OrderConfirmedEvent:
                Status = "Confirmed";
                break;
            case OrderCancelledEvent:
                Status = "Cancelled";
                break;
        }
    }
}

// Event Store
public interface IEventStore
{
    Task AppendAsync(Guid streamId, IEnumerable<IDomainEvent> events, int expectedVersion, CancellationToken ct = default);
    Task<IEnumerable<IDomainEvent>> ReadAsync(Guid streamId, CancellationToken ct = default);
    Task<IEnumerable<IDomainEvent>> ReadAsync(Guid streamId, int fromVersion, CancellationToken ct = default);
}

// Event-Sourced Repository
public class EventSourcedOrderRepository(IEventStore eventStore, IPublisher publisher)
    : IOrderRepository
{
    public async Task<Order?> GetByIdAsync(Guid id, CancellationToken ct = default)
    {
        var events = await eventStore.ReadAsync(id, ct);
        if (!events.Any()) return null;

        var order = new Order();
        order.LoadFromHistory(events);
        return order;
    }

    public async Task SaveAsync(Order order, CancellationToken ct = default)
    {
        var events = order.UncommittedEvents;
        if (!events.Any()) return;

        await eventStore.AppendAsync(order.Id, events, order.Version - events.Count, ct);

        // Publish events for projections
        foreach (var @event in events)
            await publisher.Publish(@event, ct);

        order.MarkEventsAsCommitted();
    }
}
```

---

## Step 2217: Read Model Projections

```csharp
// Event-sourced write side → denormalized read model
// Order read model projection

public class OrderSummaryProjection :
    INotificationHandler<OrderPlacedEvent>,
    INotificationHandler<OrderConfirmedEvent>,
    INotificationHandler<OrderCancelledEvent>,
    INotificationHandler<OrderShippedEvent>
{
    private readonly IReadDbContext _readDb;

    public async Task Handle(OrderPlacedEvent @event, CancellationToken ct)
    {
        var summary = new OrderSummaryReadModel
        {
            Id = @event.OrderId,
            CustomerId = @event.CustomerId,
            Status = "Pending",
            Total = @event.Total,
            ItemCount = @event.Items.Count,
            CreatedAt = @event.OccurredAt
        };
        await _readDb.OrderSummaries.AddAsync(summary, ct);
        await _readDb.SaveChangesAsync(ct);
    }

    public async Task Handle(OrderConfirmedEvent @event, CancellationToken ct)
    {
        await _readDb.OrderSummaries
            .Where(o => o.Id == @event.OrderId)
            .ExecuteUpdateAsync(s => s
                .SetProperty(o => o.Status, "Confirmed")
                .SetProperty(o => o.LastUpdatedAt, @event.OccurredAt), ct);
    }

    public async Task Handle(OrderCancelledEvent @event, CancellationToken ct)
    {
        await _readDb.OrderSummaries
            .Where(o => o.Id == @event.OrderId)
            .ExecuteUpdateAsync(s => s
                .SetProperty(o => o.Status, "Cancelled")
                .SetProperty(o => o.LastUpdatedAt, @event.OccurredAt), ct);
    }

    public async Task Handle(OrderShippedEvent @event, CancellationToken ct)
    {
        await _readDb.OrderSummaries
            .Where(o => o.Id == @event.OrderId)
            .ExecuteUpdateAsync(s => s
                .SetProperty(o => o.Status, "Shipped")
                .SetProperty(o => o.TrackingNumber, @event.TrackingNumber)
                .SetProperty(o => o.LastUpdatedAt, @event.OccurredAt), ct);
    }
}

// Elasticsearch projection for full-text search
public class OrderSearchProjection : INotificationHandler<OrderPlacedEvent>
{
    private readonly IElasticClient _elastic;

    public async Task Handle(OrderPlacedEvent @event, CancellationToken ct)
    {
        var doc = new OrderSearchDocument
        {
            Id = @event.OrderId.ToString(),
            CustomerId = @event.CustomerId.ToString(),
            Status = "Pending",
            Total = @event.Total,
            Items = @event.Items.Select(i => i.ProductName).ToList(),
            CreatedAt = @event.OccurredAt
        };

        await _elastic.IndexAsync(doc, i => i.Index("orders"), ct);
    }
}
```

---

## Step 2218: Saga Pattern - Distributed Transactions

```csharp
// Choreography Saga (event-driven, no central coordinator)

// OrderService publishes: OrderPlaced
// InventoryService listens → reserves stock → publishes: StockReserved
// PaymentService listens → charges card → publishes: PaymentProcessed
// FulfillmentService listens → creates shipment → publishes: OrderShipped
// Each step handles failure by publishing compensation events

// Orchestration Saga (central coordinator)
// Using MassTransit StateMachine (see Part 89 for full example)

// The key patterns for saga resilience:
public class OrderFulfillmentSaga : MassTransitStateMachine<OrderFulfillmentState>
{
    // State machine handles:
    // 1. Compensating transactions on failure
    // 2. Timeouts for external service responses
    // 3. Idempotency (duplicate event handling)
    // 4. Dead letter queue for unrecoverable failures
}

// Idempotent saga step
public class ReserveStockConsumer(
    IInventoryService inventory,
    IPublishEndpoint publish,
    ILogger<ReserveStockConsumer> logger)
    : IConsumer<OrderPlaced>
{
    public async Task Consume(ConsumeContext<OrderPlaced> context)
    {
        var @event = context.Message;

        // Idempotency check - already processed?
        if (await inventory.IsReservationExistsAsync(@event.OrderId))
        {
            logger.LogInformation("Stock reservation already exists for order {OrderId}", @event.OrderId);
            return;
        }

        try
        {
            await inventory.ReserveStockAsync(@event.OrderId, @event.Items);
            await publish.Publish(new StockReserved
            {
                OrderId = @event.OrderId,
                ReservationId = Guid.NewGuid(),
                ReservedAt = DateTimeOffset.UtcNow
            });
        }
        catch (InsufficientStockException ex)
        {
            await publish.Publish(new StockReservationFailed
            {
                OrderId = @event.OrderId,
                Reason = ex.Message
            });
        }
    }
}
```

---

## Step 2219: API Gateway / BFF Pattern

```csharp
// Backend for Frontend (BFF) aggregates multiple service calls
// Package: Microsoft.ReverseProxy (YARP)

// yarp-config.json
{
  "ReverseProxy": {
    "Routes": {
      "catalog-route": {
        "ClusterId": "catalog-cluster",
        "Match": {
          "Path": "/api/catalog/{**catch-all}"
        },
        "Transforms": [
          { "PathRemovePrefix": "/api/catalog" }
        ]
      },
      "orders-route": {
        "ClusterId": "orders-cluster",
        "Match": {
          "Path": "/api/orders/{**catch-all}"
        }
      }
    },
    "Clusters": {
      "catalog-cluster": {
        "Destinations": {
          "catalog1": { "Address": "http://catalog-api:8080" },
          "catalog2": { "Address": "http://catalog-api-2:8080" }
        },
        "LoadBalancingPolicy": "RoundRobin",
        "HealthCheck": {
          "Active": {
            "Enabled": true,
            "Interval": "00:00:10",
            "Timeout": "00:00:10",
            "Policy": "ConsecutiveFailures",
            "Path": "/health/ready"
          }
        }
      }
    }
  }
}

// Custom BFF aggregation endpoint
app.MapGet("/api/bff/home", async (
    ICatalogClient catalog,
    IOrderClient orders,
    IRecommendationClient recommendations,
    ICurrentUserService currentUser,
    CancellationToken ct) =>
{
    // Parallel calls to multiple services
    var (featuredProducts, recentOrders, recommended) = await (
        catalog.GetFeaturedProductsAsync(ct),
        orders.GetRecentOrdersAsync(currentUser.UserId, 5, ct),
        recommendations.GetRecommendationsAsync(currentUser.UserId, 10, ct)
    ).WhenAll();

    return TypedResults.Ok(new HomePageDto
    {
        FeaturedProducts = featuredProducts,
        RecentOrders = recentOrders,
        Recommended = recommended
    });
});

// Extension for tuple WhenAll
public static async Task<(T1, T2, T3)> WhenAll<T1, T2, T3>(
    this (Task<T1> t1, Task<T2> t2, Task<T3> t3) tasks)
{
    await Task.WhenAll(tasks.t1, tasks.t2, tasks.t3);
    return (tasks.t1.Result, tasks.t2.Result, tasks.t3.Result);
}
```

---

## Step 2220: Strangler Fig Pattern - Legacy Migration

```csharp
// Incrementally migrate from legacy monolith to microservices

// 1. Route new features to new service, old features to monolith
// Using YARP for routing

app.MapReverseProxy(proxyPipeline =>
{
    proxyPipeline.UseRouting();
    proxyPipeline.Use(async (context, next) =>
    {
        var path = context.Request.Path.Value ?? "";

        // New order service handles v2+ paths
        if (path.StartsWith("/api/v2/orders") || path.StartsWith("/api/v3/orders"))
        {
            context.Request.RouteValues["clusterId"] = "new-orders-cluster";
        }
        else if (path.StartsWith("/api/orders"))
        {
            // Legacy monolith handles v1
            context.Request.RouteValues["clusterId"] = "legacy-cluster";
        }

        await next();
    });
});

// 2. Anti-Corruption Layer between new and legacy
public class LegacyOrderAdapter(LegacyOrderSystemClient legacy) : IOrderRepository
{
    public async Task<Order?> GetByIdAsync(Guid id, CancellationToken ct)
    {
        var legacyOrder = await legacy.GetOrderAsync(id.ToString(), ct);
        if (legacyOrder is null) return null;

        // Translate legacy model to domain model
        return new Order
        {
            Id = new OrderId(Guid.Parse(legacyOrder.order_id)),
            CustomerId = new CustomerId(Guid.Parse(legacyOrder.cust_id)),
            Status = MapLegacyStatus(legacyOrder.status_code),
            Total = new Money(legacyOrder.total_amount, legacyOrder.currency_code)
        };
    }

    private static string MapLegacyStatus(int statusCode) => statusCode switch
    {
        1 => "Pending",
        2 => "Confirmed",
        3 => "Shipped",
        4 => "Delivered",
        5 => "Cancelled",
        _ => "Unknown"
    };
}
```

---

## Step 2221: Resilience Patterns

```csharp
// Comprehensive resilience with Polly v8
builder.Services.AddResiliencePipeline("order-service", builder =>
{
    builder
        // Timeout per attempt
        .AddTimeout(new TimeoutStrategyOptions
        {
            Timeout = TimeSpan.FromSeconds(5)
        })
        // Retry with exponential backoff + jitter
        .AddRetry(new RetryStrategyOptions
        {
            MaxRetryAttempts = 3,
            BackoffType = DelayBackoffType.Exponential,
            UseJitter = true,
            Delay = TimeSpan.FromMilliseconds(500),
            ShouldHandle = new PredicateBuilder()
                .Handle<HttpRequestException>()
                .Handle<TimeoutRejectedException>(),
            OnRetry = args =>
            {
                logger.LogWarning(args.Outcome.Exception,
                    "Retry {Attempt} after {Delay}ms",
                    args.AttemptNumber, args.RetryDelay.TotalMilliseconds);
                return ValueTask.CompletedTask;
            }
        })
        // Circuit breaker
        .AddCircuitBreaker(new CircuitBreakerStrategyOptions
        {
            SamplingDuration = TimeSpan.FromSeconds(30),
            MinimumThroughput = 10,
            FailureRatio = 0.5,
            BreakDuration = TimeSpan.FromSeconds(30),
            OnOpened = args =>
            {
                logger.LogError("Circuit opened for {Duration}", args.BreakDuration);
                return ValueTask.CompletedTask;
            },
            OnClosed = _ =>
            {
                logger.LogInformation("Circuit closed - service recovering");
                return ValueTask.CompletedTask;
            }
        })
        // Total timeout across all retries
        .AddTimeout(new TimeoutStrategyOptions
        {
            Timeout = TimeSpan.FromSeconds(30),
            TimeoutGenerator = args => ValueTask.FromResult(TimeSpan.FromSeconds(30))
        });
});

// Bulkhead pattern - limit concurrent calls
builder.Services.AddResiliencePipeline("external-payment", builder =>
{
    builder.AddConcurrencyLimiter(new ConcurrencyLimiterStrategyOptions
    {
        PermitLimit = 20,  // Max 20 concurrent payment calls
        QueueLimit = 50
    });
});

// Fallback pattern
builder.Services.AddResiliencePipeline<Product[]>("catalog-with-fallback", builder =>
{
    builder.AddFallback(new FallbackStrategyOptions<Product[]>
    {
        FallbackAction = _ => ValueTask.FromResult(Array.Empty<Product>()),
        ShouldHandle = new PredicateBuilder<Product[]>()
            .Handle<Exception>()
    });
});
```

---

## Step 2222: Performance Optimization Patterns

```csharp
// 1. Compiled queries
private static readonly Func<AppDbContext, Guid, Task<Order?>> CompiledGetOrder =
    EF.CompileAsyncQuery((AppDbContext db, Guid id) =>
        db.Orders
            .AsNoTracking()
            .Include(o => o.Items)
            .FirstOrDefault(o => o.Id == id));

// Usage
var order = await CompiledGetOrder(db, orderId);

// 2. Bulk operations with EFCore.BulkExtensions
await db.BulkInsertAsync(orders, options =>
{
    options.BatchSize = 1000;
    options.SetOutputIdentity = true;
    options.PreserveInsertOrder = true;
});

// 3. Projection to DTOs (avoid loading entire entity)
var summaries = await db.Orders
    .AsNoTracking()
    .Where(o => o.CustomerId == customerId)
    .Select(o => new OrderSummaryDto(o.Id, o.Status, o.Total.Amount, o.CreatedAt))
    .ToListAsync(ct);

// 4. Pagination with keyset (cursor-based)
// Offset pagination: SKIP N (slow for large N)
// Keyset pagination: WHERE Id > lastId (always O(1))
public async Task<CursorPage<OrderDto>> GetOrdersAsync(
    Guid? afterId, int pageSize, CancellationToken ct)
{
    var query = db.Orders.AsNoTracking().OrderBy(o => o.Id);

    if (afterId.HasValue)
        query = (IOrderedQueryable<Order>)query.Where(o => o.Id > afterId.Value);

    var items = await query
        .Take(pageSize + 1)
        .Select(o => new OrderDto(o.Id, o.Status, o.Total.Amount, o.CreatedAt))
        .ToListAsync(ct);

    var hasMore = items.Count > pageSize;
    if (hasMore) items.RemoveAt(items.Count - 1);

    return new CursorPage<OrderDto>
    {
        Items = items,
        NextCursor = hasMore ? items.Last().Id.ToString() : null
    };
}

// 5. Response compression
builder.Services.AddResponseCompression(options =>
{
    options.EnableForHttps = true;
    options.Providers.Add<BrotliCompressionProvider>();
    options.Providers.Add<GzipCompressionProvider>();
    options.MimeTypes = ResponseCompressionDefaults.MimeTypes
        .Concat(["application/json", "text/json"]);
});
builder.Services.Configure<BrotliCompressionProviderOptions>(options =>
    options.Level = CompressionLevel.Optimal);

// 6. Response caching headers
app.Use(async (context, next) =>
{
    await next();
    if (context.Response.StatusCode == 200
        && context.Request.Method == HttpMethods.Get)
    {
        context.Response.Headers.CacheControl = "public, max-age=300, stale-while-revalidate=600";
        context.Response.Headers.Vary = "Accept-Encoding, Accept";
    }
});
```

---

## Step 2223: Security Best Practices

```csharp
// Comprehensive security configuration
var builder = WebApplication.CreateBuilder(args);

// HTTPS
builder.WebHost.UseKestrel(options =>
{
    options.ConfigureHttpsDefaults(https =>
    {
        https.SslProtocols = SslProtocols.Tls12 | SslProtocols.Tls13;
    });
});

// Security headers
builder.Services.AddHsts(options =>
{
    options.MaxAge = TimeSpan.FromDays(365);
    options.IncludeSubDomains = true;
    options.Preload = true;
});

var app = builder.Build();

// Security middleware
app.UseHsts();
app.UseHttpsRedirection();

app.Use(async (context, next) =>
{
    context.Response.Headers["X-Content-Type-Options"] = "nosniff";
    context.Response.Headers["X-Frame-Options"] = "DENY";
    context.Response.Headers["X-XSS-Protection"] = "1; mode=block";
    context.Response.Headers["Referrer-Policy"] = "strict-origin-when-cross-origin";
    context.Response.Headers["Permissions-Policy"] = "geolocation=(), microphone=()";
    context.Response.Headers["Content-Security-Policy"] =
        "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'";
    await next();
});

// CORS - explicit allowlist
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowedOrigins", policy =>
    {
        policy.WithOrigins(
            builder.Configuration.GetSection("AllowedOrigins").Get<string[]>() ?? [])
        .AllowAnyMethod()
        .AllowAnyHeader()
        .AllowCredentials()
        .SetPreflightMaxAge(TimeSpan.FromMinutes(10));
    });
});

// Input sanitization
public class SanitizationFilter : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context,
        EndpointFilterDelegate next)
    {
        foreach (var arg in context.Arguments.OfType<string>())
        {
            if (ContainsSqlInjection(arg) || ContainsXss(arg))
                return TypedResults.BadRequest("Invalid input detected");
        }
        return await next(context);
    }

    private static bool ContainsSqlInjection(string input) =>
        input.Contains("'") || input.Contains(";--") || input.ToUpper().Contains("DROP TABLE");

    private static bool ContainsXss(string input) =>
        input.Contains("<script") || input.Contains("javascript:") || input.Contains("onerror=");
}
```

---

## Step 2224: Multi-Tenancy Patterns

```csharp
// Three multi-tenancy models:
// 1. Database per tenant (strongest isolation)
// 2. Schema per tenant (medium isolation)
// 3. Shared schema with TenantId column (weakest isolation, most cost-effective)

// Shared schema with global query filter (most common)
public class MultiTenantDbContext(
    DbContextOptions options,
    ITenantContextAccessor tenantAccessor)
    : DbContext(options)
{
    private readonly Guid _tenantId = tenantAccessor.TenantId;

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        foreach (var entityType in modelBuilder.Model.GetEntityTypes())
        {
            if (typeof(ITenantEntity).IsAssignableFrom(entityType.ClrType))
            {
                modelBuilder.Entity(entityType.ClrType)
                    .HasQueryFilter(BuildTenantFilter(entityType.ClrType));
            }
        }
    }

    private LambdaExpression BuildTenantFilter(Type entityType)
    {
        var parameter = Expression.Parameter(entityType, "e");
        var property = Expression.Property(parameter, nameof(ITenantEntity.TenantId));
        var tenantId = Expression.Constant(_tenantId);
        var equal = Expression.Equal(property, tenantId);
        return Expression.Lambda(equal, parameter);
    }

    public override int SaveChanges()
    {
        SetTenantId();
        return base.SaveChanges();
    }

    public override Task<int> SaveChangesAsync(CancellationToken ct = default)
    {
        SetTenantId();
        return base.SaveChangesAsync(ct);
    }

    private void SetTenantId()
    {
        foreach (var entry in ChangeTracker.Entries<ITenantEntity>()
            .Where(e => e.State == EntityState.Added))
        {
            entry.Entity.TenantId = _tenantId;
        }
    }
}

// Tenant resolution middleware
public class TenantResolutionMiddleware(RequestDelegate next)
{
    public async Task InvokeAsync(HttpContext context, ITenantRepository tenantRepo)
    {
        // Resolve tenant from subdomain: tenant1.api.example.com
        var host = context.Request.Host.Host;
        var subdomain = host.Split('.')[0];

        var tenant = await tenantRepo.GetBySubdomainAsync(subdomain);
        if (tenant is null)
        {
            context.Response.StatusCode = 400;
            await context.Response.WriteAsJsonAsync(new { error = "Unknown tenant" });
            return;
        }

        context.Items["TenantId"] = tenant.Id;
        context.Items["Tenant"] = tenant;

        await next(context);
    }
}
```

---

## Step 2225: Monitoring และ SRE Practices

```csharp
// SLO (Service Level Objectives) implementation
// Target: 99.9% availability, p99 latency < 500ms

// Custom SLO metrics
public class SloMetrics
{
    private readonly Counter<long> _requestsTotal;
    private readonly Counter<long> _errorsTotal;
    private readonly Histogram<double> _requestDuration;

    public SloMetrics(IMeterFactory meterFactory)
    {
        var meter = meterFactory.Create("ECommerce.SLO");
        _requestsTotal = meter.CreateCounter<long>("slo_requests_total");
        _errorsTotal = meter.CreateCounter<long>("slo_errors_total");
        _requestDuration = meter.CreateHistogram<double>(
            "slo_request_duration_seconds",
            unit: "s",
            advice: new InstrumentAdvice<double>
            {
                HistogramBucketBoundaries = [0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0]
            });
    }
}

// Error budget calculation:
// Error budget = 1 - SLO = 0.1% = 43.8 minutes/month
// Current error rate tracked by Prometheus
// Alert when error budget is being consumed too fast

// Prometheus alert for SLO burn rate
/*
- alert: SLOHighBurnRate
  expr: |
    (
      rate(slo_errors_total[1h]) / rate(slo_requests_total[1h])
    ) > 14.4 * (1 - 0.999)
  for: 2m
  annotations:
    summary: "SLO error budget burning too fast"
    description: "At current rate, 2% of monthly error budget consumed in 1h"
*/

// Graceful shutdown
var lifetime = app.Lifetime;
lifetime.ApplicationStopping.Register(() =>
{
    logger.LogInformation("Application stopping - waiting for requests to complete");
    // Give in-flight requests time to complete
    Thread.Sleep(TimeSpan.FromSeconds(5));
});

app.UseShutdownTimeout(TimeSpan.FromSeconds(30));
```

---

## Step 2226: Load Testing with NBomber

```csharp
// LoadTests/OrderServiceLoadTest.cs
public class OrderServiceLoadTest
{
    [Fact]
    public void RunLoadTest()
    {
        var httpClient = new HttpClient
        {
            BaseAddress = new Uri("https://localhost:7001")
        };

        var scenario = Scenario.Create("place_order", async context =>
        {
            var customerId = Guid.NewGuid();
            var request = new
            {
                customerId,
                items = new[]
                {
                    new { productId = Guid.NewGuid(), quantity = 1 }
                }
            };

            var response = await httpClient.PostAsJsonAsync("/api/v1/orders", request);
            return response.IsSuccessStatusCode
                ? Response.Ok(statusCode: (int)response.StatusCode)
                : Response.Fail(statusCode: (int)response.StatusCode, error: response.StatusCode.ToString());
        })
        .WithLoadSimulations(
            // Warm up: 10 req/s for 30s
            Simulation.RampingConstant(10, TimeSpan.FromSeconds(30)),
            // Steady state: 100 req/s for 2 minutes
            Simulation.KeepConstant(100, TimeSpan.FromMinutes(2)),
            // Spike: 500 req/s for 30s
            Simulation.RampingConstant(500, TimeSpan.FromSeconds(30)),
            // Cool down
            Simulation.RampingConstant(10, TimeSpan.FromSeconds(30)));

        NBomberRunner
            .RegisterScenarios(scenario)
            .WithWorkerPlugins(new HttpMetricsPlugin([HttpVersion.Version2]))
            .WithReportFormats(ReportFormat.Html, ReportFormat.Csv)
            .Run();
    }
}
```

---

## Step 2227: Architecture Decision Records (ADR)

```markdown
# ADR-001: Use CQRS with MediatR for Command/Query Separation

## Status: Accepted

## Context
Our system has complex read and write requirements. Reads need to be highly optimized 
for reporting and dashboards. Writes need strong consistency and domain validation.

## Decision
Use CQRS pattern with MediatR for command/query separation:
- Commands use domain model with full validation
- Queries use optimized read models (no-tracking, projections)
- MediatR pipeline behaviors for cross-cutting concerns

## Consequences
Positive:
- Read and write models can be optimized independently
- Clear separation of concerns
- Easy to add cross-cutting via pipeline behaviors

Negative:
- More code to write (separate handlers)
- Eventual consistency for read models

---

# ADR-002: PostgreSQL as Primary Database with Read Replicas

## Status: Accepted

## Context
Need ACID transactions for orders, high read throughput for product catalog.

## Decision
- PostgreSQL for transactional data
- Read replicas for query-heavy operations
- ElasticSearch for full-text search
- Redis for caching

## Consequences
- Strong consistency for writes
- Eventual consistency for read replicas (acceptable: <1s lag)
- Operational complexity of replica management
```

---

## Step 2228: Architecture Checklist for Production

```
Production Readiness Checklist
================================

Application:
□ Health checks (liveness + readiness)
□ Graceful shutdown with timeout
□ Structured logging with correlation IDs
□ OpenTelemetry traces + metrics + logs
□ Global error handling → Problem Details
□ Input validation at all entry points
□ Security headers
□ Rate limiting
□ CORS properly configured
□ JWT with algorithm allowlist + expiry
□ HTTPS-only with HSTS

Database:
□ Connection pooling configured
□ Optimistic concurrency on write operations
□ Database migrations automated (CI/CD)
□ Read replicas for heavy queries
□ Slow query monitoring
□ Index strategy reviewed
□ Connection string in secrets vault

Resilience:
□ Retry with exponential backoff + jitter
□ Circuit breaker for external calls
□ Timeouts on all outbound calls
□ Bulkhead isolation for critical paths
□ Fallback values where appropriate

Infrastructure:
□ Auto-scaling configured (HPA)
□ Resource limits + requests set
□ Pod disruption budgets
□ Topology spread constraints
□ Readiness probe prevents traffic to unready pods
□ Image vulnerabilities scanned
□ Secrets in vault (not env vars or config files)

Operations:
□ Dashboards for key business metrics
□ Alerts for SLO violations
□ Runbooks documented
□ On-call rotation established
□ Incident response playbook
□ Disaster recovery tested
□ Backup and restore verified
□ Blue/Green or Canary deployment
□ Feature flags for risk mitigation

Security:
□ OWASP Top 10 review
□ Dependency scanning (Snyk/Dependabot)
□ Container scanning (Trivy)
□ SAST (CodeQL)
□ Secrets scanning
□ Penetration testing
□ Data classification and encryption at rest
```
