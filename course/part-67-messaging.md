# Part 67: Messaging Patterns — MassTransit, Wolverine, Outbox & Sagas (Steps 1701-1716)

## Steps 1701-1716: Reliable Asynchronous Messaging for Distributed Systems

Distributed systems fail. Networks partition. Services crash mid-operation. This part covers the patterns and tools that make your message-driven workflows reliable: the Transactional Outbox, exactly-once semantics, MassTransit consumer pipelines, Saga orchestration vs choreography, Wolverine as an alternative, and Dead Letter Queue processing.

---

## Step 1701: Why Distributed Messaging is Hard

```
// The dual-write problem — without Outbox, this is NOT atomic:
await _orderRepository.SaveAsync(order);   // 1. Saves to DB
await _bus.PublishAsync(new OrderPlaced(order.Id)); // 2. Publishes to broker
// If the process crashes between 1 and 2:
// → DB has the order, broker never received the event
// → Downstream services never know the order was placed
// → Silent data inconsistency
```

**Solutions:**
- **Transactional Outbox**: write to DB + outbox atomically, relay process publishes to broker
- **Saga Pattern**: coordinate multi-step workflows with compensating transactions
- **Idempotent consumers**: handle duplicate messages safely

---

## Step 1702: MassTransit Setup

```xml
<PackageReference Include="MassTransit" Version="8.2.5" />
<PackageReference Include="MassTransit.RabbitMQ" Version="8.2.5" />
<PackageReference Include="MassTransit.EntityFrameworkCore" Version="8.2.5" />
```

```csharp
using MassTransit;

builder.Services.AddMassTransit(x =>
{
    // Scan for consumers, sagas, activities in this assembly
    x.AddConsumers(typeof(Program).Assembly);
    x.AddSagaStateMachines(typeof(Program).Assembly);
    x.AddActivities(typeof(Program).Assembly);

    x.UsingRabbitMq((ctx, cfg) =>
    {
        cfg.Host("rabbitmq://localhost", h =>
        {
            h.Username("guest");
            h.Password("guest");
        });

        // Retry policy: 3 retries with exponential backoff
        cfg.UseMessageRetry(r => r.Exponential(
            retryLimit: 3,
            minInterval: TimeSpan.FromSeconds(1),
            maxInterval: TimeSpan.FromSeconds(30),
            intervalDelta: TimeSpan.FromSeconds(2)));

        // Move to DLQ after all retries exhausted
        cfg.UseDelayedMessageScheduler();
        cfg.UseInMemoryOutbox(ctx); // optional in-process outbox

        cfg.ConfigureEndpoints(ctx);
    });
});
```

---

## Step 1703: Publishing and Consuming Messages

```csharp
// Define your messages (use records or classes, prefer immutable)
namespace MyApp.Contracts;

public record OrderPlaced(
    Guid OrderId,
    string CustomerId,
    decimal TotalAmount,
    DateTimeOffset PlacedAt);

public record OrderShipped(
    Guid OrderId,
    string TrackingNumber,
    DateTimeOffset ShippedAt);

public record SendOrderConfirmationEmail(
    Guid OrderId,
    string CustomerEmail,
    decimal TotalAmount);
```

```csharp
// Publishing (fire-and-forget)
public class OrderService(IPublishEndpoint publisher)
{
    public async Task PlaceOrderAsync(PlaceOrderCommand cmd, CancellationToken ct)
    {
        var order = Order.Create(cmd);
        await _repository.SaveAsync(order, ct);

        // Publish to all subscribers
        await publisher.Publish(new OrderPlaced(
            order.Id,
            order.CustomerId,
            order.TotalAmount,
            DateTimeOffset.UtcNow), ct);
    }
}

// Sending to a specific endpoint (point-to-point)
public class NotificationService(ISendEndpointProvider sender)
{
    public async Task SendEmailAsync(SendOrderConfirmationEmail msg, CancellationToken ct)
    {
        var endpoint = await sender.GetSendEndpoint(
            new Uri("queue:send-order-confirmation"));
        await endpoint.Send(msg, ct);
    }
}
```

