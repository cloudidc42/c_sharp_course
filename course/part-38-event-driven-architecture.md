# Part 38: Event-Driven Architecture

## Steps 1081-1120 | ระดับโลก (World-Class)

---

## Step 1081: Event-Driven Architecture คืออะไร

Event-Driven Architecture (EDA) คือแนวทางการออกแบบที่ services สื่อสารกันผ่าน **events** แทนการเรียกกันโดยตรง

```
Traditional (Synchronous):
OrderService → [HTTP Call] → InventoryService
             → [HTTP Call] → PaymentService
             → [HTTP Call] → EmailService
             (ถ้า 1 service down → ทั้งระบบ fail)

Event-Driven (Asynchronous):
OrderService → [Publish OrderConfirmed event]
                ↓
           Message Broker (RabbitMQ/Kafka)
                ↓         ↓         ↓
         Inventory  Payment    Email
         Service    Service    Service
         (แต่ละ service ทำงาน independently)
```

### ประโยชน์

- **Loose coupling** — services ไม่รู้จักกัน
- **High availability** — consumer down ≠ producer down
- **Scalability** — scale consumers independently
- **Eventual consistency** — ข้อมูลสอดคล้องกันในที่สุด
- **Event sourcing** — history ของ events

---

## Step 1082: Message Broker Concepts

```
Producer → [Exchange/Topic] → [Queue/Partition] → Consumer

RabbitMQ:
Producer → Exchange → Routing → Queue → Consumer
                      Key

Kafka:
Producer → Topic (Partition 0) → Consumer Group
              ↓ (Partition 1) → Consumer Group
              ↓ (Partition 2) → Consumer Group

Azure Service Bus:
Producer → Topic → Subscription (Filter) → Consumer
```

---

## Step 1083: Setup RabbitMQ + MassTransit

```xml
<!-- packages -->
<PackageReference Include="MassTransit" Version="8.3.3" />
<PackageReference Include="MassTransit.RabbitMQ" Version="8.3.3" />
<PackageReference Include="MassTransit.EntityFrameworkCore" Version="8.3.3" />
```

```csharp
// Infrastructure/Messaging/MassTransitConfiguration.cs
public static class MassTransitConfiguration
{
    public static IServiceCollection AddMessaging(
        this IServiceCollection services, IConfiguration config)
    {
        services.AddMassTransit(x =>
        {
            // Auto-register consumers from assembly
            x.AddConsumers(typeof(MassTransitConfiguration).Assembly);

            // Saga state machines
            x.AddSagaStateMachine<OrderProcessingSagaMachine, OrderProcessingSagaState>()
                .EntityFrameworkRepository(r =>
                {
                    r.ExistingDbContext<AppDbContext>();
                    r.UsePostgres();
                });

            x.UsingRabbitMq((context, cfg) =>
            {
                var rabbitSettings = config.GetSection("RabbitMQ").Get<RabbitMqSettings>()!;

                cfg.Host(rabbitSettings.Host, rabbitSettings.VirtualHost, h =>
                {
                    h.Username(rabbitSettings.Username);
                    h.Password(rabbitSettings.Password);
                });

                // Dead Letter Queue
                cfg.UseMessageRetry(r => r.Exponential(5,
                    TimeSpan.FromSeconds(1),
                    TimeSpan.FromSeconds(30),
                    TimeSpan.FromSeconds(5)));

                cfg.UseInMemoryOutbox(context);

                cfg.ConfigureEndpoints(context, new KebabCaseEndpointNameFormatter("app", false));
            });
        });

        return services;
    }
}
```

---

## Step 1084: Domain Events vs Integration Events

```csharp
// Domain/Events - published within same process (MediatR)
public record OrderConfirmedEvent(OrderId OrderId, CustomerId CustomerId) : DomainEvent;

// Application/IntegrationEvents - published to message broker
public record OrderConfirmedIntegrationEvent
{
    public Guid OrderId { get; init; }
    public Guid CustomerId { get; init; }
    public decimal TotalAmount { get; init; }
    public string Currency { get; init; } = "THB";
    public DateTime ConfirmedAt { get; init; }
    public IReadOnlyList<OrderItemDto> Items { get; init; } = [];
}

public record OrderItemDto(Guid ProductId, string ProductName, int Quantity, decimal UnitPrice);
```

---

## Step 1085: Outbox Pattern

Outbox pattern รับประกันว่า event จะถูกส่งหลัง transaction commit

```csharp
// Infrastructure/Outbox/OutboxMessage.cs
[Table("OutboxMessages")]
public class OutboxMessage
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public string Type { get; set; } = string.Empty;
    public string Content { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime? ProcessedAt { get; set; }
    public string? Error { get; set; }
    public int RetryCount { get; set; }
}

// Infrastructure/Outbox/OutboxInterceptor.cs
public class OutboxInterceptor : SaveChangesInterceptor
{
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData,
        InterceptionResult<int> result,
        CancellationToken ct = default)
    {
        var context = eventData.Context;
        if (context is null) return base.SavingChangesAsync(eventData, result, ct);

        // Capture all domain events and convert to outbox messages
        var aggregates = context.ChangeTracker
            .Entries<AggregateRoot<dynamic>>()
            .Select(e => e.Entity)
            .Where(a => a.DomainEvents.Any())
            .ToList();

        var outboxMessages = aggregates
            .SelectMany(a => a.DomainEvents)
            .Select(e => new OutboxMessage
            {
                Type = e.GetType().AssemblyQualifiedName!,
                Content = JsonSerializer.Serialize(e, e.GetType())
            })
            .ToList();

        context.Set<OutboxMessage>().AddRange(outboxMessages);

        foreach (var agg in aggregates)
            agg.ClearDomainEvents();

        return base.SavingChangesAsync(eventData, result, ct);
    }
}
```

