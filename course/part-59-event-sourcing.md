# Part 59: Event Sourcing Deep Dive

## Steps 1583-1605: Marten & EventStoreDB in Production

---

## Step 1583: Event Sourcing Architecture

```
Event Sourcing vs Traditional State
══════════════════════════════════════════════════════════════

  Traditional (State)         Event Sourcing
  ───────────────────         ──────────────
  orders table:               order_events stream:
  ┌────────────────┐          ┌──────────────────────────────────┐
  │ id │ status    │          │ OrderPlaced       (t=0)   →      │
  │ 1  │ Shipped   │  ←replay─│ PaymentConfirmed  (t=1)   →      │
  └────────────────┘          │ OrderShipped      (t=2)   → NOW  │
                              └──────────────────────────────────┘
  
  Benefits:
  ┌────────────────────────────────────────────────────────────┐
  │ ✅ Full audit trail (who did what, when, why)              │
  │ ✅ Time travel (replay to any point in time)               │
  │ ✅ Event-driven naturally (publish what happened)          │
  │ ✅ Multiple read models from same events                   │
  │ ✅ No UPDATE/DELETE — only INSERTs                         │
  │ ✅ Perfect for CQRS + DDD aggregates                       │
  └────────────────────────────────────────────────────────────┘
  
  CQRS + Event Sourcing:
  
  Command → Aggregate → Events → Event Store
                                      │
                  ┌───────────────────┤
                  │                   │
              Projector           Projector
                  │                   │
           ReadModel-1          ReadModel-2
         (order details)       (analytics)
```

### NuGet Packages

```xml
<!-- Marten (PostgreSQL event store) -->
<PackageReference Include="Marten"                         Version="7.*" />
<PackageReference Include="Marten.AspNetCore"              Version="7.*" />

<!-- EventStoreDB -->
<PackageReference Include="EventStore.Client.Grpc.Streams"  Version="23.*" />
<PackageReference Include="EventStore.Client.Grpc.PersistentSubscriptions" Version="23.*" />

<!-- Optional: Wolverine for message handling -->
<PackageReference Include="Wolverine"                       Version="3.*" />
<PackageReference Include="WolverineFx.Marten"             Version="3.*" />
```

---

## Step 1584: Domain Events Design

```csharp
// Domain/Events.cs — Immutable event records
namespace EventSourcingDemo.Domain;

// Base event interface
public interface IDomainEvent
{
    Guid   EventId   { get; }
    Guid   StreamId  { get; }  // aggregate id
    string EventType { get; }
    DateTimeOffset OccurredAt { get; }
}

// Base record
public abstract record DomainEvent : IDomainEvent
{
    public Guid           EventId    { get; init; } = Guid.NewGuid();
    public DateTimeOffset OccurredAt { get; init; } = DateTimeOffset.UtcNow;
    public abstract string EventType { get; }
    public abstract Guid   StreamId  { get; }
}

// ── Order Events ──────────────────────────────────────────────────────────

public record OrderPlaced(
    Guid   OrderId,
    string CustomerId,
    IReadOnlyList<OrderLineItem> Items,
    Money Total,
    string ShippingAddress) : DomainEvent
{
    public override string EventType => "order.placed";
    public override Guid   StreamId  => OrderId;
}

public record PaymentAuthorized(
    Guid   OrderId,
    string PaymentReference,
    string PaymentMethod,
    Money  Amount) : DomainEvent
{
    public override string EventType => "order.payment_authorized";
    public override Guid   StreamId  => OrderId;
}

public record OrderConfirmed(
    Guid   OrderId,
    string ConfirmedBy,
    string WarehouseId) : DomainEvent
{
    public override string EventType => "order.confirmed";
    public override Guid   StreamId  => OrderId;
}

public record OrderShipped(
    Guid   OrderId,
    string TrackingNumber,
    string Carrier,
    DateTimeOffset EstimatedDelivery) : DomainEvent
{
    public override string EventType => "order.shipped";
    public override Guid   StreamId  => OrderId;
}

public record OrderDelivered(
    Guid   OrderId,
    string DeliveredTo,
    DateTimeOffset DeliveredAt) : DomainEvent
{
    public override string EventType => "order.delivered";
    public override Guid   StreamId  => OrderId;
}

public record OrderCancelled(
    Guid   OrderId,
    string Reason,
    string CancelledBy,
    bool   RefundIssued) : DomainEvent
{
    public override string EventType => "order.cancelled";
    public override Guid   StreamId  => OrderId;
}

public record OrderNoteAdded(
    Guid   OrderId,
    string Note,
    string AddedBy) : DomainEvent
{
    public override string EventType => "order.note_added";
    public override Guid   StreamId  => OrderId;
}

// Value objects
public record OrderLineItem(string ProductId, string Name, int Quantity, Money UnitPrice)
{
    public Money Total => UnitPrice with { Amount = UnitPrice.Amount * Quantity };
}

public record Money(decimal Amount, string Currency = "USD")
{
    public static Money operator +(Money a, Money b)
    {
        if (a.Currency != b.Currency) throw new InvalidOperationException("Currency mismatch");
        return a with { Amount = a.Amount + b.Amount };
    }
}
```

---

## Step 1585: Order Aggregate — Event Sourced

