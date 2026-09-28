# Part 89: Message-Driven Architecture — Outbox, Inbox & Advanced Patterns

## Steps 2053–2068

---

## Step 2053: Transactional Outbox Pattern

The Outbox pattern ensures events are published exactly-once even if the application crashes between saving to DB and publishing to the message broker.

```
Problem Without Outbox:
┌──────────┐    Save order    ┌─────┐
│  Handler │ ─────────────── >│ DB  │  ✓ Saved
│          │    Crash here!   └─────┘
│          │    Publish msg   ┌─────────┐
│          │ ─── × ──────── > │ Broker  │  ✗ Never published!
└──────────┘                  └─────────┘

Solution — Transactional Outbox:
┌──────────┐  Save order +   ┌─────────────────────┐
│  Handler │  outbox message >│ DB (same tx)        │
└──────────┘                  │  orders table ✓     │
                              │  outbox table ✓     │
                              └──────────┬──────────┘
                                         │ Background worker
                              ┌──────────▼──────────┐
                              │  Outbox Processor   │
                              │  - Read unprocessed │
                              │  - Publish to broker│
                              │  - Mark processed   │
                              └──────────┬──────────┘
                                         │
                              ┌──────────▼──────────┐
                              │  Message Broker     │
                              │  (RabbitMQ/SB)      │
                              └─────────────────────┘
```

```csharp
// Outbox message entity
public class OutboxMessage
{
    public Guid    Id           { get; init; } = Guid.NewGuid();
    public string  Type         { get; init; } = null!;   // fully-qualified type name
    public string  Payload      { get; init; } = null!;   // JSON
    public DateTimeOffset OccurredAt { get; init; } = DateTimeOffset.UtcNow;
    public DateTimeOffset? ProcessedAt { get; set; }
    public int     RetryCount   { get; set; }
    public string? Error        { get; set; }
}

// EF Core config
public class OutboxMessageConfiguration : IEntityTypeConfiguration<OutboxMessage>
{
    public void Configure(EntityTypeBuilder<OutboxMessage> b)
    {
        b.ToTable("outbox_messages");
        b.HasKey(x => x.Id);
        b.Property(x => x.Type).HasMaxLength(500).IsRequired();
        b.Property(x => x.Payload).HasColumnType("jsonb").IsRequired();
        b.HasIndex(x => x.ProcessedAt).HasFilter("processed_at IS NULL"); // fast polling
    }
}

// IOutboxService
public interface IOutboxService
{
    Task AddMessageAsync<T>(T message, CancellationToken ct = default) where T : class;
}

public class EfCoreOutboxService : IOutboxService
{
    private readonly AppDbContext _db;
    private readonly JsonSerializerOptions _json;

    public EfCoreOutboxService(AppDbContext db)
    {
        _db   = db;
        _json = new JsonSerializerOptions { PropertyNamingPolicy = JsonNamingPolicy.CamelCase };
    }

    public async Task AddMessageAsync<T>(T message, CancellationToken ct = default) where T : class
    {
        var outbox = new OutboxMessage
        {
            Type    = typeof(T).AssemblyQualifiedName!,
            Payload = JsonSerializer.Serialize(message, _json)
        };

        await _db.Set<OutboxMessage>().AddAsync(outbox, ct);
        // Note: SaveChanges is called by the caller (same transaction as the business operation)
    }
}

// Usage in a command handler
public class PlaceOrderHandler : IRequestHandler<PlaceOrderCommand, PlaceOrderResult>
{
    private readonly AppDbContext _db;
    private readonly IOutboxService _outbox;

    public PlaceOrderHandler(AppDbContext db, IOutboxService outbox)
    {
        _db     = db;
        _outbox = outbox;
    }

    public async Task<PlaceOrderResult> Handle(PlaceOrderCommand cmd, CancellationToken ct)
    {
        var order = Order.Create(cmd.CustomerId, cmd.Items);
        await _db.Orders.AddAsync(order, ct);

        // Add domain event to outbox — same transaction as order save
        await _outbox.AddMessageAsync(new OrderPlaced
        {
            OrderId    = order.Id,
            CustomerId = order.CustomerId,
            TotalAmount = order.TotalAmount,
            OccurredAt = DateTimeOffset.UtcNow
        }, ct);

        // Single SaveChanges — both order and outbox message committed atomically
        await _db.SaveChangesAsync(ct);

        return new PlaceOrderResult(order.Id);
    }
}
```

---

## Step 2054: Outbox Processor — Background Worker

