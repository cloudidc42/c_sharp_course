# Part 72: Domain-Driven Design Advanced — Aggregates, Bounded Contexts & Anti-Corruption Layer

## Steps 1781–1796 | World-Class DDD Architecture

---

## Step 1781: DDD Building Blocks — The Full Vocabulary

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    DDD Strategic Design                                  │
│                                                                          │
│  ┌─────────────────────┐      ┌─────────────────────┐                  │
│  │   Bounded Context:  │      │   Bounded Context:  │                  │
│  │   Order Management  │      │   Catalog           │                  │
│  │                     │      │                     │                  │
│  │  • Order Aggregate  │      │  • Product Entity   │                  │
│  │  • OrderItem VO     │─────▶│  • Price VO         │                  │
│  │  • OrderPlaced Evt  │      │  • Category Entity  │                  │
│  └─────────────────────┘      └─────────────────────┘                  │
│           │                             │                               │
│           ▼                             ▼                               │
│  ┌─────────────────────────────────────────────────────────────┐        │
│  │              Context Map (ACL, Shared Kernel, Partnership)  │        │
│  └─────────────────────────────────────────────────────────────┘        │
│                                                                          │
│  DDD Tactical Patterns:                                                  │
│  • Entity — has identity (Id), mutable state                            │
│  • Value Object — no identity, immutable, structural equality           │
│  • Aggregate — consistency boundary, root enforces invariants           │
│  • Domain Event — something that happened in the past tense             │
│  • Repository — collection abstraction over persistence                 │
│  • Domain Service — stateless operation spanning aggregates             │
│  • Factory — complex aggregate construction                             │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Step 1782: Value Objects — Correct Implementation

```csharp
// Domain/Common/ValueObject.cs
public abstract class ValueObject
{
    protected abstract IEnumerable<object?> GetEqualityComponents();

    public override bool Equals(object? obj)
    {
        if (obj is null || obj.GetType() != GetType()) return false;
        return GetEqualityComponents()
            .SequenceEqual(((ValueObject)obj).GetEqualityComponents());
    }

    public override int GetHashCode()
        => GetEqualityComponents()
            .Aggregate(0, (hash, component) =>
                HashCode.Combine(hash, component?.GetHashCode() ?? 0));

    public static bool operator ==(ValueObject? a, ValueObject? b)
    {
        if (a is null && b is null) return true;
        if (a is null || b is null) return false;
        return a.Equals(b);
    }

    public static bool operator !=(ValueObject? a, ValueObject? b) => !(a == b);
}

// Domain/Orders/Money.cs
public sealed class Money : ValueObject
{
    public decimal Amount { get; }
    public string Currency { get; }

    private Money(decimal amount, string currency)
    {
        Amount = amount;
        Currency = currency;
    }

    public static Money Of(decimal amount, string currency)
    {
        if (amount < 0) throw new DomainException("Money amount cannot be negative.");
        if (string.IsNullOrWhiteSpace(currency) || currency.Length != 3)
            throw new DomainException("Currency must be a 3-letter ISO code.");
        return new Money(amount, currency.ToUpperInvariant());
    }

    public static Money Zero(string currency) => Of(0, currency);

    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new DomainException($"Cannot add {Currency} and {other.Currency}.");
        return Of(Amount + other.Amount, Currency);
    }

    public Money Multiply(int quantity)
    {
        if (quantity < 0) throw new DomainException("Quantity cannot be negative.");
        return Of(Amount * quantity, Currency);
    }

    public Money Subtract(Money other)
    {
        if (Currency != other.Currency)
            throw new DomainException($"Cannot subtract {other.Currency} from {Currency}.");
        if (Amount < other.Amount)
            throw new DomainException("Cannot subtract: result would be negative.");
        return Of(Amount - other.Amount, Currency);
    }

    protected override IEnumerable<object?> GetEqualityComponents()
    {
        yield return Amount;
        yield return Currency;
    }

    public override string ToString() => $"{Amount:F2} {Currency}";
}

// Domain/Orders/Address.cs
public sealed class Address : ValueObject
{
    public string Street { get; }
    public string City { get; }
    public string PostalCode { get; }
    public string Country { get; }

    private Address(string street, string city, string postalCode, string country)
    {
        Street = street;
        City = city;
        PostalCode = postalCode;
        Country = country;
    }

    public static Address Create(
        string street, string city, string postalCode, string country)
    {
        if (string.IsNullOrWhiteSpace(street)) throw new DomainException("Street is required.");
        if (string.IsNullOrWhiteSpace(city)) throw new DomainException("City is required.");
        if (string.IsNullOrWhiteSpace(country)) throw new DomainException("Country is required.");
        return new Address(street.Trim(), city.Trim(), postalCode.Trim(), country.Trim());
    }

    protected override IEnumerable<object?> GetEqualityComponents()
    {
        yield return Street;
        yield return City;
        yield return PostalCode;
        yield return Country;
    }

    public override string ToString() => $"{Street}, {City} {PostalCode}, {Country}";
}
```

---

## Step 1783: Entity Base Class with Domain Events