```csharp
// Domain/OrderAggregate.cs
namespace EventSourcingDemo.Domain;

public enum OrderStatus
{
    Pending, PaymentAuthorized, Confirmed, Shipped, Delivered, Cancelled
}

public class Order
{
    // ── State ──────────────────────────────────────────────────────────────
    public Guid        Id             { get; private set; }
    public string      CustomerId     { get; private set; } = string.Empty;
    public OrderStatus Status         { get; private set; }
    public Money       Total          { get; private set; } = new(0);
    public string?     TrackingNumber { get; private set; }
    public string?     PaymentRef     { get; private set; }
    public List<string> Notes         { get; private set; } = [];

    private readonly List<IDomainEvent> _uncommittedEvents = [];
    public IReadOnlyList<IDomainEvent> UncommittedEvents => _uncommittedEvents;

    // ── Marten needs parameterless constructor ─────────────────────────────
    private Order() { }

    // ── Factory method ─────────────────────────────────────────────────────
    public static Order Place(
        Guid orderId,
        string customerId,
        IReadOnlyList<OrderLineItem> items,
        string shippingAddress)
    {
        if (!items.Any())
            throw new DomainException("Cannot place order with no items");

        var total = items.Aggregate(new Money(0), (sum, i) => sum + i.Total);

        var order = new Order();
        order.Apply(new OrderPlaced(orderId, customerId, items, total, shippingAddress));
        return order;
    }

    // ── Commands ───────────────────────────────────────────────────────────
    public void AuthorizePayment(string paymentRef, string method)
    {
        GuardStatus(OrderStatus.Pending);
        Apply(new PaymentAuthorized(Id, paymentRef, method, Total));
    }

    public void Confirm(string confirmedBy, string warehouseId)
    {
        GuardStatus(OrderStatus.PaymentAuthorized);
        Apply(new OrderConfirmed(Id, confirmedBy, warehouseId));
    }

    public void Ship(string trackingNumber, string carrier)
    {
        GuardStatus(OrderStatus.Confirmed);
        Apply(new OrderShipped(Id, trackingNumber, carrier,
            DateTimeOffset.UtcNow.AddDays(3)));
    }

    public void MarkDelivered(string deliveredTo)
    {
        GuardStatus(OrderStatus.Shipped);
        Apply(new OrderDelivered(Id, deliveredTo, DateTimeOffset.UtcNow));
    }

    public void Cancel(string reason, string cancelledBy)
    {
        if (Status is OrderStatus.Shipped or OrderStatus.Delivered)
            throw new DomainException("Cannot cancel order that has already shipped");

        if (Status == OrderStatus.Cancelled)
            throw new DomainException("Order already cancelled");

        Apply(new OrderCancelled(Id, reason, cancelledBy, Status >= OrderStatus.PaymentAuthorized));
    }

    public void AddNote(string note, string addedBy)
    {
        if (string.IsNullOrWhiteSpace(note))
            throw new DomainException("Note cannot be empty");

        Apply(new OrderNoteAdded(Id, note, addedBy));
    }

    // ── Event Application ─────────────────────────────────────────────────
    // Called both when raising new events AND when replaying from store

    private void Apply(IDomainEvent @event)
    {
        When(@event);
        _uncommittedEvents.Add(@event);
    }

    // Replay method — no side effects, no validation (history already happened)
    public void Rehydrate(IEnumerable<IDomainEvent> events)
    {
        foreach (var @event in events)
            When(@event);
    }

    private void When(IDomainEvent @event)
    {
        switch (@event)
        {
            case OrderPlaced e:
                Id         = e.OrderId;
                CustomerId = e.CustomerId;
                Total      = e.Total;
                Status     = OrderStatus.Pending;
                break;

            case PaymentAuthorized e:
                PaymentRef = e.PaymentReference;
                Status     = OrderStatus.PaymentAuthorized;
                break;

            case OrderConfirmed:
                Status = OrderStatus.Confirmed;
                break;

            case OrderShipped e:
                TrackingNumber = e.TrackingNumber;
                Status         = OrderStatus.Shipped;
                break;

            case OrderDelivered:
                Status = OrderStatus.Delivered;
                break;

            case OrderCancelled:
                Status = OrderStatus.Cancelled;
                break;

            case OrderNoteAdded e:
                Notes.Add(e.Note);
                break;
        }
    }

    private void GuardStatus(OrderStatus expected)
    {
        if (Status != expected)
            throw new DomainException(
                $"Cannot perform operation in status '{Status}'. Expected '{expected}'.");
    }
}
```

---

## Step 1586: Marten Setup — PostgreSQL Event Store

```csharp
// Program.cs — Marten configuration
using Marten;
using Marten.Events.Projections;
using Weasel.Core;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddMarten(options =>
{
    options.Connection(builder.Configuration.GetConnectionString("Postgres")!);

    // Schema
    options.DatabaseSchemaName = "orders";

    // Auto-create in development
    if (builder.Environment.IsDevelopment())
        options.AutoCreateSchemaObjects = AutoCreate.All;

    // ── Event Store ───────────────────────────────────────────────────────
    options.Events.StreamIdentity = StreamIdentity.AsGuid;

    // Register all event types
    options.Events.AddEventType<OrderPlaced>();
    options.Events.AddEventType<PaymentAuthorized>();
    options.Events.AddEventType<OrderConfirmed>();
    options.Events.AddEventType<OrderShipped>();
    options.Events.AddEventType<OrderDelivered>();
    options.Events.AddEventType<OrderCancelled>();
    options.Events.AddEventType<OrderNoteAdded>();

    // ── Projections ───────────────────────────────────────────────────────
    options.Projections.Add<OrderDetailsProjection>(ProjectionLifecycle.Inline);
    options.Projections.Add<OrderSummaryProjection>(ProjectionLifecycle.Async);
    options.Projections.Add<CustomerOrderHistoryProjection>(ProjectionLifecycle.Async);

    // Snapshots every 50 events
    options.Projections.Snapshot<Order>(SnapshotLifecycle.Inline);

    // ── JSON Serialization ────────────────────────────────────────────────
    options.UseSystemTextJsonForSerialization(opts =>
    {
        opts.PropertyNamingPolicy = System.Text.Json.JsonNamingPolicy.SnakeCaseLower;
    });
})
.UseLightweightSessions()
.AddAsyncDaemon(DaemonMode.HotCold);  // async projections daemon

builder.Services.AddScoped<IOrderRepository, MartenOrderRepository>();
```

---