```csharp
// Outbox processor polls for unprocessed messages and publishes them
public class OutboxProcessor : BackgroundService
{
    private readonly IServiceProvider _sp;
    private readonly ILogger<OutboxProcessor> _logger;
    private static readonly TimeSpan PollInterval = TimeSpan.FromSeconds(5);

    public OutboxProcessor(IServiceProvider sp, ILogger<OutboxProcessor> logger)
    {
        _sp     = sp;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        _logger.LogInformation("Outbox processor started");

        while (!ct.IsCancellationRequested)
        {
            try
            {
                await ProcessBatchAsync(ct);
            }
            catch (Exception ex) when (!ct.IsCancellationRequested)
            {
                _logger.LogError(ex, "Error processing outbox batch");
            }

            await Task.Delay(PollInterval, ct);
        }
    }

    private async Task ProcessBatchAsync(CancellationToken ct)
    {
        await using var scope = _sp.CreateAsyncScope();
        var db       = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        var publisher = scope.ServiceProvider.GetRequiredService<IMessagePublisher>();

        // Fetch up to 50 unprocessed messages, ordered by occurrence
        var messages = await db.Set<OutboxMessage>()
            .Where(m => m.ProcessedAt == null && m.RetryCount < 5)
            .OrderBy(m => m.OccurredAt)
            .Take(50)
            .ToListAsync(ct);

        if (messages.Count == 0) return;

        _logger.LogDebug("Processing {Count} outbox messages", messages.Count);

        foreach (var message in messages)
        {
            try
            {
                await publisher.PublishAsync(message.Type, message.Payload, ct);
                message.ProcessedAt = DateTimeOffset.UtcNow;
                message.Error       = null;
            }
            catch (Exception ex)
            {
                message.RetryCount++;
                message.Error = ex.Message[..Math.Min(ex.Message.Length, 500)];
                _logger.LogWarning(ex, "Failed to publish outbox message {Id} (attempt {Retry})",
                    message.Id, message.RetryCount);
            }
        }

        await db.SaveChangesAsync(ct);
    }
}

// Message publisher
public class MassTransitMessagePublisher : IMessagePublisher
{
    private readonly IBus _bus;
    private readonly JsonSerializerOptions _json;

    public MassTransitMessagePublisher(IBus bus)
    {
        _bus  = bus;
        _json = new JsonSerializerOptions { PropertyNamingPolicy = JsonNamingPolicy.CamelCase };
    }

    public async Task PublishAsync(string typeName, string payload, CancellationToken ct = default)
    {
        var type    = Type.GetType(typeName) ?? throw new InvalidOperationException($"Unknown type: {typeName}");
        var message = JsonSerializer.Deserialize(payload, type, _json)
                      ?? throw new InvalidOperationException($"Failed to deserialize: {typeName}");

        await _bus.Publish(message, type, ct);
    }
}

builder.Services.AddHostedService<OutboxProcessor>();
```

---

## Step 2055: Optimistic Outbox with Change Interceptors

```csharp
// Intercept SaveChanges to automatically add domain events to outbox
public class DomainEventOutboxInterceptor : SaveChangesInterceptor
{
    private readonly JsonSerializerOptions _json = new() { PropertyNamingPolicy = JsonNamingPolicy.CamelCase };

    public override async ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData,
        InterceptionResult<int> result,
        CancellationToken ct = default)
    {
        if (eventData.Context is not AppDbContext db)
            return await base.SavingChangesAsync(eventData, result, ct);

        // Find all aggregates with pending domain events
        var aggregates = db.ChangeTracker.Entries<IAggregateRoot>()
            .Select(e => e.Entity)
            .Where(a => a.DomainEvents.Count > 0)
            .ToList();

        foreach (var aggregate in aggregates)
        {
            foreach (var domainEvent in aggregate.DomainEvents)
            {
                var outbox = new OutboxMessage
                {
                    Type    = domainEvent.GetType().AssemblyQualifiedName!,
                    Payload = JsonSerializer.Serialize(domainEvent, domainEvent.GetType(), _json)
                };

                db.Set<OutboxMessage>().Add(outbox);
            }

            aggregate.ClearDomainEvents();
        }

        return await base.SavingChangesAsync(eventData, result, ct);
    }
}

// Aggregate root with domain events
public abstract class AggregateRoot : IAggregateRoot
{
    private readonly List<IDomainEvent> _domainEvents = [];

    public IReadOnlyList<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    protected void AddDomainEvent(IDomainEvent @event) => _domainEvents.Add(@event);

    public void ClearDomainEvents() => _domainEvents.Clear();
}

// Order aggregate using the pattern
public class Order : AggregateRoot
{
    public static Order Create(Guid customerId, Money totalAmount)
    {
        var order = new Order
        {
            Id         = Guid.NewGuid(),
            CustomerId = customerId,
            TotalAmount = totalAmount,
            Status     = OrderStatus.Pending,
            CreatedAt  = DateTimeOffset.UtcNow
        };

        // Domain event is automatically picked up by the interceptor
        order.AddDomainEvent(new OrderCreatedEvent
        {
            OrderId     = order.Id,
            CustomerId  = customerId,
            TotalAmount = totalAmount.Amount,
            Currency    = totalAmount.Currency
        });

        return order;
    }

    public Guid        Id          { get; private set; }
    public Guid        CustomerId  { get; private set; }
    public Money       TotalAmount { get; private set; } = null!;
    public OrderStatus Status      { get; private set; }
    public DateTimeOffset CreatedAt { get; private set; }
}
```

---

## Step 2056: Inbox Pattern — Idempotent Message Processing

The Inbox pattern prevents duplicate processing when messages are redelivered.

