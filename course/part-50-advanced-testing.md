# Part 50: Advanced Testing Strategies
## Steps 1401-1440: TDD, BDD, Mutation Testing, Architecture Tests, Property-Based Testing

---

## Step 1401: Test-Driven Development (TDD) with Red-Green-Refactor

```csharp
// TDD cycle: Write failing test → Make it pass → Refactor

// 1. RED: Write failing test first
public class OrderServiceTests
{
    [Fact]
    public void CreateOrder_WithValidItems_ShouldReturnOrderWithCorrectTotal()
    {
        // Arrange
        var service = new OrderService();
        var items = new List<OrderItem>
        {
            new("Product A", 2, 10.00m),
            new("Product B", 1, 25.50m)
        };

        // Act
        var order = service.CreateOrder(Guid.NewGuid(), items);

        // Assert
        order.Total.Should().Be(45.50m);
        order.Status.Should().Be(OrderStatus.Pending);
        order.Items.Should().HaveCount(2);
    }

    [Fact]
    public void CreateOrder_WithEmptyItems_ShouldThrowDomainException()
    {
        var service = new OrderService();

        var act = () => service.CreateOrder(Guid.NewGuid(), []);

        act.Should().Throw<DomainException>()
           .WithMessage("Order must have at least one item");
    }

    [Theory]
    [InlineData(0)]
    [InlineData(-1)]
    [InlineData(-100)]
    public void CreateOrder_WithInvalidQuantity_ShouldThrowDomainException(int quantity)
    {
        var service = new OrderService();
        var items = new List<OrderItem> { new("Product", quantity, 10.00m) };

        var act = () => service.CreateOrder(Guid.NewGuid(), items);

        act.Should().Throw<DomainException>()
           .WithMessage("*quantity*");
    }
}

// 2. GREEN: Minimal implementation to pass tests
public record OrderItem(string Name, int Quantity, decimal UnitPrice);

public class Order
{
    public Guid Id { get; init; }
    public Guid CustomerId { get; init; }
    public IReadOnlyList<OrderItem> Items { get; init; } = [];
    public decimal Total => Items.Sum(i => i.Quantity * i.UnitPrice);
    public OrderStatus Status { get; set; } = OrderStatus.Pending;
    public DateTimeOffset CreatedAt { get; init; }
}

public enum OrderStatus { Pending, Confirmed, Shipped, Delivered, Cancelled }

public class DomainException(string message) : Exception(message);

public class OrderService
{
    public Order CreateOrder(Guid customerId, IList<OrderItem> items)
    {
        if (items is null || items.Count == 0)
            throw new DomainException("Order must have at least one item");

        foreach (var item in items)
        {
            if (item.Quantity <= 0)
                throw new DomainException($"Invalid quantity {item.Quantity} for item '{item.Name}'");
        }

        return new Order
        {
            Id = Guid.NewGuid(),
            CustomerId = customerId,
            Items = items.ToList(),
            CreatedAt = DateTimeOffset.UtcNow
        };
    }
}

// 3. REFACTOR: Extract validation, improve design
public class OrderValidator
{
    public static void Validate(IList<OrderItem> items)
    {
        if (items is null || items.Count == 0)
            throw new DomainException("Order must have at least one item");

        foreach (var item in items.Where(i => i.Quantity <= 0))
            throw new DomainException($"Invalid quantity {item.Quantity} for item '{item.Name}'");
    }
}
```

---

## Step 1402: Behavior-Driven Development (BDD) with Reqnroll/SpecFlow

```gherkin
# Features/OrderManagement.feature
Feature: Order Management
  As a customer
  I want to place orders
  So that I can purchase products

  Background:
    Given the catalog contains the following products:
      | Name      | Price | Stock |
      | Widget A  | 10.00 | 100   |
      | Widget B  | 25.50 | 50    |
      | Widget C  | 5.00  | 0     |

  Scenario: Successfully place an order
    Given I am a registered customer with id "cust-001"
    When I add 2 units of "Widget A" to my cart
    And I add 1 unit of "Widget B" to my cart
    And I place the order
    Then the order should be created successfully
    And the order total should be 45.50
    And the order status should be "Pending"

  Scenario: Cannot order out-of-stock product
    Given I am a registered customer with id "cust-002"
    When I add 1 unit of "Widget C" to my cart
    And I place the order
    Then the order should fail with error "Widget C is out of stock"

  Scenario Outline: Quantity validation
    Given I am a registered customer with id "cust-003"
    When I try to add <quantity> units of "Widget A" to my cart
    Then I should see a validation error "Quantity must be greater than zero"

    Examples:
      | quantity |
      | 0        |
      | -1       |
      | -100     |
```

```csharp
// Features/OrderManagementStepDefinitions.cs
[Binding]
public class OrderManagementStepDefinitions(ScenarioContext scenarioContext)
{
    private readonly Dictionary<string, Product> _catalog = new();
    private readonly List<CartItem> _cart = [];
    private Guid _customerId;
    private Order? _placedOrder;
    private string? _errorMessage;

    [Given(@"the catalog contains the following products:")]
    public void GivenTheCatalogContainsProducts(DataTable dataTable)
    {
        foreach (var row in dataTable.Rows)
        {
            var product = new Product(
                row["Name"],
                decimal.Parse(row["Price"]),
                int.Parse(row["Stock"])
            );
            _catalog[product.Name] = product;
        }
    }

    [Given(@"I am a registered customer with id ""(.*)""")]
    public void GivenIAmARegisteredCustomer(string customerId)
    {
        _customerId = Guid.Parse(customerId.Replace("cust-", "00000000-0000-0000-0000-0000000000"));
    }

    [When(@"I add (\d+) units? of ""(.*)"" to my cart")]
    public void WhenIAddUnitsToMyCart(int quantity, string productName)
    {
        if (!_catalog.TryGetValue(productName, out var product))
            throw new InvalidOperationException($"Product '{productName}' not found");

        _cart.Add(new CartItem(product, quantity));
    }

    [When(@"I place the order")]
    public void WhenIPlaceTheOrder()
    {
        try
        {
            var orderService = new OrderService(new InventoryChecker(_catalog));
            var items = _cart.Select(c => new OrderItem(c.Product.Name, c.Quantity, c.Product.Price)).ToList();
            _placedOrder = orderService.CreateOrder(_customerId, items);
        }
        catch (DomainException ex)
        {
            _errorMessage = ex.Message;
        }
    }

    [Then(@"the order should be created successfully")]
    public void ThenTheOrderShouldBeCreatedSuccessfully()
    {
        _errorMessage.Should().BeNull("no error should have occurred");
        _placedOrder.Should().NotBeNull();
    }

    [Then(@"the order total should be (.*)")]
    public void ThenTheOrderTotalShouldBe(decimal expectedTotal)
    {
        _placedOrder!.Total.Should().Be(expectedTotal);
    }

    [Then(@"the order status should be ""(.*)""")]
    public void ThenTheOrderStatusShouldBe(string expectedStatus)
    {
        _placedOrder!.Status.ToString().Should().Be(expectedStatus);
    }

    [Then(@"the order should fail with error ""(.*)""")]
    public void ThenTheOrderShouldFailWithError(string expectedError)
    {
        _errorMessage.Should().Be(expectedError);
        _placedOrder.Should().BeNull();
    }

    [When(@"I try to add (.*) units? of ""(.*)"" to my cart")]
    public void WhenITryToAddUnitsToMyCart(int quantity, string productName)
    {
        try
        {
            if (!_catalog.TryGetValue(productName, out var product))
                throw new InvalidOperationException($"Product '{productName}' not found");

            if (quantity <= 0)
                throw new DomainException("Quantity must be greater than zero");

            _cart.Add(new CartItem(product, quantity));
        }
        catch (DomainException ex)
        {
            _errorMessage = ex.Message;
        }
    }

    [Then(@"I should see a validation error ""(.*)""")]
    public void ThenIShouldSeeAValidationError(string expectedError)
    {
        _errorMessage.Should().Be(expectedError);
    }
}
```

---

## Step 1403: Property-Based Testing with FsCheck

```csharp
// Install: FsCheck, FsCheck.Xunit

public class OrderPropertyTests
{
    [Property]
    public Property OrderTotal_AlwaysEqualsSumOfItemTotals()
    {
        return Prop.ForAll(
            GenerateNonEmptyOrderItems(),
            items =>
            {
                var service = new OrderService();
                var order = service.CreateOrder(Guid.NewGuid(), items);
                var expectedTotal = items.Sum(i => i.Quantity * i.UnitPrice);
                return order.Total == expectedTotal;
            }
        );
    }

    [Property]
    public Property CreateOrder_WithPositiveQuantities_NeverThrows()
    {
        var positiveItems = Arb.From(
            from name in Arb.Default.NonEmptyString().Generator
            from qty in Gen.Choose(1, 1000)
            from price in Gen.Choose(1, 100000).Select(p => p / 100m)
            select new OrderItem(name.Get, qty, price)
        );

        return Prop.ForAll(
            Gen.NonEmptyListOf(positiveItems.Generator).ToArbitrary(),
            items =>
            {
                var service = new OrderService();
                try
                {
                    service.CreateOrder(Guid.NewGuid(), items);
                    return true;
                }
                catch
                {
                    return false;
                }
            }
        );
    }

    [Property]
    public Property ApplyDiscount_NeverIncreasesTotal()
    {
        return Prop.ForAll(
            GenerateNonEmptyOrderItems(),
            Arb.From(Gen.Choose(0, 100).Select(p => p / 100m)),
            (items, discountRate) =>
            {
                var service = new OrderService();
                var order = service.CreateOrder(Guid.NewGuid(), items);
                var originalTotal = order.Total;
                order.ApplyDiscount(discountRate);
                return order.Total <= originalTotal;
            }
        );
    }

    [Property]
    public Property OrderItems_AreIdempotentWhenSerialized()
    {
        return Prop.ForAll(
            GenerateNonEmptyOrderItems(),
            items =>
            {
                var json = JsonSerializer.Serialize(items);
                var deserialized = JsonSerializer.Deserialize<List<OrderItem>>(json)!;
                return items.SequenceEqual(deserialized);
            }
        );
    }

    // Custom generator for non-empty order items
    private static Arbitrary<List<OrderItem>> GenerateNonEmptyOrderItems()
    {
        var itemGen =
            from name in Arb.Default.NonEmptyString().Generator
            from qty in Gen.Choose(1, 100)
            from priceCents in Gen.Choose(1, 99999)
            select new OrderItem(name.Get, qty, priceCents / 100m);

        return Gen.NonEmptyListOf(itemGen).ToArbitrary();
    }
}

// FsCheck with custom generators for domain objects
public class CustomerGenerator
{
    public static Arbitrary<Customer> Customers()
    {
        var gen =
            from firstName in Arb.Default.NonEmptyString().Generator
            from lastName in Arb.Default.NonEmptyString().Generator
            from email in Gen.Elements("test@example.com", "user@test.org", "demo@sample.net")
            select new Customer(Guid.NewGuid(), firstName.Get, lastName.Get, email);

        return gen.ToArbitrary();
    }
}
```

