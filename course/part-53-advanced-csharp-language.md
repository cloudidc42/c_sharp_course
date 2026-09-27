# Part 53: Advanced C# Language Features
## Steps 1491-1520: C# 12/13/14 — Primary Constructors, Collection Expressions, Interceptors, Extensions

---

## Step 1491: Primary Constructors (C# 12)

```csharp
// Primary constructors — parameters available throughout the class body
// No need to declare fields manually when you only need them in the class

// Before C# 12
public class OrderService_Old
{
    private readonly IOrderRepository _repository;
    private readonly ILogger<OrderService_Old> _logger;

    public OrderService_Old(IOrderRepository repository, ILogger<OrderService_Old> logger)
    {
        _repository = repository;
        _logger = logger;
    }

    public async Task<Order> GetOrderAsync(Guid id)
    {
        _logger.LogInformation("Getting order {Id}", id);
        return await _repository.GetByIdAsync(id);
    }
}

// C# 12: Primary constructor — clean and concise
public class OrderService(IOrderRepository repository, ILogger<OrderService> logger)
{
    // Parameters are in scope for all instance members
    public async Task<Order> GetOrderAsync(Guid id)
    {
        logger.LogInformation("Getting order {Id}", id);
        return await repository.GetByIdAsync(id);
    }

    public async Task<Order> CreateOrderAsync(CreateOrderRequest request)
    {
        logger.LogInformation("Creating order for customer {CustomerId}", request.CustomerId);
        var order = new Order { CustomerId = request.CustomerId };
        await repository.SaveAsync(order);
        return order;
    }
}

// Primary constructor with validation
public class Money(decimal amount, string currency)
{
    // Validate at construction time
    private readonly decimal _amount = amount >= 0
        ? amount
        : throw new ArgumentException("Amount cannot be negative");

    private readonly string _currency = string.IsNullOrWhiteSpace(currency)
        ? throw new ArgumentException("Currency required")
        : currency.ToUpperInvariant();

    public decimal Amount => _amount;
    public string Currency => _currency;
}

// Struct with primary constructor
public readonly struct Point(double x, double y)
{
    public double X => x;
    public double Y => y;
    public double DistanceTo(Point other) =>
        Math.Sqrt(Math.Pow(X - other.X, 2) + Math.Pow(Y - other.Y, 2));

    public override string ToString() => $"({X:F2}, {Y:F2})";
}

// Primary constructor in record struct
public record struct Range(double Min, double Max)
{
    public bool Contains(double value) => value >= Min && value <= Max;
    public double Span => Max - Min;
}
```

---

## Step 1492: Collection Expressions (C# 12)

```csharp
// Collection expressions — unified syntax for all collection types

// Arrays
int[] squares = [1, 4, 9, 16, 25];
string[] names = ["Alice", "Bob", "Charlie"];

// Lists
List<int> evens = [2, 4, 6, 8, 10];
List<string> greetings = ["Hello", "Hi", "Hey"];

// Immutable collections
ImmutableArray<int> primes = [2, 3, 5, 7, 11, 13];
ImmutableList<string> colors = ["Red", "Green", "Blue"];

// Span<T>
Span<byte> buffer = [0x48, 0x65, 0x6C, 0x6C, 0x6F];  // "Hello"

// Spread operator (..) — inline another collection
int[] first = [1, 2, 3];
int[] second = [4, 5, 6];
int[] combined = [..first, ..second];        // [1, 2, 3, 4, 5, 6]
int[] withExtra = [0, ..first, ..second, 7]; // [0, 1, 2, 3, 4, 5, 6, 7]

// Spread with filtering
int[] oddNumbers = [..Enumerable.Range(1, 20).Where(n => n % 2 != 0)];

// In method calls
void ProcessItems(int[] items) { }
ProcessItems([1, 2, 3]);

// In LINQ
var result = new List<int> { ..evens, ..squares }
    .Where(n => n > 5)
    .ToList();

// Empty collection
string[] empty = [];
List<Order> noOrders = [];
IEnumerable<int> emptySeq = [];

// Dictionary expressions (C# 14 proposal / upcoming)
// Dictionary<string, int> scores = ["Alice": 95, "Bob": 87];

// Custom collection types that work with collection expressions
// Implement IEnumerable<T> and Add method
public class OrderCollection : IEnumerable<Order>
{
    private readonly List<Order> _orders = [];

    public void Add(Order order) => _orders.Add(order);
    public IEnumerator<Order> GetEnumerator() => _orders.GetEnumerator();
    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}

OrderCollection orders = [
    new Order { Id = Guid.NewGuid() },
    new Order { Id = Guid.NewGuid() }
];
```

---

## Step 1493: ref readonly Parameters and Scoped Refs (C# 12)

