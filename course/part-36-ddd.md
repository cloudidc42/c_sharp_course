# Part 36: Domain-Driven Design (DDD)

## Steps 1011-1050 | ระดับโลก (World-Class)

---

## Step 1011: DDD คืออะไร และทำไมต้องใช้

Domain-Driven Design (DDD) คือแนวทางการออกแบบซอฟต์แวร์ที่ให้ความสำคัญกับ **domain** (โดเมน/ธุรกิจ) เป็นศูนย์กลาง

### แนวคิดหลัก

```
Business Domain
    ↓
Ubiquitous Language (ภาษากลางร่วมกัน)
    ↓
Domain Model (โมเดลธุรกิจ)
    ↓
Code (โค้ดที่สะท้อน Domain)
```

### ทำไมต้องใช้ DDD?

- **ลด complexity** ในระบบขนาดใหญ่
- **สื่อสาร** ระหว่าง developer และ domain expert ได้ดีขึ้น
- **โค้ดสะท้อน business** อย่างแท้จริง
- **แก้ไขและขยาย** ระบบได้ง่ายขึ้น
- **ลด technical debt** ในระยะยาว

---

## Step 1012: Ubiquitous Language

**Ubiquitous Language** คือภาษาร่วมที่ใช้ทั้ง business และ developer

### ตัวอย่าง: ระบบ E-Commerce

```
❌ ภาษา Technical:
- UserRecord, ProductRow, OrderEntity
- InsertOrder, UpdateProductQty, DeleteUser

✅ Ubiquitous Language:
- Customer, Product, Order
- PlaceOrder, FulfillOrder, CancelOrder
- ShoppingCart, Invoice, Shipment
```

### สร้าง Glossary

```csharp
// Domain/Glossary.md หรือ comments ใน code
// Order = คำสั่งซื้อที่ถูก confirmed แล้ว (ไม่ใช่ Cart)
// Customer = ผู้ใช้ที่ทำการซื้อแล้ว (ไม่ใช่ แค่ registered user)
// Fulfillment = กระบวนการจัดส่งสินค้าหลัง Order ถูก paid
// SKU = Stock Keeping Unit - รหัสสินค้าเฉพาะ

namespace ECommerceApp.Domain.Orders
{
    // Order ใช้ศัพท์จาก business
    public class Order { }
    public class OrderLine { }
    public class OrderStatus { }
}
```

---

## Step 1013: Strategic Design - Bounded Context

**Bounded Context** คือขอบเขตที่ชัดเจนของ domain model

```
┌─────────────────────────────────────────────────────┐
│                  E-Commerce System                   │
│                                                       │
│  ┌──────────────┐    ┌──────────────┐               │
│  │   Ordering   │    │  Inventory   │               │
│  │   Context    │    │   Context    │               │
│  │              │    │              │               │
│  │  Order       │    │  StockItem   │               │
│  │  Customer    │    │  Warehouse   │               │
│  │  OrderLine   │    │  Supplier    │               │
│  └──────┬───────┘    └──────┬───────┘               │
│         │                   │                        │
│         └─────────┬─────────┘                       │
│                   │ Domain Events                    │
│  ┌──────────────┐ │ ┌──────────────┐               │
│  │   Shipping   │ │ │   Payment    │               │
│  │   Context    │ │ │   Context    │               │
│  │              │ │ │              │               │
│  │  Shipment    │ │ │  Invoice     │               │
│  │  Address     │ │ │  Transaction │               │
│  │  Carrier     │ │ │  PaymentMethod│              │
│  └──────────────┘ │ └──────────────┘               │
└─────────────────────────────────────────────────────┘
```

### โครงสร้าง Project

```
src/
├── Ordering/
│   ├── Domain/
│   ├── Application/
│   ├── Infrastructure/
│   └── API/
├── Inventory/
│   ├── Domain/
│   ├── Application/
│   ├── Infrastructure/
│   └── API/
├── Shipping/
└── Payment/
```

---

## Step 1014: Tactical Design - Building Blocks

```
Building Blocks
├── Entities          - มี identity, mutable
├── Value Objects     - ไม่มี identity, immutable
├── Aggregates        - กลุ่มของ Entity/VO ที่มี root
├── Domain Events     - สิ่งที่เกิดขึ้นใน domain
├── Repositories      - เข้าถึง Aggregate
├── Domain Services   - logic ที่ไม่เป็นของ Entity ใดเลย
└── Factories         - สร้าง complex objects
```

---

## Step 1015: Entities

Entity คือ object ที่มี **identity** ชัดเจน และ identity ไม่เปลี่ยน แม้ attributes จะเปลี่ยน

```csharp
// Domain/Common/Entity.cs
public abstract class Entity<TId> where TId : notnull
{
    public TId Id { get; protected init; }

    protected Entity(TId id) => Id = id;

    public override bool Equals(object? obj)
    {
        if (obj is not Entity<TId> other) return false;
        if (ReferenceEquals(this, other)) return true;
        if (GetType() != other.GetType()) return false;
        return Id.Equals(other.Id);
    }

    public override int GetHashCode() => HashCode.Combine(GetType(), Id);

    public static bool operator ==(Entity<TId>? left, Entity<TId>? right)
        => left?.Equals(right) ?? right is null;

    public static bool operator !=(Entity<TId>? left, Entity<TId>? right)
        => !(left == right);
}
```

```csharp
// Domain/Orders/OrderLine.cs
public class OrderLine : Entity<OrderLineId>
{
    public ProductId ProductId { get; private set; }
    public string ProductName { get; private set; }
    public Money UnitPrice { get; private set; }
    public Quantity Quantity { get; private set; }
    public Money TotalPrice => UnitPrice * Quantity.Value;

    private OrderLine() { } // EF Core

    public static OrderLine Create(
        OrderLineId id,
        ProductId productId,
        string productName,
        Money unitPrice,
        Quantity quantity)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(productName);

        return new OrderLine
        {
            Id = id,
            ProductId = productId,
            ProductName = productName,
            UnitPrice = unitPrice,
            Quantity = quantity
        };
    }

    public void UpdateQuantity(Quantity newQuantity)
    {
        if (newQuantity.Value <= 0)
            throw new DomainException("Quantity must be positive");

        Quantity = newQuantity;
    }
}
```

---

## Step 1016: Value Objects

Value Object คือ object ที่ **ไม่มี identity** แต่มีค่า ความเท่ากันดูจากค่า ไม่ใช่ reference

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
            .Aggregate(0, HashCode.Combine);

    public static bool operator ==(ValueObject? left, ValueObject? right)
        => left?.Equals(right) ?? right is null;

    public static bool operator !=(ValueObject? left, ValueObject? right)
        => !(left == right);
}
```

```csharp
// Domain/Common/Money.cs
public sealed class Money : ValueObject
{
    public decimal Amount { get; }
    public string Currency { get; }

    private Money() { } // EF Core

    private Money(decimal amount, string currency)
    {
        if (amount < 0) throw new DomainException("Amount cannot be negative");
        if (string.IsNullOrWhiteSpace(currency)) throw new DomainException("Currency required");
        Amount = Math.Round(amount, 2);
        Currency = currency.ToUpperInvariant();
    }

    public static Money Of(decimal amount, string currency = "THB")
        => new(amount, currency);

    public static Money Zero(string currency = "THB") => new(0, currency);

    public Money Add(Money other)
    {
        EnsureSameCurrency(other);
        return new Money(Amount + other.Amount, Currency);
    }

    public Money Subtract(Money other)
    {
        EnsureSameCurrency(other);
        return new Money(Amount - other.Amount, Currency);
    }

    public Money Multiply(decimal factor) => new(Amount * factor, Currency);

    public static Money operator +(Money left, Money right) => left.Add(right);
    public static Money operator -(Money left, Money right) => left.Subtract(right);
    public static Money operator *(Money left, decimal right) => left.Multiply(right);

    private void EnsureSameCurrency(Money other)
    {
        if (Currency != other.Currency)
            throw new DomainException($"Cannot operate on different currencies: {Currency} and {other.Currency}");
    }

    protected override IEnumerable<object?> GetEqualityComponents()
    {
        yield return Amount;
        yield return Currency;
    }

    public override string ToString() => $"{Amount:N2} {Currency}";
}
```

```csharp
// Domain/Common/Address.cs
public sealed class Address : ValueObject
{
    public string Street { get; }
    public string City { get; }
    public string Province { get; }
    public string PostalCode { get; }
    public string Country { get; }

    private Address(string street, string city, string province, string postalCode, string country)
    {
        Street = street;
        City = city;
        Province = province;
        PostalCode = postalCode;
        Country = country;
    }

    public static Address Create(string street, string city, string province, string postalCode, string country = "Thailand")
    {
        if (string.IsNullOrWhiteSpace(street)) throw new DomainException("Street required");
        if (string.IsNullOrWhiteSpace(city)) throw new DomainException("City required");
        if (!IsValidPostalCode(postalCode)) throw new DomainException("Invalid postal code");

        return new Address(street, city, province, postalCode, country);
    }

    private static bool IsValidPostalCode(string code) =>
        code?.Length == 5 && code.All(char.IsDigit);

    protected override IEnumerable<object?> GetEqualityComponents()
    {
        yield return Street;
        yield return City;
        yield return Province;
        yield return PostalCode;
        yield return Country;
    }

    public override string ToString() => $"{Street}, {City}, {Province} {PostalCode}, {Country}";
}
```

```csharp
// Domain/Common/Quantity.cs
public sealed class Quantity : ValueObject
{
    public int Value { get; }

    private Quantity(int value)
    {
        if (value <= 0) throw new DomainException("Quantity must be positive");
        Value = value;
    }

    public static Quantity Of(int value) => new(value);

    public static implicit operator int(Quantity q) => q.Value;
    public static implicit operator Quantity(int value) => new(value);