---

## Step 1404: Mutation Testing with Stryker.NET

```xml
<!-- stryker-config.json -->
{
  "stryker-config": {
    "project": "OrderService.csproj",
    "test-projects": ["../OrderService.Tests/OrderService.Tests.csproj"],
    "reporters": ["html", "json", "progress"],
    "mutation-level": "Advanced",
    "threshold": {
      "high": 90,
      "low": 70,
      "break": 60
    },
    "ignore-methods": [
      "ToString",
      "GetHashCode",
      "Equals"
    ],
    "ignore-mutations": [
      "StringLiteral"
    ],
    "coverage-analysis": "perTest",
    "since": {
      "enabled": true,
      "target": "main"
    }
  }
}
```

```csharp
// Example: Code that mutation testing reveals as under-tested

public class PricingEngine
{
    // Mutation: Change > to >= reveals test gap
    public decimal CalculateDiscount(decimal total, CustomerTier tier)
    {
        if (total > 1000m)  // Mutation: > becomes >=, < , ==
        {
            return tier switch
            {
                CustomerTier.Gold => total * 0.15m,      // Mutation: 0.15 → 0.14, 0.16
                CustomerTier.Silver => total * 0.10m,    // Mutation: 0.10 → 0.09, 0.11
                CustomerTier.Bronze => total * 0.05m,    // Mutation: 0.05 → 0.04, 0.06
                _ => 0m
            };
        }
        return 0m;
    }

    // Strong tests that kill mutations
    [Fact]
    public void CalculateDiscount_ExactlyAt1000_ReturnsZero()
    {
        var engine = new PricingEngine();
        engine.CalculateDiscount(1000m, CustomerTier.Gold).Should().Be(0m);
    }

    [Fact]
    public void CalculateDiscount_OnePennyOver1000_ReturnsDiscount()
    {
        var engine = new PricingEngine();
        engine.CalculateDiscount(1000.01m, CustomerTier.Gold).Should().BeGreaterThan(0m);
    }

    [Theory]
    [InlineData(CustomerTier.Gold, 0.15)]
    [InlineData(CustomerTier.Silver, 0.10)]
    [InlineData(CustomerTier.Bronze, 0.05)]
    public void CalculateDiscount_ForEachTier_ReturnsCorrectRate(CustomerTier tier, double expectedRate)
    {
        var engine = new PricingEngine();
        var total = 2000m;
        var discount = engine.CalculateDiscount(total, tier);
        discount.Should().BeApproximately(total * (decimal)expectedRate, 0.001m);
    }
}

// Run Stryker from CLI:
// dotnet stryker --config-file stryker-config.json
// Results: HTML report at StrykerOutput/reports/mutation-report.html
// Mutation score = killed mutations / total mutations * 100
```

---

## Step 1405: Architecture Tests with NetArchTest

```csharp
// Install: NetArchTest.Rules

public class ArchitectureTests
{
    private static readonly Assembly DomainAssembly = typeof(Order).Assembly;
    private static readonly Assembly ApplicationAssembly = typeof(CreateOrderCommand).Assembly;
    private static readonly Assembly InfrastructureAssembly = typeof(OrderRepository).Assembly;
    private static readonly Assembly ApiAssembly = typeof(OrdersController).Assembly;

    // Domain layer should not depend on infrastructure
    [Fact]
    public void DomainLayer_ShouldNotDependOnInfrastructure()
    {
        var result = Types.InAssembly(DomainAssembly)
            .ShouldNot()
            .HaveDependencyOn("Microsoft.EntityFrameworkCore")
            .GetResult();

        result.IsSuccessful.Should().BeTrue(
            because: "Domain layer must be infrastructure-independent");
    }

    [Fact]
    public void DomainLayer_ShouldNotDependOnApplication()
    {
        var result = Types.InAssembly(DomainAssembly)
            .ShouldNot()
            .HaveDependencyOnAny("Application", "Infrastructure", "Api")
            .GetResult();

        result.IsSuccessful.Should().BeTrue();
    }

    // Application layer can depend on domain but not infrastructure
    [Fact]
    public void ApplicationLayer_ShouldNotDependOnInfrastructure()
    {
        var result = Types.InAssembly(ApplicationAssembly)
            .ShouldNot()
            .HaveDependencyOnAny(
                "Microsoft.EntityFrameworkCore",
                "StackExchange.Redis",
                "RabbitMQ")
            .GetResult();

        result.IsSuccessful.Should().BeTrue(
            because: "Application layer should depend on abstractions, not infrastructure");
    }

    // Controllers should not contain business logic
    [Fact]
    public void Controllers_ShouldBeInApiNamespace()
    {
        var result = Types.InAssembly(ApiAssembly)
            .That()
            .HaveNameEndingWith("Controller")
            .Should()
            .ResideInNamespace("MyApp.Api.Controllers")
            .GetResult();

        result.IsSuccessful.Should().BeTrue();
    }

    [Fact]
    public void Controllers_ShouldNotDirectlyAccessRepository()
    {
        var result = Types.InAssembly(ApiAssembly)
            .That()
            .HaveNameEndingWith("Controller")
            .ShouldNot()
            .HaveDependencyOnAny("IOrderRepository", "OrderRepository")
            .GetResult();

        result.IsSuccessful.Should().BeTrue(
            because: "Controllers should go through application services/mediator");
    }

    // Domain entities should be in domain namespace
    [Fact]
    public void Entities_ShouldResideInDomainNamespace()
    {
        var result = Types.InAssembly(DomainAssembly)
            .That()
            .Inherit(typeof(Entity))
            .Should()
            .ResideInNamespace("MyApp.Domain.Entities")
            .GetResult();

        result.IsSuccessful.Should().BeTrue();
    }

    // Interfaces should start with I
    [Fact]
    public void Interfaces_ShouldHaveIPrefix()
    {
        var result = Types.InAssemblies(
                [DomainAssembly, ApplicationAssembly, InfrastructureAssembly])
            .That()
            .AreInterfaces()
            .Should()
            .HaveNameStartingWith("I")
            .GetResult();

        result.IsSuccessful.Should().BeTrue();
    }

    // Command handlers should be sealed
    [Fact]
    public void CommandHandlers_ShouldBeSealed()
    {
        var result = Types.InAssembly(ApplicationAssembly)
            .That()
            .HaveNameEndingWith("Handler")
            .Should()
            .BeSealed()
            .GetResult();

        result.IsSuccessful.Should().BeTrue(
            because: "Handlers are concrete implementations and should not be inherited");
    }

    // Repository implementations should be internal
    [Fact]
    public void RepositoryImplementations_ShouldNotBePublic()
    {
        var result = Types.InAssembly(InfrastructureAssembly)
            .That()
            .HaveNameEndingWith("Repository")
            .And()
            .AreNotInterfaces()
            .Should()
            .NotBePublic()
            .GetResult();

        result.IsSuccessful.Should().BeTrue(
            because: "Repository implementations should be internal; use interfaces");
    }
}
```

---

## Step 1406: Integration Tests with Testcontainers

```csharp
// Install: Testcontainers, Testcontainers.PostgreSql, Testcontainers.Redis

public class OrderRepositoryIntegrationTests : IAsyncLifetime
{
    private readonly PostgreSqlContainer _postgres = new PostgreSqlBuilder()
        .WithImage("postgres:16-alpine")
        .WithDatabase("testdb")
        .WithUsername("testuser")
        .WithPassword("testpass")
        .Build();

    private readonly RedisContainer _redis = new RedisBuilder()
        .WithImage("redis:7-alpine")
        .Build();

    private AppDbContext _dbContext = null!;
    private IOrderRepository _repository = null!;

    public async Task InitializeAsync()
    {
        await Task.WhenAll(_postgres.StartAsync(), _redis.StartAsync());

        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseNpgsql(_postgres.GetConnectionString())
            .Options;

        _dbContext = new AppDbContext(options);
        await _dbContext.Database.MigrateAsync();

        var redis = await ConnectionMultiplexer.ConnectAsync(_redis.GetConnectionString());
        _repository = new OrderRepository(_dbContext, redis);
    }

    public async Task DisposeAsync()
    {
        await _dbContext.DisposeAsync();
        await Task.WhenAll(_postgres.DisposeAsync().AsTask(), _redis.DisposeAsync().AsTask());
    }

    [Fact]
    public async Task CreateOrder_ShouldPersistAndRetrieve()
    {
        var order = new Order
        {
            Id = Guid.NewGuid(),
            CustomerId = Guid.NewGuid(),
            Items = [new OrderItem("Product A", 2, 10.00m)],
            Status = OrderStatus.Pending,
            CreatedAt = DateTimeOffset.UtcNow
        };

        await _repository.CreateAsync(order);
        var retrieved = await _repository.GetByIdAsync(order.Id);

        retrieved.Should().NotBeNull();
        retrieved!.Id.Should().Be(order.Id);
        retrieved.Items.Should().HaveCount(1);
        retrieved.Total.Should().Be(20.00m);
    }

    [Fact]
    public async Task GetOrdersByCustomer_ShouldReturnOnlyCustomerOrders()
    {
        var customerId = Guid.NewGuid();
        var otherCustomerId = Guid.NewGuid();

        var customerOrders = Enumerable.Range(0, 3).Select(_ => new Order
        {
            Id = Guid.NewGuid(),
            CustomerId = customerId,
            Items = [new OrderItem("Product", 1, 10m)],
            Status = OrderStatus.Pending,
            CreatedAt = DateTimeOffset.UtcNow
        }).ToList();

        var otherOrder = new Order
        {
            Id = Guid.NewGuid(),
            CustomerId = otherCustomerId,
            Items = [new OrderItem("Product", 1, 10m)],
            Status = OrderStatus.Pending,
            CreatedAt = DateTimeOffset.UtcNow
        };

        foreach (var order in customerOrders) await _repository.CreateAsync(order);
        await _repository.CreateAsync(otherOrder);

        var results = await _repository.GetByCustomerIdAsync(customerId);

        results.Should().HaveCount(3);
        results.Should().AllSatisfy(o => o.CustomerId.Should().Be(customerId));
    }

    [Fact]
    public async Task UpdateOrderStatus_ShouldInvalidateCache()
    {
        var order = new Order
        {
            Id = Guid.NewGuid(),
            CustomerId = Guid.NewGuid(),
            Items = [new OrderItem("Product", 1, 10m)],
            Status = OrderStatus.Pending,
            CreatedAt = DateTimeOffset.UtcNow
        };

        await _repository.CreateAsync(order);

        // First read — populates cache
        var first = await _repository.GetByIdAsync(order.Id);
        first!.Status.Should().Be(OrderStatus.Pending);

        // Update status
        await _repository.UpdateStatusAsync(order.Id, OrderStatus.Confirmed);

        // Second read — should reflect update (cache invalidated)
        var second = await _repository.GetByIdAsync(order.Id);
        second!.Status.Should().Be(OrderStatus.Confirmed);
    }
}
```