## Step 1587: Repository — Append & Load

```csharp
// Infrastructure/MartenOrderRepository.cs
using Marten;

namespace EventSourcingDemo.Infrastructure;

public interface IOrderRepository
{
    Task<Order?> LoadAsync(Guid id, CancellationToken ct = default);
    Task SaveAsync(Order order, CancellationToken ct = default);
    Task<Order> LoadRequiredAsync(Guid id, CancellationToken ct = default);
}

public class MartenOrderRepository : IOrderRepository
{
    private readonly IDocumentSession _session;

    public MartenOrderRepository(IDocumentSession session)
        => _session = session;

    public async Task<Order?> LoadAsync(Guid id, CancellationToken ct = default)
    {
        // Marten reconstructs aggregate by replaying events
        var aggregate = await _session.Events.AggregateStreamAsync<Order>(id, token: ct);
        return aggregate;
    }

    public async Task<Order> LoadRequiredAsync(Guid id, CancellationToken ct = default)
    {
        var order = await LoadAsync(id, ct);
        if (order is null)
            throw new NotFoundException($"Order {id} not found");
        return order;
    }

    public async Task SaveAsync(Order order, CancellationToken ct = default)
    {
        var uncommitted = order.UncommittedEvents.ToList();
        if (!uncommitted.Any()) return;

        // Append with optimistic concurrency (pass expected version)
        _session.Events.Append(order.Id, uncommitted.Cast<object>().ToArray());
        await _session.SaveChangesAsync(ct);
    }
}

// Optimistic concurrency version
public class VersionedMartenOrderRepository : IOrderRepository
{
    private readonly IDocumentSession _session;
    private readonly Dictionary<Guid, long> _versions = new();

    public VersionedMartenOrderRepository(IDocumentSession session)
        => _session = session;

    public async Task<Order?> LoadAsync(Guid id, CancellationToken ct = default)
    {
        var stream = await _session.Events.FetchStreamStateAsync(id, ct);
        if (stream is null) return null;

        _versions[id] = stream.Version;

        var events = await _session.Events.FetchStreamAsync(id, token: ct);
        var order  = new Order();
        order.Rehydrate(events.Select(e => (IDomainEvent)e.Data!));
        return order;
    }

    public async Task<Order> LoadRequiredAsync(Guid id, CancellationToken ct = default)
        => await LoadAsync(id, ct) ?? throw new NotFoundException($"Order {id} not found");

    public async Task SaveAsync(Order order, CancellationToken ct = default)
    {
        var uncommitted = order.UncommittedEvents.ToList();
        if (!uncommitted.Any()) return;

        // Optimistic concurrency: fail if another writer changed the stream
        if (_versions.TryGetValue(order.Id, out var expectedVersion))
        {
            _session.Events.Append(order.Id, expectedVersion, uncommitted.Cast<object>().ToArray());
        }
        else
        {
            _session.Events.StartStream<Order>(order.Id, uncommitted.Cast<object>().ToArray());
        }

        try
        {
            await _session.SaveChangesAsync(ct);
        }
        catch (Marten.Exceptions.EventStreamUnexpectedMaxEventIdException)
        {
            throw new ConcurrencyException($"Order {order.Id} was modified concurrently");
        }
    }
}
```

---

## Step 1588: Projections — Read Models

```csharp
// Projections/OrderDetailsProjection.cs
using Marten.Events.Projections;

namespace EventSourcingDemo.Projections;

// Inline projection — updated synchronously when events are appended
public class OrderDetailsProjection : SingleStreamProjection<OrderDetails>
{
    public OrderDetailsProjection()
    {
        // Tell Marten which events this projection handles
        ProjectEvent<OrderPlaced>(Apply);
        ProjectEvent<PaymentAuthorized>(Apply);
        ProjectEvent<OrderConfirmed>(Apply);
        ProjectEvent<OrderShipped>(Apply);
        ProjectEvent<OrderDelivered>(Apply);
        ProjectEvent<OrderCancelled>(Apply);
        ProjectEvent<OrderNoteAdded>(Apply);
    }

    private static OrderDetails Apply(OrderDetails view, OrderPlaced e) => view with
    {
        Id             = e.OrderId,
        CustomerId     = e.CustomerId,
        Status         = "Pending",
        Total          = e.Total.Amount,
        Currency       = e.Total.Currency,
        Items          = e.Items.Select(i => new OrderItemView(i.ProductId, i.Name, i.Quantity, i.UnitPrice.Amount)).ToList(),
        ShippingAddress = e.ShippingAddress,
        PlacedAt       = e.OccurredAt
    };

    private static OrderDetails Apply(OrderDetails view, PaymentAuthorized e) => view with
    {
        Status     = "PaymentAuthorized",
        PaymentRef = e.PaymentReference
    };

    private static OrderDetails Apply(OrderDetails view, OrderConfirmed e) => view with
    {
        Status      = "Confirmed",
        WarehouseId = e.WarehouseId
    };

    private static OrderDetails Apply(OrderDetails view, OrderShipped e) => view with
    {
        Status            = "Shipped",
        TrackingNumber    = e.TrackingNumber,
        Carrier           = e.Carrier,
        EstimatedDelivery = e.EstimatedDelivery,
        ShippedAt         = e.OccurredAt
    };

    private static OrderDetails Apply(OrderDetails view, OrderDelivered e) => view with
    {
        Status      = "Delivered",
        DeliveredAt = e.DeliveredAt
    };

    private static OrderDetails Apply(OrderDetails view, OrderCancelled e) => view with
    {
        Status          = "Cancelled",
        CancellationReason = e.Reason,
        CancelledAt     = e.OccurredAt
    };

    private static OrderDetails Apply(OrderDetails view, OrderNoteAdded e) => view with
    {
        Notes = [..view.Notes, new NoteView(e.Note, e.AddedBy, e.OccurredAt)]
    };
}

// Read model
public record OrderDetails
{
    public Guid    Id             { get; init; }
    public string  CustomerId     { get; init; } = string.Empty;
    public string  Status         { get; init; } = "Pending";
    public decimal Total          { get; init; }
    public string  Currency       { get; init; } = "USD";
    public string  ShippingAddress { get; init; } = string.Empty;
    public string? PaymentRef     { get; init; }
    public string? WarehouseId    { get; init; }
    public string? TrackingNumber { get; init; }
    public string? Carrier        { get; init; }
    public DateTimeOffset? EstimatedDelivery { get; init; }
    public DateTimeOffset  PlacedAt          { get; init; }
    public DateTimeOffset? ShippedAt         { get; init; }
    public DateTimeOffset? DeliveredAt       { get; init; }
    public DateTimeOffset? CancelledAt       { get; init; }
    public string? CancellationReason        { get; init; }
    public List<OrderItemView> Items         { get; init; } = [];
    public List<NoteView>      Notes         { get; init; } = [];
}

public record OrderItemView(string ProductId, string Name, int Quantity, decimal UnitPrice);
public record NoteView(string Text, string AddedBy, DateTimeOffset AddedAt);
```