    protected override IEnumerable<object?> GetEqualityComponents()
    {
        yield return Value;
    }
}
```

---

## Step 1017: Strongly-Typed IDs

```csharp
// Domain/Common/StronglyTypedId.cs
public abstract class StronglyTypedId<T> : ValueObject where T : notnull
{
    public T Value { get; }

    protected StronglyTypedId(T value)
    {
        if (value is string s && string.IsNullOrEmpty(s))
            throw new DomainException($"Id cannot be empty");
        Value = value;
    }

    public override string ToString() => Value.ToString()!;
}

// Domain/Orders/OrderId.cs
public sealed class OrderId : StronglyTypedId<Guid>
{
    public OrderId(Guid value) : base(value) { }
    public static OrderId New() => new(Guid.NewGuid());
    public static OrderId From(Guid value) => new(value);

    protected override IEnumerable<object?> GetEqualityComponents()
    {
        yield return Value;
    }
}

// Domain/Orders/OrderLineId.cs
public sealed class OrderLineId : StronglyTypedId<Guid>
{
    public OrderLineId(Guid value) : base(value) { }
    public static OrderLineId New() => new(Guid.NewGuid());

    protected override IEnumerable<object?> GetEqualityComponents()
    {
        yield return Value;
    }
}

// Domain/Customers/CustomerId.cs
public sealed class CustomerId : StronglyTypedId<Guid>
{
    public CustomerId(Guid value) : base(value) { }
    public static CustomerId New() => new(Guid.NewGuid());

    protected override IEnumerable<object?> GetEqualityComponents()
    {
        yield return Value;
    }
}
```

---

## Step 1018: Aggregate Root

**Aggregate** คือกลุ่มของ Entities และ Value Objects ที่มี **Aggregate Root** เป็น entry point เดียว

```csharp
// Domain/Common/AggregateRoot.cs
public abstract class AggregateRoot<TId> : Entity<TId> where TId : notnull
{
    private readonly List<IDomainEvent> _domainEvents = [];

    protected AggregateRoot(TId id) : base(id) { }

    public IReadOnlyList<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    protected void RaiseDomainEvent(IDomainEvent domainEvent)
        => _domainEvents.Add(domainEvent);

    public void ClearDomainEvents() => _domainEvents.Clear();
}
```

```csharp
// Domain/Orders/Order.cs
public class Order : AggregateRoot<OrderId>
{
    private readonly List<OrderLine> _lines = [];

    public CustomerId CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    public Address ShippingAddress { get; private set; }
    public Money SubTotal => _lines.Aggregate(Money.Zero(), (sum, l) => sum + l.TotalPrice);
    public Money ShippingFee { get; private set; }
    public Money TotalAmount => SubTotal + ShippingFee;
    public DateTime PlacedAt { get; private set; }
    public DateTime? ConfirmedAt { get; private set; }
    public DateTime? ShippedAt { get; private set; }
    public IReadOnlyList<OrderLine> Lines => _lines.AsReadOnly();

    private Order() : base(default!) { } // EF Core

    private Order(
        OrderId id,
        CustomerId customerId,
        Address shippingAddress,
        Money shippingFee) : base(id)
    {
        CustomerId = customerId;
        ShippingAddress = shippingAddress;
        ShippingFee = shippingFee;
        Status = OrderStatus.Draft;
        PlacedAt = DateTime.UtcNow;
    }

    public static Order Create(
        CustomerId customerId,
        Address shippingAddress,
        Money? shippingFee = null)
    {
        var order = new Order(
            OrderId.New(),
            customerId,
            shippingAddress,
            shippingFee ?? Money.Of(50, "THB"));

        order.RaiseDomainEvent(new OrderCreatedEvent(order.Id, order.CustomerId));
        return order;
    }

    public void AddLine(ProductId productId, string productName, Money unitPrice, Quantity quantity)
    {
        if (Status != OrderStatus.Draft)
            throw new DomainException($"Cannot add line to {Status} order");

        var existingLine = _lines.FirstOrDefault(l => l.ProductId == productId);
        if (existingLine is not null)
        {
            existingLine.UpdateQuantity(Quantity.Of(existingLine.Quantity.Value + quantity.Value));
            return;
        }

        var line = OrderLine.Create(OrderLineId.New(), productId, productName, unitPrice, quantity);
        _lines.Add(line);
    }

    public void RemoveLine(OrderLineId lineId)
    {
        if (Status != OrderStatus.Draft)
            throw new DomainException($"Cannot remove line from {Status} order");

        var line = _lines.Find(l => l.Id == lineId)
            ?? throw new DomainException($"Order line {lineId} not found");

        _lines.Remove(line);
    }

    public void Confirm()
    {
        if (Status != OrderStatus.Draft)
            throw new DomainException($"Cannot confirm {Status} order");

        if (_lines.Count == 0)
            throw new DomainException("Cannot confirm empty order");

        Status = OrderStatus.Confirmed;
        ConfirmedAt = DateTime.UtcNow;

        RaiseDomainEvent(new OrderConfirmedEvent(Id, CustomerId, TotalAmount));
    }

    public void Ship(DateTime? shippedAt = null)
    {
        if (Status != OrderStatus.Confirmed)
            throw new DomainException($"Cannot ship {Status} order");

        Status = OrderStatus.Shipped;
        ShippedAt = shippedAt ?? DateTime.UtcNow;

        RaiseDomainEvent(new OrderShippedEvent(Id, ShippedAt.Value));
    }

    public void Cancel(string reason)
    {
        if (Status is OrderStatus.Shipped or OrderStatus.Delivered or OrderStatus.Cancelled)
            throw new DomainException($"Cannot cancel {Status} order");

        Status = OrderStatus.Cancelled;

        RaiseDomainEvent(new OrderCancelledEvent(Id, reason));
    }
}
```

```csharp
// Domain/Orders/OrderStatus.cs
public enum OrderStatus
{
    Draft,
    Confirmed,
    Paid,
    Processing,
    Shipped,
    Delivered,
    Cancelled,
    Refunded
}
```

---

## Step 1019: Domain Events

Domain Events คือ events ที่แทนสิ่งที่เกิดขึ้นใน domain

```csharp
// Domain/Common/IDomainEvent.cs
public interface IDomainEvent : MediatR.INotification
{
    Guid EventId { get; }
    DateTime OccurredAt { get; }
}

// Domain/Common/DomainEvent.cs
public abstract record DomainEvent : IDomainEvent
{
    public Guid EventId { get; } = Guid.NewGuid();
    public DateTime OccurredAt { get; } = DateTime.UtcNow;
}
```

```csharp
// Domain/Orders/Events/OrderCreatedEvent.cs
public record OrderCreatedEvent(OrderId OrderId, CustomerId CustomerId) : DomainEvent;

// Domain/Orders/Events/OrderConfirmedEvent.cs
public record OrderConfirmedEvent(
    OrderId OrderId,
    CustomerId CustomerId,
    Money TotalAmount) : DomainEvent;

// Domain/Orders/Events/OrderShippedEvent.cs
public record OrderShippedEvent(OrderId OrderId, DateTime ShippedAt) : DomainEvent;

// Domain/Orders/Events/OrderCancelledEvent.cs
public record OrderCancelledEvent(OrderId OrderId, string Reason) : DomainEvent;
```

---

## Step 1020: Domain Exceptions

```csharp
// Domain/Common/DomainException.cs
public class DomainException : Exception
{
    public DomainException(string message) : base(message) { }
    public DomainException(string message, Exception innerException) : base(message, innerException) { }
}

// Domain/Common/NotFoundException.cs
public class NotFoundException : DomainException
{
    public NotFoundException(string entityName, object id)
        : base($"{entityName} with id '{id}' was not found") { }
}

// Domain/Common/ConflictException.cs
public class ConflictException : DomainException
{
    public ConflictException(string message) : base(message) { }
}
```

---

## Step 1021: Repositories

Repository pattern ใน DDD ทำหน้าที่เป็น **collection** ของ Aggregate Roots

```csharp
// Domain/Orders/IOrderRepository.cs
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(OrderId id, CancellationToken ct = default);
    Task<IReadOnlyList<Order>> GetByCustomerAsync(CustomerId customerId, CancellationToken ct = default);
    Task<bool> ExistsAsync(OrderId id, CancellationToken ct = default);
    Task AddAsync(Order order, CancellationToken ct = default);
    Task UpdateAsync(Order order, CancellationToken ct = default);
    Task DeleteAsync(OrderId id, CancellationToken ct = default);
}

// Domain/Customers/ICustomerRepository.cs
public interface ICustomerRepository
{
    Task<Customer?> GetByIdAsync(CustomerId id, CancellationToken ct = default);
    Task<Customer?> GetByEmailAsync(Email email, CancellationToken ct = default);
    Task AddAsync(Customer customer, CancellationToken ct = default);
    Task UpdateAsync(Customer customer, CancellationToken ct = default);
}
```

---

## Step 1022: Customer Aggregate

```csharp
// Domain/Customers/Customer.cs
public class Customer : AggregateRoot<CustomerId>
{
    private readonly List<Address> _addresses = [];

    public PersonName Name { get; private set; }
    public Email Email { get; private set; }
    public PhoneNumber? Phone { get; private set; }
    public CustomerTier Tier { get; private set; }
    public bool IsEmailVerified { get; private set; }
    public DateTime CreatedAt { get; private init; }
    public IReadOnlyList<Address> Addresses => _addresses.AsReadOnly();

    private Customer() : base(default!) { }

    private Customer(CustomerId id, PersonName name, Email email) : base(id)
    {
        Name = name;
        Email = email;
        Tier = CustomerTier.Regular;
        CreatedAt = DateTime.UtcNow;
    }

    public static Customer Register(string firstName, string lastName, string email)
    {
        var customer = new Customer(
            CustomerId.New(),
            PersonName.Create(firstName, lastName),
            Email.Create(email));

        customer.RaiseDomainEvent(new CustomerRegisteredEvent(customer.Id, customer.Email));
        return customer;
    }