```csharp
// Inbox message entity
public class InboxMessage
{
    public Guid    MessageId    { get; init; }   // from broker message ID
    public string  Type         { get; init; } = null!;
    public string  Payload      { get; init; } = null!;
    public DateTimeOffset ReceivedAt  { get; init; } = DateTimeOffset.UtcNow;
    public DateTimeOffset? ProcessedAt { get; set; }
    public string? Error        { get; set; }
}

// EF Core config
public class InboxMessageConfiguration : IEntityTypeConfiguration<InboxMessage>
{
    public void Configure(EntityTypeBuilder<InboxMessage> b)
    {
        b.ToTable("inbox_messages");
        b.HasKey(x => x.MessageId);
        b.Property(x => x.MessageId).ValueGeneratedNever(); // we control the ID
        b.HasIndex(x => x.ProcessedAt).HasFilter("processed_at IS NULL");
    }
}

// Idempotent MassTransit consumer
public class IdempotentOrderPlacedConsumer : IConsumer<OrderPlaced>
{
    private readonly AppDbContext _db;
    private readonly IOrderFulfillmentService _fulfillment;
    private readonly ILogger<IdempotentOrderPlacedConsumer> _logger;

    public IdempotentOrderPlacedConsumer(
        AppDbContext db,
        IOrderFulfillmentService fulfillment,
        ILogger<IdempotentOrderPlacedConsumer> logger)
    {
        _db          = db;
        _fulfillment = fulfillment;
        _logger      = logger;
    }

    public async Task Consume(ConsumeContext<OrderPlaced> ctx)
    {
        var messageId = ctx.MessageId ?? Guid.NewGuid();

        // Check if already processed (inbox check)
        var existing = await _db.Set<InboxMessage>()
            .FirstOrDefaultAsync(m => m.MessageId == messageId, ctx.CancellationToken);

        if (existing?.ProcessedAt is not null)
        {
            _logger.LogDebug("Duplicate message {MessageId} ignored", messageId);
            return;
        }

        await using var tx = await _db.Database.BeginTransactionAsync(ctx.CancellationToken);

        try
        {
            // Record in inbox (idempotency token)
            var inbox = new InboxMessage
            {
                MessageId = messageId,
                Type      = typeof(OrderPlaced).AssemblyQualifiedName!,
                Payload   = JsonSerializer.Serialize(ctx.Message)
            };

            if (existing is null)
                _db.Set<InboxMessage>().Add(inbox);

            // Process the message
            await _fulfillment.StartFulfillmentAsync(ctx.Message.OrderId, ctx.CancellationToken);

            // Mark as processed
            (existing ?? inbox).ProcessedAt = DateTimeOffset.UtcNow;

            await _db.SaveChangesAsync(ctx.CancellationToken);
            await tx.CommitAsync(ctx.CancellationToken);

            _logger.LogInformation("Processed OrderPlaced {OrderId}", ctx.Message.OrderId);
        }
        catch (Exception)
        {
            await tx.RollbackAsync(ctx.CancellationToken);
            throw; // re-throw to trigger MassTransit retry
        }
    }
}
```

---

## Step 2057: Advanced Saga Patterns — Compensation & Timeouts

```csharp
// Saga with compensation (rollback on failure) and timeout
public class OrderFulfillmentState : SagaStateMachineInstance
{
    public Guid   CorrelationId  { get; set; }
    public string CurrentState   { get; set; } = null!;
    public Guid   OrderId        { get; set; }
    public Guid   CustomerId     { get; set; }
    public decimal TotalAmount   { get; set; }
    public Guid?  PaymentId      { get; set; }
    public Guid?  ReservationId  { get; set; }
    public DateTimeOffset CreatedAt { get; set; }
    public DateTimeOffset? TimeoutAt { get; set; }
}

public class OrderFulfillmentSaga : MassTransitStateMachine<OrderFulfillmentState>
{
    // States
    public State WaitingForPayment   { get; private set; } = null!;
    public State WaitingForInventory { get; private set; } = null!;
    public State Processing          { get; private set; } = null!;
    public State Completed           { get; private set; } = null!;
    public State Compensating        { get; private set; } = null!;
    public State Failed              { get; private set; } = null!;

    // Events
    public Event<OrderPlaced>            OrderPlaced            { get; private set; } = null!;
    public Event<PaymentProcessed>       PaymentProcessed       { get; private set; } = null!;
    public Event<PaymentFailed>          PaymentFailed          { get; private set; } = null!;
    public Event<StockReserved>          StockReserved          { get; private set; } = null!;
    public Event<StockReservationFailed> StockReservationFailed { get; private set; } = null!;
    public Event<PaymentRefunded>        PaymentRefunded        { get; private set; } = null!;

    // Scheduled events (timeouts)
    public Schedule<OrderFulfillmentState, OrderFulfillmentTimeout> FulfillmentTimeout { get; private set; } = null!;

    public OrderFulfillmentSaga()
    {
        InstanceState(x => x.CurrentState);
        CorrelateBy<Guid>(x => x.OrderId, ctx => ctx.Message.OrderId);

        // Timeout schedule
        Schedule(() => FulfillmentTimeout, x => x.TimeoutAt, s =>
        {
            s.Delay   = TimeSpan.FromMinutes(30);
            s.Received = r => r.CorrelateById(m => m.Message.CorrelationId);
        });

        Initially(
            When(OrderPlaced)
                .Then(ctx =>
                {
                    ctx.Saga.OrderId     = ctx.Message.OrderId;
                    ctx.Saga.CustomerId  = ctx.Message.CustomerId;
                    ctx.Saga.TotalAmount = ctx.Message.TotalAmount;
                    ctx.Saga.CreatedAt   = DateTimeOffset.UtcNow;
                })
                .Schedule(FulfillmentTimeout, ctx => new OrderFulfillmentTimeout
                {
                    CorrelationId = ctx.Saga.CorrelationId
                })
                .Publish(ctx => new ProcessPayment
                {
                    OrderId    = ctx.Saga.OrderId,
                    CustomerId = ctx.Saga.CustomerId,
                    Amount     = ctx.Saga.TotalAmount
                })
                .TransitionTo(WaitingForPayment));

        During(WaitingForPayment,
            When(PaymentProcessed)
                .Then(ctx => ctx.Saga.PaymentId = ctx.Message.PaymentId)
                .Publish(ctx => new ReserveStock
                {
                    OrderId = ctx.Saga.OrderId,
                    Items   = ctx.Message.Items
                })
                .TransitionTo(WaitingForInventory),

            When(PaymentFailed)
                .Publish(ctx => new NotifyOrderFailed
                {
                    OrderId = ctx.Saga.OrderId,
                    Reason  = ctx.Message.Reason
                })
                .TransitionTo(Failed)
                .Finalize(),

            When(FulfillmentTimeout.Received)
                .Publish(ctx => new NotifyOrderFailed
                {
                    OrderId = ctx.Saga.OrderId,
                    Reason  = "Payment timeout"
                })
                .TransitionTo(Failed)
                .Finalize());

        During(WaitingForInventory,
            When(StockReserved)
                .Then(ctx => ctx.Saga.ReservationId = ctx.Message.ReservationId)
                .Publish(ctx => new ShipOrder { OrderId = ctx.Saga.OrderId })
                .TransitionTo(Processing),

            When(StockReservationFailed)
                // Compensate: refund the payment
                .Publish(ctx => new RefundPayment
                {
                    PaymentId = ctx.Saga.PaymentId!.Value,
                    Amount    = ctx.Saga.TotalAmount,
                    Reason    = "Stock unavailable"
                })
                .TransitionTo(Compensating),

            When(FulfillmentTimeout.Received)
                .Publish(ctx => new RefundPayment
                {
                    PaymentId = ctx.Saga.PaymentId!.Value,
                    Amount    = ctx.Saga.TotalAmount,
                    Reason    = "Fulfillment timeout"
                })
                .TransitionTo(Compensating));

        During(Compensating,
            When(PaymentRefunded)
                .Publish(ctx => new NotifyOrderFailed
                {
                    OrderId = ctx.Saga.OrderId,
                    Reason  = "Order failed — payment refunded"
                })
                .TransitionTo(Failed)
                .Finalize());

        During(Processing,
            When(FulfillmentTimeout.Received)
                .If(ctx => ctx.Saga.PaymentId.HasValue,
                    binder => binder
                        .Publish(ctx => new RefundPayment
                        {
                            PaymentId = ctx.Saga.PaymentId!.Value,
                            Amount    = ctx.Saga.TotalAmount,
                            Reason    = "Processing timeout"
                        })
                        .TransitionTo(Compensating)));

        SetCompletedWhenFinalized();
    }
}
```