```csharp
// Projections/CustomerOrderHistoryProjection.cs
// Multi-stream projection — aggregates events across many streams
public class CustomerOrderHistoryProjection : MultiStreamProjection<CustomerOrderHistory, string>
{
    public CustomerOrderHistoryProjection()
    {
        Identity<OrderPlaced>(e => e.CustomerId);
        Identity<OrderCancelled>(e => e.OrderId.ToString()); // need to map
        Identity<OrderDelivered>(e => e.OrderId.ToString());

        ProjectEvent<OrderPlaced>((view, e) => view with
        {
            CustomerId  = e.CustomerId,
            TotalOrders = view.TotalOrders + 1,
            TotalSpend  = view.TotalSpend + e.Total.Amount,
            RecentOrders = [
                new RecentOrder(e.OrderId, e.OccurredAt, e.Total.Amount, "Pending"),
                ..view.RecentOrders.Take(9)
            ]
        });
    }
}

public record CustomerOrderHistory
{
    public string  CustomerId   { get; init; } = string.Empty;
    public int     TotalOrders  { get; init; }
    public decimal TotalSpend   { get; init; }
    public List<RecentOrder> RecentOrders { get; init; } = [];
}

public record RecentOrder(Guid OrderId, DateTimeOffset PlacedAt, decimal Total, string Status);
```

```csharp
// Projections/OrderSummaryProjection.cs — async dashboard aggregation
public class OrderSummaryProjection : EventProjection
{
    public async Task Project(OrderPlaced e, IDocumentOperations ops)
    {
        var summary = await ops.LoadAsync<DailySummary>(e.OccurredAt.Date)
                   ?? new DailySummary { Date = e.OccurredAt.Date };

        ops.Store(summary with
        {
            TotalOrders  = summary.TotalOrders + 1,
            TotalRevenue = summary.TotalRevenue + e.Total.Amount
        });
    }

    public async Task Project(OrderCancelled e, IDocumentOperations ops)
    {
        // Decrement from summary
        var summary = await ops.LoadAsync<DailySummary>(DateOnly.FromDateTime(DateTime.UtcNow));
        if (summary is null) return;

        ops.Store(summary with { CancelledOrders = summary.CancelledOrders + 1 });
    }
}

public record DailySummary
{
    public DateOnly Date            { get; init; }
    public int      TotalOrders     { get; init; }
    public int      CancelledOrders { get; init; }
    public decimal  TotalRevenue    { get; init; }
}
```

---

## Step 1589: Application Service — Command Handlers

```csharp
// Application/OrderCommandHandler.cs
namespace EventSourcingDemo.Application;

public class OrderCommandHandler
{
    private readonly IOrderRepository _repo;
    private readonly IEventPublisher _publisher;
    private readonly ILogger<OrderCommandHandler> _log;

    public OrderCommandHandler(
        IOrderRepository repo,
        IEventPublisher publisher,
        ILogger<OrderCommandHandler> log)
    {
        _repo      = repo;
        _publisher = publisher;
        _log       = log;
    }

    public async Task<Guid> HandleAsync(PlaceOrderCommand cmd, CancellationToken ct = default)
    {
        var orderId = Guid.NewGuid();
        var order   = Order.Place(orderId, cmd.CustomerId, cmd.Items, cmd.ShippingAddress);

        await _repo.SaveAsync(order, ct);

        // Publish events to message bus for downstream services
        foreach (var @event in order.UncommittedEvents)
            await _publisher.PublishAsync(@event, ct);

        _log.LogInformation("Order {OrderId} placed for customer {CustomerId}",
            orderId, cmd.CustomerId);

        return orderId;
    }

    public async Task HandleAsync(AuthorizePaymentCommand cmd, CancellationToken ct = default)
    {
        var order = await _repo.LoadRequiredAsync(cmd.OrderId, ct);
        order.AuthorizePayment(cmd.PaymentReference, cmd.Method);

        await _repo.SaveAsync(order, ct);

        foreach (var @event in order.UncommittedEvents)
            await _publisher.PublishAsync(@event, ct);
    }

    public async Task HandleAsync(CancelOrderCommand cmd, CancellationToken ct = default)
    {
        var order = await _repo.LoadRequiredAsync(cmd.OrderId, ct);
        order.Cancel(cmd.Reason, cmd.RequestedBy);

        await _repo.SaveAsync(order, ct);

        foreach (var @event in order.UncommittedEvents)
            await _publisher.PublishAsync(@event, ct);
    }
}

// Commands
public record PlaceOrderCommand(
    string CustomerId,
    IReadOnlyList<OrderLineItem> Items,
    string ShippingAddress);

public record AuthorizePaymentCommand(
    Guid OrderId,
    string PaymentReference,
    string Method);

public record CancelOrderCommand(
    Guid OrderId,
    string Reason,
    string RequestedBy);
```

