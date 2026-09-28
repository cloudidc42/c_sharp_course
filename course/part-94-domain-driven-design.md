# Part 94: Domain-Driven Design (DDD) — Tactical & Strategic Patterns

## Steps 2133–2148

---

## Step 2133: DDD Building Blocks

```
DDD Tactical Patterns:
┌─────────────────────────────────────────────────────────────┐
│  Entity       │  Has identity (ID), mutable lifecycle       │
│  Value Object │  No identity, immutable, equality by value  │
│  Aggregate    │  Cluster of entities, consistency boundary  │
│  Domain Event │  Something that happened in the domain      │
│  Repository   │  Persistence abstraction for aggregates     │
│  Domain Svc   │  Logic that doesn't belong to an entity     │
│  Factory      │  Complex object creation logic              │
└─────────────────────────────────────────────────────────────┘

DDD Strategic Patterns:
┌─────────────────────────────────────────────────────────────┐
│  Bounded Context  │  Explicit model boundary                │
│  Ubiquitous Lang  │  Shared vocabulary dev ↔ domain expert  │
│  Context Map      │  Relationships between BCs              │
│  Anti-Corruption  │  Layer preventing model pollution       │
│  Layer            │                                         │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 2134: Value Objects

```csharp
// Value Object: equality by value, immutable
public sealed class Money : IEquatable<Money>
{
    public decimal Amount   { get; }
    public string  Currency { get; }

    private Money(decimal amount, string currency)
    {
        if (amount < 0)          throw new ArgumentException("Amount cannot be negative", nameof(amount));
        if (string.IsNullOrWhiteSpace(currency)) throw new ArgumentException("Currency required", nameof(currency));
        if (currency.Length != 3) throw new ArgumentException("Currency must be ISO 4217 code", nameof(currency));

        Amount   = amount;
        Currency = currency.ToUpperInvariant();
    }

    public static Money Of(decimal amount, string currency) => new(amount, currency);

    public static Money Zero(string currency = "USD") => new(0m, currency);

    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException($"Cannot add {Currency} and {other.Currency}");
        return new Money(Amount + other.Amount, Currency);
    }

    public Money Subtract(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException($"Cannot subtract {Currency} from {other.Currency}");
        if (Amount < other.Amount)
            throw new InvalidOperationException("Cannot subtract: result would be negative");
        return new Money(Amount - other.Amount, Currency);
    }

    public Money Multiply(decimal factor) => new(Amount * factor, Currency);

    public bool Equals(Money? other) =>
        other is not null && Amount == other.Amount && Currency == other.Currency;

    public override bool Equals(object? obj) => obj is Money other && Equals(other);
    public override int  GetHashCode()        => HashCode.Combine(Amount, Currency);
    public override string ToString()         => $"{Amount:F2} {Currency}";

    public static bool operator ==(Money? left, Money? right) => left?.Equals(right) ?? right is null;
    public static bool operator !=(Money? left, Money? right) => !(left == right);
    public static Money operator +(Money left, Money right)   => left.Add(right);
    public static Money operator -(Money left, Money right)   => left.Subtract(right);
}

// Address value object
public sealed record Address
{
    public string Street     { get; }
    public string City       { get; }
    public string State      { get; }
    public string PostalCode { get; }
    public string Country    { get; }

    private Address(string street, string city, string state, string postalCode, string country)
    {
        if (string.IsNullOrWhiteSpace(street))     throw new ArgumentException("Street required");
        if (string.IsNullOrWhiteSpace(city))       throw new ArgumentException("City required");
        if (string.IsNullOrWhiteSpace(postalCode)) throw new ArgumentException("Postal code required");
        if (string.IsNullOrWhiteSpace(country))    throw new ArgumentException("Country required");

        Street     = street.Trim();
        City       = city.Trim();
        State      = state?.Trim() ?? string.Empty;
        PostalCode = postalCode.Trim();
        Country    = country.ToUpperInvariant().Trim();
    }