---

## Step 2058: MassTransit — Dead Letter Queue & Error Handling

```csharp
// Configure fault queues and retries
builder.Services.AddMassTransit(cfg =>
{
    cfg.UsingRabbitMq((ctx, rmq) =>
    {
        rmq.Host(builder.Configuration.GetConnectionString("RabbitMQ"), h =>
        {
            h.Username("guest");
            h.Password("guest");
        });

        // Global retry policy
        rmq.UseMessageRetry(r =>
        {
            r.Incremental(3, TimeSpan.FromSeconds(1), TimeSpan.FromSeconds(5));
            r.Ignore<ArgumentException>(); // don't retry programming errors
            r.Handle<HttpRequestException>();
            r.Handle<TimeoutException>();
        });

        // Global outbox (coordinates with EF Core)
        rmq.UseEntityFrameworkOutbox<AppDbContext>(ctx);

        // Circuit breaker
        rmq.UseCircuitBreaker(cb =>
        {
            cb.TrackingPeriod  = TimeSpan.FromMinutes(1);
            cb.TripThreshold   = 15;   // 15% failure rate trips the breaker
            cb.ActiveThreshold = 10;   // Need 10 messages to evaluate
            cb.ResetInterval   = TimeSpan.FromMinutes(5);
        });

        rmq.ConfigureEndpoints(ctx);
    });

    cfg.AddConsumers(Assembly.GetExecutingAssembly());
    cfg.AddSagaStateMachines(Assembly.GetExecutingAssembly());
    cfg.AddSagas(Assembly.GetExecutingAssembly());
});

// Consumer with explicit fault handling
public class CriticalOrderConsumer : IConsumer<ProcessCriticalOrder>
{
    private readonly IOrderService _service;
    private readonly ILogger<CriticalOrderConsumer> _logger;

    public CriticalOrderConsumer(IOrderService service, ILogger<CriticalOrderConsumer> logger)
    {
        _service = service;
        _logger  = logger;
    }

    public async Task Consume(ConsumeContext<ProcessCriticalOrder> ctx)
    {
        try
        {
            await _service.ProcessAsync(ctx.Message.OrderId, ctx.CancellationToken);
        }
        catch (OrderNotFoundException ex)
        {
            // Don't retry — the order simply doesn't exist
            _logger.LogError(ex, "Order {OrderId} not found", ctx.Message.OrderId);
            throw new DoNotRedeliverException($"Order {ctx.Message.OrderId} not found", ex);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error processing order {OrderId}", ctx.Message.OrderId);
            throw; // allow retry
        }
    }
}

// Fault consumer — processes messages that failed all retries
public class OrderProcessingFaultConsumer : IConsumer<Fault<ProcessCriticalOrder>>
{
    private readonly IAlertService _alerts;

    public OrderProcessingFaultConsumer(IAlertService alerts) => _alerts = alerts;

    public async Task Consume(ConsumeContext<Fault<ProcessCriticalOrder>> ctx)
    {
        var fault = ctx.Message;

        await _alerts.SendAsync(new Alert
        {
            Severity = AlertSeverity.Critical,
            Title    = $"Order processing failed after retries",
            Message  = $"Order {fault.Message.OrderId} failed with: {fault.Exceptions.FirstOrDefault()?.Message}",
            Metadata = new { fault.FaultId, fault.FaultedMessageId, fault.Timestamp }
        }, ctx.CancellationToken);
    }
}
```

---

## Step 2059: Event-Driven Integration — Publishing & Subscribing Cross-Service