---

## Step 1590: Query Side — Reading Projections

```csharp
// Application/OrderQueryService.cs
using Marten;

public class OrderQueryService
{
    private readonly IQuerySession _session;

    public OrderQueryService(IQuerySession session) => _session = session;

    // Load from inline projection (fast, always up-to-date)
    public Task<OrderDetails?> GetOrderAsync(Guid id, CancellationToken ct = default)
        => _session.LoadAsync<OrderDetails>(id, ct);

    // LINQ query on projection
    public async Task<IReadOnlyList<OrderDetails>> GetCustomerOrdersAsync(
        string customerId,
        OrderStatus? status = null,
        int take = 20,
        CancellationToken ct = default)
    {
        var query = _session.Query<OrderDetails>()
            .Where(o => o.CustomerId == customerId);

        if (status.HasValue)
            query = query.Where(o => o.Status == status.Value.ToString());

        return await query
            .OrderByDescending(o => o.PlacedAt)
            .Take(take)
            .ToListAsync(ct);
    }

    // Full-text search
    public async Task<IReadOnlyList<OrderDetails>> SearchOrdersAsync(
        string term,
        CancellationToken ct = default)
    {
        return await _session.Query<OrderDetails>()
            .Where(o => o.CustomerId.Contains(term) ||
                        o.TrackingNumber!.Contains(term))
            .ToListAsync(ct);
    }

    // Read from async projection (may be slightly behind)
    public Task<CustomerOrderHistory?> GetCustomerHistoryAsync(
        string customerId,
        CancellationToken ct = default)
        => _session.LoadAsync<CustomerOrderHistory>(customerId, ct);

    // Time travel — load aggregate at specific point in time
    public async Task<OrderDetails?> GetOrderAtAsync(
        Guid orderId,
        DateTimeOffset asOf,
        CancellationToken ct = default)
    {
        var events = await _session.Events.FetchStreamAsync(orderId, timestamp: asOf.DateTime, token: ct);

        if (!events.Any()) return null;

        var order = new Order();
        order.Rehydrate(events.Select(e => (IDomainEvent)e.Data!));

        // Map to read model manually for time travel
        return new OrderDetails
        {
            Id         = order.Id,
            CustomerId = order.CustomerId,
            Status     = order.Status.ToString(),
            Total      = order.Total.Amount
        };
    }

    // Raw event stream for audit
    public async Task<IReadOnlyList<EventAuditEntry>> GetAuditTrailAsync(
        Guid orderId,
        CancellationToken ct = default)
    {
        var stream = await _session.Events.FetchStreamAsync(orderId, token: ct);

        return stream.Select(e => new EventAuditEntry(
            EventType: e.EventTypeName,
            OccurredAt: e.Timestamp,
            Version: e.Version,
            Data: System.Text.Json.JsonSerializer.Serialize(e.Data)
        )).ToList();
    }
}

public record EventAuditEntry(
    string EventType,
    DateTimeOffset OccurredAt,
    long Version,
    string Data);
```

---

## Step 1591: EventStoreDB Alternative

```csharp
// Infrastructure/EventStoreOrderRepository.cs
using EventStore.Client;
using System.Text;
using System.Text.Json;

public class EventStoreOrderRepository : IOrderRepository
{
    private readonly EventStoreClient _client;
    private const string StreamPrefix = "order-";

    public EventStoreOrderRepository(EventStoreClient client)
        => _client = client;

    public async Task<Order?> LoadAsync(Guid id, CancellationToken ct = default)
    {
        var streamName = $"{StreamPrefix}{id}";

        try
        {
            var events  = new List<IDomainEvent>();
            var result  = _client.ReadStreamAsync(
                Direction.Forwards,
                streamName,
                StreamPosition.Start,
                cancellationToken: ct);

            await foreach (var resolvedEvent in result)
            {
                var domainEvent = Deserialize(resolvedEvent);
                if (domainEvent is not null)
                    events.Add(domainEvent);
            }

            if (!events.Any()) return null;

            var order = new Order();
            order.Rehydrate(events);
            return order;
        }
        catch (StreamNotFoundException)
        {
            return null;
        }
    }

    public async Task<Order> LoadRequiredAsync(Guid id, CancellationToken ct = default)
        => await LoadAsync(id, ct) ?? throw new NotFoundException($"Order {id} not found");

    public async Task SaveAsync(Order order, CancellationToken ct = default)
    {
        var uncommitted = order.UncommittedEvents.ToList();
        if (!uncommitted.Any()) return;

        var streamName    = $"{StreamPrefix}{order.Id}";
        var eventData     = uncommitted.Select(Serialize).ToList();

        // Optimistic concurrency via expected stream state
        var expectedState = uncommitted.Count == order.UncommittedEvents.Count
            ? StreamState.NoStream  // first time appending
            : StreamState.Any;

        await _client.AppendToStreamAsync(streamName, expectedState, eventData, cancellationToken: ct);
    }

    private static EventData Serialize(IDomainEvent @event)
    {
        var json = JsonSerializer.SerializeToUtf8Bytes(@event, @event.GetType());
        return new EventData(
            Uuid.NewUuid(),
            @event.EventType,
            json,
            Encoding.UTF8.GetBytes($"{{\"clrType\":\"{@event.GetType().AssemblyQualifiedName}\"}}"));
    }

    private static IDomainEvent? Deserialize(ResolvedEvent resolvedEvent)
    {
        var meta    = JsonSerializer.Deserialize<Dictionary<string, string>>(resolvedEvent.Event.Metadata.Span);
        if (meta?.TryGetValue("clrType", out var clrType) != true) return null;

        var type = Type.GetType(clrType);
        if (type is null) return null;

        return JsonSerializer.Deserialize(resolvedEvent.Event.Data.Span, type) as IDomainEvent;
    }
}

// EventStoreDB client registration
builder.Services.AddSingleton(_ =>
    new EventStoreClient(EventStoreClientSettings.Create(
        builder.Configuration.GetConnectionString("EventStoreDB")!)));
```

