# Part 21: SOLID Principles

## ขั้นตอนที่ 616: SOLID คืออะไร?

SOLID คือ 5 หลักการออกแบบ OOP ที่ทำให้โค้ดยืดหยุ่น บำรุงรักษาได้ง่าย และขยายตัวได้

```
S - Single Responsibility Principle (SRP)
O - Open/Closed Principle (OCP)
L - Liskov Substitution Principle (LSP)
I - Interface Segregation Principle (ISP)
D - Dependency Inversion Principle (DIP)
```

## ขั้นตอนที่ 617: S - Single Responsibility Principle (SRP)

**"A class should have only one reason to change"**

```csharp
// ❌ BAD: Class ทำหลายอย่างเกินไป
public class UserService_BAD
{
    private readonly string _connectionString;
    
    public UserService_BAD(string connectionString)
    {
        _connectionString = connectionString;
    }
    
    // Responsibility 1: Database
    public User GetUser(int id)
    {
        using var conn = new System.Data.SqlClient.SqlConnection(_connectionString);
        conn.Open();
        // ...query...
        return new User();
    }
    
    // Responsibility 2: Business Logic
    public bool ValidateUser(User user)
    {
        return !string.IsNullOrEmpty(user.Email) && user.Age >= 18;
    }
    
    // Responsibility 3: Email
    public void SendWelcomeEmail(User user)
    {
        var smtp = new System.Net.Mail.SmtpClient("smtp.gmail.com");
        // ...send email...
    }
    
    // Responsibility 4: Logging
    public void LogUserAction(int userId, string action)
    {
        File.AppendAllText("log.txt", $"{DateTime.Now}: User {userId} - {action}\n");
    }
    
    // Responsibility 5: Report generation
    public string GenerateUserReport(int userId)
    {
        return $"Report for user {userId}...";
    }
}

// ✅ GOOD: แต่ละ class มีหน้าที่เดียว
// 1. Data Access
public class UserRepository
{
    private readonly IDbConnection _db;
    
    public UserRepository(IDbConnection db) => _db = db;
    
    public async Task<User?> GetByIdAsync(int id)
    {
        // Only responsible for data access
        return await _db.QueryFirstOrDefaultAsync<User>(
            "SELECT * FROM Users WHERE Id = @Id", new { Id = id });
    }
    
    public async Task<int> CreateAsync(User user)
        => await _db.ExecuteScalarAsync<int>(
            "INSERT INTO Users (Name, Email) OUTPUT INSERTED.Id VALUES (@Name, @Email)", user);
}

// 2. Business Logic / Validation
public class UserValidator
{
    public ValidationResult Validate(User user)
    {
        var errors = new List<string>();
        
        if (string.IsNullOrWhiteSpace(user.Name))
            errors.Add("Name is required");
        
        if (!IsValidEmail(user.Email))
            errors.Add("Invalid email format");
        
        if (user.Age < 18)
            errors.Add("Must be at least 18 years old");
        
        return errors.Count == 0
            ? ValidationResult.Success()
            : ValidationResult.Failure(errors);
    }
    
    private static bool IsValidEmail(string email)
    {
        try { _ = new System.Net.Mail.MailAddress(email); return true; }
        catch { return false; }
    }
}

// 3. Email Service
public class EmailService
{
    private readonly IEmailProvider _provider;
    
    public EmailService(IEmailProvider provider) => _provider = provider;
    
    public async Task SendWelcomeEmailAsync(User user)
    {
        await _provider.SendAsync(
            to: user.Email,
            subject: $"Welcome, {user.Name}!",
            body: GetWelcomeTemplate(user));
    }
    
    private static string GetWelcomeTemplate(User user)
        => $"<h1>Welcome {user.Name}!</h1><p>Thanks for joining us.</p>";
}

// 4. Application Service (orchestrates the above)
public class UserApplicationService
{
    private readonly UserRepository _repo;
    private readonly UserValidator _validator;
    private readonly EmailService _email;
    private readonly ILogger<UserApplicationService> _logger;
    
    public UserApplicationService(
        UserRepository repo,
        UserValidator validator,
        EmailService email,
        ILogger<UserApplicationService> logger)
    {
        _repo = repo;
        _validator = validator;
        _email = email;
        _logger = logger;
    }
    
    public async Task<Result<int>> RegisterAsync(User user)
    {
        // Orchestrate: validate -> save -> email
        var validation = _validator.Validate(user);
        if (!validation.IsValid)
            return Result<int>.Failure(validation.Errors);
        
        var id = await _repo.CreateAsync(user);
        _logger.LogInformation("User {UserId} registered", id);
        
        await _email.SendWelcomeEmailAsync(user with { Id = id });
        
        return Result<int>.Success(id);
    }
}

public record ValidationResult(bool IsValid, List<string> Errors)
{
    public static ValidationResult Success() => new(true, []);
    public static ValidationResult Failure(List<string> errors) => new(false, errors);
}
```

