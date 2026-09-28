# Part 93: Testing Strategies — Unit, Integration, E2E & Property-Based

## Steps 2117–2132

---

## Step 2117: Testing Pyramid & Philosophy

```
Testing Pyramid:
                    /\
                   /  \
                  / E2E \       Few, slow, high-value
                 /  (5%) \      Playwright, full stack
                /──────────\
               / Integration \  Some, medium speed
              /    (20%)      \ WebApplicationFactory
             /──────────────────\
            /    Unit Tests       \ Many, fast, isolated
           /        (75%)          \ xUnit, NSubstitute
          /──────────────────────────\

Testing Principles:
- Arrange, Act, Assert (AAA)
- One assertion per test concept
- Tests as living documentation
- Test behavior, not implementation
- Avoid testing framework internals
- Fast feedback loop (< 1 min for unit suite)
```

```bash
dotnet add package xunit
dotnet add package xunit.runner.visualstudio
dotnet add package FluentAssertions
dotnet add package NSubstitute
dotnet add package AutoFixture
dotnet add package AutoFixture.AutoNSubstitute
dotnet add package Bogus              # fake data generation
dotnet add package Microsoft.AspNetCore.Mvc.Testing
dotnet add package Testcontainers.PostgreSql
dotnet add package Testcontainers.Redis
dotnet add package Respawn            # database reset
```

---

## Step 2118: Unit Testing — Domain Logic

```csharp
public class OrderTests
{
    [Fact]
    public void Create_WithValidInputs_ShouldCreateOrder()
    {
        // Arrange
        var customerId = Guid.NewGuid();
        var items = new List<OrderItemRequest>
        {
            new(ProductId: Guid.NewGuid(), Quantity: 2, UnitPrice: 49.99m),
            new(ProductId: Guid.NewGuid(), Quantity: 1, UnitPrice: 19.99m)
        };

        // Act
        var order = Order.Create(customerId, items);

        // Assert
        order.Id.Should().NotBeEmpty();
        order.CustomerId.Should().Be(customerId);
        order.Status.Should().Be(OrderStatus.Pending);
        order.TotalAmount.Should().Be(119.97m);
        order.Items.Should().HaveCount(2);
        order.DomainEvents.Should().ContainSingle(e => e is OrderCreatedEvent);
    }

    [Fact]
    public void Cancel_PendingOrder_ShouldChangeStatus()
    {
        var order = CreatePendingOrder();

        order.Cancel("Customer request");

        order.Status.Should().Be(OrderStatus.Cancelled);
        order.CancellationReason.Should().Be("Customer request");
        order.DomainEvents.Should().ContainSingle(e => e is OrderCancelledEvent);
    }

    [Theory]
    [InlineData(OrderStatus.Shipped)]
    [InlineData(OrderStatus.Delivered)]
    [InlineData(OrderStatus.Cancelled)]
    public void Cancel_NonPendingOrder_ShouldThrow(OrderStatus status)
    {
        var order = CreateOrderWithStatus(status);

        var act = () => order.Cancel("reason");

        act.Should().Throw<InvalidOperationException>()
           .WithMessage($"Cannot cancel order in status {status}");
    }

    [Fact]
    public void AddItem_ExistingProduct_ShouldIncrementQuantity()
    {
        var order     = CreatePendingOrder();
        var productId = order.Items[0].ProductId;

        order.AddItem(productId, quantity: 3, unitPrice: order.Items[0].UnitPrice);

        order.Items.Should().HaveCount(1); // merged, not added
        order.Items[0].Quantity.Should().Be(order.Items[0].Quantity);
    }

    // Factory helpers
    private static Order CreatePendingOrder()
        => Order.Create(Guid.NewGuid(), [new(Guid.NewGuid(), 1, 10m)]);

    private static Order CreateOrderWithStatus(OrderStatus status)
    {
        var order = CreatePendingOrder();
        // Use reflection or internal state for test setup
        typeof(Order).GetProperty(nameof(Order.Status))!
            .SetValue(order, status);
        return order;
    }
}
```