    public static Address Create(string street, string city, string state, string postalCode, string country)
        => new(street, city, state, postalCode, country);

    public Address WithStreet(string newStreet) =>
        Create(newStreet, City, State, PostalCode, Country);
}

// Email value object
public sealed class Email : IEquatable<Email>
{
    private static readonly Regex Pattern = new(
        @"^[^@\s]+@[^@\s]+\.[^@\s]+$",
        RegexOptions.Compiled | RegexOptions.IgnoreCase);

    public string Value { get; }

    private Email(string value) => Value = value.ToLowerInvariant();

    public static Email Create(string value)
    {
        if (string.IsNullOrWhiteSpace(value))
            throw new ArgumentException("Email is required");
        if (!Pattern.IsMatch(value))
            throw new ArgumentException($"'{value}' is not a valid email address");
        return new Email(value);
    }

    public bool Equals(Email? other)     => other is not null && Value == other.Value;
    public override bool Equals(object? obj) => obj is Email other && Equals(other);
    public override int  GetHashCode()    => Value.GetHashCode(StringComparison.OrdinalIgnoreCase);
    public override string ToString()     => Value;

    public static implicit operator string(Email email) => email.Value;
}
```

---

## Step 2135: Entities & Aggregate Root

```csharp
// Base Entity
public abstract class Entity<TId> : IEquatable<Entity<TId>>
    where TId : notnull
{
    public TId Id { get; protected set; } = default!;

    public bool Equals(Entity<TId>? other) =>
        other is not null && Id.Equals(other.Id);

    public override bool Equals(object? obj) =>
        obj is Entity<TId> entity && Equals(entity);

    public override int GetHashCode() => Id.GetHashCode();

    public static bool operator ==(Entity<TId>? left, Entity<TId>? right) =>
        left?.Equals(right) ?? right is null;

    public static bool operator !=(Entity<TId>? left, Entity<TId>? right) =>
        !(left == right);
}

// Aggregate Root with domain events
public abstract class AggregateRoot<TId> : Entity<TId>
    where TId : notnull
{
    private readonly List<IDomainEvent> _events = [];

    public IReadOnlyList<IDomainEvent> DomainEvents => _events.AsReadOnly();
    public int Version { get; protected set; }

    protected void RaiseDomainEvent(IDomainEvent @event)
    {
        _events.Add(@event);
        Version++;
    }

    public IReadOnlyList<IDomainEvent> PopDomainEvents()
    {
        var events = _events.ToList();
        _events.Clear();
        return events;
    }
}

// Order Aggregate
public class Order : AggregateRoot<OrderId>
{
    private readonly List<OrderItem> _items = [];

    // Private constructor — use factory method
    private Order() { }

    public CustomerId  CustomerId { get; private set; } = null!;
    public Money       Total      { get; private set; } = null!;
    public OrderStatus Status     { get; private set; }
    public Address     ShippingAddress { get; private set; } = null!;
    public IReadOnlyList<OrderItem> Items => _items.AsReadOnly();
    public DateTimeOffset CreatedAt  { get; private set; }
    public DateTimeOffset? UpdatedAt { get; private set; }

    // Factory method — ensures valid state on creation
    public static Order Place(
        CustomerId customerId,
        IReadOnlyList<OrderItemRequest> items,
        Address shippingAddress)
    {
        Guard.Against.Null(customerId, nameof(customerId));
        Guard.Against.NullOrEmpty(items, nameof(items));
        Guard.Against.Null(shippingAddress, nameof(shippingAddress));

        var order = new Order
        {
            Id              = OrderId.New(),
            CustomerId      = customerId,
            ShippingAddress = shippingAddress,
            Status          = OrderStatus.Pending,
            CreatedAt       = DateTimeOffset.UtcNow
        };

        foreach (var item in items)
            order._items.Add(OrderItem.Create(item.ProductId, item.Quantity, item.UnitPrice));

        order.RecalculateTotal();

        order.RaiseDomainEvent(new OrderPlaced(
            order.Id, order.CustomerId, order.Total, order.CreatedAt));

        return order;
    }