```csharp
// Consumer
public class OrderPlacedConsumer : IConsumer<OrderPlaced>
{
    private readonly IEmailService _email;
    private readonly ILogger<OrderPlacedConsumer> _logger;

    public OrderPlacedConsumer(IEmailService email, ILogger<OrderPlacedConsumer> logger)
    {
        _email = email;
        _logger = logger;
    }

    public async Task Consume(ConsumeContext<OrderPlaced> context)
    {
        var evt = context.Message;
        _logger.LogInformation("Processing order {OrderId} for customer {CustomerId}",
            evt.OrderId, evt.CustomerId);

        // Idempotency: check if already processed
        await _email.SendConfirmationAsync(evt.CustomerId, evt.OrderId, evt.TotalAmount,
            context.CancellationToken);

        // Optionally publish follow-up events
        await context.Publish(new SendOrderConfirmationEmail(
            evt.OrderId,
            await _customerRepo.GetEmailAsync(evt.CustomerId),
            evt.TotalAmount));
    }
}

// Consumer definition (optional — override defaults)
public class OrderPlacedConsumerDefinition : ConsumerDefinition<OrderPlacedConsumer>
{
    public OrderPlacedConsumerDefinition()
    {
        ConcurrentMessageLimit = 10; // max 10 concurrent messages
        EndpointName = "order-placed-handler";
    }

    protected override void ConfigureConsumer(
        IReceiveEndpointConfigurator endpointConfigurator,
        IConsumerConfigurator<OrderPlacedConsumer> consumerConfigurator)
    {
        endpointConfigurator.UseMessageRetry(r =>
            r.Intervals(TimeSpan.FromSeconds(5), TimeSpan.FromSeconds(30)));
        endpointConfigurator.UseInMemoryOutbox(Context);
    }
}
```

---

## Step 1704: Transactional Outbox Pattern

The Outbox guarantees atomic: DB write + event publication, via a relay process that reads from the outbox table and forwards to the broker.

```csharp
// Install: MassTransit.EntityFrameworkCore
// Configure EF Core outbox
builder.Services.AddMassTransit(x =>
{
    x.AddEntityFrameworkOutbox<AppDbContext>(o =>
    {
        o.UsePostgres();                // or UseSqlServer(), UseSqlite()
        o.UseBusOutbox();               // route all publishes through the outbox
        o.QueryDelay = TimeSpan.FromSeconds(1);
        o.QueryTimeout = TimeSpan.FromSeconds(30);
        o.MessageDeliveryLimit = 100;   // max messages per delivery batch
    });

    x.UsingRabbitMq((ctx, cfg) =>
    {
        cfg.UseInMemoryOutbox(ctx);     // combine with EF outbox for saga state machines
        cfg.ConfigureEndpoints(ctx);
    });
});
```

```csharp
// EF Core migration for the outbox tables
// dotnet ef migrations add AddMassTransitOutbox
public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Add MassTransit inbox/outbox tables
        modelBuilder.AddInboxStateEntity();
        modelBuilder.AddOutboxMessageEntity();
        modelBuilder.AddOutboxStateEntity();
    }
}
```

```csharp
// Atomic: save order + enqueue event in the SAME transaction
public class OrderService(AppDbContext db, IPublishEndpoint publisher)
{
    public async Task PlaceOrderAsync(PlaceOrderCommand cmd, CancellationToken ct)
    {
        await using var tx = await db.Database.BeginTransactionAsync(ct);
        try
        {
            var order = Order.Create(cmd);
            db.Orders.Add(order);

            // This write goes to the OUTBOX TABLE in the same transaction
            await publisher.Publish(new OrderPlaced(order.Id, order.CustomerId,
                order.TotalAmount, DateTimeOffset.UtcNow), ct);

            await db.SaveChangesAsync(ct);    // DB save + outbox write are atomic
            await tx.CommitAsync(ct);
        }
        catch
        {
            await tx.RollbackAsync(ct);
            throw;
        }
        // Relay process polls the outbox table and delivers to RabbitMQ
    }
}
```

---

## Step 1705: Saga Orchestration — State Machine Pattern