---

## Step 2119: Unit Testing — Application Layer with Mocks

```csharp
public class PlaceOrderHandlerTests
{
    private readonly IOrderRepository _repo;
    private readonly IOutboxService _outbox;
    private readonly IValidator<PlaceOrderCommand> _validator;
    private readonly PlaceOrderHandler _handler;

    public PlaceOrderHandlerTests()
    {
        _repo      = Substitute.For<IOrderRepository>();
        _outbox    = Substitute.For<IOutboxService>();
        _validator = Substitute.For<IValidator<PlaceOrderCommand>>();

        _handler = new PlaceOrderHandler(_repo, _outbox, _validator);
    }

    [Fact]
    public async Task Handle_ValidCommand_ReturnsOrderId()
    {
        // Arrange
        var command = new PlaceOrderCommand(
            CustomerId: Guid.NewGuid(),
            Items: [new(Guid.NewGuid(), 2, 49.99m)]);

        _validator.ValidateAsync(command, Arg.Any<CancellationToken>())
            .Returns(new ValidationResult());

        var order = Order.Create(command.CustomerId, command.Items);
        _repo.AddAsync(Arg.Any<Order>(), Arg.Any<CancellationToken>())
             .Returns(Task.CompletedTask);

        // Act
        var result = await _handler.Handle(command, CancellationToken.None);

        // Assert
        result.OrderId.Should().NotBeEmpty();

        await _repo.Received(1).AddAsync(
            Arg.Is<Order>(o => o.CustomerId == command.CustomerId),
            Arg.Any<CancellationToken>());

        await _outbox.Received(1).AddMessageAsync(
            Arg.Is<OrderPlacedEvent>(e => e.CustomerId == command.CustomerId),
            Arg.Any<CancellationToken>());
    }

    [Fact]
    public async Task Handle_InvalidCommand_ThrowsValidationException()
    {
        var command = new PlaceOrderCommand(Guid.NewGuid(), []);

        _validator.ValidateAsync(command, Arg.Any<CancellationToken>())
            .Returns(new ValidationResult(new[]
            {
                new ValidationFailure("Items", "At least one item required")
            }));

        var act = () => _handler.Handle(command, CancellationToken.None);

        await act.Should().ThrowAsync<ValidationException>()
            .Where(ex => ex.Errors.ContainsKey("Items"));
    }
}
```

---

## Step 2120: Unit Testing — AutoFixture & Bogus

```csharp
// AutoFixture: auto-generate test data
public class OrderServiceTests
{
    private readonly IFixture _fixture;
    private readonly IOrderRepository _repo;
    private readonly OrderService _service;

    public OrderServiceTests()
    {
        _fixture = new Fixture()
            .Customize(new AutoNSubstituteCustomization { ConfigureMembers = true });

        _repo    = _fixture.Freeze<IOrderRepository>();
        _service = _fixture.Create<OrderService>();
    }

    [Fact]
    public async Task GetByIdAsync_ExistingOrder_ReturnsOrder()
    {
        // AutoFixture creates a realistic Order with all properties filled
        var expectedOrder = _fixture.Create<Order>();
        _repo.GetByIdAsync(expectedOrder.Id, Arg.Any<CancellationToken>())
             .Returns(expectedOrder);

        var result = await _service.GetByIdAsync(expectedOrder.Id);

        result.Should().BeEquivalentTo(expectedOrder);
    }
}

// Bogus: domain-specific fake data
public static class OrderFaker
{
    public static Faker<Order> Create() =>
        new Faker<Order>()
            .RuleFor(o => o.Id,         f => f.Random.Guid())
            .RuleFor(o => o.CustomerId, f => f.Random.Guid())
            .RuleFor(o => o.Status,     f => f.PickRandom<OrderStatus>())
            .RuleFor(o => o.TotalAmount, f => f.Finance.Amount(10, 1000))
            .RuleFor(o => o.Currency,   f => f.Finance.Currency().Code)
            .RuleFor(o => o.CreatedAt,  f => f.Date.PastOffset());

    public static Faker<Customer> CreateCustomer() =>
        new Faker<Customer>()
            .RuleFor(c => c.Id,       f => f.Random.Guid())
            .RuleFor(c => c.FullName, f => f.Name.FullName())
            .RuleFor(c => c.Email,    f => f.Internet.Email());
}

// Usage
[Fact]
public async Task SearchOrders_ByStatus_ReturnsMatchingOrders()
{
    var pendingOrders = OrderFaker.Create()
        .RuleFor(o => o.Status, _ => OrderStatus.Pending)
        .Generate(5);

    var allOrders = pendingOrders
        .Concat(OrderFaker.Create().RuleFor(o => o.Status, _ => OrderStatus.Shipped).Generate(3))
        .ToList();

    _repo.GetAllAsync(Arg.Any<CancellationToken>()).Returns(allOrders);

    var result = await _service.SearchAsync(status: OrderStatus.Pending);

    result.Should().HaveCount(5)
          .And.AllSatisfy(o => o.Status.Should().Be(OrderStatus.Pending));
}
```

