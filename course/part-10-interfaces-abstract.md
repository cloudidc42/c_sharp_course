# Part 10: Interface และ Abstract Class
## Steps 251-290: การออกแบบ Contracts และ Abstractions

---

## Step 251: Interface คืออะไร

```csharp
// ============ Interface ============
// Contract: กำหนดว่า class ต้องมีอะไร โดยไม่บอกว่าทำอย่างไร

// Basic interface
interface IAnimal
{
    string Name { get; }
    string Sound { get; }
    void MakeSound();
}

// Interface กับ default implementation (C# 8+)
interface ILoggable
{
    string LogPrefix { get; }
    
    // Default implementation
    void Log(string message)
    {
        Console.WriteLine($"[{LogPrefix}] [{DateTime.Now:HH:mm:ss}] {message}");
    }
    
    void LogError(string message) => Log($"ERROR: {message}");
    void LogInfo(string message) => Log($"INFO: {message}");
}

// Interface กับ static members (C# 11)
interface ICreatable<T>
{
    static abstract T Create();
    static virtual string TypeName => typeof(T).Name;
}

// Implementing interface
class Dog : IAnimal, ILoggable
{
    public string Name { get; }
    public string Sound => "Woof";
    public string LogPrefix => "Dog";
    
    public Dog(string name) => Name = name;
    
    public void MakeSound()
    {
        Log($"{Name} says {Sound}!");
    }
}

class Cat : IAnimal, ILoggable
{
    public string Name { get; }
    public string Sound => "Meow";
    public string LogPrefix => "Cat";
    
    public Cat(string name) => Name = name;
    
    public void MakeSound()
    {
        Log($"{Name} says {Sound}!");
    }
    
    // Override default implementation
    public void Log(string message)
    {
        Console.WriteLine($"🐱 [{Name}] {message}");
    }
}

// ============ Using Interfaces ============
IAnimal dog = new Dog("Rex");
IAnimal cat = new Cat("Whiskers");

dog.MakeSound();
cat.MakeSound();

// Polymorphism through interface
var animals = new List<IAnimal> { dog, cat, new Dog("Buddy"), new Cat("Luna") };
foreach (IAnimal animal in animals)
{
    Console.Write($"{animal.Name}: ");
    animal.MakeSound();
}

// Check interface
if (dog is ILoggable loggable)
{
    loggable.LogInfo("Dog is also loggable");
    loggable.LogError("Something went wrong");
}

// ============ Interface Segregation ============
// แยก interface ให้เล็กและเฉพาะเจาะจง (ISP - SOLID)

interface IReadable<T>
{
    T Read(int id);
    IEnumerable<T> ReadAll();
}

interface IWritable<T>
{
    void Create(T item);
    void Update(T item);
}

interface IDeletable
{
    bool Delete(int id);
}

// Compose interfaces
interface IRepository<T> : IReadable<T>, IWritable<T>, IDeletable { }

// Read-only repository
interface IReadOnlyRepository<T> : IReadable<T> { }
```

---

## Step 252: Multiple Interface Implementation