    public void Ship(string trackingNumber)
    {
        EnsureStatus(OrderStatus.Confirmed, "Can only ship confirmed orders");

        Status    = OrderStatus.Shipped;
        UpdatedAt = DateTimeOffset.UtcNow;

        RaiseDomainEvent(new OrderShipped(Id, trackingNumber, UpdatedAt.Value));
    }

    public void Deliver()
    {
        EnsureStatus(OrderStatus.Shipped, "Can only deliver shipped orders");

        Status    = OrderStatus.Delivered;
        UpdatedAt = DateTimeOffset.UtcNow;

        RaiseDomainEvent(new OrderDelivered(Id, UpdatedAt.Value));
    }

    public void Cancel(string reason)
    {
        if (Status is OrderStatus.Shipped or OrderStatus.Delivered)
            throw new DomainException($"Cannot cancel order in status {Status}");

        Status    = OrderStatus.Cancelled;
        UpdatedAt = DateTimeOffset.UtcNow;

        RaiseDomainEvent(new OrderCancelled(Id, CustomerId, reason, UpdatedAt.Value));
    }

    public void AddItem(ProductId productId, int quantity, Money unitPrice)
    {
        EnsureStatus(OrderStatus.Pending, "Can only modify pending orders");
        Guard.Against.NegativeOrZero(quantity, nameof(quantity));

        var existing = _items.Find(i => i.ProductId == productId);
        if (existing is not null)
            existing.IncreaseQuantity(quantity);
        else
            _items.Add(OrderItem.Create(productId, quantity, unitPrice));

        RecalculateTotal();
        UpdatedAt = DateTimeOffset.UtcNow;
    }

    private void RecalculateTotal()
    {
        var currency = _items.FirstOrDefault()?.UnitPrice.Currency ?? "USD";
        Total = _items.Aggregate(Money.Zero(currency),
            (sum, item) => sum + item.UnitPrice.Multiply(item.Quantity));
    }

    private void EnsureStatus(OrderStatus required, string message)
    {
        if (Status != required)
            throw new DomainException(message);
    }
}

// Strongly-typed IDs
public record OrderId(Guid Value)
{
    public static OrderId New() => new(Guid.NewGuid());
    public override string ToString() => Value.ToString();
}

public record CustomerId(Guid Value);
public record ProductId(Guid Value);
```

---

## Step 2136: Domain Services

```csharp
// Domain service: logic that spans multiple aggregates or requires external info
public interface IOrderPricingService
{
    Task<Money> CalculateTotalAsync(
        IReadOnlyList<OrderItemRequest> items,
        CustomerId customerId,
        CancellationToken ct = default);
}

public class OrderPricingService : IOrderPricingService
{
    private readonly IPricingRepository _pricing;
    private readonly ICustomerRepository _customers;

    public OrderPricingService(IPricingRepository pricing, ICustomerRepository customers)
    {
        _pricing   = pricing;
        _customers = customers;
    }

    public async Task<Money> CalculateTotalAsync(
        IReadOnlyList<OrderItemRequest> items,
        CustomerId customerId,
        CancellationToken ct = default)
    {
        var customer = await _customers.GetByIdAsync(customerId, ct)
                       ?? throw new DomainException($"Customer {customerId} not found");

        var discount = customer.Tier switch
        {
            CustomerTier.Bronze => 0.0m,
            CustomerTier.Silver => 0.05m,
            CustomerTier.Gold   => 0.10m,
            CustomerTier.Platinum => 0.15m,
            _ => 0.0m
        };

        var total = Money.Zero("USD");

        foreach (var item in items)
        {
            var price = await _pricing.GetCurrentPriceAsync(item.ProductId, ct);
            if (price is null)
                throw new DomainException($"No price found for product {item.ProductId}");

            total += price.Multiply(item.Quantity);
        }

        return total.Multiply(1m - discount);
    }
}