```csharp
// Domain/Common/Entity.cs
public abstract class Entity<TId> : IEquatable<Entity<TId>>
    where TId : notnull
{
    public TId Id { get; protected init; }

    private readonly List<IDomainEvent> _domainEvents = [];
    public IReadOnlyCollection<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    protected Entity(TId id) => Id = id;

    protected void RaiseDomainEvent(IDomainEvent domainEvent)
        => _domainEvents.Add(domainEvent);

    public void ClearDomainEvents() => _domainEvents.Clear();

    public bool Equals(Entity<TId>? other)
    {
        if (other is null) return false;
        if (ReferenceEquals(this, other)) return true;
        if (GetType() != other.GetType()) return false;
        return Id.Equals(other.Id);
    }

    public override bool Equals(object? obj) => Equals(obj as Entity<TId>);

    public override int GetHashCode() => HashCode.Combine(GetType(), Id);

    public static bool operator ==(Entity<TId>? a, Entity<TId>? b)
        => a?.Equals(b) ?? b is null;

    public static bool operator !=(Entity<TId>? a, Entity<TId>? b) => !(a == b);
}

// Domain/Common/AggregateRoot.cs
public abstract class AggregateRoot<TId> : Entity<TId>
    where TId : notnull
{
    public int Version { get; private set; }

    protected AggregateRoot(TId id) : base(id) { }

    protected new void RaiseDomainEvent(IDomainEvent domainEvent)
    {
        base.RaiseDomainEvent(domainEvent);
        Version++;
    }
}

// Domain/Common/IDomainEvent.cs
public interface IDomainEvent : INotification // MediatR
{
    Guid EventId { get; }
    DateTime OccurredAt { get; }
}

public abstract record DomainEvent : IDomainEvent
{
    public Guid EventId { get; } = Guid.NewGuid();
    public DateTime OccurredAt { get; } = DateTime.UtcNow;
}
```

---

## Step 1784: The Order Aggregate — Full Implementation

```csharp
// Domain/Orders/OrderId.cs
public readonly record struct OrderId(Guid Value)
{
    public static OrderId New() => new(Guid.NewGuid());
    public static OrderId From(Guid value) => new(value);
    public override string ToString() => Value.ToString();
}

// Domain/Orders/Order.cs — The Aggregate Root
public sealed class Order : AggregateRoot<OrderId>
{
    private readonly List<OrderItem> _items = [];

    public Guid CustomerId { get; private set; }
    public Address ShippingAddress { get; private set; }
    public OrderStatus Status { get; private set; }
    public Money Total => _items
        .Aggregate(Money.Zero("USD"), (acc, item) => acc.Add(item.LineTotal));

    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();

    // EF Core needs a parameterless ctor — keep it private
    private Order() : base(OrderId.New()) { }

    private Order(OrderId id, Guid customerId, Address shippingAddress)
        : base(id)
    {
        CustomerId = customerId;
        ShippingAddress = shippingAddress;
        Status = OrderStatus.Draft;
    }

    // Factory method — the ONLY way to create an Order
    public static Order Create(Guid customerId, Address shippingAddress)
    {
        if (customerId == Guid.Empty)
            throw new DomainException("CustomerId is required.");

        var order = new Order(OrderId.New(), customerId, shippingAddress);
        order.RaiseDomainEvent(new OrderCreatedEvent(order.Id, customerId));
        return order;
    }

    public void AddItem(Guid productId, string productName, Money unitPrice, int quantity)
    {
        if (Status != OrderStatus.Draft)
            throw new DomainException("Cannot modify an order that is not in Draft status.");

        if (quantity <= 0)
            throw new DomainException("Quantity must be positive.");

        var existing = _items.FirstOrDefault(i => i.ProductId == productId);
        if (existing is not null)
        {
            existing.IncreaseQuantity(quantity);
            return;
        }

        _items.Add(OrderItem.Create(Id, productId, productName, unitPrice, quantity));
    }

    public void RemoveItem(Guid productId)
    {
        if (Status != OrderStatus.Draft)
            throw new DomainException("Cannot modify a non-Draft order.");

        var item = _items.FirstOrDefault(i => i.ProductId == productId)
            ?? throw new DomainException($"Item {productId} not found in order.");

        _items.Remove(item);
    }

    public void Submit()
    {
        if (Status != OrderStatus.Draft)
            throw new DomainException("Only Draft orders can be submitted.");

        if (!_items.Any())
            throw new DomainException("Cannot submit an empty order.");

        Status = OrderStatus.Submitted;
        RaiseDomainEvent(new OrderSubmittedEvent(Id, CustomerId, Total));
    }

    public void ConfirmPayment(string paymentTransactionId)
    {
        if (Status != OrderStatus.Submitted)
            throw new DomainException("Payment can only be confirmed for Submitted orders.");

        Status = OrderStatus.PaymentConfirmed;
        RaiseDomainEvent(new OrderPaymentConfirmedEvent(Id, paymentTransactionId));
    }

    public void StartFulfillment()
    {
        if (Status != OrderStatus.PaymentConfirmed)
            throw new DomainException("Fulfillment requires confirmed payment.");

        Status = OrderStatus.Fulfilling;
        RaiseDomainEvent(new OrderFulfillmentStartedEvent(Id));
    }

    public void Complete()
    {
        if (Status != OrderStatus.Fulfilling)
            throw new DomainException("Only Fulfilling orders can be completed.");

        Status = OrderStatus.Completed;
        RaiseDomainEvent(new OrderCompletedEvent(Id, CustomerId, Total));
    }

    public void Cancel(string reason)
    {
        if (Status is OrderStatus.Completed or OrderStatus.Cancelled)
            throw new DomainException($"Cannot cancel a {Status} order.");

        Status = OrderStatus.Cancelled;
        RaiseDomainEvent(new OrderCancelledEvent(Id, reason));
    }

    public void UpdateShippingAddress(Address newAddress)
    {
        if (Status != OrderStatus.Draft)
            throw new DomainException("Address can only be changed on Draft orders.");

        ShippingAddress = newAddress;
    }
}

// Domain/Orders/OrderItem.cs — NOT an Aggregate Root, part of Order's boundary
public sealed class OrderItem : Entity<Guid>
{
    public OrderId OrderId { get; }
    public Guid ProductId { get; }
    public string ProductName { get; }
    public Money UnitPrice { get; }
    public int Quantity { get; private set; }
    public Money LineTotal => UnitPrice.Multiply(Quantity);

    private OrderItem() : base(Guid.NewGuid()) { } // EF Core

    private OrderItem(OrderId orderId, Guid productId, string productName, Money unitPrice, int quantity)
        : base(Guid.NewGuid())
    {
        OrderId = orderId;
        ProductId = productId;
        ProductName = productName;
        UnitPrice = unitPrice;
        Quantity = quantity;
    }

    internal static OrderItem Create(
        OrderId orderId, Guid productId, string productName, Money unitPrice, int quantity)
        => new(orderId, productId, productName, unitPrice, quantity);

    internal void IncreaseQuantity(int by)
    {
        if (by <= 0) throw new DomainException("Must increase by at least 1.");
        Quantity += by;
    }
}

public enum OrderStatus
{
    Draft,
    Submitted,
    PaymentConfirmed,
    Fulfilling,
    Completed,
    Cancelled
}
```

