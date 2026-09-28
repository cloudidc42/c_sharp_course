# Part 55: Microsoft Orleans — Distributed Actors
## Steps 1522-1550: Virtual Actors, Grains, Cluster, Streams, Reminders

---

## Step 1522: Orleans Concepts and Setup

```
Orleans Virtual Actor Model:
┌─────────────────────────────────────────────────────┐
│                    Orleans Cluster                  │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────┐ │
│  │   Silo 1     │  │   Silo 2     │  │  Silo 3   │ │
│  │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌───────┐ │ │
│  │ │OrderGrain│ │  │ │UserGrain │ │  │ │CartGr.│ │ │
│  │ │(id=123)  │ │  │ │(id=alice)│ │  │ │(id=42)│ │ │
│  │ └──────────┘ │  │ └──────────┘ │  │ └───────┘ │ │
│  └──────────────┘  └──────────────┘  └───────────┘ │
│         ↑                  ↑                ↑        │
│    Each grain = one logical entity, one at a time   │
│    Location transparent — Orleans routes calls      │
└─────────────────────────────────────────────────────┘
```

```csharp
// Install: Microsoft.Orleans.Server, Microsoft.Orleans.Client
// Microsoft.Orleans.Persistence.AdoNet, Microsoft.Orleans.Streaming.Kafka

// Program.cs (Silo host)
var builder = WebApplication.CreateBuilder(args);

builder.UseOrleans(siloBuilder =>
{
    siloBuilder
        // Clustering: all silos find each other via this store
        .UseAdoNetClustering(options =>
        {
            options.ConnectionString = builder.Configuration.GetConnectionString("Postgres")!;
            options.Invariant = "Npgsql";
        })
        // Grain state persistence
        .AddAdoNetGrainStorage("orders", options =>
        {
            options.ConnectionString = builder.Configuration.GetConnectionString("Postgres")!;
            options.Invariant = "Npgsql";
            options.UseJsonFormat = true;
        })
        // Reminder service (persistent timers)
        .UseAdoNetReminderService(options =>
        {
            options.ConnectionString = builder.Configuration.GetConnectionString("Postgres")!;
            options.Invariant = "Npgsql";
        })
        // Streams
        .AddKafkaStreams("OrderStreams", configurator =>
        {
            configurator.ConfigureKafka(options =>
            {
                options.BrokerList = "kafka:9092";
            });
        })
        // Endpoints (auto-configured in cloud environments)
        .ConfigureEndpoints(siloPort: 11111, gatewayPort: 30000)
        // Dashboard
        .UseDashboard(options => { options.Port = 8080; });
});

var app = builder.Build();
app.Run();
```

---

## Step 1523: Grain Interfaces

```csharp
// Grain interfaces — define the API that clients and other grains call

// Order grain — one instance per order ID
public interface IOrderGrain : IGrainWithGuidKey
{
    Task<OrderState> GetStateAsync();
    Task<OrderSummary> CreateAsync(CreateOrderCommand command);
    Task ConfirmAsync(string confirmedBy);
    Task ShipAsync(string trackingNumber, string carrier);
    Task DeliverAsync();
    Task CancelAsync(string reason);
    Task<bool> IsActiveAsync();
}

// User grain — one instance per user ID
public interface IUserGrain : IGrainWithStringKey
{
    Task<UserProfile> GetProfileAsync();
    Task UpdateProfileAsync(UpdateProfileCommand command);
    Task<List<OrderSummary>> GetOrderHistoryAsync(int page, int pageSize);
    Task RecordOrderAsync(Guid orderId);
    Task<int> GetOrderCountAsync();
}

// Cart grain — one instance per user session
public interface ICartGrain : IGrainWithStringKey
{
    Task AddItemAsync(CartItem item);
    Task RemoveItemAsync(Guid productId);
    Task UpdateQuantityAsync(Guid productId, int quantity);
    Task<CartState> GetCartAsync();
    Task<decimal> GetTotalAsync();
    Task ClearAsync();
    Task<OrderSummary> CheckoutAsync(PaymentDetails payment);
}

// Inventory grain — one per product
public interface IInventoryGrain : IGrainWithGuidKey
{
    Task<int> GetAvailableStockAsync();
    Task<bool> ReserveAsync(int quantity, Guid orderId);
    Task ReleaseReservationAsync(Guid orderId);
    Task ConfirmReservationAsync(Guid orderId);
    Task RestockAsync(int quantity);
}

// Grain observer — for pub/sub notifications
public interface IOrderObserver : IGrainObserver
{
    void OnOrderStatusChanged(Guid orderId, string newStatus);
    void OnOrderShipped(Guid orderId, string trackingNumber);
}
```