```csharp
// ref readonly — pass large struct by reference without allowing modification

public readonly struct Matrix4x4
{
    public readonly float M11, M12, M13, M14;
    public readonly float M21, M22, M23, M24;
    public readonly float M31, M32, M33, M34;
    public readonly float M41, M42, M43, M44;
    // 16 floats = 64 bytes — expensive to copy
}

// ref readonly avoids the copy while preventing mutation
public static Matrix4x4 Multiply(ref readonly Matrix4x4 a, ref readonly Matrix4x4 b)
{
    // a and b are passed by reference, cannot be modified
    return new Matrix4x4
    {
        M11 = a.M11 * b.M11 + a.M12 * b.M21 + a.M13 * b.M31 + a.M14 * b.M41,
        // ... etc
    };
}

// scoped ref — ref cannot escape the scope
public static ref int FindFirst(scoped ref int[] array, int value)
{
    // The returned ref cannot escape beyond this method
    for (var i = 0; i < array.Length; i++)
        if (array[i] == value)
            return ref array[i];
    throw new InvalidOperationException("Not found");
}

// Inline arrays (C# 12) — fixed-size arrays on stack
[InlineArray(16)]
public struct Buffer16<T>
{
    private T _element;
}

// Usage: zero-allocation fixed buffer
var buffer = new Buffer16<float>();
buffer[0] = 1.0f;
buffer[1] = 2.0f;
// Works with Span<T>
Span<float> span = buffer;
```

---

## Step 1494: Pattern Matching Enhancements (C# 9-12)

```csharp
// Pattern matching — comprehensive examples

// List patterns (C# 11)
int[] CheckList(int[] numbers) => numbers switch
{
    []          => throw new ArgumentException("Empty"),
    [var single]        => [single * 2],          // exactly one element
    [var first, .. var rest] => [first, ..rest],  // first + rest
    [var first, var second] => [first + second],  // exactly two
    [>0, >0, >0] => numbers,                       // all positive
    _ => throw new ArgumentException("Unexpected")
};

// "Hello, World!".Split(", ") == ["Hello", "World!"]
static bool IsHelloWorld(string[] parts) => parts is ["Hello", "World!"];

// Null-handling in patterns
public string DescribeObject(object? obj) => obj switch
{
    null => "null",
    int i when i < 0 => $"negative int: {i}",
    int i => $"int: {i}",
    string { Length: 0 } => "empty string",
    string s => $"string: {s}",
    IEnumerable<int> { } items when !items.Any() => "empty sequence",
    IEnumerable<int> items => $"sequence with {items.Count()} items",
    _ => obj.GetType().Name
};

// Extended property patterns (C# 10)
public decimal GetShippingCost(Order order) => order switch
{
    // Nested property pattern
    { Customer.Tier: CustomerTier.Gold, Total: > 100m } => 0m,
    { Customer.Address.Country: "US", Total: > 50m } => 5m,
    { Customer.Address.Country: "US" } => 10m,
    { Customer.Address.Country: "CA" } => 15m,
    _ => 25m
};

// Pattern matching with positional deconstruction
public record Coordinate(double Lat, double Lon);

public static string GetRegion(Coordinate coord) => coord switch
{
    (>= -90 and <= 90, >= -180 and < -100) => "Americas West",
    (>= -90 and <= 90, >= -100 and < -50) => "Americas East",
    (>= -90 and <= 90, >= -50 and < 50) => "Europe/Africa",
    (>= -90 and <= 90, >= 50 and <= 180) => "Asia/Pacific",
    _ => "Unknown"
};

// Type pattern with guard
public static decimal CalculateTax(object payment) => payment switch
{
    CreditCard { CardNumber: { Length: 16 } } cc => 0.029m,
    CreditCard cc when cc.IsBusinessCard => 0.035m,
    BankTransfer bt when bt.AccountNumber.StartsWith("US") => 0m,
    PayPal { Email: var email } when email.EndsWith(".org") => 0.01m,
    _ => 0.025m
};
```

---

## Step 1495: Required Members (C# 11)

```csharp
// required keyword — must be initialized at object construction

public class OrderRequest
{
    public required Guid CustomerId { get; init; }
    public required List<CartItem> Items { get; init; }
    public required Address ShippingAddress { get; init; }
    public string? Notes { get; init; } // Optional
    public PaymentMethod? PaymentMethod { get; init; } // Optional

    // Compiler error if required members not set:
    // var r = new OrderRequest(); // Error: CustomerId, Items, ShippingAddress required
    // var r = new OrderRequest { CustomerId = Guid.NewGuid() }; // Error: Items, ShippingAddress required
}

// Correct usage
var request = new OrderRequest
{
    CustomerId = Guid.NewGuid(),  // required ✅
    Items = [new CartItem(Guid.NewGuid(), 1, 10m)],  // required ✅
    ShippingAddress = new Address("123 Main St", "NYC", "10001", "US")  // required ✅
};

// With [SetsRequiredMembers] — constructor that sets all required members
public class User
{
    public required string Email { get; init; }
    public required string Name { get; init; }
    public Guid Id { get; init; } = Guid.NewGuid();

    [SetsRequiredMembers]
    public User(string email, string name)
    {
        Email = email;
        Name = name;
    }
}

// Now this works (constructor satisfies required)
var user = new User("test@example.com", "Alice");
// But this still requires setting Email and Name:
// var u = new User { Email = "test@example.com" }; // Error: Name required
```