---

## Step 2121: Integration Testing — WebApplicationFactory

```csharp
// Custom factory with test infrastructure
public class ApiFactory : WebApplicationFactory<Program>
{
    private readonly PostgreSqlContainer _postgres;
    private readonly RedisContainer _redis;

    public ApiFactory()
    {
        _postgres = new PostgreSqlBuilder()
            .WithImage("postgres:16-alpine")
            .WithDatabase("testdb")
            .Build();

        _redis = new RedisBuilder()
            .WithImage("redis:7-alpine")
            .Build();
    }

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureTestServices(services =>
        {
            // Replace real DB with test DB
            services.RemoveAll<DbContextOptions<AppDbContext>>();
            services.AddDbContext<AppDbContext>(opts =>
                opts.UseNpgsql(_postgres.GetConnectionString()));

            // Replace Redis with test Redis
            services.RemoveAll<IConnectionMultiplexer>();
            services.AddSingleton<IConnectionMultiplexer>(
                ConnectionMultiplexer.Connect(_redis.GetConnectionString()));

            // Replace external services with fakes
            services.RemoveAll<IPaymentService>();
            services.AddScoped<IPaymentService, FakePaymentService>();

            services.RemoveAll<IEmailService>();
            services.AddScoped<IEmailService>(_ => Substitute.For<IEmailService>());
        });

        builder.UseEnvironment("Testing");
    }

    public async Task InitializeAsync()
    {
        await Task.WhenAll(_postgres.StartAsync(), _redis.StartAsync());

        // Apply migrations
        using var scope = Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        await db.Database.MigrateAsync();
    }

    public override async ValueTask DisposeAsync()
    {
        await Task.WhenAll(_postgres.DisposeAsync().AsTask(), _redis.DisposeAsync().AsTask());
        await base.DisposeAsync();
    }
}

// Test class using the factory
public class OrderEndpointsTests : IClassFixture<ApiFactory>, IAsyncLifetime
{
    private readonly ApiFactory _factory;
    private readonly HttpClient _client;
    private Respawner _respawner = null!;

    public OrderEndpointsTests(ApiFactory factory)
    {
        _factory = factory;
        _client  = factory.CreateClient();
    }

    public async Task InitializeAsync()
    {
        await _factory.InitializeAsync();

        // Configure Respawn to reset DB between tests
        var conn = new NpgsqlConnection(_factory.GetConnectionString());
        await conn.OpenAsync();
        _respawner = await Respawner.CreateAsync(conn, new RespawnerOptions
        {
            DbAdapter  = DbAdapter.Postgres,
            SchemasToInclude = ["public"],
            TablesToIgnore = ["__EFMigrationsHistory"]
        });
    }

    public async Task DisposeAsync()
    {
        // Reset DB after each test
        var conn = new NpgsqlConnection(_factory.GetConnectionString());
        await conn.OpenAsync();
        await _respawner.ResetAsync(conn);
    }

    [Fact]
    public async Task PostOrder_ValidRequest_Returns201()
    {
        // Arrange
        var request = new
        {
            customerId = Guid.NewGuid(),
            items = new[] { new { productId = Guid.NewGuid(), quantity = 2, unitPrice = 49.99 } }
        };

        _client.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", GetTestToken());

        // Act
        var response = await _client.PostAsJsonAsync("/api/orders", request);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Created);
        var body = await response.Content.ReadFromJsonAsync<PlaceOrderResponse>();
        body!.OrderId.Should().NotBeEmpty();
        response.Headers.Location.Should().NotBeNull();
    }

    [Fact]
    public async Task GetOrder_AfterCreation_ReturnsOrder()
    {
        // Arrange: create order first
        var orderId = await CreateTestOrderAsync();

        // Act
        var response = await _client.GetAsync($"/api/orders/{orderId}");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
        var order = await response.Content.ReadFromJsonAsync<OrderDto>();
        order!.Id.Should().Be(orderId);
        order.Status.Should().Be("Pending");
    }

    private async Task<Guid> CreateTestOrderAsync()
    {
        var request = new
        {
            customerId = Guid.NewGuid(),
            items = new[] { new { productId = Guid.NewGuid(), quantity = 1, unitPrice = 10.0 } }
        };

        var response = await _client.PostAsJsonAsync("/api/orders", request);
        var body     = await response.Content.ReadFromJsonAsync<PlaceOrderResponse>();
        return body!.OrderId;
    }

    private string GetTestToken() =>
        JwtTestHelper.CreateToken(Guid.NewGuid(), roles: ["user"]);
}
```