---

## Step 1785: Domain Events

```csharp
// Domain/Orders/Events/
public sealed record OrderCreatedEvent(OrderId OrderId, Guid CustomerId) : DomainEvent;

public sealed record OrderSubmittedEvent(
    OrderId OrderId,
    Guid CustomerId,
    Money Total) : DomainEvent;

public sealed record OrderPaymentConfirmedEvent(
    OrderId OrderId,
    string PaymentTransactionId) : DomainEvent;

public sealed record OrderFulfillmentStartedEvent(OrderId OrderId) : DomainEvent;

public sealed record OrderCompletedEvent(
    OrderId OrderId,
    Guid CustomerId,
    Money Total) : DomainEvent;

public sealed record OrderCancelledEvent(OrderId OrderId, string Reason) : DomainEvent;

// Domain Event Handler — in a different bounded context
// Application/Orders/Handlers/SendOrderConfirmationEmailHandler.cs
public sealed class SendOrderConfirmationEmailHandler
    : INotificationHandler<OrderSubmittedEvent>
{
    private readonly IEmailService _emailService;
    private readonly ICustomerRepository _customers;
    private readonly ILogger<SendOrderConfirmationEmailHandler> _logger;

    public SendOrderConfirmationEmailHandler(
        IEmailService emailService,
        ICustomerRepository customers,
        ILogger<SendOrderConfirmationEmailHandler> logger)
    {
        _emailService = emailService;
        _customers = customers;
        _logger = logger;
    }

    public async Task Handle(OrderSubmittedEvent notification, CancellationToken ct)
    {
        _logger.LogInformation(
            "Sending confirmation email for Order {OrderId}", notification.OrderId);

        var customer = await _customers.FindByIdAsync(notification.CustomerId, ct);
        if (customer is null)
        {
            _logger.LogWarning("Customer {CustomerId} not found", notification.CustomerId);
            return;
        }

        await _emailService.SendOrderConfirmationAsync(
            customer.Email,
            notification.OrderId.Value,
            notification.Total.Amount,
            notification.Total.Currency,
            ct);
    }
}
```

---

## Step 1786: Publishing Domain Events via EF Core SaveChanges

```csharp
// Infrastructure/Data/DomainEventPublishingInterceptor.cs
public sealed class DomainEventPublishingInterceptor : SaveChangesInterceptor
{
    private readonly IPublisher _publisher; // MediatR

    public DomainEventPublishingInterceptor(IPublisher publisher)
        => _publisher = publisher;

    public override async ValueTask<int> SavedChangesAsync(
        SaveChangesCompletedEventData eventData,
        int result,
        CancellationToken ct)
    {
        await PublishDomainEventsAsync(eventData.Context, ct);
        return result;
    }

    private async Task PublishDomainEventsAsync(DbContext? context, CancellationToken ct)
    {
        if (context is null) return;

        var aggregates = context.ChangeTracker
            .Entries<AggregateRoot<OrderId>>()
            .Where(e => e.Entity.DomainEvents.Any())
            .Select(e => e.Entity)
            .ToList();

        var events = aggregates.SelectMany(a => a.DomainEvents).ToList();
        aggregates.ForEach(a => a.ClearDomainEvents());

        foreach (var domainEvent in events)
            await _publisher.Publish(domainEvent, ct);
    }
}

// Generic version using marker interface
public sealed class GenericDomainEventInterceptor : SaveChangesInterceptor
{
    private readonly IPublisher _publisher;

    public GenericDomainEventInterceptor(IPublisher publisher)
        => _publisher = publisher;

    public override async ValueTask<int> SavedChangesAsync(
        SaveChangesCompletedEventData eventData,
        int result,
        CancellationToken ct)
    {
        if (eventData.Context is null) return result;

        var domainEntities = eventData.Context.ChangeTracker
            .Entries<IHasDomainEvents>()
            .Where(e => e.Entity.DomainEvents.Any())
            .ToList();

        var allEvents = domainEntities
            .SelectMany(e => e.Entity.DomainEvents)
            .OrderBy(e => e.OccurredAt)
            .ToList();

        domainEntities.ForEach(e => e.Entity.ClearDomainEvents());

        foreach (var evt in allEvents)
            await _publisher.Publish(evt, ct);

        return result;
    }
}

public interface IHasDomainEvents
{
    IReadOnlyCollection<IDomainEvent> DomainEvents { get; }
    void ClearDomainEvents();
}
```

---

## Step 1787: Repository Pattern