---

## Step 1407: Database Reset with Respawn

```csharp
// Install: Respawn

public class DatabaseFixture : IAsyncLifetime
{
    public readonly PostgreSqlContainer Postgres = new PostgreSqlBuilder()
        .WithImage("postgres:16-alpine")
        .Build();

    public AppDbContext DbContext { get; private set; } = null!;
    private Respawner _respawner = null!;

    public async Task InitializeAsync()
    {
        await Postgres.StartAsync();

        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseNpgsql(Postgres.GetConnectionString())
            .Options;

        DbContext = new AppDbContext(options);
        await DbContext.Database.MigrateAsync();

        await using var conn = new NpgsqlConnection(Postgres.GetConnectionString());
        await conn.OpenAsync();

        _respawner = await Respawner.CreateAsync(conn, new RespawnerOptions
        {
            DbAdapter = DbAdapter.Postgres,
            SchemasToInclude = ["public"],
            TablesToIgnore = ["__EFMigrationsHistory"]
        });
    }

    public async Task ResetDatabaseAsync()
    {
        await using var conn = new NpgsqlConnection(Postgres.GetConnectionString());
        await conn.OpenAsync();
        await _respawner.ResetAsync(conn);
    }

    public async Task DisposeAsync()
    {
        await DbContext.DisposeAsync();
        await Postgres.DisposeAsync();
    }
}

// Use Respawn to reset between tests
[Collection("Database")]
public class OrderIntegrationTests(DatabaseFixture fixture) : IAsyncLifetime
{
    public Task InitializeAsync() => fixture.ResetDatabaseAsync();
    public Task DisposeAsync() => Task.CompletedTask;

    [Fact]
    public async Task Test1_CreateOrder()
    {
        // Always starts with a clean database
        var order = new Order { /* ... */ };
        await fixture.DbContext.Orders.AddAsync(order);
        await fixture.DbContext.SaveChangesAsync();

        fixture.DbContext.Orders.Should().HaveCount(1);
    }

    [Fact]
    public async Task Test2_ListOrders()
    {
        // Also starts with clean database — no interference from Test1
        fixture.DbContext.Orders.Should().BeEmpty();
    }
}
```

---

## Step 1408: API Integration Tests with WebApplicationFactory

```csharp
public class OrderApiIntegrationTests : IClassFixture<CustomWebApplicationFactory>
{
    private readonly HttpClient _client;

    public OrderApiIntegrationTests(CustomWebApplicationFactory factory)
    {
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task POST_CreateOrder_Returns201WithOrderId()
    {
        var request = new CreateOrderRequest(
            CustomerId: Guid.NewGuid(),
            Items: [new(ProductId: Guid.NewGuid(), Quantity: 2, UnitPrice: 10.00m)]
        );

        var response = await _client.PostAsJsonAsync("/api/v1/orders", request);

        response.StatusCode.Should().Be(HttpStatusCode.Created);
        var order = await response.Content.ReadFromJsonAsync<OrderResponse>();
        order!.Id.Should().NotBeEmpty();
        order.Total.Should().Be(20.00m);
        order.Status.Should().Be("Pending");
    }

    [Fact]
    public async Task GET_Order_WithValidId_Returns200()
    {
        // Arrange: create an order first
        var createRequest = new CreateOrderRequest(Guid.NewGuid(), [new(Guid.NewGuid(), 1, 50m)]);
        var createResponse = await _client.PostAsJsonAsync("/api/v1/orders", createRequest);
        var created = await createResponse.Content.ReadFromJsonAsync<OrderResponse>();

        // Act
        var response = await _client.GetAsync($"/api/v1/orders/{created!.Id}");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
        var order = await response.Content.ReadFromJsonAsync<OrderResponse>();
        order!.Id.Should().Be(created.Id);
    }

    [Fact]
    public async Task GET_Order_WithInvalidId_Returns404()
    {
        var response = await _client.GetAsync($"/api/v1/orders/{Guid.NewGuid()}");
        response.StatusCode.Should().Be(HttpStatusCode.NotFound);
    }

    [Fact]
    public async Task POST_CreateOrder_WithEmptyItems_Returns400WithValidationErrors()
    {
        var request = new CreateOrderRequest(Guid.NewGuid(), []);

        var response = await _client.PostAsJsonAsync("/api/v1/orders", request);

        response.StatusCode.Should().Be(HttpStatusCode.BadRequest);
        var problem = await response.Content.ReadFromJsonAsync<ValidationProblemDetails>();
        problem!.Errors.Should().ContainKey("Items");
    }
}

// Custom factory with test overrides
public class CustomWebApplicationFactory : WebApplicationFactory<Program>
{
    private readonly PostgreSqlContainer _postgres = new PostgreSqlBuilder()
        .WithImage("postgres:16-alpine")
        .Build();

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureTestServices(services =>
        {
            // Replace real database with test container
            var descriptor = services.SingleOrDefault(d => d.ServiceType == typeof(DbContextOptions<AppDbContext>));
            if (descriptor != null) services.Remove(descriptor);

            services.AddDbContext<AppDbContext>(options =>
                options.UseNpgsql(_postgres.GetConnectionString()));

            // Replace external services with mocks
            services.AddScoped<IPaymentService>(_ => new Mock<IPaymentService>().Object);
            services.AddScoped<IEmailService>(_ => new Mock<IEmailService>().Object);

            // Use in-memory event bus for tests
            services.AddScoped<IEventBus, InMemoryEventBus>();
        });
    }

    public override async ValueTask DisposeAsync()
    {
        await _postgres.DisposeAsync();
        await base.DisposeAsync();
    }

    public new async Task<HttpClient> CreateClientAsync()
    {
        await _postgres.StartAsync();

        using var scope = Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        await db.Database.MigrateAsync();

        return CreateClient();
    }
}
```

---

## Step 1409: Contract Testing with Pact

```csharp
// Consumer side: OrderService → NotificationService
// Install: PactNet

public class OrderNotificationConsumerTests : IDisposable
{
    private readonly IPactBuilderV4 _pactBuilder;
    private readonly PactVerifier _pactVerifier;
    private const int MockServerPort = 9001;

    public OrderNotificationConsumerTests()
    {
        var pact = Pact.V4("OrderService", "NotificationService", new PactConfig
        {
            PactDir = "../../../pacts",
            LogLevel = PactLogLevel.Information
        });
        _pactBuilder = pact.WithHttpInteractions();
    }

    [Fact]
    public async Task OrderService_CanSendOrderCreatedNotification()
    {
        // Arrange: define what the consumer expects
        _pactBuilder
            .UponReceiving("a request to send order created notification")
            .Given("notification service is available")
            .WithRequest(HttpMethod.Post, "/api/notifications")
            .WithHeader("Content-Type", "application/json; charset=utf-8")
            .WithJsonBody(new
            {
                type = "ORDER_CREATED",
                customerId = Match.Type("00000000-0000-0000-0000-000000000001"),
                orderId = Match.Type("00000000-0000-0000-0000-000000000002"),
                total = Match.Decimal(45.50)
            })
            .WillRespond()
            .WithStatus(202)
            .WithJsonBody(new { notificationId = Match.Type("notif-123") });

        await _pactBuilder.VerifyAsync(async ctx =>
        {
            // Act: call notification service through real HTTP client
            var client = new NotificationServiceClient($"http://localhost:{MockServerPort}");
            var result = await client.SendOrderCreatedAsync(new OrderCreatedEvent(
                CustomerId: Guid.Parse("00000000-0000-0000-0000-000000000001"),
                OrderId: Guid.Parse("00000000-0000-0000-0000-000000000002"),
                Total: 45.50m
            ));

            // Assert
            result.NotificationId.Should().NotBeNullOrEmpty();
        });
    }

    public void Dispose() => _pactBuilder.Dispose();
}

// Provider side: NotificationService verifies the pact
public class NotificationServiceProviderTests : IClassFixture<CustomWebApplicationFactory>
{
    private readonly CustomWebApplicationFactory _factory;

    public NotificationServiceProviderTests(CustomWebApplicationFactory factory)
    {
        _factory = factory;
    }

    [Fact]
    public void NotificationService_HonoursPactWithOrderService()
    {
        var port = _factory.Server.BaseAddress.Port;

        var pactVerifier = new PactVerifier("NotificationService", new PactVerifierConfig
        {
            LogLevel = PactLogLevel.Information
        });

        pactVerifier
            .WithHttpEndpoint(new Uri($"http://localhost:{port}"))
            .WithFileSource(new FileInfo("../../../pacts/OrderService-NotificationService.json"))
            .WithProviderStateUrl(new Uri($"http://localhost:{port}/provider-states"))
            .Verify();
    }
}

// Provider state setup endpoint
[ApiController]
[Route("provider-states")]
public class ProviderStateController(AppDbContext db) : ControllerBase
{
    [Post("")]
    public async Task<IActionResult> SetupState([FromBody] ProviderState state)
    {
        switch (state.State)
        {
            case "notification service is available":
                // Nothing to set up — service is always available
                break;
        }
        return Ok();
    }
}
```

---

## Step 1410: Load Testing with NBomber