A Saga coordinates a long-running, multi-step workflow. The state machine version (Orchestration) has one central coordinator that drives the steps.

```csharp
using MassTransit;

// Saga state — persisted to DB between steps
public class OrderSagaState : SagaStateMachineInstance
{
    public Guid CorrelationId { get; set; }  // = OrderId
    public string CurrentState { get; set; } = null!;
    public string CustomerId { get; set; } = null!;
    public decimal TotalAmount { get; set; }
    public string? PaymentReference { get; set; }
    public DateTimeOffset PlacedAt { get; set; }
    public DateTimeOffset? PaidAt { get; set; }
    public DateTimeOffset? ShippedAt { get; set; }
}

// Saga state machine
public class OrderSaga : MassTransitStateMachine<OrderSagaState>
{
    // States
    public State Pending { get; private set; } = null!;
    public State WaitingForPayment { get; private set; } = null!;
    public State Paid { get; private set; } = null!;
    public State Shipped { get; private set; } = null!;
    public State Cancelled { get; private set; } = null!;

    // Events (messages that drive the state machine)
    public Event<OrderPlaced> OrderPlaced { get; private set; } = null!;
    public Event<PaymentAuthorized> PaymentAuthorized { get; private set; } = null!;
    public Event<PaymentFailed> PaymentFailed { get; private set; } = null!;
    public Event<OrderShipped> OrderShipped { get; private set; } = null!;
    public Event<OrderCancelled> OrderCancelled { get; private set; } = null!;

    // Requests (request/response via saga)
    public Request<OrderSagaState, ProcessPaymentRequest, ProcessPaymentResponse>
        ProcessPayment { get; private set; } = null!;

    public OrderSaga()
    {
        // How to find/correlate saga instance from message
        CorrelateById(saga => saga.CorrelationId, context => context.Message.OrderId);
        // For PaymentAuthorized, etc.
        CorrelateById(saga => saga.CorrelationId, context => context.Message.OrderId);

        // Request/response configuration
        Request(() => ProcessPayment, r =>
        {
            r.Timeout = TimeSpan.FromMinutes(5);
        });

        Initially(
            When(OrderPlaced)
                .Then(ctx =>
                {
                    ctx.Saga.CustomerId = ctx.Message.CustomerId;
                    ctx.Saga.TotalAmount = ctx.Message.TotalAmount;
                    ctx.Saga.PlacedAt = ctx.Message.PlacedAt;
                })
                .Request(ProcessPayment, ctx => new ProcessPaymentRequest(
                    ctx.Saga.CorrelationId,
                    ctx.Saga.CustomerId,
                    ctx.Saga.TotalAmount))
                .TransitionTo(WaitingForPayment));

        During(WaitingForPayment,
            When(ProcessPayment.Completed)
                .Then(ctx =>
                {
                    ctx.Saga.PaymentReference = ctx.Message.PaymentReference;
                    ctx.Saga.PaidAt = DateTimeOffset.UtcNow;
                })
                .Publish(ctx => new PaymentConfirmed(
                    ctx.Saga.CorrelationId,
                    ctx.Saga.PaymentReference!))
                .TransitionTo(Paid),

            When(ProcessPayment.Faulted)
                .Publish(ctx => new OrderCancelled(
                    ctx.Saga.CorrelationId, "Payment failed"))
                .TransitionTo(Cancelled),

            When(ProcessPayment.TimeoutExpired)
                .Publish(ctx => new OrderCancelled(
                    ctx.Saga.CorrelationId, "Payment timeout"))
                .TransitionTo(Cancelled));

        During(Paid,
            When(OrderShipped)
                .Then(ctx => ctx.Saga.ShippedAt = ctx.Message.ShippedAt)
                .TransitionTo(Shipped));

        DuringAny(
            When(OrderCancelled)
                .TransitionTo(Cancelled));

        SetCompletedWhenFinalized();
    }
}
```