## ขั้นตอนที่ 618: O - Open/Closed Principle (OCP)

**"Software entities should be open for extension, but closed for modification"**

```csharp
// ❌ BAD: ต้องแก้ไข class เดิมทุกครั้งที่เพิ่ม discount type ใหม่
public class DiscountCalculator_BAD
{
    public decimal Calculate(Order order, string customerType)
    {
        // ทุกครั้งที่เพิ่ม customer type ใหม่ ต้องแก้ method นี้
        return customerType switch
        {
            "Regular" => order.Total * 0.95m,
            "Silver" => order.Total * 0.90m,
            "Gold" => order.Total * 0.85m,
            // ถ้าเพิ่ม "Platinum" ต้องแก้ method นี้
            _ => order.Total
        };
    }
}

// ✅ GOOD: Open for extension via abstraction
public interface IDiscountStrategy
{
    string CustomerType { get; }
    decimal Apply(decimal amount);
    bool IsApplicable(Customer customer);
}

public class RegularDiscount : IDiscountStrategy
{
    public string CustomerType => "Regular";
    public decimal Apply(decimal amount) => amount * 0.95m;
    public bool IsApplicable(Customer customer) => customer.TotalOrders < 10;
}

public class SilverDiscount : IDiscountStrategy
{
    public string CustomerType => "Silver";
    public decimal Apply(decimal amount) => amount * 0.90m;
    public bool IsApplicable(Customer customer) => customer.TotalOrders >= 10;
}

public class GoldDiscount : IDiscountStrategy
{
    public string CustomerType => "Gold";
    public decimal Apply(decimal amount) => amount * 0.85m;
    public bool IsApplicable(Customer customer) => 
        customer.TotalSpent >= 50000 || customer.TotalOrders >= 50;
}

// Extension: เพิ่ม Platinum โดยไม่ต้องแก้โค้ดเดิม
public class PlatinumDiscount : IDiscountStrategy
{
    public string CustomerType => "Platinum";
    public decimal Apply(decimal amount) => amount * 0.75m; // 25% off
    public bool IsApplicable(Customer customer) => customer.TotalSpent >= 200000;
}

// Birthday discount - เพิ่มใหม่โดยไม่แก้ class เดิม
public class BirthdayDiscount : IDiscountStrategy
{
    public string CustomerType => "Birthday";
    public decimal Apply(decimal amount) => amount * 0.80m;
    public bool IsApplicable(Customer customer) => 
        customer.BirthDate.Month == DateTime.Now.Month &&
        customer.BirthDate.Day == DateTime.Now.Day;
}

// Calculator ไม่ต้องแก้ไข
public class DiscountCalculator
{
    private readonly IEnumerable<IDiscountStrategy> _strategies;
    
    public DiscountCalculator(IEnumerable<IDiscountStrategy> strategies)
    {
        _strategies = strategies;
    }
    
    public decimal Calculate(Order order, Customer customer)
    {
        // Apply all applicable discounts (stack)
        var discountedAmount = order.Total;
        
        foreach (var strategy in _strategies.Where(s => s.IsApplicable(customer)))
        {
            var before = discountedAmount;
            discountedAmount = strategy.Apply(discountedAmount);
            Console.WriteLine($"Applied {strategy.CustomerType} discount: {before:F2} -> {discountedAmount:F2}");
        }
        
        return discountedAmount;
    }
    
    // หรือ apply เฉพาะ best discount
    public decimal CalculateBest(Order order, Customer customer)
    {
        var applicable = _strategies
            .Where(s => s.IsApplicable(customer))
            .ToList();
        
        if (!applicable.Any()) return order.Total;
        
        return applicable
            .Min(s => s.Apply(order.Total));
    }
}

// Real-world: Specification Pattern (OCP compliant)
public interface ISpecification<T>
{
    bool IsSatisfiedBy(T entity);
    ISpecification<T> And(ISpecification<T> other);
    ISpecification<T> Or(ISpecification<T> other);
    ISpecification<T> Not();
}

public abstract class Specification<T> : ISpecification<T>
{
    public abstract bool IsSatisfiedBy(T entity);
    
    public ISpecification<T> And(ISpecification<T> other)
        => new AndSpecification<T>(this, other);
    
    public ISpecification<T> Or(ISpecification<T> other)
        => new OrSpecification<T>(this, other);
    
    public ISpecification<T> Not()
        => new NotSpecification<T>(this);
}

public class AndSpecification<T>(ISpecification<T> left, ISpecification<T> right) 
    : Specification<T>
{
    public override bool IsSatisfiedBy(T entity)
        => left.IsSatisfiedBy(entity) && right.IsSatisfiedBy(entity);
}

public class OrSpecification<T>(ISpecification<T> left, ISpecification<T> right) 
    : Specification<T>
{
    public override bool IsSatisfiedBy(T entity)
        => left.IsSatisfiedBy(entity) || right.IsSatisfiedBy(entity);
}

public class NotSpecification<T>(ISpecification<T> inner) : Specification<T>
{
    public override bool IsSatisfiedBy(T entity) => !inner.IsSatisfiedBy(entity);
}

// Product Specifications - เพิ่มได้โดยไม่แก้ code เดิม
public class InStockSpec : Specification<Product>
{
    public override bool IsSatisfiedBy(Product p) => p.Stock > 0;
}

public class PriceRangeSpec : Specification<Product>
{
    private readonly decimal _min, _max;
    
    public PriceRangeSpec(decimal min, decimal max) { _min = min; _max = max; }
    
    public override bool IsSatisfiedBy(Product p) => p.Price >= _min && p.Price <= _max;
}

public class CategorySpec : Specification<Product>
{
    private readonly string _category;
    
    public CategorySpec(string category) => _category = category;
    
    public override bool IsSatisfiedBy(Product p) => p.Category == _category;
}

// การใช้งาน
var spec = new InStockSpec()
    .And(new PriceRangeSpec(100, 500))
    .And(new CategorySpec("Electronics").Or(new CategorySpec("Books")));

var filtered = products.Where(spec.IsSatisfiedBy).ToList();
```

