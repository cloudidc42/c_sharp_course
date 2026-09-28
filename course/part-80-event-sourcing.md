# Part 80: Event Sourcing & CQRS with EventStoreDB

## Steps 1909–1924

---

## Step 1909: Event Sourcing Fundamentals

Instead of storing current state, store the sequence of events that led to it.

```
Traditional (state store):
  Order { Status: "Shipped", Items: [...], UpdatedAt: ... }

Event Sourced (event store):
  OrderCreated { ... }
  OrderItemAdded { ... }
  OrderItemAdded { ... }
  OrderConfirmed { ... }
  OrderShipped { ... }

Rebuild state: replay events in order → current state
```

**Benefits**:
- Full audit trail — every change captured
- Time travel — reconstruct state at any point in time
- Event replay — rebuild projections, fix bugs
- Temporal decoupling — consumers process events independently

**Trade-offs**:
- No simple UPDATE / DELETE — append only
- Eventual consistency for query models
- Snapshots needed for aggregates with many events

```bash
# Docker: EventStoreDB
docker run --rm -it -p 2113:2113 -p 1113:1113 \
  eventstore/eventstore:latest \
  --dev --insecure

dotnet add package EventStore.Client.Grpc.Streams
dotnet add package EventStore.Client.Grpc.ProjectionManagement
```

---

## Step 1910: Domain Events — The Building Blocks

```csharp
// Domain/Events/IDomainEvent.cs
namespace Orders.Domain.Events;

public interface IDomainEvent
{
    Guid EventId { get; }
    DateTimeOffset OccurredAt { get; }
    string EventType { get; }
}

public abstract record DomainEvent : IDomainEvent
{
    public Guid EventId { get; init; } = Guid.NewGuid();
    public DateTimeOffset OccurredAt { get; init; } = DateTimeOffset.UtcNow;
    public abstract string EventType { get; }
}
```

```csharp
// Domain/Events/OrderEvents.cs
namespace Orders.Domain.Events;

public record OrderCreated : DomainEvent
{
    public override string EventType => nameof(OrderCreated);
    public required Guid OrderId { get; init; }
    public required Guid CustomerId { get; init; }
    public required string ShippingStreet { get; init; }
    public required string ShippingCity { get; init; }
    public required string ShippingCountry { get; init; }
}

public record OrderItemAdded : DomainEvent
{
    public override string EventType => nameof(OrderItemAdded);
    public required Guid OrderId { get; init; }
    public required Guid ItemId { get; init; }
    public required Guid ProductId { get; init; }
    public required string Sku { get; init; }
    public required int Quantity { get; init; }
    public required decimal UnitPrice { get; init; }
}

public record OrderConfirmed : DomainEvent
{
    public override string EventType => nameof(OrderConfirmed);
    public required Guid OrderId { get; init; }
    public required decimal TotalAmount { get; init; }
}

public record OrderShipped : DomainEvent
{
    public override string EventType => nameof(OrderShipped);
    public required Guid OrderId { get; init; }
    public required string TrackingNumber { get; init; }
    public required string Carrier { get; init; }
}

public record OrderCancelled : DomainEvent
{
    public override string EventType => nameof(OrderCancelled);
    public required Guid OrderId { get; init; }
    public required string Reason { get; init; }
}
```

---

## Step 1911: Event-Sourced Aggregate

```csharp
// Domain/Aggregates/EventSourcedAggregate.cs
namespace Orders.Domain.Aggregates;

public abstract class EventSourcedAggregate
{
    private readonly List<IDomainEvent> _uncommittedEvents = [];

    public Guid Id { get; protected set; }
    public long Version { get; protected set; } = -1;

    public IReadOnlyList<IDomainEvent> UncommittedEvents => _uncommittedEvents.AsReadOnly();

    // Apply event and update version
    protected void Raise(IDomainEvent @event)
    {
        Apply(@event);
        _uncommittedEvents.Add(@event);
        Version++;
    }

    // Replay from history (no side effects)
    public void LoadFromHistory(IEnumerable<IDomainEvent> events)
    {
        foreach (var @event in events)
        {
            Apply(@event);
            Version++;
        }
    }

    protected abstract void Apply(IDomainEvent @event);

    public void MarkEventsAsCommitted() => _uncommittedEvents.Clear();
}
```

