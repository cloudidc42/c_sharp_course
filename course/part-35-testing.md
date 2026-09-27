# Part 35: Testing Strategies - Unit, Integration, E2E

## Steps 981-1010: Comprehensive Testing in .NET

---

## Step 981: Testing Pyramid

```
         /\
        /  \
       / E2E\         ← Few tests, high confidence, slow
      /------\
     / Integr \       ← Medium tests, test boundaries
    /----------\
   /    Unit    \     ← Many tests, fast, isolated
  /--------------\
```

### Tools
- **Unit**: xUnit, NUnit, MSTest + Moq/NSubstitute + FluentAssertions
- **Integration**: WebApplicationFactory, Testcontainers, SpecFlow
- **E2E**: Playwright, Selenium, Cypress

---

## Step 982: xUnit Basics

```bash
dotnet add package xunit
dotnet add package xunit.runner.visualstudio
dotnet add package Microsoft.NET.Test.Sdk
dotnet add package FluentAssertions
dotnet add package Moq
dotnet add package NSubstitute
dotnet add package AutoFixture
dotnet add package AutoFixture.AutoMoq
```

```csharp
// Basic test structure
using FluentAssertions;
using Xunit;

namespace MyApp.Tests.Unit;

public class ProductServiceTests
{
    // Naming: MethodName_StateUnderTest_ExpectedBehavior
    
    [Fact]
    public void CalculateDiscount_WhenAmountOver1000_AppliesTenPercent()
    {
        // Arrange
        var service = new DiscountService();
        var orderAmount = 1500m;
        
        // Act
        var discount = service.CalculateDiscount(orderAmount);
        
        // Assert
        discount.Should().Be(150m);
    }
    
    [Theory]
    [InlineData(100, 0)]
    [InlineData(999, 0)]
    [InlineData(1000, 100)]
    [InlineData(5000, 750)]
    public void CalculateDiscount_VariousAmounts_ReturnsExpectedDiscount(
        decimal amount, decimal expectedDiscount)
    {
        var service = new DiscountService();
        
        var discount = service.CalculateDiscount(amount);
        
        discount.Should().Be(expectedDiscount);
    }
    
    [Fact]
    public void CalculateDiscount_WithNegativeAmount_ThrowsArgumentException()
    {
        var service = new DiscountService();
        
        var act = () => service.CalculateDiscount(-100);
        
        act.Should().Throw<ArgumentException>()
            .WithMessage("*must be non-negative*");
    }
}

public class DiscountService
{
    public decimal CalculateDiscount(decimal amount)
    {
        if (amount < 0) throw new ArgumentException("Amount must be non-negative");
        return amount >= 1000 ? amount * 0.10m : 0;
    }
}
```

---

## Step 983: Mocking with Moq

```csharp
using Moq;

public class OrderServiceTests
{
    private readonly Mock<IOrderRepository> _repoMock = new();
    private readonly Mock<IEmailService> _emailMock = new();
    private readonly Mock<ILogger<OrderService>> _loggerMock = new();
    
    private OrderService CreateSut()
        => new(_repoMock.Object, _emailMock.Object, _loggerMock.Object);
    
    [Fact]
    public async Task CreateOrder_ValidData_SavesAndSendsEmail()
    {
        // Arrange
        var request = new CreateOrderRequest("user1", [new(1, 2)]);
        var expectedOrder = new Order { Id = Guid.NewGuid(), UserId = "user1" };
        
        _repoMock
            .Setup(r => r.AddAsync(It.IsAny<Order>(), It.IsAny<CancellationToken>()))
            .ReturnsAsync(expectedOrder);
        
        _emailMock
            .Setup(e => e.SendOrderConfirmationAsync(
                It.IsAny<string>(), 
                It.IsAny<Guid>(),
                It.IsAny<CancellationToken>()))
            .Returns(Task.CompletedTask);
        
        var sut = CreateSut();
        
        // Act
        var result = await sut.CreateOrderAsync(request, CancellationToken.None);
        
        // Assert
        result.Should().NotBeNull();
        result.Id.Should().Be(expectedOrder.Id);
        
        _repoMock.Verify(
            r => r.AddAsync(It.IsAny<Order>(), It.IsAny<CancellationToken>()),
            Times.Once);
        
        _emailMock.Verify(
            e => e.SendOrderConfirmationAsync("user1", expectedOrder.Id, It.IsAny<CancellationToken>()),
            Times.Once);
    }
    
    [Fact]
    public async Task CreateOrder_RepositoryFails_ThrowsAndNoEmail()
    {
        // Arrange
        _repoMock
            .Setup(r => r.AddAsync(It.IsAny<Order>(), It.IsAny<CancellationToken>()))
            .ThrowsAsync(new InvalidOperationException("DB error"));
        
        var sut = CreateSut();
        
        // Act
        var act = async () => await sut.CreateOrderAsync(
            new CreateOrderRequest("user1", []), 
            CancellationToken.None);
        
        // Assert
        await act.Should().ThrowAsync<InvalidOperationException>();
        
        _emailMock.Verify(
            e => e.SendOrderConfirmationAsync(
                It.IsAny<string>(), 
                It.IsAny<Guid>(), 
                It.IsAny<CancellationToken>()),
            Times.Never);
    }
    
    // Moq Callback
    [Fact]
    public async Task CreateOrder_CapturesOrderPassedToRepo()
    {
        Order? capturedOrder = null;
        
        _repoMock
            .Setup(r => r.AddAsync(It.IsAny<Order>(), It.IsAny<CancellationToken>()))
            .Callback<Order, CancellationToken>((order, _) => capturedOrder = order)
            .ReturnsAsync((Order o, CancellationToken _) => o);
        
        var sut = CreateSut();
        await sut.CreateOrderAsync(
            new CreateOrderRequest("user42", [new(5, 3)]), 
            CancellationToken.None);
        
        capturedOrder.Should().NotBeNull();
        capturedOrder!.UserId.Should().Be("user42");
        capturedOrder.Items.Should().HaveCount(1);
    }
}
```