    public void VerifyEmail()
    {
        if (IsEmailVerified) return;
        IsEmailVerified = true;
        RaiseDomainEvent(new CustomerEmailVerifiedEvent(Id));
    }

    public void UpdatePhone(string phoneNumber)
        => Phone = PhoneNumber.Create(phoneNumber);

    public void AddAddress(Address address)
    {
        if (_addresses.Count >= 5)
            throw new DomainException("Cannot have more than 5 addresses");
        _addresses.Add(address);
    }

    public void UpgradeTier(CustomerTier newTier)
    {
        if (newTier <= Tier)
            throw new DomainException($"Cannot downgrade from {Tier} to {newTier}");
        Tier = newTier;
        RaiseDomainEvent(new CustomerTierUpgradedEvent(Id, newTier));
    }
}
```

```csharp
// Domain/Customers/PersonName.cs
public sealed class PersonName : ValueObject
{
    public string FirstName { get; }
    public string LastName { get; }
    public string FullName => $"{FirstName} {LastName}";

    private PersonName(string firstName, string lastName)
    {
        FirstName = firstName;
        LastName = lastName;
    }

    public static PersonName Create(string firstName, string lastName)
    {
        if (string.IsNullOrWhiteSpace(firstName)) throw new DomainException("First name required");
        if (string.IsNullOrWhiteSpace(lastName)) throw new DomainException("Last name required");
        return new PersonName(firstName.Trim(), lastName.Trim());
    }

    protected override IEnumerable<object?> GetEqualityComponents()
    {
        yield return FirstName;
        yield return LastName;
    }
}

// Domain/Customers/Email.cs
public sealed class Email : ValueObject
{
    public string Value { get; }

    private Email(string value) => Value = value;

    public static Email Create(string email)
    {
        if (string.IsNullOrWhiteSpace(email)) throw new DomainException("Email required");
        email = email.Trim().ToLowerInvariant();
        if (!email.Contains('@')) throw new DomainException("Invalid email format");
        return new Email(email);
    }

    protected override IEnumerable<object?> GetEqualityComponents()
    {
        yield return Value;
    }

    public override string ToString() => Value;
    public static implicit operator string(Email email) => email.Value;
}

// Domain/Customers/PhoneNumber.cs
public sealed class PhoneNumber : ValueObject
{
    public string Value { get; }

    private PhoneNumber(string value) => Value = value;

    public static PhoneNumber Create(string phone)
    {
        var digits = new string(phone.Where(char.IsDigit).ToArray());
        if (digits.Length < 9 || digits.Length > 15)
            throw new DomainException("Invalid phone number");
        return new PhoneNumber(digits);
    }

    protected override IEnumerable<object?> GetEqualityComponents()
    {
        yield return Value;
    }
}

public enum CustomerTier { Regular = 1, Silver = 2, Gold = 3, Platinum = 4 }
```

---

## Step 1023: Domain Services

Domain Service ใช้เมื่อ logic ไม่เป็นของ Entity ใดเลย หรือต้องการ Entity หลายตัว

```csharp
// Domain/Orders/Services/OrderPricingService.cs
public class OrderPricingService(IDiscountRepository discountRepository)
{
    public async Task<Money> CalculateDiscountAsync(Order order, Customer customer)
    {
        var discounts = await discountRepository.GetActiveDiscountsAsync();
        var totalDiscount = Money.Zero();

        // Tier discount
        var tierDiscount = customer.Tier switch
        {
            CustomerTier.Silver => order.SubTotal.Multiply(0.05m),
            CustomerTier.Gold => order.SubTotal.Multiply(0.10m),
            CustomerTier.Platinum => order.SubTotal.Multiply(0.15m),
            _ => Money.Zero()
        };
        totalDiscount += tierDiscount;

        // Volume discount
        if (order.SubTotal.Amount >= 5000)
            totalDiscount += order.SubTotal.Multiply(0.03m);

        // Apply active promo codes
        foreach (var discount in discounts)
        {
            if (discount.IsApplicableTo(order))
                totalDiscount += discount.CalculateDiscount(order.SubTotal);
        }

        // Max discount = 30% of subtotal
        var maxDiscount = order.SubTotal.Multiply(0.30m);
        return totalDiscount.Amount > maxDiscount.Amount ? maxDiscount : totalDiscount;
    }
}
```

```csharp
// Domain/Inventory/Services/StockReservationService.cs
public class StockReservationService(IProductRepository productRepository)
{
    public async Task<ReservationResult> TryReserveStockAsync(
        IReadOnlyList<(ProductId ProductId, Quantity Quantity)> items,
        CancellationToken ct = default)
    {
        var failures = new List<StockReservationFailure>();

        foreach (var (productId, requestedQty) in items)
        {
            var product = await productRepository.GetByIdAsync(productId, ct);
            if (product is null)
            {
                failures.Add(new StockReservationFailure(productId, "Product not found"));
                continue;
            }

            if (!product.HasSufficientStock(requestedQty))
            {
                failures.Add(new StockReservationFailure(productId,
                    $"Insufficient stock. Available: {product.StockQuantity.Value}"));
            }
        }

        if (failures.Count > 0)
            return ReservationResult.Failed(failures);

        // Reserve all
        foreach (var (productId, qty) in items)
        {
            var product = await productRepository.GetByIdAsync(productId, ct);
            product!.ReserveStock(qty);
            await productRepository.UpdateAsync(product, ct);
        }

        return ReservationResult.Success();
    }
}

public record ReservationResult(bool IsSuccess, IReadOnlyList<StockReservationFailure> Failures)
{
    public static ReservationResult Success() => new(true, []);
    public static ReservationResult Failed(IReadOnlyList<StockReservationFailure> failures) => new(false, failures);
}

public record StockReservationFailure(ProductId ProductId, string Reason);
```

---

## Step 1024: Infrastructure - EF Core Configuration

```csharp
// Infrastructure/Persistence/AppDbContext.cs
public class AppDbContext(DbContextOptions<AppDbContext> options, IPublisher publisher)
    : DbContext(options)
{
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<OrderLine> OrderLines => Set<OrderLine>();
    public DbSet<Customer> Customers => Set<Customer>();
    public DbSet<Product> Products => Set<Product>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
        base.OnModelCreating(modelBuilder);
    }

    public override async Task<int> SaveChangesAsync(CancellationToken ct = default)
    {
        var result = await base.SaveChangesAsync(ct);
        await PublishDomainEventsAsync(ct);
        return result;
    }

    private async Task PublishDomainEventsAsync(CancellationToken ct)
    {
        var aggregates = ChangeTracker
            .Entries<AggregateRoot<OrderId>>()
            .Select(e => e.Entity)
            .Where(a => a.DomainEvents.Any())
            .ToList();

        var events = aggregates.SelectMany(a => a.DomainEvents).ToList();

        foreach (var aggregate in aggregates)
            aggregate.ClearDomainEvents();

        foreach (var domainEvent in events)
            await publisher.Publish(domainEvent, ct);
    }
}
```

```csharp
// Infrastructure/Persistence/Configurations/OrderConfiguration.cs
public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.ToTable("Orders");

        builder.HasKey(o => o.Id);
        builder.Property(o => o.Id)
            .HasConversion(id => id.Value, value => OrderId.From(value))
            .ValueGeneratedNever();

        builder.Property(o => o.CustomerId)
            .HasConversion(id => id.Value, value => new CustomerId(value));

        builder.Property(o => o.Status)
            .HasConversion<string>()
            .HasMaxLength(20);

        // Value Object: Money (owned entity)
        builder.OwnsOne(o => o.ShippingFee, money =>
        {
            money.Property(m => m.Amount).HasColumnName("ShippingFeeAmount").HasPrecision(18, 2);
            money.Property(m => m.Currency).HasColumnName("ShippingFeeCurrency").HasMaxLength(3);
        });

        // Value Object: Address (owned entity)
        builder.OwnsOne(o => o.ShippingAddress, addr =>
        {
            addr.Property(a => a.Street).HasMaxLength(200);
            addr.Property(a => a.City).HasMaxLength(100);
            addr.Property(a => a.Province).HasMaxLength(100);
            addr.Property(a => a.PostalCode).HasMaxLength(10);
            addr.Property(a => a.Country).HasMaxLength(100);
        });

        // Navigation
        builder.HasMany(o => o.Lines)
            .WithOne()
            .HasForeignKey("OrderId")
            .OnDelete(DeleteBehavior.Cascade);

        builder.Navigation(o => o.Lines).UsePropertyAccessMode(PropertyAccessMode.Field);
    }
}
```

```csharp
// Infrastructure/Persistence/Configurations/CustomerConfiguration.cs
public class CustomerConfiguration : IEntityTypeConfiguration<Customer>
{
    public void Configure(EntityTypeBuilder<Customer> builder)
    {
        builder.ToTable("Customers");

        builder.HasKey(c => c.Id);
        builder.Property(c => c.Id)
            .HasConversion(id => id.Value, value => new CustomerId(value))
            .ValueGeneratedNever();

        // Value Object: PersonName (owned)
        builder.OwnsOne(c => c.Name, name =>
        {
            name.Property(n => n.FirstName).HasColumnName("FirstName").HasMaxLength(100);
            name.Property(n => n.LastName).HasColumnName("LastName").HasMaxLength(100);
        });

        // Value Object: Email (owned)
        builder.OwnsOne(c => c.Email, email =>
        {
            email.Property(e => e.Value).HasColumnName("Email").HasMaxLength(256);
            email.HasIndex(e => e.Value).IsUnique();
        });

        // Value Object: PhoneNumber (owned, nullable)
        builder.OwnsOne(c => c.Phone, phone =>
        {
            phone.Property(p => p.Value).HasColumnName("Phone").HasMaxLength(20);
        });

        // Collection of Value Objects: Addresses (owned collection)
        builder.OwnsMany(c => c.Addresses, addr =>
        {
            addr.ToTable("CustomerAddresses");
            addr.Property(a => a.Street).HasMaxLength(200);
            addr.Property(a => a.City).HasMaxLength(100);
            addr.Property(a => a.Province).HasMaxLength(100);
            addr.Property(a => a.PostalCode).HasMaxLength(10);
            addr.Property(a => a.Country).HasMaxLength(100);
        });

        builder.Property(c => c.Tier).HasConversion<string>().HasMaxLength(20);
    }
}
```

---

## Step 1025: Repository Implementation

```csharp
// Infrastructure/Repositories/OrderRepository.cs
public class OrderRepository(AppDbContext db) : IOrderRepository
{
    public async Task<Order?> GetByIdAsync(OrderId id, CancellationToken ct = default)
        => await db.Orders
            .Include(o => o.Lines)
            .FirstOrDefaultAsync(o => o.Id == id, ct);