## ขั้นตอนที่ 619: L - Liskov Substitution Principle (LSP)

**"Objects of a subtype must be substitutable for objects of the supertype"**

```csharp
// ❌ BAD: Square ทำให้ Rectangle LSP ขาด
public class Rectangle
{
    public virtual double Width { get; set; }
    public virtual double Height { get; set; }
    
    public double Area => Width * Height;
}

public class Square : Rectangle
{
    // Square override: ต้อง set ทั้งคู่พร้อมกัน
    public override double Width
    {
        get => base.Width;
        set { base.Width = value; base.Height = value; } // ❌ ทำให้ Rectangle behavior เปลี่ยน
    }
    
    public override double Height
    {
        get => base.Height;
        set { base.Height = value; base.Width = value; } // ❌
    }
}

// Test ที่ fail ด้วย Square
static void TestRectangle(Rectangle r)
{
    r.Width = 5;
    r.Height = 3;
    // ควรจะได้ 15 แต่ Square จะคืน 9!
    Console.WriteLine($"Expected: 15, Got: {r.Area}"); 
}

// ✅ GOOD: แยก hierarchy ตาม behavior จริง
public interface IShape
{
    double Area { get; }
    double Perimeter { get; }
}

public class Rectangle : IShape
{
    public double Width { get; init; }
    public double Height { get; init; }
    
    public Rectangle(double width, double height)
    {
        Width = width;
        Height = height;
    }
    
    public double Area => Width * Height;
    public double Perimeter => 2 * (Width + Height);
}

public class Square : IShape
{
    public double Side { get; init; }
    
    public Square(double side) => Side = side;
    
    public double Area => Side * Side;
    public double Perimeter => 4 * Side;
}

// ตัวอย่าง LSP ที่ดี: Bird hierarchy
public abstract class Bird
{
    public string Name { get; init; } = "";
    
    // LSP: ทุก Bird สามารถทำ behaviors เหล่านี้ได้
    public abstract void Eat();
    public abstract void Breathe();
}

public abstract class FlyingBird : Bird
{
    public abstract void Fly();
}

public class Eagle : FlyingBird
{
    public override void Eat() => Console.WriteLine($"{Name} eats fish");
    public override void Breathe() => Console.WriteLine($"{Name} breathes");
    public override void Fly() => Console.WriteLine($"{Name} soars high");
}

public class Parrot : FlyingBird
{
    public override void Eat() => Console.WriteLine($"{Name} eats seeds");
    public override void Breathe() => Console.WriteLine($"{Name} breathes");
    public override void Fly() => Console.WriteLine($"{Name} flaps wings");
    public void Speak(string phrase) => Console.WriteLine($"{Name}: {phrase}");
}

public class Penguin : Bird  // Penguin ไม่ fly!
{
    public override void Eat() => Console.WriteLine($"{Name} eats fish");
    public override void Breathe() => Console.WriteLine($"{Name} breathes");
    public void Swim() => Console.WriteLine($"{Name} swims");
}

// LSP Test: FlyingBirds ต้อง fly ได้ทั้งหมด
static void MakeFly(FlyingBird bird)
{
    bird.Fly(); // Penguin จะไม่ถูกส่งมาที่นี่ - LSP ถูกต้อง
}

// Real-world LSP: Repository Pattern
public interface IRepository<T>
{
    Task<T?> GetByIdAsync(int id);
    Task<IEnumerable<T>> GetAllAsync();
    Task<int> CreateAsync(T entity);
    Task UpdateAsync(T entity);
    Task DeleteAsync(int id);
}

public class SqlRepository<T> : IRepository<T> where T : class
{
    private readonly IDbConnection _db;
    
    public SqlRepository(IDbConnection db) => _db = db;
    
    public async Task<T?> GetByIdAsync(int id)
        => await _db.QueryFirstOrDefaultAsync<T>($"SELECT * FROM {typeof(T).Name}s WHERE Id = @Id", new { Id = id });
    
    public async Task<IEnumerable<T>> GetAllAsync()
        => await _db.QueryAsync<T>($"SELECT * FROM {typeof(T).Name}s");
    
    public async Task<int> CreateAsync(T entity)
        => await _db.ExecuteAsync($"INSERT INTO {typeof(T).Name}s ...", entity);
    
    public async Task UpdateAsync(T entity)
        => await _db.ExecuteAsync($"UPDATE {typeof(T).Name}s ...", entity);
    
    public async Task DeleteAsync(int id)
        => await _db.ExecuteAsync($"DELETE FROM {typeof(T).Name}s WHERE Id = @Id", new { Id = id });
}

// Read-only repository ที่ยัง LSP compliant
public interface IReadOnlyRepository<T>
{
    Task<T?> GetByIdAsync(int id);
    Task<IEnumerable<T>> GetAllAsync();
}

public class CachedReadOnlyRepository<T> : IReadOnlyRepository<T> where T : class
{
    private readonly IReadOnlyRepository<T> _inner;
    private readonly IDistributedCache _cache;
    
    public CachedReadOnlyRepository(IReadOnlyRepository<T> inner, IDistributedCache cache)
    {
        _inner = inner;
        _cache = cache;
    }
    
    public async Task<T?> GetByIdAsync(int id)
    {
        var key = $"{typeof(T).Name}:{id}";
        // check cache -> fallback to inner -> cache result
        return await _inner.GetByIdAsync(id);
    }
    
    public async Task<IEnumerable<T>> GetAllAsync()
        => await _inner.GetAllAsync();
}
```