// Domain service for transfer between accounts
public class TransferService
{
    public void Transfer(BankAccount from, BankAccount to, Money amount)
    {
        // Spans two aggregates — belongs in domain service, not in either aggregate
        from.Debit(amount, $"Transfer to {to.Id}");
        to.Credit(amount, $"Transfer from {from.Id}");
    }
}
```

---

## Step 2137: Repository Pattern — Domain-Focused

```csharp
// Repository interface — in Domain layer
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(OrderId id, CancellationToken ct = default);
    Task<IReadOnlyList<Order>> GetByCustomerAsync(CustomerId customerId, CancellationToken ct = default);
    Task AddAsync(Order order, CancellationToken ct = default);
    Task UpdateAsync(Order order, CancellationToken ct = default);
    Task<bool> ExistsAsync(OrderId id, CancellationToken ct = default);
}

// EF Core implementation — in Infrastructure layer
public class EfCoreOrderRepository : IOrderRepository
{
    private readonly AppDbContext _db;

    public EfCoreOrderRepository(AppDbContext db) => _db = db;

    public async Task<Order?> GetByIdAsync(OrderId id, CancellationToken ct = default)
        => await _db.Orders
            .Include(o => o.Items)
            .FirstOrDefaultAsync(o => o.Id == id, ct);

    public async Task<IReadOnlyList<Order>> GetByCustomerAsync(CustomerId customerId, CancellationToken ct = default)
        => await _db.Orders
            .Include(o => o.Items)
            .Where(o => o.CustomerId == customerId)
            .OrderByDescending(o => o.CreatedAt)
            .ToListAsync(ct);

    public async Task AddAsync(Order order, CancellationToken ct = default)
        => await _db.Orders.AddAsync(order, ct);

    public Task UpdateAsync(Order order, CancellationToken ct = default)
    {
        _db.Orders.Update(order);
        return Task.CompletedTask;
    }

    public Task<bool> ExistsAsync(OrderId id, CancellationToken ct = default)
        => _db.Orders.AnyAsync(o => o.Id == id, ct);
}

// Unit of Work — coordinates multiple repositories
public interface IUnitOfWork : IDisposable
{
    IOrderRepository   Orders   { get; }
    ICustomerRepository Customers { get; }
    Task<int> SaveChangesAsync(CancellationToken ct = default);
}

public class EfCoreUnitOfWork : IUnitOfWork
{
    private readonly AppDbContext _db;

    public IOrderRepository    Orders    { get; }
    public ICustomerRepository Customers { get; }

    public EfCoreUnitOfWork(AppDbContext db)
    {
        _db       = db;
        Orders    = new EfCoreOrderRepository(db);
        Customers = new EfCoreCustomerRepository(db);
    }

    public Task<int> SaveChangesAsync(CancellationToken ct = default)
        => _db.SaveChangesAsync(ct);

    public void Dispose() => _db.Dispose();
}
```

---

## Step 2138: Domain Events & Integration Events

```csharp
// Domain events (in-process)
public interface IDomainEvent
{
    Guid          EventId    { get; }
    DateTimeOffset OccurredAt { get; }
}

public abstract record DomainEventBase : IDomainEvent
{
    public Guid           EventId    { get; } = Guid.NewGuid();
    public DateTimeOffset OccurredAt { get; } = DateTimeOffset.UtcNow;
}

public record OrderPlaced(
    OrderId    OrderId,
    CustomerId CustomerId,
    Money      Total,
    DateTimeOffset PlacedAt) : DomainEventBase;

public record OrderShipped(
    OrderId OrderId,
    string  TrackingNumber,
    DateTimeOffset ShippedAt) : DomainEventBase;

public record OrderCancelled(
    OrderId    OrderId,
    CustomerId CustomerId,
    string     Reason,
    DateTimeOffset CancelledAt) : DomainEventBase;

// Domain event dispatcher (MediatR)
public interface IDomainEventDispatcher
{
    Task DispatchAsync(IReadOnlyList<IDomainEvent> events, CancellationToken ct = default);
}