```csharp
// Register saga with EF Core persistence
builder.Services.AddMassTransit(x =>
{
    x.AddSagaStateMachine<OrderSaga, OrderSagaState>()
        .EntityFrameworkRepository(r =>
        {
            r.ExistingDbContext<AppDbContext>();
            r.UsePostgres();
            r.ConcurrencyMode = ConcurrencyMode.Optimistic;  // EF concurrency tokens
        });
});
```

---

## Step 1706: Saga Choreography — Event-Driven Coordination

In choreography, each service reacts to events and decides its own next step. No central coordinator.

```csharp
// Service 1: Order Service
public class OrderService
{
    public async Task PlaceOrderAsync(PlaceOrderCommand cmd)
    {
        var order = Order.Place(cmd);
        await _repository.SaveAsync(order);
        await _publisher.Publish(new OrderPlaced(order.Id, order.CustomerId, order.TotalAmount));
        // That's it — no knowledge of what happens next
    }
}

// Service 2: Payment Service — reacts to OrderPlaced
public class PaymentConsumer : IConsumer<OrderPlaced>
{
    public async Task Consume(ConsumeContext<OrderPlaced> context)
    {
        var result = await _paymentGateway.ChargeAsync(
            context.Message.CustomerId,
            context.Message.TotalAmount);

        if (result.Success)
            await context.Publish(new PaymentAuthorized(context.Message.OrderId, result.Reference));
        else
            await context.Publish(new PaymentFailed(context.Message.OrderId, result.Error));
    }
}

// Service 3: Fulfillment Service — reacts to PaymentAuthorized
public class FulfillmentConsumer : IConsumer<PaymentAuthorized>
{
    public async Task Consume(ConsumeContext<PaymentAuthorized> context)
    {
        var shipment = await _warehouse.PickAndShipAsync(context.Message.OrderId);
        await context.Publish(new OrderShipped(context.Message.OrderId, shipment.TrackingNumber, DateTimeOffset.UtcNow));
    }
}
```

**Choreography vs Orchestration:**

| | Choreography | Orchestration |
|---|---|---|
| Coupling | Loose — services don't know each other | Tighter — saga knows all steps |
| Visibility | Hard to see the full workflow | State machine is the workflow map |
| Error handling | Each service must compensate independently | Central compensating transactions |
| Testing | Need integration tests across services | Unit-test the state machine |
| Use when | Simple reactive flows | Complex multi-step business workflows |

---

## Step 1707: Idempotent Consumers

```csharp
// Consumers must handle duplicate deliveries safely
// RabbitMQ provides at-least-once delivery — you WILL get duplicates

public class SendEmailConsumer(IEmailService email, AppDbContext db) : IConsumer<SendOrderConfirmationEmail>
{
    public async Task Consume(ConsumeContext<SendOrderConfirmationEmail> context)
    {
        var msg = context.Message;

        // Idempotency check: has this email already been sent?
        var alreadySent = await db.SentEmails
            .AnyAsync(e => e.OrderId == msg.OrderId && e.EmailType == "OrderConfirmation",
                context.CancellationToken);

        if (alreadySent)
        {
            // Already processed — safe to acknowledge without re-sending
            return;
        }

        await email.SendConfirmationAsync(msg.CustomerEmail, msg.OrderId, msg.TotalAmount,
            context.CancellationToken);

        // Record that we sent this email
        db.SentEmails.Add(new SentEmail(msg.OrderId, "OrderConfirmation", DateTimeOffset.UtcNow));
        await db.SaveChangesAsync(context.CancellationToken);
    }
}
```

```csharp
// MassTransit's built-in inbox (deduplication) — uses the message ID
// Configure with the EF Core outbox setup:
builder.Services.AddMassTransit(x =>
{
    x.AddEntityFrameworkOutbox<AppDbContext>(o =>
    {
        o.UsePostgres();
        o.UseBusOutbox();
        // Inbox tracks received message IDs and deduplicates automatically
        // MessageDeliveryTimeout = how long to keep inbox records
        o.MessageDeliveryTimeout = TimeSpan.FromDays(1);
    });
});
```

---

## Step 1708: Dead Letter Queue Processing

