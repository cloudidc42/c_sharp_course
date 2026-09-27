# Part 51: Functional Programming in C#
## Steps 1430-1460: Immutability, Monads, Railway-Oriented Programming, Pure Functions

---

## Step 1430: Immutability with Records and Init-Only Properties

```csharp
// Records — immutable by default, value semantics
public record Money(decimal Amount, string Currency)
{
    // Validate in constructor
    public Money
    {
        if (Amount < 0) throw new ArgumentException("Amount cannot be negative", nameof(Amount));
        if (string.IsNullOrWhiteSpace(Currency)) throw new ArgumentException("Currency required", nameof(Currency));
        Currency = Currency.ToUpperInvariant(); // Normalize
    }

    // Non-destructive mutation — returns new record
    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException($"Cannot add {Currency} and {other.Currency}");
        return this with { Amount = Amount + other.Amount };
    }

    public Money Subtract(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException($"Cannot subtract different currencies");
        return this with { Amount = Amount - other.Amount };
    }

    public Money Multiply(decimal factor) => this with { Amount = Amount * factor };

    public static Money Zero(string currency) => new(0m, currency);
    public bool IsZero => Amount == 0m;

    public override string ToString() => $"{Amount:F2} {Currency}";
}

// Deep immutable aggregate
public record OrderAggregate(
    Guid Id,
    Guid CustomerId,
    ImmutableList<OrderLine> Lines,
    Money Total,
    OrderStatus Status,
    DateTimeOffset CreatedAt
)
{
    public static OrderAggregate Create(Guid customerId) => new(
        Id: Guid.NewGuid(),
        CustomerId: customerId,
        Lines: ImmutableList<OrderLine>.Empty,
        Total: Money.Zero("USD"),
        Status: OrderStatus.Pending,
        CreatedAt: DateTimeOffset.UtcNow
    );

    public OrderAggregate AddLine(OrderLine line) => this with
    {
        Lines = Lines.Add(line),
        Total = Total.Add(line.LineTotal)
    };

    public OrderAggregate RemoveLine(Guid lineId)
    {
        var line = Lines.FirstOrDefault(l => l.Id == lineId)
            ?? throw new InvalidOperationException($"Line {lineId} not found");

        return this with
        {
            Lines = Lines.Remove(line),
            Total = Total.Subtract(line.LineTotal)
        };
    }

    public OrderAggregate Confirm()
    {
        if (Status != OrderStatus.Pending)
            throw new InvalidOperationException("Only pending orders can be confirmed");
        return this with { Status = OrderStatus.Confirmed };
    }
}

public record OrderLine(Guid Id, string ProductName, int Quantity, Money UnitPrice)
{
    public Money LineTotal => UnitPrice.Multiply(Quantity);
}
```

---

## Step 1431: The Option/Maybe Monad

```csharp
// Option<T> — explicit representation of "value or nothing"
// Eliminates null reference exceptions by making absence explicit

public readonly struct Option<T>
{
    private readonly T? _value;
    private readonly bool _hasValue;

    private Option(T value) { _value = value; _hasValue = true; }

    public static Option<T> Some(T value) => new(value);
    public static Option<T> None() => default;

    public bool HasValue => _hasValue;

    // Map: transform the value if present
    public Option<TResult> Map<TResult>(Func<T, TResult> mapper) =>
        _hasValue ? Option<TResult>.Some(mapper(_value!)) : Option<TResult>.None();

    // Bind (flatMap): chain operations that return Option
    public Option<TResult> Bind<TResult>(Func<T, Option<TResult>> binder) =>
        _hasValue ? binder(_value!) : Option<TResult>.None();

    // Match: handle both cases explicitly
    public TResult Match<TResult>(Func<T, TResult> onSome, Func<TResult> onNone) =>
        _hasValue ? onSome(_value!) : onNone();

    // GetValueOr: provide a default
    public T GetValueOr(T defaultValue) => _hasValue ? _value! : defaultValue;
    public T GetValueOr(Func<T> defaultFactory) => _hasValue ? _value! : defaultFactory();

    // Filter: make None if predicate fails
    public Option<T> Filter(Func<T, bool> predicate) =>
        _hasValue && predicate(_value!) ? this : None();

    // Convert to/from nullable
    public T? ToNullable() => _hasValue ? _value : default;
    public static Option<T> FromNullable(T? value) =>
        value is null ? None() : Some(value);

    public override string ToString() => _hasValue ? $"Some({_value})" : "None";
}

// Extension methods for cleaner usage
public static class OptionExtensions
{
    public static Option<T> Some<T>(this T value) => Option<T>.Some(value);
    public static Option<T> AsOption<T>(this T? value) where T : class =>
        value is null ? Option<T>.None() : Option<T>.Some(value);
    public static Option<T> AsOption<T>(this T? value) where T : struct =>
        value.HasValue ? Option<T>.Some(value.Value) : Option<T>.None();
}

// Usage
public class UserService(IUserRepository repo)
{
    public async Task<string> GetUserEmailAsync(Guid userId)
    {
        var user = await repo.FindByIdAsync(userId);

        return Option<User>.FromNullable(user)
            .Filter(u => u.IsActive)
            .Map(u => u.Email)
            .GetValueOr("unknown@example.com");
    }

    public async Task<Option<UserProfile>> GetUserProfileAsync(Guid userId)
    {
        var user = await repo.FindByIdAsync(userId);
        if (user is null) return Option<UserProfile>.None();

        var profile = await repo.GetProfileAsync(user.Id);
        return Option<UserProfile>.FromNullable(profile)
            .Filter(p => p.IsPublic);
    }
}

// Chaining multiple Option operations
public Option<decimal> CalculateOrderDiscount(Guid customerId, decimal orderTotal)
{
    return GetCustomer(customerId)
        .Filter(c => c.IsActive)
        .Bind(c => GetLoyaltyTier(c.Id))
        .Filter(tier => orderTotal >= tier.MinimumOrderAmount)
        .Map(tier => orderTotal * tier.DiscountRate);
}
```