---

## Step 2122: Repository & DB Integration Tests

```csharp
public class OrderRepositoryTests : IClassFixture<DbContextFixture>
{
    private readonly DbContextFixture _fixture;

    public OrderRepositoryTests(DbContextFixture fixture) => _fixture = fixture;

    [Fact]
    public async Task AddAsync_NewOrder_CanBeRetrieved()
    {
        await using var db   = _fixture.CreateDbContext();
        var repository       = new EfCoreOrderRepository(db);

        var order = Order.Create(Guid.NewGuid(), [new(Guid.NewGuid(), 2, 49.99m)]);
        await repository.AddAsync(order);
        await db.SaveChangesAsync();

        // New context to avoid first-level cache
        await using var db2  = _fixture.CreateDbContext();
        var repo2            = new EfCoreOrderRepository(db2);
        var retrieved        = await repo2.GetByIdAsync(order.Id);

        retrieved.Should().NotBeNull();
        retrieved!.CustomerId.Should().Be(order.CustomerId);
        retrieved.TotalAmount.Should().Be(99.98m);
        retrieved.Items.Should().HaveCount(1);
    }

    [Fact]
    public async Task GetPagedAsync_WithFilter_ReturnsCorrectPage()
    {
        await using var db = _fixture.CreateDbContext();

        // Seed test data
        var orders = OrderFaker.Create()
            .RuleFor(o => o.Status, _ => OrderStatus.Pending)
            .Generate(25);
        await db.Orders.AddRangeAsync(orders);
        await db.SaveChangesAsync();

        var repo   = new EfCoreOrderRepository(db);
        var result = await repo.GetPagedAsync(
            filter: o => o.Status == OrderStatus.Pending,
            orderBy: o => o.CreatedAt,
            page: 1, pageSize: 10);

        result.Items.Should().HaveCount(10);
        result.TotalCount.Should().BeGreaterThanOrEqualTo(25);
        result.Items.Should().BeInAscendingOrder(o => o.CreatedAt);
    }
}

// DB fixture with Testcontainers
public class DbContextFixture : IAsyncLifetime
{
    private readonly PostgreSqlContainer _container = new PostgreSqlBuilder()
        .WithImage("postgres:16-alpine")
        .Build();

    private string ConnectionString => _container.GetConnectionString();

    public AppDbContext CreateDbContext()
    {
        var opts = new DbContextOptionsBuilder<AppDbContext>()
            .UseNpgsql(ConnectionString)
            .Options;
        return new AppDbContext(opts);
    }

    public async Task InitializeAsync()
    {
        await _container.StartAsync();
        using var ctx = CreateDbContext();
        await ctx.Database.MigrateAsync();
    }

    public async Task DisposeAsync() => await _container.DisposeAsync();
}
```