---

## Step 1496: Generic Math (C# 11)

```csharp
// Generic math interfaces — write algorithms once for all numeric types

// Sum works with int, long, float, double, decimal — any INumber<T>
public static T Sum<T>(IEnumerable<T> values) where T : INumber<T>
{
    var total = T.Zero;
    foreach (var value in values)
        total += value;
    return total;
}

// Average for any numeric type
public static T Average<T>(IEnumerable<T> values) where T : INumber<T>
{
    var list = values.ToList();
    if (list.Count == 0) throw new InvalidOperationException("Empty sequence");
    return Sum(list) / T.CreateChecked(list.Count);
}

// Clamp — works with anything that implements IComparable<T>
public static T Clamp<T>(T value, T min, T max) where T : INumber<T> =>
    T.Clamp(value, min, max);

// Statistics over any numeric type
public static class Statistics<T> where T : INumber<T>, IFloatingPoint<T>
{
    public static T StandardDeviation(IEnumerable<T> values)
    {
        var list = values.ToList();
        var avg = Average(list);
        var sumOfSquaredDiffs = list.Sum(v => (v - avg) * (v - avg));
        return T.Sqrt(sumOfSquaredDiffs / T.CreateChecked(list.Count));
    }

    private static T Sum(IEnumerable<T> values)
    {
        var total = T.Zero;
        foreach (var v in values) total += v;
        return total;
    }

    private static T Average(List<T> list) =>
        Sum(list) / T.CreateChecked(list.Count);
}

// Usage with different types
double doubleSum = Sum([1.0, 2.0, 3.0, 4.0]);    // 10.0
decimal decimalSum = Sum([1m, 2m, 3m, 4m]);       // 10
int intSum = Sum([1, 2, 3, 4]);                    // 10

// Generic vector operations
public static T DotProduct<T>(T[] a, T[] b) where T : INumber<T>
{
    if (a.Length != b.Length) throw new ArgumentException("Vectors must be same length");
    var sum = T.Zero;
    for (var i = 0; i < a.Length; i++)
        sum += a[i] * b[i];
    return sum;
}
```

---

## Step 1497: Raw String Literals (C# 11)

```csharp
// Raw string literals — no escaping, preserves whitespace exactly

// Multi-line JSON without escape characters
var json = """
    {
        "name": "Alice",
        "email": "alice@example.com",
        "address": {
            "street": "123 Main St",
            "city": "New York"
        }
    }
    """;

// SQL query without @"" workarounds
var sql = """
    SELECT
        o.id,
        o.order_number,
        c.email AS customer_email,
        SUM(oi.quantity * oi.unit_price) AS total
    FROM orders o
    JOIN customers c ON c.id = o.customer_id
    JOIN order_items oi ON oi.order_id = o.id
    WHERE o.status = 'pending'
      AND o.created_at >= @startDate
    GROUP BY o.id, o.order_number, c.email
    ORDER BY total DESC
    LIMIT @pageSize OFFSET @offset;
    """;

// XML without escaping
var xml = """
    <?xml version="1.0" encoding="utf-8"?>
    <Order xmlns="http://schemas.example.com/orders">
        <Id>{orderId}</Id>
        <Status>Pending</Status>
    </Order>
    """;

// Regex without double-escaping
var emailPattern = """^[\w\.-]+@[\w\.-]+\.\w{2,}$""";

// Interpolated raw string literals
var customerId = Guid.NewGuid();
var orderJson = $"""
    {{
        "customerId": "{customerId}",
        "items": [
            {{"productId": "prod-1", "quantity": 2}},
            {{"productId": "prod-2", "quantity": 1}}
        ]
    }}
    """;
// Note: {{ and }} are literal braces in interpolated raw strings

// $$ for multiple dollar signs (allows {variable} inside)
var template = $$"""
    Hello, {{customerName}}!
    Your order #{{orderNumber}} has been confirmed.
    Total: {{total:C}}
    """;
```

---

## Step 1498: Interceptors (C# 12, Experimental)