    public async Task<IReadOnlyList<Order>> GetByCustomerAsync(
        CustomerId customerId, CancellationToken ct = default)
        => await db.Orders
            .Include(o => o.Lines)
            .Where(o => o.CustomerId == customerId)
            .OrderByDescending(o => o.PlacedAt)
            .ToListAsync(ct);

    public async Task<bool> ExistsAsync(OrderId id, CancellationToken ct = default)
        => await db.Orders.AnyAsync(o => o.Id == id, ct);

    public async Task AddAsync(Order order, CancellationToken ct = default)
    {
        await db.Orders.AddAsync(order, ct);
        await db.SaveChangesAsync(ct);
    }

    public async Task UpdateAsync(Order order, CancellationToken ct = default)
    {
        db.Orders.Update(order);
        await db.SaveChangesAsync(ct);
    }

    public async Task DeleteAsync(OrderId id, CancellationToken ct = default)
    {
        var order = await GetByIdAsync(id, ct)
            ?? throw new NotFoundException("Order", id);
        db.Orders.Remove(order);
        await db.SaveChangesAsync(ct);
    }
}
```

---

## Step 1026: Application Layer - Use Cases

Application layer ประสาน domain objects เพื่อทำ use case

```csharp
// Application/Orders/Commands/PlaceOrder/PlaceOrderCommand.cs
public record PlaceOrderCommand(
    Guid CustomerId,
    string Street,
    string City,
    string Province,
    string PostalCode,
    IReadOnlyList<OrderLineRequest> Lines) : IRequest<OrderId>;

public record OrderLineRequest(Guid ProductId, int Quantity);
```

```csharp
// Application/Orders/Commands/PlaceOrder/PlaceOrderHandler.cs
public class PlaceOrderHandler(
    IOrderRepository orderRepository,
    ICustomerRepository customerRepository,
    IProductRepository productRepository,
    OrderPricingService pricingService) : IRequestHandler<PlaceOrderCommand, OrderId>
{
    public async Task<OrderId> Handle(PlaceOrderCommand cmd, CancellationToken ct)
    {
        var customer = await customerRepository.GetByIdAsync(new CustomerId(cmd.CustomerId), ct)
            ?? throw new NotFoundException("Customer", cmd.CustomerId);

        if (!customer.IsEmailVerified)
            throw new DomainException("Email must be verified before placing an order");

        var shippingAddress = Address.Create(cmd.Street, cmd.City, cmd.Province, cmd.PostalCode);
        var order = Order.Create(customer.Id, shippingAddress);

        foreach (var lineReq in cmd.Lines)
        {
            var product = await productRepository.GetByIdAsync(new ProductId(lineReq.ProductId), ct)
                ?? throw new NotFoundException("Product", lineReq.ProductId);

            if (!product.IsAvailable)
                throw new DomainException($"Product '{product.Name}' is not available");

            order.AddLine(
                product.Id,
                product.Name,
                product.Price,
                Quantity.Of(lineReq.Quantity));
        }

        order.Confirm();

        await orderRepository.AddAsync(order, ct);
        return order.Id;
    }
}
```

```csharp
// Application/Orders/Commands/CancelOrder/CancelOrderCommand.cs
public record CancelOrderCommand(Guid OrderId, string Reason) : IRequest;

// Application/Orders/Commands/CancelOrder/CancelOrderHandler.cs
public class CancelOrderHandler(IOrderRepository orderRepository)
    : IRequestHandler<CancelOrderCommand>
{
    public async Task Handle(CancelOrderCommand cmd, CancellationToken ct)
    {
        var order = await orderRepository.GetByIdAsync(OrderId.From(cmd.OrderId), ct)
            ?? throw new NotFoundException("Order", cmd.OrderId);

        order.Cancel(cmd.Reason);
        await orderRepository.UpdateAsync(order, ct);
    }
}
```

---

## Step 1027: Application Layer - Queries

```csharp
// Application/Orders/Queries/GetOrder/GetOrderQuery.cs
public record GetOrderQuery(Guid OrderId) : IRequest<OrderDto?>;

// Application/Orders/Queries/GetOrder/OrderDto.cs
public record OrderDto(
    Guid Id,
    Guid CustomerId,
    string Status,
    AddressDto ShippingAddress,
    decimal SubTotal,
    decimal ShippingFee,
    decimal TotalAmount,
    DateTime PlacedAt,
    IReadOnlyList<OrderLineDto> Lines);

public record OrderLineDto(
    Guid Id,
    Guid ProductId,
    string ProductName,
    decimal UnitPrice,
    int Quantity,
    decimal TotalPrice);

public record AddressDto(string Street, string City, string Province, string PostalCode, string Country);

// Application/Orders/Queries/GetOrder/GetOrderHandler.cs
public class GetOrderHandler(IOrderRepository orderRepository)
    : IRequestHandler<GetOrderQuery, OrderDto?>
{
    public async Task<OrderDto?> Handle(GetOrderQuery query, CancellationToken ct)
    {
        var order = await orderRepository.GetByIdAsync(OrderId.From(query.OrderId), ct);
        return order is null ? null : MapToDto(order);
    }

    private static OrderDto MapToDto(Order order) => new(
        order.Id.Value,
        order.CustomerId.Value,
        order.Status.ToString(),
        new AddressDto(
            order.ShippingAddress.Street,
            order.ShippingAddress.City,
            order.ShippingAddress.Province,
            order.ShippingAddress.PostalCode,
            order.ShippingAddress.Country),
        order.SubTotal.Amount,
        order.ShippingFee.Amount,
        order.TotalAmount.Amount,
        order.PlacedAt,
        order.Lines.Select(l => new OrderLineDto(
            l.Id.Value,
            l.ProductId.Value,
            l.ProductName,
            l.UnitPrice.Amount,
            l.Quantity.Value,
            l.TotalPrice.Amount)).ToList());
}
```

---

## Step 1028: Domain Event Handlers

```csharp
// Application/Orders/EventHandlers/OrderConfirmedEventHandler.cs
public class OrderConfirmedEventHandler(
    IInventoryService inventoryService,
    IEmailService emailService,
    ICustomerRepository customerRepository,
    ILogger<OrderConfirmedEventHandler> logger)
    : INotificationHandler<OrderConfirmedEvent>
{
    public async Task Handle(OrderConfirmedEvent notification, CancellationToken ct)
    {
        logger.LogInformation("Order {OrderId} confirmed for customer {CustomerId}",
            notification.OrderId, notification.CustomerId);

        // Reserve inventory
        await inventoryService.ReserveStockForOrderAsync(notification.OrderId, ct);

        // Send confirmation email
        var customer = await customerRepository.GetByIdAsync(notification.CustomerId, ct);
        if (customer is not null)
        {
            await emailService.SendOrderConfirmationAsync(
                customer.Email.Value,
                notification.OrderId.Value,
                notification.TotalAmount.Amount,
                ct);
        }
    }
}
```

```csharp
// Application/Customers/EventHandlers/CustomerRegisteredEventHandler.cs
public class CustomerRegisteredEventHandler(
    IEmailService emailService,
    ILogger<CustomerRegisteredEventHandler> logger)
    : INotificationHandler<CustomerRegisteredEvent>
{
    public async Task Handle(CustomerRegisteredEvent notification, CancellationToken ct)
    {
        logger.LogInformation("New customer registered: {CustomerId}", notification.CustomerId);

        await emailService.SendWelcomeEmailAsync(notification.Email.Value, ct);
    }
}
```

---

## Step 1029: API Layer - Controllers

```csharp
// API/Controllers/OrdersController.cs
[ApiController]
[Route("api/[controller]")]
public class OrdersController(ISender sender) : ControllerBase
{
    [HttpPost]
    [ProducesResponseType(typeof(Guid), StatusCodes.Status201Created)]
    [ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status400BadRequest)]
    [ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status404NotFound)]
    public async Task<IActionResult> PlaceOrder(
        [FromBody] PlaceOrderRequest request,
        CancellationToken ct)
    {
        var command = new PlaceOrderCommand(
            request.CustomerId,
            request.Street, request.City, request.Province, request.PostalCode,
            request.Lines.Select(l => new OrderLineRequest(l.ProductId, l.Quantity)).ToList());

        var orderId = await sender.Send(command, ct);
        return CreatedAtAction(nameof(GetOrder), new { id = orderId.Value }, orderId.Value);
    }

    [HttpGet("{id:guid}")]
    [ProducesResponseType(typeof(OrderDto), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> GetOrder(Guid id, CancellationToken ct)
    {
        var order = await sender.Send(new GetOrderQuery(id), ct);
        return order is null ? NotFound() : Ok(order);
    }

    [HttpDelete("{id:guid}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    public async Task<IActionResult> CancelOrder(Guid id, [FromBody] CancelOrderRequest request, CancellationToken ct)
    {
        await sender.Send(new CancelOrderCommand(id, request.Reason), ct);
        return NoContent();
    }
}

// API/Models/PlaceOrderRequest.cs
public record PlaceOrderRequest(
    Guid CustomerId,
    string Street,
    string City,
    string Province,
    string PostalCode,
    IReadOnlyList<OrderLineItemRequest> Lines);

public record OrderLineItemRequest(Guid ProductId, int Quantity);
public record CancelOrderRequest(string Reason);
```

---

## Step 1030: Exception Handling Middleware

```csharp
// API/Middleware/DomainExceptionMiddleware.cs
public class DomainExceptionMiddleware(RequestDelegate next, ILogger<DomainExceptionMiddleware> logger)
{
    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await next(context);
        }
        catch (DomainException ex)
        {
            logger.LogWarning(ex, "Domain exception: {Message}", ex.Message);
            context.Response.StatusCode = StatusCodes.Status400BadRequest;
            await context.Response.WriteAsJsonAsync(new ProblemDetails
            {
                Title = "Business Rule Violation",
                Detail = ex.Message,
                Status = StatusCodes.Status400BadRequest
            });
        }
        catch (NotFoundException ex)
        {
            logger.LogWarning(ex, "Not found: {Message}", ex.Message);
            context.Response.StatusCode = StatusCodes.Status404NotFound;
            await context.Response.WriteAsJsonAsync(new ProblemDetails
            {
                Title = "Not Found",
                Detail = ex.Message,
                Status = StatusCodes.Status404NotFound
            });
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "Unhandled exception");
            context.Response.StatusCode = StatusCodes.Status500InternalServerError;
            await context.Response.WriteAsJsonAsync(new ProblemDetails
            {
                Title = "Internal Server Error",
                Status = StatusCodes.Status500InternalServerError
            });
        }
    }
}
```

---

## Step 1031: DI Registration

```csharp
// Infrastructure/DependencyInjection.cs
public static class DependencyInjection
{
    public static IServiceCollection AddInfrastructure(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddDbContext<AppDbContext>(options =>
            options.UseNpgsql(configuration.GetConnectionString("DefaultConnection")));

        services.AddScoped<IOrderRepository, OrderRepository>();
        services.AddScoped<ICustomerRepository, CustomerRepository>();
        services.AddScoped<IProductRepository, ProductRepository>();
        services.AddScoped<IDiscountRepository, DiscountRepository>();

        return services;
    }
}