---

## Step 984: NSubstitute (Alternative to Moq)

```csharp
using NSubstitute;

public class ProductServiceTests
{
    private readonly IProductRepository _repo = Substitute.For<IProductRepository>();
    private readonly ILogger<ProductService> _logger = Substitute.For<ILogger<ProductService>>();
    
    [Fact]
    public async Task GetProduct_Exists_ReturnsProduct()
    {
        // Arrange
        var product = new Product { Id = 1, Name = "Test", Price = 100 };
        _repo.GetByIdAsync(1, Arg.Any<CancellationToken>()).Returns(product);
        
        var sut = new ProductService(_repo, _logger);
        
        // Act
        var result = await sut.GetProductAsync(1, CancellationToken.None);
        
        // Assert
        result.Should().BeEquivalentTo(new { Id = 1, Name = "Test", Price = 100m });
        
        // Verify calls
        await _repo.Received(1).GetByIdAsync(1, Arg.Any<CancellationToken>());
    }
    
    [Fact]
    public async Task GetProduct_NotFound_ReturnsNull()
    {
        _repo.GetByIdAsync(999, Arg.Any<CancellationToken>()).Returns((Product?)null);
        
        var sut = new ProductService(_repo, _logger);
        var result = await sut.GetProductAsync(999, CancellationToken.None);
        
        result.Should().BeNull();
    }
}
```

---

## Step 985: FluentAssertions Advanced

```csharp
using FluentAssertions;
using FluentAssertions.Execution;

public class FluentAssertionsExamples
{
    [Fact]
    public void Collections()
    {
        var numbers = new[] { 1, 2, 3, 4, 5 };
        
        numbers.Should().HaveCount(5);
        numbers.Should().Contain(3);
        numbers.Should().BeInAscendingOrder();
        numbers.Should().AllSatisfy(n => n.Should().BePositive());
        numbers.Should().ContainInOrder(1, 2, 3);
        numbers.Should().NotContain(10);
    }
    
    [Fact]
    public void Objects()
    {
        var product = new Product { Id = 1, Name = "Test", Price = 99.99m };
        
        product.Should().NotBeNull();
        product.Should().BeOfType<Product>();
        
        // Property equality
        product.Should().BeEquivalentTo(new { Id = 1, Name = "Test" },
            opts => opts.ExcludingMissingMembers());
        
        // Deep equality
        var expected = new Product { Id = 1, Name = "Test", Price = 99.99m };
        product.Should().BeEquivalentTo(expected,
            opts => opts
                .ComparingByMembers<Product>()
                .Using<decimal>(ctx => ctx.Subject.Should().BeApproximately(ctx.Expectation, 0.01m))
                .WhenTypeIs<decimal>());
    }
    
    [Fact]
    public void StringAssertions()
    {
        var message = "Hello, World!";
        
        message.Should().StartWith("Hello");
        message.Should().EndWith("!");
        message.Should().Contain("World");
        message.Should().MatchRegex(@"Hello, \w+!");
        message.Should().HaveLength(13);
    }
    
    [Fact]
    public async Task ExceptionAssertions()
    {
        // Synchronous
        var act = () => throw new ArgumentException("bad value");
        act.Should().Throw<ArgumentException>()
            .WithMessage("bad value")
            .Which.ParamName.Should().BeNull();
        
        // Async
        var asyncAct = async () => 
        {
            await Task.Delay(1);
            throw new InvalidOperationException("async fail");
        };
        
        await asyncAct.Should().ThrowAsync<InvalidOperationException>()
            .WithMessage("async fail");
    }
    
    [Fact]
    public void MultipleAssertions()
    {
        var order = new Order { Id = 1, Status = "Pending", TotalAmount = 0 };
        
        // All assertions run even if some fail
        using (new AssertionScope())
        {
            order.Id.Should().Be(1);
            order.Status.Should().Be("Pending");
            order.TotalAmount.Should().BePositive(); // This fails
        }
        // Reports ALL failures, not just the first
    }
}
```