---

## Step 1432: The Result Monad (Railway-Oriented Programming)

```csharp
// Result<T, TError> — explicit success or failure, no exceptions for domain errors
public readonly struct Result<T, TError>
{
    private readonly T? _value;
    private readonly TError? _error;
    private readonly bool _isSuccess;

    private Result(T value) { _value = value; _isSuccess = true; _error = default; }
    private Result(TError error) { _error = error; _isSuccess = false; _value = default; }

    public static Result<T, TError> Success(T value) => new(value);
    public static Result<T, TError> Failure(TError error) => new(error);

    public bool IsSuccess => _isSuccess;
    public bool IsFailure => !_isSuccess;

    public Result<TResult, TError> Map<TResult>(Func<T, TResult> mapper) =>
        _isSuccess
            ? Result<TResult, TError>.Success(mapper(_value!))
            : Result<TResult, TError>.Failure(_error!);

    public Result<TResult, TError> Bind<TResult>(Func<T, Result<TResult, TError>> binder) =>
        _isSuccess ? binder(_value!) : Result<TResult, TError>.Failure(_error!);

    public Result<T, TNewError> MapError<TNewError>(Func<TError, TNewError> mapper) =>
        _isSuccess
            ? Result<T, TNewError>.Success(_value!)
            : Result<T, TNewError>.Failure(mapper(_error!));

    public TResult Match<TResult>(Func<T, TResult> onSuccess, Func<TError, TResult> onFailure) =>
        _isSuccess ? onSuccess(_value!) : onFailure(_error!);

    public T GetValueOrThrow() =>
        _isSuccess ? _value! : throw new InvalidOperationException($"Result is failure: {_error}");

    public override string ToString() =>
        _isSuccess ? $"Success({_value})" : $"Failure({_error})";
}

// Simplified Result<T> with string error (common pattern)
public readonly struct Result<T>
{
    public static Result<T> Success(T value) => new(value, default, true);
    public static Result<T> Failure(string error) => new(default, error, false);

    public T? Value { get; }
    public string? Error { get; }
    public bool IsSuccess { get; }

    private Result(T? value, string? error, bool isSuccess)
    {
        Value = value; Error = error; IsSuccess = isSuccess;
    }

    public Result<TResult> Map<TResult>(Func<T, TResult> mapper) =>
        IsSuccess ? Result<TResult>.Success(mapper(Value!)) : Result<TResult>.Failure(Error!);

    public Result<TResult> Bind<TResult>(Func<T, Result<TResult>> binder) =>
        IsSuccess ? binder(Value!) : Result<TResult>.Failure(Error!);

    public TResult Match<TResult>(Func<T, TResult> onSuccess, Func<string, TResult> onFailure) =>
        IsSuccess ? onSuccess(Value!) : onFailure(Error!);
}

// Domain errors as discriminated unions (via hierarchy)
public abstract record DomainError(string Message);
public record ValidationError(string Field, string Message) : DomainError($"{Field}: {Message}");
public record NotFoundError(string Resource, object Id) : DomainError($"{Resource} {Id} not found");
public record ConflictError(string Message) : DomainError(Message);
public record UnauthorizedError(string Message) : DomainError(Message);

// Railway-oriented programming — chain of operations, first error short-circuits
public class OrderApplicationService(
    IOrderRepository orderRepo,
    IInventoryService inventory,
    IPaymentService payment,
    IEventBus eventBus)
{
    public async Task<Result<OrderConfirmation, DomainError>> ProcessOrderAsync(
        Guid customerId,
        List<CartItem> items,
        PaymentDetails paymentDetails,
        CancellationToken ct = default)
    {
        // Each step returns Result; chain via Bind
        return await ValidateItems(items)
            .Bind(validItems => CreateOrder(customerId, validItems))
            .BindAsync(order => CheckInventoryAsync(order, ct))
            .BindAsync(order => ChargePaymentAsync(order, paymentDetails, ct))
            .BindAsync(order => SaveAndPublishAsync(order, ct))
            .MapAsync(order => new OrderConfirmation(order.Id, order.Total));
    }

    private Result<List<CartItem>, DomainError> ValidateItems(List<CartItem> items)
    {
        if (items is null || items.Count == 0)
            return Result<List<CartItem>, DomainError>.Failure(
                new ValidationError("Items", "Order must have at least one item"));

        var invalidItem = items.FirstOrDefault(i => i.Quantity <= 0);
        if (invalidItem is not null)
            return Result<List<CartItem>, DomainError>.Failure(
                new ValidationError("Quantity", $"Item '{invalidItem.ProductName}' has invalid quantity"));

        return Result<List<CartItem>, DomainError>.Success(items);
    }

    private Result<Order, DomainError> CreateOrder(Guid customerId, List<CartItem> items)
    {
        try
        {
            var order = Order.Create(customerId, items);
            return Result<Order, DomainError>.Success(order);
        }
        catch (DomainException ex)
        {
            return Result<Order, DomainError>.Failure(new ValidationError("Order", ex.Message));
        }
    }

    private async Task<Result<Order, DomainError>> CheckInventoryAsync(Order order, CancellationToken ct)
    {
        foreach (var item in order.Items)
        {
            var available = await inventory.CheckStockAsync(item.ProductId, item.Quantity, ct);
            if (!available)
                return Result<Order, DomainError>.Failure(
                    new ConflictError($"Insufficient stock for product {item.ProductId}"));
        }
        return Result<Order, DomainError>.Success(order);
    }

    private async Task<Result<Order, DomainError>> ChargePaymentAsync(
        Order order, PaymentDetails details, CancellationToken ct)
    {
        var charge = await payment.ChargeAsync(order.Total, details, ct);
        if (!charge.Success)
            return Result<Order, DomainError>.Failure(new ConflictError($"Payment failed: {charge.DeclineReason}"));

        return Result<Order, DomainError>.Success(order with { PaymentTransactionId = charge.TransactionId });
    }

    private async Task<Result<Order, DomainError>> SaveAndPublishAsync(Order order, CancellationToken ct)
    {
        await orderRepo.SaveAsync(order, ct);
        await eventBus.PublishAsync(new OrderCreatedEvent(order.Id, order.CustomerId, order.Total), ct);
        return Result<Order, DomainError>.Success(order);
    }
}

// Async extension methods for Result chaining
public static class ResultExtensions
{
    public static async Task<Result<TResult, TError>> BindAsync<T, TResult, TError>(
        this Result<T, TError> result,
        Func<T, Task<Result<TResult, TError>>> binder) =>
        result.IsSuccess ? await binder(result.GetValue()) : Result<TResult, TError>.Failure(result.GetError());

    public static async Task<Result<TResult, TError>> MapAsync<T, TResult, TError>(
        this Task<Result<T, TError>> resultTask,
        Func<T, TResult> mapper)
    {
        var result = await resultTask;
        return result.Map(mapper);
    }

    public static async Task<Result<TResult, TError>> BindAsync<T, TResult, TError>(
        this Task<Result<T, TError>> resultTask,
        Func<T, Task<Result<TResult, TError>>> binder)
    {
        var result = await resultTask;
        return result.IsSuccess ? await binder(result.GetValue()) : Result<TResult, TError>.Failure(result.GetError());
    }
}
```