```csharp
// Infrastructure/Outbox/OutboxProcessor.cs
public class OutboxProcessor(
    IServiceScopeFactory scopeFactory,
    ILogger<OutboxProcessor> logger) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await ProcessPendingMessagesAsync(stoppingToken);
            await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
        }
    }

    private async Task ProcessPendingMessagesAsync(CancellationToken ct)
    {
        using var scope = scopeFactory.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        var publishEndpoint = scope.ServiceProvider.GetRequiredService<IPublishEndpoint>();

        var messages = await db.OutboxMessages
            .Where(m => m.ProcessedAt == null && m.RetryCount < 5)
            .OrderBy(m => m.CreatedAt)
            .Take(20)
            .ToListAsync(ct);

        foreach (var message in messages)
        {
            try
            {
                var type = Type.GetType(message.Type)
                    ?? throw new InvalidOperationException($"Type not found: {message.Type}");

                var @event = JsonSerializer.Deserialize(message.Content, type)
                    ?? throw new InvalidOperationException("Failed to deserialize event");

                await publishEndpoint.Publish(@event, type, ct);

                message.ProcessedAt = DateTime.UtcNow;
                logger.LogInformation("Published outbox message {Id} of type {Type}",
                    message.Id, message.Type);
            }
            catch (Exception ex)
            {
                message.RetryCount++;
                message.Error = ex.Message;
                logger.LogError(ex, "Failed to process outbox message {Id}", message.Id);
            }
        }

        await db.SaveChangesAsync(ct);
    }
}
```

---

## Step 1086: Publishing Events

```csharp
// Application/Orders/EventHandlers/OrderConfirmedDomainEventHandler.cs
public class OrderConfirmedDomainEventHandler(
    IPublishEndpoint publishEndpoint,
    IOrderRepository orderRepository)
    : INotificationHandler<OrderConfirmedEvent>
{
    public async Task Handle(OrderConfirmedEvent notification, CancellationToken ct)
    {
        var order = await orderRepository.GetByIdAsync(notification.OrderId, ct);
        if (order is null) return;

        // Publish integration event to message broker
        await publishEndpoint.Publish(new OrderConfirmedIntegrationEvent
        {
            OrderId = notification.OrderId.Value,
            CustomerId = notification.CustomerId.Value,
            TotalAmount = order.TotalAmount.Amount,
            Currency = order.TotalAmount.Currency,
            ConfirmedAt = DateTime.UtcNow,
            Items = order.Lines.Select(l => new OrderItemDto(
                l.ProductId.Value,
                l.ProductName,
                l.Quantity.Value,
                l.UnitPrice.Amount)).ToList()
        }, ct);
    }
}
```

---

## Step 1087: Consumers - Inventory Service

```csharp
// Inventory.Service/Consumers/OrderConfirmedConsumer.cs
public class OrderConfirmedConsumer(
    IProductRepository productRepository,
    IPublishEndpoint publisher,
    ILogger<OrderConfirmedConsumer> logger)
    : IConsumer<OrderConfirmedIntegrationEvent>
{
    public async Task Consume(ConsumeContext<OrderConfirmedIntegrationEvent> context)
    {
        var message = context.Message;
        logger.LogInformation("Reserving stock for order {OrderId}", message.OrderId);

        var failures = new List<string>();

        foreach (var item in message.Items)
        {
            var product = await productRepository.GetByIdAsync(new ProductId(item.ProductId));
            if (product is null)
            {
                failures.Add($"Product {item.ProductId} not found");
                continue;
            }

            if (!product.HasSufficientStock(Quantity.Of(item.Quantity)))
            {
                failures.Add($"Insufficient stock for {product.Name}");
                continue;
            }

            product.ReserveStock(Quantity.Of(item.Quantity));
            await productRepository.UpdateAsync(product);
        }

        if (failures.Count > 0)
        {
            logger.LogWarning("Stock reservation failed for order {OrderId}: {Failures}",
                message.OrderId, string.Join(", ", failures));

            await publisher.Publish(new StockReservationFailedEvent
            {
                OrderId = message.OrderId,
                Reason = string.Join("; ", failures)
            }, context.CancellationToken);
        }
        else
        {
            await publisher.Publish(new StockReservedEvent
            {
                OrderId = message.OrderId,
                ReservedAt = DateTime.UtcNow
            }, context.CancellationToken);
        }
    }
}
```

---

## Step 1088: Consumers - Email Service

```csharp
// Notification.Service/Consumers/OrderConfirmedEmailConsumer.cs
public class OrderConfirmedEmailConsumer(
    IEmailService emailService,
    ICustomerQueryService customerQuery,
    ILogger<OrderConfirmedEmailConsumer> logger)
    : IConsumer<OrderConfirmedIntegrationEvent>
{
    public async Task Consume(ConsumeContext<OrderConfirmedIntegrationEvent> context)
    {
        var message = context.Message;

        var customer = await customerQuery.GetByIdAsync(message.CustomerId, context.CancellationToken);
        if (customer is null)
        {
            logger.LogWarning("Customer {CustomerId} not found for order confirmation email",
                message.CustomerId);
            return;
        }

        var itemsHtml = string.Join("",
            message.Items.Select(i =>
                $"<tr><td>{i.ProductName}</td><td>{i.Quantity}</td><td>{i.UnitPrice:C}</td></tr>"));

        var body = $"""
            <h2>Order Confirmed!</h2>
            <p>Thank you {customer.FullName}, your order has been confirmed.</p>
            <table>
              <thead><tr><th>Product</th><th>Qty</th><th>Price</th></tr></thead>
              <tbody>{itemsHtml}</tbody>
            </table>
            <p><strong>Total: {message.TotalAmount:C} {message.Currency}</strong></p>
            """;

        await emailService.SendAsync(
            customer.Email,
            $"Order #{message.OrderId:N} Confirmed",
            body,
            context.CancellationToken);

        logger.LogInformation("Order confirmation email sent to {Email}", customer.Email);
    }
}
```

---

## Step 1089: Saga State Machine