```csharp
// Domain/Orders/IOrderRepository.cs
public interface IOrderRepository
{
    Task<Order?> FindByIdAsync(OrderId id, CancellationToken ct = default);
    Task<IReadOnlyList<Order>> GetByCustomerIdAsync(Guid customerId, CancellationToken ct = default);
    void Add(Order order);
    void Update(Order order);
    void Remove(Order order);
}

// Infrastructure/Data/Repositories/OrderRepository.cs
public sealed class OrderRepository : IOrderRepository
{
    private readonly AppDbContext _db;

    public OrderRepository(AppDbContext db) => _db = db;

    public async Task<Order?> FindByIdAsync(OrderId id, CancellationToken ct = default)
        => await _db.Orders
            .Include(o => o.Items)
            .FirstOrDefaultAsync(o => o.Id == id, ct);

    public async Task<IReadOnlyList<Order>> GetByCustomerIdAsync(
        Guid customerId,
        CancellationToken ct = default)
        => await _db.Orders
            .Include(o => o.Items)
            .Where(o => o.CustomerId == customerId)
            .OrderByDescending(o => o.Version)
            .ToListAsync(ct);

    public void Add(Order order) => _db.Orders.Add(order);
    public void Update(Order order) => _db.Orders.Update(order);
    public void Remove(Order order) => _db.Orders.Remove(order);
}

// Unit of Work pattern — coordinates multiple repositories
public interface IUnitOfWork
{
    IOrderRepository Orders { get; }
    ICustomerRepository Customers { get; }
    Task<int> CommitAsync(CancellationToken ct = default);
}

public sealed class EfCoreUnitOfWork : IUnitOfWork
{
    private readonly AppDbContext _db;

    public EfCoreUnitOfWork(AppDbContext db, IOrderRepository orders, ICustomerRepository customers)
    {
        _db = db;
        Orders = orders;
        Customers = customers;
    }

    public IOrderRepository Orders { get; }
    public ICustomerRepository Customers { get; }

    public Task<int> CommitAsync(CancellationToken ct = default)
        => _db.SaveChangesAsync(ct);
}
```

---

## Step 1788: Domain Service — Spanning Multiple Aggregates

```csharp
// Domain/Services/OrderPricingService.cs
// A domain service is STATELESS and operates on domain logic that doesn't belong to one aggregate
public interface IOrderPricingService
{
    Money CalculateDiscount(Order order, CustomerTier customerTier, PromoCode? promoCode);
}

public sealed class OrderPricingService : IOrderPricingService
{
    public Money CalculateDiscount(
        Order order,
        CustomerTier customerTier,
        PromoCode? promoCode)
    {
        var subtotal = order.Total;
        var discount = Money.Zero(subtotal.Currency);

        // Tier-based discount
        discount = discount.Add(customerTier switch
        {
            CustomerTier.Bronze => Money.Zero(subtotal.Currency),
            CustomerTier.Silver => subtotal.Multiply(5).Multiply(-1), // 5%? No — wrong
            // Correct percentage discount:
            CustomerTier.Silver => Money.Of(subtotal.Amount * 0.05m, subtotal.Currency),
            CustomerTier.Gold => Money.Of(subtotal.Amount * 0.10m, subtotal.Currency),
            CustomerTier.Platinum => Money.Of(subtotal.Amount * 0.15m, subtotal.Currency),
            _ => Money.Zero(subtotal.Currency)
        });

        // Promo code discount
        if (promoCode is not null && promoCode.IsValid())
        {
            var promoDiscount = promoCode.DiscountType switch
            {
                DiscountType.Percentage =>
                    Money.Of(subtotal.Amount * promoCode.Value / 100m, subtotal.Currency),
                DiscountType.Fixed =>
                    Money.Of(Math.Min(promoCode.Value, subtotal.Amount), subtotal.Currency),
                _ => Money.Zero(subtotal.Currency)
            };
            discount = discount.Add(promoDiscount);
        }

        // Cap discount at 30% of total
        var maxDiscount = Money.Of(subtotal.Amount * 0.30m, subtotal.Currency);
        return discount.Amount > maxDiscount.Amount ? maxDiscount : discount;
    }
}

// Domain/Services/InventoryReservationService.cs
public interface IInventoryReservationService
{
    Task<bool> ReserveItemsAsync(Order order, CancellationToken ct);
    Task ReleaseReservationAsync(OrderId orderId, CancellationToken ct);
}

public sealed class InventoryReservationService : IInventoryReservationService
{
    private readonly IInventoryRepository _inventory;

    public InventoryReservationService(IInventoryRepository inventory)
        => _inventory = inventory;

    public async Task<bool> ReserveItemsAsync(Order order, CancellationToken ct)
    {
        foreach (var item in order.Items)
        {
            var inventoryItem = await _inventory.FindByProductIdAsync(item.ProductId, ct);
            if (inventoryItem is null || !inventoryItem.CanReserve(item.Quantity))
                return false;
        }

        // All checks pass — reserve atomically
        foreach (var item in order.Items)
        {
            var inventoryItem = await _inventory.FindByProductIdAsync(item.ProductId, ct)!;
            inventoryItem!.Reserve(item.Quantity, order.Id);
        }

        return true;
    }

    public async Task ReleaseReservationAsync(OrderId orderId, CancellationToken ct)
    {
        var reservedItems = await _inventory.GetReservedByOrderAsync(orderId, ct);
        foreach (var item in reservedItems)
            item.ReleaseReservation(orderId);
    }
}
```

---

## Step 1789: Anti-Corruption Layer (ACL) Pattern