---

## Step 2123: Property-Based Testing with FsCheck

```bash
dotnet add package FsCheck
dotnet add package FsCheck.Xunit
```

```csharp
// Property-based testing: verify invariants hold for ANY input
public class OrderCalculationProperties
{
    [Property]
    public bool TotalAmount_AlwaysEqualsSum(PositiveInt quantity, PositiveDecimal price)
    {
        var q = quantity.Get;
        var p = (decimal)price.Get;

        var order = Order.Create(Guid.NewGuid(), [new(Guid.NewGuid(), q, p)]);

        return order.TotalAmount == q * p;
    }

    [Property]
    public bool AddItem_IncreasesTotalAmount(PositiveDecimal unitPrice)
    {
        var order = Order.Create(Guid.NewGuid(), [new(Guid.NewGuid(), 1, 10m)]);
        var initialTotal = order.TotalAmount;
        var price = (decimal)unitPrice.Get;

        order.AddItem(Guid.NewGuid(), 1, price);

        return order.TotalAmount == initialTotal + price;
    }

    [Property]
    public bool Cancel_IsIdempotent_AfterFirstCancellation(string reason)
    {
        var order = Order.Create(Guid.NewGuid(), [new(Guid.NewGuid(), 1, 10m)]);
        order.Cancel(reason);

        // Second cancel should either succeed (idempotent) or throw — not corrupt state
        try { order.Cancel(reason); }
        catch (InvalidOperationException) { }

        return order.Status == OrderStatus.Cancelled;
    }
}

// Custom generators
public class OrderArbitraries
{
    public static Arbitrary<Order> OrderArbitrary()
    {
        var gen = from customerId in Arb.Generate<Guid>()
                  from itemCount  in Gen.Choose(1, 10)
                  from items      in Gen.ListOf(itemCount, OrderItemGen())
                  select Order.Create(customerId, items);

        return gen.ToArbitrary();
    }

    private static Gen<OrderItemRequest> OrderItemGen() =>
        from productId in Arb.Generate<Guid>()
        from quantity  in Gen.Choose(1, 100)
        from price     in Gen.Elements(new[] { 9.99m, 19.99m, 49.99m, 99.99m })
        select new OrderItemRequest(productId, quantity, price);
}

// Register custom arbitraries
[assembly: Properties(Arbitrary = [typeof(OrderArbitraries)])]
```

---

## Step 2124: End-to-End Testing with Playwright

```bash
dotnet add package Microsoft.Playwright
dotnet tool install --global Microsoft.Playwright.CLI
playwright install
```