## ขั้นตอนที่ 620: I - Interface Segregation Principle (ISP)

**"Clients should not be forced to depend on interfaces they don't use"**

```csharp
// ❌ BAD: Fat interface
public interface IWorker_BAD
{
    void Work();
    void Eat();
    void Sleep();
    void Report();
    void AttendMeeting();
    void TakeVacation();
}

// Robot ต้อง implement methods ที่ไม่เกี่ยวข้อง
public class Robot_BAD : IWorker_BAD
{
    public void Work() => Console.WriteLine("Working");
    public void Eat() => throw new NotImplementedException(); // ❌ Robot ไม่กิน!
    public void Sleep() => throw new NotImplementedException(); // ❌ Robot ไม่นอน!
    public void Report() => Console.WriteLine("Reporting");
    public void AttendMeeting() => throw new NotImplementedException();
    public void TakeVacation() => throw new NotImplementedException();
}

// ✅ GOOD: แยก interface ตาม responsibility
public interface IWorkable
{
    void Work();
    void Report();
}

public interface IRestable
{
    void Eat();
    void Sleep();
    void TakeVacation();
}

public interface IMeetable
{
    void AttendMeeting();
    void PresentReport();
}

// Human implements all
public class HumanWorker : IWorkable, IRestable, IMeetable
{
    public string Name { get; init; } = "";
    
    public void Work() => Console.WriteLine($"{Name} is working");
    public void Report() => Console.WriteLine($"{Name} is reporting");
    public void Eat() => Console.WriteLine($"{Name} is eating");
    public void Sleep() => Console.WriteLine($"{Name} is sleeping");
    public void TakeVacation() => Console.WriteLine($"{Name} is on vacation");
    public void AttendMeeting() => Console.WriteLine($"{Name} attends meeting");
    public void PresentReport() => Console.WriteLine($"{Name} presents report");
}

// Robot implements only what's relevant
public class RobotWorker : IWorkable
{
    public string Id { get; init; } = "";
    
    public void Work() => Console.WriteLine($"Robot {Id} is working 24/7");
    public void Report() => Console.WriteLine($"Robot {Id}: Status OK");
}

// Real-world ISP: Repository interfaces
public interface IReadRepository<T>
{
    Task<T?> GetByIdAsync(int id, CancellationToken ct = default);
    Task<IEnumerable<T>> GetAllAsync(CancellationToken ct = default);
    Task<int> CountAsync(CancellationToken ct = default);
}

public interface IWriteRepository<T>
{
    Task<int> CreateAsync(T entity, CancellationToken ct = default);
    Task UpdateAsync(T entity, CancellationToken ct = default);
    Task DeleteAsync(int id, CancellationToken ct = default);
}

public interface IRepository<T> : IReadRepository<T>, IWriteRepository<T>
{
    Task<IEnumerable<T>> FindAsync(
        System.Linq.Expressions.Expression<Func<T, bool>> predicate,
        CancellationToken ct = default);
}

// Read-only service ใช้เฉพาะ IReadRepository
public class ProductQueryService
{
    private readonly IReadRepository<Product> _repo;  // ไม่รู้เรื่อง write operations
    
    public ProductQueryService(IReadRepository<Product> repo) => _repo = repo;
    
    public async Task<ProductDto?> GetByIdAsync(int id)
    {
        var product = await _repo.GetByIdAsync(id);
        return product is null ? null : new ProductDto(product.Id, product.Name, product.Price);
    }
}

// Command service ใช้เฉพาะ IWriteRepository
public class ProductCommandService
{
    private readonly IWriteRepository<Product> _repo;
    
    public ProductCommandService(IWriteRepository<Product> repo) => _repo = repo;
    
    public async Task<int> CreateProductAsync(CreateProductDto dto)
    {
        var product = new Product { Name = dto.Name, Price = dto.Price };
        return await _repo.CreateAsync(product);
    }
}

// ISP in ASP.NET Core: IMiddleware vs RequestDelegate
public interface IHealthCheck
{
    Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext context, CancellationToken ct);
}

// Segregated HTTP interfaces
public interface IHttpGetHandler { Task<IResult> HandleGetAsync(); }
public interface IHttpPostHandler<TRequest> { Task<IResult> HandlePostAsync(TRequest request); }

public class ProductEndpoint : IHttpGetHandler, IHttpPostHandler<CreateProductDto>
{
    private readonly IProductRepository _repo;
    
    public ProductEndpoint(IProductRepository repo) => _repo = repo;
    
    public async Task<IResult> HandleGetAsync()
    {
        var products = await _repo.GetAllAsync();
        return Results.Ok(products);
    }
    
    public async Task<IResult> HandlePostAsync(CreateProductDto request)
    {
        var id = await _repo.CreateAsync(new Product { Name = request.Name, Price = request.Price });
        return Results.Created($"/api/products/{id}", new { id });
    }
}
```