---

## Step 986: AutoFixture

```csharp
using AutoFixture;
using AutoFixture.AutoMoq;
using AutoFixture.Xunit2;

public class AutoFixtureTests
{
    [Theory]
    [AutoData]
    public void CreateOrder_AutoGenerated_Works(
        string userId,
        List<OrderItem> items)
    {
        var request = new CreateOrderRequest(userId, items);
        
        request.UserId.Should().NotBeEmpty();
        request.Items.Should().NotBeEmpty();
    }
    
    [Theory]
    [AutoMoqData]  // Custom attribute
    public async Task ProcessOrder_UsesMoq_Works(
        [Frozen] Mock<IOrderRepository> repoMock,
        OrderService sut,
        Order expectedOrder)
    {
        repoMock
            .Setup(r => r.AddAsync(It.IsAny<Order>(), It.IsAny<CancellationToken>()))
            .ReturnsAsync(expectedOrder);
        
        var result = await sut.CreateOrderAsync(
            new CreateOrderRequest("user1", []), CancellationToken.None);
        
        result.Should().NotBeNull();
    }
    
    // Custom AutoData attribute
    public class AutoMoqDataAttribute : AutoDataAttribute
    {
        public AutoMoqDataAttribute() 
            : base(() => new Fixture().Customize(new AutoMoqCustomization())) { }
    }
}
```

---

## Step 987: Integration Tests with WebApplicationFactory

```csharp
using Microsoft.AspNetCore.Mvc.Testing;
using System.Net;
using System.Net.Http.Json;

public class ProductApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;
    
    public ProductApiTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                // Replace DbContext with in-memory
                services.RemoveAll<DbContextOptions<AppDbContext>>();
                services.AddDbContext<AppDbContext>(opts =>
                    opts.UseInMemoryDatabase("TestDb"));
                
                // Replace external services
                services.RemoveAll<IEmailService>();
                services.AddSingleton<IEmailService, FakeEmailService>();
            });
        }).CreateClient();
    }
    
    [Fact]
    public async Task GetProducts_ReturnsOk()
    {
        var response = await _client.GetAsync("/api/products");
        
        response.StatusCode.Should().Be(HttpStatusCode.OK);
        
        var products = await response.Content.ReadFromJsonAsync<List<ProductDto>>();
        products.Should().NotBeNull();
    }
    
    [Fact]
    public async Task CreateProduct_ValidData_Returns201()
    {
        var request = new CreateProductRequest("Test Product", "Desc", 100m, 10, "Electronics");
        
        var response = await _client.PostAsJsonAsync("/api/products", request);
        
        response.StatusCode.Should().Be(HttpStatusCode.Created);
        
        var created = await response.Content.ReadFromJsonAsync<ProductDto>();
        created!.Name.Should().Be("Test Product");
        response.Headers.Location.Should().NotBeNull();
    }
    
    [Fact]
    public async Task CreateProduct_InvalidPrice_Returns400()
    {
        var request = new CreateProductRequest("Test", "Desc", -100m, 10, "Electronics");
        
        var response = await _client.PostAsJsonAsync("/api/products", request);
        
        response.StatusCode.Should().Be(HttpStatusCode.BadRequest);
        
        var errors = await response.Content.ReadFromJsonAsync<ValidationProblemDetails>();
        errors!.Errors.Should().ContainKey("Price");
    }
    
    [Fact]
    public async Task GetProduct_NotFound_Returns404()
    {
        var response = await _client.GetAsync("/api/products/99999");
        
        response.StatusCode.Should().Be(HttpStatusCode.NotFound);
    }
}
```

---

## Step 988: Testcontainers Integration Tests