```csharp
// Configure DLQ handling
x.UsingRabbitMq((ctx, cfg) =>
{
    cfg.ReceiveEndpoint("order-placed-handler", e =>
    {
        // After 3 retries, move to DLQ
        e.UseMessageRetry(r => r.Interval(3, TimeSpan.FromSeconds(5)));
        e.UseDeadLetterQueue("order-placed-handler-dlq");
        e.ConfigureConsumer<OrderPlacedConsumer>(ctx);
    });
});
```

```csharp
// DLQ processor — inspect, fix, and re-queue failed messages
public class DlqProcessor(IBusControl bus, ILogger<DlqProcessor> logger)
{
    public async Task ProcessDeadLettersAsync(CancellationToken ct)
    {
        // Read from DLQ and analyze
        var deadLetters = await _dlqRepository.GetPendingAsync(100, ct);

        foreach (var dl in deadLetters)
        {
            logger.LogWarning(
                "Dead letter: {MessageType} {MessageId} failed {RetryCount} times. Error: {Error}",
                dl.MessageType, dl.MessageId, dl.RetryCount, dl.LastError);

            if (IsTransient(dl.LastError))
            {
                // Re-queue for retry
                await bus.Publish(dl.Deserialize(), ct);
                await _dlqRepository.MarkRequeuedAsync(dl.Id, ct);
            }
            else
            {
                // Requires manual investigation
                await _alertService.NotifyAsync($"Unprocessable message: {dl.MessageId}", ct);
                await _dlqRepository.MarkRequiresManualReviewAsync(dl.Id, ct);
            }
        }
    }

    private static bool IsTransient(string? error) =>
        error?.Contains("timeout", StringComparison.OrdinalIgnoreCase) == true ||
        error?.Contains("connection", StringComparison.OrdinalIgnoreCase) == true;
}
```

---

## Step 1709: MassTransit Middleware — Filters and Behaviors

```csharp
// Custom consume filter — adds correlation ID to logging scope
public class CorrelationIdFilter<T> : IFilter<ConsumeContext<T>>
    where T : class
{
    private readonly ILogger<CorrelationIdFilter<T>> _logger;

    public CorrelationIdFilter(ILogger<CorrelationIdFilter<T>> logger)
        => _logger = logger;

    public async Task Send(ConsumeContext<T> context, IPipe<ConsumeContext<T>> next)
    {
        using (_logger.BeginScope(new Dictionary<string, object>
        {
            ["CorrelationId"] = context.CorrelationId?.ToString() ?? "unknown",
            ["MessageId"] = context.MessageId?.ToString() ?? "unknown",
            ["MessageType"] = typeof(T).Name
        }))
        {
            await next.Send(context);
        }
    }

    public void Probe(ProbeContext context) => context.CreateFilterScope("correlation-id");
}

// Register globally
x.UsingRabbitMq((ctx, cfg) =>
{
    cfg.UseConsumeFilter(typeof(CorrelationIdFilter<>), ctx);
    cfg.ConfigureEndpoints(ctx);
});
```

---

## Step 1710: Wolverine — High-Performance Messaging Alternative

Wolverine (formerly Jasper) is an alternative to MassTransit with a focus on simplicity and performance.

```xml
<PackageReference Include="Wolverine" Version="3.1.0" />
<PackageReference Include="Wolverine.RabbitMQ" Version="3.1.0" />
<PackageReference Include="Wolverine.EntityFrameworkCore" Version="3.1.0" />
```

```csharp
using Wolverine;
using Wolverine.RabbitMQ;
using Wolverine.EntityFrameworkCore;

builder.Host.UseWolverine(opts =>
{
    // RabbitMQ transport
    opts.UseRabbitMq(rabbit =>
    {
        rabbit.ConnectionFactory.HostName = "localhost";
    })
    .AutoProvisionAll();   // create exchanges/queues automatically

    // Transactional Outbox with EF Core
    opts.Policies.AutoApplyTransactions();  // auto-wrap handlers in transactions
    opts.PersistMessagesWithEntityFramework<AppDbContext>(); // outbox via EF Core

    // Retry policy
    opts.Policies.OnException<HttpRequestException>()
        .RetryWithCooldown(250.Milliseconds(), 500.Milliseconds(), 1.Seconds());

    opts.Policies.OnException<Exception>()
        .MoveToErrorQueue();

    // Subscriptions
    opts.ListenToRabbitQueue("order-placed");
});
```