```csharp
// ============ Multiple Interfaces ============

interface IComparable2<T>
{
    int CompareTo(T other);
    bool IsGreaterThan(T other) => CompareTo(other) > 0;
    bool IsLessThan(T other) => CompareTo(other) < 0;
}

interface IFormattable2
{
    string Format(string? format = null);
}

interface ICloneable2<T>
{
    T Clone();
}

interface IValidatable
{
    bool IsValid();
    IEnumerable<string> GetValidationErrors();
}

class Temperature : IComparable2<Temperature>, IFormattable2, ICloneable2<Temperature>, IValidatable
{
    public double Celsius { get; private set; }
    
    public Temperature(double celsius) => Celsius = celsius;
    
    public double Fahrenheit => Celsius * 9 / 5 + 32;
    public double Kelvin => Celsius + 273.15;
    
    // IComparable2<Temperature>
    public int CompareTo(Temperature other) => Celsius.CompareTo(other.Celsius);
    
    // IFormattable2
    public string Format(string? format = null) => format switch
    {
        "F" => $"{Fahrenheit:F1}°F",
        "K" => $"{Kelvin:F1}K",
        "all" => $"{Celsius:F1}°C / {Fahrenheit:F1}°F / {Kelvin:F1}K",
        _ => $"{Celsius:F1}°C"
    };
    
    // ICloneable2<Temperature>
    public Temperature Clone() => new Temperature(Celsius);
    
    // IValidatable
    public bool IsValid() => Celsius >= -273.15; // Absolute zero
    
    public IEnumerable<string> GetValidationErrors()
    {
        if (Celsius < -273.15)
            yield return "Temperature below absolute zero";
    }
    
    public override string ToString() => Format();
    
    // Operators
    public static bool operator >(Temperature a, Temperature b) => a.CompareTo(b) > 0;
    public static bool operator <(Temperature a, Temperature b) => a.CompareTo(b) < 0;
    public static bool operator ==(Temperature a, Temperature b) => a.Celsius == b.Celsius;
    public static bool operator !=(Temperature a, Temperature b) => a.Celsius != b.Celsius;
    public override bool Equals(object? obj) => obj is Temperature t && Celsius == t.Celsius;
    public override int GetHashCode() => Celsius.GetHashCode();
}

var temps = new[]
{
    new Temperature(100), // Boiling
    new Temperature(0),   // Freezing
    new Temperature(37),  // Body temp
    new Temperature(-40), // Same in C and F
};

Console.WriteLine("Temperatures:");
foreach (var t in temps)
    Console.WriteLine($"  {t.Format("all")}");

Array.Sort(temps, (a, b) => a.CompareTo(b));
Console.WriteLine("\nSorted:");
foreach (var t in temps) Console.WriteLine($"  {t}");

Console.WriteLine($"\nBody temp > Boiling: {temps[^1].IsGreaterThan(temps[0])}");

// Validation
var invalid = new Temperature(-300);
Console.WriteLine($"\nValid: {invalid.IsValid()}");
foreach (string error in invalid.GetValidationErrors())
    Console.WriteLine($"  Error: {error}");

// ============ Explicit Interface Implementation ============
interface IAreaCalculator
{
    double Calculate();
}

interface IVolumeCalculator
{
    double Calculate();
}

class Cylinder : IAreaCalculator, IVolumeCalculator
{
    public double Radius { get; }
    public double Height { get; }
    
    public Cylinder(double r, double h) { Radius = r; Height = h; }
    
    // Explicit implementation - ต้อง cast เพื่อเรียก
    double IAreaCalculator.Calculate() =>
        2 * Math.PI * Radius * (Radius + Height);
    
    double IVolumeCalculator.Calculate() =>
        Math.PI * Radius * Radius * Height;
    
    public double SurfaceArea => ((IAreaCalculator)this).Calculate();
    public double Volume => ((IVolumeCalculator)this).Calculate();
}

var cyl = new Cylinder(5, 10);
Console.WriteLine($"\nCylinder (r=5, h=10):");
Console.WriteLine($"  Surface Area: {cyl.SurfaceArea:F2}");
Console.WriteLine($"  Volume: {cyl.Volume:F2}");

// ผ่าน interface
IAreaCalculator area = cyl;
IVolumeCalculator vol = cyl;
Console.WriteLine($"  Via IAreaCalculator: {area.Calculate():F2}");
Console.WriteLine($"  Via IVolumeCalculator: {vol.Calculate():F2}");
```

---

## Step 253: Common .NET Interfaces