```csharp
// Application/Sagas/OrderProcessing/OrderProcessingSagaMachine.cs
public class OrderProcessingSagaMachine : MassTransitStateMachine<OrderProcessingSagaState>
{
    // States
    public State WaitingForStock { get; private set; } = default!;
    public State WaitingForPayment { get; private set; } = default!;
    public State Processing { get; private set; } = default!;
    public State Completed { get; private set; } = default!;
    public State Failed { get; private set; } = default!;

    // Events
    public Event<OrderConfirmedIntegrationEvent> OrderConfirmed { get; private set; } = default!;
    public Event<StockReservedEvent> StockReserved { get; private set; } = default!;
    public Event<StockReservationFailedEvent> StockReservationFailed { get; private set; } = default!;
    public Event<PaymentProcessedEvent> PaymentProcessed { get; private set; } = default!;
    public Event<PaymentFailedEvent> PaymentFailed { get; private set; } = default!;

    // Timeouts
    public Schedule<OrderProcessingSagaState, OrderProcessingTimeout> StockTimeout { get; private set; } = default!;

    public OrderProcessingSagaMachine()
    {
        InstanceState(x => x.CurrentState);

        // Correlation
        Event(() => OrderConfirmed, x => x.CorrelateById(m => m.Message.OrderId));
        Event(() => StockReserved, x => x.CorrelateById(m => m.Message.OrderId));
        Event(() => StockReservationFailed, x => x.CorrelateById(m => m.Message.OrderId));
        Event(() => PaymentProcessed, x => x.CorrelateById(m => m.Message.OrderId));
        Event(() => PaymentFailed, x => x.CorrelateById(m => m.Message.OrderId));

        // Timeout schedule: 5 minutes
        Schedule(() => StockTimeout, x => x.StockTimeoutToken,
            s => s.Delay = TimeSpan.FromMinutes(5));

        // Initial → WaitingForStock
        Initially(
            When(OrderConfirmed)
                .Then(ctx =>
                {
                    ctx.Saga.OrderId = ctx.Message.OrderId;
                    ctx.Saga.CustomerId = ctx.Message.CustomerId;
                    ctx.Saga.TotalAmount = ctx.Message.TotalAmount;
                })
                .Schedule(StockTimeout, ctx => new OrderProcessingTimeout { OrderId = ctx.Saga.OrderId })
                .TransitionTo(WaitingForStock));

        // WaitingForStock
        During(WaitingForStock,
            When(StockReserved)
                .Unschedule(StockTimeout)
                .Publish(ctx => new ProcessPaymentCommand
                {
                    OrderId = ctx.Saga.OrderId,
                    CustomerId = ctx.Saga.CustomerId,
                    Amount = ctx.Saga.TotalAmount
                })
                .TransitionTo(WaitingForPayment),

            When(StockReservationFailed)
                .Unschedule(StockTimeout)
                .Then(ctx => ctx.Saga.FailureReason = ctx.Message.Reason)
                .Publish(ctx => new OrderFailedIntegrationEvent
                {
                    OrderId = ctx.Saga.OrderId,
                    Reason = ctx.Saga.FailureReason
                })
                .TransitionTo(Failed)
                .Finalize(),

            When(StockTimeout.Received)
                .Then(ctx => ctx.Saga.FailureReason = "Stock reservation timed out")
                .TransitionTo(Failed)
                .Finalize());

        // WaitingForPayment
        During(WaitingForPayment,
            When(PaymentProcessed)
                .Publish(ctx => new OrderShipmentRequestedEvent { OrderId = ctx.Saga.OrderId })
                .TransitionTo(Completed)
                .Finalize(),

            When(PaymentFailed)
                .Then(ctx => ctx.Saga.FailureReason = ctx.Message.Reason)
                .Publish(ctx => new ReleaseStockCommand { OrderId = ctx.Saga.OrderId })
                .Publish(ctx => new OrderFailedIntegrationEvent
                {
                    OrderId = ctx.Saga.OrderId,
                    Reason = ctx.Saga.FailureReason
                })
                .TransitionTo(Failed)
                .Finalize());

        SetCompletedWhenFinalized();
    }
}

// Application/Sagas/OrderProcessing/OrderProcessingSagaState.cs
public class OrderProcessingSagaState : SagaStateMachineInstance
{
    public Guid CorrelationId { get; set; }
    public string CurrentState { get; set; } = string.Empty;
    public Guid OrderId { get; set; }
    public Guid CustomerId { get; set; }
    public decimal TotalAmount { get; set; }
    public string? FailureReason { get; set; }
    public Guid? StockTimeoutToken { get; set; }
}

public record OrderProcessingTimeout { public Guid OrderId { get; init; } }
```

---

## Step 1090: Kafka Integration

```xml
<PackageReference Include="MassTransit.Kafka" Version="8.3.3" />
<PackageReference Include="Confluent.Kafka" Version="2.6.0" />
```

```csharp
// Infrastructure/Messaging/KafkaConfiguration.cs
public static class KafkaConfiguration
{
    public static IServiceCollection AddKafkaMessaging(
        this IServiceCollection services, IConfiguration config)
    {
        services.AddMassTransit(x =>
        {
            x.UsingInMemory();  // for command bus

            x.AddRider(rider =>
            {
                // Producers
                rider.AddProducer<OrderConfirmedIntegrationEvent>("order-confirmed");
                rider.AddProducer<OrderCancelledIntegrationEvent>("order-cancelled");

                // Consumers
                rider.AddConsumer<OrderConfirmedConsumer>();
                rider.AddConsumer<OrderCancelledConsumer>();

                rider.UsingKafka((context, cfg) =>
                {
                    cfg.Host(config["Kafka:BootstrapServers"]);

                    // Consumer group
                    cfg.TopicEndpoint<OrderConfirmedIntegrationEvent>(
                        "order-confirmed",
                        config["Kafka:GroupId"]!,
                        e =>
                        {
                            e.ConfigureConsumer<OrderConfirmedConsumer>(context);
                            e.AutoOffsetReset = AutoOffsetReset.Earliest;
                            e.CheckpointInterval = TimeSpan.FromSeconds(10);
                            e.CheckpointMessageCount = 100;
                        });

                    cfg.TopicEndpoint<OrderCancelledIntegrationEvent>(
                        "order-cancelled",
                        config["Kafka:GroupId"]!,
                        e => e.ConfigureConsumer<OrderCancelledConsumer>(context));
                });
            });
        });

        return services;
    }
}
```