---

## Step 1433: Converting Result to HTTP Responses

```csharp
// Map domain errors to HTTP status codes
public static class ResultHttpExtensions
{
    public static IResult ToHttpResult<T>(this Result<T, DomainError> result) =>
        result.Match(
            onSuccess: value => Results.Ok(value),
            onFailure: error => error switch
            {
                ValidationError ve => Results.ValidationProblem(
                    new Dictionary<string, string[]> { [ve.Field] = [ve.Message] }),
                NotFoundError nfe => Results.Problem(nfe.Message, statusCode: 404),
                ConflictError ce => Results.Problem(ce.Message, statusCode: 409),
                UnauthorizedError ue => Results.Problem(ue.Message, statusCode: 403),
                _ => Results.Problem(error.Message, statusCode: 500)
            }
        );
}

// Minimal API usage
app.MapPost("/api/orders", async (
    CreateOrderRequest request,
    OrderApplicationService service,
    CancellationToken ct) =>
{
    var result = await service.ProcessOrderAsync(
        request.CustomerId, request.Items, request.Payment, ct);

    return result.ToHttpResult();
});
```

---

## Step 1434: Pure Functions and Referential Transparency

```csharp
// Pure functions: same input → always same output, no side effects

// IMPURE: depends on external state, has side effects
public class ImpurePricingService
{
    private decimal _taxRate; // mutable state

    public decimal CalculatePrice(decimal basePrice)
    {
        _taxRate = GetTaxRateFromDatabase(); // I/O side effect
        return basePrice * (1 + _taxRate);  // non-deterministic
    }
}

// PURE: deterministic, testable, cacheable, parallelizable
public static class PurePricingCalculator
{
    public static decimal ApplyTax(decimal basePrice, decimal taxRate) =>
        basePrice * (1 + taxRate);

    public static decimal ApplyDiscount(decimal price, decimal discountRate) =>
        price * (1 - discountRate);

    public static decimal ApplyVolumeDiscount(decimal price, int quantity) =>
        quantity switch
        {
            >= 100 => ApplyDiscount(price, 0.20m),
            >= 50  => ApplyDiscount(price, 0.15m),
            >= 10  => ApplyDiscount(price, 0.10m),
            _      => price
        };

    // Function composition
    public static decimal CalculateFinalPrice(decimal basePrice, decimal taxRate, decimal discountRate, int quantity) =>
        basePrice
        |> (p => ApplyVolumeDiscount(p, quantity))
        |> (p => ApplyDiscount(p, discountRate))
        |> (p => ApplyTax(p, taxRate));
        // Note: C# doesn't have |> operator yet; use LINQ or explicit calls

    // With explicit composition:
    public static decimal CalculateFinalPrice2(decimal basePrice, decimal taxRate, decimal discountRate, int quantity)
    {
        var afterVolume = ApplyVolumeDiscount(basePrice, quantity);
        var afterDiscount = ApplyDiscount(afterVolume, discountRate);
        var afterTax = ApplyTax(afterDiscount, taxRate);
        return afterTax;
    }
}

// Function pipeline helper
public static class FunctionPipeline
{
    public static Func<T, TResult> Compose<T, TMiddle, TResult>(
        Func<T, TMiddle> first,
        Func<TMiddle, TResult> second) =>
        input => second(first(input));

    public static Func<T, TResult3> Compose<T, T2, T3, TResult3>(
        Func<T, T2> f1,
        Func<T2, T3> f2,
        Func<T3, TResult3> f3) =>
        input => f3(f2(f1(input)));
}

// Usage
var calculatePrice = FunctionPipeline.Compose<(decimal price, int qty), decimal, decimal>(
    t => PurePricingCalculator.ApplyVolumeDiscount(t.price, t.qty),
    p => PurePricingCalculator.ApplyTax(p, 0.20m)
);

var finalPrice = calculatePrice((100m, 10));  // 100 * 0.90 * 1.20 = 108
```