```csharp
// The ACL translates between YOUR domain model and EXTERNAL domain models
// Scenario: Integrating with legacy CRM that has a different "Customer" concept

// External CRM model (external system's data shape — DO NOT pollute your domain with this)
public sealed class CrmCustomerDto
{
    public string AccountId { get; set; } = string.Empty;    // "ACC-00123"
    public string FullName { get; set; } = string.Empty;
    public string EmailAddress { get; set; } = string.Empty;
    public string AccountType { get; set; } = string.Empty;  // "GOLD_TIER_MEMBER"
    public bool IsDeactivated { get; set; }
    public Dictionary<string, string> Attributes { get; set; } = new();
}

// Your domain model
public sealed class Customer : AggregateRoot<Guid>
{
    public string FullName { get; private set; }
    public string Email { get; private set; }
    public CustomerTier Tier { get; private set; }
    public bool IsActive { get; private set; }

    private Customer(Guid id, string fullName, string email, CustomerTier tier)
        : base(id)
    {
        FullName = fullName;
        Email = email;
        Tier = tier;
        IsActive = true;
    }

    public static Customer Create(string fullName, string email, CustomerTier tier)
        => new(Guid.NewGuid(), fullName, email, tier);
}

public enum CustomerTier { Bronze, Silver, Gold, Platinum }

// ACL: ICrmCustomerGateway — insulates YOUR domain from CRM changes
public interface ICrmCustomerGateway
{
    Task<Customer?> GetCustomerAsync(string crmAccountId, CancellationToken ct);
    Task<IReadOnlyList<Customer>> GetActiveCustomersAsync(CancellationToken ct);
}

public sealed class CrmCustomerGateway : ICrmCustomerGateway
{
    private readonly ICrmApiClient _crmClient; // raw HTTP client for external system
    private readonly ILogger<CrmCustomerGateway> _logger;

    public CrmCustomerGateway(ICrmApiClient crmClient, ILogger<CrmCustomerGateway> logger)
    {
        _crmClient = crmClient;
        _logger = logger;
    }

    public async Task<Customer?> GetCustomerAsync(string crmAccountId, CancellationToken ct)
    {
        CrmCustomerDto? dto;
        try
        {
            dto = await _crmClient.GetAccountAsync(crmAccountId, ct);
        }
        catch (CrmApiException ex)
        {
            _logger.LogWarning(ex, "CRM API error for account {AccountId}", crmAccountId);
            return null;
        }

        if (dto is null || dto.IsDeactivated) return null;

        return TranslateToDomain(dto);
    }

    public async Task<IReadOnlyList<Customer>> GetActiveCustomersAsync(CancellationToken ct)
    {
        var dtos = await _crmClient.GetAllActiveAccountsAsync(ct);
        return dtos.Select(TranslateToDomain).ToList().AsReadOnly();
    }

    // Translation lives HERE, not in the domain
    private static Customer TranslateToDomain(CrmCustomerDto dto)
    {
        var tier = TranslateTier(dto.AccountType);

        // Parse CRM's legacy "ACC-00123" format to our Guid-based ID
        // We use a deterministic hash to maintain stable IDs
        var customerId = GenerateDeterministicId(dto.AccountId);

        return Customer.Create(dto.FullName, dto.EmailAddress, tier);
    }

    private static CustomerTier TranslateTier(string crmAccountType) => crmAccountType switch
    {
        "BRONZE_MEMBER" or "NEW_CUSTOMER" => CustomerTier.Bronze,
        "SILVER_MEMBER" or "PREFERRED_CUSTOMER" => CustomerTier.Silver,
        "GOLD_TIER_MEMBER" or "VIP_CUSTOMER" => CustomerTier.Gold,
        "PLATINUM_ELITE" or "ENTERPRISE_ACCOUNT" => CustomerTier.Platinum,
        _ => CustomerTier.Bronze
    };

    private static Guid GenerateDeterministicId(string externalId)
    {
        var bytes = SHA256.HashData(Encoding.UTF8.GetBytes($"crm:{externalId}"));
        return new Guid(bytes[..16]);
    }
}
```

---

## Step 1790: Bounded Context — Catalog BC

```csharp
// Catalog Bounded Context has its OWN definition of "Product"
// This is NOT the same Product as what Order Management tracks

namespace Catalog.Domain;

public sealed class Product : AggregateRoot<Guid>
{
    public string Name { get; private set; }
    public string Description { get; private set; }
    public Money Price { get; private set; }
    public string Sku { get; private set; }
    public bool IsPublished { get; private set; }
    public IReadOnlyCollection<ProductImage> Images => _images.AsReadOnly();

    private readonly List<ProductImage> _images = [];
    private readonly List<ProductCategory> _categories = [];

    private Product(Guid id, string name, string description, Money price, string sku)
        : base(id)
    {
        Name = name;
        Description = description;
        Price = price;
        Sku = sku;
        IsPublished = false;
    }

    public static Product Create(string name, string description, Money price, string sku)
    {
        if (string.IsNullOrWhiteSpace(name)) throw new DomainException("Product name is required.");
        if (string.IsNullOrWhiteSpace(sku)) throw new DomainException("SKU is required.");

        var product = new Product(Guid.NewGuid(), name, description, price, sku);
        product.RaiseDomainEvent(new ProductCreatedEvent(product.Id, product.Name));
        return product;
    }

    public void UpdatePrice(Money newPrice)
    {
        var oldPrice = Price;
        Price = newPrice;
        RaiseDomainEvent(new ProductPriceChangedEvent(Id, oldPrice, newPrice));
    }

    public void Publish()
    {
        if (!_images.Any()) throw new DomainException("Cannot publish a product without images.");
        IsPublished = true;
        RaiseDomainEvent(new ProductPublishedEvent(Id));
    }

    public void AddImage(string url, string altText, bool isPrimary)
    {
        if (isPrimary) _images.ForEach(img => img.SetNonPrimary());
        _images.Add(new ProductImage(url, altText, isPrimary));
    }
}

// In Order Management BC, "Product" is just a simple read model:
namespace Orders.Domain;

public sealed record ProductSnapshot(
    Guid ProductId,
    string ProductName,
    Money UnitPrice);
// Order Management doesn't CARE about images, categories, publishing state
```

---

## Step 1791: Context Map — Integration Patterns