---

## Step 1592: EventStoreDB Persistent Subscriptions

```csharp
// Projections/EventStoreProjectionHost.cs
using EventStore.Client;

public class EventStoreProjectionHost : BackgroundService
{
    private readonly EventStorePersistentSubscriptionsClient _subs;
    private readonly IServiceProvider _services;
    private readonly ILogger<EventStoreProjectionHost> _log;

    public EventStoreProjectionHost(
        EventStorePersistentSubscriptionsClient subs,
        IServiceProvider services,
        ILogger<EventStoreProjectionHost> log)
    {
        _subs     = subs;
        _services = services;
        _log      = log;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        // Ensure subscription group exists
        try
        {
            await _subs.CreateToAllAsync(
                "order-projections",
                new PersistentSubscriptionSettings(
                    startFrom: Position.Start,
                    resolveLinkTos: true,
                    maxRetryCount: 5),
                cancellationToken: stoppingToken);
        }
        catch (RpcException ex) when (ex.StatusCode == Grpc.Core.StatusCode.AlreadyExists)
        {
            // Already exists — OK
        }

        // Subscribe
        await _subs.SubscribeToAllAsync(
            "order-projections",
            HandleEventAsync,
            (sub, reason, ex) =>
            {
                _log.LogWarning(ex, "Subscription dropped: {Reason}", reason);
            },
            cancellationToken: stoppingToken);
    }

    private async Task HandleEventAsync(
        PersistentSubscription subscription,
        ResolvedEvent resolvedEvent,
        int? retryCount,
        CancellationToken ct)
    {
        try
        {
            using var scope = _services.CreateScope();
            var handler = scope.ServiceProvider.GetRequiredService<IEventHandler>();

            await handler.HandleAsync(resolvedEvent, ct);
            await subscription.Ack(resolvedEvent);
        }
        catch (Exception ex)
        {
            _log.LogError(ex, "Error processing event {EventType}", resolvedEvent.Event.EventType);
            await subscription.Nack(PersistentSubscriptionNakEventAction.Retry, ex.Message, resolvedEvent);
        }
    }
}
```

---

## Step 1593: Snapshotting for Performance

```csharp
// Snapshot every N events to avoid full replay
public class SnapshottingOrderRepository : IOrderRepository
{
    private readonly IDocumentSession _session;
    private const int SnapshotThreshold = 50;

    public SnapshottingOrderRepository(IDocumentSession session)
        => _session = session;

    public async Task<Order?> LoadAsync(Guid id, CancellationToken ct = default)
    {
        // Load latest snapshot
        var snapshot = await _session.LoadAsync<OrderSnapshot>(id, ct);

        if (snapshot is not null)
        {
            // Only replay events AFTER the snapshot
            var events = await _session.Events.FetchStreamAsync(
                id,
                fromVersion: snapshot.Version + 1,
                token: ct);

            var order = snapshot.ToAggregate();

            if (events.Any())
                order.Rehydrate(events.Select(e => (IDomainEvent)e.Data!));

            return order;
        }

        // Full replay from start
        return await _session.Events.AggregateStreamAsync<Order>(id, token: ct);
    }

    public async Task SaveAsync(Order order, CancellationToken ct = default)
    {
        var uncommitted = order.UncommittedEvents.ToList();
        if (!uncommitted.Any()) return;

        _session.Events.Append(order.Id, uncommitted.Cast<object>().ToArray());

        // Check if we should snapshot
        var state = await _session.Events.FetchStreamStateAsync(order.Id, ct);
        var newVersion = (state?.Version ?? 0) + uncommitted.Count;

        if (newVersion % SnapshotThreshold == 0)
        {
            _session.Store(OrderSnapshot.From(order, newVersion));
        }

        await _session.SaveChangesAsync(ct);
    }
}

public class OrderSnapshot
{
    public Guid   Id        { get; init; }
    public long   Version   { get; init; }
    public string CustomerId { get; init; } = string.Empty;
    public string Status    { get; init; } = string.Empty;
    public decimal Total    { get; init; }
    public string? PaymentRef { get; init; }
    public string? TrackingNumber { get; init; }

    public static OrderSnapshot From(Order order, long version) => new()
    {
        Id             = order.Id,
        Version        = version,
        CustomerId     = order.CustomerId,
        Status         = order.Status.ToString(),
        Total          = order.Total.Amount,
        PaymentRef     = order.PaymentRef,
        TrackingNumber = order.TrackingNumber
    };

    public Order ToAggregate()
    {
        var order = new Order();
        // Reconstruct minimal state without event replay
        order.Rehydrate(new[]
        {
            new OrderPlaced(Id, CustomerId,
                Array.Empty<OrderLineItem>(),
                new Money(Total), "")
        });
        return order;
    }
}
```

---

## Step 1594: API Layer — CQRS Endpoints