```csharp
// Wolverine handler — just a method, no interface needed
public static class OrderPlacedHandler
{
    // Wolverine finds this by convention: handles OrderPlaced messages
    public static async Task Handle(
        OrderPlaced evt,
        AppDbContext db,
        IEmailService email,
        CancellationToken ct)
    {
        var order = await db.Orders.FindAsync(new object[] { evt.OrderId }, ct);
        if (order is null) return;

        await email.SendConfirmationAsync(order.CustomerId, order.Id, order.TotalAmount, ct);
    }
}

// Alternatively, return outgoing messages from the handler
public static class PaymentConsumerHandler
{
    // Return a message to be published automatically
    public static PaymentConfirmed Handle(PaymentAuthorized evt, AppDbContext db)
    {
        // ... process payment ...
        return new PaymentConfirmed(evt.OrderId, evt.Reference);
    }
}

// Batch handler — process multiple messages at once
public static class BulkOrderHandler
{
    public static async Task Handle(
        IReadOnlyList<OrderPlaced> orders,  // batch of up to N messages
        AppDbContext db,
        CancellationToken ct)
    {
        var orderIds = orders.Select(o => o.OrderId).ToList();
        var dbOrders = await db.Orders
            .Where(o => orderIds.Contains(o.Id))
            .ToListAsync(ct);
        // process batch
    }
}
```

```csharp
// Wolverine command bus (local in-process messaging)
public class OrderController(IMessageBus bus) : ControllerBase
{
    [HttpPost]
    public async Task<IActionResult> PlaceOrder(
        PlaceOrderCommand cmd,
        CancellationToken ct)
    {
        // Dispatches locally; if EF Core integration is set up,
        // this is automatically wrapped in a transaction
        var orderId = await bus.InvokeAsync<Guid>(cmd, ct);
        return Ok(new { orderId });
    }
}

// Local handler
public static class PlaceOrderHandler
{
    // Wolverine automatically enlist this in the EF Core transaction
    public static async Task<Guid> Handle(
        PlaceOrderCommand cmd,
        AppDbContext db,
        CancellationToken ct)
    {
        var order = Order.Create(cmd);
        db.Orders.Add(order);
        // No SaveChanges — Wolverine calls it after handler returns
        return order.Id;
    }
}
```

---

## Step 1711: Request/Response Pattern

```csharp
// MassTransit request/response
public class OrderController(IRequestClient<GetOrderStatusQuery> client) : ControllerBase
{
    [HttpGet("{id:guid}/status")]
    public async Task<IActionResult> GetStatus(Guid id, CancellationToken ct)
    {
        try
        {
            // Times out after 10 seconds
            var response = await client.GetResponse<OrderStatusResponse>(
                new GetOrderStatusQuery(id), ct,
                timeout: RequestTimeout.After(s: 10));

            return Ok(response.Message);
        }
        catch (RequestTimeoutException)
        {
            return StatusCode(504, "Order status service timed out");
        }
    }
}

// Responder
public class GetOrderStatusConsumer(IOrderRepository repo) : IConsumer<GetOrderStatusQuery>
{
    public async Task Consume(ConsumeContext<GetOrderStatusQuery> context)
    {
        var order = await repo.GetByIdAsync(context.Message.OrderId, context.CancellationToken);

        if (order is null)
        {
            await context.RespondAsync(new OrderStatusResponse(
                context.Message.OrderId, "NotFound", null));
            return;
        }

        await context.RespondAsync(new OrderStatusResponse(
            order.Id, order.Status.ToString(), order.LastUpdated));
    }
}
```

---

## Step 1712: Delayed Messages and Scheduling