```csharp
// ============ IEnumerable<T> และ IEnumerator<T> ============
class NumberRange : IEnumerable<int>
{
    private int _start, _end, _step;
    
    public NumberRange(int start, int end, int step = 1)
    {
        _start = start;
        _end = end;
        _step = step;
    }
    
    public IEnumerator<int> GetEnumerator()
    {
        for (int i = _start; _step > 0 ? i <= _end : i >= _end; i += _step)
            yield return i;
    }
    
    System.Collections.IEnumerator System.Collections.IEnumerable.GetEnumerator()
        => GetEnumerator();
}

var range = new NumberRange(1, 20, 2);
Console.Write("Odd numbers: ");
foreach (int n in range) Console.Write($"{n} ");
Console.WriteLine();

// LINQ works on IEnumerable<T>
var sum = range.Sum();
var avg = range.Average();
Console.WriteLine($"Sum: {sum}, Avg: {avg}");

// ============ IComparable<T> ============
class Student : IComparable<Student>
{
    public string Name { get; }
    public double GPA { get; }
    
    public Student(string name, double gpa) { Name = name; GPA = gpa; }
    
    public int CompareTo(Student? other)
    {
        if (other is null) return 1;
        // เรียงจาก GPA สูงสุด
        int result = other.GPA.CompareTo(GPA);
        if (result != 0) return result;
        // ถ้า GPA เท่ากัน เรียงตามชื่อ
        return Name.CompareTo(other.Name);
    }
    
    public override string ToString() => $"{Name} (GPA: {GPA:F2})";
}

var students = new[]
{
    new Student("Alice", 3.8),
    new Student("Bob", 3.5),
    new Student("Charlie", 3.8),
    new Student("Dave", 3.9),
    new Student("Eve", 3.2),
};

Array.Sort(students);
Console.WriteLine("\nStudents by GPA (desc):");
foreach (var s in students) Console.WriteLine($"  {s}");

// ============ IComparer<T> ============
class StudentComparer : IComparer<Student>
{
    public enum SortBy { Name, GPA }
    private SortBy _sortBy;
    private bool _descending;
    
    public StudentComparer(SortBy sortBy, bool descending = false)
    {
        _sortBy = sortBy;
        _descending = descending;
    }
    
    public int Compare(Student? x, Student? y)
    {
        if (x is null && y is null) return 0;
        if (x is null) return -1;
        if (y is null) return 1;
        
        int result = _sortBy switch
        {
            SortBy.Name => x.Name.CompareTo(y.Name),
            SortBy.GPA => x.GPA.CompareTo(y.GPA),
            _ => 0
        };
        
        return _descending ? -result : result;
    }
}

Array.Sort(students, new StudentComparer(StudentComparer.SortBy.Name));
Console.WriteLine("\nStudents by Name (asc):");
foreach (var s in students) Console.WriteLine($"  {s}");

// ============ IEquatable<T> ============
class Product : IEquatable<Product>
{
    public string SKU { get; }
    public string Name { get; }
    public decimal Price { get; }
    
    public Product(string sku, string name, decimal price)
    { SKU = sku; Name = name; Price = price; }
    
    // Equality based on SKU
    public bool Equals(Product? other) =>
        other is not null && SKU == other.SKU;
    
    public override bool Equals(object? obj) => Equals(obj as Product);
    public override int GetHashCode() => SKU.GetHashCode();
    public override string ToString() => $"{SKU}: {Name} (฿{Price:N0})";
}

var p1 = new Product("SKU001", "iPhone", 35000);
var p2 = new Product("SKU001", "iPhone", 35000);  // Same SKU
var p3 = new Product("SKU002", "MacBook", 65000); // Different SKU

Console.WriteLine($"\np1 == p2 (same SKU): {p1.Equals(p2)}"); // True
Console.WriteLine($"p1 == p3 (diff SKU): {p1.Equals(p3)}");  // False

var productSet = new HashSet<Product> { p1, p2, p3 };
Console.WriteLine($"Set count (p1 & p2 are same): {productSet.Count}"); // 2

// ============ IDisposable ============
class FileProcessor : IDisposable
{
    private System.IO.StreamWriter? _writer;
    private bool _disposed = false;
    
    public FileProcessor(string path)
    {
        _writer = new System.IO.StreamWriter(path, append: true);
        Console.WriteLine($"FileProcessor opened: {path}");
    }
    
    public void Process(string data)
    {
        ObjectDisposedException.ThrowIf(_disposed, this);
        _writer?.WriteLine($"[{DateTime.Now:yyyy-MM-dd HH:mm:ss}] {data}");
        Console.WriteLine($"Processed: {data}");
    }
    
    // Implement IDisposable
    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }
    
    protected virtual void Dispose(bool disposing)
    {
        if (!_disposed)
        {
            if (disposing)
            {
                _writer?.Flush();
                _writer?.Dispose();
                _writer = null;
                Console.WriteLine("FileProcessor disposed");
            }
            _disposed = true;
        }
    }
    
    // Finalizer (กรณีไม่ได้เรียก Dispose)
    ~FileProcessor()
    {
        Dispose(false);
    }
}

// using statement - Dispose อัตโนมัติ
using (var processor = new FileProcessor("/tmp/test.log"))
{
    processor.Process("Hello");
    processor.Process("World");
} // Dispose ถูกเรียกที่นี่

// using declaration (C# 8+)
// {
//     using var proc2 = new FileProcessor("/tmp/test2.log");
//     proc2.Process("C# 8 style");
// } // Dispose ที่สิ้นสุด scope
```

---

## Step 254-265: Interface Design Patterns