```csharp
// Domain/Aggregates/Order.cs
using Orders.Domain.Events;

namespace Orders.Domain.Aggregates;

public sealed class Order : EventSourcedAggregate
{
    private readonly List<OrderItem> _items = [];

    public Guid CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    public Address ShippingAddress { get; private set; } = null!;
    public decimal TotalAmount { get; private set; }
    public IReadOnlyList<OrderItem> Items => _items.AsReadOnly();

    private Order() { } // For reconstitution

    // Factory — creates new order via event
    public static Order Create(Guid customerId, Address shippingAddress)
    {
        var order = new Order();
        order.Raise(new OrderCreated
        {
            OrderId = Guid.NewGuid(),
            CustomerId = customerId,
            ShippingStreet = shippingAddress.Street,
            ShippingCity = shippingAddress.City,
            ShippingCountry = shippingAddress.Country
        });
        return order;
    }

    public void AddItem(Guid productId, string sku, int quantity, decimal unitPrice)
    {
        if (Status != OrderStatus.Draft)
            throw new InvalidOrderStateException("Can only add items to draft orders");

        Raise(new OrderItemAdded
        {
            OrderId = Id,
            ItemId = Guid.NewGuid(),
            ProductId = productId,
            Sku = sku,
            Quantity = quantity,
            UnitPrice = unitPrice
        });
    }

    public void Confirm()
    {
        if (Status != OrderStatus.Draft)
            throw new InvalidOrderStateException("Can only confirm draft orders");
        if (!_items.Any())
            throw new InvalidOrderStateException("Cannot confirm an empty order");

        Raise(new OrderConfirmed
        {
            OrderId = Id,
            TotalAmount = _items.Sum(i => i.Quantity * i.UnitPrice)
        });
    }

    public void Ship(string trackingNumber, string carrier)
    {
        if (Status != OrderStatus.Confirmed)
            throw new InvalidOrderStateException("Can only ship confirmed orders");

        Raise(new OrderShipped
        {
            OrderId = Id,
            TrackingNumber = trackingNumber,
            Carrier = carrier
        });
    }

    public void Cancel(string reason)
    {
        if (Status is OrderStatus.Shipped or OrderStatus.Cancelled)
            throw new InvalidOrderStateException($"Cannot cancel order in {Status} status");

        Raise(new OrderCancelled { OrderId = Id, Reason = reason });
    }

    // Apply — pure state mutation, no side effects
    protected override void Apply(IDomainEvent @event)
    {
        switch (@event)
        {
            case OrderCreated e:
                Id = e.OrderId;
                CustomerId = e.CustomerId;
                Status = OrderStatus.Draft;
                ShippingAddress = new Address(e.ShippingStreet, e.ShippingCity, e.ShippingCountry);
                break;

            case OrderItemAdded e:
                _items.Add(new OrderItem(e.ItemId, e.ProductId, e.Sku, e.Quantity, e.UnitPrice));
                break;

            case OrderConfirmed e:
                Status = OrderStatus.Confirmed;
                TotalAmount = e.TotalAmount;
                break;

            case OrderShipped:
                Status = OrderStatus.Shipped;
                break;

            case OrderCancelled:
                Status = OrderStatus.Cancelled;
                break;
        }
    }
}

public enum OrderStatus { Draft, Confirmed, Shipped, Cancelled }
public record Address(string Street, string City, string Country);
public record OrderItem(Guid Id, Guid ProductId, string Sku, int Quantity, decimal UnitPrice);
```

---

## Step 1912: EventStoreDB Repository

```csharp
// Infrastructure/EventStore/EventStoreRepository.cs
using EventStore.Client;
using System.Text.Json;

namespace Orders.Infrastructure.EventStore;

public interface IEventSourcedRepository<T> where T : EventSourcedAggregate
{
    Task<T?> LoadAsync(Guid id, CancellationToken ct = default);
    Task SaveAsync(T aggregate, CancellationToken ct = default);
}

public sealed class EventStoreRepository<T> : IEventSourcedRepository<T>
    where T : EventSourcedAggregate, new()
{
    private readonly EventStoreClient _client;
    private readonly IEventTypeRegistry _registry;
    private static readonly JsonSerializerOptions JsonOpts = new()
    {
        PropertyNamingPolicy = JsonNamingPolicy.CamelCase
    };

    public EventStoreRepository(EventStoreClient client, IEventTypeRegistry registry)
    {
        _client = client;
        _registry = registry;
    }

    private static string StreamName(Guid id) =>
        $"{typeof(T).Name.ToLowerInvariant()}-{id}";

    public async Task<T?> LoadAsync(Guid id, CancellationToken ct = default)
    {
        var streamName = StreamName(id);

        try
        {
            var result = _client.ReadStreamAsync(
                Direction.Forwards,
                streamName,
                StreamPosition.Start,
                cancellationToken: ct);

            if (await result.ReadState == ReadState.StreamNotFound)
                return null;

            var events = new List<IDomainEvent>();

            await foreach (var resolvedEvent in result)
            {
                var eventType = _registry.GetType(resolvedEvent.Event.EventType);
                if (eventType is null) continue;

                var @event = (IDomainEvent)JsonSerializer.Deserialize(
                    resolvedEvent.Event.Data.Span, eventType, JsonOpts)!;

                events.Add(@event);
            }

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
        var streamName = StreamName(aggregate.Id);
        var uncommitted = aggregate.UncommittedEvents;

        if (!uncommitted.Any()) return;

        var eventData = uncommitted.Select(e =>
        {
            var data = JsonSerializer.SerializeToUtf8Bytes(e, e.GetType(), JsonOpts);
            var metadata = JsonSerializer.SerializeToUtf8Bytes(new
            {
                clrType = e.GetType().AssemblyQualifiedName,
                occurredAt = e.OccurredAt
            });

            return new EventData(
                Uuid.NewUuid(),
                e.EventType,
                data,
                metadata);
        }).ToList();

        var expectedRevision = aggregate.Version == 0
            ? StreamRevision.None
            : StreamRevision.FromInt64(aggregate.Version - uncommitted.Count);

        await _client.AppendToStreamAsync(
            streamName,
            expectedRevision,
            eventData,
            cancellationToken: ct);

        aggregate.MarkEventsAsCommitted();
    }
}
```

---

## Step 1913: Event Type Registry