---

## Step 1524: Grain Implementation

```csharp
// OrderGrain — stateful actor
[StorageProvider(ProviderName = "orders")]
public class OrderGrain(
    ILogger<OrderGrain> logger,
    [PersistentState("state", "orders")] IPersistentState<OrderGrainState> state)
    : Grain, IOrderGrain
{
    private readonly ObserverManager<IOrderObserver> _observers = new(
        TimeSpan.FromMinutes(5),
        logger
    );

    public Task<OrderState> GetStateAsync() =>
        Task.FromResult(state.State.ToOrderState());

    public async Task<OrderSummary> CreateAsync(CreateOrderCommand command)
    {
        if (state.State.Status != OrderStatus.None)
            throw new InvalidOperationException("Order already exists");

        var orderId = this.GetPrimaryKey();

        state.State = new OrderGrainState
        {
            Id = orderId,
            CustomerId = command.CustomerId,
            Items = command.Items,
            Total = command.Items.Sum(i => i.Quantity * i.UnitPrice),
            Status = OrderStatus.Pending,
            CreatedAt = DateTimeOffset.UtcNow
        };

        await state.WriteStateAsync();

        // Reserve inventory for each item
        foreach (var item in command.Items)
        {
            var inventoryGrain = GrainFactory.GetGrain<IInventoryGrain>(item.ProductId);
            var reserved = await inventoryGrain.ReserveAsync(item.Quantity, orderId);
            if (!reserved)
            {
                await CancelAsync($"Insufficient stock for product {item.ProductId}");
                throw new InsufficientStockException(item.ProductId);
            }
        }

        // Notify user grain
        var userGrain = GrainFactory.GetGrain<IUserGrain>(command.CustomerId.ToString());
        await userGrain.RecordOrderAsync(orderId);

        logger.LogInformation("Order {OrderId} created for customer {CustomerId}", orderId, command.CustomerId);

        return state.State.ToSummary();
    }

    public async Task ConfirmAsync(string confirmedBy)
    {
        if (state.State.Status != OrderStatus.Pending)
            throw new InvalidOperationException($"Cannot confirm order in {state.State.Status} state");

        state.State.Status = OrderStatus.Confirmed;
        state.State.ConfirmedAt = DateTimeOffset.UtcNow;
        state.State.ConfirmedBy = confirmedBy;

        await state.WriteStateAsync();

        // Confirm inventory reservations
        foreach (var item in state.State.Items)
        {
            var inventoryGrain = GrainFactory.GetGrain<IInventoryGrain>(item.ProductId);
            await inventoryGrain.ConfirmReservationAsync(this.GetPrimaryKey());
        }

        // Notify observers
        _observers.Notify(o => o.OnOrderStatusChanged(this.GetPrimaryKey(), "Confirmed"));

        logger.LogInformation("Order {OrderId} confirmed by {User}", this.GetPrimaryKey(), confirmedBy);
    }

    public async Task ShipAsync(string trackingNumber, string carrier)
    {
        if (state.State.Status != OrderStatus.Confirmed)
            throw new InvalidOperationException($"Cannot ship order in {state.State.Status} state");

        state.State.Status = OrderStatus.Shipped;
        state.State.TrackingNumber = trackingNumber;
        state.State.Carrier = carrier;
        state.State.ShippedAt = DateTimeOffset.UtcNow;

        await state.WriteStateAsync();

        _observers.Notify(o => o.OnOrderShipped(this.GetPrimaryKey(), trackingNumber));
    }

    public async Task DeliverAsync()
    {
        if (state.State.Status != OrderStatus.Shipped)
            throw new InvalidOperationException("Order must be shipped before delivery");

        state.State.Status = OrderStatus.Delivered;
        state.State.DeliveredAt = DateTimeOffset.UtcNow;
        await state.WriteStateAsync();

        _observers.Notify(o => o.OnOrderStatusChanged(this.GetPrimaryKey(), "Delivered"));

        // Grain is logically done — can be deactivated from memory
        DeactivateOnIdle();
    }

    public async Task CancelAsync(string reason)
    {
        if (state.State.Status is OrderStatus.Delivered or OrderStatus.Shipped)
            throw new InvalidOperationException("Cannot cancel delivered/shipped order");

        // Release inventory reservations
        foreach (var item in state.State.Items)
        {
            var inventoryGrain = GrainFactory.GetGrain<IInventoryGrain>(item.ProductId);
            await inventoryGrain.ReleaseReservationAsync(this.GetPrimaryKey());
        }

        state.State.Status = OrderStatus.Cancelled;
        state.State.CancellationReason = reason;
        state.State.CancelledAt = DateTimeOffset.UtcNow;
        await state.WriteStateAsync();

        _observers.Notify(o => o.OnOrderStatusChanged(this.GetPrimaryKey(), "Cancelled"));
        DeactivateOnIdle();
    }

    public Task<bool> IsActiveAsync() =>
        Task.FromResult(state.State.Status is OrderStatus.Pending or OrderStatus.Confirmed or OrderStatus.Shipped);

    // Observer subscription
    public Task Subscribe(IOrderObserver observer)
    {
        _observers.Subscribe(observer, observer);
        return Task.CompletedTask;
    }

    public Task Unsubscribe(IOrderObserver observer)
    {
        _observers.Unsubscribe(observer);
        return Task.CompletedTask;
    }
}

// Grain state (persisted)
public class OrderGrainState
{
    public Guid Id { get; set; }
    public Guid CustomerId { get; set; }
    public List<OrderItemState> Items { get; set; } = [];
    public decimal Total { get; set; }
    public OrderStatus Status { get; set; } = OrderStatus.None;
    public string? ConfirmedBy { get; set; }
    public string? TrackingNumber { get; set; }
    public string? Carrier { get; set; }
    public string? CancellationReason { get; set; }
    public DateTimeOffset CreatedAt { get; set; }
    public DateTimeOffset? ConfirmedAt { get; set; }
    public DateTimeOffset? ShippedAt { get; set; }
    public DateTimeOffset? DeliveredAt { get; set; }
    public DateTimeOffset? CancelledAt { get; set; }

    public OrderSummary ToSummary() => new(Id, CustomerId, Total, Status, CreatedAt);
    public OrderState ToOrderState() => new(/* map fields */);
}
```