```csharp
// Install: NBomber, NBomber.Http

public class OrderApiLoadTests
{
    [Fact]
    public void OrderCreation_ShouldHandleLoad()
    {
        var httpClient = new HttpClient { BaseAddress = new Uri("http://localhost:5000") };

        var scenario = Scenario.Create("order_creation", async context =>
        {
            var request = new CreateOrderRequest(
                CustomerId: Guid.NewGuid(),
                Items: [new(Guid.NewGuid(), Random.Shared.Next(1, 10), Random.Shared.Next(1, 100) * 1.0m)]
            );

            using var response = await httpClient.PostAsJsonAsync("/api/v1/orders", request);

            return response.IsSuccessStatusCode ? Response.Ok() : Response.Fail(response.StatusCode.ToString());
        })
        .WithWarmUpDuration(TimeSpan.FromSeconds(5))
        .WithLoadSimulations(
            Simulation.RampingInject(rate: 10, interval: TimeSpan.FromSeconds(1), during: TimeSpan.FromSeconds(10)),
            Simulation.Inject(rate: 50, interval: TimeSpan.FromSeconds(1), during: TimeSpan.FromSeconds(30)),
            Simulation.RampingInject(rate: 0, interval: TimeSpan.FromSeconds(1), during: TimeSpan.FromSeconds(10))
        );

        var stats = NBomberRunner
            .RegisterScenarios(scenario)
            .WithReportFolder("./load-test-reports")
            .WithReportFormats(ReportFormat.Html, ReportFormat.Csv)
            .Run();

        // Assert SLOs
        var orderScenario = stats.ScenarioStats[0];
        orderScenario.Ok.Request.Percent.Should().BeGreaterThan(99.0,
            because: "99%+ of requests should succeed");
        orderScenario.Ok.Latency.P99Ms.Should().BeLessThan(500,
            because: "P99 latency should be under 500ms");
        orderScenario.Ok.Latency.P95Ms.Should().BeLessThan(200,
            because: "P95 latency should be under 200ms");
    }

    [Fact]
    public void MixedWorkload_ReadHeavy_ShouldHandleLoad()
    {
        var httpClient = new HttpClient { BaseAddress = new Uri("http://localhost:5000") };
        var orderIds = new ConcurrentBag<Guid>();

        var writeScenario = Scenario.Create("write_orders", async context =>
        {
            var response = await httpClient.PostAsJsonAsync("/api/v1/orders",
                new CreateOrderRequest(Guid.NewGuid(), [new(Guid.NewGuid(), 1, 10m)]));

            if (response.IsSuccessStatusCode)
            {
                var order = await response.Content.ReadFromJsonAsync<OrderResponse>();
                orderIds.Add(order!.Id);
                return Response.Ok();
            }
            return Response.Fail();
        })
        .WithLoadSimulations(Simulation.Inject(rate: 5, interval: TimeSpan.FromSeconds(1), during: TimeSpan.FromSeconds(60)));

        var readScenario = Scenario.Create("read_orders", async context =>
        {
            if (orderIds.IsEmpty) return Response.Ok();

            var orderId = orderIds.ElementAt(Random.Shared.Next(orderIds.Count));
            using var response = await httpClient.GetAsync($"/api/v1/orders/{orderId}");
            return response.IsSuccessStatusCode ? Response.Ok() : Response.Fail();
        })
        .WithLoadSimulations(Simulation.Inject(rate: 45, interval: TimeSpan.FromSeconds(1), during: TimeSpan.FromSeconds(60)));

        NBomberRunner
            .RegisterScenarios(writeScenario, readScenario)
            .WithReportFolder("./load-test-reports")
            .Run();
    }
}
```

---

## Step 1411: Test Data Builders (Fluent API)

```csharp
// Builder pattern for test data — no more fragile object graph construction
public class OrderBuilder
{
    private Guid _id = Guid.NewGuid();
    private Guid _customerId = Guid.NewGuid();
    private readonly List<OrderItem> _items = [];
    private OrderStatus _status = OrderStatus.Pending;
    private DateTimeOffset _createdAt = DateTimeOffset.UtcNow;

    public static OrderBuilder AnOrder() => new();

    public OrderBuilder WithId(Guid id) { _id = id; return this; }
    public OrderBuilder ForCustomer(Guid customerId) { _customerId = customerId; return this; }
    public OrderBuilder WithStatus(OrderStatus status) { _status = status; return this; }
    public OrderBuilder CreatedAt(DateTimeOffset createdAt) { _createdAt = createdAt; return this; }

    public OrderBuilder WithItem(string name, int quantity, decimal price)
    {
        _items.Add(new OrderItem(name, quantity, price));
        return this;
    }

    public OrderBuilder WithItems(int count, decimal price = 10m)
    {
        for (var i = 0; i < count; i++)
            _items.Add(new OrderItem($"Product {i+1}", 1, price));
        return this;
    }

    public OrderBuilder AsConfirmed() => WithStatus(OrderStatus.Confirmed);
    public OrderBuilder AsShipped() => WithStatus(OrderStatus.Shipped);
    public OrderBuilder AsDelivered() => WithStatus(OrderStatus.Delivered);

    public Order Build() => new()
    {
        Id = _id,
        CustomerId = _customerId,
        Items = _items.Count > 0 ? _items : [new OrderItem("Default Product", 1, 10m)],
        Status = _status,
        CreatedAt = _createdAt
    };
}

// Customer builder
public class CustomerBuilder
{
    private Guid _id = Guid.NewGuid();
    private string _firstName = "John";
    private string _lastName = "Doe";
    private string _email = "john.doe@example.com";
    private CustomerTier _tier = CustomerTier.Bronze;
    private bool _isActive = true;

    public static CustomerBuilder ACustomer() => new();

    public CustomerBuilder WithId(Guid id) { _id = id; return this; }
    public CustomerBuilder WithName(string first, string last) { _firstName = first; _lastName = last; return this; }
    public CustomerBuilder WithEmail(string email) { _email = email; return this; }
    public CustomerBuilder AsTier(CustomerTier tier) { _tier = tier; return this; }
    public CustomerBuilder AsGold() => AsTier(CustomerTier.Gold);
    public CustomerBuilder Inactive() { _isActive = false; return this; }

    public Customer Build() => new(_id, _firstName, _lastName, _email, _tier, _isActive);
}

// Usage in tests
public class DiscountCalculatorTests
{
    [Fact]
    public void GoldCustomer_Over1000_GetsMaxDiscount()
    {
        var customer = CustomerBuilder.ACustomer().AsGold().Build();
        var order = OrderBuilder.AnOrder()
            .ForCustomer(customer.Id)
            .WithItems(5, price: 300m)
            .Build();

        var calculator = new DiscountCalculator();
        var discount = calculator.Calculate(order, customer);

        discount.Should().Be(order.Total * 0.15m);
    }

    [Fact]
    public void InactiveCustomer_GetsNoDiscount()
    {
        var customer = CustomerBuilder.ACustomer().AsGold().Inactive().Build();
        var order = OrderBuilder.AnOrder().ForCustomer(customer.Id).WithItems(1, 2000m).Build();

        var calculator = new DiscountCalculator();
        var discount = calculator.Calculate(order, customer);

        discount.Should().Be(0m);
    }
}
```

---

## Step 1412: Snapshot Testing with Verify

```csharp
// Install: Verify.Xunit, Verify.Http

[UsesVerify]
public class OrderSerializationTests
{
    [Fact]
    public async Task Order_SerializesToExpectedJson()
    {
        var order = OrderBuilder.AnOrder()
            .WithId(Guid.Parse("12345678-1234-1234-1234-123456789abc"))
            .ForCustomer(Guid.Parse("aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa"))
            .WithItem("Widget A", 2, 10.00m)
            .WithItem("Widget B", 1, 25.50m)
            .CreatedAt(new DateTimeOffset(2024, 1, 15, 10, 0, 0, TimeSpan.Zero))
            .Build();

        await Verify(order).UseStrictJson();
        // Creates/updates: OrderSerializationTests.Order_SerializesToExpectedJson.verified.json
    }

    [Fact]
    public async Task OrderListResponse_MatchesSnapshot()
    {
        var orders = Enumerable.Range(1, 3).Select(i =>
            OrderBuilder.AnOrder()
                .WithId(Guid.Parse($"0000000{i}-0000-0000-0000-000000000000"))
                .WithItem($"Product {i}", i, i * 10m)
                .Build()
        ).ToList();

        var response = new OrderListResponse(orders, TotalCount: 3, Page: 1, PageSize: 10);

        await Verify(response);
    }
}

// Snapshot file (auto-generated on first run):
// OrderSerializationTests.Order_SerializesToExpectedJson.verified.json
/*
{
  "id": "12345678-1234-1234-1234-123456789abc",
  "customerId": "aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa",
  "items": [
    { "name": "Widget A", "quantity": 2, "unitPrice": 10.00 },
    { "name": "Widget B", "quantity": 1, "unitPrice": 25.50 }
  ],
  "total": 45.50,
  "status": "Pending",
  "createdAt": "2024-01-15T10:00:00+00:00"
}
*/

// Scrubbing date/time and GUIDs in snapshots
[UsesVerify]
public class ApiSnapshotTests
{
    static ApiSnapshotTests()
    {
        VerifierSettings.ScrubInlineGuids();
        VerifierSettings.ScrubMembers("CreatedAt", "UpdatedAt", "Id");
    }

    [Fact]
    public async Task CreateOrder_ResponseMatchesSnapshot()
    {
        // Date and IDs scrubbed — stable snapshots
        var response = new OrderResponse(
            Id: Guid.NewGuid(),     // scrubbed → "Guid_1"
            Total: 45.50m,
            Status: "Pending",
            CreatedAt: DateTimeOffset.UtcNow  // scrubbed
        );

        await Verify(response);
    }
}
```

---

## Step 1413: Testing MediatR Handlers