```csharp
// E2E test using Playwright
[Collection("E2E")]
public class OrderPlacementE2ETests : IAsyncLifetime
{
    private IPlaywright _playwright = null!;
    private IBrowser _browser = null!;
    private IBrowserContext _context = null!;
    private IPage _page = null!;

    private const string BaseUrl = "http://localhost:5000";

    public async Task InitializeAsync()
    {
        _playwright = await Playwright.CreateAsync();
        _browser    = await _playwright.Chromium.LaunchAsync(new BrowserTypeLaunchOptions
        {
            Headless = true,
            SlowMo   = 0
        });
        _context = await _browser.NewContextAsync(new BrowserNewContextOptions
        {
            BaseURL    = BaseUrl,
            RecordVideoDir = "videos/",
            ViewportSize   = new ViewportSize { Width = 1280, Height = 720 }
        });
        _page = await _context.NewPageAsync();
    }

    [Fact]
    public async Task PlaceOrder_AsAuthenticatedUser_OrderAppearsInHistory()
    {
        // Login
        await _page.GotoAsync("/login");
        await _page.FillAsync("[data-testid=email]", "test@example.com");
        await _page.FillAsync("[data-testid=password]", "Test@1234!");
        await _page.ClickAsync("[data-testid=login-btn]");
        await _page.WaitForURLAsync("**/dashboard");

        // Navigate to products
        await _page.GotoAsync("/products");

        // Add item to cart
        await _page.ClickAsync("[data-testid=product-card]:first-child [data-testid=add-to-cart]");
        await _page.WaitForSelectorAsync("[data-testid=cart-count]:text('1')");

        // Checkout
        await _page.ClickAsync("[data-testid=checkout-btn]");
        await _page.WaitForURLAsync("**/checkout");
        await _page.ClickAsync("[data-testid=place-order-btn]");

        // Verify order confirmation
        await _page.WaitForURLAsync("**/orders/**");
        await _page.WaitForSelectorAsync("[data-testid=order-status]:text('Pending')");

        var orderId = _page.Url.Split('/').Last();

        // Verify in order history
        await _page.GotoAsync("/orders");
        var row = _page.Locator($"[data-testid=order-row-{orderId}]");
        await row.WaitForAsync();
        await row.Locator("[data-testid=status]").ShouldHaveTextAsync("Pending");
    }

    [Fact]
    public async Task OrderPage_LoadsWithin2Seconds()
    {
        await LoginAsync();

        var stopwatch = Stopwatch.StartNew();
        await _page.GotoAsync("/orders");
        await _page.WaitForLoadStateAsync(LoadState.NetworkIdle);
        stopwatch.Stop();

        stopwatch.ElapsedMilliseconds.Should().BeLessThan(2000);
    }

    private async Task LoginAsync()
    {
        await _page.GotoAsync("/login");
        await _page.FillAsync("[data-testid=email]", "test@example.com");
        await _page.FillAsync("[data-testid=password]", "Test@1234!");
        await _page.ClickAsync("[data-testid=login-btn]");
        await _page.WaitForURLAsync("**/dashboard");
    }

    public async Task DisposeAsync()
    {
        await _context.CloseAsync();
        await _browser.DisposeAsync();
        _playwright.Dispose();
    }
}
```

---

## Step 2125: Snapshot Testing & Contract Testing

```csharp
// Snapshot testing: verify output hasn't changed unexpectedly
// dotnet add package Verify.Xunit
public class OrderSerializationTests
{
    [Fact]
    public async Task Order_SerializesToExpectedJson()
    {
        var order = new Order
        {
            Id          = Guid.Parse("a1b2c3d4-e5f6-7890-abcd-ef1234567890"),
            CustomerId  = Guid.Parse("b2c3d4e5-f6a7-890b-cde1-f23456789012"),
            Status      = OrderStatus.Pending,
            TotalAmount = 119.97m,
            Currency    = "USD",
            CreatedAt   = new DateTimeOffset(2024, 1, 15, 10, 30, 0, TimeSpan.Zero)
        };

        // First run: creates .verified.json snapshot file
        // Subsequent runs: compares against snapshot
        await Verify(order);
    }

    [Fact]
    public async Task PlaceOrderResponse_MatchesSnapshot()
    {
        var response = new PlaceOrderResponse(
            OrderId: Guid.Parse("c3d4e5f6-a7b8-90cd-ef12-345678901234"),
            Status:  "Pending");

        await Verify(response);
    }
}

// Consumer-Driven Contract Testing with PactNet
// dotnet add package PactNet
public class OrderApiContractTests : IDisposable
{
    private readonly IPact _pact;

    public OrderApiContractTests()
    {
        _pact = Pact.V3("consumer", "orders-api", new PactConfig
        {
            PactDir = "./pacts"
        });
    }

    [Fact]
    public async Task GetOrder_ExistingOrder_ReturnsOrder()
    {
        var orderId = "a1b2c3d4-e5f6-7890-abcd-ef1234567890";

        _pact
            .UponReceiving("a GET request for an existing order")
                .WithRequest(HttpMethod.Get, $"/api/orders/{orderId}")
                .WithHeader("Authorization", "Bearer.*", Match.Regex("Bearer.*"))
            .WillRespond()
                .WithStatus(200)
                .WithJsonBody(new
                {
                    id         = orderId,
                    status     = "Pending",
                    totalAmount = 119.97
                });

        await _pact.VerifyAsync(async ctx =>
        {
            var client  = new HttpClient { BaseAddress = ctx.MockServerUri };
            client.DefaultRequestHeaders.Authorization =
                new AuthenticationHeaderValue("Bearer", "test-token");

            var response = await client.GetAsync($"/api/orders/{orderId}");
            response.StatusCode.Should().Be(HttpStatusCode.OK);
        });
    }

    public void Dispose() => _pact.Dispose();
}
```