```csharp
// Infrastructure/EventStore/EventTypeRegistry.cs
namespace Orders.Infrastructure.EventStore;

public interface IEventTypeRegistry
{
    Type? GetType(string eventTypeName);
    string GetName(Type eventType);
}

public class EventTypeRegistry : IEventTypeRegistry
{
    private readonly Dictionary<string, Type> _typesByName;
    private readonly Dictionary<Type, string> _namesByType;

    public EventTypeRegistry(IEnumerable<Assembly> assemblies)
    {
        var types = assemblies
            .SelectMany(a => a.GetTypes())
            .Where(t => t.IsAssignableTo(typeof(IDomainEvent)) && !t.IsAbstract)
            .ToList();

        _typesByName = types.ToDictionary(t => t.Name, t => t);
        _namesByType = types.ToDictionary(t => t, t => t.Name);
    }

    public Type? GetType(string eventTypeName) =>
        _typesByName.TryGetValue(eventTypeName, out var type) ? type : null;

    public string GetName(Type eventType) =>
        _namesByType.TryGetValue(eventType, out var name) ? name : eventType.Name;
}
```

---

## Step 1914: Snapshots for Long-Lived Aggregates

```csharp
// Infrastructure/EventStore/SnapshotStrategy.cs
namespace Orders.Infrastructure.EventStore;

public interface ISnapshotStrategy
{
    bool ShouldTakeSnapshot(EventSourcedAggregate aggregate);
}

public class EveryNEventsSnapshotStrategy : ISnapshotStrategy
{
    private readonly int _threshold;

    public EveryNEventsSnapshotStrategy(int threshold = 100)
    {
        _threshold = threshold;
    }

    public bool ShouldTakeSnapshot(EventSourcedAggregate aggregate) =>
        aggregate.Version > 0 && aggregate.Version % _threshold == 0;
}
```

```csharp
// Infrastructure/EventStore/SnapshotRepository.cs
public class SnapshotEventStoreRepository<T> : IEventSourcedRepository<T>
    where T : EventSourcedAggregate, new()
{
    private readonly EventStoreClient _client;
    private readonly ISnapshotStore _snapshots;
    private readonly ISnapshotStrategy _strategy;
    private readonly IEventTypeRegistry _registry;

    public SnapshotEventStoreRepository(
        EventStoreClient client,
        ISnapshotStore snapshots,
        ISnapshotStrategy strategy,
        IEventTypeRegistry registry)
    {
        _client = client;
        _snapshots = snapshots;
        _strategy = strategy;
        _registry = registry;
    }

    public async Task<T?> LoadAsync(Guid id, CancellationToken ct = default)
    {
        // Try to load from snapshot first
        var snapshot = await _snapshots.GetLatestSnapshotAsync<T>(id, ct);

        T aggregate;
        StreamPosition startPosition;

        if (snapshot is not null)
        {
            aggregate = snapshot.Aggregate;
            startPosition = StreamPosition.FromInt64(snapshot.Version + 1);
        }
        else
        {
            aggregate = new T();
            startPosition = StreamPosition.Start;
        }

        // Load events after snapshot
        var streamName = $"{typeof(T).Name.ToLowerInvariant()}-{id}";
        var result = _client.ReadStreamAsync(
            Direction.Forwards, streamName, startPosition, cancellationToken: ct);

        if (await result.ReadState == ReadState.StreamNotFound)
            return snapshot is not null ? aggregate : null;

        var events = new List<IDomainEvent>();
        await foreach (var resolvedEvent in result)
        {
            var eventType = _registry.GetType(resolvedEvent.Event.EventType);
            if (eventType is null) continue;
            events.Add((IDomainEvent)JsonSerializer.Deserialize(
                resolvedEvent.Event.Data.Span, eventType)!);
        }

        aggregate.LoadFromHistory(events);
        return aggregate;
    }

    public async Task SaveAsync(T aggregate, CancellationToken ct = default)
    {
        // Save events (same as base)
        // ... (omitted for brevity, same as EventStoreRepository.SaveAsync)

        // Check if we should snapshot
        if (_strategy.ShouldTakeSnapshot(aggregate))
        {
            await _snapshots.SaveSnapshotAsync(aggregate, aggregate.Version, ct);
        }
    }
}
```

---

## Step 1915: Read-Side Projections

Projections build query-optimized read models from events.

```csharp
// Application/Projections/OrderSummaryProjection.cs
using MassTransit;

namespace Orders.Application.Projections;

// Listens to integration events and builds the read model
public class OrderSummaryProjection :
    IConsumer<OrderCreated>,
    IConsumer<OrderConfirmed>,
    IConsumer<OrderShipped>,
    IConsumer<OrderCancelled>
{
    private readonly IOrderReadModelRepository _readModel;

    public OrderSummaryProjection(IOrderReadModelRepository readModel)
    {
        _readModel = readModel;
    }

    public async Task Consume(ConsumeContext<OrderCreated> context)
    {
        var evt = context.Message;
        var view = new OrderSummaryView
        {
            Id = evt.OrderId,
            CustomerId = evt.CustomerId,
            Status = "Draft",
            ItemCount = 0,
            TotalAmount = 0,
            CreatedAt = evt.OccurredAt,
            UpdatedAt = evt.OccurredAt
        };

        await _readModel.UpsertAsync(view, context.CancellationToken);
    }

    public async Task Consume(ConsumeContext<OrderConfirmed> context)
    {
        var evt = context.Message;
        await _readModel.UpdateAsync(evt.OrderId, view =>
        {
            view.Status = "Confirmed";
            view.TotalAmount = evt.TotalAmount;
            view.UpdatedAt = evt.OccurredAt;
        }, context.CancellationToken);
    }

    public async Task Consume(ConsumeContext<OrderShipped> context)
    {
        var evt = context.Message;
        await _readModel.UpdateAsync(evt.OrderId, view =>
        {
            view.Status = "Shipped";
            view.TrackingNumber = evt.TrackingNumber;
            view.UpdatedAt = evt.OccurredAt;
        }, context.CancellationToken);
    }

    public async Task Consume(ConsumeContext<OrderCancelled> context)
    {
        var evt = context.Message;
        await _readModel.UpdateAsync(evt.OrderId, view =>
        {
            view.Status = "Cancelled";
            view.CancellationReason = evt.Reason;
            view.UpdatedAt = evt.OccurredAt;
        }, context.CancellationToken);
    }
}
```