// Application/DependencyInjection.cs
public static class DependencyInjection
{
    public static IServiceCollection AddApplication(this IServiceCollection services)
    {
        services.AddMediatR(cfg =>
        {
            cfg.RegisterServicesFromAssembly(typeof(DependencyInjection).Assembly);
            cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
            cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
        });

        services.AddValidatorsFromAssembly(typeof(DependencyInjection).Assembly);
        services.AddScoped<OrderPricingService>();
        services.AddScoped<StockReservationService>();

        return services;
    }
}
```

---

## Step 1032: MediatR Pipeline Behaviors

```csharp
// Application/Common/Behaviors/ValidationBehavior.cs
public class ValidationBehavior<TRequest, TResponse>(
    IEnumerable<IValidator<TRequest>> validators)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken ct)
    {
        if (!validators.Any()) return await next();

        var context = new ValidationContext<TRequest>(request);
        var results = await Task.WhenAll(validators.Select(v => v.ValidateAsync(context, ct)));
        var failures = results.SelectMany(r => r.Errors).Where(f => f is not null).ToList();

        if (failures.Count != 0)
            throw new ValidationException(failures);

        return await next();
    }
}

// Application/Common/Behaviors/LoggingBehavior.cs
public class LoggingBehavior<TRequest, TResponse>(ILogger<LoggingBehavior<TRequest, TResponse>> logger)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken ct)
    {
        var requestName = typeof(TRequest).Name;
        logger.LogInformation("Handling {RequestName}", requestName);

        var sw = Stopwatch.StartNew();
        var response = await next();
        sw.Stop();

        logger.LogInformation("Handled {RequestName} in {ElapsedMs}ms", requestName, sw.ElapsedMilliseconds);
        return response;
    }
}
```

---

## Step 1033: FluentValidation for Commands

```csharp
// Application/Orders/Commands/PlaceOrder/PlaceOrderValidator.cs
public class PlaceOrderValidator : AbstractValidator<PlaceOrderCommand>
{
    public PlaceOrderValidator()
    {
        RuleFor(x => x.CustomerId).NotEmpty();

        RuleFor(x => x.Street).NotEmpty().MaximumLength(200);
        RuleFor(x => x.City).NotEmpty().MaximumLength(100);
        RuleFor(x => x.Province).NotEmpty().MaximumLength(100);
        RuleFor(x => x.PostalCode)
            .NotEmpty()
            .Length(5)
            .Matches(@"^\d{5}$").WithMessage("Postal code must be 5 digits");

        RuleFor(x => x.Lines).NotEmpty().WithMessage("Order must have at least one item");
        RuleForEach(x => x.Lines).ChildRules(line =>
        {
            line.RuleFor(l => l.ProductId).NotEmpty();
            line.RuleFor(l => l.Quantity).GreaterThan(0).LessThanOrEqualTo(100);
        });
    }
}
```

---

## Step 1034: Unit Tests for Domain

```csharp
// Tests/Domain/OrderTests.cs
public class OrderTests
{
    [Fact]
    public void Create_ValidData_ShouldCreateDraftOrder()
    {
        var customerId = CustomerId.New();
        var address = Address.Create("123 Main St", "Bangkok", "Bangkok", "10100");

        var order = Order.Create(customerId, address);

        order.Status.Should().Be(OrderStatus.Draft);
        order.Lines.Should().BeEmpty();
        order.DomainEvents.Should().ContainSingle(e => e is OrderCreatedEvent);
    }

    [Fact]
    public void AddLine_ValidProduct_ShouldAddLine()
    {
        var order = CreateDraftOrder();
        var productId = new ProductId(Guid.NewGuid());

        order.AddLine(productId, "Test Product", Money.Of(100), Quantity.Of(2));

        order.Lines.Should().HaveCount(1);
        order.SubTotal.Amount.Should().Be(200);
    }

    [Fact]
    public void AddLine_SameProduct_ShouldMergeQuantity()
    {
        var order = CreateDraftOrder();
        var productId = new ProductId(Guid.NewGuid());

        order.AddLine(productId, "Product", Money.Of(100), Quantity.Of(2));
        order.AddLine(productId, "Product", Money.Of(100), Quantity.Of(3));

        order.Lines.Should().HaveCount(1);
        order.Lines[0].Quantity.Value.Should().Be(5);
    }

    [Fact]
    public void Confirm_EmptyOrder_ShouldThrow()
    {
        var order = CreateDraftOrder();

        var act = () => order.Confirm();

        act.Should().Throw<DomainException>()
            .WithMessage("Cannot confirm empty order");
    }

    [Fact]
    public void Confirm_WithLines_ShouldConfirmAndRaiseEvent()
    {
        var order = CreateDraftOrder();
        order.AddLine(new ProductId(Guid.NewGuid()), "Product", Money.Of(500), Quantity.Of(1));

        order.Confirm();

        order.Status.Should().Be(OrderStatus.Confirmed);
        order.ConfirmedAt.Should().NotBeNull();
        order.DomainEvents.OfType<OrderConfirmedEvent>().Should().HaveCount(1);
    }

    [Fact]
    public void Cancel_ShippedOrder_ShouldThrow()
    {
        var order = CreateConfirmedOrder();
        order.Ship();

        var act = () => order.Cancel("Changed mind");

        act.Should().Throw<DomainException>()
            .WithMessage("Cannot cancel Shipped order");
    }

    private static Order CreateDraftOrder()
    {
        var address = Address.Create("123 Main St", "Bangkok", "Bangkok", "10100");
        return Order.Create(CustomerId.New(), address);
    }

    private static Order CreateConfirmedOrder()
    {
        var order = CreateDraftOrder();
        order.AddLine(new ProductId(Guid.NewGuid()), "Product", Money.Of(100), Quantity.Of(1));
        order.Confirm();
        return order;
    }
}
```

---

## Step 1035: Unit Tests for Value Objects

```csharp
// Tests/Domain/MoneyTests.cs
public class MoneyTests
{
    [Fact]
    public void Create_NegativeAmount_ShouldThrow()
    {
        var act = () => Money.Of(-1);
        act.Should().Throw<DomainException>();
    }

    [Fact]
    public void Add_SameCurrency_ShouldWork()
    {
        var a = Money.Of(100, "THB");
        var b = Money.Of(200, "THB");

        var result = a + b;

        result.Amount.Should().Be(300);
        result.Currency.Should().Be("THB");
    }

    [Fact]
    public void Add_DifferentCurrency_ShouldThrow()
    {
        var thb = Money.Of(100, "THB");
        var usd = Money.Of(10, "USD");

        var act = () => thb + usd;
        act.Should().Throw<DomainException>();
    }

    [Theory]
    [InlineData(100, 0.1, 10)]
    [InlineData(1000, 0.05, 50)]
    [InlineData(250, 0.2, 50)]
    public void Multiply_Factor_ShouldCalculateCorrectly(decimal amount, decimal factor, decimal expected)
    {
        var money = Money.Of(amount);
        var result = money.Multiply(factor);
        result.Amount.Should().Be(expected);
    }