```csharp
// Shared Kernel: types shared between bounded contexts
// (minimize! — each change ripples to all contexts)
namespace SharedKernel;

public readonly record struct CustomerId(Guid Value)
{
    public static CustomerId New() => new(Guid.NewGuid());
}

public readonly record struct ProductId(Guid Value)
{
    public static ProductId New() => new(Guid.NewGuid());
}

// Integration Events — crossing context boundaries (unlike domain events)
// Published to a message bus (Kafka, RabbitMQ)
namespace SharedKernel.IntegrationEvents;

public sealed record ProductPriceUpdatedIntegrationEvent(
    Guid ProductId,
    decimal NewPrice,
    string Currency,
    DateTime OccurredAt);

public sealed record OrderPlacedIntegrationEvent(
    Guid OrderId,
    Guid CustomerId,
    decimal TotalAmount,
    string Currency,
    IReadOnlyList<OrderLineItem> Items,
    DateTime PlacedAt);

public sealed record OrderLineItem(
    Guid ProductId,
    string ProductName,
    int Quantity,
    decimal UnitPrice);

// Outbox pattern for reliable integration event publishing
public sealed class IntegrationEventOutbox
{
    private readonly AppDbContext _db;
    private readonly JsonSerializerOptions _jsonOptions = new(JsonSerializerDefaults.Web)
    {
        WriteIndented = false
    };

    public IntegrationEventOutbox(AppDbContext db) => _db = db;

    public void Schedule<TEvent>(TEvent integrationEvent) where TEvent : class
    {
        var outboxMessage = new OutboxMessage
        {
            Id = Guid.NewGuid(),
            Type = typeof(TEvent).AssemblyQualifiedName!,
            Payload = JsonSerializer.Serialize(integrationEvent, _jsonOptions),
            CreatedAt = DateTime.UtcNow,
            Status = OutboxMessageStatus.Pending
        };
        _db.OutboxMessages.Add(outboxMessage);
    }
}
```

---

## Step 1792: CQRS — Separating Read and Write Models

```csharp
// Command (Write) model — goes through the domain
public record PlaceOrderCommand(
    Guid CustomerId,
    string Street,
    string City,
    string PostalCode,
    string Country,
    IReadOnlyList<OrderLineRequest> Items);

public record OrderLineRequest(Guid ProductId, int Quantity);

public sealed class PlaceOrderCommandHandler : IRequestHandler<PlaceOrderCommand, OrderId>
{
    private readonly IUnitOfWork _uow;
    private readonly ICatalogService _catalog;
    private readonly IntegrationEventOutbox _outbox;

    public PlaceOrderCommandHandler(
        IUnitOfWork uow,
        ICatalogService catalog,
        IntegrationEventOutbox outbox)
    {
        _uow = uow;
        _catalog = catalog;
        _outbox = outbox;
    }

    public async Task<OrderId> Handle(PlaceOrderCommand cmd, CancellationToken ct)
    {
        var address = Address.Create(cmd.Street, cmd.City, cmd.PostalCode, cmd.Country);
        var order = Order.Create(cmd.CustomerId, address);

        foreach (var line in cmd.Items)
        {
            // ACL call — translate from Catalog's model to Order's model
            var product = await _catalog.GetProductSnapshotAsync(line.ProductId, ct)
                ?? throw new DomainException($"Product {line.ProductId} not found.");

            order.AddItem(product.ProductId, product.ProductName, product.UnitPrice, line.Quantity);
        }

        order.Submit();

        _uow.Orders.Add(order);

        // Schedule integration event (Outbox pattern — same transaction)
        _outbox.Schedule(new OrderPlacedIntegrationEvent(
            order.Id.Value,
            order.CustomerId,
            order.Total.Amount,
            order.Total.Currency,
            order.Items.Select(i => new OrderLineItem(
                i.ProductId, i.ProductName, i.Quantity, i.UnitPrice.Amount)).ToList(),
            DateTime.UtcNow));

        await _uow.CommitAsync(ct);

        return order.Id;
    }
}

// Query (Read) model — bypass domain, go directly to DB for performance
public record GetOrderSummaryQuery(Guid OrderId);

public sealed record OrderSummaryDto(
    Guid OrderId,
    string Status,
    decimal Total,
    string Currency,
    int ItemCount,
    DateTime SubmittedAt,
    string ShippingAddress);

public sealed class GetOrderSummaryQueryHandler
    : IRequestHandler<GetOrderSummaryQuery, OrderSummaryDto?>
{
    private readonly IDbConnection _db; // Dapper — bypass EF for reads

    public GetOrderSummaryQueryHandler(IDbConnection db) => _db = db;

    public async Task<OrderSummaryDto?> Handle(GetOrderSummaryQuery query, CancellationToken ct)
    {
        const string sql = """
            SELECT
                o."Id"         AS OrderId,
                o."Status"     AS Status,
                SUM(i."UnitPrice_Amount" * i."Quantity") AS Total,
                MAX(i."UnitPrice_Currency")              AS Currency,
                COUNT(i."Id")  AS ItemCount,
                o."CreatedAt"  AS SubmittedAt,
                CONCAT(o."ShippingAddress_Street", ', ', o."ShippingAddress_City") AS ShippingAddress
            FROM "Orders" o
            LEFT JOIN "OrderItems" i ON i."OrderId" = o."Id"
            WHERE o."Id" = @OrderId
            GROUP BY o."Id", o."Status", o."CreatedAt",
                     o."ShippingAddress_Street", o."ShippingAddress_City"
            """;

        return await _db.QueryFirstOrDefaultAsync<OrderSummaryDto>(
            sql, new { query.OrderId });
    }
}
```

---

## Step 1793: EF Core Configuration for DDD Aggregates