---

## Step 1435: LINQ as Functional Programming

```csharp
// LINQ is functional: map (Select), filter (Where), reduce (Aggregate)

public class FunctionalOrderAnalytics
{
    // Declarative data transformation pipeline
    public record RevenueByCategory(string Category, decimal Revenue, int OrderCount);

    public IEnumerable<RevenueByCategory> GetRevenueByCategory(IEnumerable<Order> orders) =>
        orders
            .Where(o => o.Status == OrderStatus.Delivered)
            .SelectMany(o => o.Items.Select(i => (i.Category, i.LineTotal, OrderId: o.Id)))
            .GroupBy(x => x.Category)
            .Select(g => new RevenueByCategory(
                Category: g.Key,
                Revenue: g.Sum(x => x.LineTotal),
                OrderCount: g.Select(x => x.OrderId).Distinct().Count()
            ))
            .OrderByDescending(x => x.Revenue);

    // Fold/Reduce pattern
    public decimal CalculateTotalRevenue(IEnumerable<Order> orders) =>
        orders.Aggregate(0m, (total, order) => total + order.Total);

    // Partition (split into two groups)
    public (IEnumerable<Order> profitable, IEnumerable<Order> unprofitable)
        PartitionByProfitability(IEnumerable<Order> orders, decimal threshold) =>
        (
            orders.Where(o => o.Total >= threshold),
            orders.Where(o => o.Total < threshold)
        );

    // Unfold pattern — generate sequence from a seed
    public IEnumerable<DateTime> GetMonthlyDates(DateTime start, int months) =>
        Enumerable.Range(0, months).Select(i => start.AddMonths(i));

    // Zip two sequences
    public IEnumerable<(Order order, decimal taxAmount)> CalculateTaxes(
        IEnumerable<Order> orders,
        IEnumerable<decimal> taxRates) =>
        orders.Zip(taxRates, (order, rate) => (order, order.Total * rate));

    // Window function (sliding window)
    public IEnumerable<decimal> MovingAverage(IEnumerable<decimal> values, int windowSize) =>
        values
            .Select((_, i) => values.Skip(Math.Max(0, i - windowSize + 1)).Take(windowSize).Average())
            .ToList();
}

// Lazy evaluation with IEnumerable
public class LazyPipeline
{
    // This builds a query object — no data is read until enumerated
    public IEnumerable<string> GetExpensiveProductNames(IEnumerable<Order> orders, decimal threshold) =>
        from order in orders
        where order.Status == OrderStatus.Delivered
        from item in order.Items
        where item.UnitPrice > threshold
        orderby item.UnitPrice descending
        select item.ProductName;

    // Materialize only when needed
    public void ProcessLargeDataset(IEnumerable<Order> orders)
    {
        // Streams records one at a time — no memory spike
        foreach (var name in GetExpensiveProductNames(orders, 100m))
        {
            Console.WriteLine(name);
        }
    }
}
```