```csharp
// Interceptors — intercept specific method calls via source generators
// Enabled in .NET 8+ with [InterceptsLocation]

// Use case: Compile-time optimizations (e.g. EF Core compiled queries)
// This is what EF Core uses internally for query compilation

// Example: intercepting a specific method call
[InterceptsLocation("OrderService.cs", 15, 25)]  // file, line, column
public static Order InterceptedGetOrder(Guid id)
{
    // Optimized implementation called at that specific call site
    return new Order();  // In reality, uses compiled query
}

// More practical use: source generators that generate interceptors
// EF Core 8 compiled queries use this pattern:
public partial class AppDbContext : DbContext
{
    // This generates an interceptor at compile time
    [DbContextCompiledQuery]
    private static readonly Func<AppDbContext, Guid, Task<Order?>> GetOrderById =
        EF.CompileAsyncQuery((AppDbContext db, Guid id) =>
            db.Orders.Include(o => o.Items).FirstOrDefault(o => o.Id == id));

    public Task<Order?> FindOrderAsync(Guid id) => GetOrderById(this, id);
}
```

---

## Step 1499: Params Collections (C# 13)

```csharp
// params now works with any collection type, not just arrays

// C# 12 and earlier — only arrays
public void ProcessIds_Old(params Guid[] ids)
{
    foreach (var id in ids) Console.WriteLine(id);
}

// C# 13 — works with Span<T>, IEnumerable<T>, List<T>, etc.
public void ProcessIds(params ReadOnlySpan<Guid> ids)
{
    // Zero allocation when called with inline values
    foreach (var id in ids) Console.WriteLine(id);
}

public void ProcessOrders(params IEnumerable<Order> orders)
{
    foreach (var order in orders) Console.WriteLine(order.Id);
}

// Zero-allocation call with stack allocation
ProcessIds(Guid.NewGuid(), Guid.NewGuid(), Guid.NewGuid());

// Can still pass a collection
Guid[] ids = [Guid.NewGuid(), Guid.NewGuid()];
ProcessIds(ids);

// Works with custom types
public static T Max<T>(params ReadOnlySpan<T> values) where T : INumber<T>
{
    if (values.IsEmpty) throw new ArgumentException("No values");
    var max = values[0];
    foreach (var value in values[1..])
        if (value > max) max = value;
    return max;
}

var maximum = Max(3, 1, 4, 1, 5, 9, 2, 6);  // 9 — no array allocation
```

---

## Step 1500: Lock Statement Object (C# 13)

```csharp
// New System.Threading.Lock type — replaces object-based locks
// Better performance: no monitor overhead for uncontested locks

public class ThreadSafeCounter
{
    private int _count;
    private readonly Lock _lock = new();  // New Lock type (C# 13)

    public void Increment()
    {
        lock (_lock)  // Uses Lock.Scope under the hood — faster
        {
            _count++;
        }
    }

    public int GetCount()
    {
        lock (_lock)
        {
            return _count;
        }
    }

    // Or use the Scope API directly
    public void IncrementWithScope()
    {
        using (_lock.EnterScope())
        {
            _count++;
        }
    }
}

// Before C# 13 — had to use object
public class OldThreadSafeCounter
{
    private int _count;
    private readonly object _lockObj = new();  // Wasteful — allocates object header + overhead

    public void Increment()
    {
        lock (_lockObj)  // Monitor.Enter/Exit — heavier
        {
            _count++;
        }
    }
}
```

---

## Step 1501: Extensions (C# 14, Preview)

```csharp
// Extension members — upcoming C# 14 feature
// More powerful than extension methods — can add properties, static members

// Current approach (extension methods)
public static class OrderExtensions
{
    public static bool IsOverdue(this Order order) =>
        order.Status == OrderStatus.Pending &&
        order.CreatedAt < DateTimeOffset.UtcNow.AddDays(-3);

    public static string GetStatusDisplay(this Order order) =>
        order.Status switch
        {
            OrderStatus.Pending => "⏳ Pending",
            OrderStatus.Confirmed => "✅ Confirmed",
            OrderStatus.Shipped => "🚚 Shipped",
            OrderStatus.Delivered => "📦 Delivered",
            OrderStatus.Cancelled => "❌ Cancelled",
            _ => order.Status.ToString()
        };
}

// C# 14 preview: extension blocks with properties and statics
extension OrderExtensions for Order
{
    // Extension property (not possible before C# 14)
    public bool IsOverdue =>
        Status == OrderStatus.Pending &&
        CreatedAt < DateTimeOffset.UtcNow.AddDays(-3);

    public string StatusDisplay => Status switch
    {
        OrderStatus.Pending => "⏳ Pending",
        OrderStatus.Confirmed => "✅ Confirmed",
        OrderStatus.Shipped => "🚚 Shipped",
        _ => Status.ToString()
    };

    // Static extension member
    public static Order CreateTest() =>
        new Order { Id = Guid.NewGuid(), Status = OrderStatus.Pending };
}

// Usage
var order = new Order { /* ... */ };
if (order.IsOverdue) { /* ... */ }
Console.WriteLine(order.StatusDisplay);
var testOrder = Order.CreateTest();
```

---

