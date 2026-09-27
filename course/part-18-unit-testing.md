# Part 18: Unit Testing ด้วย xUnit (Steps 541-580)

## เป้าหมายการเรียนรู้
- Unit testing fundamentals และ xUnit
- Test-Driven Development (TDD)
- Mocking ด้วย Moq
- Test coverage และ assertions
- Integration testing
- Test organization และ best practices

---

## Step 541: xUnit Basics

```csharp
// Project: MyApp.Tests
// dotnet add package xunit
// dotnet add package xunit.runner.visualstudio
// dotnet add package Microsoft.NET.Test.Sdk

using Xunit;

// Test class — ไม่ต้องมี attribute (xUnit ค้นหา classes ที่มี [Fact]/[Theory])
public class CalculatorTests
{
    // [Fact] — test ที่รันเพียงครั้งเดียว ไม่มี parameters
    [Fact]
    public void Add_TwoPositiveNumbers_ReturnsSum()
    {
        // Arrange
        var calculator = new Calculator();
        
        // Act
        int result = calculator.Add(3, 4);
        
        // Assert
        Assert.Equal(7, result);
    }
    
    [Fact]
    public void Divide_ByZero_ThrowsDivideByZeroException()
    {
        var calc = new Calculator();
        
        Assert.Throws<DivideByZeroException>(() => calc.Divide(10, 0));
    }
    
    [Fact]
    public async Task GetDataAsync_ReturnsData()
    {
        var service = new DataService();
        
        string result = await service.GetDataAsync();
        
        Assert.NotNull(result);
        Assert.NotEmpty(result);
    }
    
    // [Theory] — parameterized test
    [Theory]
    [InlineData(3, 4, 7)]
    [InlineData(-1, 1, 0)]
    [InlineData(0, 0, 0)]
    [InlineData(int.MaxValue, 0, int.MaxValue)]
    public void Add_VariousInputs_ReturnsCorrectSum(int a, int b, int expected)
    {
        var calc = new Calculator();
        Assert.Equal(expected, calc.Add(a, b));
    }
    
    // [MemberData] — test data from property/field/method
    public static IEnumerable<object[]> MultiplyData => new List<object[]>
    {
        new object[] { 2, 3, 6 },
        new object[] { -2, 3, -6 },
        new object[] { 0, 100, 0 },
        new object[] { 7, 7, 49 },
    };
    
    [Theory]
    [MemberData(nameof(MultiplyData))]
    public void Multiply_VariousInputs_ReturnsCorrectProduct(int a, int b, int expected)
    {
        var calc = new Calculator();
        Assert.Equal(expected, calc.Multiply(a, b));
    }
    
    // [ClassData] — test data from a class
    [Theory]
    [ClassData(typeof(AddTestData))]
    public void Add_ClassData_ReturnsCorrectResult(int a, int b, int expected)
    {
        Assert.Equal(expected, new Calculator().Add(a, b));
    }
}

class AddTestData : IEnumerable<object[]>
{
    public IEnumerator<object[]> GetEnumerator()
    {
        yield return new object[] { 1, 2, 3 };
        yield return new object[] { 10, 20, 30 };
        yield return new object[] { -5, 5, 0 };
    }
    
    System.Collections.IEnumerator System.Collections.IEnumerable.GetEnumerator() => GetEnumerator();
}

// Production code
class Calculator
{
    public int Add(int a, int b) => a + b;
    public int Subtract(int a, int b) => a - b;
    public int Multiply(int a, int b) => a * b;
    public int Divide(int a, int b) => b == 0 ? throw new DivideByZeroException() : a / b;
}

class DataService
{
    public Task<string> GetDataAsync() => Task.FromResult("some data");
}
```

---

## Step 542: xUnit Assertions