    [Fact]
    public void Equality_SameAmountAndCurrency_ShouldBeEqual()
    {
        var a = Money.Of(100, "THB");
        var b = Money.Of(100, "THB");

        (a == b).Should().BeTrue();
        a.Equals(b).Should().BeTrue();
    }
}
```

---

## Step 1036: Integration Tests

```csharp
// Tests/Integration/OrderIntegrationTests.cs
public class OrderIntegrationTests(WebApplicationFactory<Program> factory)
    : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client = factory.CreateClient();

    [Fact]
    public async Task PlaceOrder_ValidRequest_ShouldReturn201()
    {
        // Arrange
        var customerId = await CreateCustomerAsync();
        var productId = await CreateProductAsync();

        var request = new
        {
            customerId,
            street = "123 Test St",
            city = "Bangkok",
            province = "Bangkok",
            postalCode = "10100",
            lines = new[] { new { productId, quantity = 2 } }
        };

        // Act
        var response = await _client.PostAsJsonAsync("/api/orders", request);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Created);
        var orderId = await response.Content.ReadFromJsonAsync<Guid>();
        orderId.Should().NotBeEmpty();
    }
}
```

---

## Step 1037: Context Map - Anti-Corruption Layer

Anti-Corruption Layer (ACL) ป้องกัน domain ของเราจาก external systems

```csharp
// Integration/ExternalSystems/LegacyInventoryAdapter.cs
// ACL: แปลง Legacy System model → Domain model
public class LegacyInventoryAdapter(ILegacyInventoryClient legacyClient)
    : IInventoryService
{
    public async Task<bool> CheckStockAsync(ProductId productId, Quantity quantity, CancellationToken ct)
    {
        // Legacy system ใช้ string product code
        var legacyProductCode = $"PROD-{productId.Value:N}";
        var legacyResponse = await legacyClient.CheckAvailabilityAsync(legacyProductCode, quantity.Value, ct);

        // แปลง legacy response → domain concept
        return legacyResponse.AvailableUnits >= quantity.Value
            && legacyResponse.Status == "ACTIVE";
    }

    public async Task ReserveStockForOrderAsync(OrderId orderId, CancellationToken ct)
    {
        // แปลง domain OrderId → legacy format
        var legacyRequest = new LegacyReservationRequest
        {
            ExternalOrderRef = $"ORD-{orderId.Value:N}",
            ReservationTimestamp = DateTime.UtcNow.ToString("yyyyMMddHHmmss")
        };
        await legacyClient.CreateReservationAsync(legacyRequest, ct);
    }
}
```

---

## Step 1038: Specification Pattern

Specification Pattern ช่วยทำ query logic ที่ reusable และ composable

```csharp
// Domain/Common/Specification.cs
public abstract class Specification<T>
{
    public abstract Expression<Func<T, bool>> ToExpression();

    public bool IsSatisfiedBy(T entity) => ToExpression().Compile()(entity);

    public Specification<T> And(Specification<T> other) => new AndSpecification<T>(this, other);
    public Specification<T> Or(Specification<T> other) => new OrSpecification<T>(this, other);
    public Specification<T> Not() => new NotSpecification<T>(this);
}

public class AndSpecification<T>(Specification<T> left, Specification<T> right) : Specification<T>
{
    public override Expression<Func<T, bool>> ToExpression()
    {
        var leftExpr = left.ToExpression();
        var rightExpr = right.ToExpression();
        var param = Expression.Parameter(typeof(T));
        var body = Expression.AndAlso(
            Expression.Invoke(leftExpr, param),
            Expression.Invoke(rightExpr, param));
        return Expression.Lambda<Func<T, bool>>(body, param);
    }
}

public class OrSpecification<T>(Specification<T> left, Specification<T> right) : Specification<T>
{
    public override Expression<Func<T, bool>> ToExpression()
    {
        var leftExpr = left.ToExpression();
        var rightExpr = right.ToExpression();
        var param = Expression.Parameter(typeof(T));
        var body = Expression.OrElse(
            Expression.Invoke(leftExpr, param),
            Expression.Invoke(rightExpr, param));
        return Expression.Lambda<Func<T, bool>>(body, param);
    }
}

public class NotSpecification<T>(Specification<T> inner) : Specification<T>
{
    public override Expression<Func<T, bool>> ToExpression()
    {
        var expr = inner.ToExpression();
        var param = Expression.Parameter(typeof(T));
        var body = Expression.Not(Expression.Invoke(expr, param));
        return Expression.Lambda<Func<T, bool>>(body, param);
    }
}
```

```csharp
// Domain/Orders/Specifications/OrderSpecifications.cs
public class OrdersByCustomerSpec(CustomerId customerId) : Specification<Order>
{
    public override Expression<Func<Order, bool>> ToExpression()
        => order => order.CustomerId == customerId;
}

public class PendingOrdersSpec : Specification<Order>
{
    public override Expression<Func<Order, bool>> ToExpression()
        => order => order.Status == OrderStatus.Confirmed || order.Status == OrderStatus.Processing;
}

public class HighValueOrdersSpec(decimal threshold) : Specification<Order>
{
    public override Expression<Func<Order, bool>> ToExpression()
        => order => order.Lines.Sum(l => l.UnitPrice.Amount * l.Quantity.Value) > threshold;
}

// Usage:
// var spec = new OrdersByCustomerSpec(customerId).And(new PendingOrdersSpec());
// var orders = await repo.FindAsync(spec);
```

---

## Step 1039: Policy Pattern

Policy encapsulates business rules ที่สามารถ test ได้ง่าย

```csharp
// Domain/Orders/Policies/IOrderPolicy.cs
public interface IOrderPolicy
{
    Task<PolicyResult> EvaluateAsync(Order order, Customer customer, CancellationToken ct = default);
}

public record PolicyResult(bool IsAllowed, string? ViolationReason = null)
{
    public static PolicyResult Allow() => new(true);
    public static PolicyResult Deny(string reason) => new(false, reason);
}

// Domain/Orders/Policies/MaxOrderValuePolicy.cs
public class MaxOrderValuePolicy : IOrderPolicy
{
    private static readonly Money MaxOrderValue = Money.Of(100_000, "THB");

    public Task<PolicyResult> EvaluateAsync(Order order, Customer customer, CancellationToken ct)
    {
        if (order.TotalAmount.Amount > MaxOrderValue.Amount && customer.Tier == CustomerTier.Regular)
            return Task.FromResult(PolicyResult.Deny(
                $"Order exceeds maximum allowed value of {MaxOrderValue} for Regular customers"));

        return Task.FromResult(PolicyResult.Allow());
    }
}

// Domain/Orders/Policies/MinimumOrderValuePolicy.cs
public class MinimumOrderValuePolicy : IOrderPolicy
{
    private static readonly Money MinOrderValue = Money.Of(100, "THB");

    public Task<PolicyResult> EvaluateAsync(Order order, Customer customer, CancellationToken ct)
    {
        if (order.SubTotal.Amount < MinOrderValue.Amount)
            return Task.FromResult(PolicyResult.Deny(
                $"Minimum order value is {MinOrderValue}"));

        return Task.FromResult(PolicyResult.Allow());
    }
}
```

---

## Step 1040: Saga / Process Manager

สำหรับ workflows ที่ span หลาย Bounded Contexts

```csharp
// Application/Sagas/OrderProcessingSaga.cs
public class OrderProcessingSagaState
{
    public Guid OrderId { get; set; }
    public Guid CustomerId { get; set; }
    public bool IsStockReserved { get; set; }
    public bool IsPaymentProcessed { get; set; }
    public string CurrentStep { get; set; } = "Started";
}

public class OrderProcessingSaga
{
    private readonly IOrderRepository _orderRepo;
    private readonly IInventoryService _inventory;
    private readonly IPaymentService _payment;

    public OrderProcessingSaga(
        IOrderRepository orderRepo,
        IInventoryService inventory,
        IPaymentService payment)
    {
        _orderRepo = orderRepo;
        _inventory = inventory;
        _payment = payment;
    }

    public async Task<SagaResult> ExecuteAsync(
        OrderProcessingSagaState state,
        CancellationToken ct = default)
    {
        var compensations = new Stack<Func<CancellationToken, Task>>();

        try
        {
            // Step 1: Reserve Stock
            if (!state.IsStockReserved)
            {
                var reserved = await _inventory.ReserveStockForOrderAsync(
                    OrderId.From(state.OrderId), ct);

                if (!reserved)
                    return SagaResult.Failed("Stock not available");

                state.IsStockReserved = true;
                state.CurrentStep = "StockReserved";
                compensations.Push(async token =>
                    await _inventory.ReleaseReservationAsync(OrderId.From(state.OrderId), token));
            }

            // Step 2: Process Payment
            if (!state.IsPaymentProcessed)
            {
                var paymentResult = await _payment.ProcessPaymentAsync(
                    state.OrderId, state.CustomerId, ct);

                if (!paymentResult.IsSuccess)
                {
                    // Compensate: release stock
                    await RunCompensationsAsync(compensations, ct);
                    return SagaResult.Failed($"Payment failed: {paymentResult.ErrorMessage}");
                }

                state.IsPaymentProcessed = true;
                state.CurrentStep = "PaymentProcessed";
                compensations.Push(async token =>
                    await _payment.RefundAsync(paymentResult.TransactionId, token));
            }

            // Step 3: Confirm Order
            var order = await _orderRepo.GetByIdAsync(OrderId.From(state.OrderId), ct)
                ?? throw new NotFoundException("Order", state.OrderId);
            order.Ship();
            await _orderRepo.UpdateAsync(order, ct);

            return SagaResult.Completed();
        }
        catch (Exception ex)
        {
            await RunCompensationsAsync(compensations, ct);
            return SagaResult.Failed(ex.Message);
        }
    }

    private static async Task RunCompensationsAsync(
        Stack<Func<CancellationToken, Task>> compensations,
        CancellationToken ct)
    {
        while (compensations.TryPop(out var compensate))
        {
            try { await compensate(ct); }
            catch { /* log but continue */ }
        }
    }
}

public record SagaResult(bool IsCompleted, string? ErrorMessage)
{
    public static SagaResult Completed() => new(true, null);
    public static SagaResult Failed(string error) => new(false, error);
}
```

---

## Step 1041: Read Model / Projection

สำหรับ CQRS: แยก Read Model จาก Write Model

```csharp
// Infrastructure/ReadModels/OrderSummaryReadModel.cs
[Table("OrderSummaries")]
public class OrderSummaryReadModel
{
    public Guid Id { get; set; }
    public Guid CustomerId { get; set; }
    public string CustomerName { get; set; } = string.Empty;
    public string Status { get; set; } = string.Empty;
    public decimal TotalAmount { get; set; }
    public int ItemCount { get; set; }
    public DateTime PlacedAt { get; set; }
}