## ขั้นตอนที่ 621: D - Dependency Inversion Principle (DIP)

**"Depend on abstractions, not concretions"**

```csharp
// ❌ BAD: High-level module depends on low-level module directly
public class OrderService_BAD
{
    // ❌ สร้าง concrete dependency ตรงๆ
    private readonly SqlOrderRepository _repo = new SqlOrderRepository("Server=...");
    private readonly SmtpEmailSender _email = new SmtpEmailSender("smtp.gmail.com", 587);
    private readonly FileLogger _logger = new FileLogger("app.log");
    
    public void ProcessOrder(Order order)
    {
        _logger.Log($"Processing order {order.Id}");
        _repo.Save(order);
        _email.Send(order.CustomerEmail, "Order Confirmed", "...");
    }
}

// ✅ GOOD: Depend on abstractions
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(int id);
    Task<int> SaveAsync(Order order);
}

public interface IEmailSender
{
    Task SendAsync(string to, string subject, string body);
}

public interface ILogger<T>
{
    void LogInformation(string message, params object[] args);
    void LogError(Exception ex, string message, params object[] args);
}

// High-level module depends on abstractions only
public class OrderService
{
    private readonly IOrderRepository _repo;
    private readonly IEmailSender _email;
    private readonly ILogger<OrderService> _logger;
    
    // Dependencies injected from outside (DIP + DI)
    public OrderService(
        IOrderRepository repo,
        IEmailSender email,
        ILogger<OrderService> logger)
    {
        _repo = repo;
        _email = email;
        _logger = logger;
    }
    
    public async Task<Result<int>> ProcessOrderAsync(Order order)
    {
        _logger.LogInformation("Processing order {OrderId}", order.Id);
        
        try
        {
            order.Status = OrderStatus.Processing;
            var id = await _repo.SaveAsync(order);
            
            await _email.SendAsync(
                order.CustomerEmail,
                "Order Confirmed",
                $"Your order #{id} has been confirmed.");
            
            _logger.LogInformation("Order {OrderId} processed successfully", id);
            return Result<int>.Success(id);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to process order {OrderId}", order.Id);
            return Result<int>.Failure(ex.Message);
        }
    }
}

// Low-level modules implement abstractions
public class SqlOrderRepository : IOrderRepository
{
    private readonly IDbConnection _db;
    
    public SqlOrderRepository(IDbConnection db) => _db = db;
    
    public async Task<Order?> GetByIdAsync(int id)
        => await _db.QueryFirstOrDefaultAsync<Order>(
            "SELECT * FROM Orders WHERE Id = @Id", new { Id = id });
    
    public async Task<int> SaveAsync(Order order)
        => await _db.ExecuteScalarAsync<int>(
            "INSERT INTO Orders (...) OUTPUT INSERTED.Id VALUES (...)", order);
}

public class SmtpEmailSender : IEmailSender
{
    private readonly string _host;
    private readonly int _port;
    
    public SmtpEmailSender(string host, int port) { _host = host; _port = port; }
    
    public async Task SendAsync(string to, string subject, string body)
    {
        using var client = new System.Net.Mail.SmtpClient(_host, _port);
        await client.SendMailAsync("noreply@myapp.com", to, subject, body);
    }
}

// Testable: Mock implementations
public class FakeOrderRepository : IOrderRepository
{
    private readonly Dictionary<int, Order> _store = [];
    private int _nextId = 1;
    
    public Task<Order?> GetByIdAsync(int id)
        => Task.FromResult(_store.GetValueOrDefault(id));
    
    public Task<int> SaveAsync(Order order)
    {
        order.Id = _nextId++;
        _store[order.Id] = order;
        return Task.FromResult(order.Id);
    }
}

public class FakeEmailSender : IEmailSender
{
    public List<(string To, string Subject, string Body)> SentEmails { get; } = [];
    
    public Task SendAsync(string to, string subject, string body)
    {
        SentEmails.Add((to, subject, body));
        return Task.CompletedTask;
    }
}
```