```csharp
using Xunit;
using System.Collections.Generic;

public class AssertionExamples
{
    [Fact]
    public void BasicAssertions()
    {
        // Equality
        Assert.Equal(42, 42);
        Assert.NotEqual(42, 43);
        
        // Null checks
        Assert.Null(null);
        Assert.NotNull("hello");
        
        // Boolean
        Assert.True(1 == 1);
        Assert.False(1 == 2);
        
        // Type checks
        object obj = "hello";
        Assert.IsType<string>(obj);
        Assert.IsAssignableFrom<object>(obj);
        
        // String assertions
        string text = "Hello, World!";
        Assert.Contains("World", text);
        Assert.DoesNotContain("xyz", text);
        Assert.StartsWith("Hello", text);
        Assert.EndsWith("!", text);
        Assert.Matches(@"Hello, \w+!", text); // regex
    }
    
    [Fact]
    public void CollectionAssertions()
    {
        var list = new List<int> { 1, 2, 3, 4, 5 };
        
        Assert.Empty(new List<int>());
        Assert.NotEmpty(list);
        Assert.Single(new[] { "only one" });
        Assert.Contains(3, list);
        Assert.DoesNotContain(10, list);
        
        // All elements satisfy condition
        Assert.All(list, item => Assert.True(item > 0));
        
        // Collection equality
        Assert.Equal(new[] { 1, 2, 3 }, new[] { 1, 2, 3 });
        
        // Count
        Assert.Equal(5, list.Count);
    }
    
    [Fact]
    public void ExceptionAssertions()
    {
        // Throws specific exception
        var ex = Assert.Throws<ArgumentNullException>(() => 
            throw new ArgumentNullException("param"));
        
        Assert.Equal("param", ex.ParamName);
        
        // ThrowsAny (base class)
        Assert.ThrowsAny<Exception>(() => throw new InvalidOperationException());
        
        // Async throws
        // var asyncEx = await Assert.ThrowsAsync<Exception>(async () =>
        // {
        //     await Task.Delay(1);
        //     throw new Exception();
        // });
        
        // Does NOT throw
        var ex2 = Record.Exception(() => { int x = 1 + 1; });
        Assert.Null(ex2);
    }
    
    [Fact]
    public void FloatingPointAssertions()
    {
        // Floating point — use precision
        Assert.Equal(3.14, Math.PI, precision: 2);     // 2 decimal places
        Assert.Equal(1.0 / 3.0, 0.333, precision: 3);
        
        // InRange
        double value = 3.14159;
        Assert.InRange(value, 3.0, 4.0);
        Assert.NotInRange(value, 4.0, 5.0);
    }
    
    [Fact]
    public void CustomMessages()
    {
        int expected = 42;
        int actual = 43;
        
        // Custom failure message
        Assert.True(expected == actual, $"Expected {expected} but got {actual}");
    }
}
```

---

## Step 543: Test Fixtures and Lifecycle

```csharp
using Xunit;
using System;
using System.IO;

// IClassFixture<T> — shared setup/teardown for all tests in a class
public class DatabaseFixture : IDisposable
{
    public string ConnectionString { get; } = "Data Source=test.db";
    private readonly string _dbPath = "test.db";
    
    public DatabaseFixture()
    {
        Console.WriteLine("DatabaseFixture: Setting up test database");
        // Create test database, seed data, etc.
        File.WriteAllText(_dbPath, "test");
    }
    
    public void Dispose()
    {
        Console.WriteLine("DatabaseFixture: Cleaning up");
        File.Delete(_dbPath);
    }
}

// xUnit creates one instance of fixture per class
public class OrderRepositoryTests : IClassFixture<DatabaseFixture>
{
    private readonly DatabaseFixture _fixture;
    
    public OrderRepositoryTests(DatabaseFixture fixture)
    {
        _fixture = fixture;
    }
    
    [Fact]
    public void GetOrder_ExistingId_ReturnsOrder()
    {
        // Use _fixture.ConnectionString
        Console.WriteLine($"Running test with DB: {_fixture.ConnectionString}");
        Assert.True(true); // simplified
    }
}

// ICollectionFixture<T> — shared across multiple test classes
[CollectionDefinition("Database")]
public class DatabaseCollection : ICollectionFixture<DatabaseFixture> { }

[Collection("Database")]
public class UserRepositoryTests
{
    private readonly DatabaseFixture _fixture;
    
    public UserRepositoryTests(DatabaseFixture fixture) => _fixture = fixture;
    
    [Fact]
    public void GetUser_ReturnsUser()
    {
        // Shared database fixture
        Assert.NotNull(_fixture.ConnectionString);
    }
}

// Constructor/Dispose for per-test setup
public class ServiceTests : IDisposable
{
    private readonly TempFileService _service;
    private readonly string _tempDir;
    
    public ServiceTests()
    {
        // Called before EACH test
        _tempDir = Path.Combine(Path.GetTempPath(), Guid.NewGuid().ToString());
        Directory.CreateDirectory(_tempDir);
        _service = new TempFileService(_tempDir);
        Console.WriteLine($"Test setup: {_tempDir}");
    }
    
    public void Dispose()
    {
        // Called after EACH test
        Directory.Delete(_tempDir, recursive: true);
        Console.WriteLine("Test cleanup done");
    }
    
    [Fact]
    public void CreateFile_Succeeds()
    {
        _service.CreateFile("test.txt", "content");
        Assert.True(File.Exists(Path.Combine(_tempDir, "test.txt")));
    }
}

class TempFileService
{
    private readonly string _dir;
    public TempFileService(string dir) => _dir = dir;
    public void CreateFile(string name, string content) => File.WriteAllText(Path.Combine(_dir, name), content);
}

// IAsyncLifetime — async setup/teardown
public class AsyncSetupTests : IAsyncLifetime
{
    private string? _resourceUrl;
    
    public async Task InitializeAsync()
    {
        // Async setup
        await Task.Delay(10); // simulate async init
        _resourceUrl = "https://example.com/resource";
    }
    
    public async Task DisposeAsync()
    {
        // Async cleanup
        await Task.Delay(5);
        _resourceUrl = null;
    }
    
    [Fact]
    public void Resource_IsInitialized()
    {
        Assert.NotNull(_resourceUrl);
    }
}
```