```csharp
// ============ Repository Pattern ============
interface IEntity
{
    int Id { get; }
}

interface IRepository<T> where T : IEntity
{
    T? GetById(int id);
    IEnumerable<T> GetAll();
    IEnumerable<T> Find(Func<T, bool> predicate);
    void Add(T entity);
    void Update(T entity);
    bool Delete(int id);
    int Count();
}

// Base implementation
abstract class InMemoryRepository<T> : IRepository<T> where T : IEntity
{
    protected List<T> _items = new();
    
    public T? GetById(int id) => _items.FirstOrDefault(e => e.Id == id);
    public IEnumerable<T> GetAll() => _items.ToList();
    public IEnumerable<T> Find(Func<T, bool> predicate) => _items.Where(predicate);
    public void Add(T entity) => _items.Add(entity);
    public int Count() => _items.Count;
    
    public void Update(T entity)
    {
        int index = _items.FindIndex(e => e.Id == entity.Id);
        if (index >= 0) _items[index] = entity;
        else throw new KeyNotFoundException($"Entity {entity.Id} not found");
    }
    
    public bool Delete(int id)
    {
        int removed = _items.RemoveAll(e => e.Id == id);
        return removed > 0;
    }
}

// Models
record CustomerEntity(int Id, string Name, string Email, decimal TotalPurchases) : IEntity;
record OrderEntity(int Id, int CustomerId, DateTime Date, decimal Amount, string Status) : IEntity;

// Concrete Repository
class CustomerRepository : InMemoryRepository<CustomerEntity>
{
    public IEnumerable<CustomerEntity> GetVIPCustomers(decimal threshold = 10000)
        => Find(c => c.TotalPurchases >= threshold);
    
    public CustomerEntity? GetByEmail(string email)
        => _items.FirstOrDefault(c => c.Email.Equals(email, StringComparison.OrdinalIgnoreCase));
}

class OrderRepository : InMemoryRepository<OrderEntity>
{
    public IEnumerable<OrderEntity> GetByCustomer(int customerId)
        => Find(o => o.CustomerId == customerId);
    
    public IEnumerable<OrderEntity> GetByStatus(string status)
        => Find(o => o.Status == status);
    
    public decimal GetTotalRevenue()
        => _items.Where(o => o.Status == "Completed").Sum(o => o.Amount);
}

// ============ Observer Pattern ============
interface IEvent<TData>
{
    string Name { get; }
    TData Data { get; }
    DateTime Timestamp { get; }
}

interface IEventHandler<TData>
{
    Task HandleAsync(IEvent<TData> evt);
}

interface IEventBus
{
    void Subscribe<T>(string eventName, IEventHandler<T> handler);
    Task PublishAsync<T>(IEvent<T> evt);
}

record OrderEvent(string Name, OrderEntity Data) : IEvent<OrderEntity>
{
    public DateTime Timestamp { get; } = DateTime.Now;
}

class EmailNotificationHandler : IEventHandler<OrderEntity>
{
    public async Task HandleAsync(IEvent<OrderEntity> evt)
    {
        await Task.Delay(10); // Simulate async
        Console.WriteLine($"📧 Email sent for order #{evt.Data.Id}: {evt.Name}");
    }
}

class InventoryUpdateHandler : IEventHandler<OrderEntity>
{
    public async Task HandleAsync(IEvent<OrderEntity> evt)
    {
        await Task.Delay(10);
        Console.WriteLine($"📦 Inventory updated for order #{evt.Data.Id}");
    }
}

class SimpleEventBus : IEventBus
{
    private Dictionary<string, List<object>> _handlers = new();
    
    public void Subscribe<T>(string eventName, IEventHandler<T> handler)
    {
        if (!_handlers.ContainsKey(eventName))
            _handlers[eventName] = new();
        _handlers[eventName].Add(handler);
    }
    
    public async Task PublishAsync<T>(IEvent<T> evt)
    {
        if (!_handlers.TryGetValue(evt.Name, out var handlers)) return;
        
        var tasks = handlers
            .OfType<IEventHandler<T>>()
            .Select(h => h.HandleAsync(evt));
        
        await Task.WhenAll(tasks);
    }
}

// Demo
var eventBus = new SimpleEventBus();
eventBus.Subscribe<OrderEntity>("OrderCreated", new EmailNotificationHandler());
eventBus.Subscribe<OrderEntity>("OrderCreated", new InventoryUpdateHandler());
eventBus.Subscribe<OrderEntity>("OrderShipped", new EmailNotificationHandler());

var newOrder = new OrderEntity(1, 100, DateTime.Now, 5000, "Created");
await eventBus.PublishAsync(new OrderEvent("OrderCreated", newOrder));
await eventBus.PublishAsync(new OrderEvent("OrderShipped", newOrder));
```