public class MediatRDomainEventDispatcher : IDomainEventDispatcher
{
    private readonly IPublisher _publisher;

    public MediatRDomainEventDispatcher(IPublisher publisher) => _publisher = publisher;

    public async Task DispatchAsync(IReadOnlyList<IDomainEvent> events, CancellationToken ct = default)
    {
        foreach (var @event in events)
            await _publisher.Publish(@event, ct);
    }
}

// Domain event handler — starts loyalty points accrual
public class OrderPlacedHandler : INotificationHandler<OrderPlaced>
{
    private readonly ILoyaltyService _loyalty;

    public OrderPlacedHandler(ILoyaltyService loyalty) => _loyalty = loyalty;

    public async Task Handle(OrderPlaced notification, CancellationToken ct)
        => await _loyalty.AccruePointsAsync(
            notification.CustomerId,
            notification.Total,
            notification.OrderId,
            ct);
}

// Integration event — crosses bounded context boundary (via message broker)
public record OrderPlacedIntegrationEvent(
    Guid    OrderId,
    Guid    CustomerId,
    decimal TotalAmount,
    string  Currency,
    DateTimeOffset OccurredAt);

// Domain event → Integration event mapping
public class OrderPlacedDomainEventToIntegrationEvent : INotificationHandler<OrderPlaced>
{
    private readonly IOutboxService _outbox;

    public OrderPlacedDomainEventToIntegrationEvent(IOutboxService outbox) => _outbox = outbox;

    public async Task Handle(OrderPlaced notification, CancellationToken ct)
    {
        // Publish via outbox for reliability
        await _outbox.AddMessageAsync(new OrderPlacedIntegrationEvent(
            OrderId:    notification.OrderId.Value,
            CustomerId: notification.CustomerId.Value,
            TotalAmount: notification.Total.Amount,
            Currency:   notification.Total.Currency,
            OccurredAt: notification.OccurredAt), ct);
    }
}
```

---

## Step 2139: Bounded Contexts & Anti-Corruption Layer

```csharp
// Bounded Context: Orders
// Bounded Context: Inventory
// They have different models for "Product"

// Orders BC: Product means something with price and identity
// Inventory BC: Product means something with stock level

// Anti-Corruption Layer (ACL): translates between BCs
public class InventoryAntiCorruptionLayer : IInventoryService
{
    private readonly IInventoryGrpcClient _grpcClient;

    public InventoryAntiCorruptionLayer(IInventoryGrpcClient grpcClient) => _grpcClient = grpcClient;

    // Translate Inventory BC concepts to Orders BC concepts
    public async Task<ProductAvailability> CheckAvailabilityAsync(
        ProductId productId, int quantity, CancellationToken ct = default)
    {
        // Inventory BC uses its own StockItemId type
        var response = await _grpcClient.CheckStockAsync(
            new CheckStockRequest
            {
                StockItemId = productId.Value.ToString(), // name differs
                RequiredQty = quantity
            }, ct);

        // Translate Inventory response to Orders BC model
        return new ProductAvailability(
            ProductId:   productId,
            IsAvailable: response.AvailableQty >= quantity,
            CurrentStock: response.AvailableQty,
            EstimatedDeliveryDays: response.LeadTimeDays);
    }
}

// Conformist: downstream BC conforms to upstream (no ACL needed)
// Customer-Supplier: upstream provides what downstream needs
// Partnership: both BCs evolve together
// Separate Ways: no integration
```

---

## Step 2140: Specification Pattern

```csharp
// Specifications encapsulate business rules as composable predicates
public abstract class Specification<T>
{
    public abstract Expression<Func<T, bool>> ToExpression();

    public bool IsSatisfiedBy(T entity) => ToExpression().Compile()(entity);

    public Specification<T> And(Specification<T> other) => new AndSpecification<T>(this, other);
    public Specification<T> Or(Specification<T>  other) => new OrSpecification<T>(this, other);
    public Specification<T> Not()                        => new NotSpecification<T>(this);
}