---

## Step 544: Mocking with Moq

```csharp
// dotnet add package Moq
using Moq;
using Xunit;
using System.Threading.Tasks;

// Interfaces to mock
interface IEmailService
{
    Task SendEmailAsync(string to, string subject, string body);
    bool IsAvailable();
}

interface IUserRepository
{
    Task<User?> GetByIdAsync(int id);
    Task<User?> GetByEmailAsync(string email);
    Task SaveAsync(User user);
    bool Exists(int id);
}

record User(int Id, string Name, string Email);

// Service under test
class UserService
{
    private readonly IUserRepository _repo;
    private readonly IEmailService _email;
    
    public UserService(IUserRepository repo, IEmailService email)
    {
        _repo = repo;
        _email = email;
    }
    
    public async Task<User> RegisterAsync(string name, string email)
    {
        // Check if email exists
        var existing = await _repo.GetByEmailAsync(email);
        if (existing != null)
            throw new InvalidOperationException($"Email {email} already registered");
        
        var user = new User(0, name, email);
        await _repo.SaveAsync(user);
        
        if (_email.IsAvailable())
            await _email.SendEmailAsync(email, "Welcome!", $"Hello {name}, welcome!");
        
        return user;
    }
    
    public async Task<User> GetUserAsync(int id)
    {
        var user = await _repo.GetByIdAsync(id);
        return user ?? throw new KeyNotFoundException($"User {id} not found");
    }
}

// Tests with Moq
public class UserServiceTests
{
    private readonly Mock<IUserRepository> _mockRepo;
    private readonly Mock<IEmailService> _mockEmail;
    private readonly UserService _service;
    
    public UserServiceTests()
    {
        _mockRepo = new Mock<IUserRepository>();
        _mockEmail = new Mock<IEmailService>();
        _service = new UserService(_mockRepo.Object, _mockEmail.Object);
    }
    
    [Fact]
    public async Task RegisterAsync_NewEmail_CreatesUser()
    {
        // Arrange
        _mockRepo.Setup(r => r.GetByEmailAsync("alice@test.com"))
            .ReturnsAsync((User?)null); // no existing user
        
        _mockRepo.Setup(r => r.SaveAsync(It.IsAny<User>()))
            .Returns(Task.CompletedTask);
        
        _mockEmail.Setup(e => e.IsAvailable()).Returns(true);
        
        _mockEmail.Setup(e => e.SendEmailAsync(
            It.IsAny<string>(),
            It.IsAny<string>(),
            It.IsAny<string>()))
            .Returns(Task.CompletedTask);
        
        // Act
        User user = await _service.RegisterAsync("Alice", "alice@test.com");
        
        // Assert
        Assert.Equal("Alice", user.Name);
        Assert.Equal("alice@test.com", user.Email);
        
        // Verify interactions
        _mockRepo.Verify(r => r.SaveAsync(It.IsAny<User>()), Times.Once);
        _mockEmail.Verify(e => e.SendEmailAsync("alice@test.com", "Welcome!", It.IsAny<string>()), Times.Once);
    }
    
    [Fact]
    public async Task RegisterAsync_DuplicateEmail_ThrowsException()
    {
        // Arrange — email already exists
        var existingUser = new User(1, "Existing", "taken@test.com");
        _mockRepo.Setup(r => r.GetByEmailAsync("taken@test.com"))
            .ReturnsAsync(existingUser);
        
        // Act & Assert
        var ex = await Assert.ThrowsAsync<InvalidOperationException>(
            () => _service.RegisterAsync("New User", "taken@test.com"));
        
        Assert.Contains("taken@test.com", ex.Message);
        
        // Email should NOT be sent
        _mockEmail.Verify(e => e.SendEmailAsync(
            It.IsAny<string>(), It.IsAny<string>(), It.IsAny<string>()),
            Times.Never);
    }
    
    [Fact]
    public async Task RegisterAsync_EmailUnavailable_SkipsSendEmail()
    {
        // Arrange
        _mockRepo.Setup(r => r.GetByEmailAsync(It.IsAny<string>())).ReturnsAsync((User?)null);
        _mockRepo.Setup(r => r.SaveAsync(It.IsAny<User>())).Returns(Task.CompletedTask);
        _mockEmail.Setup(e => e.IsAvailable()).Returns(false); // Email down
        
        // Act
        await _service.RegisterAsync("Bob", "bob@test.com");
        
        // Assert — email not sent
        _mockEmail.Verify(e => e.SendEmailAsync(
            It.IsAny<string>(), It.IsAny<string>(), It.IsAny<string>()),
            Times.Never);
    }
    
    [Fact]
    public async Task GetUserAsync_NotFound_ThrowsKeyNotFoundException()
    {
        _mockRepo.Setup(r => r.GetByIdAsync(999)).ReturnsAsync((User?)null);
        
        await Assert.ThrowsAsync<KeyNotFoundException>(() => _service.GetUserAsync(999));
    }
}

// Advanced Moq features
public class AdvancedMockTests
{
    [Fact]
    public void MockCallbacks_TrackCalls()
    {
        var mock = new Mock<IUserRepository>();
        var savedUsers = new List<User>();
        
        mock.Setup(r => r.SaveAsync(It.IsAny<User>()))
            .Callback<User>(u => savedUsers.Add(u))
            .Returns(Task.CompletedTask);
        
        // ... call the service ...
        
        Assert.Empty(savedUsers); // Not called yet
    }
    
    [Fact]
    public void MockSequence_ReturnsOnEachCall()
    {
        var mock = new Mock<IUserRepository>();
        var user1 = new User(1, "Alice", "alice@test.com");
        var user2 = new User(2, "Bob", "bob@test.com");
        
        // Returns different values on successive calls
        mock.SetupSequence(r => r.GetByIdAsync(1))
            .ReturnsAsync(user1)    // 1st call
            .ReturnsAsync(user2)    // 2nd call
            .ThrowsAsync(new Exception("Not found")); // 3rd call
    }
    
    [Fact]
    public void MockStrict_FailsOnUnexpectedCall()
    {
        var strict = new Mock<IEmailService>(MockBehavior.Strict);
        
        // Only setup exactly what will be called
        strict.Setup(e => e.IsAvailable()).Returns(true);
        
        Assert.True(strict.Object.IsAvailable());
        
        // Calling anything not set up will throw
        Assert.Throws<MockException>(() => strict.Object.IsAvailable() == false);
    }
}
```