```csharp
// Infrastructure/Data/Configurations/OrderConfiguration.cs
public sealed class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.ToTable("Orders");

        // Strongly-typed ID — stored as Guid
        builder.HasKey(o => o.Id);
        builder.Property(o => o.Id)
            .HasConversion(
                id => id.Value,
                value => OrderId.From(value))
            .ValueGeneratedNever();

        builder.Property(o => o.CustomerId).IsRequired();

        // Value Object: Money — stored as owned entity (2 columns)
        // Note: Money.Total is computed, not stored
        // We store it per OrderItem

        // Value Object: Address — owned entity
        builder.OwnsOne(o => o.ShippingAddress, addr =>
        {
            addr.Property(a => a.Street).HasColumnName("ShippingAddress_Street").HasMaxLength(200);
            addr.Property(a => a.City).HasColumnName("ShippingAddress_City").HasMaxLength(100);
            addr.Property(a => a.PostalCode).HasColumnName("ShippingAddress_PostalCode").HasMaxLength(20);
            addr.Property(a => a.Country).HasColumnName("ShippingAddress_Country").HasMaxLength(100);
        });

        builder.Property(o => o.Status)
            .HasConversion<string>()
            .HasMaxLength(50);

        builder.Property(o => o.Version).IsConcurrencyToken();

        // Navigation: OrderItems are part of Order's aggregate boundary
        builder.HasMany(o => o.Items)
            .WithOne()
            .HasForeignKey(i => i.OrderId)
            .OnDelete(DeleteBehavior.Cascade);

        // Ignore computed property Total — it's calculated from Items
        builder.Ignore(o => o.Total);

        // Ignore domain events — not persisted
        builder.Ignore(o => o.DomainEvents);
    }
}

public sealed class OrderItemConfiguration : IEntityTypeConfiguration<OrderItem>
{
    public void Configure(EntityTypeBuilder<OrderItem> builder)
    {
        builder.ToTable("OrderItems");
        builder.HasKey(i => i.Id);

        builder.Property(i => i.OrderId)
            .HasConversion(id => id.Value, v => OrderId.From(v));

        builder.OwnsOne(i => i.UnitPrice, money =>
        {
            money.Property(m => m.Amount).HasColumnName("UnitPrice_Amount").HasPrecision(18, 4);
            money.Property(m => m.Currency).HasColumnName("UnitPrice_Currency").HasMaxLength(3);
        });

        builder.Ignore(i => i.LineTotal);
        builder.Property(i => i.ProductName).HasMaxLength(200);
    }
}
```

---

## Step 1794: Specification Pattern

```csharp
// Domain/Common/Specification.cs
public abstract class Specification<T>
{
    public abstract Expression<Func<T, bool>> ToExpression();

    public bool IsSatisfiedBy(T entity) => ToExpression().Compile()(entity);

    public Specification<T> And(Specification<T> other)
        => new AndSpecification<T>(this, other);

    public Specification<T> Or(Specification<T> other)
        => new OrSpecification<T>(this, other);

    public Specification<T> Not()
        => new NotSpecification<T>(this);
}

internal sealed class AndSpecification<T> : Specification<T>
{
    private readonly Specification<T> _left;
    private readonly Specification<T> _right;

    public AndSpecification(Specification<T> left, Specification<T> right)
    {
        _left = left;
        _right = right;
    }

    public override Expression<Func<T, bool>> ToExpression()
    {
        var leftExpr = _left.ToExpression();
        var rightExpr = _right.ToExpression();
        var param = Expression.Parameter(typeof(T));
        var body = Expression.AndAlso(
            Expression.Invoke(leftExpr, param),
            Expression.Invoke(rightExpr, param));
        return Expression.Lambda<Func<T, bool>>(body, param);
    }
}

internal sealed class OrSpecification<T> : Specification<T>
{
    private readonly Specification<T> _left;
    private readonly Specification<T> _right;

    public OrSpecification(Specification<T> left, Specification<T> right)
    {
        _left = left;
        _right = right;
    }

    public override Expression<Func<T, bool>> ToExpression()
    {
        var leftExpr = _left.ToExpression();
        var rightExpr = _right.ToExpression();
        var param = Expression.Parameter(typeof(T));
        var body = Expression.OrElse(
            Expression.Invoke(leftExpr, param),
            Expression.Invoke(rightExpr, param));
        return Expression.Lambda<Func<T, bool>>(body, param);
    }
}

internal sealed class NotSpecification<T> : Specification<T>
{
    private readonly Specification<T> _inner;
    public NotSpecification(Specification<T> inner) => _inner = inner;

    public override Expression<Func<T, bool>> ToExpression()
    {
        var inner = _inner.ToExpression();
        var param = Expression.Parameter(typeof(T));
        var body = Expression.Not(Expression.Invoke(inner, param));
        return Expression.Lambda<Func<T, bool>>(body, param);
    }
}

// Domain/Orders/Specifications/OrderSpecifications.cs
public sealed class ActiveOrdersSpec : Specification<Order>
{
    public override Expression<Func<Order, bool>> ToExpression()
        => o => o.Status != OrderStatus.Completed && o.Status != OrderStatus.Cancelled;
}

public sealed class OrdersByCustomerSpec : Specification<Order>
{
    private readonly Guid _customerId;
    public OrdersByCustomerSpec(Guid customerId) => _customerId = customerId;

    public override Expression<Func<Order, bool>> ToExpression()
        => o => o.CustomerId == _customerId;
}

public sealed class HighValueOrdersSpec : Specification<Order>
{
    private readonly decimal _minimumAmount;
    private readonly string _currency;

    public HighValueOrdersSpec(decimal minimumAmount, string currency)
    {
        _minimumAmount = minimumAmount;
        _currency = currency;
    }

    // Note: computed Total can't be used in EF expression — use separate DB query
    public override Expression<Func<Order, bool>> ToExpression()
        => _ => true; // simplified — real impl uses raw SQL for computed total
}

// Usage in repository
public async Task<IReadOnlyList<Order>> FindAsync(
    Specification<Order> spec,
    CancellationToken ct = default)
    => await _db.Orders
        .Include(o => o.Items)
        .Where(spec.ToExpression())
        .ToListAsync(ct);

// Composing specifications
var activeOrdersForCustomer = new ActiveOrdersSpec()
    .And(new OrdersByCustomerSpec(customerId));

var orders = await _orderRepo.FindAsync(activeOrdersForCustomer, ct);
```

---

## Step 1795: Domain Exception Hierarchy