```csharp
using Testcontainers.PostgreSql;
using Testcontainers.Redis;

public class OrderServiceIntegrationTests : IAsyncLifetime
{
    private readonly PostgreSqlContainer _postgres = new PostgreSqlBuilder()
        .WithImage("postgres:16-alpine")
        .WithDatabase("testdb")
        .WithUsername("test")
        .WithPassword("test")
        .Build();
    
    private readonly RedisContainer _redis = new RedisBuilder()
        .WithImage("redis:7-alpine")
        .Build();
    
    private WebApplicationFactory<Program> _factory = null!;
    private HttpClient _client = null!;
    
    public async Task InitializeAsync()
    {
        await Task.WhenAll(_postgres.StartAsync(), _redis.StartAsync());
        
        _factory = new WebApplicationFactory<Program>()
            .WithWebHostBuilder(builder =>
            {
                builder.ConfigureServices(services =>
                {
                    services.RemoveAll<DbContextOptions<AppDbContext>>();
                    services.AddDbContext<AppDbContext>(opts =>
                        opts.UseNpgsql(_postgres.GetConnectionString()));
                    
                    services.RemoveAll<IDistributedCache>();
                    services.AddStackExchangeRedisCache(opts =>
                        opts.Configuration = _redis.GetConnectionString());
                });
            });
        
        _client = _factory.CreateClient();
        
        // Apply migrations
        using var scope = _factory.Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        await db.Database.MigrateAsync();
    }
    
    [Fact]
    public async Task FullOrderFlow_Works()
    {
        // Create product
        var product = await CreateProductAsync("Laptop", 45000m);
        
        // Get access token
        var token = await LoginAsync("user@test.com", "Password123!");
        _client.DefaultRequestHeaders.Authorization = 
            new AuthenticationHeaderValue("Bearer", token);
        
        // Create order
        var orderRequest = new CreateOrderRequest(
            "user@test.com",
            [new(product.Id, 2)]);
        
        var orderResponse = await _client.PostAsJsonAsync("/api/orders", orderRequest);
        orderResponse.StatusCode.Should().Be(HttpStatusCode.Created);
        
        var order = await orderResponse.Content.ReadFromJsonAsync<OrderDto>();
        order!.TotalAmount.Should().Be(90000m);
        
        // Check stock reduced
        var updatedProduct = await _client.GetFromJsonAsync<ProductDto>(
            $"/api/products/{product.Id}");
        updatedProduct!.StockQuantity.Should().BeLessThan(product.StockQuantity);
    }
    
    private async Task<ProductDto> CreateProductAsync(string name, decimal price)
    {
        var response = await _client.PostAsJsonAsync("/api/products",
            new CreateProductRequest(name, "Desc", price, 100, "Electronics"));
        response.EnsureSuccessStatusCode();
        return (await response.Content.ReadFromJsonAsync<ProductDto>())!;
    }
    
    private async Task<string> LoginAsync(string email, string password)
    {
        var response = await _client.PostAsJsonAsync("/api/auth/login",
            new LoginRequest(email, password));
        var result = await response.Content.ReadFromJsonAsync<TokenResponse>();
        return result!.AccessToken;
    }
    
    public async Task DisposeAsync()
    {
        _factory.Dispose();
        await Task.WhenAll(_postgres.StopAsync(), _redis.StopAsync());
    }
}
```

---

## Step 989: Fake/Stub Objects

```csharp
// Fakes are simpler than mocks - real implementations for testing
public class FakeEmailService : IEmailService
{
    private readonly List<SentEmail> _sentEmails = [];
    
    public IReadOnlyList<SentEmail> SentEmails => _sentEmails;
    
    public Task SendOrderConfirmationAsync(
        string email, Guid orderId, CancellationToken ct)
    {
        _sentEmails.Add(new SentEmail(email, "Order Confirmation", orderId.ToString()));
        return Task.CompletedTask;
    }
    
    public Task SendWelcomeEmailAsync(string email, string name, CancellationToken ct)
    {
        _sentEmails.Add(new SentEmail(email, "Welcome", name));
        return Task.CompletedTask;
    }
}

public record SentEmail(string To, string Subject, string Content);

// In-memory repository
public class InMemoryProductRepository : IProductRepository
{
    private readonly Dictionary<int, Product> _products = [];
    private int _nextId = 1;
    
    public Task<Product> AddAsync(Product product, CancellationToken ct)
    {
        product.Id = _nextId++;
        _products[product.Id] = product;
        return Task.FromResult(product);
    }
    
    public Task<Product?> GetByIdAsync(int id, CancellationToken ct)
        => Task.FromResult(_products.GetValueOrDefault(id));
    
    public Task<IEnumerable<Product>> GetAllAsync(CancellationToken ct)
        => Task.FromResult(_products.Values.AsEnumerable());
    
    public Task UpdateAsync(Product product, CancellationToken ct)
    {
        _products[product.Id] = product;
        return Task.CompletedTask;
    }
    
    public Task DeleteAsync(int id, CancellationToken ct)
    {
        _products.Remove(id);
        return Task.CompletedTask;
    }
}
```