```csharp
// Read model — optimized for queries
public class OrderSummaryView
{
    public Guid Id { get; set; }
    public Guid CustomerId { get; set; }
    public string Status { get; set; } = null!;
    public int ItemCount { get; set; }
    public decimal TotalAmount { get; set; }
    public string? TrackingNumber { get; set; }
    public string? CancellationReason { get; set; }
    public DateTimeOffset CreatedAt { get; set; }
    public DateTimeOffset UpdatedAt { get; set; }
}
```

---

## Step 1916: EventStoreDB Persistent Subscriptions

```csharp
// Infrastructure/EventStore/EventStoreSubscriptionManager.cs
using EventStore.Client;

namespace Orders.Infrastructure.EventStore;

public class EventStoreSubscriptionManager : IHostedService
{
    private readonly EventStorePersistentSubscriptionsClient _subscriptions;
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<EventStoreSubscriptionManager> _logger;
    private readonly List<PersistentSubscription> _activeSubscriptions = [];

    public EventStoreSubscriptionManager(
        EventStorePersistentSubscriptionsClient subscriptions,
        IServiceScopeFactory scopeFactory,
        ILogger<EventStoreSubscriptionManager> logger)
    {
        _subscriptions = subscriptions;
        _scopeFactory = scopeFactory;
        _logger = logger;
    }

    public async Task StartAsync(CancellationToken ct)
    {
        // Subscribe to the $all stream for projection building
        await EnsureSubscriptionExistsAsync("$all", "order-projections", ct);

        var subscription = await _subscriptions.SubscribeToAllAsync(
            "order-projections",
            HandleEventAsync,
            HandleSubscriptionDropped,
            cancellationToken: ct);

        _activeSubscriptions.Add(subscription);
        _logger.LogInformation("Subscribed to $all with group 'order-projections'");
    }

    private async Task HandleEventAsync(
        PersistentSubscription subscription,
        ResolvedEvent resolvedEvent,
        int? retryCount,
        CancellationToken ct)
    {
        try
        {
            await using var scope = _scopeFactory.CreateAsyncScope();
            var dispatcher = scope.ServiceProvider.GetRequiredService<IEventDispatcher>();

            await dispatcher.DispatchAsync(resolvedEvent, ct);
            await subscription.Ack(resolvedEvent);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error handling event {EventType} {EventId}",
                resolvedEvent.Event.EventType, resolvedEvent.Event.EventId);

            // Nack — will be retried
            await subscription.Nack(
                PersistentSubscriptionNakEventAction.Retry,
                ex.Message,
                resolvedEvent);
        }
    }

    private void HandleSubscriptionDropped(
        PersistentSubscription subscription,
        SubscriptionDroppedReason reason,
        Exception? exception)
    {
        _logger.LogWarning(exception,
            "Subscription dropped. Reason: {Reason}", reason);

        // Reconnect logic would go here
    }

    private async Task EnsureSubscriptionExistsAsync(
        string stream, string group, CancellationToken ct)
    {
        try
        {
            await _subscriptions.CreateToAllAsync(
                group,
                new PersistentSubscriptionSettings(
                    startFrom: Position.Start,
                    resolveLinkTos: true,
                    maxRetryCount: 10,
                    messageTimeoutMs: 30_000,
                    checkPointLowerBound: 10,
                    checkPointUpperBound: 1000,
                    consumerStrategyName: SystemConsumerStrategies.RoundRobin),
                cancellationToken: ct);
        }
        catch (RpcException ex) when (ex.StatusCode == StatusCode.AlreadyExists)
        {
            // Subscription already exists — OK
        }
    }

    public async Task StopAsync(CancellationToken ct)
    {
        foreach (var sub in _activeSubscriptions)
            sub.Dispose();
    }
}
```

---

## Step 1917: CQRS — Command and Query Separation