---

## Step 1525: Grain Timers and Reminders

```csharp
// Grain with reminders — survive silo restarts
[StorageProvider(ProviderName = "orders")]
public class SubscriptionGrain(
    [PersistentState("state", "orders")] IPersistentState<SubscriptionState> state,
    ILogger<SubscriptionGrain> logger)
    : Grain, ISubscriptionGrain, IRemindable
{
    private const string RenewalReminderName = "renewal";
    private const string ExpiryReminderName = "expiry-warning";

    public override async Task OnActivateAsync(CancellationToken ct)
    {
        await base.OnActivateAsync(ct);

        // Restore reminders when grain activates (after silo restart)
        if (state.State.IsActive && state.State.RenewalDate > DateTimeOffset.UtcNow)
        {
            var delay = state.State.RenewalDate - DateTimeOffset.UtcNow - TimeSpan.FromDays(7);
            if (delay > TimeSpan.Zero)
                await RegisterOrUpdateReminder(RenewalReminderName, delay, TimeSpan.FromDays(365));
        }
    }

    public async Task ActivateSubscriptionAsync(SubscriptionPlan plan)
    {
        state.State = new SubscriptionState
        {
            SubscriberId = this.GetPrimaryKey(),
            Plan = plan,
            StartDate = DateTimeOffset.UtcNow,
            RenewalDate = DateTimeOffset.UtcNow.AddDays(plan.DurationDays),
            IsActive = true
        };

        await state.WriteStateAsync();

        // Register reminder 7 days before renewal — survives silo restarts
        var daysUntilRenewal = plan.DurationDays - 7;
        if (daysUntilRenewal > 0)
        {
            await RegisterOrUpdateReminder(
                RenewalReminderName,
                TimeSpan.FromDays(daysUntilRenewal),
                TimeSpan.FromDays(365)
            );
        }

        // Reminder 1 day before expiry if not renewed
        await RegisterOrUpdateReminder(
            ExpiryReminderName,
            TimeSpan.FromDays(plan.DurationDays - 1),
            TimeSpan.FromDays(365)
        );
    }

    // Called by Orleans when reminder fires (even after cluster restart)
    public async Task ReceiveReminder(string reminderName, TickStatus status)
    {
        switch (reminderName)
        {
            case RenewalReminderName:
                logger.LogInformation("Renewal reminder for subscriber {Id}", this.GetPrimaryKey());
                var emailGrain = GrainFactory.GetGrain<IEmailGrain>(this.GetPrimaryKey());
                await emailGrain.SendRenewalReminderAsync(state.State);
                break;

            case ExpiryReminderName:
                if (!state.State.IsRenewed)
                {
                    logger.LogInformation("Expiry warning for subscriber {Id}", this.GetPrimaryKey());
                    var notifyGrain = GrainFactory.GetGrain<IEmailGrain>(this.GetPrimaryKey());
                    await notifyGrain.SendExpiryWarningAsync(state.State);
                }
                break;
        }
    }

    public async Task RenewAsync(SubscriptionPlan newPlan)
    {
        state.State.RenewalDate = DateTimeOffset.UtcNow.AddDays(newPlan.DurationDays);
        state.State.IsRenewed = true;
        await state.WriteStateAsync();

        // Reschedule renewal reminder
        await RegisterOrUpdateReminder(
            RenewalReminderName,
            TimeSpan.FromDays(newPlan.DurationDays - 7),
            TimeSpan.FromDays(365)
        );
    }
}

// Grain with in-memory timer (does NOT survive silo restarts)
public class RateLimiterGrain : Grain, IRateLimiterGrain
{
    private int _requestCount;
    private IDisposable? _resetTimer;

    public override Task OnActivateAsync(CancellationToken ct)
    {
        // Reset counter every minute
        _resetTimer = RegisterGrainTimer(ResetCounterAsync, TimeSpan.FromMinutes(1), TimeSpan.FromMinutes(1));
        return base.OnActivateAsync(ct);
    }

    public Task<bool> TryConsumeAsync()
    {
        const int maxRequestsPerMinute = 100;
        if (_requestCount >= maxRequestsPerMinute) return Task.FromResult(false);
        _requestCount++;
        return Task.FromResult(true);
    }

    private Task ResetCounterAsync()
    {
        _requestCount = 0;
        return Task.CompletedTask;
    }

    public override Task OnDeactivateAsync(DeactivationReason reason, CancellationToken ct)
    {
        _resetTimer?.Dispose();
        return base.OnDeactivateAsync(reason, ct);
    }
}
```