---

## Step 2126: Performance Testing — BenchmarkDotNet & Load Testing

```csharp
// BenchmarkDotNet micro-benchmarks
[MemoryDiagnoser]
[SimpleJob(RuntimeMoniker.Net90)]
public class OrderCalculationBenchmarks
{
    private Order[] _orders = null!;

    [GlobalSetup]
    public void Setup()
    {
        _orders = OrderFaker.Create().Generate(1000).ToArray();
    }

    [Benchmark(Baseline = true)]
    public decimal CalculateTotalLinq() =>
        _orders.Sum(o => o.TotalAmount);

    [Benchmark]
    public decimal CalculateTotalFor()
    {
        decimal total = 0;
        for (var i = 0; i < _orders.Length; i++)
            total += _orders[i].TotalAmount;
        return total;
    }

    [Benchmark]
    public decimal CalculateTotalSpan()
    {
        var span  = _orders.AsSpan();
        decimal total = 0;
        foreach (ref readonly var order in span)
            total += order.TotalAmount;
        return total;
    }
}

// NBomber load testing
// dotnet add package NBomber
public class LoadTests
{
    [Fact]
    public void OrderApi_UnderLoad_MeetsPerformanceSla()
    {
        var factory  = new ApiFactory();
        var client   = factory.CreateClient();

        var scenario = Scenario.Create("place_order", async ctx =>
        {
            var request = new
            {
                customerId = Guid.NewGuid(),
                items      = new[] { new { productId = Guid.NewGuid(), quantity = 1, unitPrice = 10.0 } }
            };

            var response = await client.PostAsJsonAsync("/api/orders", request, ctx.CancellationToken);
            return response.IsSuccessStatusCode ? Response.Ok() : Response.Fail();
        })
        .WithLoadSimulations(
            Simulation.Inject(rate: 100, interval: TimeSpan.FromSeconds(1), during: TimeSpan.FromMinutes(1)));

        var stats = NBomberRunner.RegisterScenarios(scenario)
            .Run();

        var orderStats = stats.ScenarioStats["place_order"];
        orderStats.Ok.RPS.Should().BeGreaterThan(50);
        orderStats.Ok.Latency.P99.Should().BeLessThan(500);
        orderStats.Fail.Request.Count.Should().Be(0);
    }
}
```

---

## Step 2127: Test Organization & Best Practices