---

## Step 266-290: โปรแกรม Payment System สมบูรณ์

```csharp
// ============ Payment System ============

// ============ Interfaces ============
interface IPaymentMethod
{
    string Name { get; }
    string Description { get; }
    bool IsAvailable { get; }
    decimal MinAmount { get; }
    decimal MaxAmount { get; }
    decimal CalculateFee(decimal amount);
    Task<PaymentResult> ProcessAsync(PaymentRequest request);
}

interface IPaymentValidator
{
    ValidationResult Validate(PaymentRequest request);
}

interface IPaymentLogger
{
    void Log(PaymentResult result);
    IEnumerable<PaymentResult> GetHistory(int count = 10);
}

interface IRefundable
{
    Task<RefundResult> RefundAsync(string transactionId, decimal amount);
}

// ============ Value Objects ============
record PaymentRequest(
    string TransactionId,
    string CustomerId,
    decimal Amount,
    string Currency = "THB",
    string? Description = null
);

record PaymentResult(
    string TransactionId,
    bool Success,
    string Message,
    decimal Amount,
    decimal Fee,
    string PaymentMethod,
    DateTime Timestamp
)
{
    public decimal TotalAmount => Amount + Fee;
    public override string ToString() =>
        $"[{(Success ? "✅" : "❌")}] {TransactionId}: ฿{Amount:N2} via {PaymentMethod} " +
        $"(Fee: ฿{Fee:N2}) - {Message}";
}

record RefundResult(string OriginalTransactionId, string RefundId, bool Success, string Message);

record ValidationResult(bool IsValid, IEnumerable<string> Errors)
{
    public static ValidationResult Ok() => new(true, Enumerable.Empty<string>());
    public static ValidationResult Fail(params string[] errors) => new(false, errors);
}

// ============ Abstract Base ============
abstract class PaymentMethodBase : IPaymentMethod, IPaymentValidator, IPaymentLogger
{
    private List<PaymentResult> _history = new();
    
    public abstract string Name { get; }
    public abstract string Description { get; }
    public abstract decimal MinAmount { get; }
    public abstract decimal MaxAmount { get; }
    public virtual bool IsAvailable => true;
    
    public abstract decimal CalculateFee(decimal amount);
    
    public async Task<PaymentResult> ProcessAsync(PaymentRequest request)
    {
        // Validate first
        var validation = Validate(request);
        if (!validation.IsValid)
        {
            var failResult = new PaymentResult(
                request.TransactionId, false,
                string.Join("; ", validation.Errors),
                request.Amount, 0, Name, DateTime.Now
            );
            Log(failResult);
            return failResult;
        }
        
        // Process
        decimal fee = CalculateFee(request.Amount);
        var result = await DoProcessAsync(request, fee);
        Log(result);
        return result;
    }
    
    protected abstract Task<PaymentResult> DoProcessAsync(PaymentRequest request, decimal fee);
    
    public virtual ValidationResult Validate(PaymentRequest request)
    {
        var errors = new List<string>();
        
        if (request.Amount < MinAmount)
            errors.Add($"ยอดเงินขั้นต่ำ: ฿{MinAmount:N0}");
        
        if (request.Amount > MaxAmount)
            errors.Add($"ยอดเงินสูงสุด: ฿{MaxAmount:N0}");
        
        if (!IsAvailable)
            errors.Add($"{Name} ไม่พร้อมใช้งาน");
        
        return errors.Count == 0 ? ValidationResult.Ok() : ValidationResult.Fail(errors.ToArray());
    }
    
    public void Log(PaymentResult result) => _history.Add(result);
    
    public IEnumerable<PaymentResult> GetHistory(int count = 10)
        => _history.TakeLast(count);
}

// ============ Concrete Implementations ============
class CreditCardPayment : PaymentMethodBase, IRefundable
{
    private string _cardType;
    
    public CreditCardPayment(string cardType = "Visa") => _cardType = cardType;
    
    public override string Name => $"Credit Card ({_cardType})";
    public override string Description => $"Pay with {_cardType} credit card";
    public override decimal MinAmount => 1;
    public override decimal MaxAmount => 500000;
    
    public override decimal CalculateFee(decimal amount) =>
        Math.Round(amount * 0.015m, 2); // 1.5%
    
    protected override async Task<PaymentResult> DoProcessAsync(PaymentRequest request, decimal fee)
    {
        Console.WriteLine($"Processing credit card payment...");
        await Task.Delay(500); // Simulate network call
        
        // Simulate 95% success rate
        bool success = Random.Shared.NextDouble() < 0.95;
        
        return new PaymentResult(
            request.TransactionId,
            success,
            success ? "Payment approved" : "Card declined",
            request.Amount,
            success ? fee : 0,
            Name,
            DateTime.Now
        );
    }
    
    public async Task<RefundResult> RefundAsync(string transactionId, decimal amount)
    {
        await Task.Delay(300);
        string refundId = $"REF-{Guid.NewGuid():N}"[..12];
        Console.WriteLine($"Refunding ฿{amount:N2} for {transactionId}...");
        return new RefundResult(transactionId, refundId, true, $"Refund processed: {refundId}");
    }
    
    public override ValidationResult Validate(PaymentRequest request)
    {
        var baseResult = base.Validate(request);
        if (!baseResult.IsValid) return baseResult;
        
        // Additional credit card validations
        var errors = baseResult.Errors.ToList();
        if (request.Amount > 300000)
            errors.Add("ต้องได้รับการอนุมัติพิเศษสำหรับยอดเกิน ฿300,000");
        
        return errors.Count == 0 ? ValidationResult.Ok() : ValidationResult.Fail(errors.ToArray());
    }
}

class PromptPayPayment : PaymentMethodBase
{
    public override string Name => "PromptPay";
    public override string Description => "Pay via PromptPay QR code";
    public override decimal MinAmount => 1;
    public override decimal MaxAmount => 100000;
    
    public override decimal CalculateFee(decimal amount) => 0; // No fee!
    
    protected override async Task<PaymentResult> DoProcessAsync(PaymentRequest request, decimal fee)
    {
        Console.WriteLine($"Generating PromptPay QR code...");
        await Task.Delay(100);
        
        Console.WriteLine($"QR Code: [QR{request.TransactionId}]");
        Console.WriteLine("Waiting for customer to scan...");
        await Task.Delay(200);
        
        return new PaymentResult(
            request.TransactionId,
            true,
            "PromptPay payment confirmed",
            request.Amount,
            0,
            Name,
            DateTime.Now
        );
    }
}

class BankTransferPayment : PaymentMethodBase, IRefundable
{
    public override string Name => "Bank Transfer";
    public override string Description => "Direct bank transfer";
    public override decimal MinAmount => 100;
    public override decimal MaxAmount => 5000000;
    
    public override decimal CalculateFee(decimal amount) =>
        amount <= 100000 ? 25 : 50; // Flat fee
    
    protected override async Task<PaymentResult> DoProcessAsync(PaymentRequest request, decimal fee)
    {
        Console.WriteLine($"Initiating bank transfer...");
        await Task.Delay(300);
        
        return new PaymentResult(
            request.TransactionId,
            true,
            $"Bank transfer initiated. Reference: TXN{request.TransactionId}",
            request.Amount,
            fee,
            Name,
            DateTime.Now
        );
    }
    
    public async Task<RefundResult> RefundAsync(string transactionId, decimal amount)
    {
        await Task.Delay(500);
        return new RefundResult(transactionId, $"RBANK-{DateTime.Now:yyyyMMdd}", true, "Bank refund processing (1-3 days)");
    }
}

// ============ Payment Gateway ============
class PaymentGateway
{
    private Dictionary<string, IPaymentMethod> _methods = new();
    private List<PaymentResult> _allTransactions = new();
    
    public void RegisterMethod(IPaymentMethod method)
    {
        _methods[method.Name] = method;
        Console.WriteLine($"✅ Registered: {method.Name}");
    }
    
    public async Task<PaymentResult> PayAsync(string methodName, PaymentRequest request)
    {
        if (!_methods.TryGetValue(methodName, out var method))
            return new PaymentResult(request.TransactionId, false, $"Payment method '{methodName}' not found",
                request.Amount, 0, "Unknown", DateTime.Now);
        
        var result = await method.ProcessAsync(request);
        _allTransactions.Add(result);
        
        Console.WriteLine(result);
        return result;
    }
    
    public void PrintReport()
    {
        Console.WriteLine("\n📊 Transaction Report:");
        Console.WriteLine(new string('─', 60));
        
        var successful = _allTransactions.Where(t => t.Success);
        var failed = _allTransactions.Where(t => !t.Success);
        
        Console.WriteLine($"Total: {_allTransactions.Count}");
        Console.WriteLine($"Success: {successful.Count()} | Failed: {failed.Count()}");
        
        decimal totalRevenue = successful.Sum(t => t.Amount);
        decimal totalFees = successful.Sum(t => t.Fee);
        
        Console.WriteLine($"Total Revenue: ฿{totalRevenue:N2}");
        Console.WriteLine($"Total Fees: ฿{totalFees:N2}");
        
        Console.WriteLine("\nBy Payment Method:");
        var byMethod = _allTransactions.GroupBy(t => t.PaymentMethod);
        foreach (var group in byMethod)
        {
            var success = group.Where(t => t.Success);
            Console.WriteLine($"  {group.Key}: {success.Count()}/{group.Count()} success, ฿{success.Sum(t => t.Amount):N2}");
        }
    }
    
    public IEnumerable<IPaymentMethod> GetAvailableMethods()
        => _methods.Values.Where(m => m.IsAvailable);
}

// ============ Demo ============
Console.WriteLine("🏦 Payment System Demo");
Console.WriteLine(new string('═', 50));

var gateway = new PaymentGateway();
gateway.RegisterMethod(new CreditCardPayment("Visa"));
gateway.RegisterMethod(new CreditCardPayment("Mastercard"));
gateway.RegisterMethod(new PromptPayPayment());
gateway.RegisterMethod(new BankTransferPayment());

Console.WriteLine("\nAvailable payment methods:");
foreach (var method in gateway.GetAvailableMethods())
{
    Console.WriteLine($"  {method.Name}: {method.Description}");
    Console.WriteLine($"    Limit: ฿{method.MinAmount:N0} - ฿{method.MaxAmount:N0}");
    Console.WriteLine($"    Fee for ฿1000: ฿{method.CalculateFee(1000):N2}");
}

Console.WriteLine("\nProcessing payments:");
var transactions = new[]
{
    ("Credit Card (Visa)", new PaymentRequest("TXN001", "C001", 5000, Description: "Order #1001")),
    ("PromptPay", new PaymentRequest("TXN002", "C002", 1200, Description: "Order #1002")),
    ("Bank Transfer", new PaymentRequest("TXN003", "C003", 50000, Description: "Order #1003")),
    ("Credit Card (Mastercard)", new PaymentRequest("TXN004", "C001", 15000, Description: "Order #1004")),
    ("PromptPay", new PaymentRequest("TXN005", "C004", 500, Description: "Order #1005")),
};

foreach (var (method, request) in transactions)
{
    Console.WriteLine($"\n--- Payment {request.TransactionId} ---");
    var result = await gateway.PayAsync(method, request);
    
    // Handle refund for some
    if (result.Success && method.Contains("Credit") && Random.Shared.NextDouble() < 0.3)
    {
        var paymentMethod = gateway.GetAvailableMethods().First(m => m.Name == method);
        if (paymentMethod is IRefundable refundable)
        {
            Console.WriteLine("Customer requested refund...");
            var refund = await refundable.RefundAsync(result.TransactionId, result.Amount / 2);
            Console.WriteLine($"  Refund: {refund.Success} - {refund.Message}");
        }
    }
}

gateway.PrintReport();
```

---

## สรุป Part 10

```
✅ Step 251: Interface พื้นฐานและ default implementation
✅ Step 252: Multiple Interface Implementation
✅ Step 253: Common .NET Interfaces (IEnumerable, IComparable, IDisposable)
✅ Steps 254-265: Interface Design Patterns (Repository, Observer)
✅ Steps 266-290: Payment System สมบูรณ์
```

## แบบฝึกหัด Part 10

1. สร้าง `IPlugin` interface และ plugin loader system
2. Implement `ICache<TKey, TValue>` ด้วย Memory Cache + File Cache
3. สร้าง `INotificationChannel` hierarchy: Email, SMS, LINE, Telegram
4. เขียน `ISerializer<T>` interface พร้อม JSON/XML/Binary implementations
5. สร้าง game engine ด้วย `IGameObject`, `IRenderable`, `ICollidable` interfaces

---

**ถัดไป: [Part 11 - Generics →](part-11-generics.md)**