---

## Step 545: Test-Driven Development (TDD)

```csharp
// TDD Cycle: RED → GREEN → REFACTOR
// 1. Write a failing test (RED)
// 2. Write minimum code to pass (GREEN)
// 3. Refactor while keeping tests green

// Example: Building a Stack<T> with TDD

// Step 1: RED — write test first
public class StackTests
{
    [Fact]
    public void NewStack_IsEmpty()
    {
        var stack = new Stack<int>();
        Assert.True(stack.IsEmpty);
    }
    
    [Fact]
    public void Push_OneItem_IsNotEmpty()
    {
        var stack = new Stack<int>();
        stack.Push(1);
        Assert.False(stack.IsEmpty);
        Assert.Equal(1, stack.Count);
    }
    
    [Fact]
    public void Pop_EmptyStack_ThrowsInvalidOperation()
    {
        var stack = new Stack<int>();
        Assert.Throws<InvalidOperationException>(() => stack.Pop());
    }
    
    [Fact]
    public void Pop_OneItem_ReturnsItem()
    {
        var stack = new Stack<int>();
        stack.Push(42);
        Assert.Equal(42, stack.Pop());
    }
    
    [Fact]
    public void Pop_MultipleItems_ReturnsLIFO()
    {
        var stack = new Stack<int>();
        stack.Push(1);
        stack.Push(2);
        stack.Push(3);
        
        Assert.Equal(3, stack.Pop());
        Assert.Equal(2, stack.Pop());
        Assert.Equal(1, stack.Pop());
        Assert.True(stack.IsEmpty);
    }
    
    [Fact]
    public void Peek_DoesNotRemoveItem()
    {
        var stack = new Stack<int>();
        stack.Push(99);
        
        Assert.Equal(99, stack.Peek());
        Assert.Equal(99, stack.Peek()); // Still there
        Assert.Equal(1, stack.Count);
    }
    
    [Fact]
    public void Contains_ExistingItem_ReturnsTrue()
    {
        var stack = new Stack<int>();
        stack.Push(1);
        stack.Push(2);
        stack.Push(3);
        
        Assert.True(stack.Contains(2));
        Assert.False(stack.Contains(99));
    }
}

// Step 2: GREEN — minimum implementation
class Stack<T>
{
    private readonly List<T> _items = new();
    
    public bool IsEmpty => _items.Count == 0;
    public int Count => _items.Count;
    
    public void Push(T item) => _items.Add(item);
    
    public T Pop()
    {
        if (IsEmpty) throw new InvalidOperationException("Stack is empty");
        T item = _items[^1];
        _items.RemoveAt(_items.Count - 1);
        return item;
    }
    
    public T Peek()
    {
        if (IsEmpty) throw new InvalidOperationException("Stack is empty");
        return _items[^1];
    }
    
    public bool Contains(T item) => _items.Contains(item);
}

// Step 3: REFACTOR — improve without breaking tests
// (Tests act as a safety net)
```