---

## Step 1526: Orleans Streams

```csharp
// Orleans Streams — pub/sub within the cluster

// Producer grain
public class OrderProcessingGrain(ILogger<OrderProcessingGrain> logger)
    : Grain, IOrderProcessingGrain
{
    private IAsyncStream<OrderEvent>? _orderStream;

    public override async Task OnActivateAsync(CancellationToken ct)
    {
        await base.OnActivateAsync(ct);

        var streamProvider = this.GetStreamProvider("OrderStreams");
        _orderStream = streamProvider.GetStream<OrderEvent>(
            StreamId.Create("orders", this.GetPrimaryKey()));
    }

    public async Task ProcessOrderAsync(CreateOrderCommand command)
    {
        // Do work...
        var order = CreateOrder(command);

        // Publish to stream — all subscribers receive this
        await _orderStream!.OnNextAsync(new OrderCreatedEvent(
            OrderId: order.Id,
            CustomerId: order.CustomerId,
            Total: order.Total,
            OccurredAt: DateTimeOffset.UtcNow
        ));
    }
}

// Consumer grain — subscribes to the stream
public class OrderNotificationGrain(ILogger<OrderNotificationGrain> logger)
    : Grain, IOrderNotificationGrain
{
    private StreamSubscriptionHandle<OrderEvent>? _subscription;

    public override async Task OnActivateAsync(CancellationToken ct)
    {
        await base.OnActivateAsync(ct);

        var streamProvider = this.GetStreamProvider("OrderStreams");

        // Subscribe to ALL orders (namespace-level subscription)
        var allOrdersStream = streamProvider.GetStream<OrderEvent>(
            StreamId.Create("all-orders", Guid.Empty));

        _subscription = await allOrdersStream.SubscribeAsync(
            onNextAsync: HandleOrderEventAsync,
            onErrorAsync: HandleErrorAsync,
            onCompletedAsync: () => Task.CompletedTask
        );
    }

    private async Task HandleOrderEventAsync(OrderEvent @event, StreamSequenceToken? token)
    {
        switch (@event)
        {
            case OrderCreatedEvent created:
                logger.LogInformation("New order {OrderId} for ${Total}", created.OrderId, created.Total);
                var userGrain = GrainFactory.GetGrain<IUserGrain>(created.CustomerId.ToString());
                // Send notification through email grain
                break;

            case OrderShippedEvent shipped:
                logger.LogInformation("Order {OrderId} shipped: {Tracking}", shipped.OrderId, shipped.TrackingNumber);
                break;
        }
    }

    private Task HandleErrorAsync(Exception ex)
    {
        logger.LogError(ex, "Stream error");
        return Task.CompletedTask;
    }

    public override async Task OnDeactivateAsync(DeactivationReason reason, CancellationToken ct)
    {
        if (_subscription is not null)
            await _subscription.UnsubscribeAsync();
        await base.OnDeactivateAsync(reason, ct);
    }
}

// Rewindable stream consumer — replay from a specific sequence token
public class AuditGrain : Grain, IAuditGrain
{
    public async Task ReplayOrdersFromAsync(StreamSequenceToken fromToken)
    {
        var streamProvider = this.GetStreamProvider("OrderStreams");
        var stream = streamProvider.GetStream<OrderEvent>(
            StreamId.Create("all-orders", Guid.Empty));

        // Resume from a specific point — handles crash recovery
        await stream.SubscribeAsync(
            HandleOrderEventAsync,
            _ => Task.CompletedTask,
            () => Task.CompletedTask,
            fromToken  // Replay from this position
        );
    }

    private Task HandleOrderEventAsync(OrderEvent @event, StreamSequenceToken? token)
    {
        // Write to audit log
        return Task.CompletedTask;
    }
}
```