```csharp
// MassTransit: schedule a message to be delivered in the future
public class OrderService(IMessageScheduler scheduler)
{
    public async Task PlaceOrderAsync(PlaceOrderCommand cmd, CancellationToken ct)
    {
        var order = Order.Place(cmd);
        await _repository.SaveAsync(order, ct);

        // Schedule an expiry check in 24 hours
        await scheduler.SchedulePublish(
            DateTimeOffset.UtcNow.AddHours(24),
            new OrderExpiryCheck(order.Id),
            ct);
    }
}

// Wolverine: schedule with delays
public static class PlaceOrderHandler
{
    public static (OrderCreated, DeliveryOptions) Handle(PlaceOrderCommand cmd)
    {
        var order = Order.Create(cmd);
        var evt = new OrderCreated(order.Id);

        // Schedule via Wolverine's delivery options
        var options = new DeliveryOptions
        {
            ScheduleDelay = TimeSpan.FromHours(24)
        };

        return (evt, options);
    }
}
```

---

## Step 1713: Message Versioning

```csharp
// Version your contracts carefully — consumers may be on older versions
namespace MyApp.Contracts.V1;
public record OrderPlaced(Guid OrderId, string CustomerId, decimal TotalAmount);

namespace MyApp.Contracts.V2;
public record OrderPlaced(
    Guid OrderId,
    string CustomerId,
    decimal TotalAmount,
    string? CouponCode,     // new in v2 — nullable so v1 publishers work
    string Region = "US");  // new in v2 — has default so v1 publishers work

// Handle both versions
public class OrderPlacedV1Consumer : IConsumer<V1.OrderPlaced>
{
    public async Task Consume(ConsumeContext<V1.OrderPlaced> context)
    {
        // Map v1 to internal domain model, set defaults for new fields
        var evt = context.Message;
        await _handler.HandleAsync(new InternalOrderPlaced(
            evt.OrderId, evt.CustomerId, evt.TotalAmount,
            CouponCode: null, Region: "US"));
    }
}

public class OrderPlacedV2Consumer : IConsumer<V2.OrderPlaced>
{
    public async Task Consume(ConsumeContext<V2.OrderPlaced> context)
    {
        var evt = context.Message;
        await _handler.HandleAsync(new InternalOrderPlaced(
            evt.OrderId, evt.CustomerId, evt.TotalAmount,
            evt.CouponCode, evt.Region));
    }
}
```

---

## Step 1714: Integration Testing with MassTransit

```csharp
using MassTransit.Testing;
using Microsoft.Extensions.DependencyInjection;

public class OrderSagaTests : IAsyncLifetime
{
    private ServiceProvider _provider = null!;
    private ITestHarness _harness = null!;

    public async Task InitializeAsync()
    {
        _provider = new ServiceCollection()
            .AddMassTransitTestHarness(x =>
            {
                x.AddSagaStateMachine<OrderSaga, OrderSagaState>()
                    .InMemoryRepository();
                x.AddConsumer<PaymentConsumer>();
            })
            .BuildServiceProvider(true);

        _harness = _provider.GetRequiredService<ITestHarness>();
        await _harness.Start();
    }

    public async Task DisposeAsync()
    {
        await _harness.Stop();
        await _provider.DisposeAsync();
    }

    [Fact]
    public async Task OrderSaga_WhenPaymentSucceeds_TransitionsToPaid()
    {
        var sagaHarness = _harness.GetSagaStateMachineHarness<OrderSaga, OrderSagaState>();

        var orderId = Guid.NewGuid();

        // Publish the triggering event
        await _harness.Bus.Publish(new OrderPlaced(orderId, "CUST-001", 99.99m, DateTimeOffset.UtcNow));

        // Wait for saga to be created
        Assert.True(await sagaHarness.Created.Any(x => x.CorrelationId == orderId));

        // Simulate payment success
        await _harness.Bus.Publish(new PaymentAuthorized(orderId, "PAY-123"));

        // Assert state machine transitioned to Paid
        var saga = sagaHarness.Created.Select(x => x.CorrelationId == orderId).First().Saga;
        Assert.Equal("Paid", saga.CurrentState);
        Assert.NotNull(saga.PaidAt);

        // Assert downstream event was published
        Assert.True(await _harness.Published.Any<PaymentConfirmed>(
            x => x.Context.Message.OrderId == orderId));
    }

    [Fact]
    public async Task OrderPlacedConsumer_SendsConfirmationEmail()
    {
        var consumerHarness = _harness.GetConsumerHarness<OrderPlacedConsumer>();

        await _harness.Bus.Publish(new OrderPlaced(
            Guid.NewGuid(), "CUST-001", 49.99m, DateTimeOffset.UtcNow));

        Assert.True(await consumerHarness.Consumed.Any<OrderPlaced>());
    }
}
```