---

## Step 546: Integration Testing

```csharp
// Integration tests — test real dependencies (DB, HTTP, etc.)

// ASP.NET Core Integration Testing
// dotnet add package Microsoft.AspNetCore.Mvc.Testing

using Microsoft.AspNetCore.Mvc.Testing;
using System.Net.Http.Json;
using Xunit;

// Test with real HTTP calls
public class ApiIntegrationTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;
    
    public ApiIntegrationTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.CreateClient();
    }
    
    [Fact]
    public async Task GetProducts_Returns200WithProducts()
    {
        var response = await _client.GetAsync("/api/products");
        
        response.EnsureSuccessStatusCode();
        var products = await response.Content.ReadFromJsonAsync<List<Product>>();
        Assert.NotNull(products);
        Assert.NotEmpty(products!);
    }
    
    [Fact]
    public async Task CreateProduct_ValidData_Returns201()
    {
        var newProduct = new Product { Name = "Test Widget", Price = 9.99m };
        
        var response = await _client.PostAsJsonAsync("/api/products", newProduct);
        
        Assert.Equal(System.Net.HttpStatusCode.Created, response.StatusCode);
        
        var created = await response.Content.ReadFromJsonAsync<Product>();
        Assert.NotNull(created);
        Assert.True(created!.Id > 0);
        Assert.Equal("Test Widget", created.Name);
    }
    
    [Fact]
    public async Task GetProduct_NotFound_Returns404()
    {
        var response = await _client.GetAsync("/api/products/99999");
        Assert.Equal(System.Net.HttpStatusCode.NotFound, response.StatusCode);
    }
}

// Custom WebApplicationFactory for testing with custom config
public class TestWebApplicationFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(Microsoft.AspNetCore.Hosting.IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // Replace real DB with in-memory DB for tests
            // services.Remove(services.SingleOrDefault(d => d.ServiceType == typeof(DbContext))!);
            // services.AddDbContext<AppDbContext>(opt => opt.UseInMemoryDatabase("TestDb"));
        });
        
        builder.UseEnvironment("Testing");
    }
}

class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public decimal Price { get; set; }
}

// Placeholder for ASP.NET Core minimal API
class Program { }
```

---

## Step 547: Test Organization และ Best Practices