## Step 1502: Nullable Reference Types — Advanced Patterns

```csharp
// Nullable annotations — advanced usage

// NotNullWhen — true means parameter is not null when return is true
public bool TryGetOrder(Guid id, [NotNullWhen(true)] out Order? order)
{
    order = _cache.TryGetValue(id, out var cached) ? cached : null;
    return order is not null;
}

// Usage — compiler knows order is not null after this
if (TryGetOrder(id, out var order))
    Console.WriteLine(order.Total);  // No null warning

// MaybeNullWhen — could be null when return is false
public bool TryParse(string input, [MaybeNullWhen(false)] out Order result)
{
    // Returns false if parsing fails, result is null in that case
    result = null;
    if (string.IsNullOrWhiteSpace(input)) return false;
    result = JsonSerializer.Deserialize<Order>(input);
    return result is not null;
}

// DisallowNull — null not accepted even though type is nullable
public void SetCustomer([DisallowNull] Customer? customer)
{
    // Ensures callers don't pass null, but internal field might be null
    _customer = customer;
}

// AllowNull — allows null even though type is not nullable
public string Name
{
    get => _name ?? string.Empty;
    [param: AllowNull] set => _name = value;  // Allows null assignment internally
}

// Conditional nullable — field can be null before some operation
public class OrderProcessor
{
    private Order? _currentOrder;

    [MemberNotNull(nameof(_currentOrder))]
    public void Initialize(Order order)
    {
        _currentOrder = order ?? throw new ArgumentNullException(nameof(order));
    }

    public void Process()
    {
        // Compiler error if Initialize() not called first
        // Because _currentOrder might be null
        Initialize(new Order());

        // Now safe — MemberNotNull guarantees _currentOrder is not null
        Console.WriteLine(_currentOrder.Total);
    }
}

// NotNull — asserts value is not null after the call
public static void ThrowIfNull([NotNull] object? value, string paramName)
{
    if (value is null) throw new ArgumentNullException(paramName);
}

// DoesNotReturnIf — code after this call is unreachable if condition is true
[DoesNotReturn]
public static void Fail(string message) => throw new InvalidOperationException(message);

public static int Divide(int a, int b)
{
    if (b == 0) Fail("Division by zero");
    return a / b;  // No CS8602 — compiler knows Fail doesn't return
}
```

---

## Step 1503: Unsafe Code and Fixed-Size Buffers

```csharp
// Unsafe code — when you need raw pointer access

// Fixed-size buffer in struct (stack-allocated)
public unsafe struct NetworkPacket
{
    public byte Version;
    public byte Type;
    public ushort Length;
    public fixed byte Data[1024];  // Fixed-size inline array

    public ReadOnlySpan<byte> GetData() =>
        MemoryMarshal.CreateReadOnlySpan(ref Data[0], Length);
}

// Unsafe image processing
public static unsafe void FlipHorizontal(byte[] pixels, int width, int height, int stride)
{
    fixed (byte* ptr = pixels)
    {
        for (var y = 0; y < height; y++)
        {
            var rowStart = ptr + y * stride;
            var left = rowStart;
            var right = rowStart + (width - 1) * 4;  // 4 bytes per pixel (BGRA)

            while (left < right)
            {
                // Swap pixels
                var tmp0 = *left;     var tmp1 = *(left + 1);
                var tmp2 = *(left + 2); var tmp3 = *(left + 3);
                *left = *right;       *(left + 1) = *(right + 1);
                *(left + 2) = *(right + 2); *(left + 3) = *(right + 3);
                *right = tmp0;        *(right + 1) = tmp1;
                *(right + 2) = tmp2;  *(right + 3) = tmp3;

                left += 4;
                right -= 4;
            }
        }
    }
}

// Pointer arithmetic for parsing binary protocols
public static unsafe BinaryMessage ParseBinaryMessage(ReadOnlySpan<byte> data)
{
    if (data.Length < 8) throw new ArgumentException("Too short");

    fixed (byte* ptr = data)
    {
        var type = *(ushort*)ptr;
        var length = *(int*)(ptr + 2);
        var payload = new ReadOnlySpan<byte>(ptr + 6, length);
        return new BinaryMessage(type, payload.ToArray());
    }
}

// NativeMemory — allocate unmanaged memory directly
public class UnmanagedBuffer : IDisposable
{
    private unsafe void* _ptr;
    private readonly int _size;
    private bool _disposed;

    public unsafe UnmanagedBuffer(int size)
    {
        _size = size;
        _ptr = NativeMemory.Alloc((nuint)size);
        NativeMemory.Clear(_ptr, (nuint)size);
    }

    public unsafe Span<byte> AsSpan()
    {
        ObjectDisposedException.ThrowIf(_disposed, this);
        return new Span<byte>(_ptr, _size);
    }

    public unsafe void Dispose()
    {
        if (!_disposed)
        {
            NativeMemory.Free(_ptr);
            _ptr = null;
            _disposed = true;
        }
    }
}
```