---

## Step 1527: Cart Grain — Stateful Session

```csharp
[StorageProvider(ProviderName = "orders")]
public class CartGrain(
    [PersistentState("cart", "orders")] IPersistentState<CartGrainState> state,
    ILogger<CartGrain> logger)
    : Grain, ICartGrain
{
    public async Task AddItemAsync(CartItem item)
    {
        var existing = state.State.Items.FirstOrDefault(i => i.ProductId == item.ProductId);
        if (existing is not null)
            state.State.Items.Remove(existing);

        state.State.Items.Add(item);
        state.State.UpdatedAt = DateTimeOffset.UtcNow;

        await state.WriteStateAsync();
    }

    public async Task RemoveItemAsync(Guid productId)
    {
        state.State.Items.RemoveAll(i => i.ProductId == productId);
        state.State.UpdatedAt = DateTimeOffset.UtcNow;
        await state.WriteStateAsync();
    }

    public async Task UpdateQuantityAsync(Guid productId, int quantity)
    {
        var item = state.State.Items.FirstOrDefault(i => i.ProductId == productId);
        if (item is null) return;

        if (quantity <= 0)
            state.State.Items.Remove(item);
        else
            state.State.Items[state.State.Items.IndexOf(item)] = item with { Quantity = quantity };

        state.State.UpdatedAt = DateTimeOffset.UtcNow;
        await state.WriteStateAsync();
    }

    public Task<CartState> GetCartAsync() =>
        Task.FromResult(new CartState(
            Items: state.State.Items,
            Total: state.State.Items.Sum(i => i.Quantity * i.UnitPrice),
            UpdatedAt: state.State.UpdatedAt
        ));

    public Task<decimal> GetTotalAsync() =>
        Task.FromResult(state.State.Items.Sum(i => i.Quantity * i.UnitPrice));

    public async Task ClearAsync()
    {
        state.State.Items.Clear();
        await state.WriteStateAsync();
    }

    public async Task<OrderSummary> CheckoutAsync(PaymentDetails payment)
    {
        if (state.State.Items.Count == 0)
            throw new InvalidOperationException("Cart is empty");

        var userId = this.GetPrimaryKeyString();
        var orderId = Guid.NewGuid();
        var orderGrain = GrainFactory.GetGrain<IOrderGrain>(orderId);

        var summary = await orderGrain.CreateAsync(new CreateOrderCommand(
            CustomerId: Guid.Parse(userId),
            Items: state.State.Items.Select(i => new OrderItem(i.ProductId, i.Name, i.Quantity, i.UnitPrice)).ToList(),
            Payment: payment
        ));

        // Clear cart after successful checkout
        await ClearAsync();

        logger.LogInformation("Cart checkout complete: order {OrderId}", orderId);
        return summary;
    }
}
```