```csharp
// BEST PRACTICES for Unit Testing

// 1. Naming: MethodName_Scenario_ExpectedResult
// Good:
[Fact] public void Add_TwoNegativeNumbers_ReturnsNegativeSum() { }
[Fact] public void GetUser_UserNotFound_ThrowsNotFoundException() { }
[Fact] public void IsEmailValid_EmptyString_ReturnsFalse() { }

// Bad:
[Fact] public void Test1() { }
[Fact] public void AddTest() { }

// 2. AAA Pattern: Arrange, Act, Assert
[Fact]
public void Transfer_ValidAmount_UpdatesBothBalances()
{
    // Arrange
    var source = new BankAccount(1000m);
    var target = new BankAccount(500m);
    
    // Act
    source.TransferTo(target, 200m);
    
    // Assert
    Assert.Equal(800m, source.Balance);
    Assert.Equal(700m, target.Balance);
}

// 3. One assertion per test (ideally)
[Fact] public void AccountCreated_HasZeroBalance() => Assert.Equal(0, new BankAccount().Balance);
[Fact] public void AccountCreated_IsActive() => Assert.True(new BankAccount().IsActive);
[Fact] public void AccountCreated_HasNullOwner() => Assert.Null(new BankAccount().Owner);

// 4. Don't test implementation details, test behavior
// Bad: tests that "how" something works (fragile)
// Good: tests that "what" the output is

// 5. Use descriptive test data
[Theory]
[InlineData("", false)]          // Empty email
[InlineData("notanemail", false)] // No @ symbol
[InlineData("@no-local.com", false)] // No local part
[InlineData("valid@email.com", true)] // Valid
[InlineData("a+b@c.co.uk", true)]    // Valid with + and multiple dots
public void IsEmailValid_Returns(string email, bool expected)
{
    Assert.Equal(expected, EmailValidator.IsValid(email));
}

// 6. Use Skip for known failures
[Fact(Skip = "Feature not yet implemented")]
public void NewFeature_Works() { }

// 7. Use Trait for categorization
[Fact]
[Trait("Category", "Fast")]
[Trait("Category", "Unit")]
public void FastUnitTest() { }

[Fact]
[Trait("Category", "Slow")]
[Trait("Category", "Integration")]
public void SlowIntegrationTest() { }

// Run only fast tests: dotnet test --filter "Category=Fast"

// 8. Test edge cases
public class PasswordValidatorTests
{
    [Theory]
    [InlineData(null, false)]          // null
    [InlineData("", false)]            // empty
    [InlineData("short", false)]       // too short
    [InlineData("nouppercase1!", false)] // no uppercase
    [InlineData("NOLOWERCASE1!", false)] // no lowercase
    [InlineData("NoNumbers!", false)]   // no number
    [InlineData("NoSpecial1", false)]   // no special char
    [InlineData("Valid123!", true)]     // valid
    [InlineData("A1!bbbbb", true)]      // minimum valid
    public void Validate_Returns(string? password, bool expected)
    {
        Assert.Equal(expected, PasswordValidator.IsValid(password));
    }
}

static class PasswordValidator
{
    public static bool IsValid(string? password)
    {
        if (string.IsNullOrEmpty(password) || password.Length < 8) return false;
        return password.Any(char.IsUpper) &&
               password.Any(char.IsLower) &&
               password.Any(char.IsDigit) &&
               password.Any(c => "!@#$%^&*".Contains(c));
    }
}

static class EmailValidator
{
    public static bool IsValid(string email) =>
        !string.IsNullOrEmpty(email) &&
        email.Contains('@') &&
        email.IndexOf('@') > 0 &&
        email.IndexOf('@') < email.Length - 1;
}

class BankAccount
{
    public decimal Balance { get; private set; }
    public bool IsActive { get; } = true;
    public string? Owner { get; set; }
    
    public BankAccount(decimal initialBalance = 0) => Balance = initialBalance;
    
    public void TransferTo(BankAccount target, decimal amount)
    {
        if (amount > Balance) throw new InvalidOperationException("Insufficient funds");
        Balance -= amount;
        target.Balance += amount;
    }
}
```

---

## Step 548: โปรแกรมตัวอย่าง — Shopping Cart with Full TDD