```csharp
// Cross-service event contract (shared library or NuGet package)
// Events are defined as records for immutability
namespace Contracts.Events;

public record OrderPlaced(
    Guid   OrderId,
    Guid   CustomerId,
    decimal TotalAmount,
    string  Currency,
    IReadOnlyList<OrderItem> Items,
    DateTimeOffset OccurredAt);

public record OrderCancelled(
    Guid   OrderId,
    Guid   CustomerId,
    string Reason,
    DateTimeOffset OccurredAt);

public record StockReserved(
    Guid   OrderId,
    Guid   ReservationId,
    IReadOnlyList<ReservationItem> Items,
    DateTimeOffset OccurredAt);

// Publishing service
public class OrderEventPublisher : IOrderEventPublisher
{
    private readonly IPublishEndpoint _publish;

    public OrderEventPublisher(IPublishEndpoint publish) => _publish = publish;

    public async Task PublishOrderPlacedAsync(Order order, CancellationToken ct = default)
    {
        await _publish.Publish(new OrderPlaced(
            OrderId:     order.Id,
            CustomerId:  order.CustomerId,
            TotalAmount: order.TotalAmount.Amount,
            Currency:    order.TotalAmount.Currency,
            Items:       order.Items.Select(i => new OrderItem(i.ProductId, i.Quantity, i.UnitPrice)).ToList(),
            OccurredAt:  DateTimeOffset.UtcNow), ct);
    }
}

// Subscribing service (Inventory bounded context)
public class InventoryOrderPlacedConsumer : IConsumer<OrderPlaced>
{
    private readonly IInventoryService _inventory;
    private readonly ILogger<InventoryOrderPlacedConsumer> _logger;

    public InventoryOrderPlacedConsumer(IInventoryService inventory, ILogger<InventoryOrderPlacedConsumer> logger)
    {
        _inventory = inventory;
        _logger    = logger;
    }

    public async Task Consume(ConsumeContext<OrderPlaced> ctx)
    {
        var msg = ctx.Message;
        _logger.LogInformation("Reserving stock for order {OrderId}", msg.OrderId);

        foreach (var item in msg.Items)
        {
            await _inventory.ReserveAsync(item.ProductId, item.Quantity, msg.OrderId, ctx.CancellationToken);
        }

        await ctx.Publish(new StockReserved(
            OrderId:       msg.OrderId,
            ReservationId: Guid.NewGuid(),
            Items:         msg.Items.Select(i => new ReservationItem(i.ProductId, i.Quantity)).ToList(),
            OccurredAt:    DateTimeOffset.UtcNow), ctx.CancellationToken);
    }
}
```

---

## Step 2060: Request-Reply Pattern

```csharp
// Request-Reply: synchronous-style call over async messaging
// Useful when you need a response but want broker durability

// Request/Response types
public record GetProductPriceRequest(Guid ProductId, string Currency);
public record GetProductPriceResponse(Guid ProductId, decimal Price, string Currency, bool IsAvailable);

// Requester (Orders service)
public class PricingClient : IPricingClient
{
    private readonly IRequestClient<GetProductPriceRequest> _client;

    public PricingClient(IRequestClient<GetProductPriceRequest> client) => _client = client;

    public async Task<GetProductPriceResponse> GetPriceAsync(
        Guid productId, string currency, CancellationToken ct = default)
    {
        using var cts = CancellationTokenSource.CreateLinkedTokenSource(ct);
        cts.CancelAfter(TimeSpan.FromSeconds(10)); // 10s timeout

        var response = await _client.GetResponse<GetProductPriceResponse>(
            new GetProductPriceRequest(productId, currency), cts.Token);

        return response.Message;
    }
}

// Responder (Pricing service)
public class GetProductPriceConsumer : IConsumer<GetProductPriceRequest>
{
    private readonly IPricingRepository _pricing;

    public GetProductPriceConsumer(IPricingRepository pricing) => _pricing = pricing;

    public async Task Consume(ConsumeContext<GetProductPriceRequest> ctx)
    {
        var price = await _pricing.GetCurrentPriceAsync(
            ctx.Message.ProductId, ctx.Message.Currency, ctx.CancellationToken);

        await ctx.RespondAsync(new GetProductPriceResponse(
            ProductId:   ctx.Message.ProductId,
            Price:       price?.Amount ?? 0,
            Currency:    ctx.Message.Currency,
            IsAvailable: price is not null));
    }
}

// Register request client
builder.Services.AddMassTransit(cfg =>
{
    cfg.AddRequestClient<GetProductPriceRequest>(new Uri("queue:pricing-service"));
    // ...
});
```

---

## Step 2061: Message Scheduling & Deferred Processing

```csharp
// Schedule messages for future processing
public class OrderReminderService
{
    private readonly IMessageScheduler _scheduler;
    private readonly IBus _bus;

    public OrderReminderService(IMessageScheduler scheduler, IBus bus)
    {
        _scheduler = scheduler;
        _bus       = bus;
    }

    public async Task ScheduleAbandonedCartReminderAsync(Guid cartId, Guid customerId, CancellationToken ct = default)
    {
        var sendAt = DateTimeOffset.UtcNow.AddHours(24);

        await _scheduler.SchedulePublish(sendAt,
            new SendAbandonedCartReminder
            {
                CartId     = cartId,
                CustomerId = customerId,
                ScheduledAt = sendAt
            }, ct);
    }

    public async Task ScheduleOrderFollowUpAsync(Guid orderId, CancellationToken ct = default)
    {
        // Send 3 days after order delivered
        var sendAt = DateTimeOffset.UtcNow.AddDays(3);

        var token = await _scheduler.ScheduleSend(
            new Uri("queue:email-service"),
            sendAt,
            new SendOrderReviewRequest
            {
                OrderId  = orderId,
                SendAt   = sendAt
            }, ct);

        // Store token to cancel if needed
        return token;
    }

    public async Task CancelScheduledReminderAsync(Guid scheduledMessageToken, CancellationToken ct = default)
    {
        await _scheduler.CancelScheduledSend(
            new Uri("queue:email-service"),
            scheduledMessageToken, ct);
    }
}

// Configure Quartz scheduler for MassTransit
builder.Services.AddMassTransit(cfg =>
{
    cfg.AddPublishMessageScheduler();
    cfg.AddQuartzConsumers();

    cfg.UsingRabbitMq((ctx, rmq) =>
    {
        rmq.UsePublishMessageScheduler();
        rmq.ConfigureEndpoints(ctx);
    });
});

builder.Services.AddQuartz(q =>
{
    q.UseMicrosoftDependencyInjectionJobFactory();
});

builder.Services.AddQuartzHostedService(options =>
    options.WaitForJobsToComplete = true);
```