```csharp
public class CreateOrderCommandHandlerTests
{
    private readonly Mock<IOrderRepository> _orderRepoMock = new();
    private readonly Mock<IInventoryService> _inventoryMock = new();
    private readonly Mock<IEventBus> _eventBusMock = new();
    private readonly Mock<ILogger<CreateOrderCommandHandler>> _loggerMock = new();
    private readonly CreateOrderCommandHandler _handler;

    public CreateOrderCommandHandlerTests()
    {
        _handler = new CreateOrderCommandHandler(
            _orderRepoMock.Object,
            _inventoryMock.Object,
            _eventBusMock.Object,
            _loggerMock.Object
        );
    }

    [Fact]
    public async Task Handle_ValidCommand_CreatesOrderAndPublishesEvent()
    {
        // Arrange
        var command = new CreateOrderCommand(
            CustomerId: Guid.NewGuid(),
            Items: [new CreateOrderCommand.Item(Guid.NewGuid(), 2, 10m)]
        );

        _inventoryMock
            .Setup(x => x.CheckAvailabilityAsync(It.IsAny<Guid>(), It.IsAny<int>(), It.IsAny<CancellationToken>()))
            .ReturnsAsync(true);

        _orderRepoMock
            .Setup(x => x.CreateAsync(It.IsAny<Order>(), It.IsAny<CancellationToken>()))
            .Returns(Task.CompletedTask);

        // Act
        var result = await _handler.Handle(command, CancellationToken.None);

        // Assert
        result.Should().NotBeNull();
        result.Total.Should().Be(20m);

        _orderRepoMock.Verify(x => x.CreateAsync(
            It.Is<Order>(o => o.CustomerId == command.CustomerId),
            It.IsAny<CancellationToken>()
        ), Times.Once);

        _eventBusMock.Verify(x => x.PublishAsync(
            It.Is<OrderCreatedEvent>(e => e.OrderId == result.OrderId),
            It.IsAny<CancellationToken>()
        ), Times.Once);
    }

    [Fact]
    public async Task Handle_InsufficientInventory_ThrowsInsufficientInventoryException()
    {
        var command = new CreateOrderCommand(
            CustomerId: Guid.NewGuid(),
            Items: [new CreateOrderCommand.Item(Guid.NewGuid(), 100, 10m)]
        );

        _inventoryMock
            .Setup(x => x.CheckAvailabilityAsync(It.IsAny<Guid>(), It.IsAny<int>(), It.IsAny<CancellationToken>()))
            .ReturnsAsync(false);

        var act = async () => await _handler.Handle(command, CancellationToken.None);

        await act.Should().ThrowAsync<InsufficientInventoryException>();

        _orderRepoMock.Verify(x => x.CreateAsync(It.IsAny<Order>(), It.IsAny<CancellationToken>()), Times.Never);
        _eventBusMock.Verify(x => x.PublishAsync(It.IsAny<OrderCreatedEvent>(), It.IsAny<CancellationToken>()), Times.Never);
    }
}

// Pipeline behavior tests
public class ValidationBehaviorTests
{
    [Fact]
    public async Task Handle_InvalidRequest_ThrowsValidationException()
    {
        var validators = new List<IValidator<CreateOrderCommand>>
        {
            new CreateOrderCommandValidator()
        };

        var behavior = new ValidationBehavior<CreateOrderCommand, OrderCreatedResponse>(validators);
        var invalidCommand = new CreateOrderCommand(Guid.Empty, []); // invalid: empty customer ID, no items

        var act = async () => await behavior.Handle(
            invalidCommand,
            () => Task.FromResult(new OrderCreatedResponse(Guid.NewGuid(), 0m)),
            CancellationToken.None
        );

        var exception = await act.Should().ThrowAsync<ValidationException>();
        exception.Which.Errors.Should().ContainSingle(e => e.PropertyName == "CustomerId");
        exception.Which.Errors.Should().ContainSingle(e => e.PropertyName == "Items");
    }
}
```

---

## Step 1414: Testing Background Services

```csharp
public class OutboxProcessorTests
{
    [Fact]
    public async Task ExecuteAsync_PublishesUnsentMessages()
    {
        // Arrange
        var messages = new List<OutboxMessage>
        {
            new() { Id = Guid.NewGuid(), Type = "OrderCreated", Payload = "{}", CreatedAt = DateTimeOffset.UtcNow },
            new() { Id = Guid.NewGuid(), Type = "OrderShipped", Payload = "{}", CreatedAt = DateTimeOffset.UtcNow }
        };

        var repoMock = new Mock<IOutboxRepository>();
        repoMock.Setup(x => x.GetUnsentAsync(It.IsAny<int>(), It.IsAny<CancellationToken>()))
                .ReturnsAsync(messages);

        var eventBusMock = new Mock<IEventBus>();
        eventBusMock.Setup(x => x.PublishRawAsync(It.IsAny<string>(), It.IsAny<string>(), It.IsAny<CancellationToken>()))
                    .Returns(Task.CompletedTask);

        var processor = new OutboxProcessor(repoMock.Object, eventBusMock.Object, NullLogger<OutboxProcessor>.Instance);

        using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));

        // Act
        await processor.StartAsync(cts.Token);
        await Task.Delay(100); // Allow one iteration

        // Assert
        eventBusMock.Verify(x => x.PublishRawAsync(
            It.IsAny<string>(), It.IsAny<string>(), It.IsAny<CancellationToken>()),
            Times.Exactly(2));

        repoMock.Verify(x => x.MarkAsSentAsync(
            It.IsIn(messages.Select(m => m.Id)),
            It.IsAny<CancellationToken>()),
            Times.Exactly(2));

        await processor.StopAsync(cts.Token);
    }

    [Fact]
    public async Task ExecuteAsync_OnEventBusFailure_MarksMessageAsFailed()
    {
        var message = new OutboxMessage { Id = Guid.NewGuid(), Type = "OrderCreated", Payload = "{}", CreatedAt = DateTimeOffset.UtcNow };

        var repoMock = new Mock<IOutboxRepository>();
        repoMock.Setup(x => x.GetUnsentAsync(It.IsAny<int>(), It.IsAny<CancellationToken>()))
                .ReturnsAsync([message]);

        var eventBusMock = new Mock<IEventBus>();
        eventBusMock.Setup(x => x.PublishRawAsync(It.IsAny<string>(), It.IsAny<string>(), It.IsAny<CancellationToken>()))
                    .ThrowsAsync(new Exception("Event bus unavailable"));

        var processor = new OutboxProcessor(repoMock.Object, eventBusMock.Object, NullLogger<OutboxProcessor>.Instance);

        using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
        await processor.StartAsync(cts.Token);
        await Task.Delay(100);

        repoMock.Verify(x => x.MarkAsFailedAsync(message.Id, It.IsAny<string>(), It.IsAny<CancellationToken>()), Times.Once);
        await processor.StopAsync(cts.Token);
    }
}
```

---

## Step 1415: Testing SignalR Hubs

```csharp
// Install: Microsoft.AspNetCore.SignalR.Client, Microsoft.AspNetCore.TestHost

public class OrderHubIntegrationTests : IClassFixture<CustomWebApplicationFactory>
{
    private readonly CustomWebApplicationFactory _factory;

    public OrderHubIntegrationTests(CustomWebApplicationFactory factory)
    {
        _factory = factory;
    }

    private HubConnection CreateConnection(string path = "/hubs/orders")
    {
        return new HubConnectionBuilder()
            .WithUrl($"{_factory.Server.BaseAddress.AbsoluteUri.TrimEnd('/')}{path}",
                options => options.HttpMessageHandlerFactory = _ => _factory.Server.CreateHandler())
            .Build();
    }

    [Fact]
    public async Task Client_ReceivesOrderStatusUpdate_WhenOrderIsUpdated()
    {
        var orderId = Guid.NewGuid();
        var customerId = Guid.NewGuid();

        // Arrange: create an order in the database
        await _factory.SeedOrderAsync(orderId, customerId);

        using var connection = CreateConnection();
        var received = new TaskCompletionSource<OrderStatusUpdate>(TaskCreationOptions.RunContinuationsAsynchronously);

        connection.On<OrderStatusUpdate>("OrderStatusUpdated", update =>
        {
            if (update.OrderId == orderId)
                received.TrySetResult(update);
        });

        await connection.StartAsync();
        await connection.InvokeAsync("JoinCustomerGroup", customerId);

        // Act: trigger a status update
        var hubContext = _factory.Services.GetRequiredService<IHubContext<OrderHub, IOrderClient>>();
        await hubContext.Clients.Group($"customer:{customerId}")
            .OrderStatusUpdated(new OrderStatusUpdate(orderId, "Confirmed", DateTimeOffset.UtcNow));

        // Assert
        var update = await received.Task.WaitAsync(TimeSpan.FromSeconds(5));
        update.OrderId.Should().Be(orderId);
        update.Status.Should().Be("Confirmed");

        await connection.StopAsync();
    }

    [Fact]
    public async Task MultipleClients_BothReceiveGroupBroadcast()
    {
        var customerId = Guid.NewGuid();
        var orderId = Guid.NewGuid();

        using var conn1 = CreateConnection();
        using var conn2 = CreateConnection();

        var received1 = new TaskCompletionSource<OrderStatusUpdate>();
        var received2 = new TaskCompletionSource<OrderStatusUpdate>();

        conn1.On<OrderStatusUpdate>("OrderStatusUpdated", u => received1.TrySetResult(u));
        conn2.On<OrderStatusUpdate>("OrderStatusUpdated", u => received2.TrySetResult(u));

        await Task.WhenAll(conn1.StartAsync(), conn2.StartAsync());
        await Task.WhenAll(
            conn1.InvokeAsync("JoinCustomerGroup", customerId),
            conn2.InvokeAsync("JoinCustomerGroup", customerId)
        );

        var hubContext = _factory.Services.GetRequiredService<IHubContext<OrderHub, IOrderClient>>();
        await hubContext.Clients.Group($"customer:{customerId}")
            .OrderStatusUpdated(new OrderStatusUpdate(orderId, "Shipped", DateTimeOffset.UtcNow));

        await Task.WhenAll(
            received1.Task.WaitAsync(TimeSpan.FromSeconds(5)),
            received2.Task.WaitAsync(TimeSpan.FromSeconds(5))
        );

        received1.Task.Result.Status.Should().Be("Shipped");
        received2.Task.Result.Status.Should().Be("Shipped");

        await Task.WhenAll(conn1.StopAsync(), conn2.StopAsync());
    }
}
```

---

## Step 1416: Testing Minimal APIs

```csharp
// Minimal API endpoint testing
public class ProductEndpointTests : IClassFixture<CustomWebApplicationFactory>
{
    private readonly HttpClient _client;

    public ProductEndpointTests(CustomWebApplicationFactory factory)
    {
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task GetProducts_Returns200WithPaginatedList()
    {
        var response = await _client.GetAsync("/api/v1/products?page=1&pageSize=10");

        response.StatusCode.Should().Be(HttpStatusCode.OK);
        var result = await response.Content.ReadFromJsonAsync<PagedResponse<ProductDto>>();
        result.Should().NotBeNull();
        result!.Items.Should().NotBeNull();
        result.Page.Should().Be(1);
        result.PageSize.Should().Be(10);
    }

    [Fact]
    public async Task CreateProduct_Authenticated_Returns201()
    {
        _client.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", GenerateTestJwt(roles: ["Admin"]));

        var request = new CreateProductRequest("Test Product", 29.99m, 100);

        var response = await _client.PostAsJsonAsync("/api/v1/products", request);

        response.StatusCode.Should().Be(HttpStatusCode.Created);
        response.Headers.Location.Should().NotBeNull();
    }

    [Fact]
    public async Task CreateProduct_Unauthenticated_Returns401()
    {
        _client.DefaultRequestHeaders.Authorization = null;

        var response = await _client.PostAsJsonAsync("/api/v1/products",
            new CreateProductRequest("Test", 10m, 1));

        response.StatusCode.Should().Be(HttpStatusCode.Unauthorized);
    }

    [Fact]
    public async Task CreateProduct_WithoutAdminRole_Returns403()
    {
        _client.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", GenerateTestJwt(roles: ["User"]));

        var response = await _client.PostAsJsonAsync("/api/v1/products",
            new CreateProductRequest("Test", 10m, 1));

        response.StatusCode.Should().Be(HttpStatusCode.Forbidden);
    }

    private string GenerateTestJwt(string[] roles)
    {
        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes("test-secret-key-minimum-256-bits!"));
        var credentials = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

        var claims = new List<Claim>
        {
            new(ClaimTypes.NameIdentifier, Guid.NewGuid().ToString()),
            new(ClaimTypes.Email, "test@example.com")
        };
        claims.AddRange(roles.Select(r => new Claim(ClaimTypes.Role, r)));

        var token = new JwtSecurityToken(
            issuer: "test",
            audience: "test",
            claims: claims,
            expires: DateTime.UtcNow.AddHours(1),
            signingCredentials: credentials
        );

        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}
```