---

## Step 990: Test Fixtures และ Shared Context

```csharp
// Class fixture - shared across tests in class
public class DatabaseFixture : IDisposable
{
    public AppDbContext DbContext { get; private set; }
    
    public DatabaseFixture()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseInMemoryDatabase(Guid.NewGuid().ToString())
            .Options;
        
        DbContext = new AppDbContext(options);
        DbContext.Database.EnsureCreated();
        
        // Seed test data
        DbContext.Products.AddRange(
            new Product { Id = 1, Name = "Product 1", Price = 100, Category = "Electronics" },
            new Product { Id = 2, Name = "Product 2", Price = 200, Category = "Books" }
        );
        DbContext.SaveChanges();
    }
    
    public void Dispose() => DbContext.Dispose();
}

public class ProductQueryTests(DatabaseFixture fixture) 
    : IClassFixture<DatabaseFixture>
{
    [Fact]
    public async Task GetByCategory_ReturnsFilteredProducts()
    {
        var products = await fixture.DbContext.Products
            .Where(p => p.Category == "Electronics")
            .ToListAsync();
        
        products.Should().HaveCount(1);
        products.Single().Name.Should().Be("Product 1");
    }
}

// Collection fixture - shared across test classes
[CollectionDefinition("Database")]
public class DatabaseCollection : ICollectionFixture<DatabaseFixture> { }

[Collection("Database")]
public class ProductReadsTests(DatabaseFixture fixture)
{
    [Fact]
    public async Task GetAll_ReturnsAllProducts()
    {
        var count = await fixture.DbContext.Products.CountAsync();
        count.Should().Be(2);
    }
}
```

---

## Step 991: Playwright E2E Tests

```bash
dotnet add package Microsoft.Playwright
dotnet tool install --global Microsoft.Playwright.CLI
playwright install
```

```csharp
using Microsoft.Playwright;
using Microsoft.Playwright.NUnit;

[Parallelizable(ParallelScope.Self)]
[TestFixture]
public class ProductE2ETests : PageTest
{
    [Test]
    public async Task AddProduct_ToCart_ShowsInCart()
    {
        await Page.GotoAsync("http://localhost:5000");
        
        // Navigate to products
        await Page.ClickAsync("[data-testid='products-link']");
        await Page.WaitForURLAsync("**/products");
        
        // Add first product to cart
        await Page.ClickAsync("[data-testid='add-to-cart-btn']");
        
        // Check cart count updated
        var cartCount = await Page.TextContentAsync("[data-testid='cart-count']");
        cartCount.Should().Be("1");
        
        // Go to cart
        await Page.ClickAsync("[data-testid='cart-icon']");
        await Page.WaitForURLAsync("**/cart");
        
        // Verify product in cart
        var cartItems = await Page.QuerySelectorAllAsync("[data-testid='cart-item']");
        cartItems.Should().HaveCount(1);
    }
    
    [Test]
    public async Task Checkout_ValidPayment_OrderConfirmed()
    {
        // Login
        await Page.GotoAsync("http://localhost:5000/login");
        await Page.FillAsync("[name='email']", "test@example.com");
        await Page.FillAsync("[name='password']", "Password123!");
        await Page.ClickAsync("[type='submit']");
        await Page.WaitForURLAsync("**/dashboard");
        
        // Add product and checkout
        await Page.GotoAsync("http://localhost:5000/products");
        await Page.ClickAsync("[data-testid='add-to-cart-btn']");
        await Page.ClickAsync("[data-testid='checkout-btn']");
        
        // Fill payment
        await Page.FillAsync("[name='cardNumber']", "4242424242424242");
        await Page.FillAsync("[name='expiry']", "12/25");
        await Page.FillAsync("[name='cvc']", "123");
        
        await Page.ClickAsync("[type='submit']");
        
        // Wait for confirmation
        await Page.WaitForSelectorAsync("[data-testid='order-confirmation']");
        
        var confirmationText = await Page.TextContentAsync("[data-testid='order-confirmation']");
        confirmationText.Should().Contain("Order confirmed");
    }
    
    [Test]
    public async Task Search_FindsProducts()
    {
        await Page.GotoAsync("http://localhost:5000/products");
        
        await Page.FillAsync("[data-testid='search-input']", "laptop");
        await Page.PressAsync("[data-testid='search-input']", "Enter");
        
        await Page.WaitForResponseAsync(resp => 
            resp.Url.Contains("/api/products") && resp.Status == 200);
        
        var results = await Page.QuerySelectorAllAsync("[data-testid='product-card']");
        results.Should().NotBeEmpty();
        
        // All results should contain "laptop"
        foreach (var result in results)
        {
            var name = await result.TextContentAsync();
            name!.ToLower().Should().Contain("laptop");
        }
    }
}
```