// Infrastructure/Projections/OrderProjection.cs
public class OrderProjection(ReadDbContext readDb) :
    INotificationHandler<OrderCreatedEvent>,
    INotificationHandler<OrderConfirmedEvent>,
    INotificationHandler<OrderCancelledEvent>
{
    public async Task Handle(OrderCreatedEvent notification, CancellationToken ct)
    {
        var summary = new OrderSummaryReadModel
        {
            Id = notification.OrderId.Value,
            CustomerId = notification.CustomerId.Value,
            Status = "Draft",
            PlacedAt = DateTime.UtcNow
        };
        await readDb.OrderSummaries.AddAsync(summary, ct);
        await readDb.SaveChangesAsync(ct);
    }

    public async Task Handle(OrderConfirmedEvent notification, CancellationToken ct)
    {
        var summary = await readDb.OrderSummaries.FindAsync([notification.OrderId.Value], ct);
        if (summary is null) return;

        summary.Status = "Confirmed";
        summary.TotalAmount = notification.TotalAmount.Amount;
        await readDb.SaveChangesAsync(ct);
    }

    public async Task Handle(OrderCancelledEvent notification, CancellationToken ct)
    {
        var summary = await readDb.OrderSummaries.FindAsync([notification.OrderId.Value], ct);
        if (summary is null) return;

        summary.Status = "Cancelled";
        await readDb.SaveChangesAsync(ct);
    }
}
```

---

## Step 1042: Domain-Driven API Design

```csharp
// API/Endpoints/OrderEndpoints.cs - Minimal API style
public static class OrderEndpoints
{
    public static void MapOrderEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/orders")
            .WithTags("Orders")
            .RequireAuthorization();

        group.MapPost("/", PlaceOrderAsync)
            .WithName("PlaceOrder")
            .Produces<Guid>(StatusCodes.Status201Created)
            .ProducesValidationProblem()
            .ProducesProblem(StatusCodes.Status404NotFound);

        group.MapGet("/{id:guid}", GetOrderAsync)
            .WithName("GetOrder")
            .Produces<OrderDto>()
            .ProducesProblem(StatusCodes.Status404NotFound);

        group.MapGet("/customer/{customerId:guid}", GetCustomerOrdersAsync)
            .WithName("GetCustomerOrders")
            .Produces<IReadOnlyList<OrderDto>>();

        group.MapPut("/{id:guid}/confirm", ConfirmOrderAsync)
            .WithName("ConfirmOrder")
            .Produces(StatusCodes.Status204NoContent);

        group.MapDelete("/{id:guid}", CancelOrderAsync)
            .WithName("CancelOrder")
            .Produces(StatusCodes.Status204NoContent);
    }

    private static async Task<IResult> PlaceOrderAsync(
        PlaceOrderRequest request,
        ISender sender,
        CancellationToken ct)
    {
        var command = request.ToCommand();
        var orderId = await sender.Send(command, ct);
        return Results.CreatedAtRoute("GetOrder", new { id = orderId.Value }, orderId.Value);
    }

    private static async Task<IResult> GetOrderAsync(
        Guid id,
        ISender sender,
        CancellationToken ct)
    {
        var order = await sender.Send(new GetOrderQuery(id), ct);
        return order is null ? Results.NotFound() : Results.Ok(order);
    }

    private static async Task<IResult> GetCustomerOrdersAsync(
        Guid customerId,
        ISender sender,
        CancellationToken ct)
    {
        var orders = await sender.Send(new GetCustomerOrdersQuery(customerId), ct);
        return Results.Ok(orders);
    }

    private static async Task<IResult> ConfirmOrderAsync(
        Guid id, ISender sender, CancellationToken ct)
    {
        await sender.Send(new ConfirmOrderCommand(id), ct);
        return Results.NoContent();
    }

    private static async Task<IResult> CancelOrderAsync(
        Guid id,
        CancelOrderRequest request,
        ISender sender,
        CancellationToken ct)
    {
        await sender.Send(new CancelOrderCommand(id, request.Reason), ct);
        return Results.NoContent();
    }
}
```

---

## Step 1043: Product Aggregate

```csharp
// Domain/Inventory/Product.cs
public class Product : AggregateRoot<ProductId>
{
    public string Name { get; private set; }
    public string Description { get; private set; }
    public Money Price { get; private set; }
    public Quantity StockQuantity { get; private set; }
    public int ReservedQuantity { get; private set; }
    public bool IsAvailable { get; private set; }
    public CategoryId CategoryId { get; private set; }
    public Sku Sku { get; private set; }

    public int AvailableStock => StockQuantity.Value - ReservedQuantity;

    private Product() : base(default!) { }

    private Product(
        ProductId id, string name, string description, Money price,
        Quantity initialStock, CategoryId categoryId, Sku sku) : base(id)
    {
        Name = name;
        Description = description;
        Price = price;
        StockQuantity = initialStock;
        ReservedQuantity = 0;
        IsAvailable = true;
        CategoryId = categoryId;
        Sku = sku;
    }

    public static Product Create(
        string name, string description, Money price,
        Quantity initialStock, CategoryId categoryId, string skuCode)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(name);
        return new Product(
            ProductId.New(), name, description, price,
            initialStock, categoryId, Sku.Create(skuCode));
    }

    public bool HasSufficientStock(Quantity requested) => AvailableStock >= requested.Value;

    public void ReserveStock(Quantity quantity)
    {
        if (!HasSufficientStock(quantity))
            throw new DomainException($"Insufficient stock for product {Name}. Available: {AvailableStock}");

        ReservedQuantity += quantity.Value;
        RaiseDomainEvent(new StockReservedEvent(Id, quantity));
    }

    public void ReleaseReservation(Quantity quantity)
    {
        ReservedQuantity = Math.Max(0, ReservedQuantity - quantity.Value);
        RaiseDomainEvent(new StockReleasedEvent(Id, quantity));
    }

    public void AdjustStock(Quantity newQuantity, string reason)
    {
        var oldQty = StockQuantity;
        StockQuantity = newQuantity;
        RaiseDomainEvent(new StockAdjustedEvent(Id, oldQty, newQuantity, reason));
    }

    public void UpdatePrice(Money newPrice)
    {
        var oldPrice = Price;
        Price = newPrice;
        RaiseDomainEvent(new ProductPriceChangedEvent(Id, oldPrice, newPrice));
    }

    public void Deactivate()
    {
        IsAvailable = false;
        RaiseDomainEvent(new ProductDeactivatedEvent(Id));
    }
}
```

---

## Step 1044: Bounded Context Integration via Domain Events

```csharp
// Ordering Context hears about Inventory events
// Application/Inventory/EventHandlers/StockDepletedEventHandler.cs
public class StockDepletedEventHandler(
    IOrderRepository orderRepository,
    ILogger<StockDepletedEventHandler> logger)
    : INotificationHandler<StockDepletedEvent>
{
    public async Task Handle(StockDepletedEvent notification, CancellationToken ct)
    {
        logger.LogWarning("Stock depleted for product {ProductId}", notification.ProductId);

        // Find pending orders with this product and notify
        var affectedOrders = await orderRepository.FindOrdersWithProductAsync(
            notification.ProductId, OrderStatus.Draft, ct);

        foreach (var order in affectedOrders)
        {
            // Remove the line or mark for review - business decision
            logger.LogInformation(
                "Order {OrderId} may be affected by stock depletion of {ProductId}",
                order.Id, notification.ProductId);
        }
    }
}
```

---

## Step 1045: Event Sourcing สำหรับ Aggregate

```csharp
// Domain/Common/IEventSourcedAggregate.cs
public interface IEventSourcedAggregate
{
    IReadOnlyList<IDomainEvent> UncommittedEvents { get; }
    void LoadFromHistory(IEnumerable<IDomainEvent> history);
    void MarkEventsAsCommitted();
    int Version { get; }
}

// Domain/Orders/EventSourcedOrder.cs
public class EventSourcedOrder : IEventSourcedAggregate
{
    private readonly List<IDomainEvent> _uncommittedEvents = [];
    private readonly List<IDomainEvent> _history = [];

    public OrderId Id { get; private set; } = default!;
    public CustomerId CustomerId { get; private set; } = default!;
    public OrderStatus Status { get; private set; }
    public int Version { get; private set; }

    public IReadOnlyList<IDomainEvent> UncommittedEvents => _uncommittedEvents.AsReadOnly();

    private EventSourcedOrder() { }

    public static EventSourcedOrder Create(CustomerId customerId, Address address)
    {
        var order = new EventSourcedOrder();
        order.Apply(new OrderCreatedEvent(OrderId.New(), customerId));
        return order;
    }

    public void Confirm()
    {
        if (Status != OrderStatus.Draft) throw new DomainException("Invalid state");
        Apply(new OrderConfirmedEvent(Id, CustomerId, Money.Zero()));
    }

    private void Apply(IDomainEvent @event)
    {
        When(@event);
        _uncommittedEvents.Add(@event);
    }

    private void When(IDomainEvent @event)
    {
        switch (@event)
        {
            case OrderCreatedEvent e:
                Id = e.OrderId;
                CustomerId = e.CustomerId;
                Status = OrderStatus.Draft;
                break;
            case OrderConfirmedEvent:
                Status = OrderStatus.Confirmed;
                break;
            case OrderCancelledEvent:
                Status = OrderStatus.Cancelled;
                break;
        }
        Version++;
    }

    public void LoadFromHistory(IEnumerable<IDomainEvent> history)
    {
        foreach (var @event in history)
        {
            When(@event);
            _history.Add(@event);
        }
    }