```csharp
// CQRS: Commands mutate write side, Queries read from read models

// Commands — write side
public record PlaceOrderCommand : ICommand<PlaceOrderResult>
{
    public required Guid CustomerId { get; init; }
    public required Address ShippingAddress { get; init; }
    public required IReadOnlyList<OrderItemInput> Items { get; init; }
}

public record PlaceOrderResult(Guid OrderId, string Status);

public class PlaceOrderCommandHandler : ICommandHandler<PlaceOrderCommand, PlaceOrderResult>
{
    private readonly IEventSourcedRepository<Order> _orders;
    private readonly IUnitOfWork _uow;

    public async Task<PlaceOrderResult> Handle(PlaceOrderCommand cmd, CancellationToken ct)
    {
        var order = Order.Create(cmd.CustomerId, cmd.ShippingAddress);

        foreach (var item in cmd.Items)
            order.AddItem(item.ProductId, item.Sku, item.Quantity, item.UnitPrice);

        order.Confirm();

        await _orders.SaveAsync(order, ct);
        // Events published via EventStoreDB catch-up subscription

        return new PlaceOrderResult(order.Id, order.Status.ToString());
    }
}
```

```csharp
// Queries — read side (no aggregates, no event store)
public record GetOrderQuery(Guid OrderId) : IQuery<OrderDetailDto?>;

public record GetCustomerOrdersQuery(Guid CustomerId, int Page, int PageSize)
    : IQuery<PagedResult<OrderSummaryDto>>;

public class GetOrderQueryHandler : IQueryHandler<GetOrderQuery, OrderDetailDto?>
{
    private readonly IOrderReadModelRepository _readModels;

    public GetOrderQueryHandler(IOrderReadModelRepository readModels)
    {
        _readModels = readModels;
    }

    public async Task<OrderDetailDto?> Handle(GetOrderQuery query, CancellationToken ct)
    {
        var view = await _readModels.GetAsync(query.OrderId, ct);
        return view is null ? null : MapToDto(view);
    }

    private static OrderDetailDto MapToDto(OrderDetailView view) =>
        new(view.Id, view.Status, view.TotalAmount, view.Items.Select(MapItem).ToList(),
            view.CreatedAt, view.UpdatedAt);

    private static OrderItemDto MapItem(OrderItemView item) =>
        new(item.ProductId, item.Sku, item.Quantity, item.UnitPrice);
}
```

---

## Step 1918: Event Sourcing with EF Core (Alternative to EventStoreDB)

```csharp
// Infrastructure/EF/EventSourcingDbContext.cs
public class EventSourcingDbContext : DbContext
{
    public DbSet<StoredEvent> Events => Set<StoredEvent>();
    public DbSet<OrderSummaryView> OrderSummaries => Set<OrderSummaryView>();

    protected override void OnModelCreating(ModelBuilder model)
    {
        model.Entity<StoredEvent>(e =>
        {
            e.HasKey(x => x.Id);
            e.Property(x => x.StreamId).HasMaxLength(200).IsRequired();
            e.Property(x => x.EventType).HasMaxLength(200).IsRequired();
            e.Property(x => x.Data).HasColumnType("jsonb").IsRequired();
            e.HasIndex(x => new { x.StreamId, x.Version }).IsUnique();
            e.HasIndex(x => x.CreatedAt);
        });

        model.Entity<OrderSummaryView>(e =>
        {
            e.HasKey(x => x.Id);
            e.HasIndex(x => x.CustomerId);
            e.HasIndex(x => x.Status);
        });
    }
}

public class StoredEvent
{
    public Guid Id { get; set; }
    public string StreamId { get; set; } = null!;
    public long Version { get; set; }
    public string EventType { get; set; } = null!;
    public string Data { get; set; } = null!;
    public string? Metadata { get; set; }
    public DateTimeOffset CreatedAt { get; set; }
}
```

```csharp
// Infrastructure/EF/EfEventSourcedRepository.cs
public class EfEventSourcedRepository<T> : IEventSourcedRepository<T>
    where T : EventSourcedAggregate, new()
{
    private readonly EventSourcingDbContext _db;
    private readonly IEventTypeRegistry _registry;

    public EfEventSourcedRepository(
        EventSourcingDbContext db,
        IEventTypeRegistry registry)
    {
        _db = db;
        _registry = registry;
    }

    public async Task<T?> LoadAsync(Guid id, CancellationToken ct = default)
    {
        var streamId = $"{typeof(T).Name.ToLowerInvariant()}-{id}";

        var storedEvents = await _db.Events
            .Where(e => e.StreamId == streamId)
            .OrderBy(e => e.Version)
            .ToListAsync(ct);

        if (!storedEvents.Any()) return null;

        var aggregate = new T();
        var domainEvents = storedEvents
            .Select(e => Deserialize(e))
            .Where(e => e is not null)
            .Cast<IDomainEvent>();

        aggregate.LoadFromHistory(domainEvents);
        return aggregate;
    }

    public async Task SaveAsync(T aggregate, CancellationToken ct = default)
    {
        var streamId = $"{typeof(T).Name.ToLowerInvariant()}-{aggregate.Id}";
        var uncommitted = aggregate.UncommittedEvents;

        if (!uncommitted.Any()) return;

        var startVersion = aggregate.Version - uncommitted.Count + 1;

        for (var i = 0; i < uncommitted.Count; i++)
        {
            var @event = uncommitted[i];
            _db.Events.Add(new StoredEvent
            {
                Id = @event.EventId,
                StreamId = streamId,
                Version = startVersion + i,
                EventType = @event.EventType,
                Data = JsonSerializer.Serialize(@event, @event.GetType()),
                CreatedAt = @event.OccurredAt
            });
        }

        try
        {
            await _db.SaveChangesAsync(ct);
        }
        catch (DbUpdateException ex) when (IsOptimisticConcurrencyViolation(ex))
        {
            throw new ConcurrencyException(
                $"Concurrency conflict saving {typeof(T).Name} {aggregate.Id}", ex);
        }

        aggregate.MarkEventsAsCommitted();
    }

    private IDomainEvent? Deserialize(StoredEvent stored)
    {
        var type = _registry.GetType(stored.EventType);
        if (type is null) return null;
        return (IDomainEvent?)JsonSerializer.Deserialize(stored.Data, type);
    }

    private static bool IsOptimisticConcurrencyViolation(DbUpdateException ex) =>
        ex.InnerException?.Message.Contains("unique") == true ||
        ex.InnerException?.Message.Contains("duplicate") == true;
}
```