public class AndSpecification<T> : Specification<T>
{
    private readonly Specification<T> _left;
    private readonly Specification<T> _right;

    public AndSpecification(Specification<T> left, Specification<T> right)
    {
        _left  = left;
        _right = right;
    }

    public override Expression<Func<T, bool>> ToExpression()
    {
        var leftExpr  = _left.ToExpression();
        var rightExpr = _right.ToExpression();
        var param     = Expression.Parameter(typeof(T));
        var body      = Expression.AndAlso(
            Expression.Invoke(leftExpr,  param),
            Expression.Invoke(rightExpr, param));

        return Expression.Lambda<Func<T, bool>>(body, param);
    }
}

public class OrSpecification<T> : Specification<T>
{
    private readonly Specification<T> _left;
    private readonly Specification<T> _right;

    public OrSpecification(Specification<T> left, Specification<T> right)
    {
        _left = left; _right = right;
    }

    public override Expression<Func<T, bool>> ToExpression()
    {
        var leftExpr  = _left.ToExpression();
        var rightExpr = _right.ToExpression();
        var param     = Expression.Parameter(typeof(T));
        var body      = Expression.OrElse(
            Expression.Invoke(leftExpr, param),
            Expression.Invoke(rightExpr, param));
        return Expression.Lambda<Func<T, bool>>(body, param);
    }
}

public class NotSpecification<T> : Specification<T>
{
    private readonly Specification<T> _inner;
    public NotSpecification(Specification<T> inner) => _inner = inner;

    public override Expression<Func<T, bool>> ToExpression()
    {
        var expr  = _inner.ToExpression();
        var param = Expression.Parameter(typeof(T));
        var body  = Expression.Not(Expression.Invoke(expr, param));
        return Expression.Lambda<Func<T, bool>>(body, param);
    }
}

// Domain-specific specifications
public class PendingOrderSpecification : Specification<Order>
{
    public override Expression<Func<Order, bool>> ToExpression()
        => o => o.Status == OrderStatus.Pending;
}

public class HighValueOrderSpecification : Specification<Order>
{
    private readonly decimal _threshold;
    public HighValueOrderSpecification(decimal threshold) => _threshold = threshold;

    public override Expression<Func<Order, bool>> ToExpression()
        => o => o.Total.Amount >= _threshold;
}

public class RecentOrderSpecification : Specification<Order>
{
    private readonly int _days;
    public RecentOrderSpecification(int days) => _days = days;

    public override Expression<Func<Order, bool>> ToExpression()
    {
        var cutoff = DateTimeOffset.UtcNow.AddDays(-_days);
        return o => o.CreatedAt >= cutoff;
    }
}

// Usage
var spec = new PendingOrderSpecification()
    .And(new HighValueOrderSpecification(500m))
    .And(new RecentOrderSpecification(30));

var eligibleOrders = await _db.Orders
    .Where(spec.ToExpression())
    .ToListAsync(ct);
```

---

## Step 2141: Domain Exceptions & Guard Clauses

```csharp
// Domain exceptions (business rule violations)
public class DomainException : Exception
{
    public string Code { get; }

    public DomainException(string message, string? code = null)
        : base(message)
    {
        Code = code ?? "DOMAIN_ERROR";
    }
}

public class OrderNotFoundException : DomainException
{
    public OrderId OrderId { get; }

    public OrderNotFoundException(OrderId orderId)
        : base($"Order {orderId} was not found", "ORDER_NOT_FOUND")
    {
        OrderId = orderId;
    }
}

public class InsufficientStockException : DomainException
{
    public ProductId ProductId { get; }
    public int       Requested { get; }
    public int       Available { get; }

    public InsufficientStockException(ProductId productId, int requested, int available)
        : base($"Insufficient stock for product {productId}: requested {requested}, available {available}",
               "INSUFFICIENT_STOCK")
    {
        ProductId = productId;
        Requested = requested;
        Available = available;
    }
}