---

## Step 2062: Event Streaming with Kafka

```csharp
// MassTransit + Kafka for high-throughput event streaming
// dotnet add package MassTransit.Kafka

builder.Services.AddMassTransit(cfg =>
{
    cfg.UsingKafka((ctx, kafka) =>
    {
        kafka.Host("localhost:9092");

        // Producer
        kafka.TopicEndpoint<OrderPlaced>("orders.placed", "orders-group", e =>
        {
            e.CreateIfMissing(t =>
            {
                t.NumPartitions     = 6;
                t.ReplicationFactor = 3;
            });
        });

        // Consumer
        kafka.TopicEndpoint<OrderPlaced>("orders.placed", "inventory-group", e =>
        {
            e.AutoOffsetReset = AutoOffsetReset.Earliest;
            e.ConfigureConsumer<InventoryOrderPlacedConsumer>(ctx);

            e.DiscardSkippedMessages();
        });
    });

    cfg.AddConsumer<InventoryOrderPlacedConsumer>();
});

// Kafka partition key (ensures all events for an order go to same partition)
public class OrderPlacedPartitioner : IMessagePartitioner<OrderPlaced>
{
    public Partition GetPartition(Message<Null, OrderPlaced> message, int partitionCount)
    {
        var hash = HashCode.Combine(message.Value.OrderId);
        return new Partition(Math.Abs(hash % partitionCount));
    }
}
```

---

## Step 2063: Azure Service Bus Integration

```csharp
// MassTransit + Azure Service Bus
// dotnet add package MassTransit.Azure.ServiceBus.Core

builder.Services.AddMassTransit(cfg =>
{
    cfg.AddConsumers(Assembly.GetExecutingAssembly());
    cfg.AddSagaStateMachines(Assembly.GetExecutingAssembly());

    cfg.UsingAzureServiceBus((ctx, asb) =>
    {
        asb.Host(builder.Configuration.GetConnectionString("ServiceBus"));

        // Topics for pub/sub (queues for point-to-point)
        asb.Message<OrderPlaced>(m => m.SetEntityName("orders-topic"));
        asb.Message<OrderCancelled>(m => m.SetEntityName("orders-topic"));

        // Subscription rules (filter messages per consumer)
        asb.SubscriptionEndpoint<OrderPlaced>("inventory-service", e =>
        {
            e.ConfigureConsumer<InventoryOrderPlacedConsumer>(ctx);
        });

        // Queue for direct messages
        asb.ReceiveEndpoint("notification-service", e =>
        {
            e.PrefetchCount  = 50;
            e.LockDuration   = TimeSpan.FromMinutes(5);
            e.MaxDeliveryCount = 3;
            e.EnableDeadLetteringOnMessageExpiration = true;
            e.ConfigureConsumers(ctx);
        });

        // Enable sessions (ordered processing per entity)
        asb.ReceiveEndpoint("order-processor", e =>
        {
            e.RequiresSession = true;
        });

        asb.ConfigureEndpoints(ctx);
    });
});
```

---

## Step 2064: Message Versioning & Schema Evolution

```csharp
// Versioned message contracts
// V1 (original)
public record OrderPlaced_V1(Guid OrderId, Guid CustomerId, decimal Amount);

// V2 (added Currency and Items)
[MessageUrn("OrderPlaced")]  // same URN for backward compat
public record OrderPlaced(
    Guid   OrderId,
    Guid   CustomerId,
    decimal TotalAmount,
    string  Currency    = "USD",   // default for V1 consumers
    IReadOnlyList<OrderItem>? Items = null); // nullable for V1 consumers

// Consumer that handles multiple versions
public class OrderPlacedV1Consumer : IConsumer<OrderPlaced_V1>
{
    private readonly IConsumer<OrderPlaced> _v2Consumer;

    public OrderPlacedV1Consumer(IConsumer<OrderPlaced> v2Consumer)
        => _v2Consumer = v2Consumer;

    public async Task Consume(ConsumeContext<OrderPlaced_V1> ctx)
    {
        // Upcast V1 to V2
        var v2Message = new OrderPlaced(
            OrderId:     ctx.Message.OrderId,
            CustomerId:  ctx.Message.CustomerId,
            TotalAmount: ctx.Message.Amount,
            Currency:    "USD",
            Items:       null);

        // Forward to V2 consumer
        await ctx.Forward(new Uri($"loopback://localhost/{KebabCaseEndpointNameFormatter.Instance.Consumer<OrderPlacedConsumer>()}"));
    }
}

// Transform middleware for schema evolution
public class MessageVersionTransform<T> : IFilter<ConsumeContext<T>>
    where T : class
{
    public async Task Send(ConsumeContext<T> context, IPipe<ConsumeContext<T>> next)
    {
        // Apply transformations based on message version header
        if (context.Headers.TryGetHeader("message-version", out var version) &&
            version?.ToString() == "1.0")
        {
            // Transform V1 message to current version
        }

        await next.Send(context);
    }

    public void Probe(ProbeContext context) { }
}
```