---

## Step 1417: Testing EF Core with In-Memory Provider

```csharp
// Using SQLite in-memory for realistic EF Core tests (better than InMemory provider)
public class OrderQueryServiceTests : IDisposable
{
    private readonly SqliteConnection _connection;
    private readonly AppDbContext _dbContext;
    private readonly OrderQueryService _service;

    public OrderQueryServiceTests()
    {
        _connection = new SqliteConnection("DataSource=:memory:");
        _connection.Open();

        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlite(_connection)
            .Options;

        _dbContext = new AppDbContext(options);
        _dbContext.Database.EnsureCreated();

        _service = new OrderQueryService(_dbContext);
    }

    [Fact]
    public async Task GetOrdersWithPaging_ReturnsCorrectPage()
    {
        // Seed 15 orders
        var orders = Enumerable.Range(1, 15).Select(i => new Order
        {
            Id = Guid.NewGuid(),
            CustomerId = Guid.NewGuid(),
            Items = [new OrderItem($"Product {i}", 1, i * 10m)],
            Status = OrderStatus.Pending,
            CreatedAt = DateTimeOffset.UtcNow.AddDays(-i)
        }).ToList();

        await _dbContext.Orders.AddRangeAsync(orders);
        await _dbContext.SaveChangesAsync();

        // Get page 2 (items 6-10)
        var result = await _service.GetOrdersAsync(page: 2, pageSize: 5);

        result.Items.Should().HaveCount(5);
        result.TotalCount.Should().Be(15);
        result.TotalPages.Should().Be(3);
    }

    [Fact]
    public async Task GetOrdersByStatus_FiltersCorrectly()
    {
        var pending = Enumerable.Range(0, 3).Select(_ => new Order
        {
            Id = Guid.NewGuid(), CustomerId = Guid.NewGuid(),
            Items = [new OrderItem("P", 1, 10m)],
            Status = OrderStatus.Pending,
            CreatedAt = DateTimeOffset.UtcNow
        });

        var confirmed = Enumerable.Range(0, 2).Select(_ => new Order
        {
            Id = Guid.NewGuid(), CustomerId = Guid.NewGuid(),
            Items = [new OrderItem("P", 1, 10m)],
            Status = OrderStatus.Confirmed,
            CreatedAt = DateTimeOffset.UtcNow
        });

        await _dbContext.Orders.AddRangeAsync([..pending, ..confirmed]);
        await _dbContext.SaveChangesAsync();

        var pendingResult = await _service.GetOrdersByStatusAsync(OrderStatus.Pending);
        pendingResult.Should().HaveCount(3);

        var confirmedResult = await _service.GetOrdersByStatusAsync(OrderStatus.Confirmed);
        confirmedResult.Should().HaveCount(2);
    }

    public void Dispose()
    {
        _dbContext.Dispose();
        _connection.Dispose();
    }
}
```

---

## Step 1418: Test Coverage Configuration

```xml
<!-- Directory.Build.props - coverage for all test projects -->
<PropertyGroup Condition="'$(IsTestProject)' == 'true'">
  <CollectCoverage>true</CollectCoverage>
  <CoverletOutputFormat>opencover</CoverletOutputFormat>
  <CoverletOutput>./TestResults/coverage.opencover.xml</CoverletOutput>
  <Exclude>[*.Tests]*,[*.*Tests]*</Exclude>
  <ExcludeByAttribute>GeneratedCodeAttribute,CompilerGeneratedAttribute</ExcludeByAttribute>
  <ThresholdType>line,branch,method</ThresholdType>
  <Threshold>80</Threshold>  <!-- Fail CI if coverage drops below 80% -->
</PropertyGroup>
```

```yaml
# .github/workflows/test.yml
name: Tests & Coverage

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: testdb
        ports: ["5432:5432"]
      redis:
        image: redis:7-alpine
        ports: ["6379:6379"]

    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: 9.0.x

      - name: Run tests with coverage
        run: |
          dotnet test --configuration Release \
            --collect:"XPlat Code Coverage" \
            --results-directory TestResults/ \
            --logger "trx;LogFileName=test-results.trx" \
            -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.Format=opencover

      - name: Generate coverage report
        run: |
          dotnet tool install -g dotnet-reportgenerator-globaltool
          reportgenerator \
            -reports:"TestResults/**/coverage.opencover.xml" \
            -targetdir:"TestResults/CoverageReport" \
            -reporttypes:"Html;Badges;TextSummary"

      - name: Upload coverage report
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: TestResults/CoverageReport/

      - name: Comment PR with coverage
        uses: MishaKav/pytest-coverage-comment@main
        if: github.event_name == 'pull_request'
        with:
          coverage-path: TestResults/CoverageReport/Summary.txt
```

---

## Step 1419: Testing Resilience Policies (Polly)

```csharp
public class PaymentServiceResilienceTests
{
    [Fact]
    public async Task RetryPolicy_RetriesOnTransientFailure()
    {
        var callCount = 0;
        var mockPayment = new Mock<IPaymentGateway>();

        mockPayment.Setup(x => x.ChargeAsync(It.IsAny<ChargeRequest>(), It.IsAny<CancellationToken>()))
                   .ReturnsAsync(() =>
                   {
                       callCount++;
                       if (callCount < 3) throw new HttpRequestException("Transient error");
                       return new ChargeResult { Success = true, TransactionId = "tx-123" };
                   });

        var pipeline = new ResiliencePipelineBuilder<ChargeResult>()
            .AddRetry(new RetryStrategyOptions<ChargeResult>
            {
                MaxRetryAttempts = 3,
                Delay = TimeSpan.Zero, // No delay in tests
                ShouldHandle = new PredicateBuilder<ChargeResult>().Handle<HttpRequestException>()
            })
            .Build();

        var result = await pipeline.ExecuteAsync(async ct =>
            await mockPayment.Object.ChargeAsync(new ChargeRequest(100m, "tok_test"), ct));

        result.Success.Should().BeTrue();
        callCount.Should().Be(3); // Failed twice, succeeded on third
    }

    [Fact]
    public async Task CircuitBreaker_OpensAfterThreshold()
    {
        var callCount = 0;
        var mockPayment = new Mock<IPaymentGateway>();

        mockPayment.Setup(x => x.ChargeAsync(It.IsAny<ChargeRequest>(), It.IsAny<CancellationToken>()))
                   .ThrowsAsync(new HttpRequestException("Service down"));

        var pipeline = new ResiliencePipelineBuilder<ChargeResult>()
            .AddCircuitBreaker(new CircuitBreakerStrategyOptions<ChargeResult>
            {
                FailureRatio = 0.5,
                MinimumThroughput = 4,
                SamplingDuration = TimeSpan.FromSeconds(10),
                BreakDuration = TimeSpan.FromSeconds(30),
                ShouldHandle = new PredicateBuilder<ChargeResult>().Handle<HttpRequestException>()
            })
            .Build();

        // Make enough calls to open the circuit
        for (var i = 0; i < 4; i++)
        {
            try { await pipeline.ExecuteAsync(async ct => await mockPayment.Object.ChargeAsync(new(100m, "tok"), ct)); }
            catch { /* expected */ }
        }

        // Circuit should now be open
        var act = async () => await pipeline.ExecuteAsync(async ct =>
            await mockPayment.Object.ChargeAsync(new(100m, "tok"), ct));

        await act.Should().ThrowAsync<BrokenCircuitException>();
    }

    [Fact]
    public async Task Timeout_ThrowsAfterDeadline()
    {
        var mockPayment = new Mock<IPaymentGateway>();
        mockPayment.Setup(x => x.ChargeAsync(It.IsAny<ChargeRequest>(), It.IsAny<CancellationToken>()))
                   .Returns(async (ChargeRequest _, CancellationToken ct) =>
                   {
                       await Task.Delay(TimeSpan.FromSeconds(10), ct);
                       return new ChargeResult { Success = true };
                   });

        var pipeline = new ResiliencePipelineBuilder<ChargeResult>()
            .AddTimeout(TimeSpan.FromMilliseconds(100))
            .Build();

        var act = async () => await pipeline.ExecuteAsync(async ct =>
            await mockPayment.Object.ChargeAsync(new(100m, "tok"), ct));

        await act.Should().ThrowAsync<TimeoutRejectedException>();
    }
}
```

---

## Step 1420: Testing Domain Events

```csharp
public class OrderAggregateTests
{
    [Fact]
    public void CreateOrder_RaisesOrderCreatedEvent()
    {
        var customerId = Guid.NewGuid();
        var items = new List<OrderItem> { new("Product A", 2, 10m) };

        var order = Order.Create(customerId, items);

        order.DomainEvents.Should().ContainSingle(e => e is OrderCreatedEvent);
        var @event = order.DomainEvents.OfType<OrderCreatedEvent>().Single();
        @event.OrderId.Should().Be(order.Id);
        @event.CustomerId.Should().Be(customerId);
        @event.Total.Should().Be(20m);
    }

    [Fact]
    public void ConfirmOrder_RaisesOrderConfirmedEvent()
    {
        var order = CreatePendingOrder();

        order.Confirm();

        order.DomainEvents.Should().Contain(e => e is OrderConfirmedEvent);
        order.Status.Should().Be(OrderStatus.Confirmed);
    }

    [Fact]
    public void ConfirmOrder_AlreadyConfirmed_ThrowsDomainException()
    {
        var order = CreatePendingOrder();
        order.Confirm();
        order.ClearEvents(); // Clear events between steps

        var act = () => order.Confirm();

        act.Should().Throw<DomainException>().WithMessage("*already confirmed*");
    }

    [Fact]
    public void CancelOrder_AfterShipment_ThrowsDomainException()
    {
        var order = CreatePendingOrder();
        order.Confirm();
        order.Ship("TRACK-123");
        order.ClearEvents();

        var act = () => order.Cancel("Changed mind");

        act.Should().Throw<DomainException>().WithMessage("*cannot cancel*shipped*");
    }

    [Fact]
    public void ApplyEvents_RehydratesAggregate()
    {
        var orderId = Guid.NewGuid();
        var customerId = Guid.NewGuid();

        var events = new List<DomainEvent>
        {
            new OrderCreatedEvent(orderId, customerId, [new("Product", 1, 50m)], 50m, DateTimeOffset.UtcNow),
            new OrderConfirmedEvent(orderId, DateTimeOffset.UtcNow.AddMinutes(5)),
            new OrderShippedEvent(orderId, "TRACK-456", DateTimeOffset.UtcNow.AddHours(2))
        };

        var order = Order.Rehydrate(events);

        order.Id.Should().Be(orderId);
        order.CustomerId.Should().Be(customerId);
        order.Status.Should().Be(OrderStatus.Shipped);
        order.TrackingNumber.Should().Be("TRACK-456");
        order.DomainEvents.Should().BeEmpty(); // Events were applied, not raised
    }

    private static Order CreatePendingOrder()
    {
        var order = Order.Create(Guid.NewGuid(), [new OrderItem("Product", 1, 10m)]);
        order.ClearEvents();
        return order;
    }
}
```