---

## Step 1436: Discriminated Unions with Sealed Hierarchies

```csharp
// C# approach to discriminated unions — sealed class/record hierarchies

// Payment method — each case has different data
public abstract record PaymentMethod;
public record CreditCard(string CardNumber, string ExpiryMonth, string ExpiryYear, string Cvv) : PaymentMethod;
public record BankTransfer(string AccountNumber, string RoutingNumber) : PaymentMethod;
public record PayPal(string Email) : PaymentMethod;
public record Cryptocurrency(string WalletAddress, string Network) : PaymentMethod;

// Pattern matching as "switch on union"
public static class PaymentProcessor
{
    public static string GetDisplayName(PaymentMethod method) => method switch
    {
        CreditCard cc => $"Card ending in {cc.CardNumber[^4..]}",
        BankTransfer bt => $"Bank transfer ({bt.RoutingNumber})",
        PayPal pp => $"PayPal ({pp.Email})",
        Cryptocurrency crypto => $"{crypto.Network} wallet",
        _ => throw new UnreachableException()
    };

    public static decimal GetProcessingFee(PaymentMethod method, decimal amount) => method switch
    {
        CreditCard => amount * 0.029m + 0.30m,  // 2.9% + $0.30
        BankTransfer => 0.50m,                   // Flat fee
        PayPal pp when pp.Email.EndsWith(".business") => amount * 0.035m,
        PayPal => amount * 0.029m + 0.30m,
        Cryptocurrency { Network: "ethereum" } => amount * 0.01m,
        Cryptocurrency => amount * 0.005m,
        _ => throw new UnreachableException()
    };

    public static async Task<ChargeResult> ProcessAsync(PaymentMethod method, decimal amount, CancellationToken ct) =>
        method switch
        {
            CreditCard cc => await ProcessCardAsync(cc, amount, ct),
            BankTransfer bt => await ProcessBankTransferAsync(bt, amount, ct),
            PayPal pp => await ProcessPayPalAsync(pp, amount, ct),
            Cryptocurrency crypto => await ProcessCryptoAsync(crypto, amount, ct),
            _ => throw new UnreachableException()
        };

    private static Task<ChargeResult> ProcessCardAsync(CreditCard cc, decimal amount, CancellationToken ct)
        => Task.FromResult(new ChargeResult(true, Guid.NewGuid().ToString()));

    private static Task<ChargeResult> ProcessBankTransferAsync(BankTransfer bt, decimal amount, CancellationToken ct)
        => Task.FromResult(new ChargeResult(true, Guid.NewGuid().ToString()));

    private static Task<ChargeResult> ProcessPayPalAsync(PayPal pp, decimal amount, CancellationToken ct)
        => Task.FromResult(new ChargeResult(true, Guid.NewGuid().ToString()));

    private static Task<ChargeResult> ProcessCryptoAsync(Cryptocurrency crypto, decimal amount, CancellationToken ct)
        => Task.FromResult(new ChargeResult(true, Guid.NewGuid().ToString()));
}

// Event types as discriminated union
public abstract record OrderEvent(Guid OrderId, DateTimeOffset OccurredAt);
public record OrderPlaced(Guid OrderId, Guid CustomerId, decimal Total, DateTimeOffset OccurredAt) : OrderEvent(OrderId, OccurredAt);
public record OrderConfirmed(Guid OrderId, string ConfirmedBy, DateTimeOffset OccurredAt) : OrderEvent(OrderId, OccurredAt);
public record OrderShipped(Guid OrderId, string TrackingNumber, string Carrier, DateTimeOffset OccurredAt) : OrderEvent(OrderId, OccurredAt);
public record OrderDelivered(Guid OrderId, DateTimeOffset OccurredAt) : OrderEvent(OrderId, OccurredAt);
public record OrderCancelled(Guid OrderId, string Reason, DateTimeOffset OccurredAt) : OrderEvent(OrderId, OccurredAt);

// Event handler dispatch
public static class OrderEventHandler
{
    public static string Describe(OrderEvent @event) => @event switch
    {
        OrderPlaced p => $"Order {p.OrderId} placed by customer {p.CustomerId} for ${p.Total:F2}",
        OrderConfirmed c => $"Order {c.OrderId} confirmed by {c.ConfirmedBy}",
        OrderShipped s => $"Order {s.OrderId} shipped via {s.Carrier} ({s.TrackingNumber})",
        OrderDelivered d => $"Order {d.OrderId} delivered",
        OrderCancelled c => $"Order {c.OrderId} cancelled: {c.Reason}",
        _ => $"Unknown event for order {@event.OrderId}"
    };
}
```