---

## Step 1919: Temporal Queries — Time Travel

```csharp
// Application/Queries/GetOrderAtPointInTimeQuery.cs
public record GetOrderAtPointInTimeQuery(
    Guid OrderId,
    DateTimeOffset AsOf) : IQuery<OrderDetailDto?>;

public class GetOrderAtPointInTimeHandler
    : IQueryHandler<GetOrderAtPointInTimeQuery, OrderDetailDto?>
{
    private readonly EventSourcingDbContext _db;
    private readonly IEventTypeRegistry _registry;

    public GetOrderAtPointInTimeHandler(
        EventSourcingDbContext db,
        IEventTypeRegistry registry)
    {
        _db = db;
        _registry = registry;
    }

    public async Task<OrderDetailDto?> Handle(
        GetOrderAtPointInTimeQuery query, CancellationToken ct)
    {
        var streamId = $"order-{query.OrderId}";

        var eventsUpToDate = await _db.Events
            .Where(e => e.StreamId == streamId && e.CreatedAt <= query.AsOf)
            .OrderBy(e => e.Version)
            .ToListAsync(ct);

        if (!eventsUpToDate.Any()) return null;

        var order = new Order();
        var domainEvents = eventsUpToDate
            .Select(e => Deserialize(e))
            .Where(e => e is not null)
            .Cast<IDomainEvent>();

        order.LoadFromHistory(domainEvents);

        return new OrderDetailDto(
            order.Id,
            order.Status.ToString(),
            order.TotalAmount,
            order.Items.Select(i => new OrderItemDto(
                i.ProductId, i.Sku, i.Quantity, i.UnitPrice)).ToList(),
            order.Items.Sum(i => i.Quantity * i.UnitPrice), // TotalAmount at that point
            query.AsOf);
    }

    private IDomainEvent? Deserialize(StoredEvent stored)
    {
        var type = _registry.GetType(stored.EventType);
        if (type is null) return null;
        return (IDomainEvent?)JsonSerializer.Deserialize(stored.Data, type);
    }
}
```

---

## Step 1920: Event Replay — Rebuilding Projections

```csharp
// Infrastructure/EventStore/ProjectionRebuilder.cs
public class ProjectionRebuilder
{
    private readonly EventSourcingDbContext _db;
    private readonly IEventTypeRegistry _registry;
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<ProjectionRebuilder> _logger;

    public ProjectionRebuilder(
        EventSourcingDbContext db,
        IEventTypeRegistry registry,
        IServiceScopeFactory scopeFactory,
        ILogger<ProjectionRebuilder> logger)
    {
        _db = db;
        _registry = registry;
        _scopeFactory = scopeFactory;
        _logger = logger;
    }

    public async Task RebuildAsync<TProjection>(CancellationToken ct = default)
        where TProjection : IProjection
    {
        _logger.LogInformation("Starting projection rebuild for {Projection}",
            typeof(TProjection).Name);

        await using var scope = _scopeFactory.CreateAsyncScope();
        var projection = scope.ServiceProvider.GetRequiredService<TProjection>();

        // Clear existing read model
        await projection.ResetAsync(ct);

        // Stream all events in batches
        const int batchSize = 1000;
        var offset = 0;
        var totalProcessed = 0;

        while (true)
        {
            var batch = await _db.Events
                .OrderBy(e => e.CreatedAt)
                .ThenBy(e => e.Version)
                .Skip(offset)
                .Take(batchSize)
                .ToListAsync(ct);

            if (!batch.Any()) break;

            foreach (var stored in batch)
            {
                var @event = Deserialize(stored);
                if (@event is not null)
                    await projection.HandleAsync(@event, ct);
            }

            offset += batch.Count;
            totalProcessed += batch.Count;
            _logger.LogInformation("Rebuilt {Count} events...", totalProcessed);
        }

        _logger.LogInformation("Projection rebuild complete. Total: {Total}", totalProcessed);
    }

    private IDomainEvent? Deserialize(StoredEvent stored)
    {
        var type = _registry.GetType(stored.EventType);
        if (type is null) return null;
        return (IDomainEvent?)JsonSerializer.Deserialize(stored.Data, type);
    }
}
```

```csharp
// CLI command to trigger rebuild
public class RebuildProjectionCommand : AsyncCommand<RebuildProjectionCommand.Settings>
{
    private readonly ProjectionRebuilder _rebuilder;

    public class Settings : CommandSettings
    {
        [CommandArgument(0, "<projection>")]
        public string ProjectionName { get; init; } = null!;
    }

    public override async Task<int> ExecuteAsync(
        CommandContext context, Settings settings)
    {
        await AnsiConsole.Status()
            .StartAsync($"Rebuilding {settings.ProjectionName}...", async ctx =>
            {
                ctx.Spinner(Spinner.Known.Dots);
                // Dynamic dispatch based on name
                await _rebuilder.RebuildAsync<OrderSummaryProjection>();
            });

        AnsiConsole.MarkupLine("[green]✓ Rebuild complete[/]");
        return 0;
    }
}
```