---

## Step 1504: Source Generators — Custom Generator

```csharp
// Build a simple source generator that generates ToString() for records

// Generator project (separate project with Analyzer SDK)
[Generator]
public class ToStringGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        // Find all classes with [GenerateToString] attribute
        var classDeclarations = context.SyntaxProvider
            .ForAttributeWithMetadataName(
                "GenerateToStringAttribute",
                predicate: (node, _) => node is ClassDeclarationSyntax,
                transform: (ctx, _) => GetClassInfo(ctx)
            )
            .Where(info => info is not null);

        context.RegisterSourceOutput(classDeclarations, GenerateToString);
    }

    private ClassInfo? GetClassInfo(GeneratorAttributeSyntaxContext ctx)
    {
        if (ctx.TargetSymbol is not INamedTypeSymbol classSymbol) return null;

        var properties = classSymbol.GetMembers()
            .OfType<IPropertySymbol>()
            .Where(p => p.DeclaredAccessibility == Accessibility.Public)
            .Select(p => (p.Name, p.Type.Name))
            .ToList();

        return new ClassInfo(
            classSymbol.Name,
            classSymbol.ContainingNamespace.ToDisplayString(),
            properties
        );
    }

    private void GenerateToString(SourceProductionContext ctx, ClassInfo? info)
    {
        if (info is null) return;

        var propertyParts = string.Join(", ",
            info.Properties.Select(p => $"{p.Name}={{{p.Name}}}"));

        var source = $$"""
            namespace {{info.Namespace}};

            partial class {{info.ClassName}}
            {
                public override string ToString() =>
                    $"{{info.ClassName}} { {{propertyParts}} }";
            }
            """;

        ctx.AddSource($"{info.ClassName}.g.cs", source);
    }

    private record ClassInfo(
        string ClassName,
        string Namespace,
        List<(string Name, string TypeName)> Properties
    );
}

// Usage in user code
[GenerateToString]
public partial class Product
{
    public Guid Id { get; init; }
    public string Name { get; init; } = string.Empty;
    public decimal Price { get; init; }
}

// Generated code:
// public override string ToString() =>
//     $"Product { Id={Id}, Name={Name}, Price={Price} }";
```

---

## Step 1505: Caller Argument Expressions (C# 10)

```csharp
// CallerArgumentExpression — capture the expression used as an argument

public static void ThrowIfNull<T>(
    T? value,
    [CallerArgumentExpression(nameof(value))] string? paramName = null)
{
    if (value is null)
        throw new ArgumentNullException(paramName, $"Value '{paramName}' cannot be null");
}

// Usage
string? name = null;
ThrowIfNull(name);  // Throws: ArgumentNullException: Value 'name' cannot be null

// Assertion helper with expression in error message
public static void Assert(
    bool condition,
    [CallerArgumentExpression(nameof(condition))] string? expression = null,
    [CallerFilePath] string? file = null,
    [CallerLineNumber] int line = 0)
{
    if (!condition)
        throw new AssertionException(
            $"Assertion failed: '{expression}' at {Path.GetFileName(file)}:{line}");
}

// Usage
var order = GetOrder();
Assert(order.Total > 0);        // "Assertion failed: 'order.Total > 0'"
Assert(order.Status == OrderStatus.Pending);  // Full expression in message

// Fluent API with expression capture
public class FluentValidator<T>(T subject,
    [CallerArgumentExpression(nameof(subject))] string subjectExpression = "")
{
    public FluentValidator<T> Is(
        Func<T, bool> predicate,
        [CallerArgumentExpression(nameof(predicate))] string predicateExpression = "")
    {
        if (!predicate(subject))
            throw new ValidationException(
                $"Validation failed: '{subjectExpression}' does not satisfy '{predicateExpression}'");
        return this;
    }
}

// Usage: error messages include actual code expressions
var validator = new FluentValidator<Order>(order);
validator
    .Is(o => o.Total > 0)        // "order does not satisfy 'o => o.Total > 0'"
    .Is(o => o.Items.Count > 0);
```

---

## Step 1506: Partial Methods and Classes — Advanced