```csharp
// Publishing to Kafka
public class OrderConfirmedDomainEventHandler(ITopicProducer<OrderConfirmedIntegrationEvent> producer)
    : INotificationHandler<OrderConfirmedEvent>
{
    public async Task Handle(OrderConfirmedEvent notification, CancellationToken ct)
    {
        await producer.Produce(new OrderConfirmedIntegrationEvent
        {
            OrderId = notification.OrderId.Value,
            CustomerId = notification.CustomerId.Value
        }, ct);
    }
}
```

---

## Step 1091: Event Store (EventStoreDB)

```xml
<PackageReference Include="EventStore.Client.Grpc.Streams" Version="23.0.0" />
```

```csharp
// Infrastructure/EventSourcing/EventStoreRepository.cs
public class EventStoreRepository<T>(EventStoreClient client)
    where T : AggregateRoot, IEventSourcedAggregate, new()
{
    private static readonly JsonSerializerOptions JsonOptions = new()
    {
        PropertyNameCaseInsensitive = true
    };

    public async Task<T?> GetByIdAsync(Guid id, CancellationToken ct = default)
    {
        var streamName = GetStreamName(id);

        try
        {
            var events = new List<IDomainEvent>();
            var result = client.ReadStreamAsync(Direction.Forwards,
                streamName,
                StreamPosition.Start,
                cancellationToken: ct);

            await foreach (var resolvedEvent in result)
            {
                var eventType = Type.GetType(
                    Encoding.UTF8.GetString(resolvedEvent.Event.Metadata.Span));

                if (eventType is null) continue;

                var domainEvent = (IDomainEvent?)JsonSerializer.Deserialize(
                    resolvedEvent.Event.Data.Span, eventType, JsonOptions);

                if (domainEvent is not null) events.Add(domainEvent);
            }

            if (events.Count == 0) return null;

            var aggregate = new T();
            aggregate.LoadFromHistory(events);
            return aggregate;
        }
        catch (StreamNotFoundException)
        {
            return null;
        }
    }

    public async Task SaveAsync(T aggregate, CancellationToken ct = default)
    {
        var uncommitted = aggregate.UncommittedEvents;
        if (uncommitted.Count == 0) return;

        var streamName = GetStreamName(aggregate.Id.Value);
        var expectedRevision = aggregate.Version == uncommitted.Count
            ? StreamRevision.None  // new stream
            : StreamRevision.FromInt64(aggregate.Version - uncommitted.Count - 1);

        var eventData = uncommitted.Select(e =>
        {
            var eventType = e.GetType().AssemblyQualifiedName!;
            var data = JsonSerializer.SerializeToUtf8Bytes(e, e.GetType(), JsonOptions);
            var metadata = Encoding.UTF8.GetBytes(eventType);

            return new EventData(
                Uuid.NewUuid(),
                e.GetType().Name,
                data,
                metadata);
        });

        await client.AppendToStreamAsync(
            streamName,
            expectedRevision,
            eventData,
            cancellationToken: ct);

        aggregate.MarkEventsAsCommitted();
    }

    private static string GetStreamName(Guid id) => $"{typeof(T).Name.ToLower()}-{id:N}";
}
```

---

## Step 1092: CQRS with Event Sourcing

```csharp
// Domain/Accounts/Account.cs - Event Sourced Aggregate
public class Account : IEventSourcedAggregate
{
    private readonly List<IDomainEvent> _uncommitted = [];
    private readonly List<AccountTransaction> _transactions = [];

    public Guid Id { get; private set; }
    public string OwnerId { get; private set; } = string.Empty;
    public decimal Balance { get; private set; }
    public bool IsActive { get; private set; }
    public int Version { get; private set; }

    public IReadOnlyList<IDomainEvent> UncommittedEvents => _uncommitted.AsReadOnly();
    public IReadOnlyList<AccountTransaction> Transactions => _transactions.AsReadOnly();

    public static Account Open(Guid id, string ownerId, decimal initialDeposit = 0)
    {
        var account = new Account();
        account.Apply(new AccountOpenedEvent(id, ownerId, initialDeposit));
        return account;
    }

    public void Deposit(decimal amount, string description)
    {
        if (!IsActive) throw new DomainException("Account is closed");
        if (amount <= 0) throw new DomainException("Deposit must be positive");

        Apply(new MoneyDepositedEvent(Id, amount, description, DateTime.UtcNow));
    }

    public void Withdraw(decimal amount, string description)
    {
        if (!IsActive) throw new DomainException("Account is closed");
        if (amount <= 0) throw new DomainException("Withdrawal must be positive");
        if (Balance < amount) throw new DomainException($"Insufficient balance. Available: {Balance}");

        Apply(new MoneyWithdrawnEvent(Id, amount, description, DateTime.UtcNow));
    }

    public void Close()
    {
        if (!IsActive) throw new DomainException("Account is already closed");
        if (Balance > 0) throw new DomainException("Cannot close account with balance");
        Apply(new AccountClosedEvent(Id, DateTime.UtcNow));
    }

    private void Apply(IDomainEvent @event)
    {
        When(@event);
        _uncommitted.Add(@event);
        Version++;
    }

    private void When(IDomainEvent @event)
    {
        switch (@event)
        {
            case AccountOpenedEvent e:
                Id = e.AccountId;
                OwnerId = e.OwnerId;
                Balance = e.InitialDeposit;
                IsActive = true;
                if (e.InitialDeposit > 0)
                    _transactions.Add(new AccountTransaction(e.InitialDeposit, "Initial deposit", e.OccurredAt));
                break;

            case MoneyDepositedEvent e:
                Balance += e.Amount;
                _transactions.Add(new AccountTransaction(e.Amount, e.Description, e.Timestamp));
                break;

            case MoneyWithdrawnEvent e:
                Balance -= e.Amount;
                _transactions.Add(new AccountTransaction(-e.Amount, e.Description, e.Timestamp));
                break;

            case AccountClosedEvent:
                IsActive = false;
                break;
        }
    }

    public void LoadFromHistory(IEnumerable<IDomainEvent> history)
    {
        foreach (var @event in history)
        {
            When(@event);
            Version++;
        }
    }

    public void MarkEventsAsCommitted() => _uncommitted.Clear();
}

public record AccountTransaction(decimal Amount, string Description, DateTime Timestamp);
```