```csharp
// Api/OrderEndpoints.cs
public static class OrderEndpoints
{
    public static IEndpointRouteBuilder MapOrderEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/orders").WithTags("Orders");

        // Commands (write side)
        group.MapPost("/", PlaceOrderAsync)
            .WithName("PlaceOrder");

        group.MapPost("/{id:guid}/payment", AuthorizePaymentAsync)
            .WithName("AuthorizePayment");

        group.MapPost("/{id:guid}/confirm", ConfirmOrderAsync)
            .WithName("ConfirmOrder");

        group.MapPost("/{id:guid}/ship", ShipOrderAsync)
            .WithName("ShipOrder");

        group.MapPost("/{id:guid}/cancel", CancelOrderAsync)
            .WithName("CancelOrder");

        group.MapPost("/{id:guid}/notes", AddNoteAsync)
            .WithName("AddNote");

        // Queries (read side)
        group.MapGet("/{id:guid}", GetOrderAsync)
            .WithName("GetOrder");

        group.MapGet("/{id:guid}/events", GetAuditTrailAsync)
            .WithName("GetAuditTrail");

        group.MapGet("/customer/{customerId}", GetCustomerOrdersAsync)
            .WithName("GetCustomerOrders");

        return app;
    }

    private static async Task<IResult> PlaceOrderAsync(
        PlaceOrderRequest req,
        OrderCommandHandler handler,
        CancellationToken ct)
    {
        var orderId = await handler.HandleAsync(new PlaceOrderCommand(
            req.CustomerId,
            req.Items.Select(i => new OrderLineItem(i.ProductId, i.Name, i.Quantity,
                new Money(i.UnitPrice))).ToList(),
            req.ShippingAddress), ct);

        return Results.CreatedAtRoute("GetOrder", new { id = orderId }, new { orderId });
    }

    private static async Task<IResult> GetOrderAsync(
        Guid id,
        OrderQueryService queries,
        CancellationToken ct)
    {
        var order = await queries.GetOrderAsync(id, ct);
        return order is null ? Results.NotFound() : Results.Ok(order);
    }

    private static async Task<IResult> GetAuditTrailAsync(
        Guid id,
        OrderQueryService queries,
        CancellationToken ct)
        => Results.Ok(await queries.GetAuditTrailAsync(id, ct));

    private static async Task<IResult> CancelOrderAsync(
        Guid id,
        CancelOrderRequest req,
        OrderCommandHandler handler,
        CancellationToken ct)
    {
        await handler.HandleAsync(new CancelOrderCommand(id, req.Reason, req.RequestedBy), ct);
        return Results.NoContent();
    }
}

// Request DTOs
public record PlaceOrderRequest(
    string CustomerId,
    List<OrderItemRequest> Items,
    string ShippingAddress);

public record OrderItemRequest(string ProductId, string Name, int Quantity, decimal UnitPrice);
public record CancelOrderRequest(string Reason, string RequestedBy);
```

---

## Step 1595: Testing Event Sourcing

```csharp
// Tests/OrderAggregateTests.cs
namespace EventSourcingDemo.Tests;

public class OrderAggregateTests
{
    [Fact]
    public void PlaceOrder_ValidItems_RaisesOrderPlacedEvent()
    {
        var orderId  = Guid.NewGuid();
        var items    = new[] { new OrderLineItem("p1", "Widget", 2, new Money(25m)) };

        var order = Order.Place(orderId, "cust-1", items, "123 Main St");

        Assert.Single(order.UncommittedEvents);
        var evt = Assert.IsType<OrderPlaced>(order.UncommittedEvents[0]);
        Assert.Equal(orderId, evt.OrderId);
        Assert.Equal(50m, evt.Total.Amount);
    }

    [Fact]
    public void PlaceOrder_NoItems_ThrowsDomainException()
    {
        Assert.Throws<DomainException>(() =>
            Order.Place(Guid.NewGuid(), "cust-1", [], "addr"));
    }

    [Fact]
    public void Cancel_AfterShip_ThrowsDomainException()
    {
        var order = CreateShippedOrder();

        Assert.Throws<DomainException>(() =>
            order.Cancel("changed mind", "customer"));
    }

    [Fact]
    public void StateTransition_FullLifecycle_IsCorrect()
    {
        var order = Order.Place(Guid.NewGuid(), "cust-1",
            [new OrderLineItem("p1", "Widget", 1, new Money(50m))],
            "addr");

        Assert.Equal(OrderStatus.Pending, order.Status);

        order.AuthorizePayment("pay-ref", "card");
        Assert.Equal(OrderStatus.PaymentAuthorized, order.Status);

        order.Confirm("warehouse-1", "wh-east");
        Assert.Equal(OrderStatus.Confirmed, order.Status);

        order.Ship("track-123", "UPS");
        Assert.Equal(OrderStatus.Shipped, order.Status);

        order.MarkDelivered("John Doe");
        Assert.Equal(OrderStatus.Delivered, order.Status);
    }

    [Fact]
    public void Rehydrate_FromEvents_ReconstructsState()
    {
        var orderId = Guid.NewGuid();
        var events  = new IDomainEvent[]
        {
            new OrderPlaced(orderId, "cust-1",
                [new OrderLineItem("p1", "Widget", 1, new Money(50m))],
                new Money(50m), "addr"),
            new PaymentAuthorized(orderId, "pay-ref", "card", new Money(50m)),
            new OrderConfirmed(orderId, "staff-1", "wh-east"),
            new OrderShipped(orderId, "track-123", "UPS", DateTimeOffset.UtcNow.AddDays(3))
        };

        var order = new Order();
        order.Rehydrate(events);

        Assert.Equal(OrderStatus.Shipped, order.Status);
        Assert.Equal("track-123", order.TrackingNumber);
        Assert.Equal("pay-ref", order.PaymentRef);
    }

    [Fact]
    public async Task ProjectionApply_FromOrderPlaced_SetsAllFields()
    {
        var projection = new OrderDetailsProjection();
        var orderId    = Guid.NewGuid();

        var evt = new OrderPlaced(
            orderId,
            "cust-1",
            [new OrderLineItem("p1", "Widget", 2, new Money(25m))],
            new Money(50m),
            "123 Main St");

        var view = projection.Apply(new OrderDetails(), evt);

        Assert.Equal("Pending", view.Status);
        Assert.Equal(50m, view.Total);
        Assert.Single(view.Items);
    }

    private static Order CreateShippedOrder()
    {
        var order = Order.Place(Guid.NewGuid(), "cust-1",
            [new OrderLineItem("p1", "Widget", 1, new Money(50m))], "addr");
        order.AuthorizePayment("pay-ref", "card");
        order.Confirm("staff-1", "wh-east");
        order.Ship("track-123", "UPS");
        return order;
    }
}
```