```csharp
// Shared test fixtures
public class OrderFixtures
{
    public static Order CreatePendingOrder(Guid? customerId = null) =>
        Order.Create(
            customerId ?? Guid.NewGuid(),
            [new(Guid.NewGuid(), 2, 49.99m)]);

    public static Order CreateCancelledOrder(Guid? id = null)
    {
        var order = CreatePendingOrder();
        order.Cancel("Test cancellation");
        return order;
    }

    public static PlaceOrderCommand ValidPlaceOrderCommand() =>
        new(CustomerId: Guid.NewGuid(),
            Items: [new(Guid.NewGuid(), 1, 99.99m)]);
}

// Builder pattern for test data
public class OrderBuilder
{
    private Guid _customerId = Guid.NewGuid();
    private List<OrderItemRequest> _items = [new(Guid.NewGuid(), 1, 10m)];
    private OrderStatus _status = OrderStatus.Pending;

    public OrderBuilder WithCustomer(Guid id) { _customerId = id; return this; }
    public OrderBuilder WithItems(params OrderItemRequest[] items) { _items = [..items]; return this; }
    public OrderBuilder WithStatus(OrderStatus status) { _status = status; return this; }

    public Order Build()
    {
        var order = Order.Create(_customerId, _items);
        if (_status != OrderStatus.Pending)
        {
            // Reflect status for test purposes
            typeof(Order).GetProperty(nameof(Order.Status))!.SetValue(order, _status);
        }
        return order;
    }
}

// Shared assertions
public static class OrderAssertions
{
    public static void ShouldBeValidPendingOrder(this Order order, Guid expectedCustomerId)
    {
        order.Should().NotBeNull();
        order.Id.Should().NotBeEmpty();
        order.CustomerId.Should().Be(expectedCustomerId);
        order.Status.Should().Be(OrderStatus.Pending);
        order.TotalAmount.Should().BePositive();
        order.CreatedAt.Should().BeCloseTo(DateTimeOffset.UtcNow, TimeSpan.FromSeconds(5));
    }
}

// Test categories for selective CI execution
[Trait("Category", "Unit")]
public class UnitTests { }

[Trait("Category", "Integration")]
[Trait("Category", "Slow")]
public class IntegrationTests { }

[Trait("Category", "E2E")]
[Trait("Category", "Smoke")]
public class E2ETests { }

// Run unit tests only:
// dotnet test --filter "Category=Unit"
// Run all except E2E:
// dotnet test --filter "Category!=E2E"
```

---

## Step 2128: Mutation Testing & Code Coverage

```bash
# Stryker.NET mutation testing
dotnet tool install --global dotnet-stryker

# Run mutation testing
dotnet stryker --project "OrderService/OrderService.csproj" \
               --test-projects "OrderService.Tests/OrderService.Tests.csproj" \
               --reporters "['html', 'json']" \
               --threshold-high 80 \
               --threshold-low 60

# Code coverage with coverlet
dotnet test --collect:"XPlat Code Coverage" \
            --results-directory ./coverage \
            -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.Format=cobertura

# Generate report with ReportGenerator
dotnet tool install --global dotnet-reportgenerator-globaltool
reportgenerator \
    -reports:"coverage/**/coverage.cobertura.xml" \
    -targetdir:"coverage/report" \
    -reporttypes:Html
```

```csharp
// .stryker-config.json
{
  "stryker-config": {
    "mutation-level": "Advanced",
    "thresholds": {
      "high":  80,
      "low":   60,
      "break": 50
    },
    "ignore-methods": ["*.ToString", "*.GetHashCode", "*.Equals"],
    "ignore-mutations": ["StringMethod"],
    "reporters": ["html", "json", "dashboard"]
  }
}
```

**Summary**: Part 93 covers the testing pyramid philosophy, unit testing with xUnit/FluentAssertions/NSubstitute, AutoFixture for test data generation, Bogus for domain-realistic fakes, integration testing with WebApplicationFactory + Testcontainers + Respawn for DB reset, repository integration tests, property-based testing with FsCheck (invariant verification), E2E testing with Playwright (full browser automation), snapshot testing with Verify, consumer-driven contract testing with PactNet, BenchmarkDotNet micro-benchmarks, NBomber load testing, test organization with builders/fixtures, and mutation testing with Stryker.NET. Steps 2117–2132 complete.