---

## Step 1921: Optimistic Concurrency Control

```csharp
// Application/Commands/ConfirmOrderCommandHandler.cs
public class ConfirmOrderCommandHandler : ICommandHandler<ConfirmOrderCommand, ConfirmOrderResult>
{
    private readonly IEventSourcedRepository<Order> _orders;
    private readonly ILogger<ConfirmOrderCommandHandler> _logger;

    public async Task<ConfirmOrderResult> Handle(
        ConfirmOrderCommand cmd, CancellationToken ct)
    {
        const int maxRetries = 3;
        var attempt = 0;

        while (attempt < maxRetries)
        {
            try
            {
                var order = await _orders.LoadAsync(cmd.OrderId, ct);
                if (order is null)
                    throw new OrderNotFoundException(cmd.OrderId);

                if (cmd.ExpectedVersion.HasValue && order.Version != cmd.ExpectedVersion)
                    throw new ConcurrencyException(
                        $"Expected version {cmd.ExpectedVersion} but was {order.Version}");

                order.Confirm();
                await _orders.SaveAsync(order, ct);

                return new ConfirmOrderResult(order.Id, order.Status.ToString(), order.Version);
            }
            catch (ConcurrencyException) when (attempt < maxRetries - 1)
            {
                attempt++;
                _logger.LogWarning("Concurrency conflict on order {OrderId}, attempt {Attempt}",
                    cmd.OrderId, attempt);
                await Task.Delay(50 * attempt, ct); // brief back-off
            }
        }

        throw new ConcurrencyException($"Failed to confirm order {cmd.OrderId} after {maxRetries} attempts");
    }
}
```

---

## Step 1922: Complete API Endpoint

```csharp
// Controllers/OrdersController.cs
[ApiController]
[ApiVersion("1.0")]
[Route("v{version:apiVersion}/orders")]
[Authorize]
public class OrdersController : ControllerBase
{
    private readonly IMediator _mediator;

    public OrdersController(IMediator mediator)
    {
        _mediator = mediator;
    }

    [HttpPost]
    [ProducesResponseType(typeof(PlaceOrderResult), 201)]
    [ProducesResponseType(400)]
    public async Task<ActionResult<PlaceOrderResult>> PlaceOrder(
        PlaceOrderRequest request, CancellationToken ct)
    {
        var cmd = new PlaceOrderCommand
        {
            CustomerId = User.GetUserId(),
            ShippingAddress = new Address(
                request.Street, request.City, request.Country),
            Items = request.Items.Select(i => new OrderItemInput(
                i.ProductId, i.Sku, i.Quantity, i.UnitPrice)).ToList()
        };

        var result = await _mediator.Send(cmd, ct);
        return CreatedAtAction(nameof(GetOrder), new { id = result.OrderId }, result);
    }

    [HttpGet("{id:guid}")]
    [ProducesResponseType(typeof(OrderDetailDto), 200)]
    [ProducesResponseType(404)]
    public async Task<ActionResult<OrderDetailDto>> GetOrder(Guid id, CancellationToken ct)
    {
        var result = await _mediator.Send(new GetOrderQuery(id), ct);
        return result is null ? NotFound() : Ok(result);
    }

    [HttpGet("{id:guid}/history")]
    [ProducesResponseType(typeof(IReadOnlyList<EventRecord>), 200)]
    public async Task<ActionResult<IReadOnlyList<EventRecord>>> GetHistory(
        Guid id, CancellationToken ct)
    {
        var events = await _mediator.Send(new GetOrderHistoryQuery(id), ct);
        return Ok(events);
    }

    [HttpGet("{id:guid}/at")]
    [ProducesResponseType(typeof(OrderDetailDto), 200)]
    public async Task<ActionResult<OrderDetailDto>> GetOrderAt(
        Guid id, [FromQuery] DateTimeOffset asOf, CancellationToken ct)
    {
        var result = await _mediator.Send(
            new GetOrderAtPointInTimeQuery(id, asOf), ct);
        return result is null ? NotFound() : Ok(result);
    }

    [HttpPost("{id:guid}/confirm")]
    [ProducesResponseType(typeof(ConfirmOrderResult), 200)]
    public async Task<ActionResult<ConfirmOrderResult>> ConfirmOrder(
        Guid id,
        [FromHeader(Name = "X-Expected-Version")] long? expectedVersion,
        CancellationToken ct)
    {
        var result = await _mediator.Send(
            new ConfirmOrderCommand(id, expectedVersion), ct);
        Response.Headers["X-Current-Version"] = result.Version.ToString();
        return Ok(result);
    }
}
```

---

## Step 1923: Event Sourcing Tests