---

## Step 1528: Calling Grains from ASP.NET Core

```csharp
// IGrainFactory injected into controllers/minimal APIs

[ApiController]
[Route("api/v1/orders")]
public class OrdersController(IGrainFactory grains, ILogger<OrdersController> logger) : ControllerBase
{
    [HttpPost]
    public async Task<IActionResult> CreateOrder(
        [FromBody] CreateOrderRequest request,
        CancellationToken ct)
    {
        var orderId = Guid.NewGuid();
        var orderGrain = grains.GetGrain<IOrderGrain>(orderId);

        var summary = await orderGrain.CreateAsync(new CreateOrderCommand(
            request.CustomerId, request.Items, request.Payment));

        return CreatedAtAction(nameof(GetOrder), new { id = orderId }, summary);
    }

    [HttpGet("{id:guid}")]
    public async Task<IActionResult> GetOrder(Guid id)
    {
        var orderGrain = grains.GetGrain<IOrderGrain>(id);
        var state = await orderGrain.GetStateAsync();

        if (state.Status == OrderStatus.None)
            return NotFound();

        return Ok(state);
    }

    [HttpPost("{id:guid}/confirm")]
    [Authorize(Roles = "Admin,Warehouse")]
    public async Task<IActionResult> ConfirmOrder(Guid id)
    {
        var orderGrain = grains.GetGrain<IOrderGrain>(id);
        await orderGrain.ConfirmAsync(User.Identity!.Name!);
        return NoContent();
    }

    [HttpPost("{id:guid}/ship")]
    [Authorize(Roles = "Warehouse")]
    public async Task<IActionResult> ShipOrder(
        Guid id,
        [FromBody] ShipOrderRequest request)
    {
        var orderGrain = grains.GetGrain<IOrderGrain>(id);
        await orderGrain.ShipAsync(request.TrackingNumber, request.Carrier);
        return NoContent();
    }
}

// Minimal API version
app.MapPost("/api/v1/cart/{userId}/items", async (
    string userId,
    [FromBody] CartItem item,
    IGrainFactory grains) =>
{
    var cartGrain = grains.GetGrain<ICartGrain>(userId);
    await cartGrain.AddItemAsync(item);
    return Results.Ok(await cartGrain.GetCartAsync());
});

app.MapPost("/api/v1/cart/{userId}/checkout", async (
    string userId,
    [FromBody] PaymentDetails payment,
    IGrainFactory grains) =>
{
    var cartGrain = grains.GetGrain<ICartGrain>(userId);
    var order = await cartGrain.CheckoutAsync(payment);
    return Results.Created($"/api/v1/orders/{order.Id}", order);
});
```