---

## Step 2065: Competing Consumers & Work Distribution

```csharp
// Competing consumers: multiple instances consume from same queue
// MassTransit handles this automatically — just register multiple consumers

// Configure concurrency per consumer
builder.Services.AddMassTransit(cfg =>
{
    cfg.AddConsumer<HeavyProcessingConsumer>(c =>
    {
        c.ConcurrentMessageLimit = 10;  // max 10 concurrent per instance
        c.PrefetchCount          = 20;  // pre-fetch 20 from broker
    });

    cfg.UsingRabbitMq((ctx, rmq) =>
    {
        rmq.ReceiveEndpoint("heavy-processing", e =>
        {
            e.PrefetchCount          = 20;
            e.ConcurrentMessageLimit = 10;

            // Fair dispatch — don't starve slow consumers
            e.UseConcurrencyLimit(10);

            e.ConfigureConsumer<HeavyProcessingConsumer>(ctx);
        });

        rmq.ConfigureEndpoints(ctx);
    });
});

// Priority queues
builder.Services.AddMassTransit(cfg =>
{
    cfg.UsingRabbitMq((ctx, rmq) =>
    {
        rmq.ReceiveEndpoint("orders-high-priority", e =>
        {
            e.SetQueueArgument("x-max-priority", 10);
            e.ConfigureConsumer<PriorityOrderConsumer>(ctx);
        });

        rmq.ConfigureEndpoints(ctx);
    });
});

// Publish with priority
await bus.Publish(new ProcessOrder { OrderId = orderId, Priority = 10 },
    ctx => ctx.SetHeader("x-priority", "9"));
```

---

## Step 2066: Observability for Message-Driven Systems

```csharp
// OpenTelemetry distributed tracing across message boundaries
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing =>
    {
        tracing
            .AddSource("MassTransit")  // auto-instrumented
            .AddSource("MyApp.Messaging")
            .AddAspNetCoreInstrumentation()
            .AddOtlpExporter();
    });

// Custom spans in consumers
public class TracedOrderConsumer : IConsumer<OrderPlaced>
{
    private static readonly ActivitySource Source = new("MyApp.Messaging");
    private readonly IOrderService _service;

    public TracedOrderConsumer(IOrderService service) => _service = service;

    public async Task Consume(ConsumeContext<OrderPlaced> ctx)
    {
        using var activity = Source.StartActivity(
            "process_order_placed",
            ActivityKind.Consumer,
            parentContext: ctx.Headers.TryGetValue("traceparent", out var parent)
                ? ActivityContext.Parse(parent?.ToString() ?? string.Empty, null)
                : default);

        activity?.SetTag("order.id",       ctx.Message.OrderId.ToString());
        activity?.SetTag("customer.id",    ctx.Message.CustomerId.ToString());
        activity?.SetTag("messaging.system", "rabbitmq");

        try
        {
            await _service.ProcessAsync(ctx.Message.OrderId, ctx.CancellationToken);
            activity?.SetStatus(ActivityStatusCode.Ok);
        }
        catch (Exception ex)
        {
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            throw;
        }
    }
}

// Message metrics
public class MessagingMetrics
{
    private readonly Counter<long> _messagesConsumed;
    private readonly Counter<long> _messagesFailed;
    private readonly Histogram<double> _processingTime;

    public MessagingMetrics(IMeterFactory meterFactory)
    {
        var meter = meterFactory.Create("MyApp.Messaging");
        _messagesConsumed = meter.CreateCounter<long>("messaging.consumed");
        _messagesFailed   = meter.CreateCounter<long>("messaging.failed");
        _processingTime   = meter.CreateHistogram<double>("messaging.processing_duration", unit: "ms");
    }

    public void RecordConsumed(string messageType, string consumer, double durationMs)
    {
        var tags = new TagList
        {
            { "message.type", messageType },
            { "consumer",     consumer }
        };
        _messagesConsumed.Add(1, tags);
        _processingTime.Record(durationMs, tags);
    }

    public void RecordFailed(string messageType, string consumer, string errorType)
    {
        _messagesFailed.Add(1, new TagList
        {
            { "message.type", messageType },
            { "consumer",     consumer },
            { "error.type",   errorType }
        });
    }
}
```

---

## Step 2067: Event-Driven Projections (CQRS Read Models)