---

## Step 1596: Marten Integration Tests

```csharp
// Tests/MartenIntegrationTests.cs
using Marten;
using Marten.Testing.Harness;
using Testcontainers.PostgreSql;

namespace EventSourcingDemo.Tests;

[Collection("Marten")]
public class MartenIntegrationTests : IAsyncLifetime
{
    private readonly PostgreSqlContainer _postgres = new PostgreSqlBuilder()
        .WithImage("postgres:16-alpine")
        .Build();

    private IDocumentStore _store = null!;

    public async Task InitializeAsync()
    {
        await _postgres.StartAsync();

        _store = DocumentStore.For(opts =>
        {
            opts.Connection(_postgres.GetConnectionString());
            opts.AutoCreateSchemaObjects = AutoCreate.All;

            opts.Events.AddEventType<OrderPlaced>();
            opts.Events.AddEventType<PaymentAuthorized>();
            opts.Events.AddEventType<OrderConfirmed>();
            opts.Events.AddEventType<OrderCancelled>();

            opts.Projections.Add<OrderDetailsProjection>(ProjectionLifecycle.Inline);
        });
    }

    public async Task DisposeAsync()
    {
        _store.Dispose();
        await _postgres.DisposeAsync();
    }

    [Fact]
    public async Task AppendAndLoad_Order_ReconstructsCorrectly()
    {
        var orderId = Guid.NewGuid();

        await using var session = _store.LightweightSession();
        var repo = new MartenOrderRepository(session);

        // Create
        var order = Order.Place(orderId, "cust-integration",
            [new OrderLineItem("p1", "Widget", 2, new Money(50m))],
            "123 Integration Ave");
        await repo.SaveAsync(order);

        // Load and verify
        var loaded = await repo.LoadAsync(orderId);

        Assert.NotNull(loaded);
        Assert.Equal(OrderStatus.Pending, loaded.Status);
        Assert.Equal("cust-integration", loaded.CustomerId);
    }

    [Fact]
    public async Task InlineProjection_IsUpdatedOnSave()
    {
        var orderId = Guid.NewGuid();

        await using var session = _store.LightweightSession();
        var repo = new MartenOrderRepository(session);

        var order = Order.Place(orderId, "proj-test",
            [new OrderLineItem("p1", "Widget", 1, new Money(100m))],
            "addr");
        order.AuthorizePayment("pay-ref", "card");
        await repo.SaveAsync(order);

        // Query the projection
        var view = await session.LoadAsync<OrderDetails>(orderId);

        Assert.NotNull(view);
        Assert.Equal("PaymentAuthorized", view.Status);
        Assert.Equal(100m, view.Total);
    }

    [Fact]
    public async Task OptimisticConcurrency_ConcurrentWrite_Throws()
    {
        var orderId = Guid.NewGuid();

        // Session A writes
        await using var sessionA = _store.LightweightSession();
        var repoA = new VersionedMartenOrderRepository(sessionA);
        var orderA = Order.Place(orderId, "concurrent-test",
            [new OrderLineItem("p1", "Widget", 1, new Money(50m))], "addr");
        await repoA.SaveAsync(orderA);

        // Session B loads same order
        await using var sessionB = _store.LightweightSession();
        var repoB = new VersionedMartenOrderRepository(sessionB);
        var orderB = await repoB.LoadRequiredAsync(orderId);

        // A makes a change
        var orderALoaded = await repoA.LoadRequiredAsync(orderId);
        orderALoaded.AuthorizePayment("pay-A", "card");
        await repoA.SaveAsync(orderALoaded);

        // B tries to write to same version — should fail
        orderB.AuthorizePayment("pay-B", "card");
        await Assert.ThrowsAsync<ConcurrencyException>(
            () => repoB.SaveAsync(orderB));
    }
}
```

---

## Step 1597: Event Sourcing Checklist

```
Event Sourcing Production Checklist
═══════════════════════════════════════════════════════════

Domain Design
✅ Events are past tense (OrderPlaced, not PlaceOrder)
✅ Events are immutable (readonly records)
✅ Event fields contain ALL needed data (no FK lookups)
✅ Apply() method: pure state mutation, no side effects
✅ Commands validated before event generation
✅ Aggregate raises events — not saved by caller directly

Event Store
✅ Stream per aggregate instance (order-{guid})
✅ Optimistic concurrency via expected stream version
✅ Event type names never renamed (schema evolution via new types)
✅ Metadata: correlation ID, causation ID, user ID
✅ Snapshots for aggregates with >50 events
✅ Compaction / archival strategy for old events

Projections
✅ Inline for read-your-own-writes (order detail)
✅ Async for cross-stream (customer history, dashboard)
✅ Idempotent: applying same event twice gives same result
✅ Rebuild-friendly: projections can be rebuilt from scratch
✅ Version projections to handle schema changes

CQRS
✅ Separate command and query models
✅ Commands go through domain aggregate
✅ Queries read from projections (denormalized)
✅ No command results in direct query response (eventual consistency)
✅ Accept eventual consistency in queries

Operations
✅ Projection rebuild procedure documented
✅ Event replay tested regularly
✅ Dead letter queue for failed projections
✅ Monitoring: event lag (async projection behind)
✅ Correlation/causation IDs for distributed tracing
```

---

**Part 59 ครอบคลุม:**
- Event Sourcing pattern — events เป็น single source of truth
- `Order` aggregate ด้วย `When()` / `Apply()` / `Rehydrate()`
- Marten setup ด้วย inline + async projections
- Optimistic concurrency ด้วย stream version
- Multi-stream projections (CustomerOrderHistory, DailySummary)
- EventStoreDB persistent subscriptions
- Snapshotting ทุก 50 events เพื่อ performance
- Full unit + integration testing patterns