```csharp
// Tests/Domain/OrderAggregateTests.cs
public class OrderAggregateTests
{
    [Fact]
    public void Create_RaisesOrderCreatedEvent()
    {
        var customerId = Guid.NewGuid();
        var address = new Address("123 Main St", "Springfield", "US");

        var order = Order.Create(customerId, address);

        order.UncommittedEvents.Should().HaveCount(1);
        var evt = order.UncommittedEvents[0].Should().BeOfType<OrderCreated>().Subject;
        evt.CustomerId.Should().Be(customerId);
        order.Status.Should().Be(OrderStatus.Draft);
    }

    [Fact]
    public void Confirm_WithItems_RaisesOrderConfirmedEvent()
    {
        var order = CreateOrderWithItems(2);

        order.Confirm();

        var uncommitted = order.UncommittedEvents;
        uncommitted.Should().Contain(e => e is OrderConfirmed);

        var confirmed = uncommitted.OfType<OrderConfirmed>().Single();
        confirmed.TotalAmount.Should().BePositive();
    }

    [Fact]
    public void Confirm_WithNoItems_Throws()
    {
        var order = Order.Create(Guid.NewGuid(), new Address("", "", ""));

        var act = () => order.Confirm();

        act.Should().Throw<InvalidOrderStateException>()
            .WithMessage("*empty*");
    }

    [Fact]
    public void LoadFromHistory_RestoresState()
    {
        var orderId = Guid.NewGuid();
        var customerId = Guid.NewGuid();

        var events = new IDomainEvent[]
        {
            new OrderCreated
            {
                OrderId = orderId, CustomerId = customerId,
                ShippingStreet = "123 Main", ShippingCity = "NYC", ShippingCountry = "US"
            },
            new OrderItemAdded
            {
                OrderId = orderId, ItemId = Guid.NewGuid(), ProductId = Guid.NewGuid(),
                Sku = "PROD-1", Quantity = 2, UnitPrice = 29.99m
            },
            new OrderConfirmed { OrderId = orderId, TotalAmount = 59.98m }
        };

        var order = new Order();
        order.LoadFromHistory(events);

        order.Id.Should().Be(orderId);
        order.Status.Should().Be(OrderStatus.Confirmed);
        order.Items.Should().HaveCount(1);
        order.TotalAmount.Should().Be(59.98m);
        order.Version.Should().Be(2); // 0-indexed
        order.UncommittedEvents.Should().BeEmpty(); // no new events
    }

    [Fact]
    public async Task Repository_SaveAndLoad_RoundTrips()
    {
        // Use InMemory EventStore or SQLite for tests
        var repository = CreateInMemoryRepository();

        var order = Order.Create(Guid.NewGuid(), new Address("St", "City", "US"));
        order.AddItem(Guid.NewGuid(), "SKU-1", 3, 10.00m);
        order.Confirm();

        await repository.SaveAsync(order);

        var loaded = await repository.LoadAsync(order.Id);

        loaded.Should().NotBeNull();
        loaded!.Status.Should().Be(OrderStatus.Confirmed);
        loaded.Items.Should().HaveCount(1);
    }

    private static Order CreateOrderWithItems(int count)
    {
        var order = Order.Create(Guid.NewGuid(), new Address("St", "City", "US"));
        for (var i = 0; i < count; i++)
            order.AddItem(Guid.NewGuid(), $"SKU-{i}", 1, 10.00m);
        return order;
    }
}
```

---

## Step 1924: Registration & Bootstrapping

```csharp
// Infrastructure/EventSourcedRegistration.cs
public static class EventSourcedRegistration
{
    public static IServiceCollection AddEventSourcing(
        this IServiceCollection services,
        IConfiguration config)
    {
        // EventStoreDB
        var connectionString = config.GetConnectionString("EventStore")!;
        services.AddSingleton(new EventStoreClient(
            EventStoreClientSettings.Create(connectionString)));
        services.AddSingleton(new EventStorePersistentSubscriptionsClient(
            EventStoreClientSettings.Create(connectionString)));

        // Or EF Core event store
        services.AddDbContext<EventSourcingDbContext>(opts =>
            opts.UseNpgsql(config.GetConnectionString("Postgres")));

        // Registry — discovers all event types
        services.AddSingleton<IEventTypeRegistry>(new EventTypeRegistry(
            new[] { typeof(OrderCreated).Assembly }));

        // Snapshot strategy
        services.AddSingleton<ISnapshotStrategy>(
            new EveryNEventsSnapshotStrategy(threshold: 100));

        // Repositories
        services.AddScoped<IEventSourcedRepository<Order>,
            EfEventSourcedRepository<Order>>();

        // Background: subscription manager
        services.AddHostedService<EventStoreSubscriptionManager>();

        // Projection rebuilder
        services.AddScoped<ProjectionRebuilder>();

        return services;
    }
}
```

---

## Summary

| Concept | Implementation | Notes |
|---------|----------------|-------|
| Event-sourced aggregate | `EventSourcedAggregate` + `Raise`/`Apply` | State from events only |
| Domain events | Immutable `record` types per change | Append-only log |
| Repository | `EventStoreRepository<T>` / `EfEventSourcedRepository<T>` | Serializes events |
| Snapshots | `ISnapshotStrategy` + periodic save | Every N events |
| Projections | Event consumers → read models | Eventual consistency |
| Subscriptions | `EventStorePersistentSubscriptionsClient` | Durable catch-up |
| CQRS split | Commands → event store, Queries → read models | No query on aggregates |
| Time travel | Load events `WHERE created_at <= asOf` | Full audit capability |
| Optimistic concurrency | Expected version check on append | Conflict detection |
| Rebuild | Replay all events → fresh read model | Bug fix or schema change |

**Next**: Part 81 — Clean Architecture & Vertical Slice Architecture