---

## Step 1093: Event Projections / Read Models

```csharp
// Infrastructure/Projections/AccountProjection.cs
public class AccountProjection(ReadDbContext db) :
    INotificationHandler<AccountOpenedEvent>,
    INotificationHandler<MoneyDepositedEvent>,
    INotificationHandler<MoneyWithdrawnEvent>,
    INotificationHandler<AccountClosedEvent>
{
    public async Task Handle(AccountOpenedEvent e, CancellationToken ct)
    {
        var view = new AccountView
        {
            Id = e.AccountId,
            OwnerId = e.OwnerId,
            Balance = e.InitialDeposit,
            IsActive = true,
            OpenedAt = e.OccurredAt,
            LastUpdatedAt = e.OccurredAt
        };
        await db.AccountViews.AddAsync(view, ct);
        await db.SaveChangesAsync(ct);
    }

    public async Task Handle(MoneyDepositedEvent e, CancellationToken ct)
    {
        var view = await db.AccountViews.FindAsync([e.AccountId], ct);
        if (view is null) return;
        view.Balance += e.Amount;
        view.LastUpdatedAt = e.Timestamp;
        await db.SaveChangesAsync(ct);
    }

    public async Task Handle(MoneyWithdrawnEvent e, CancellationToken ct)
    {
        var view = await db.AccountViews.FindAsync([e.AccountId], ct);
        if (view is null) return;
        view.Balance -= e.Amount;
        view.LastUpdatedAt = e.Timestamp;
        await db.SaveChangesAsync(ct);
    }

    public async Task Handle(AccountClosedEvent e, CancellationToken ct)
    {
        var view = await db.AccountViews.FindAsync([e.AccountId], ct);
        if (view is null) return;
        view.IsActive = false;
        view.ClosedAt = e.OccurredAt;
        view.LastUpdatedAt = e.OccurredAt;
        await db.SaveChangesAsync(ct);
    }
}
```

---

## Step 1094: Dead Letter Queue Handling

```csharp
// Infrastructure/Messaging/Consumers/DeadLetterConsumer.cs
public class DeadLetterConsumer(
    IDeadLetterRepository repository,
    IAlertService alertService,
    ILogger<DeadLetterConsumer> logger)
    : IConsumer<Fault<OrderConfirmedIntegrationEvent>>
{
    public async Task Consume(ConsumeContext<Fault<OrderConfirmedIntegrationEvent>> context)
    {
        var fault = context.Message;
        logger.LogError(
            "Message fault for OrderId {OrderId}: {Exceptions}",
            fault.Message.OrderId,
            string.Join(", ", fault.Exceptions.Select(e => e.Message)));

        // Store in dead letter table
        await repository.SaveAsync(new DeadLetterEntry
        {
            MessageType = typeof(OrderConfirmedIntegrationEvent).Name,
            MessageContent = JsonSerializer.Serialize(fault.Message),
            FaultReason = string.Join("; ", fault.Exceptions.Select(e => e.Message)),
            FaultedAt = DateTime.UtcNow,
            RetryCount = fault.RetryCount
        });

        // Alert on repeated failures
        if (fault.RetryCount >= 3)
        {
            await alertService.SendCriticalAlertAsync(
                "Dead Letter Queue",
                $"Message for OrderId {fault.Message.OrderId} failed {fault.RetryCount} times");
        }
    }
}
```

---

## Step 1095: Event Replay

สามารถ replay events เพื่อสร้าง read model ใหม่

```csharp
// Infrastructure/EventSourcing/EventReplayer.cs
public class EventReplayer(
    EventStoreClient eventStore,
    IServiceScopeFactory scopeFactory,
    ILogger<EventReplayer> logger)
{
    public async Task ReplayAllAsync(string streamPrefix, CancellationToken ct = default)
    {
        logger.LogInformation("Starting event replay for streams with prefix: {Prefix}", streamPrefix);
        var count = 0;

        var streams = eventStore.ReadAllAsync(Direction.Forwards, Position.Start, cancellationToken: ct);

        await foreach (var resolvedEvent in streams)
        {
            if (!resolvedEvent.OriginalStreamId.StartsWith(streamPrefix)) continue;

            var eventTypeName = Encoding.UTF8.GetString(resolvedEvent.Event.Metadata.Span);
            var eventType = Type.GetType(eventTypeName);
            if (eventType is null) continue;

            var @event = (IDomainEvent?)JsonSerializer.Deserialize(
                resolvedEvent.Event.Data.Span, eventType);
            if (@event is null) continue;

            using var scope = scopeFactory.CreateScope();
            var publisher = scope.ServiceProvider.GetRequiredService<IPublisher>();
            await publisher.Publish(@event, ct);

            count++;
            if (count % 100 == 0)
                logger.LogInformation("Replayed {Count} events...", count);
        }

        logger.LogInformation("Event replay completed. Total events: {Count}", count);
    }
}
```

---

## Step 1096: Azure Service Bus Integration

```xml
<PackageReference Include="MassTransit.Azure.ServiceBus.Core" Version="8.3.3" />
```