---

## Step 1437: Memoization and Lazy Computation

```csharp
// Memoize: cache pure function results
public static class Memoize
{
    public static Func<T, TResult> Create<T, TResult>(Func<T, TResult> func)
        where T : notnull
    {
        var cache = new ConcurrentDictionary<T, TResult>();
        return key => cache.GetOrAdd(key, func);
    }

    public static Func<T1, T2, TResult> Create<T1, T2, TResult>(Func<T1, T2, TResult> func)
        where T1 : notnull where T2 : notnull
    {
        var cache = new ConcurrentDictionary<(T1, T2), TResult>();
        return (k1, k2) => cache.GetOrAdd((k1, k2), k => func(k.Item1, k.Item2));
    }
}

// Usage
public class PricingService
{
    // Expensive computation — memoize by product ID and tier
    private readonly Func<(Guid productId, CustomerTier tier), decimal> _getPriceWithDiscount;

    public PricingService(IProductRepository repo)
    {
        _getPriceWithDiscount = Memoize.Create<(Guid, CustomerTier), decimal>(
            key => CalculatePrice(key.Item1, key.Item2, repo));
    }

    public decimal GetPrice(Guid productId, CustomerTier tier) =>
        _getPriceWithDiscount((productId, tier));

    private static decimal CalculatePrice(Guid productId, CustomerTier tier, IProductRepository repo)
    {
        var product = repo.GetById(productId) ?? throw new KeyNotFoundException();
        return tier switch
        {
            CustomerTier.Gold => product.Price * 0.85m,
            CustomerTier.Silver => product.Price * 0.90m,
            _ => product.Price
        };
    }
}

// Lazy<T> for deferred computation
public class OrderStatisticsService(IOrderRepository repo)
{
    // Computed once, on first access, and cached
    private readonly Lazy<Task<OrderStatsSummary>> _statistics = new(
        async () =>
        {
            var orders = await repo.GetAllAsync();
            return new OrderStatsSummary(
                TotalOrders: orders.Count,
                TotalRevenue: orders.Sum(o => o.Total),
                AverageOrderValue: orders.Average(o => o.Total),
                MaxOrderValue: orders.Max(o => o.Total)
            );
        },
        LazyThreadSafetyMode.ExecutionAndPublication
    );

    public Task<OrderStatsSummary> GetStatisticsAsync() => _statistics.Value;
}
```

---

## Step 1438: Functional Error Accumulation (Validation)