    public void MarkEventsAsCommitted() => _uncommittedEvents.Clear();
}
```

---

## Step 1046: Idempotency Handling

```csharp
// Application/Common/Behaviors/IdempotencyBehavior.cs
public class IdempotencyBehavior<TRequest, TResponse>(
    IIdempotencyStore store,
    ILogger<IdempotencyBehavior<TRequest, TResponse>> logger)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IIdempotentCommand<TResponse>
{
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken ct)
    {
        var key = $"{typeof(TRequest).Name}:{request.IdempotencyKey}";

        if (await store.ExistsAsync(key, ct))
        {
            logger.LogInformation("Idempotent request detected: {Key}", key);
            return await store.GetAsync<TResponse>(key, ct)
                ?? await next(); // fallback
        }

        var response = await next();
        await store.SetAsync(key, response, TimeSpan.FromHours(24), ct);
        return response;
    }
}

public interface IIdempotentCommand<TResponse>
{
    string IdempotencyKey { get; }
}
```

---

## Step 1047: Domain Invariants และ Business Rules

```csharp
// Domain/Orders/Rules/OrderMustHaveLinesRule.cs
public class OrderMustHaveLinesRule(Order order) : IBusinessRule
{
    public string Message => "Order must have at least one line item";
    public bool IsBroken() => !order.Lines.Any();
}

// Domain/Orders/Rules/OrderTotalCannotExceedCreditLimitRule.cs
public class OrderTotalCannotExceedCreditLimitRule(Order order, Money creditLimit) : IBusinessRule
{
    public string Message => $"Order total {order.TotalAmount} exceeds credit limit {creditLimit}";
    public bool IsBroken() => order.TotalAmount.Amount > creditLimit.Amount;
}

// Domain/Common/IBusinessRule.cs
public interface IBusinessRule
{
    string Message { get; }
    bool IsBroken();
}

// Domain/Common/BusinessRuleChecker.cs
public static class BusinessRuleChecker
{
    public static void Check(params IBusinessRule[] rules)
    {
        var brokenRules = rules.Where(r => r.IsBroken()).ToList();
        if (brokenRules.Count > 0)
            throw new BusinessRuleViolationException(brokenRules.Select(r => r.Message));
    }
}

public class BusinessRuleViolationException(IEnumerable<string> messages)
    : DomainException(string.Join("; ", messages))
{
    public IReadOnlyList<string> ViolatedRules { get; } = messages.ToList().AsReadOnly();
}
```

---

## Step 1048: Domain Snapshot

สำหรับ Aggregate ขนาดใหญ่ที่มี Event history ยาว

```csharp
// Domain/Common/ISnapshotable.cs
public interface ISnapshotable<TSnapshot>
{
    TSnapshot TakeSnapshot();
    void RestoreFromSnapshot(TSnapshot snapshot);
}

// Domain/Orders/OrderSnapshot.cs
public record OrderSnapshot(
    Guid OrderId,
    Guid CustomerId,
    string Status,
    int Version,
    DateTime TakenAt);

// Infrastructure/Snapshots/SnapshotStore.cs
public class SnapshotStore(AppDbContext db) : ISnapshotStore
{
    public async Task SaveSnapshotAsync<TSnapshot>(
        string aggregateId,
        int version,
        TSnapshot snapshot,
        CancellationToken ct)
    {
        var entry = new SnapshotEntry
        {
            AggregateId = aggregateId,
            Version = version,
            SnapshotData = JsonSerializer.Serialize(snapshot),
            TakenAt = DateTime.UtcNow
        };
        await db.Snapshots.AddAsync(entry, ct);
        await db.SaveChangesAsync(ct);
    }

    public async Task<(TSnapshot? Snapshot, int Version)> GetLatestSnapshotAsync<TSnapshot>(
        string aggregateId, CancellationToken ct)
    {
        var entry = await db.Snapshots
            .Where(s => s.AggregateId == aggregateId)
            .OrderByDescending(s => s.Version)
            .FirstOrDefaultAsync(ct);

        if (entry is null) return (default, 0);
        return (JsonSerializer.Deserialize<TSnapshot>(entry.SnapshotData), entry.Version);
    }
}
```

---

## Step 1049: โครงสร้างโปรเจกต์สมบูรณ์

```
ECommerceApp/
├── src/
│   ├── Domain/                          # Domain Layer - ไม่มี dependencies
│   │   ├── Common/
│   │   │   ├── Entity.cs
│   │   │   ├── AggregateRoot.cs
│   │   │   ├── ValueObject.cs
│   │   │   ├── IDomainEvent.cs
│   │   │   ├── DomainEvent.cs
│   │   │   ├── DomainException.cs
│   │   │   ├── IBusinessRule.cs
│   │   │   └── Specification.cs
│   │   ├── Orders/
│   │   │   ├── Order.cs
│   │   │   ├── OrderLine.cs
│   │   │   ├── OrderStatus.cs
│   │   │   ├── OrderId.cs
│   │   │   ├── IOrderRepository.cs
│   │   │   ├── Events/
│   │   │   │   ├── OrderCreatedEvent.cs
│   │   │   │   ├── OrderConfirmedEvent.cs
│   │   │   │   └── OrderCancelledEvent.cs
│   │   │   ├── Rules/
│   │   │   │   └── OrderMustHaveLinesRule.cs
│   │   │   └── Services/
│   │   │       └── OrderPricingService.cs
│   │   ├── Customers/
│   │   │   ├── Customer.cs
│   │   │   ├── CustomerId.cs
│   │   │   ├── PersonName.cs
│   │   │   ├── Email.cs
│   │   │   ├── PhoneNumber.cs
│   │   │   └── ICustomerRepository.cs
│   │   └── Inventory/
│   │       ├── Product.cs
│   │       ├── ProductId.cs
│   │       ├── Sku.cs
│   │       └── IProductRepository.cs
│   ├── Application/                     # Application Layer
│   │   ├── Orders/
│   │   │   ├── Commands/
│   │   │   │   ├── PlaceOrder/
│   │   │   │   │   ├── PlaceOrderCommand.cs
│   │   │   │   │   ├── PlaceOrderHandler.cs
│   │   │   │   │   └── PlaceOrderValidator.cs
│   │   │   │   └── CancelOrder/
│   │   │   │       ├── CancelOrderCommand.cs
│   │   │   │       └── CancelOrderHandler.cs
│   │   │   ├── Queries/
│   │   │   │   └── GetOrder/
│   │   │   │       ├── GetOrderQuery.cs
│   │   │   │       ├── GetOrderHandler.cs
│   │   │   │       └── OrderDto.cs
│   │   │   └── EventHandlers/
│   │   │       └── OrderConfirmedEventHandler.cs
│   │   └── Common/
│   │       ├── Behaviors/
│   │       │   ├── ValidationBehavior.cs
│   │       │   └── LoggingBehavior.cs
│   │       └── DependencyInjection.cs
│   ├── Infrastructure/                  # Infrastructure Layer
│   │   ├── Persistence/
│   │   │   ├── AppDbContext.cs
│   │   │   ├── Configurations/
│   │   │   │   ├── OrderConfiguration.cs
│   │   │   │   └── CustomerConfiguration.cs
│   │   │   └── Migrations/
│   │   ├── Repositories/
│   │   │   ├── OrderRepository.cs
│   │   │   └── CustomerRepository.cs
│   │   └── DependencyInjection.cs
│   └── API/                             # Presentation Layer
│       ├── Controllers/
│       │   └── OrdersController.cs
│       ├── Endpoints/
│       │   └── OrderEndpoints.cs
│       ├── Middleware/
│       │   └── DomainExceptionMiddleware.cs
│       └── Program.cs
└── tests/
    ├── Domain.Tests/
    │   ├── OrderTests.cs
    │   ├── MoneyTests.cs
    │   └── CustomerTests.cs
    ├── Application.Tests/
    │   └── PlaceOrderHandlerTests.cs
    └── Integration.Tests/
        └── OrderIntegrationTests.cs
```

---

## Step 1050: สรุป DDD Patterns

```
DDD Building Blocks ที่ใช้:
┌────────────────────────────────────────────────────────────┐
│  Strategic Design                                           │
│  ┌──────────────────┐  ┌──────────────────────────────┐   │
│  │ Bounded Context  │  │     Context Map              │   │
│  │ - Ordering       │  │ - ACL (Anti-Corruption Layer) │   │
│  │ - Inventory      │  │ - Domain Events integration  │   │
│  │ - Shipping       │  │ - Shared Kernel              │   │
│  └──────────────────┘  └──────────────────────────────┘   │
│                                                             │
│  Tactical Design                                            │
│  ┌──────────┐  ┌───────────────┐  ┌──────────────────┐   │
│  │ Entities │  │ Value Objects │  │ Aggregate Roots  │   │
│  │ - Order  │  │ - Money       │  │ - Order          │   │
│  │ - Line   │  │ - Address     │  │ - Customer       │   │
│  │ - Product│  │ - Email       │  │ - Product        │   │
│  └──────────┘  └───────────────┘  └──────────────────┘   │
│                                                             │
│  ┌──────────────┐  ┌─────────────┐  ┌────────────────┐   │
│  │Domain Events │  │Repositories │  │Domain Services │   │
│  │- OrderCreated│  │- IOrderRepo │  │- PricingService│   │
│  │- OrderConfirm│  │- ICustomer  │  │- StockReserve  │   │
│  └──────────────┘  └─────────────┘  └────────────────┘   │
│                                                             │
│  Architecture Layers:                                       │
│  Domain → Application → Infrastructure → API              │
│  (inner → outer, dependency flows inward)                  │
└────────────────────────────────────────────────────────────┘
```

### Key DDD Principles

1. **Ubiquitous Language** - ใช้ภาษา business ใน code
2. **Bounded Context** - แบ่ง domain ออกเป็น context ที่ชัดเจน
3. **Aggregate Root** - entry point เดียวสำหรับ consistency
4. **Domain Events** - communicate ระหว่าง contexts
5. **Value Objects** - immutable, equality by value
6. **Repository** - collection interface สำหรับ Aggregates
7. **Domain Services** - logic ที่ไม่เป็นของ Entity ใด
8. **Specification** - reusable query logic

---

*Part 36 ครอบคลุม Domain-Driven Design ทั้งหมด 40 Steps (1011-1050)*  
*ต่อไป Part 37: Clean Architecture*