---

## Step 992: Snapshot Testing

```csharp
// Verify library for snapshot testing
using Verify;
using VerifyXunit;

[UsesVerify]
public class ApiResponseTests
{
    [Fact]
    public async Task GetProduct_Response_MatchesSnapshot()
    {
        var factory = new WebApplicationFactory<Program>();
        var client = factory.CreateClient();
        
        var response = await client.GetAsync("/api/products/1");
        var json = await response.Content.ReadAsStringAsync();
        
        // First run: creates snapshot file
        // Subsequent runs: compares with snapshot
        await Verify(json)
            .ScrubMember("createdAt") // ignore dynamic fields
            .ScrubMember("updatedAt");
    }
    
    [Fact]
    public async Task ProductList_MatchesSnapshot()
    {
        var products = new List<ProductDto>
        {
            new(1, "Laptop", "Desc", 45000m, 10, "Electronics"),
            new(2, "Book", "Desc", 500m, 50, "Books")
        };
        
        await Verify(products);
    }
}
```

---

## Step 993: Performance Testing

```csharp
// NBomber for HTTP load testing in tests
[Fact]
public void ProductsEndpoint_Under100rps_P95Under200ms()
{
    using var httpClient = new HttpClient { BaseAddress = new Uri("http://localhost:5000") };
    
    var scenario = Scenario.Create("get_products", async context =>
    {
        var response = await httpClient.GetAsync("/api/products");
        return response.IsSuccessStatusCode
            ? Response.Ok(statusCode: (int)response.StatusCode)
            : Response.Fail(statusCode: (int)response.StatusCode);
    })
    .WithWarmUpDuration(TimeSpan.FromSeconds(5))
    .WithLoadSimulations(
        Simulation.Inject(rate: 100, interval: TimeSpan.FromSeconds(1), 
            during: TimeSpan.FromSeconds(30))
    );
    
    var stats = NBomberRunner
        .RegisterScenarios(scenario)
        .WithoutReports()
        .Run();
    
    var scenarioStats = stats.ScenarioStats.First();
    
    scenarioStats.Ok.Latency.Percent95.Should().BeLessThan(200);
    scenarioStats.Fail.Request.Count.Should().Be(0);
}
```

---

## Step 994: Test Data Builder Pattern

```csharp
// Builder for creating test data
public class ProductBuilder
{
    private int _id = 1;
    private string _name = "Test Product";
    private decimal _price = 100m;
    private int _stock = 10;
    private string _category = "Electronics";
    private bool _isActive = true;
    
    public ProductBuilder WithId(int id) { _id = id; return this; }
    public ProductBuilder WithName(string name) { _name = name; return this; }
    public ProductBuilder WithPrice(decimal price) { _price = price; return this; }
    public ProductBuilder WithStock(int stock) { _stock = stock; return this; }
    public ProductBuilder WithCategory(string cat) { _category = cat; return this; }
    public ProductBuilder AsInactive() { _isActive = false; return this; }
    
    public Product Build() => new()
    {
        Id = _id,
        Name = _name,
        Price = _price,
        StockQuantity = _stock,
        Category = _category,
        IsActive = _isActive
    };
    
    public static ProductBuilder Create() => new();
}

// Usage in tests
[Fact]
public async Task GetActiveProducts_ExcludesInactive()
{
    var activeProduct = ProductBuilder.Create()
        .WithId(1)
        .WithName("Active Product")
        .Build();
    
    var inactiveProduct = ProductBuilder.Create()
        .WithId(2)
        .WithName("Inactive Product")
        .AsInactive()
        .Build();
    
    await _repo.AddAsync(activeProduct, CancellationToken.None);
    await _repo.AddAsync(inactiveProduct, CancellationToken.None);
    
    var products = await _service.GetActiveProductsAsync(CancellationToken.None);
    
    products.Should().HaveCount(1);
    products.Single().Name.Should().Be("Active Product");
}
```

---

## Step 995: SpecFlow (BDD)