---

## Step 1529: Orleans Dashboard and Monitoring

```csharp
// Orleans Dashboard — built-in web UI
siloBuilder.UseDashboard(options =>
{
    options.Host = "*";
    options.Port = 8080;
    options.HostSelf = true;
    options.CounterUpdateIntervalMs = 1000;
});

// Custom Orleans metrics (OpenTelemetry)
siloBuilder.AddActivityPropagation(); // Propagates trace context through grain calls

// Custom grain metrics
public class MetricsOrderGrain(
    [PersistentState("state", "orders")] IPersistentState<OrderGrainState> state)
    : OrderGrain(/* ... */), IOrderGrain
{
    private static readonly Counter<long> OrdersCreatedCounter =
        OrderTelemetry.Meter.CreateCounter<long>("grains.orders.created");

    private static readonly Histogram<double> GrainActivationDuration =
        OrderTelemetry.Meter.CreateHistogram<double>("grains.activation.duration", "ms");

    private DateTimeOffset _activatedAt;

    public override Task OnActivateAsync(CancellationToken ct)
    {
        _activatedAt = DateTimeOffset.UtcNow;
        return base.OnActivateAsync(ct);
    }

    public override async Task<OrderSummary> CreateAsync(CreateOrderCommand command)
    {
        var result = await base.CreateAsync(command);
        OrdersCreatedCounter.Add(1, new("order.status", "created"));
        return result;
    }

    public override Task OnDeactivateAsync(DeactivationReason reason, CancellationToken ct)
    {
        var lifetime = (DateTimeOffset.UtcNow - _activatedAt).TotalMilliseconds;
        GrainActivationDuration.Record(lifetime, new("grain.type", nameof(OrderGrain)));
        return base.OnDeactivateAsync(reason, ct);
    }
}
```

---

## Step 1530: Orleans in Production — Cluster Configuration

```yaml
# docker-compose.orleans.yml
services:
  silo-1:
    image: myapp-silo:latest
    environment:
      ORLEANS_SILO_PORT: "11111"
      ORLEANS_GATEWAY_PORT: "30000"
      POSTGRES_CONNECTION: "Host=postgres;Database=orleans;Username=app;Password=secret"
      ASPNETCORE_URLS: "http://+:5000"
    ports:
      - "11111:11111"
      - "30000:30000"

  silo-2:
    image: myapp-silo:latest
    environment:
      ORLEANS_SILO_PORT: "11111"
      ORLEANS_GATEWAY_PORT: "30000"
      POSTGRES_CONNECTION: "Host=postgres;Database=orleans;Username=app;Password=secret"
    ports:
      - "11112:11111"
      - "30001:30000"

  silo-3:
    image: myapp-silo:latest
    environment:
      ORLEANS_SILO_PORT: "11111"
      ORLEANS_GATEWAY_PORT: "30000"
      POSTGRES_CONNECTION: "Host=postgres;Database=orleans;Username=app;Password=secret"
    ports:
      - "11113:11111"
      - "30002:30000"

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: orleans
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
    volumes:
      - postgres-data:/var/lib/postgresql/data
      # Orleans schema
      - ./orleans-schema.sql:/docker-entrypoint-initdb.d/orleans-schema.sql

  kafka:
    image: bitnami/kafka:3.7
    environment:
      KAFKA_CFG_NODE_ID: "1"
      KAFKA_CFG_PROCESS_ROLES: "broker,controller"
      KAFKA_CFG_LISTENERS: "PLAINTEXT://:9092,CONTROLLER://:9093"
      KAFKA_CFG_ADVERTISED_LISTENERS: "PLAINTEXT://kafka:9092"
      KAFKA_CFG_CONTROLLER_QUORUM_VOTERS: "1@kafka:9093"
```