---

## Step 1715: Kafka Integration

```csharp
// Install: MassTransit.Kafka
builder.Services.AddMassTransit(x =>
{
    x.UsingKafka((ctx, cfg) =>
    {
        cfg.Host("localhost:9092");

        cfg.TopicEndpoint<OrderPlaced>("order-events", "order-service-group", e =>
        {
            e.AutoOffsetReset = AutoOffsetReset.Earliest;
            e.CheckpointMessageCount = 100;         // commit offset every 100 messages
            e.CheckpointInterval = TimeSpan.FromSeconds(30);
            e.ConcurrentConsumerLimit = 5;          // parallel consumers
            e.ConfigureConsumer<OrderPlacedConsumer>(ctx);
        });
    });
});
```

```csharp
// Kafka producer with partitioning
public class KafkaOrderProducer(ITopicProducer<string, OrderPlaced> producer)
{
    public async Task PublishAsync(Order order, CancellationToken ct)
    {
        await producer.Produce(
            key: order.CustomerId,      // partition by customer ID
            value: new OrderPlaced(order.Id, order.CustomerId, order.TotalAmount, DateTimeOffset.UtcNow),
            ctx => ctx.Headers.Set("source-service", "order-api"),
            ct);
    }
}
```

---

## Step 1716: Observability for Messaging

```csharp
// MassTransit + OpenTelemetry integration
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing =>
    {
        tracing
            .AddAspNetCoreInstrumentation()
            .AddHttpClientInstrumentation()
            .AddSource("MassTransit")  // MassTransit publishes to this ActivitySource
            .AddOtlpExporter(opts =>
                opts.Endpoint = new Uri("http://otel-collector:4317"));
    })
    .WithMetrics(metrics =>
    {
        metrics
            .AddMeter("MassTransit")   // MassTransit metrics
            .AddPrometheusExporter();
    });

// Key metrics from MassTransit:
// masstransit.receive      — messages received per endpoint
// masstransit.publish      — messages published
// masstransit.send         — messages sent
// masstransit.execute      — activity executions
// masstransit.handler      — consumer handler duration
// masstransit.consumer     — consumer metrics
```

---

## Messaging Patterns Quick Reference

| Pattern | Library | Use Case |
|---|---|---|
| Publish/Subscribe | MassTransit, Wolverine | Fan-out events to multiple consumers |
| Request/Response | MassTransit `IRequestClient` | Synchronous-style distributed calls |
| Transactional Outbox | MassTransit EF + Outbox | Atomic DB write + event publish |
| Saga Orchestration | MassTransit state machine | Complex multi-step workflows |
| Saga Choreography | Any pub/sub | Reactive, loosely-coupled flows |
| Idempotent Consumer | Inbox table / explicit check | At-least-once delivery safety |
| Delayed Messages | MassTransit scheduler | Future event triggers |
| Dead Letter Queue | MassTransit DLQ | Failed message handling |
| Batch Consumer | Wolverine batch handler | High-throughput bulk processing |
| Event Streaming | MassTransit Kafka | Event log, replay, audit |

### What Next?

- **Part 68**: Database Performance — EF Core compiled queries, query splitting, raw SQL, Dapper, N+1 prevention, indexing
- **Part 69**: Observability Deep Dive — OpenTelemetry distributed tracing, Grafana Tempo, Jaeger, structured logging, correlation IDs
- **Part 70**: Resilience Patterns — Polly v8 with `ResiliencePipeline`, Bulkhead, Fallback, Hedging, Circuit Breaker