// Guard clauses (Ardalis.GuardClauses recommended)
public static class Guard
{
    public static class Against
    {
        public static T Null<T>(T? value, string paramName)
        {
            if (value is null)
                throw new ArgumentNullException(paramName);
            return value;
        }

        public static string NullOrWhiteSpace(string? value, string paramName)
        {
            if (string.IsNullOrWhiteSpace(value))
                throw new ArgumentException($"{paramName} cannot be null or whitespace", paramName);
            return value;
        }

        public static T NullOrEmpty<T>(IReadOnlyList<T>? value, string paramName)
        {
            if (value is null || value.Count == 0)
                throw new ArgumentException($"{paramName} cannot be null or empty", paramName);
            return default!;
        }

        public static int NegativeOrZero(int value, string paramName)
        {
            if (value <= 0)
                throw new ArgumentException($"{paramName} must be positive", paramName);
            return value;
        }

        public static decimal Negative(decimal value, string paramName)
        {
            if (value < 0)
                throw new ArgumentException($"{paramName} cannot be negative", paramName);
            return value;
        }
    }
}
```

---

## Step 2142: DDD with EF Core — Value Object Configuration

```csharp
// EF Core configuration for DDD aggregates
public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> b)
    {
        b.ToTable("orders");

        // Strongly-typed ID conversions
        b.HasKey(o => o.Id);
        b.Property(o => o.Id)
            .HasConversion(id => id.Value, v => new OrderId(v))
            .HasColumnName("id");

        b.Property(o => o.CustomerId)
            .HasConversion(id => id.Value, v => new CustomerId(v))
            .HasColumnName("customer_id");

        // Value Object: Money as owned entity
        b.OwnsOne(o => o.Total, money =>
        {
            money.Property(m => m.Amount).HasColumnName("total_amount").HasPrecision(18, 2);
            money.Property(m => m.Currency).HasColumnName("total_currency").HasMaxLength(3);
        });

        // Value Object: Address as owned entity
        b.OwnsOne(o => o.ShippingAddress, addr =>
        {
            addr.Property(a => a.Street).HasColumnName("shipping_street").HasMaxLength(200);
            addr.Property(a => a.City).HasColumnName("shipping_city").HasMaxLength(100);
            addr.Property(a => a.State).HasColumnName("shipping_state").HasMaxLength(100);
            addr.Property(a => a.PostalCode).HasColumnName("shipping_postal_code").HasMaxLength(20);
            addr.Property(a => a.Country).HasColumnName("shipping_country").HasMaxLength(2);
        });

        b.Property(o => o.Status)
            .HasConversion<string>()
            .HasMaxLength(20);

        b.HasMany(o => o.Items)
            .WithOne()
            .HasForeignKey("OrderId")
            .OnDelete(DeleteBehavior.Cascade);

        // Ignore domain events — not persisted
        b.Ignore(o => o.DomainEvents);
        b.Ignore(o => o.Version);
    }
}

public class OrderItemConfiguration : IEntityTypeConfiguration<OrderItem>
{
    public void Configure(EntityTypeBuilder<OrderItem> b)
    {
        b.ToTable("order_items");

        b.HasKey(i => i.Id);
        b.Property(i => i.Id)
            .HasConversion(id => id.Value, v => new OrderItemId(v));

        b.Property(i => i.ProductId)
            .HasConversion(id => id.Value, v => new ProductId(v));

        b.OwnsOne(i => i.UnitPrice, money =>
        {
            money.Property(m => m.Amount).HasColumnName("unit_price").HasPrecision(18, 2);
            money.Property(m => m.Currency).HasColumnName("currency").HasMaxLength(3);
        });
    }
}

// Dispatch domain events after SaveChanges
public class DomainEventDispatchingInterceptor : SaveChangesInterceptor
{
    private readonly IDomainEventDispatcher _dispatcher;

    public DomainEventDispatchingInterceptor(IDomainEventDispatcher dispatcher)
        => _dispatcher = dispatcher;