```csharp
// Partial classes — split across files (common with source generators)

// OrderService.cs — your code
public partial class OrderService
{
    private readonly IOrderRepository _repository;
    private readonly ILogger<OrderService> _logger;

    // Partial method declaration — implementation generated elsewhere
    partial void OnOrderCreated(Order order);
    partial void OnOrderStatusChanged(Order order, OrderStatus oldStatus);
}

// OrderService.Notifications.cs — notification implementation
public partial class OrderService
{
    partial void OnOrderCreated(Order order)
    {
        _logger.LogInformation("Order {Id} created", order.Id);
    }

    partial void OnOrderStatusChanged(Order order, OrderStatus oldStatus)
    {
        _logger.LogInformation("Order {Id} status: {Old} → {New}", order.Id, oldStatus, order.Status);
    }
}

// Returning partial methods (C# 9) — can have return types and out params
public partial class OrderProcessor
{
    // Partial method with return type — must be implemented
    private partial decimal CalculateDiscount(Order order, Customer customer);
    private partial bool ShouldApplyLoyaltyBonus(Customer customer);
}

public partial class OrderProcessor
{
    private partial decimal CalculateDiscount(Order order, Customer customer) =>
        customer.Tier switch
        {
            CustomerTier.Gold => order.Total * 0.15m,
            CustomerTier.Silver => order.Total * 0.10m,
            _ => 0m
        };

    private partial bool ShouldApplyLoyaltyBonus(Customer customer) =>
        customer.OrderCount > 10;
}

// Partial properties (C# 13)
public partial class UserProfile
{
    // Declaration in generated code
    public partial string DisplayName { get; set; }
}

public partial class UserProfile
{
    // Implementation
    private string _displayName = string.Empty;

    public partial string DisplayName
    {
        get => _displayName;
        set => _displayName = value.Trim();
    }
}
```

---

## Step 1507: Performance Analysis with Roslyn Analyzers

```csharp
// Custom Roslyn analyzer — detect boxing in hot paths

[DiagnosticAnalyzer(LanguageNames.CSharp)]
public class BoxingAnalyzer : DiagnosticAnalyzer
{
    private static readonly DiagnosticDescriptor Rule = new(
        id: "PERF001",
        title: "Avoid boxing in hot paths",
        messageFormat: "Boxing value type '{0}' — consider generic overload",
        category: "Performance",
        defaultSeverity: DiagnosticSeverity.Warning,
        isEnabledByDefault: true
    );

    public override ImmutableArray<DiagnosticDescriptor> SupportedDiagnostics =>
        ImmutableArray.Create(Rule);

    public override void Initialize(AnalysisContext context)
    {
        context.ConfigureGeneratedCodeAnalysis(GeneratedCodeAnalysisFlags.None);
        context.EnableConcurrentExecution();
        context.RegisterOperationAction(AnalyzeConversion, OperationKind.Conversion);
    }

    private void AnalyzeConversion(OperationAnalysisContext context)
    {
        var conversion = (IConversionOperation)context.Operation;

        // Check if this is a boxing conversion
        if (!conversion.IsBoxing) return;

        // Only warn if the type being boxed is in a [HotPath] method
        var containingMethod = context.ContainingSymbol as IMethodSymbol;
        if (containingMethod?.HasAttribute("HotPathAttribute") != true) return;

        var valueType = conversion.Operand.Type;
        context.ReportDiagnostic(Diagnostic.Create(Rule, conversion.Syntax.GetLocation(), valueType));
    }
}

// Suppressor — suppress specific warnings in specific contexts
[DiagnosticAnalyzer(LanguageNames.CSharp)]
public class NullableSuppressor : DiagnosticSuppressor
{
    private static readonly SuppressionDescriptor Rule = new(
        id: "SP001",
        suppressedDiagnosticId: "CS8602",
        justification: "Null checked in preceding using statement"
    );

    public override ImmutableArray<SuppressionDescriptor> SupportedSuppressions =>
        ImmutableArray.Create(Rule);

    public override void ReportSuppressions(SuppressionAnalysisContext context)
    {
        // Suppress CS8602 when the nullable was checked in a Using/Guard clause
        foreach (var diagnostic in context.ReportedDiagnostics)
        {
            // Analyze context and suppress if appropriate
        }
    }
}
```

---

## Step 1508: C# 13 — Overload Resolution Priority

```csharp
// OverloadResolutionPriority — control which overload is preferred

public class SpanProcessor
{
    // Preferred overload for performance — no allocation
    [OverloadResolutionPriority(1)]
    public static int Count(ReadOnlySpan<int> items)
    {
        var count = 0;
        foreach (var item in items)
            if (item > 0) count++;
        return count;
    }

    // Fallback for collections
    public static int Count(IEnumerable<int> items) =>
        items.Count(i => i > 0);
}

// Usage — compiler prefers the Span overload
int[] array = [1, -2, 3, -4, 5];
var count = SpanProcessor.Count(array);  // Uses ReadOnlySpan<int> overload (no boxing)

// Useful for library authors migrating APIs without breaking changes
public static class StringHelpers
{
    [OverloadResolutionPriority(1)]
    public static bool IsEmpty(ReadOnlySpan<char> value) => value.IsEmpty;

    public static bool IsEmpty(string? value) => string.IsNullOrEmpty(value);
}
```

---

## Step 1509: async/await Advanced Patterns