```bash
dotnet add package SpecFlow.xUnit
dotnet add package SpecFlow.Plus.LivingDocPlugin
```

```gherkin
# Features/Product.feature
Feature: Product Management
  As a store manager
  I want to manage products
  So that customers can purchase them

  Scenario: Create a new product successfully
    Given I am authenticated as an admin
    When I create a product with name "Laptop Pro" and price 45000
    Then the product should be created successfully
    And the product should have name "Laptop Pro"
    And the product price should be 45000
  
  Scenario Outline: Validate product price
    Given I am authenticated as an admin
    When I create a product with price <price>
    Then I should receive a <status> response
    
    Examples:
      | price  | status  |
      | 100    | 201     |
      | -1     | 400     |
      | 0      | 400     |
      | 999999 | 201     |
  
  Scenario: Out of stock prevention
    Given there is a product with stock 0
    When I try to order that product
    Then I should receive an insufficient stock error
```

```csharp
// Features/ProductSteps.cs
using TechTalk.SpecFlow;

[Binding]
public class ProductSteps
{
    private readonly HttpClient _client;
    private HttpResponseMessage? _response;
    private ProductDto? _createdProduct;
    
    public ProductSteps(WebApplicationFactory<Program> factory)
    {
        _client = factory.CreateClient();
    }
    
    [Given(@"I am authenticated as an admin")]
    public async Task GivenAuthenticatedAsAdmin()
    {
        var tokenResponse = await _client.PostAsJsonAsync("/api/auth/login",
            new { Email = "admin@test.com", Password = "Admin123!" });
        
        var token = (await tokenResponse.Content.ReadFromJsonAsync<TokenResponse>())!;
        _client.DefaultRequestHeaders.Authorization = 
            new AuthenticationHeaderValue("Bearer", token.AccessToken);
    }
    
    [When(@"I create a product with name ""(.*)"" and price (.*)")]
    public async Task WhenCreateProduct(string name, decimal price)
    {
        _response = await _client.PostAsJsonAsync("/api/products",
            new CreateProductRequest(name, "Description", price, 100, "Electronics"));
    }
    
    [When(@"I create a product with price (.*)")]
    public async Task WhenCreateProductWithPrice(decimal price)
    {
        _response = await _client.PostAsJsonAsync("/api/products",
            new CreateProductRequest("Test", "Desc", price, 10, "Electronics"));
    }
    
    [Then(@"the product should be created successfully")]
    public void ThenProductCreatedSuccessfully()
    {
        _response!.StatusCode.Should().Be(HttpStatusCode.Created);
    }
    
    [Then(@"the product should have name ""(.*)""")]
    public async Task ThenProductHasName(string expectedName)
    {
        _createdProduct ??= await _response!.Content.ReadFromJsonAsync<ProductDto>();
        _createdProduct!.Name.Should().Be(expectedName);
    }
    
    [Then(@"the product price should be (.*)")]
    public async Task ThenProductPriceIs(decimal expectedPrice)
    {
        _createdProduct ??= await _response!.Content.ReadFromJsonAsync<ProductDto>();
        _createdProduct!.Price.Should().Be(expectedPrice);
    }
    
    [Then(@"I should receive a (.*) response")]
    public void ThenStatusCode(int statusCode)
    {
        ((int)_response!.StatusCode).Should().Be(statusCode);
    }
}
```

---

## Step 996: Mutation Testing

```bash
# Stryker.NET for mutation testing
dotnet tool install -g dotnet-stryker
dotnet-stryker --project MyApp.Tests.csproj
```

```json
// stryker-config.json
{
  "stryker-config": {
    "mutation-level": "Standard",
    "thresholds": {
      "high": 90,
      "low": 70,
      "break": 0
    },
    "ignore-methods": ["ToString", "GetHashCode"],
    "reporters": ["html", "json", "progress"]
  }
}
```

---

## Step 997: Coverage Reports

```bash
# Run tests with coverage
dotnet test --collect:"XPlat Code Coverage"

# Generate report
dotnet tool install -g dotnet-reportgenerator-globaltool
reportgenerator \
  -reports:"**/coverage.cobertura.xml" \
  -targetdir:"coveragereport" \
  -reporttypes:"Html;JsonSummary"

# CI coverage check
dotnet test /p:CollectCoverage=true \
            /p:CoverletOutputFormat=opencover \
            /p:Threshold=80 \
            /p:ThresholdType=line
```

---

## Step 998: Contract Testing (Pact)

```bash
dotnet add package PactNet
```