```csharp
// Infrastructure/Messaging/ServiceBusConfiguration.cs
public static class ServiceBusConfiguration
{
    public static IServiceCollection AddAzureServiceBus(
        this IServiceCollection services, IConfiguration config)
    {
        services.AddMassTransit(x =>
        {
            x.AddConsumers(typeof(ServiceBusConfiguration).Assembly);

            x.UsingAzureServiceBus((context, cfg) =>
            {
                cfg.Host(config["AzureServiceBus:ConnectionString"]);

                // Topics and Subscriptions
                cfg.Message<OrderConfirmedIntegrationEvent>(m =>
                    m.SetEntityName("order-confirmed"));

                cfg.SubscriptionEndpoint<OrderConfirmedIntegrationEvent>(
                    "inventory-service",
                    e =>
                    {
                        e.ConfigureConsumer<OrderConfirmedConsumer>(context);
                        e.MaxDeliveryCount = 5;
                        e.DefaultMessageTimeToLive = TimeSpan.FromDays(7);
                        e.LockDuration = TimeSpan.FromMinutes(5);
                    });

                cfg.SubscriptionEndpoint<OrderConfirmedIntegrationEvent>(
                    "notification-service",
                    e => e.ConfigureConsumer<OrderConfirmedEmailConsumer>(context));

                cfg.ConfigureEndpoints(context);
            });
        });

        return services;
    }
}
```

---

## Step 1097: Event Versioning

```csharp
// Events ต้องรองรับหลาย versions
// V1 (เก่า)
public record OrderConfirmedIntegrationEventV1
{
    public Guid OrderId { get; init; }
    public Guid CustomerId { get; init; }
}

// V2 (ใหม่ - เพิ่ม fields)
public record OrderConfirmedIntegrationEventV2
{
    public Guid OrderId { get; init; }
    public Guid CustomerId { get; init; }
    public decimal TotalAmount { get; init; }  // new
    public string Currency { get; init; } = "THB";  // new
    public IReadOnlyList<OrderItemDto> Items { get; init; } = [];  // new
}

// Upcaster: แปลง V1 → V2
public class OrderConfirmedV1ToV2Upcaster
    : IConsumer<OrderConfirmedIntegrationEventV1>
{
    public async Task Consume(ConsumeContext<OrderConfirmedIntegrationEventV1> context)
    {
        // แปลงเป็น V2 แล้ว publish ต่อ
        await context.Publish(new OrderConfirmedIntegrationEventV2
        {
            OrderId = context.Message.OrderId,
            CustomerId = context.Message.CustomerId,
            TotalAmount = 0,   // unknown from V1, may need lookup
            Currency = "THB",
            Items = []
        });
    }
}
```

---

## Step 1098: Exactly-Once Processing

```csharp
// Infrastructure/Idempotency/IdempotentConsumerFilter.cs
public class IdempotentConsumerFilter<T>(IProcessedMessageStore store)
    : IFilter<ConsumeContext<T>> where T : class
{
    public async Task Send(ConsumeContext<T> context, IPipe<ConsumeContext<T>> next)
    {
        var messageId = context.MessageId?.ToString() ?? Guid.NewGuid().ToString();

        if (await store.HasBeenProcessedAsync(messageId, context.CancellationToken))
        {
            // Already processed - skip
            return;
        }

        await next.Send(context);

        await store.MarkAsProcessedAsync(messageId, context.CancellationToken);
    }

    public void Probe(ProbeContext context) { }
}

// Infrastructure/Idempotency/ProcessedMessageStore.cs
public class ProcessedMessageStore(IDistributedCache cache) : IProcessedMessageStore
{
    public async Task<bool> HasBeenProcessedAsync(string messageId, CancellationToken ct)
    {
        var value = await cache.GetStringAsync($"msg:{messageId}", ct);
        return value is not null;
    }

    public async Task MarkAsProcessedAsync(string messageId, CancellationToken ct)
    {
        await cache.SetStringAsync(
            $"msg:{messageId}",
            "processed",
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromDays(1)
            },
            ct);
    }
}
```

---

## Step 1099: Event-Driven Integration Tests

```csharp
// Tests/Integration/EventDrivenTests.cs
public class EventDrivenTests : IClassFixture<TestHarness>
{
    private readonly ITestHarness _harness;
    private readonly IConsumerTestHarness<OrderConfirmedConsumer> _consumerHarness;

    public EventDrivenTests(TestHarness factory)
    {
        _harness = factory.Services.GetRequiredService<ITestHarness>();
        _consumerHarness = factory.Services.GetRequiredService<IConsumerTestHarness<OrderConfirmedConsumer>>();
    }

    [Fact]
    public async Task OrderConfirmed_ShouldTriggerStockReservation()
    {
        // Arrange
        var @event = new OrderConfirmedIntegrationEvent
        {
            OrderId = Guid.NewGuid(),
            CustomerId = Guid.NewGuid(),
            Items = [new OrderItemDto(Guid.NewGuid(), "Product A", 2, 100)]
        };

        // Act
        await _harness.Bus.Publish(@event);

        // Assert
        (await _consumerHarness.Consumed.Any<OrderConfirmedIntegrationEvent>())
            .Should().BeTrue();

        (await _harness.Published.Any<StockReservedEvent>())
            .Should().BeTrue();
    }

    [Fact]
    public async Task SagaStateMachine_OrderConfirmed_ShouldProgressToWaitingForPayment()
    {
        var sagaHarness = _harness.GetSagaStateMachineHarness<
            OrderProcessingSagaMachine, OrderProcessingSagaState>();

        var orderId = Guid.NewGuid();
        await _harness.Bus.Publish(new OrderConfirmedIntegrationEvent { OrderId = orderId });
        await _harness.Bus.Publish(new StockReservedEvent { OrderId = orderId });

        var saga = await sagaHarness.Created
            .SelectAsync(x => x.Saga.OrderId == orderId)
            .FirstOrDefaultAsync();

        saga.Should().NotBeNull();
        saga!.CurrentState.Should().Be("WaitingForPayment");
    }
}
```

---

## Step 1100: Event-Driven Architecture Patterns Summary