```csharp
// Kubernetes deployment with Orleans
// Silo discovers cluster members via Kubernetes API

siloBuilder.UseKubernetesClustering(); // Auto-discover pods

// Or via Azure Blob / Azure Table Storage
siloBuilder.UseAzureTablesClustering(options =>
{
    options.ConfigureTableServiceClient("UseDevelopmentStorage=true");
});
```

---

## Step 1531: Testing Orleans Grains

```csharp
// Install: Microsoft.Orleans.TestingHost

public class OrderGrainTests
{
    [Fact]
    public async Task CreateOrder_SetsStatusToPending()
    {
        var host = await new TestClusterBuilder()
            .AddSiloBuilderConfigurator<TestSiloConfigurator>()
            .Build()
            .DeployAsync();

        try
        {
            var orderId = Guid.NewGuid();
            var orderGrain = host.GrainFactory.GetGrain<IOrderGrain>(orderId);

            var summary = await orderGrain.CreateAsync(new CreateOrderCommand(
                CustomerId: Guid.NewGuid(),
                Items: [new("Product A", Guid.NewGuid(), 2, 10m)],
                Payment: new PaymentDetails("tok_test")
            ));

            summary.Status.Should().Be(OrderStatus.Pending);
            summary.Total.Should().Be(20m);

            var state = await orderGrain.GetStateAsync();
            state.Status.Should().Be(OrderStatus.Pending);
        }
        finally
        {
            await host.StopAllSilosAsync();
        }
    }

    [Fact]
    public async Task ConfirmOrder_AfterCreate_StatusBecomesConfirmed()
    {
        var host = await new TestClusterBuilder()
            .AddSiloBuilderConfigurator<TestSiloConfigurator>()
            .Build()
            .DeployAsync();

        try
        {
            var orderGrain = host.GrainFactory.GetGrain<IOrderGrain>(Guid.NewGuid());
            await orderGrain.CreateAsync(new CreateOrderCommand(
                Guid.NewGuid(), [new("P", Guid.NewGuid(), 1, 50m)], new("tok_test")));

            await orderGrain.ConfirmAsync("admin@example.com");

            var state = await orderGrain.GetStateAsync();
            state.Status.Should().Be(OrderStatus.Confirmed);
        }
        finally
        {
            await host.StopAllSilosAsync();
        }
    }
}

// Test cluster configuration
public class TestSiloConfigurator : ISiloConfigurator
{
    public void Configure(ISiloBuilder siloBuilder)
    {
        siloBuilder
            .AddMemoryGrainStorage("orders")
            .AddMemoryGrainStorage("PubSubStore")  // Required for streams
            .AddMemoryStreams("OrderStreams");
    }
}
```

---

## Summary: Orleans vs. Other Approaches

```
When to use Orleans:
  ✅ Entities with natural identity (user, order, session, device)
  ✅ Many small stateful objects (millions of users, IoT devices)
  ✅ Complex workflows with state (order lifecycle, saga)
  ✅ Real-time per-user notifications
  ✅ Rate limiting, quota management
  ✅ Distributed caching with TTL

When NOT to use Orleans:
  ❌ Simple CRUD without complex logic (use EF Core + REST)
  ❌ Heavy batch processing (use Hangfire, Quartz.NET)
  ❌ Stateless microservices (use standard ASP.NET Core)
  ❌ Teams unfamiliar with actor model

Orleans vs. MassTransit Sagas:
  Orleans Grain  → live in memory, direct call, low latency
  MassTransit    → persistent workflow, message-driven, more durable
```

---

*Part 55 complete — Steps 1522-1531. Topics: virtual actor model, grain interfaces (IGrainWithGuidKey/StringKey), grain implementation with IPersistentState, grain factory, observer pattern, timers vs reminders (survive restarts), Orleans Streams (pub/sub), rewindable streams, cart grain checkout flow, ASP.NET Core integration, Orleans Dashboard, OpenTelemetry, production cluster with Docker/Kubernetes, TestCluster for unit testing.*