## ขั้นตอนที่ 622: SOLID ทั้งหมดร่วมกัน - E-Commerce Example

```csharp
// ตัวอย่างการใช้ SOLID ทั้ง 5 หลักการร่วมกัน

// === ABSTRACTIONS (DIP: depend on these) ===

public interface IProductRepository  // ISP: เฉพาะ product operations
{
    Task<Product?> GetByIdAsync(int id);
    Task<bool> IsInStockAsync(int productId, int quantity);
    Task UpdateStockAsync(int productId, int quantityChange);
}

public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(int id);
    Task<int> CreateAsync(Order order);
    Task UpdateStatusAsync(int orderId, OrderStatus status);
}

public interface IPaymentGateway  // ISP: segregated from order/product concerns
{
    Task<PaymentResult> ChargeAsync(string customerId, decimal amount, string currency);
    Task<RefundResult> RefundAsync(string chargeId, decimal amount);
}

public interface INotificationService  // ISP
{
    Task SendOrderConfirmationAsync(Order order, Customer customer);
    Task SendShippingUpdateAsync(Order order, string trackingNumber);
}

// === DOMAIN MODELS ===

public enum OrderStatus { Pending, PaymentProcessed, Fulfilling, Shipped, Delivered, Cancelled }

public class Order
{
    public int Id { get; set; }
    public string CustomerId { get; set; } = "";
    public List<OrderLine> Lines { get; set; } = [];
    public OrderStatus Status { get; set; } = OrderStatus.Pending;
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public string? PaymentId { get; set; }
    
    public decimal Subtotal => Lines.Sum(l => l.UnitPrice * l.Quantity);
    public decimal Tax => Subtotal * 0.07m;
    public decimal Total => Subtotal + Tax;
}

public record OrderLine(int ProductId, string ProductName, decimal UnitPrice, int Quantity);

// === DISCOUNT STRATEGIES (OCP: open for extension) ===

public interface IDiscountPolicy
{
    string Name { get; }
    decimal Calculate(Order order, Customer customer);
    bool IsApplicable(Order order, Customer customer);
}

public class VolumeDiscount : IDiscountPolicy
{
    public string Name => "Volume Discount";
    private readonly int _minItems;
    private readonly decimal _rate;
    
    public VolumeDiscount(int minItems = 5, decimal rate = 0.05m)
    {
        _minItems = minItems;
        _rate = rate;
    }
    
    public decimal Calculate(Order order, Customer customer)
        => order.Subtotal * _rate;
    
    public bool IsApplicable(Order order, Customer customer)
        => order.Lines.Sum(l => l.Quantity) >= _minItems;
}

public class LoyaltyDiscount : IDiscountPolicy
{
    public string Name => "Loyalty Discount";
    
    public decimal Calculate(Order order, Customer customer)
        => customer.LoyaltyPoints >= 1000 ? order.Subtotal * 0.1m : order.Subtotal * 0.05m;
    
    public bool IsApplicable(Order order, Customer customer)
        => customer.LoyaltyPoints >= 500;
}

// === VALIDATORS (SRP + OCP) ===

public interface IOrderValidator
{
    Task<ValidationResult> ValidateAsync(Order order, Customer customer);
}

public class StockValidator : IOrderValidator
{
    private readonly IProductRepository _products;
    
    public StockValidator(IProductRepository products) => _products = products;
    
    public async Task<ValidationResult> ValidateAsync(Order order, Customer customer)
    {
        var errors = new List<string>();
        
        foreach (var line in order.Lines)
        {
            if (!await _products.IsInStockAsync(line.ProductId, line.Quantity))
                errors.Add($"Product '{line.ProductName}' is out of stock");
        }
        
        return errors.Count == 0
            ? ValidationResult.Success()
            : ValidationResult.Failure(errors);
    }
}

public class CustomerValidator : IOrderValidator
{
    public Task<ValidationResult> ValidateAsync(Order order, Customer customer)
    {
        var errors = new List<string>();
        
        if (string.IsNullOrEmpty(customer.Email))
            errors.Add("Customer email is required");
        
        if (customer.IsBlocked)
            errors.Add("Customer account is blocked");
        
        if (order.Total > customer.CreditLimit)
            errors.Add($"Order total exceeds credit limit ({customer.CreditLimit:C})");
        
        return Task.FromResult(errors.Count == 0
            ? ValidationResult.Success()
            : ValidationResult.Failure(errors));
    }
}

public class MinimumOrderValidator : IOrderValidator
{
    private readonly decimal _minimumAmount;
    
    public MinimumOrderValidator(decimal minimumAmount = 100m)
    {
        _minimumAmount = minimumAmount;
    }
    
    public Task<ValidationResult> ValidateAsync(Order order, Customer customer)
    {
        return Task.FromResult(order.Total >= _minimumAmount
            ? ValidationResult.Success()
            : ValidationResult.Failure([$"Minimum order amount is {_minimumAmount:C}"]));
    }
}

// === ORDER SERVICE (SRP + DIP) ===

public class OrderProcessingService  // SRP: only processes orders
{
    private readonly IOrderRepository _orders;
    private readonly IProductRepository _products;
    private readonly IPaymentGateway _payment;
    private readonly INotificationService _notifications;
    private readonly IEnumerable<IOrderValidator> _validators;  // OCP: extensible
    private readonly IEnumerable<IDiscountPolicy> _discounts;   // OCP: extensible
    private readonly ILogger<OrderProcessingService> _logger;
    
    public OrderProcessingService(
        IOrderRepository orders,
        IProductRepository products,
        IPaymentGateway payment,
        INotificationService notifications,
        IEnumerable<IOrderValidator> validators,
        IEnumerable<IDiscountPolicy> discounts,
        ILogger<OrderProcessingService> logger)
    {
        _orders = orders;
        _products = products;
        _payment = payment;
        _notifications = notifications;
        _validators = validators;
        _discounts = discounts;
        _logger = logger;
    }
    
    public async Task<Result<Order>> PlaceOrderAsync(Order order, Customer customer)
    {
        _logger.LogInformation("Placing order for customer {CustomerId}", customer.Id);
        
        // Validate (all validators must pass)
        foreach (var validator in _validators)
        {
            var validation = await validator.ValidateAsync(order, customer);
            if (!validation.IsValid)
                return Result<Order>.Failure(validation.Errors);
        }
        
        // Apply discounts
        var discount = _discounts
            .Where(d => d.IsApplicable(order, customer))
            .Sum(d => d.Calculate(order, customer));
        
        if (discount > 0)
            _logger.LogInformation("Applied discount: {Discount:F2}", discount);
        
        // Process payment
        var paymentResult = await _payment.ChargeAsync(
            customer.Id, order.Total - discount, "THB");
        
        if (!paymentResult.Success)
        {
            _logger.LogWarning("Payment failed for customer {CustomerId}: {Error}", 
                customer.Id, paymentResult.ErrorMessage);
            return Result<Order>.Failure(paymentResult.ErrorMessage ?? "Payment failed");
        }
        
        order.PaymentId = paymentResult.TransactionId;
        order.Status = OrderStatus.PaymentProcessed;
        
        // Save order
        var orderId = await _orders.CreateAsync(order);
        order.Id = orderId;
        
        // Update stock
        foreach (var line in order.Lines)
            await _products.UpdateStockAsync(line.ProductId, -line.Quantity);
        
        // Send confirmation
        await _notifications.SendOrderConfirmationAsync(order, customer);
        
        _logger.LogInformation("Order {OrderId} placed successfully", orderId);
        return Result<Order>.Success(order);
    }
}

// === DI REGISTRATION ===

// Program.cs / Startup.cs
static void RegisterOrderServices(IServiceCollection services)
{
    // Repositories (DIP: abstractions registered)
    services.AddScoped<IProductRepository, SqlProductRepository>();
    services.AddScoped<IOrderRepository, SqlOrderRepository>();
    
    // Payment gateway
    services.AddScoped<IPaymentGateway, StripePaymentGateway>();
    
    // Notifications
    services.AddScoped<INotificationService, EmailNotificationService>();
    
    // Validators (OCP: add more validators without changing OrderProcessingService)
    services.AddScoped<IOrderValidator, StockValidator>();
    services.AddScoped<IOrderValidator, CustomerValidator>();
    services.AddScoped<IOrderValidator, MinimumOrderValidator>();
    
    // Discount policies (OCP: add more discounts without changing OrderProcessingService)
    services.AddScoped<IDiscountPolicy, VolumeDiscount>();
    services.AddScoped<IDiscountPolicy, LoyaltyDiscount>();
    
    // Service
    services.AddScoped<OrderProcessingService>();
}
```