---

## Step 1421: Test Project Configuration

```xml
<!-- OrderService.Tests.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <IsPackable>false</IsPackable>
    <IsTestProject>true</IsTestProject>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
  </PropertyGroup>

  <ItemGroup>
    <!-- Test frameworks -->
    <PackageReference Include="xunit" Version="2.9.*" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.8.*" />
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.12.*" />

    <!-- Assertions -->
    <PackageReference Include="FluentAssertions" Version="6.12.*" />

    <!-- Mocking -->
    <PackageReference Include="Moq" Version="4.20.*" />
    <PackageReference Include="NSubstitute" Version="5.3.*" />

    <!-- Test infrastructure -->
    <PackageReference Include="Testcontainers" Version="3.11.*" />
    <PackageReference Include="Testcontainers.PostgreSql" Version="3.11.*" />
    <PackageReference Include="Testcontainers.Redis" Version="3.11.*" />
    <PackageReference Include="Respawn" Version="6.2.*" />

    <!-- Integration testing -->
    <PackageReference Include="Microsoft.AspNetCore.Mvc.Testing" Version="9.0.*" />

    <!-- BDD -->
    <PackageReference Include="Reqnroll" Version="2.4.*" />
    <PackageReference Include="Reqnroll.xUnit" Version="2.4.*" />

    <!-- Property-based -->
    <PackageReference Include="FsCheck.Xunit" Version="3.1.*" />

    <!-- Snapshot testing -->
    <PackageReference Include="Verify.Xunit" Version="28.1.*" />

    <!-- Contract testing -->
    <PackageReference Include="PactNet" Version="5.0.*" />

    <!-- Load testing -->
    <PackageReference Include="NBomber" Version="5.7.*" />
    <PackageReference Include="NBomber.Http" Version="5.7.*" />

    <!-- Architecture testing -->
    <PackageReference Include="NetArchTest.Rules" Version="1.3.*" />

    <!-- Coverage -->
    <PackageReference Include="coverlet.collector" Version="6.0.*" />

    <!-- Database -->
    <PackageReference Include="Microsoft.Data.Sqlite" Version="9.0.*" />
    <PackageReference Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="9.0.*" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="../OrderService/OrderService.csproj" />
  </ItemGroup>
</Project>
```

---

## Step 1422: Testing Kafka Consumers

```csharp
public class OrderCreatedConsumerTests
{
    [Fact]
    public async Task ConsumeOrderCreated_ShouldUpdateInventory()
    {
        // Arrange
        var inventoryMock = new Mock<IInventoryService>();
        var loggerMock = new Mock<ILogger<OrderCreatedConsumer>>();

        var consumer = new OrderCreatedConsumer(inventoryMock.Object, loggerMock.Object);

        var orderCreatedEvent = new OrderCreatedEvent(
            OrderId: Guid.NewGuid(),
            CustomerId: Guid.NewGuid(),
            Items: [new(Guid.NewGuid(), 2), new(Guid.NewGuid(), 1)],
            Total: 75m,
            OccurredAt: DateTimeOffset.UtcNow
        );

        var consumeContext = new Mock<ConsumeContext<OrderCreatedEvent>>();
        consumeContext.Setup(x => x.Message).Returns(orderCreatedEvent);

        // Act
        await consumer.Consume(consumeContext.Object);

        // Assert
        inventoryMock.Verify(x =>
            x.ReserveItemsAsync(
                It.Is<IEnumerable<InventoryReservation>>(items =>
                    items.Any(i => i.Quantity == 2) &&
                    items.Any(i => i.Quantity == 1)),
                It.IsAny<CancellationToken>()),
            Times.Once);
    }

    [Fact]
    public async Task ConsumeOrderCreated_WhenInventoryFails_PublishesCompensationEvent()
    {
        var inventoryMock = new Mock<IInventoryService>();
        inventoryMock.Setup(x => x.ReserveItemsAsync(It.IsAny<IEnumerable<InventoryReservation>>(), It.IsAny<CancellationToken>()))
                     .ThrowsAsync(new InsufficientStockException("Not enough stock"));

        var publishEndpoint = new Mock<IPublishEndpoint>();
        var consumer = new OrderCreatedConsumer(inventoryMock.Object, publishEndpoint.Object, NullLogger<OrderCreatedConsumer>.Instance);

        var message = new OrderCreatedEvent(Guid.NewGuid(), Guid.NewGuid(), [], 0m, DateTimeOffset.UtcNow);
        var consumeContext = new Mock<ConsumeContext<OrderCreatedEvent>>();
        consumeContext.Setup(x => x.Message).Returns(message);

        await consumer.Consume(consumeContext.Object);

        publishEndpoint.Verify(x =>
            x.Publish(It.IsAny<OrderCancelledEvent>(), It.IsAny<CancellationToken>()),
            Times.Once);
    }
}
```

---

## Step 1423: Parameterized Test Data with ClassData/MemberData

```csharp
// ClassData - use a class to provide test cases
public class ValidOrderTestCases : IEnumerable<object[]>
{
    public IEnumerator<object[]> GetEnumerator()
    {
        // Single item
        yield return [new List<OrderItem> { new("Product A", 1, 10m) }, 10m];
        // Multiple items
        yield return [new List<OrderItem> { new("A", 2, 10m), new("B", 1, 25m) }, 45m];
        // High value
        yield return [new List<OrderItem> { new("Premium", 1, 9999.99m) }, 9999.99m];
        // Many items
        yield return [Enumerable.Range(1, 10).Select(i => new OrderItem($"P{i}", i, 1m)).ToList(), 55m];
    }

    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}

public class OrderCalculationTests
{
    [Theory]
    [ClassData(typeof(ValidOrderTestCases))]
    public void CreateOrder_CalculatesTotalCorrectly(List<OrderItem> items, decimal expectedTotal)
    {
        var service = new OrderService();
        var order = service.CreateOrder(Guid.NewGuid(), items);
        order.Total.Should().Be(expectedTotal);
    }

    // MemberData - use a static method
    public static IEnumerable<object[]> GetDiscountScenarios()
    {
        yield return [CustomerTier.Bronze, 500m, 0m];         // Below threshold
        yield return [CustomerTier.Bronze, 1001m, 50.05m];    // Bronze: 5%
        yield return [CustomerTier.Silver, 1001m, 100.10m];   // Silver: 10%
        yield return [CustomerTier.Gold, 1001m, 150.15m];     // Gold: 15%
        yield return [CustomerTier.Gold, 999.99m, 0m];        // Just below threshold
    }

    [Theory]
    [MemberData(nameof(GetDiscountScenarios))]
    public void CalculateDiscount_GivenTierAndTotal_ReturnsCorrectDiscount(
        CustomerTier tier, decimal orderTotal, decimal expectedDiscount)
    {
        var engine = new PricingEngine();
        var discount = engine.CalculateDiscount(orderTotal, tier);
        discount.Should().BeApproximately(expectedDiscount, 0.01m);
    }
}
```

---

## Step 1424: Golden Path Test Strategy

```csharp
// Complete test pyramid example for the Order domain

// ============================================================
// UNIT TESTS (70%): Fast, isolated, many
// ============================================================
public class OrderDomainUnitTests
{
    // Business rules
    [Fact] public void Confirm_PendingOrder_Succeeds() { }
    [Fact] public void Confirm_CancelledOrder_Throws() { }
    [Fact] public void AddItem_IncreasesTotal() { }
    [Fact] public void RemoveLastItem_ThrowsDomainException() { }

    // Value objects
    [Fact] public void Money_Addition_ReturnsCorrectSum() { }
    [Fact] public void Money_NegativeAmount_ThrowsArgumentException() { }

    // Services
    [Fact] public void PricingEngine_Gold1001_Returns15Percent() { }
    [Fact] public void DiscountCalculator_InactiveCustomer_Returns0() { }
}

// ============================================================
// INTEGRATION TESTS (20%): Slower, cross-boundary
// ============================================================
public class OrderIntegrationTests
{
    // Repository + DB
    [Fact] public async Task Save_And_Reload_PreservesAllFields() { }
    [Fact] public async Task Query_WithFilters_ExecutesSingleQuery() { }

    // Handler + services
    [Fact] public async Task CreateOrderCommand_ValidInput_PersistsAndPublishesEvent() { }
    [Fact] public async Task CreateOrderCommand_InvalidInventory_DoesNotPersist() { }
}

// ============================================================
// E2E / CONTRACT TESTS (10%): Slowest, full-stack
// ============================================================
public class OrderE2ETests
{
    // API contract
    [Fact] public async Task POST_CreateOrder_Returns201_AndRetrievable() { }
    [Fact] public async Task SignalR_OrderUpdate_BroadcastsToGroup() { }
    [Fact] public async Task Consumer_OrderCreated_InventoryReserved() { }

    // Pact contracts
    [Fact] public void OrderService_HonoursPactWithNotificationService() { }
}

// Test organization conventions:
// - Arrange/Act/Assert pattern in all tests
// - One assertion per test (logical assertions OK)
// - Descriptive test names: MethodName_Scenario_ExpectedBehavior
// - No production code changes for testability (avoid "for testing" comments)
// - Fast unit tests: zero I/O, zero network, zero sleeps
// - Shared fixtures with IClassFixture/ICollectionFixture for expensive setup
```

---

## Step 1425: Test Quality Checklist