```csharp
using Xunit;
using Moq;
using System.Collections.Generic;
using System.Linq;

// ============================================================
// Shopping Cart — Full TDD Example
// ============================================================

// Domain
record Product(int Id, string Name, decimal Price, int StockQuantity);
record CartItem(Product Product, int Quantity)
{
    public decimal Subtotal => Product.Price * Quantity;
}

interface IProductRepository
{
    Product? GetById(int id);
    bool UpdateStock(int productId, int newQuantity);
}

interface IDiscountService
{
    decimal CalculateDiscount(decimal total, string? promoCode);
}

// Service under test
class ShoppingCart
{
    private readonly Dictionary<int, CartItem> _items = new();
    private readonly IProductRepository _products;
    private readonly IDiscountService _discounts;
    
    public ShoppingCart(IProductRepository products, IDiscountService discounts)
    {
        _products = products;
        _discounts = discounts;
    }
    
    public IReadOnlyList<CartItem> Items => _items.Values.ToList().AsReadOnly();
    public int ItemCount => _items.Values.Sum(i => i.Quantity);
    public decimal Subtotal => _items.Values.Sum(i => i.Subtotal);
    
    public void AddItem(int productId, int quantity = 1)
    {
        if (quantity <= 0) throw new ArgumentOutOfRangeException(nameof(quantity), "Must be positive");
        
        var product = _products.GetById(productId)
            ?? throw new KeyNotFoundException($"Product {productId} not found");
        
        if (product.StockQuantity < quantity)
            throw new InvalidOperationException($"Insufficient stock for {product.Name}");
        
        if (_items.TryGetValue(productId, out var existing))
        {
            int newQty = existing.Quantity + quantity;
            if (product.StockQuantity < newQty)
                throw new InvalidOperationException($"Insufficient stock for {product.Name}");
            _items[productId] = existing with { Quantity = newQty };
        }
        else
        {
            _items[productId] = new CartItem(product, quantity);
        }
    }
    
    public void RemoveItem(int productId, int quantity = 1)
    {
        if (!_items.TryGetValue(productId, out var item))
            throw new KeyNotFoundException($"Product {productId} not in cart");
        
        if (quantity >= item.Quantity)
            _items.Remove(productId);
        else
            _items[productId] = item with { Quantity = item.Quantity - quantity };
    }
    
    public void Clear() => _items.Clear();
    
    public decimal GetTotal(string? promoCode = null)
    {
        decimal discount = _discounts.CalculateDiscount(Subtotal, promoCode);
        return Subtotal - discount;
    }
    
    public OrderSummary Checkout(string? promoCode = null)
    {
        if (_items.Count == 0) throw new InvalidOperationException("Cart is empty");
        
        decimal discount = _discounts.CalculateDiscount(Subtotal, promoCode);
        decimal total = Subtotal - discount;
        
        // Update stock
        foreach (var item in _items.Values)
            _products.UpdateStock(item.Product.Id, item.Product.StockQuantity - item.Quantity);
        
        var summary = new OrderSummary(
            Items: Items.ToList(),
            Subtotal: Subtotal,
            Discount: discount,
            Total: total);
        
        Clear();
        return summary;
    }
}

record OrderSummary(IList<CartItem> Items, decimal Subtotal, decimal Discount, decimal Total);

// Tests
public class ShoppingCartTests
{
    private readonly Mock<IProductRepository> _mockRepo;
    private readonly Mock<IDiscountService> _mockDiscounts;
    private readonly ShoppingCart _cart;
    
    private readonly Product _laptop = new(1, "Laptop", 999m, 10);
    private readonly Product _mouse  = new(2, "Mouse", 29.99m, 50);
    private readonly Product _lowStock = new(3, "Rare Item", 199m, 1);
    
    public ShoppingCartTests()
    {
        _mockRepo = new Mock<IProductRepository>();
        _mockDiscounts = new Mock<IDiscountService>();
        
        // Default setups
        _mockRepo.Setup(r => r.GetById(1)).Returns(_laptop);
        _mockRepo.Setup(r => r.GetById(2)).Returns(_mouse);
        _mockRepo.Setup(r => r.GetById(3)).Returns(_lowStock);
        _mockRepo.Setup(r => r.GetById(999)).Returns((Product?)null);
        _mockDiscounts.Setup(d => d.CalculateDiscount(It.IsAny<decimal>(), null)).Returns(0);
        
        _cart = new ShoppingCart(_mockRepo.Object, _mockDiscounts.Object);
    }
    
    [Fact] public void NewCart_IsEmpty()
    {
        Assert.Empty(_cart.Items);
        Assert.Equal(0, _cart.ItemCount);
        Assert.Equal(0, _cart.Subtotal);
    }
    
    [Fact] public void AddItem_ValidProduct_AddsToCart()
    {
        _cart.AddItem(1);
        
        Assert.Single(_cart.Items);
        Assert.Equal(1, _cart.ItemCount);
        Assert.Equal(_laptop.Price, _cart.Subtotal);
    }
    
    [Fact] public void AddItem_ProductNotFound_ThrowsKeyNotFound()
    {
        Assert.Throws<KeyNotFoundException>(() => _cart.AddItem(999));
    }
    
    [Fact] public void AddItem_InsufficientStock_ThrowsInvalidOperation()
    {
        Assert.Throws<InvalidOperationException>(() => _cart.AddItem(3, 5)); // stock = 1
    }
    
    [Fact] public void AddItem_SameProductTwice_CombinesQuantity()
    {
        _cart.AddItem(1, 2);
        _cart.AddItem(1, 3);
        
        Assert.Single(_cart.Items);
        Assert.Equal(5, _cart.Items[0].Quantity);
    }
    
    [Fact] public void AddItem_MultipleProducts_AddsAll()
    {
        _cart.AddItem(1, 1);
        _cart.AddItem(2, 2);
        
        Assert.Equal(2, _cart.Items.Count);
        Assert.Equal(3, _cart.ItemCount);
        Assert.Equal(_laptop.Price + _mouse.Price * 2, _cart.Subtotal);
    }
    
    [Fact] public void RemoveItem_PartialQuantity_DecreasesCount()
    {
        _cart.AddItem(2, 3);
        _cart.RemoveItem(2, 1);
        
        Assert.Equal(2, _cart.Items[0].Quantity);
    }
    
    [Fact] public void RemoveItem_AllQuantity_RemovesFromCart()
    {
        _cart.AddItem(2, 3);
        _cart.RemoveItem(2, 3);
        
        Assert.Empty(_cart.Items);
    }
    
    [Fact] public void GetTotal_WithPromoCode_AppliesDiscount()
    {
        _cart.AddItem(1); // $999
        _mockDiscounts.Setup(d => d.CalculateDiscount(999m, "SAVE10")).Returns(99.9m);
        
        decimal total = _cart.GetTotal("SAVE10");
        
        Assert.Equal(899.1m, total);
        _mockDiscounts.Verify(d => d.CalculateDiscount(999m, "SAVE10"), Times.Once);
    }
    
    [Fact] public void Checkout_EmptyCart_ThrowsInvalidOperation()
    {
        Assert.Throws<InvalidOperationException>(() => _cart.Checkout());
    }
    
    [Fact] public void Checkout_ValidCart_ReturnsOrderAndClearsCart()
    {
        _cart.AddItem(1);
        _cart.AddItem(2, 2);
        _mockRepo.Setup(r => r.UpdateStock(It.IsAny<int>(), It.IsAny<int>())).Returns(true);
        
        var summary = _cart.Checkout();
        
        Assert.Equal(2, summary.Items.Count);
        Assert.Equal(_laptop.Price + _mouse.Price * 2, summary.Subtotal);
        Assert.True(_cart.ItemCount == 0); // cart cleared
        
        _mockRepo.Verify(r => r.UpdateStock(1, _laptop.StockQuantity - 1), Times.Once);
        _mockRepo.Verify(r => r.UpdateStock(2, _mouse.StockQuantity - 2), Times.Once);
    }
}
```