```csharp
// Consumer test
public class ProductConsumerTests : IDisposable
{
    private readonly IPactBuilderV4 _pact;
    
    public ProductConsumerTests()
    {
        var config = new PactConfig
        {
            PactDir = "pacts",
            DefaultJsonSettings = { }
        };
        
        _pact = Pact.V4("OrderService", "ProductService", config).WithHttpInteractions();
    }
    
    [Fact]
    public async Task GetProduct_Returns200()
    {
        _pact.UponReceiving("Get product by ID")
            .Given("Product 1 exists")
            .WithRequest(HttpMethod.Get, "/api/products/1")
            .WillRespond()
            .WithStatus(200)
            .WithJsonBody(new
            {
                id = 1,
                name = "Test Product",
                price = 100.0
            });
        
        await _pact.VerifyAsync(async ctx =>
        {
            var client = new ProductApiClient(ctx.MockServerUri.ToString());
            var product = await client.GetProductAsync(1, CancellationToken.None);
            
            product.Should().NotBeNull();
            product!.Id.Should().Be(1);
        });
    }
    
    public void Dispose() => _pact.Dispose();
}

// Provider test (ProductService verifies consumer expectations)
public class ProductProviderTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;
    
    public ProductProviderTests(WebApplicationFactory<Program> factory)
        => _factory = factory;
    
    [Fact]
    public async Task VerifyPactWithConsumer()
    {
        var config = new PactVerifierConfig
        {
            ProviderName = "ProductService",
            ProviderVersion = "1.0.0"
        };
        
        await new PactVerifier(config)
            .WithHttpEndpoint(_factory.Server.BaseAddress)
            .WithFileSource(new FileInfo("pacts/OrderService-ProductService.json"))
            .WithStateHandler(async state =>
            {
                if (state.ProviderState == "Product 1 exists")
                {
                    using var scope = _factory.Services.CreateScope();
                    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
                    if (!await db.Products.AnyAsync(p => p.Id == 1))
                    {
                        db.Products.Add(new Product { Id = 1, Name = "Test Product", Price = 100 });
                        await db.SaveChangesAsync();
                    }
                }
                return Task.CompletedTask;
            })
            .VerifyAsync();
    }
}
```

---

## Step 999: Test Organization Best Practices

```
MyApp.Tests/
├── Unit/
│   ├── Services/
│   │   ├── ProductServiceTests.cs
│   │   └── OrderServiceTests.cs
│   ├── Domain/
│   │   └── OrderTests.cs
│   └── Helpers/
│       └── Builders/
│           ├── ProductBuilder.cs
│           └── OrderBuilder.cs
├── Integration/
│   ├── Api/
│   │   ├── ProductApiTests.cs
│   │   └── OrderApiTests.cs
│   ├── Repositories/
│   │   └── ProductRepositoryTests.cs
│   └── Fixtures/
│       ├── DatabaseFixture.cs
│       └── TestWebApplicationFactory.cs
├── E2E/
│   └── Playwright/
│       └── ProductE2ETests.cs
└── Shared/
    ├── Fakes/
    │   ├── FakeEmailService.cs
    │   └── InMemoryProductRepository.cs
    └── Attributes/
        └── AutoMoqDataAttribute.cs
```

---

## Step 1000: Grand Milestone - Testing Checklist

### ✅ สิ่งที่ต้องมีใน Production Codebase

#### Unit Tests
- [ ] Business logic coverage > 90%
- [ ] Edge cases tested
- [ ] Happy path + error paths
- [ ] Fast execution (< 1s per test)

#### Integration Tests
- [ ] API endpoints tested
- [ ] Database operations tested with Testcontainers
- [ ] Auth flows tested
- [ ] External service integration (with mocks)

#### E2E Tests
- [ ] Critical user journeys
- [ ] Login flow
- [ ] Checkout flow (if applicable)

#### Quality
- [ ] No flaky tests
- [ ] Tests run in parallel safely
- [ ] Coverage > 80%
- [ ] CI pipeline runs on every PR

---

### 🎉 Step 1000 - Course Progress Summary

เราได้ครอบคลุม C# .NET อย่างครบถ้วนตั้งแต่:
- **Part 1-10**: พื้นฐาน C# (Variables, OOP, Collections, LINQ)
- **Part 11-20**: Intermediate (.NET Core, EF Core, APIs)
- **Part 21-28**: Advanced (Identity, gRPC, SignalR, Blazor)
- **Part 29-35**: Professional (GraphQL, Microservices, Performance, Security, Testing)

*ต่อไป Part 36: Advanced Patterns - Domain-Driven Design (DDD)*

---

*จบ Part 35: Testing Strategies*
*ยังคงดำเนินต่อไปสู่ระดับ World-Class...*