```csharp
// Domain/Common/DomainException.cs
public class DomainException : Exception
{
    public string Code { get; }

    public DomainException(string message, string? code = null)
        : base(message)
    {
        Code = code ?? "DOMAIN_ERROR";
    }
}

public sealed class OrderNotFoundException : DomainException
{
    public OrderId OrderId { get; }

    public OrderNotFoundException(OrderId orderId)
        : base($"Order {orderId.Value} was not found.", "ORDER_NOT_FOUND")
    {
        OrderId = orderId;
    }
}

public sealed class PlanLimitExceededException : DomainException
{
    public PlanLimitExceededException(string message)
        : base(message, "PLAN_LIMIT_EXCEEDED") { }
}

public sealed class ConcurrencyException : DomainException
{
    public ConcurrencyException(string entityType, object entityId)
        : base($"{entityType} {entityId} was modified by another user. Please retry.",
               "CONCURRENCY_CONFLICT") { }
}

// Infrastructure: translate to HTTP status codes
public sealed class DomainExceptionFilter : IExceptionFilter
{
    public void OnException(ExceptionContext context)
    {
        if (context.Exception is not DomainException domainEx) return;

        var statusCode = domainEx switch
        {
            OrderNotFoundException => StatusCodes.Status404NotFound,
            PlanLimitExceededException => StatusCodes.Status402PaymentRequired,
            ConcurrencyException => StatusCodes.Status409Conflict,
            _ => StatusCodes.Status422UnprocessableEntity
        };

        context.Result = new ObjectResult(new
        {
            type = "DomainError",
            code = domainEx.Code,
            message = domainEx.Message
        })
        {
            StatusCode = statusCode
        };

        context.ExceptionHandled = true;
    }
}
```

---

## Step 1796: DDD Architecture Testing

```csharp
// Tests/Architecture/DddArchitectureTests.cs
// NetArchTest.Rules — enforce DDD layer boundaries in CI
public sealed class DddArchitectureTests
{
    private const string DomainNamespace = "MyApp.Domain";
    private const string ApplicationNamespace = "MyApp.Application";
    private const string InfrastructureNamespace = "MyApp.Infrastructure";
    private const string ApiNamespace = "MyApp.Api";

    private static readonly Types AllTypes =
        Types.InAssembly(typeof(Order).Assembly)
            .And()
            .InAssembly(typeof(PlaceOrderCommand).Assembly)
            .And()
            .InAssembly(typeof(AppDbContext).Assembly);

    [Fact]
    public void Domain_Should_Not_Depend_On_Application()
    {
        var result = Types.InAssembly(typeof(Order).Assembly)
            .Should()
            .NotHaveDependencyOn(ApplicationNamespace)
            .GetResult();

        Assert.True(result.IsSuccessful,
            $"Domain layer has forbidden dependencies: {string.Join(", ", result.FailingTypeNames ?? [])}");
    }

    [Fact]
    public void Domain_Should_Not_Depend_On_Infrastructure()
    {
        var result = Types.InAssembly(typeof(Order).Assembly)
            .Should()
            .NotHaveDependencyOn(InfrastructureNamespace)
            .GetResult();

        Assert.True(result.IsSuccessful,
            $"Domain layer depends on Infrastructure: {string.Join(", ", result.FailingTypeNames ?? [])}");
    }

    [Fact]
    public void Application_Should_Not_Depend_On_Infrastructure()
    {
        var result = Types.InAssembly(typeof(PlaceOrderCommand).Assembly)
            .Should()
            .NotHaveDependencyOn(InfrastructureNamespace)
            .GetResult();

        Assert.True(result.IsSuccessful);
    }

    [Fact]
    public void Aggregates_Should_Have_Private_Constructors()
    {
        var aggregateTypes = Types.InAssembly(typeof(Order).Assembly)
            .That()
            .Inherit(typeof(AggregateRoot<>))
            .GetTypes();

        var violations = aggregateTypes
            .Where(t => t.GetConstructors(BindingFlags.Public | BindingFlags.Instance).Length > 0)
            .Select(t => t.FullName!)
            .ToList();

        Assert.Empty(violations);
    }

    [Fact]
    public void ValueObjects_Should_Be_Sealed()
    {
        var valueObjectTypes = Types.InAssembly(typeof(Order).Assembly)
            .That()
            .Inherit(typeof(ValueObject))
            .GetTypes();

        var violations = valueObjectTypes
            .Where(t => !t.IsSealed)
            .Select(t => t.FullName!)
            .ToList();

        Assert.Empty(violations);
    }

    [Fact]
    public void CommandHandlers_Should_Be_In_Application_Layer()
    {
        var result = Types.InAssembly(typeof(PlaceOrderCommand).Assembly)
            .That()
            .ImplementInterface(typeof(IRequestHandler<,>))
            .Should()
            .ResideInNamespace(ApplicationNamespace)
            .GetResult();

        Assert.True(result.IsSuccessful);
    }

    [Fact]
    public void Repositories_Should_Only_Be_In_Infrastructure()
    {
        var result = Types.InAssembly(typeof(AppDbContext).Assembly)
            .That()
            .ImplementInterface(typeof(IOrderRepository))
            .Should()
            .ResideInNamespace(InfrastructureNamespace)
            .GetResult();

        Assert.True(result.IsSuccessful);
    }
}
```

---

## Summary: DDD Advanced Patterns

| Pattern | Purpose | Key Rule |
|---|---|---|
| Value Object | Immutable concept with structural equality | Always `sealed`, override `Equals`/`GetHashCode` |
| Aggregate | Consistency boundary | Only load what you need, save atomically |
| Domain Event | Something happened | Past tense, raised AFTER invariant check passes |
| Repository | Collection abstraction | One per aggregate root only |
| Domain Service | Cross-aggregate logic | Stateless, purely domain, no infrastructure |
| ACL | Context isolation | Translate at the boundary, not in the domain |
| Specification | Query encapsulation | Composable, reusable, testable |

**Core invariants**:
1. Aggregates enforce ALL business rules — application layer only coordinates
2. Domain events raised INSIDE the aggregate after state changes
3. Repositories return fully-loaded aggregates — no lazy loading
4. Cross-context communication via integration events (message bus), not direct DB queries
5. ACL prevents external models from polluting your domain

---

*Next: Part 73 — Advanced Testing Strategies: Contract Testing (Pact), Mutation Testing & Load Testing with k6*