```csharp
// ConfigureAwait — when to use and when not to

// Library code: always ConfigureAwait(false) — no SynchronizationContext dependency
public class OrderRepository
{
    public async Task<Order?> GetByIdAsync(Guid id, CancellationToken ct = default)
    {
        var data = await _db.Orders
            .FirstOrDefaultAsync(o => o.Id == id, ct)
            .ConfigureAwait(false);  // Don't capture context in libraries

        return data?.ToDomain();
    }
}

// Application code: ConfigureAwait(false) when you don't need UI thread
public class BackgroundSyncService
{
    public async Task SyncAsync(CancellationToken ct = default)
    {
        var orders = await _apiService.GetOrdersAsync(ct).ConfigureAwait(false);
        await _localDb.UpsertRangeAsync(orders, ct).ConfigureAwait(false);
    }
}

// IAsyncDisposable — proper async cleanup
public class DatabaseSession : IAsyncDisposable
{
    private readonly NpgsqlConnection _connection;
    private readonly NpgsqlTransaction _transaction;
    private bool _committed;

    public DatabaseSession(NpgsqlConnection connection, NpgsqlTransaction transaction)
    {
        _connection = connection;
        _transaction = transaction;
    }

    public async Task CommitAsync(CancellationToken ct = default)
    {
        await _transaction.CommitAsync(ct).ConfigureAwait(false);
        _committed = true;
    }

    public async ValueTask DisposeAsync()
    {
        if (!_committed)
            await _transaction.RollbackAsync().ConfigureAwait(false);

        await _transaction.DisposeAsync();
        await _connection.DisposeAsync();
    }
}

// await using — async disposal
public async Task ProcessOrderAsync(Guid orderId)
{
    await using var session = await _db.BeginSessionAsync();
    try
    {
        var order = await _repo.GetAsync(orderId);
        order.Confirm();
        await _repo.SaveAsync(order);
        await session.CommitAsync();
    }
    catch
    {
        // session.DisposeAsync() calls RollbackAsync()
        throw;
    }
}

// Task.WhenAll with results
public async Task<(List<Product> products, Customer customer)> LoadOrderDataAsync(Guid orderId, Guid customerId)
{
    var productsTask = _productRepo.GetByOrderAsync(orderId);
    var customerTask = _customerRepo.GetByIdAsync(customerId);

    // Run both in parallel
    await Task.WhenAll(productsTask, customerTask);

    return (await productsTask, await customerTask);
}

// CancellationToken composition
public async Task ProcessWithTimeoutAsync(Guid orderId, CancellationToken userCt)
{
    // Combine user cancellation with 30-second timeout
    using var cts = CancellationTokenSource.CreateLinkedTokenSource(userCt);
    cts.CancelAfter(TimeSpan.FromSeconds(30));

    await ProcessOrderAsync(orderId, cts.Token);
}
```

---

## Step 1510: Summary — C# Language Version Guide

```
C# Version → .NET Version → Key Features
═══════════════════════════════════════════════════════════════

C# 10 (.NET 6)
  • Global using directives
  • File-scoped namespaces
  • Record structs
  • Extended property patterns ({ Person.Name: "Alice" })
  • CallerArgumentExpression
  • Constant string interpolation

C# 11 (.NET 7)
  • Raw string literals (""" ... """)
  • Required members
  • Generic math (INumber<T>, IFloatingPoint<T>)
  • List patterns ([first, ..rest])
  • Static abstract members in interfaces
  • UTF-8 string literals ("hello"u8)
  • Checked operators

C# 12 (.NET 8)
  • Primary constructors
  • Collection expressions ([1, 2, 3])
  • Spread operator ([..a, ..b])
  • Inline arrays ([InlineArray(N)])
  • ref readonly parameters
  • Experimental interceptors
  • Default lambda parameters

C# 13 (.NET 9)
  • params ReadOnlySpan<T>
  • System.Threading.Lock
  • Partial properties
  • Overload resolution priority
  • Escape sequences (\e, \a)
  • Extension! members (preview)

C# 14 (.NET 10, preview)
  • Extension blocks
  • Params array improvements
  • Field keyword in auto-properties
  • Null-conditional assignment (?=)
```

---

*Part 53 complete — Steps 1491-1510. Topics covered: primary constructors (C# 12), collection expressions with spread operator, ref readonly parameters, inline arrays, pattern matching enhancements (list patterns, extended property patterns), required members, generic math (INumber<T>), raw string literals, interceptors, params collections (C# 13), System.Threading.Lock, extension members (C# 14 preview), advanced nullable annotations (NotNullWhen, MaybeNullWhen, MemberNotNull), unsafe code and NativeMemory, custom source generators (IIncrementalGenerator), CallerArgumentExpression, partial methods/properties, custom Roslyn analyzers, OverloadResolutionPriority, async/await advanced patterns (ConfigureAwait, IAsyncDisposable, CancellationToken composition).*