```csharp
// Collect ALL validation errors, not just the first one
// (Unlike Result<T> which short-circuits on first failure)

public class Validated<T>
{
    private readonly T? _value;
    private readonly ImmutableList<string> _errors;

    private Validated(T value) { _value = value; _errors = ImmutableList<string>.Empty; }
    private Validated(ImmutableList<string> errors) { _value = default; _errors = errors; }

    public static Validated<T> Success(T value) => new(value);
    public static Validated<T> Failure(params string[] errors) => new(ImmutableList.CreateRange(errors));
    public static Validated<T> Failure(IEnumerable<string> errors) => new(ImmutableList.CreateRange(errors));

    public bool IsValid => _errors.IsEmpty;
    public IReadOnlyList<string> Errors => _errors;

    public Validated<TResult> Map<TResult>(Func<T, TResult> mapper) =>
        IsValid ? Validated<TResult>.Success(mapper(_value!)) : Validated<TResult>.Failure(_errors);

    // Combine two validations, accumulating errors from both
    public Validated<(T, T2)> And<T2>(Validated<T2> other)
    {
        var allErrors = _errors.AddRange(other._errors);
        if (allErrors.IsEmpty) return Validated<(T, T2)>.Success((_value!, other._value!));
        return Validated<(T, T2)>.Failure(allErrors);
    }

    public TResult Match<TResult>(Func<T, TResult> onSuccess, Func<IReadOnlyList<string>, TResult> onFailure) =>
        IsValid ? onSuccess(_value!) : onFailure(_errors);
}

// Applicative validation — validate all fields
public static class CreateOrderValidator
{
    public static Validated<CreateOrderRequest> Validate(CreateOrderRequest request)
    {
        var customerIdResult = ValidateCustomerId(request.CustomerId);
        var itemsResult = ValidateItems(request.Items);
        var addressResult = ValidateShippingAddress(request.ShippingAddress);

        // Combine — collects ALL errors, not just first
        return customerIdResult
            .And(itemsResult)
            .And(addressResult)
            .Map(_ => request);
    }

    private static Validated<Guid> ValidateCustomerId(Guid id) =>
        id == Guid.Empty
            ? Validated<Guid>.Failure("CustomerId is required")
            : Validated<Guid>.Success(id);

    private static Validated<List<OrderItem>> ValidateItems(List<OrderItem> items)
    {
        var errors = new List<string>();

        if (items is null || items.Count == 0)
            errors.Add("Order must have at least one item");

        if (items?.Any(i => i.Quantity <= 0) == true)
            errors.Add("All items must have positive quantity");

        if (items?.Any(i => i.UnitPrice <= 0) == true)
            errors.Add("All items must have positive price");

        return errors.Count > 0
            ? Validated<List<OrderItem>>.Failure(errors.ToArray())
            : Validated<List<OrderItem>>.Success(items!);
    }

    private static Validated<Address> ValidateShippingAddress(Address? address)
    {
        var errors = new List<string>();

        if (address is null) return Validated<Address>.Failure("Shipping address is required");
        if (string.IsNullOrWhiteSpace(address.Street)) errors.Add("Street is required");
        if (string.IsNullOrWhiteSpace(address.City)) errors.Add("City is required");
        if (string.IsNullOrWhiteSpace(address.PostalCode)) errors.Add("Postal code is required");

        return errors.Count > 0
            ? Validated<Address>.Failure(errors.ToArray())
            : Validated<Address>.Success(address);
    }
}

// Returns ALL validation errors at once
var validation = CreateOrderValidator.Validate(request);
if (!validation.IsValid)
{
    // Errors: ["CustomerId is required", "Order must have at least one item", "Street is required"]
    return Results.ValidationProblem(
        validation.Errors.GroupBy(e => e.Split(':')[0])
            .ToDictionary(g => g.Key, g => g.ToArray()));
}
```

---

## Step 1439: Higher-Order Functions

```csharp
// Functions that take or return functions

public static class OrderProcessingPipeline
{
    // Middleware pattern — each step wraps the next
    public delegate Task<Result<Order, DomainError>> OrderProcessor(Order order, CancellationToken ct);

    // Higher-order function: adds logging to any OrderProcessor
    public static OrderProcessor WithLogging(OrderProcessor next, ILogger logger) =>
        async (order, ct) =>
        {
            logger.LogInformation("Processing order {OrderId}", order.Id);
            var result = await next(order, ct);
            result.Match(
                onSuccess: o => { logger.LogInformation("Order {OrderId} processed", o.Id); return Unit.Default; },
                onFailure: e => { logger.LogError("Order {OrderId} failed: {Error}", order.Id, e.Message); return Unit.Default; }
            );
            return result;
        };

    // Higher-order function: adds retry to any OrderProcessor
    public static OrderProcessor WithRetry(OrderProcessor next, int maxRetries = 3) =>
        async (order, ct) =>
        {
            for (var i = 0; i <= maxRetries; i++)
            {
                var result = await next(order, ct);
                if (result.IsSuccess || i == maxRetries) return result;
                await Task.Delay(TimeSpan.FromSeconds(Math.Pow(2, i)), ct);
            }
            throw new UnreachableException();
        };

    // Compose a pipeline from individual processors
    public static OrderProcessor BuildPipeline(
        OrderProcessor core,
        ILogger logger,
        int retryCount = 3) =>
        WithLogging(
            WithRetry(core, retryCount),
            logger
        );
}

// Partial application — fix some arguments
public static class PricingFunctions
{
    // Full function
    public static decimal CalculatePrice(decimal basePrice, decimal taxRate, decimal discountRate) =>
        basePrice * (1 - discountRate) * (1 + taxRate);

    // Partially applied — fix tax rate for a region
    public static Func<decimal, decimal, decimal> ForTaxRate(decimal taxRate) =>
        (price, discount) => CalculatePrice(price, taxRate, discount);

    // Further partial application — fix discount rate too
    public static Func<decimal, decimal> ForTaxAndDiscount(decimal taxRate, decimal discountRate) =>
        price => CalculatePrice(price, taxRate, discountRate);
}

// Usage
var usaTaxCalculator = PricingFunctions.ForTaxRate(0.08m);    // 8% tax
var euTaxCalculator = PricingFunctions.ForTaxRate(0.20m);     // 20% VAT
var goldPricing = PricingFunctions.ForTaxAndDiscount(0.08m, 0.15m); // USA + gold discount

decimal usaPrice = usaTaxCalculator(100m, 0m);   // $108.00
decimal euPrice = euTaxCalculator(100m, 0m);     // $120.00
decimal goldPrice = goldPricing(100m);           // 100 * 0.85 * 1.08 = $91.80

// Currying
public static Func<T2, TResult> Curry<T1, T2, TResult>(Func<T1, T2, TResult> func, T1 arg1) =>
    arg2 => func(arg1, arg2);
```