    public override async ValueTask<int> SavedChangesAsync(
        SaveChangesCompletedEventData eventData,
        int result,
        CancellationToken ct = default)
    {
        if (eventData.Context is null) return result;

        // After save, dispatch domain events
        var aggregates = eventData.Context.ChangeTracker.Entries<AggregateRoot<object>>()
            .Select(e => e.Entity)
            .Where(a => a.DomainEvents.Count > 0)
            .ToList();

        foreach (var aggregate in aggregates)
        {
            var events = aggregate.PopDomainEvents();
            await _dispatcher.DispatchAsync(events, ct);
        }

        return result;
    }
}
```

---

## Step 2143: Complete DDD Application Structure

```
Solution Structure (DDD + Clean Architecture):
src/
├── Domain/                          ← Pure domain logic, no dependencies
│   ├── Orders/
│   │   ├── Order.cs                 ← Aggregate root
│   │   ├── OrderItem.cs             ← Entity
│   │   ├── OrderStatus.cs           ← Enum
│   │   ├── Events/
│   │   │   ├── OrderPlaced.cs
│   │   │   └── OrderCancelled.cs
│   │   └── Specifications/
│   │       ├── PendingOrderSpec.cs
│   │       └── HighValueOrderSpec.cs
│   ├── Customers/
│   │   ├── Customer.cs
│   │   └── CustomerTier.cs
│   ├── Shared/
│   │   ├── ValueObjects/
│   │   │   ├── Money.cs
│   │   │   ├── Address.cs
│   │   │   └── Email.cs
│   │   ├── Entity.cs
│   │   ├── AggregateRoot.cs
│   │   ├── IDomainEvent.cs
│   │   └── DomainException.cs
│   └── Services/
│       └── OrderPricingService.cs
├── Application/                     ← Use cases, orchestration
│   ├── Orders/
│   │   ├── Commands/
│   │   │   ├── PlaceOrder/
│   │   │   │   ├── PlaceOrderCommand.cs
│   │   │   │   ├── PlaceOrderHandler.cs
│   │   │   │   └── PlaceOrderValidator.cs
│   │   │   └── CancelOrder/
│   │   └── Queries/
│   │       └── GetOrder/
│   ├── Common/
│   │   └── Behaviors/               ← MediatR pipeline
│   └── Contracts/
│       └── IOrderRepository.cs      ← Repository interfaces
├── Infrastructure/                  ← EF Core, Redis, external services
│   ├── Persistence/
│   │   ├── AppDbContext.cs
│   │   ├── Configurations/
│   │   │   └── OrderConfiguration.cs
│   │   └── Repositories/
│   │       └── EfCoreOrderRepository.cs
│   └── ExternalServices/
│       └── InventoryAntiCorruptionLayer.cs
└── Api/                             ← ASP.NET Core, minimal APIs
    ├── Endpoints/
    │   └── OrderEndpoints.cs
    └── Program.cs
```

```csharp
// DI registration — wiring the layers together
builder.Services.AddScoped<IOrderRepository, EfCoreOrderRepository>();
builder.Services.AddScoped<IUnitOfWork, EfCoreUnitOfWork>();
builder.Services.AddScoped<IOrderPricingService, OrderPricingService>();
builder.Services.AddScoped<IInventoryService, InventoryAntiCorruptionLayer>();
builder.Services.AddScoped<IDomainEventDispatcher, MediatRDomainEventDispatcher>();

builder.Services.AddMediatR(cfg =>
{
    cfg.RegisterServicesFromAssembly(typeof(PlaceOrderCommand).Assembly);
    cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
    cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
    cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(TransactionBehavior<,>));
});
```

**Summary**: Part 94 covers DDD tactical patterns (Value Objects with Money/Address/Email, strongly-typed IDs, Entity/AggregateRoot base classes, domain events, domain services), strategic patterns (bounded contexts, anti-corruption layer), Specification pattern with composable expressions, domain exceptions and guard clauses, EF Core configuration for aggregates and value objects (owned entities, type converters), domain event dispatching after SaveChanges, and complete DDD + Clean Architecture solution structure. Steps 2133–2148 complete.