```
Patterns ที่ครอบคลุม:
┌─────────────────────────────────────────────────────────┐
│              Event-Driven Patterns                       │
│                                                          │
│  ┌──────────────────┐  ┌──────────────────────────────┐ │
│  │   Pub/Sub        │  │   Saga / Process Manager     │ │
│  │ - Producer       │  │ - State machine              │ │
│  │ - Consumer       │  │ - Compensating transactions  │ │
│  │ - Dead Letter Q  │  │ - Timeout handling           │ │
│  └──────────────────┘  └──────────────────────────────┘ │
│                                                          │
│  ┌──────────────────┐  ┌──────────────────────────────┐ │
│  │  Event Sourcing  │  │    CQRS                      │ │
│  │ - EventStore     │  │ - Command side (Domain)      │ │
│  │ - Projections    │  │ - Query side (Read Model)    │ │
│  │ - Event Replay   │  │ - Eventual consistency       │ │
│  └──────────────────┘  └──────────────────────────────┘ │
│                                                          │
│  ┌──────────────────┐  ┌──────────────────────────────┐ │
│  │  Outbox Pattern  │  │  Idempotent Consumers        │ │
│  │ - At-least-once  │  │ - Exactly-once processing    │ │
│  │ - Transactional  │  │ - Message deduplication      │ │
│  └──────────────────┘  └──────────────────────────────┘ │
│                                                          │
│  Message Brokers: RabbitMQ, Kafka, Azure Service Bus    │
│  Framework: MassTransit                                  │
└─────────────────────────────────────────────────────────┘
```

---

## Step 1101: Advanced - Event Choreography vs Orchestration

```
Choreography (Decentralized):
OrderService → [OrderConfirmed] →
    InventoryService → [StockReserved] →
        PaymentService → [PaymentProcessed] →
            ShippingService → [Shipped]

✅ Loose coupling
❌ Hard to track overall flow
❌ Business logic scattered


Orchestration (Centralized - Saga):
OrderSaga orchestrates:
1. → InventoryService.ReserveStock
2. ← StockReserved
3. → PaymentService.ProcessPayment
4. ← PaymentProcessed
5. → ShippingService.RequestShipment

✅ Clear business flow in one place
✅ Easier to track and debug
❌ Saga service becomes complex
```

---

## Step 1102: Inbox Pattern

Inbox pattern รับประกัน event processing ครั้งเดียว (exactly-once semantics)

```csharp
// Infrastructure/Inbox/InboxMessage.cs
[Table("InboxMessages")]
public class InboxMessage
{
    public Guid Id { get; set; }
    public string Type { get; set; } = string.Empty;
    public string Content { get; set; } = string.Empty;
    public DateTime ReceivedAt { get; set; }
    public DateTime? ProcessedAt { get; set; }
    public string? Error { get; set; }
}

// Infrastructure/Inbox/InboxConsumerFilter.cs
public class InboxConsumerFilter<T>(AppDbContext db, ILogger logger)
    : IFilter<ConsumeContext<T>> where T : class
{
    public async Task Send(ConsumeContext<T> context, IPipe<ConsumeContext<T>> next)
    {
        var messageId = context.MessageId ?? throw new InvalidOperationException("MessageId required");
        var idStr = messageId.ToString();

        // Check if already processed
        var existing = await db.InboxMessages
            .FirstOrDefaultAsync(m => m.Id == messageId, context.CancellationToken);

        if (existing?.ProcessedAt != null) return;  // already processed

        if (existing is null)
        {
            // Store in inbox first
            await db.InboxMessages.AddAsync(new InboxMessage
            {
                Id = messageId,
                Type = typeof(T).AssemblyQualifiedName!,
                Content = JsonSerializer.Serialize(context.Message),
                ReceivedAt = DateTime.UtcNow
            }, context.CancellationToken);
            await db.SaveChangesAsync(context.CancellationToken);
        }

        // Process
        await next.Send(context);

        // Mark as processed
        existing = await db.InboxMessages.FindAsync([messageId], context.CancellationToken);
        if (existing is not null)
        {
            existing.ProcessedAt = DateTime.UtcNow;
            await db.SaveChangesAsync(context.CancellationToken);
        }
    }

    public void Probe(ProbeContext context) { }
}
```

---

## Step 1103: Event-Driven Monitoring

```csharp
// Infrastructure/Messaging/Observers/MetricsObserver.cs
public class MetricsObserver(Meter meter) : IReceiveObserver, ISendObserver, IPublishObserver
{
    private readonly Counter<long> _messagesReceived = meter.CreateCounter<long>("messages.received");
    private readonly Counter<long> _messagesSent = meter.CreateCounter<long>("messages.sent");
    private readonly Counter<long> _messagesFailed = meter.CreateCounter<long>("messages.failed");
    private readonly Histogram<double> _processingTime = meter.CreateHistogram<double>("messages.processing_time_ms");

    public Task PreReceive(ReceiveContext context)
    {
        _messagesReceived.Add(1, new TagList { { "message_type", context.InputAddress.AbsolutePath } });
        return Task.CompletedTask;
    }

    public Task PostReceive(ReceiveContext context)
    {
        _processingTime.Record(
            context.ElapsedTime.TotalMilliseconds,
            new TagList { { "message_type", context.InputAddress.AbsolutePath } });
        return Task.CompletedTask;
    }

    public Task ReceiveFault(ReceiveContext context, Exception exception)
    {
        _messagesFailed.Add(1, new TagList
        {
            { "message_type", context.InputAddress.AbsolutePath },
            { "exception_type", exception.GetType().Name }
        });
        return Task.CompletedTask;
    }

    public Task PreSend<T>(SendContext<T> context) where T : class
    {
        _messagesSent.Add(1, new TagList { { "message_type", typeof(T).Name } });
        return Task.CompletedTask;
    }

    public Task PostSend<T>(SendContext<T> context) where T : class => Task.CompletedTask;
    public Task SendFault<T>(SendContext<T> context, Exception exception) where T : class => Task.CompletedTask;
    public Task PrePublish<T>(PublishContext<T> context) where T : class => Task.CompletedTask;
    public Task PostPublish<T>(PublishContext<T> context) where T : class => Task.CompletedTask;
    public Task PublishFault<T>(PublishContext<T> context, Exception exception) where T : class => Task.CompletedTask;
}
```