```csharp
// Projecting events to optimized read models
public class OrderSummaryProjection :
    IConsumer<OrderPlaced>,
    IConsumer<OrderStatusChanged>,
    IConsumer<OrderCancelled>
{
    private readonly ReadDbContext _readDb;
    private readonly ILogger<OrderSummaryProjection> _logger;

    public OrderSummaryProjection(ReadDbContext readDb, ILogger<OrderSummaryProjection> logger)
    {
        _readDb = readDb;
        _logger = logger;
    }

    public async Task Consume(ConsumeContext<OrderPlaced> ctx)
    {
        var msg = ctx.Message;

        // Create or update read model
        var summary = new OrderSummary
        {
            OrderId      = msg.OrderId,
            CustomerId   = msg.CustomerId,
            TotalAmount  = msg.TotalAmount,
            Currency     = msg.Currency,
            Status       = "Pending",
            ItemCount    = msg.Items?.Count ?? 0,
            PlacedAt     = msg.OccurredAt,
            UpdatedAt    = msg.OccurredAt
        };

        _readDb.OrderSummaries.Add(summary);
        await _readDb.SaveChangesAsync(ctx.CancellationToken);

        _logger.LogDebug("Created OrderSummary for {OrderId}", msg.OrderId);
    }

    public async Task Consume(ConsumeContext<OrderStatusChanged> ctx)
    {
        var msg     = ctx.Message;
        var summary = await _readDb.OrderSummaries
            .FirstOrDefaultAsync(s => s.OrderId == msg.OrderId, ctx.CancellationToken);

        if (summary is null)
        {
            _logger.LogWarning("OrderSummary not found for {OrderId}", msg.OrderId);
            return;
        }

        summary.Status    = msg.NewStatus;
        summary.UpdatedAt = msg.OccurredAt;

        await _readDb.SaveChangesAsync(ctx.CancellationToken);
    }

    public async Task Consume(ConsumeContext<OrderCancelled> ctx)
    {
        var summary = await _readDb.OrderSummaries
            .FirstOrDefaultAsync(s => s.OrderId == ctx.Message.OrderId, ctx.CancellationToken);

        if (summary is null) return;

        summary.Status         = "Cancelled";
        summary.CancellationReason = ctx.Message.Reason;
        summary.UpdatedAt      = ctx.Message.OccurredAt;

        await _readDb.SaveChangesAsync(ctx.CancellationToken);
    }
}

// Read model
public class OrderSummary
{
    public Guid            OrderId            { get; set; }
    public Guid            CustomerId         { get; set; }
    public decimal         TotalAmount        { get; set; }
    public string          Currency           { get; set; } = null!;
    public string          Status             { get; set; } = null!;
    public int             ItemCount          { get; set; }
    public DateTimeOffset  PlacedAt           { get; set; }
    public DateTimeOffset  UpdatedAt          { get; set; }
    public string?         CancellationReason { get; set; }
}
```

---

## Step 2068: Complete Message Bus Configuration

```csharp
// Complete MassTransit setup with health checks, observability, and resilience
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddMassTransit(cfg =>
{
    // Consumer registration
    cfg.AddConsumers(Assembly.GetExecutingAssembly());
    cfg.AddSagaStateMachines(Assembly.GetExecutingAssembly());
    cfg.AddSagas(Assembly.GetExecutingAssembly());

    // Request clients
    cfg.AddRequestClient<GetProductPriceRequest>(new Uri("queue:pricing-service"));

    // EF Core outbox
    cfg.AddEntityFrameworkOutbox<AppDbContext>(o =>
    {
        o.UsePostgres();
        o.UseBusOutbox();
        o.QueryDelay = TimeSpan.FromSeconds(1);
    });

    // EF Core saga repository
    cfg.AddSagaRepository<OrderFulfillmentState>()
       .EntityFrameworkRepository(r =>
       {
           r.ConcurrencyMode = ConcurrencyMode.Optimistic;
           r.ExistingDbContext<AppDbContext>();
           r.UsePostgres();
       });

    cfg.UsingRabbitMq((ctx, rmq) =>
    {
        rmq.Host(builder.Configuration["RabbitMQ:Host"], h =>
        {
            h.Username(builder.Configuration["RabbitMQ:Username"]!);
            h.Password(builder.Configuration["RabbitMQ:Password"]!);
            h.Heartbeat(TimeSpan.FromSeconds(60));
        });

        // Retry with exponential backoff
        rmq.UseMessageRetry(r =>
        {
            r.Exponential(5, TimeSpan.FromSeconds(1), TimeSpan.FromSeconds(60), TimeSpan.FromSeconds(2));
            r.Ignore<ArgumentNullException>();
            r.Ignore<InvalidOperationException>();
        });

        // Circuit breaker
        rmq.UseCircuitBreaker(cb =>
        {
            cb.TrackingPeriod  = TimeSpan.FromMinutes(1);
            cb.TripThreshold   = 10;
            cb.ActiveThreshold = 5;
            cb.ResetInterval   = TimeSpan.FromMinutes(5);
        });

        // Use EF Core outbox
        rmq.UseEntityFrameworkOutbox<AppDbContext>(ctx);

        // Publish scheduled messages with Quartz
        rmq.UsePublishMessageScheduler();

        rmq.ConfigureEndpoints(ctx);
    });
});

// Health checks
builder.Services.AddHealthChecks()
    .AddRabbitMQ(
        sp => sp.GetRequiredService<IConnection>(),
        name:          "rabbitmq",
        tags:          ["messaging", "infrastructure"],
        failureStatus: HealthStatus.Degraded);

// OpenTelemetry
builder.Services.AddOpenTelemetry()
    .WithTracing(t => t.AddSource("MassTransit"));

var app = builder.Build();
app.Run();
```

```yaml
# docker-compose.yml — messaging infrastructure
services:
  rabbitmq:
    image: rabbitmq:3.13-management
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_USER: guest
      RABBITMQ_DEFAULT_PASS: guest
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7.2-alpine
    ports:
      - "6379:6379"
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data

volumes:
  rabbitmq_data:
  redis_data:
```

**Summary**: Part 89 covers the Transactional Outbox pattern with EF Core interceptors for automatic domain event capture, Inbox pattern for idempotent message processing, advanced Saga patterns with compensation and timeouts, dead letter queues and fault consumers, request-reply over async messaging, message scheduling with Quartz, Azure Service Bus integration, message versioning, competing consumers, distributed tracing across message boundaries, event-driven read model projections, and complete MassTransit configuration with health checks. Steps 2053–2068 complete.