---

## สรุป Part 18

### xUnit Basics
- ✅ `[Fact]` — single test
- ✅ `[Theory]` + `[InlineData]`, `[MemberData]`, `[ClassData]` — parameterized
- ✅ Test naming: `Method_Scenario_Expected`

### Assertions
- ✅ `Assert.Equal`, `Assert.Null`, `Assert.True/False`
- ✅ `Assert.Contains`, `Assert.Throws`, `Assert.ThrowsAsync`
- ✅ `Assert.All`, `Assert.Empty`, `Assert.Single`

### Test Lifecycle
- ✅ `IClassFixture<T>` — shared setup per class
- ✅ `ICollectionFixture<T>` — shared across classes
- ✅ Constructor/Dispose — per-test setup/teardown
- ✅ `IAsyncLifetime` — async initialization

### Mocking with Moq
- ✅ `Mock<T>`, `mock.Object`, `mock.Setup`
- ✅ `ReturnsAsync`, `Throws`, `Returns`
- ✅ `It.IsAny<T>()`, `It.Is<T>(condition)`
- ✅ `mock.Verify` — assert calls were made
- ✅ `SetupSequence` — different results per call

### TDD
- ✅ Red → Green → Refactor cycle
- ✅ Write tests before implementation
- ✅ Tests act as living documentation

### Best Practices
- ✅ AAA pattern: Arrange, Act, Assert
- ✅ One logical assertion per test
- ✅ Test edge cases (null, empty, boundary)
- ✅ `[Trait]` for test categorization

---

## แบบฝึกหัด

1. ใช้ TDD สร้าง `NumberParser` ที่แปลง "one hundred twenty three" → 123
2. เขียน tests สำหรับ `Stack<T>` ที่สร้างใน Part 11 รวมถึง edge cases ทั้งหมด
3. Mock `IHttpClientFactory` และเขียน tests สำหรับ weather API client
4. สร้าง integration test สำหรับ file-based repository ที่เขียนและอ่านข้อมูลจาก JSON files
5. เขียน property-based tests โดยใช้ FsCheck library