---

## Step 1104: Event-Driven API Gateway Pattern

```csharp
// API/Controllers/OrdersController.cs - async API
[HttpPost("async")]
[ProducesResponseType(typeof(AsyncOperationResponse), StatusCodes.Status202Accepted)]
public async Task<IActionResult> PlaceOrderAsync(
    [FromBody] PlaceOrderRequest request,
    CancellationToken ct)
{
    var correlationId = Guid.NewGuid();

    // Publish command to message bus
    await publishEndpoint.Publish(new PlaceOrderAsyncCommand
    {
        CorrelationId = correlationId,
        CustomerId = request.CustomerId,
        Lines = request.Lines
    }, ct);

    // Return 202 Accepted with location to check status
    return Accepted(new AsyncOperationResponse
    {
        CorrelationId = correlationId,
        StatusUrl = Url.Action("GetOrderStatus", new { correlationId }),
        EstimatedCompletionTime = DateTime.UtcNow.AddSeconds(5)
    });
}

[HttpGet("status/{correlationId:guid}")]
public async Task<IActionResult> GetOrderStatus(Guid correlationId, CancellationToken ct)
{
    var status = await orderStatusQuery.GetByCorrelationIdAsync(correlationId, ct);
    if (status is null) return NotFound();

    return status.IsCompleted
        ? Ok(status)
        : Accepted(status);  // still processing
}

public record AsyncOperationResponse(
    Guid CorrelationId,
    string? StatusUrl,
    DateTime EstimatedCompletionTime);
```

---

## Step 1105: Complete Event-Driven Flow Example

```csharp
/*
 * Complete flow: Place Order → Reserve Stock → Process Payment → Ship
 *
 * 1. API receives PlaceOrder request
 * 2. OrderService creates Order (DDD Aggregate)
 * 3. OrderConfirmed domain event → Integration event
 * 4. RabbitMQ/Kafka receives OrderConfirmedIntegrationEvent
 * 5. OrderProcessingSaga starts
 * 6. Saga sends StockReservationCommand to InventoryService
 * 7. InventoryService publishes StockReservedEvent
 * 8. Saga receives StockReservedEvent → sends ProcessPaymentCommand
 * 9. PaymentService publishes PaymentProcessedEvent
 * 10. Saga receives → Order complete, publishes OrderShipmentRequestedEvent
 * 11. ShippingService creates shipment
 * 12. EmailService sends confirmation
 *
 * If any step fails:
 * - Saga receives failure event
 * - Saga runs compensating transactions (reverse completed steps)
 * - OrderFailedIntegrationEvent published
 * - API/Customer notified of failure
 */
```

---

## Step 1106-1110: อ่านเพิ่มเติม

```
Resources สำหรับ Event-Driven Architecture:

Books:
- "Enterprise Integration Patterns" - Hohpe & Woolf
- "Designing Event-Driven Systems" - Ben Stopford (Kafka focused)
- "Building Microservices" - Sam Newman

Tools:
- RabbitMQ Management UI: http://localhost:15672
- Kafka UI: http://localhost:8080 (kafka-ui)
- MassTransit Documentation: masstransit.io
- EventStoreDB Documentation: eventstore.com

Patterns:
- CQRS + Event Sourcing
- Saga / Process Manager
- Outbox Pattern
- Inbox Pattern
- Event Replay
- Snapshot for large aggregates
```

---

## Step 1111: Event Schema Registry

```csharp
// Infrastructure/Messaging/SchemaRegistry.cs
public class EventSchemaRegistry
{
    private static readonly Dictionary<string, Type> _schemas = new();

    static EventSchemaRegistry()
    {
        // Register all event types
        Register<OrderConfirmedIntegrationEvent>("order.confirmed.v1");
        Register<OrderConfirmedIntegrationEventV2>("order.confirmed.v2");
        Register<StockReservedEvent>("inventory.stock.reserved.v1");
        Register<PaymentProcessedEvent>("payment.processed.v1");
    }

    public static void Register<T>(string schemaId) where T : class
        => _schemas[schemaId] = typeof(T);

    public static Type? Resolve(string schemaId)
        => _schemas.GetValueOrDefault(schemaId);

    public static string? GetSchemaId(Type type)
        => _schemas.FirstOrDefault(kv => kv.Value == type).Key;

    public static IReadOnlyDictionary<string, Type> GetAll() => _schemas.AsReadOnly();
}
```

---

## Step 1112-1120: Advanced Topics

```csharp
// Competing Consumers Pattern - หลาย consumers แข่งกัน process
// MassTransit handles this automatically with consumer groups

// Fan-out Pattern - ส่ง event ให้ทุก consumers
services.AddMassTransit(x =>
{
    x.AddConsumers(typeof(Program).Assembly); // all consumers
    x.UsingRabbitMq((ctx, cfg) =>
    {
        // Direct exchange = competing consumers (load balanced)
        // Fanout exchange = all consumers receive
        cfg.Message<OrderConfirmedIntegrationEvent>(m =>
            m.SetEntityName("order-events"));

        // Fanout
        cfg.Publish<OrderConfirmedIntegrationEvent>(p =>
            p.ExchangeType = ExchangeType.Fanout);
    });
});
```

```csharp
// Priority Queue Pattern
cfg.ReceiveEndpoint("high-priority-orders", e =>
{
    e.PrefetchCount = 10;
    e.UsePriority(10); // Max priority 10
    e.ConfigureConsumer<HighPriorityOrderConsumer>(context);
});

cfg.ReceiveEndpoint("normal-orders", e =>
{
    e.PrefetchCount = 50;
    e.ConfigureConsumer<OrderConfirmedConsumer>(context);
});
```

---

*Part 38 ครอบคลุม Event-Driven Architecture ทั้งหมด 40 Steps (1081-1120)*  
*ต่อไป Part 39: Advanced Concurrency & Parallelism*