## ขั้นตอนที่ 623: สรุป SOLID Principles

```
SOLID Principles Summary
════════════════════════════════════════════════════════════════

S - Single Responsibility Principle
    "One class, one job"
    
    BEFORE: UserService (CRUD + Email + Log + Report + Validate)
    AFTER:  UserRepo + EmailService + Logger + ReportGen + Validator
    
    When violated: class เปลี่ยนบ่อยจากหลาย reasons, file ยาว 1000+ บรรทัด

O - Open/Closed Principle  
    "Extend without modifying"
    
    BEFORE: if-else หรือ switch ใน core logic
    AFTER:  Strategy/Plugin interface + register implementations
    
    When violated: ต้อง edit core class ทุกครั้งที่เพิ่ม feature ใหม่

L - Liskov Substitution Principle
    "Subtype must behave like supertype"
    
    BEFORE: Square extends Rectangle แต่ break Area invariant
    AFTER:  แยก hierarchy ตาม behavior; ไม่ throw NotImplementedException
    
    When violated: code ที่ใช้ subtype ต้อง check type ก่อน

I - Interface Segregation Principle
    "Small, focused interfaces"
    
    BEFORE: IWorker { Work + Eat + Sleep + ... } ← fat interface
    AFTER:  IWorkable + IRestable + ... ← segregated
    
    When violated: implement interface ต้อง throw NotImplementedException
                   หรือ method เปล่า

D - Dependency Inversion Principle
    "Depend on abstractions"
    
    BEFORE: class A { B b = new B(); }  ← concrete dependency
    AFTER:  class A(IB b) { ... }       ← abstract dependency
    
    When violated: ไม่สามารถ test โดยไม่ใช้ real DB/email/file

════════════════════════════════════════════════════════════════
SOLID ช่วยให้:
  ✓ Unit testable (mock ง่าย)
  ✓ Extensible (เพิ่ม feature ไม่ต้อง modify เดิม)
  ✓ Maintainable (เปลี่ยนได้อย่างปลอดภัย)
  ✓ Understandable (แต่ละ class มีหน้าที่ชัดเจน)
════════════════════════════════════════════════════════════════
```

## สรุป Part 21

ใน Part นี้คุณได้เรียนรู้:
- **SRP**: แต่ละ class มีหน้าที่เดียว
- **OCP**: ขยายได้โดยไม่แก้โค้ดเดิม ด้วย abstraction/interface
- **LSP**: Subtype ต้องทำงานแทน supertype ได้
- **ISP**: แยก interface ให้เล็กและเฉพาะเจาะจง
- **DIP**: Depend on abstraction ไม่ใช่ concretion
- ตัวอย่าง E-Commerce ที่ใช้ SOLID ทั้ง 5 หลักการร่วมกัน

## แบบฝึกหัด

1. Refactor `OrderService` ที่มีโค้ด 200+ บรรทัดให้ปฏิบัติตาม SRP
2. สร้าง notification system ที่ extensible ด้วย OCP (Email, SMS, Push, Slack)
3. ออกแบบ Animal hierarchy ที่ LSP compliant (Bird, Fish, Dog)
4. แยก `IUserService` ขนาดใหญ่ให้เป็น interfaces เล็กๆ ตาม ISP
5. แปลง class ที่ new concrete dependencies ตรงๆ ให้ใช้ DIP