---

## Step 1440: Functional Patterns in ASP.NET Core

```csharp
// Functional-style minimal APIs with Result types

public static class OrderEndpoints
{
    public static IEndpointRouteBuilder MapOrderEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/v1/orders").WithTags("Orders");

        group.MapPost("/", CreateOrderAsync).WithName("CreateOrder");
        group.MapGet("/{id:guid}", GetOrderAsync).WithName("GetOrder");
        group.MapPut("/{id:guid}/confirm", ConfirmOrderAsync).WithName("ConfirmOrder");
        group.MapDelete("/{id:guid}", CancelOrderAsync).WithName("CancelOrder");

        return app;
    }

    private static async Task<IResult> CreateOrderAsync(
        CreateOrderRequest request,
        OrderApplicationService service,
        CancellationToken ct) =>
        (await service.ProcessOrderAsync(request.CustomerId, request.Items, request.Payment, ct))
            .Match(
                onSuccess: result => Results.Created($"/api/v1/orders/{result.OrderId}", result),
                onFailure: error => error.ToHttpResult()
            );

    private static async Task<IResult> GetOrderAsync(
        Guid id,
        IOrderRepository repo,
        CancellationToken ct)
    {
        var order = await repo.FindByIdAsync(id, ct);
        return Option<Order>.FromNullable(order)
            .Match(
                onSome: o => Results.Ok(o.ToDto()),
                onNone: () => Results.NotFound($"Order {id} not found")
            );
    }

    private static async Task<IResult> ConfirmOrderAsync(
        Guid id,
        OrderApplicationService service,
        CancellationToken ct) =>
        (await service.ConfirmOrderAsync(id, ct)).ToHttpResult();

    private static async Task<IResult> CancelOrderAsync(
        Guid id,
        string reason,
        OrderApplicationService service,
        CancellationToken ct) =>
        (await service.CancelOrderAsync(id, reason, ct)).ToHttpResult();
}

// Middleware as pure function composition
public static class FunctionalMiddleware
{
    public static RequestDelegate Compose(
        RequestDelegate next,
        params Func<RequestDelegate, RequestDelegate>[] middlewares) =>
        middlewares.Reverse().Aggregate(next, (current, mw) => mw(current));

    // Timing middleware as a function
    public static Func<RequestDelegate, RequestDelegate> TimingMiddleware(ILogger logger) =>
        next => async context =>
        {
            var sw = Stopwatch.StartNew();
            await next(context);
            sw.Stop();
            logger.LogInformation("{Method} {Path} completed in {Ms}ms",
                context.Request.Method, context.Request.Path, sw.ElapsedMilliseconds);
        };
}

// Value object equality via records
public record CustomerId(Guid Value)
{
    public static CustomerId New() => new(Guid.NewGuid());
    public static CustomerId Parse(string value) => new(Guid.Parse(value));
    public override string ToString() => Value.ToString();
}

public record ProductId(Guid Value)
{
    public static implicit operator Guid(ProductId id) => id.Value;
    public static implicit operator ProductId(Guid id) => new(id);
}

// Strongly-typed IDs prevent mixing CustomerId and ProductId
public record Order(CustomerId CustomerId, ProductId ProductId, Money Total);
// This won't compile — correct!
// new Order(new ProductId(Guid.NewGuid()), new CustomerId(Guid.NewGuid()), total);
```

---

## Summary: Functional Programming Patterns in C#

| Pattern | Purpose | Key Types |
|---------|---------|-----------|
| Immutable Records | Prevent mutation bugs | `record`, `ImmutableList<T>`, `init` |
| Option/Maybe | Explicit nullable handling | `Option<T>` monad |
| Result | Railway-oriented error handling | `Result<T, TError>` |
| Discriminated Unions | Type-safe case analysis | sealed record hierarchies |
| Pure Functions | Testable, cacheable logic | `static` methods, no side effects |
| Memoization | Cache expensive computations | `ConcurrentDictionary` |
| Higher-Order Functions | Composable behavior | `Func<T>`, middleware pattern |
| Partial Application | Pre-fill function arguments | Closures, `Func<>` |
| Validated<T> | Accumulate all errors | Applicative functor |
| LINQ | Declarative data pipelines | `IEnumerable<T>` |

---

*Part 51 complete — Steps 1430-1440. Topics covered: immutability with records and ImmutableList, Option/Maybe monad, Result monad (railway-oriented programming), DomainError hierarchy, converting Results to HTTP, pure functions, discriminated unions with sealed hierarchies, LINQ as functional programming, memoization, lazy evaluation, Validated<T> for error accumulation, higher-order functions, partial application, currying, functional-style Minimal APIs.*