```csharp
// ✅ Good test practices demonstrated

// 1. Single responsibility — each test verifies one behavior
[Fact]
public void ApplyDiscount_GoldCustomer_ReducesTotal()
{
    var order = OrderBuilder.AnOrder().WithItems(1, 2000m).Build();
    order.ApplyDiscount(CustomerTier.Gold);
    order.Total.Should().Be(1700m); // 2000 - 15% = 1700
}

// ❌ Avoid: Testing too much in one test
[Fact]
public void OrderWorkflow_BadExample()
{
    // Creates order, confirms, ships, delivers, verifies discounts, checks events...
    // When this fails, which step broke?
}

// 2. Test behavior, not implementation
// ✅ Tests WHAT the method does, not HOW
[Fact]
public void CreateOrder_SetsCorrectInitialStatus()
{
    var order = new OrderService().CreateOrder(Guid.NewGuid(), [new("P", 1, 10m)]);
    order.Status.Should().Be(OrderStatus.Pending);
}

// ❌ Avoid: Testing internal state / implementation details
[Fact]
public void CreateOrder_BadExample()
{
    // Verifying that _items field is populated, or _validator was called
}

// 3. Readable failure messages
[Fact]
public void DiscountCalculation_ShouldBeCorrect()
{
    var result = new PricingEngine().CalculateDiscount(2000m, CustomerTier.Gold);

    result.Should().Be(300m,
        because: "Gold customers get 15% discount on orders over 1000: 2000 * 0.15 = 300");
}

// 4. Arrange-Act-Assert with clear separation
[Fact]
public void GetOrderHistory_ForNewCustomer_ReturnsEmptyList()
{
    // Arrange
    var customerId = Guid.NewGuid();
    var service = new OrderService(Mock.Of<IOrderRepository>());

    // Act
    var history = service.GetOrderHistory(customerId);

    // Assert
    history.Should().BeEmpty();
}

// 5. Avoid magic numbers/strings — use named constants
private const decimal GoldDiscountRate = 0.15m;
private const decimal MinOrderForDiscount = 1000m;

[Fact]
public void GoldDiscount_At1001_AppliesRate()
{
    var total = MinOrderForDiscount + 1m;
    var expected = total * GoldDiscountRate;

    var result = new PricingEngine().CalculateDiscount(total, CustomerTier.Gold);

    result.Should().Be(expected);
}
```

---

## Step 1426: Parallel Test Execution

```csharp
// xunit.runner.json — configure parallelism
{
  "parallelizeAssembly": true,
  "parallelizeTestCollections": true,
  "maxParallelThreads": 4
}

// Test collections for shared state management
[CollectionDefinition("Database")]
public class DatabaseCollection : ICollectionFixture<DatabaseFixture> { }

[CollectionDefinition("Redis")]
public class RedisCollection : ICollectionFixture<RedisFixture> { }

// Tests in same collection run sequentially (share a fixture)
[Collection("Database")]
public class OrderDatabaseTests(DatabaseFixture db) { }

[Collection("Database")]
public class CustomerDatabaseTests(DatabaseFixture db) { }

// Tests in different collections run in parallel
[Collection("Redis")]
public class CacheTests(RedisFixture redis) { }

// Tests with NO collection attribute run in parallel with anything
public class PureDomainTests { } // No shared state — runs freely

// Avoiding flaky parallel tests
public class ThreadSafeTestFixture : IAsyncLifetime
{
    private readonly SemaphoreSlim _semaphore = new(1, 1);

    public async Task<T> WithLockAsync<T>(Func<Task<T>> action)
    {
        await _semaphore.WaitAsync();
        try { return await action(); }
        finally { _semaphore.Release(); }
    }

    public Task InitializeAsync() => Task.CompletedTask;
    public Task DisposeAsync() { _semaphore.Dispose(); return Task.CompletedTask; }
}
```

---

## Step 1427: Continuous Testing in CI/CD

```yaml
# .github/workflows/quality-gate.yml
name: Quality Gate

on:
  pull_request:
    branches: [main, develop]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: 9.0.x
      - name: Unit Tests (fast, no external deps)
        run: dotnet test --filter "Category!=Integration&Category!=E2E" --no-build

  integration-tests:
    runs-on: ubuntu-latest
    needs: unit-tests
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: 9.0.x
      - name: Integration Tests (with real DB/Redis via Testcontainers)
        run: dotnet test --filter "Category=Integration" --no-build

  mutation-tests:
    runs-on: ubuntu-latest
    needs: unit-tests
    if: github.event_name == 'pull_request'
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: 9.0.x
      - name: Install Stryker
        run: dotnet tool install --global dotnet-stryker
      - name: Run Mutation Tests
        run: dotnet stryker --since:main
      - name: Upload Mutation Report
        uses: actions/upload-artifact@v4
        with:
          name: mutation-report
          path: StrykerOutput/

  architecture-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: 9.0.x
      - name: Architecture Tests
        run: dotnet test --filter "FullyQualifiedName~ArchitectureTests" --no-build
```

---

## Step 1428: Testing Caching Layer

```csharp
public class CachedOrderRepositoryTests
{
    private readonly Mock<IOrderRepository> _innerRepoMock = new();
    private readonly IDistributedCache _cache;
    private readonly CachedOrderRepository _sut;

    public CachedOrderRepositoryTests()
    {
        _cache = new MemoryDistributedCache(Options.Create(new MemoryDistributedCacheOptions()));
        _sut = new CachedOrderRepository(_innerRepoMock.Object, _cache);
    }

    [Fact]
    public async Task GetById_FirstCall_HitsDatabase()
    {
        var orderId = Guid.NewGuid();
        var order = OrderBuilder.AnOrder().WithId(orderId).Build();

        _innerRepoMock.Setup(x => x.GetByIdAsync(orderId, It.IsAny<CancellationToken>()))
                      .ReturnsAsync(order);

        await _sut.GetByIdAsync(orderId);

        _innerRepoMock.Verify(x => x.GetByIdAsync(orderId, It.IsAny<CancellationToken>()), Times.Once);
    }

    [Fact]
    public async Task GetById_SecondCall_ReturnsCachedValue()
    {
        var orderId = Guid.NewGuid();
        var order = OrderBuilder.AnOrder().WithId(orderId).Build();

        _innerRepoMock.Setup(x => x.GetByIdAsync(orderId, It.IsAny<CancellationToken>()))
                      .ReturnsAsync(order);

        await _sut.GetByIdAsync(orderId);  // First call — hits DB
        await _sut.GetByIdAsync(orderId);  // Second call — from cache

        _innerRepoMock.Verify(x => x.GetByIdAsync(orderId, It.IsAny<CancellationToken>()), Times.Once);
    }

    [Fact]
    public async Task UpdateOrder_InvalidatesCache()
    {
        var orderId = Guid.NewGuid();
        var order = OrderBuilder.AnOrder().WithId(orderId).Build();

        _innerRepoMock.Setup(x => x.GetByIdAsync(orderId, It.IsAny<CancellationToken>()))
                      .ReturnsAsync(order);

        await _sut.GetByIdAsync(orderId);  // Populates cache
        await _sut.UpdateStatusAsync(orderId, OrderStatus.Confirmed);  // Should invalidate
        await _sut.GetByIdAsync(orderId);  // Should hit DB again

        _innerRepoMock.Verify(x => x.GetByIdAsync(orderId, It.IsAny<CancellationToken>()), Times.Exactly(2));
    }
}
```

---

## Step 1429: Summary — Testing Strategy Matrix

```
╔══════════════════════════════════════════════════════════════════════╗
║               C# Advanced Testing Strategy Matrix                   ║
╠══════════════════╦═══════════════╦══════════════╦═════════════════╣
║ Test Type        ║ Speed         ║ Coverage     ║ When to Use     ║
╠══════════════════╬═══════════════╬══════════════╬═════════════════╣
║ Unit Tests       ║ <10ms each    ║ 70% of suite ║ Business rules  ║
║ (xUnit+Moq)      ║               ║              ║ Edge cases      ║
╠══════════════════╬═══════════════╬══════════════╬═════════════════╣
║ Property-Based   ║ ~100ms each   ║ 5% of suite  ║ Math invariants ║
║ (FsCheck)        ║               ║              ║ Codec round-trips║
╠══════════════════╬═══════════════╬══════════════╬═════════════════╣
║ Integration      ║ 1-10s each    ║ 20% of suite ║ DB queries      ║
║ (Testcontainers) ║               ║              ║ HTTP flows      ║
╠══════════════════╬═══════════════╬══════════════╬═════════════════╣
║ Contract         ║ 5-30s each    ║ 3% of suite  ║ Service boundaries║
║ (Pact)           ║               ║              ║ API consumers   ║
╠══════════════════╬═══════════════╬══════════════╬═════════════════╣
║ Architecture     ║ 1-5s per run  ║ Always       ║ Layer rules     ║
║ (NetArchTest)    ║               ║              ║ Dependency rules ║
╠══════════════════╬═══════════════╬══════════════╬═════════════════╣
║ Mutation         ║ 5-30 minutes  ║ Weekly/PR    ║ Test quality    ║
║ (Stryker.NET)    ║               ║              ║ Find test gaps  ║
╠══════════════════╬═══════════════╬══════════════╬═════════════════╣
║ Load Tests       ║ Minutes       ║ Pre-release  ║ SLO validation  ║
║ (NBomber)        ║               ║              ║ Capacity planning║
╠══════════════════╬═══════════════╬══════════════╬═════════════════╣
║ Snapshot         ║ <100ms each   ║ 2% of suite  ║ Serialization   ║
║ (Verify)         ║               ║              ║ API responses   ║
╚══════════════════╩═══════════════╩══════════════╩═════════════════╝

Quality Gates:
  ✅ Line coverage    ≥ 80%
  ✅ Branch coverage  ≥ 75%
  ✅ Mutation score   ≥ 70%
  ✅ P99 latency      < 500ms (load tests)
  ✅ Error rate       < 1% (load tests)
  ✅ Architecture     100% pass
  ✅ Contract tests   100% pass
```

---

*Part 50 complete — Steps 1401-1429. Topics covered: TDD Red-Green-Refactor, BDD with Reqnroll/SpecFlow, Property-Based Testing (FsCheck), Mutation Testing (Stryker.NET), Architecture Tests (NetArchTest), Testcontainers integration tests, Respawn database reset, WebApplicationFactory API tests, Pact contract testing, NBomber load testing, Test Data Builders, Snapshot Testing (Verify), MediatR handler tests, background service tests, SignalR hub integration tests, Minimal API tests, EF Core SQLite tests, Polly resilience tests, domain event tests, parallel test execution, CI/CD quality gate pipeline.*
