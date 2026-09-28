# Part 73: Advanced Testing Strategies — Contract Testing, Mutation Testing & Load Testing

## Steps 1797–1812 | World-Class Quality Assurance

---

## Step 1797: The Testing Pyramid Revisited

```
                    ┌─────────────────────┐
                    │    E2E / Manual     │ ← Few, slow, expensive
                    │    (Playwright)     │
               ┌────┴─────────────────────┴────┐
               │   Integration / Contract Tests │ ← Moderate, DB/service
               │   (WebApplicationFactory)      │
          ┌────┴─────────────────────────────────┴────┐
          │          Unit Tests                        │ ← Many, fast, cheap
          │   (xUnit, NUnit, Moq, AutoFixture)         │
     ┌────┴───────────────────────────────────────────┴─────┐
     │              Architecture Tests                       │
     │   (NetArchTest, ArchUnitNET)  ← enforce layer rules  │
     └───────────────────────────────────────────────────────┘

  Advanced:
  • Contract Tests (Pact)      — API consumer/provider contracts
  • Mutation Tests (Stryker)   — verify tests actually catch bugs
  • Property Tests (FsCheck)   — generative inputs, find edge cases
  • Load Tests (k6, NBomber)   — performance under traffic
  • Chaos Tests (Simmy)        — resilience under fault injection
```

---

## Step 1798: FluentAssertions — Readable Test Assertions

```csharp
// Install: dotnet add package FluentAssertions
// Install: dotnet add package FluentAssertions.Web (HTTP responses)

public class OrderTests
{
    [Fact]
    public void Submit_Should_Change_Status_To_Submitted()
    {
        // Arrange
        var address = Address.Create("123 Main St", "Springfield", "62701", "US");
        var order = Order.Create(Guid.NewGuid(), address);
        var unitPrice = Money.Of(29.99m, "USD");
        order.AddItem(Guid.NewGuid(), "Widget Pro", unitPrice, 2);

        // Act
        order.Submit();

        // Assert — FluentAssertions
        order.Status.Should().Be(OrderStatus.Submitted);
        order.Total.Amount.Should().Be(59.98m);
        order.Total.Currency.Should().Be("USD");
        order.DomainEvents.Should().ContainSingle()
            .Which.Should().BeOfType<OrderSubmittedEvent>()
            .Which.Total.Should().Be(Money.Of(59.98m, "USD"));
    }

    [Fact]
    public void Submit_EmptyOrder_Should_Throw_DomainException()
    {
        var address = Address.Create("123 Main St", "Springfield", "62701", "US");
        var order = Order.Create(Guid.NewGuid(), address);

        Action act = () => order.Submit();

        act.Should().Throw<DomainException>()
            .WithMessage("*empty*");
    }

    [Fact]
    public void Money_Add_Different_Currencies_Should_Throw()
    {
        var usd = Money.Of(100, "USD");
        var eur = Money.Of(50, "EUR");

        Action act = () => usd.Add(eur);

        act.Should().Throw<DomainException>()
            .WithMessage("*USD*EUR*");
    }

    [Fact]
    public void Order_Items_Should_Merge_When_Same_Product()
    {
        var address = Address.Create("1 A St", "City", "12345", "US");
        var order = Order.Create(Guid.NewGuid(), address);
        var productId = Guid.NewGuid();
        var price = Money.Of(10m, "USD");

        order.AddItem(productId, "Widget", price, 3);
        order.AddItem(productId, "Widget", price, 2); // Same product

        order.Items.Should().HaveCount(1);
        order.Items.Single().Quantity.Should().Be(5);
        order.Total.Amount.Should().Be(50m);
    }
}
```

---

## Step 1799: AutoFixture — Reduce Arrange Boilerplate

```csharp
// Install: dotnet add package AutoFixture
// Install: dotnet add package AutoFixture.AutoMoq
// Install: dotnet add package AutoFixture.Xunit2

// Custom AutoFixture configuration for domain types
public class DomainCustomization : ICustomization
{
    public void Customize(IFixture fixture)
    {
        fixture.Customize<Money>(c => c.FromFactory(() =>
            Money.Of(fixture.Create<decimal>() % 1000 + 1, "USD")));

        fixture.Customize<Address>(c => c.FromFactory(() =>
            Address.Create(
                fixture.Create<string>()[..50],
                fixture.Create<string>()[..30],
                "12345",
                "US")));

        fixture.Customize<OrderId>(c => c.FromFactory(() => OrderId.New()));
    }
}

public class AutoDomainDataAttribute : AutoDataAttribute
{
    public AutoDomainDataAttribute()
        : base(() => new Fixture().Customize(new DomainCustomization())) { }
}

// Usage — no more Arrange boilerplate
public class OrderAutoFixtureTests
{
    [Theory, AutoDomainData]
    public void AddItem_Should_Increase_Total(
        Address address,
        Guid customerId,
        Guid productId,
        Money unitPrice)
    {
        var order = Order.Create(customerId, address);
        order.AddItem(productId, "Test Product", unitPrice, 3);

        order.Total.Amount.Should().Be(unitPrice.Amount * 3);
    }

    [Theory, AutoDomainData]
    public void Cancel_CompletedOrder_Should_Throw(
        Address address,
        Guid customerId,
        Money unitPrice)
    {
        var order = Order.Create(customerId, address);
        order.AddItem(Guid.NewGuid(), "Item", unitPrice, 1);
        order.Submit();
        order.ConfirmPayment("TXN123");
        order.StartFulfillment();
        order.Complete();

        Action act = () => order.Cancel("Too late");

        act.Should().Throw<DomainException>();
    }
}
```

---

## Step 1800: Property-Based Testing with FsCheck

```csharp
// Install: dotnet add package FsCheck.Xunit
// Install: dotnet add package CsCheck  (alternative, pure C#)

public class MoneyPropertyTests
{
    // FsCheck generators for domain types
    static Gen<Money> MoneyGen =>
        from amount in Gen.Choose(1, 100_000).Select(a => (decimal)a / 100)
        select Money.Of(amount, "USD");

    // Property: Money addition is commutative
    [Fact]
    public void Money_Addition_Is_Commutative()
    {
        Prop.ForAll(
            Arb.From(MoneyGen),
            Arb.From(MoneyGen),
            (a, b) => a.Add(b) == b.Add(a))
        .QuickCheckThrowOnFailure();
    }

    // Property: Money addition is associative
    [Fact]
    public void Money_Addition_Is_Associative()
    {
        Prop.ForAll(
            Arb.From(MoneyGen),
            Arb.From(MoneyGen),
            Arb.From(MoneyGen),
            (a, b, c) =>
            {
                var left = a.Add(b).Add(c);
                var right = a.Add(b.Add(c));
                return left == right;
            })
        .QuickCheckThrowOnFailure();
    }

    // Property: Subtract then add returns original
    [Fact]
    public void Subtract_Then_Add_Is_Identity()
    {
        var total = Money.Of(100m, "USD");
        var part = Money.Of(30m, "USD");

        var result = total.Subtract(part).Add(part);

        result.Should().Be(total);
    }
}

// CsCheck (C#-native property testing)
public class OrderPropertyTests
{
    [Fact]
    public void Order_Total_Equals_Sum_Of_Line_Totals()
    {
        Gen.Int[1, 10].SelectMany(count =>
            Gen.Select(
                Gen.Decimal[1m, 999m],
                Gen.Int[1, 100],
                (price, qty) => (price, qty))
            .List[count])
        .Sample(lines =>
        {
            var address = Address.Create("1 Test", "City", "12345", "US");
            var order = Order.Create(Guid.NewGuid(), address);

            decimal expectedTotal = 0;
            foreach (var (price, qty) in lines)
            {
                var productId = Guid.NewGuid();
                var money = Money.Of(Math.Round(price, 2), "USD");
                order.AddItem(productId, $"Product {productId}", money, qty);
                expectedTotal += Math.Round(price, 2) * qty;
            }

            return order.Total.Amount == expectedTotal;
        });
    }
}
```

---

## Step 1801: Contract Testing with Pact (Consumer-Driven)

```csharp
// Consumer side — your Order service calling Catalog service
// Install: dotnet add package PactNet (consumer project)

public class CatalogServiceContractTests : IDisposable
{
    private readonly IPactBuilderV4 _pactBuilder;
    private readonly IPact _pact;

    public CatalogServiceContractTests()
    {
        var pactConfig = new PactConfig
        {
            PactDir = Path.Join(Directory.GetCurrentDirectory(), "pacts"),
            Outputters = [new XunitOutput(output)],
            DefaultJsonSettings = new JsonSerializerSettings
            {
                ContractlessDefaultSnapshotSerializer = true
            }
        };

        _pact = Pact.V4("OrderService", "CatalogService", pactConfig);
        _pactBuilder = _pact.WithHttpInteractions();
    }

    [Fact]
    public async Task GetProduct_Should_Return_ProductDetails()
    {
        var productId = Guid.Parse("550e8400-e29b-41d4-a716-446655440000");

        _pactBuilder
            .UponReceiving("a request for product details")
            .Given("product 550e8400 exists and is published")
            .WithRequest(HttpMethod.Get, $"/api/products/{productId}")
            .WithHeader("Accept", "application/json")
            .WillRespond()
            .WithStatus(200)
            .WithHeader("Content-Type", "application/json; charset=utf-8")
            .WithJsonBody(new
            {
                id = productId.ToString(),
                name = Match.Type("Widget Pro"),
                price = Match.Decimal(29.99m),
                currency = Match.Regex("USD", "^[A-Z]{3}$"),
                isAvailable = Match.Type(true)
            });

        await _pactBuilder.VerifyAsync(async ctx =>
        {
            var client = new CatalogApiClient(ctx.MockServerUri.ToString());
            var product = await client.GetProductAsync(productId);

            product.Should().NotBeNull();
            product!.Name.Should().NotBeNullOrEmpty();
            product.Price.Should().BeGreaterThan(0);
        });
    }

    [Fact]
    public async Task GetProduct_WhenNotFound_Should_Return_404()
    {
        var nonExistentId = Guid.NewGuid();

        _pactBuilder
            .UponReceiving("a request for a non-existent product")
            .Given("product does not exist")
            .WithRequest(HttpMethod.Get, $"/api/products/{nonExistentId}")
            .WillRespond()
            .WithStatus(404)
            .WithJsonBody(new
            {
                type = Match.Type("https://tools.ietf.org/html/rfc9110#section-15.5.5"),
                title = Match.Type("Not Found"),
                status = 404
            });

        await _pactBuilder.VerifyAsync(async ctx =>
        {
            var client = new CatalogApiClient(ctx.MockServerUri.ToString());
            var product = await client.GetProductAsync(nonExistentId);

            product.Should().BeNull();
        });
    }

    public void Dispose() => _pact.Dispose();
}

// Provider side — Catalog service verifying the contract
// Install: dotnet add package PactNet (provider project)

public class CatalogServicePactProviderTests : IClassFixture<WebApplicationFactory<CatalogProgram>>
{
    private readonly WebApplicationFactory<CatalogProgram> _factory;

    public CatalogServicePactProviderTests(WebApplicationFactory<CatalogProgram> factory)
        => _factory = factory;

    [Fact]
    public void VerifyOrderServiceConsumerContract()
    {
        var config = new PactVerifierConfig
        {
            Outputters = [new XunitOutput(output)],
            LogLevel = PactLogLevel.Information
        };

        new PactVerifier("CatalogService", config)
            .WithHttpEndpoint(_factory.Server.BaseAddress)
            .WithFileSource(new FileInfo("../OrderService/pacts/OrderService-CatalogService.json"))
            .WithProviderStateUrl(new Uri(_factory.Server.BaseAddress, "/_pact/provider-states"))
            .Verify();
    }
}

// Provider state setup endpoint
[ApiController]
[Route("/_pact")]
public sealed class PactProviderStatesController : ControllerBase
{
    private readonly AppDbContext _db;

    public PactProviderStatesController(AppDbContext db) => _db = db;

    [Post("provider-states")]
    public async Task<IActionResult> SetState([FromBody] ProviderState state)
    {
        switch (state.State)
        {
            case "product 550e8400 exists and is published":
                var product = Product.Create(
                    "Widget Pro",
                    "A great widget",
                    Money.Of(29.99m, "USD"),
                    "WGT-PRO-001");
                product.Publish();
                // seed to test DB using reflection to set Id
                _db.Products.Add(product);
                await _db.SaveChangesAsync();
                break;

            case "product does not exist":
                // No setup needed — DB is empty by default
                break;
        }

        return Ok();
    }
}
```

---

## Step 1802: Mutation Testing with Stryker.NET

```bash
# Install Stryker.NET
dotnet tool install -g dotnet-stryker

# Run mutation tests on a specific project
dotnet stryker \
  --project "src/MyApp.Domain/MyApp.Domain.csproj" \
  --test-project "tests/MyApp.Domain.Tests/MyApp.Domain.Tests.csproj" \
  --mutation-level Advanced \
  --reporters html \
  --reporters dashboard \
  --threshold-high 90 \
  --threshold-low 75 \
  --threshold-break 60

# stryker-config.json
```

```json
{
  "stryker-config": {
    "project": "src/MyApp.Domain/MyApp.Domain.csproj",
    "test-projects": [
      "tests/MyApp.Domain.Tests/MyApp.Domain.Tests.csproj"
    ],
    "mutation-level": "Advanced",
    "reporters": ["html", "json", "progress"],
    "thresholds": {
      "high": 90,
      "low": 75,
      "break": 60
    },
    "ignore-mutations": [
      "string"
    ],
    "mutate": [
      "src/**/*.cs",
      "!src/**/*Generated.cs",
      "!src/**/*.Designer.cs"
    ]
  }
}
```

```csharp
// Stryker shows you WHICH mutations survived (weren't caught by tests)
// Example output:
// [Survived] Mutant: line 45 — Changed 'Status == OrderStatus.Draft' to 'Status != OrderStatus.Draft'
// This means your tests don't verify you CAN'T add items to non-Draft orders!

// Fix: add the missing test
[Fact]
public void AddItem_To_SubmittedOrder_Should_Throw()
{
    var address = Address.Create("1 Test St", "City", "12345", "US");
    var order = Order.Create(Guid.NewGuid(), address);
    order.AddItem(Guid.NewGuid(), "Widget", Money.Of(10m, "USD"), 1);
    order.Submit(); // Now in Submitted state

    Action act = () => order.AddItem(Guid.NewGuid(), "Widget2", Money.Of(5m, "USD"), 1);

    act.Should().Throw<DomainException>()
        .WithMessage("*Draft*");
}
```

---

## Step 1803: Integration Tests with WebApplicationFactory

```csharp
// Tests/Integration/OrdersApiIntegrationTests.cs
// Install: dotnet add package Microsoft.AspNetCore.Mvc.Testing
// Install: dotnet add package Testcontainers.PostgreSql

public sealed class OrdersApiIntegrationTests
    : IClassFixture<OrdersApiFactory>, IAsyncLifetime
{
    private readonly OrdersApiFactory _factory;
    private readonly HttpClient _client;

    public OrdersApiIntegrationTests(OrdersApiFactory factory)
    {
        _factory = factory;
        _client = factory.CreateClient();
    }

    public Task InitializeAsync() => _factory.ResetDatabaseAsync();
    public Task DisposeAsync() => Task.CompletedTask;

    [Fact]
    public async Task PlaceOrder_Should_Return_Created_With_OrderId()
    {
        var customerId = await CreateCustomerAsync();

        var request = new
        {
            CustomerId = customerId,
            Street = "123 Commerce St",
            City = "Austin",
            PostalCode = "78701",
            Country = "US",
            Items = new[]
            {
                new { ProductId = Guid.NewGuid(), Quantity = 2 }
            }
        };

        var response = await _client.PostAsJsonAsync("/api/orders", request);

        response.Should().HaveStatusCode(HttpStatusCode.Created);
        var body = await response.Content.ReadFromJsonAsync<PlaceOrderResponse>();
        body!.OrderId.Should().NotBeEmpty();
        response.Headers.Location.Should().NotBeNull();
    }

    [Fact]
    public async Task GetOrder_WhenNotFound_Should_Return_404()
    {
        var response = await _client.GetAsync($"/api/orders/{Guid.NewGuid()}");

        response.Should().HaveStatusCode(HttpStatusCode.NotFound);
    }

    [Fact]
    public async Task PlaceOrder_With_Empty_Items_Should_Return_422()
    {
        var request = new
        {
            CustomerId = Guid.NewGuid(),
            Street = "123 St",
            City = "City",
            PostalCode = "12345",
            Country = "US",
            Items = Array.Empty<object>()
        };

        var response = await _client.PostAsJsonAsync("/api/orders", request);

        response.Should().HaveStatusCode(HttpStatusCode.UnprocessableEntity);
    }

    private async Task<Guid> CreateCustomerAsync()
    {
        var response = await _client.PostAsJsonAsync("/api/customers", new
        {
            FullName = "John Doe",
            Email = "john@example.com"
        });
        var body = await response.Content.ReadFromJsonAsync<CreateCustomerResponse>();
        return body!.CustomerId;
    }
}

// Test factory with Testcontainers
public sealed class OrdersApiFactory : WebApplicationFactory<Program>, IAsyncLifetime
{
    private readonly PostgreSqlContainer _postgres = new PostgreSqlBuilder()
        .WithDatabase("orders_test")
        .WithUsername("postgres")
        .WithPassword("postgres")
        .WithImage("postgres:16-alpine")
        .Build();

    public async Task InitializeAsync()
    {
        await _postgres.StartAsync();
    }

    public async Task DisposeAsync()
    {
        await _postgres.DisposeAsync();
    }

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureTestServices(services =>
        {
            // Replace real DB connection with test container
            var descriptor = services.SingleOrDefault(
                d => d.ServiceType == typeof(DbContextOptions<AppDbContext>));
            if (descriptor is not null) services.Remove(descriptor);

            services.AddDbContext<AppDbContext>(options =>
                options.UseNpgsql(_postgres.GetConnectionString()));

            // Replace external services with fakes
            services.AddScoped<IEmailService, FakeEmailService>();
            services.AddScoped<ICatalogService, FakeCatalogService>();
        });

        builder.UseEnvironment("Testing");
    }

    public async Task ResetDatabaseAsync()
    {
        using var scope = Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();

        // Respawn — fast DB reset without dropping schema
        await using var conn = new NpgsqlConnection(_postgres.GetConnectionString());
        await conn.OpenAsync();

        var respawner = await Respawner.CreateAsync(conn, new RespawnerOptions
        {
            DbAdapter = DbAdapter.Postgres,
            SchemasToInclude = ["public"]
        });

        await respawner.ResetAsync(conn);
        await db.Database.MigrateAsync();
    }
}

// Fake implementations for tests
public sealed class FakeEmailService : IEmailService
{
    public List<(string To, string Subject)> SentEmails { get; } = [];

    public Task SendOrderConfirmationAsync(
        string email, Guid orderId, decimal amount, string currency, CancellationToken ct)
    {
        SentEmails.Add((email, $"Order Confirmation - {orderId}"));
        return Task.CompletedTask;
    }
}
```

---

## Step 1804: Load Testing with k6

```javascript
// load-tests/order-flow.js
import http from 'k6/http';
import { check, group, sleep } from 'k6';
import { Rate, Trend, Counter } from 'k6/metrics';

// Custom metrics
const errorRate = new Rate('errors');
const orderCreationDuration = new Trend('order_creation_duration', true);
const successfulOrders = new Counter('successful_orders');

export const options = {
  stages: [
    { duration: '30s', target: 10 },   // Ramp up to 10 VUs
    { duration: '1m', target: 50 },    // Ramp up to 50 VUs
    { duration: '3m', target: 50 },    // Stay at 50 VUs (steady state)
    { duration: '30s', target: 100 },  // Spike to 100 VUs
    { duration: '1m', target: 100 },   // Stay at spike
    { duration: '30s', target: 0 },    // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1000'], // 95th pct < 500ms
    errors: ['rate<0.01'],                           // Error rate < 1%
    order_creation_duration: ['p(95)<800'],          // Order creation < 800ms
  },
};

const BASE_URL = __ENV.BASE_URL || 'http://localhost:5000';

function getAuthToken() {
  const res = http.post(`${BASE_URL}/api/auth/token`, JSON.stringify({
    clientId: 'loadtest',
    clientSecret: __ENV.TEST_SECRET,
  }), { headers: { 'Content-Type': 'application/json' } });

  check(res, { 'auth token returned': r => r.status === 200 });
  return res.json('access_token');
}

export function setup() {
  return { token: getAuthToken() };
}

export default function (data) {
  const headers = {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${data.token}`,
  };

  group('Order creation flow', () => {
    // 1. Create order
    const orderPayload = JSON.stringify({
      customerId: '550e8400-e29b-41d4-a716-446655440000',
      street: '123 Test St',
      city: 'Austin',
      postalCode: '78701',
      country: 'US',
      items: [
        { productId: 'a716e8400-e29b-41d4-a716-446655440001', quantity: 2 },
      ],
    });

    const start = Date.now();
    const createRes = http.post(`${BASE_URL}/api/orders`, orderPayload, { headers });
    orderCreationDuration.add(Date.now() - start);

    const created = check(createRes, {
      'order created (201)': r => r.status === 201,
      'order id returned': r => r.json('orderId') !== null,
      'location header present': r => r.headers['Location'] !== undefined,
    });

    errorRate.add(!created);
    if (!created) return;

    successfulOrders.add(1);
    const orderId = createRes.json('orderId');

    // 2. Read the order back
    sleep(0.5);
    const getRes = http.get(`${BASE_URL}/api/orders/${orderId}`, { headers });
    check(getRes, {
      'order retrieved (200)': r => r.status === 200,
      'order status is Submitted': r => r.json('status') === 'Submitted',
    });
  });

  group('Product listing', () => {
    const listRes = http.get(`${BASE_URL}/api/products?page=1&pageSize=20`, { headers });
    check(listRes, {
      'products listed (200)': r => r.status === 200,
      'has items': r => r.json('data').length > 0,
    });
    errorRate.add(listRes.status !== 200);
  });

  sleep(Math.random() * 2 + 1); // Think time: 1-3 seconds
}

export function teardown(data) {
  console.log(`Test complete. Successful orders: ${data}`);
}
```

```bash
# Run load test
k6 run \
  --env BASE_URL=https://staging.myapp.com \
  --env TEST_SECRET=your_secret \
  --out influxdb=http://influxdb:8086/k6 \
  load-tests/order-flow.js

# HTML report
k6 run --out web-dashboard load-tests/order-flow.js
```

---

## Step 1805: NBomber — Load Testing in .NET

```csharp
// Install: dotnet add package NBomber
// Install: dotnet add package NBomber.Http

// LoadTests/OrderLoadTest.cs
public static class OrderLoadTest
{
    public static void Run()
    {
        using var httpClient = new HttpClient { BaseAddress = new Uri("http://localhost:5000") };

        // Scenario 1: Create orders
        var createOrderScenario = Scenario.Create("create_order", async context =>
        {
            var payload = new StringContent(
                JsonSerializer.Serialize(new
                {
                    customerId = Guid.NewGuid(),
                    street = "123 Test St",
                    city = "Austin",
                    postalCode = "78701",
                    country = "US",
                    items = new[] { new { productId = Guid.NewGuid(), quantity = 1 } }
                }),
                Encoding.UTF8,
                "application/json");

            var response = await httpClient.PostAsync("/api/orders", payload);
            return response.IsSuccessStatusCode ? Response.Ok() : Response.Fail();
        })
        .WithLoadSimulations(
            Simulation.RampingInject(rate: 50, interval: TimeSpan.FromSeconds(1), during: TimeSpan.FromMinutes(1)),
            Simulation.Inject(rate: 100, interval: TimeSpan.FromSeconds(1), during: TimeSpan.FromMinutes(3)),
            Simulation.RampingInject(rate: 0, interval: TimeSpan.FromSeconds(1), during: TimeSpan.FromSeconds(30))
        );

        // Scenario 2: Read orders (heavier read load)
        var readOrderScenario = Scenario.Create("read_orders", async context =>
        {
            var response = await httpClient.GetAsync("/api/orders?page=1&pageSize=20");
            return response.IsSuccessStatusCode ? Response.Ok() : Response.Fail();
        })
        .WithLoadSimulations(
            Simulation.KeepConstant(copies: 50, during: TimeSpan.FromMinutes(5))
        );

        NBomberRunner
            .RegisterScenarios(createOrderScenario, readOrderScenario)
            .WithReportFormats(ReportFormat.Html, ReportFormat.Csv)
            .WithReportFileName("load-test-results")
            .WithTestName("Order Service Load Test")
            .Run();
    }
}
```

---

## Step 1806: Snapshot Testing

```csharp
// Install: dotnet add package Verify.Xunit
// Install: dotnet add package Verify.Http

// Snapshot tests verify serialization output doesn't change unexpectedly
[UsesVerify]
public class OrderSerializationTests
{
    [Fact]
    public async Task OrderDto_Serializes_Correctly()
    {
        var dto = new OrderSummaryDto(
            OrderId: Guid.Parse("550e8400-e29b-41d4-a716-446655440000"),
            Status: "Submitted",
            Total: 59.98m,
            Currency: "USD",
            ItemCount: 2,
            SubmittedAt: new DateTime(2024, 1, 15, 10, 30, 0, DateTimeKind.Utc),
            ShippingAddress: "123 Main St, Springfield");

        await Verify(dto);
        // Creates: OrderSerializationTests.OrderDto_Serializes_Correctly.verified.txt
        // On first run, creates the snapshot
        // On subsequent runs, compares against it
        // Any serialization change is caught immediately
    }

    [Fact]
    public async Task PlaceOrder_Response_Matches_Snapshot()
    {
        using var factory = new OrdersApiFactory();
        var client = factory.CreateClient();

        var response = await client.PostAsJsonAsync("/api/orders", new
        {
            CustomerId = Guid.Parse("550e8400-e29b-41d4-a716-446655440000"),
            Street = "123 Test St",
            City = "Austin",
            PostalCode = "78701",
            Country = "US",
            Items = new[] { new { ProductId = Guid.NewGuid(), Quantity = 1 } }
        });

        await VerifyJson(await response.Content.ReadAsStringAsync());
    }
}
```

---

## Step 1807: Behavior-Driven Testing with SpecFlow

```gherkin
# Tests/Features/OrderPlacement.feature
Feature: Order Placement
  As a customer
  I want to place an order
  So that I can purchase products

  Background:
    Given the following products exist:
      | Id                                   | Name        | Price | Currency |
      | 550e8400-e29b-41d4-a716-446655440001 | Widget Pro  | 29.99 | USD      |
      | 550e8400-e29b-41d4-a716-446655440002 | Gadget Plus | 49.99 | USD      |

  Scenario: Successfully place an order
    Given I am customer "john@example.com"
    And my shipping address is "123 Main St, Springfield, 62701, US"
    When I add 2 x "Widget Pro" to my order
    And I add 1 x "Gadget Plus" to my order
    And I submit the order
    Then the order status should be "Submitted"
    And the order total should be "$109.97"
    And I should receive a confirmation email

  Scenario: Cannot submit an empty order
    Given I am customer "jane@example.com"
    And my shipping address is "456 Oak Ave, Chicago, 60601, US"
    When I submit the order
    Then I should receive an error "Cannot submit an empty order"
    And no confirmation email should be sent

  Scenario Outline: Plan limit enforcement
    Given I am on the "<Plan>" plan
    And I already have <Existing> products
    When I try to create a new product
    Then I <Result>

    Examples:
      | Plan       | Existing | Result                |
      | Free       | 10       | receive a plan error  |
      | Free       | 5        | succeed               |
      | Pro        | 999      | receive a plan error  |
      | Pro        | 500      | succeed               |
      | Enterprise | 9999     | succeed               |
```

```csharp
// Tests/StepDefinitions/OrderPlacementSteps.cs
[Binding]
public sealed class OrderPlacementSteps
{
    private readonly OrdersApiFactory _factory;
    private HttpClient _client = null!;
    private HttpResponseMessage _lastResponse = null!;
    private Guid _customerId;

    public OrderPlacementSteps(OrdersApiFactory factory) => _factory = factory;

    [Given(@"I am customer ""(.*)""")]
    public async Task GivenIAmCustomer(string email)
    {
        _client = _factory.CreateClient();
        var response = await _client.PostAsJsonAsync("/api/customers", new { Email = email, FullName = "Test User" });
        var body = await response.Content.ReadFromJsonAsync<CreateCustomerResponse>();
        _customerId = body!.CustomerId;
    }

    [When(@"I add (\d+) x ""(.*)"" to my order")]
    public async Task WhenIAddItemToOrder(int quantity, string productName)
    {
        // store items for later submission
        _pendingItems.Add((productName, quantity));
    }

    [When(@"I submit the order")]
    public async Task WhenISubmitTheOrder()
    {
        _lastResponse = await _client.PostAsJsonAsync("/api/orders", new
        {
            CustomerId = _customerId,
            Items = _pendingItems.Select(i => new { ProductId = LookupProductId(i.Name), i.Quantity })
        });
    }

    [Then(@"the order status should be ""(.*)""")]
    public async Task ThenOrderStatusIs(string expectedStatus)
    {
        _lastResponse.StatusCode.Should().Be(HttpStatusCode.Created);
        var body = await _lastResponse.Content.ReadFromJsonAsync<PlaceOrderResponse>();
        var getResponse = await _client.GetAsync($"/api/orders/{body!.OrderId}");
        var order = await getResponse.Content.ReadFromJsonAsync<OrderSummaryDto>();
        order!.Status.Should().Be(expectedStatus);
    }
}
```

---

## Step 1808: Test Data Builders

```csharp
// Tests/Builders/OrderBuilder.cs
// Builder pattern for complex test data setup
public sealed class OrderBuilder
{
    private Guid _customerId = Guid.NewGuid();
    private string _street = "123 Test St";
    private string _city = "Austin";
    private string _postalCode = "78701";
    private string _country = "US";
    private readonly List<(Guid ProductId, string Name, decimal Price, int Qty)> _items = [];

    public static OrderBuilder Default() => new();

    public OrderBuilder WithCustomer(Guid customerId)
    {
        _customerId = customerId;
        return this;
    }

    public OrderBuilder WithAddress(string street, string city, string postalCode, string country)
    {
        _street = street;
        _city = city;
        _postalCode = postalCode;
        _country = country;
        return this;
    }

    public OrderBuilder WithItem(decimal price = 29.99m, int quantity = 1, string? name = null)
    {
        _items.Add((Guid.NewGuid(), name ?? "Widget", price, quantity));
        return this;
    }

    public OrderBuilder WithItems(int count, decimal priceEach = 10m)
    {
        for (int i = 0; i < count; i++)
            WithItem(priceEach, 1, $"Item {i + 1}");
        return this;
    }

    public Order Build()
    {
        var address = Address.Create(_street, _city, _postalCode, _country);
        var order = Order.Create(_customerId, address);

        foreach (var (productId, name, price, qty) in _items)
            order.AddItem(productId, name, Money.Of(price, "USD"), qty);

        return order;
    }

    public Order BuildSubmitted()
    {
        if (!_items.Any()) WithItem();
        var order = Build();
        order.Submit();
        return order;
    }

    public Order BuildWithPaymentConfirmed()
    {
        var order = BuildSubmitted();
        order.ConfirmPayment("TXN_TEST_001");
        return order;
    }
}

// Usage in tests
[Fact]
public void StartFulfillment_On_UnconfirmedOrder_Should_Throw()
{
    var order = OrderBuilder.Default()
        .WithItem(price: 99.99m, quantity: 3)
        .BuildSubmitted(); // NOT payment confirmed

    Action act = () => order.StartFulfillment();

    act.Should().Throw<DomainException>()
        .WithMessage("*confirmed payment*");
}

[Fact]
public void Complete_HappyPath_Should_Succeed()
{
    var order = OrderBuilder.Default()
        .WithItem(49.99m, 2)
        .BuildWithPaymentConfirmed();
    order.StartFulfillment();

    Action act = () => order.Complete();

    act.Should().NotThrow();
    order.Status.Should().Be(OrderStatus.Completed);
    order.DomainEvents.Should().ContainSingle()
        .Which.Should().BeOfType<OrderCompletedEvent>();
}
```

---

## Step 1809: Testing Concurrency and Race Conditions

```csharp
// Tests/Concurrency/ConcurrencyTests.cs
public sealed class ConcurrencyTests : IClassFixture<OrdersApiFactory>
{
    private readonly OrdersApiFactory _factory;

    public ConcurrencyTests(OrdersApiFactory factory) => _factory = factory;

    [Fact]
    public async Task ConcurrentOrderSubmits_Should_Only_Succeed_Once()
    {
        // Arrange: create an order in Draft state
        using var scope = _factory.Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        var orderRepo = scope.ServiceProvider.GetRequiredService<IOrderRepository>();
        var uow = scope.ServiceProvider.GetRequiredService<IUnitOfWork>();

        var order = OrderBuilder.Default().WithItem(10m, 1).Build();
        uow.Orders.Add(order);
        await uow.CommitAsync();

        var orderId = order.Id;

        // Act: submit the same order from two concurrent requests
        var client1 = _factory.CreateClient();
        var client2 = _factory.CreateClient();

        var task1 = client1.PostAsync($"/api/orders/{orderId}/submit", null);
        var task2 = client2.PostAsync($"/api/orders/{orderId}/submit", null);

        var results = await Task.WhenAll(task1, task2);

        // Assert: exactly one succeeds, one gets 409 Conflict
        var statusCodes = results.Select(r => r.StatusCode).ToList();
        statusCodes.Should().Contain(HttpStatusCode.OK);
        statusCodes.Should().Contain(HttpStatusCode.Conflict);
    }

    [Fact]
    public async Task ConcurrentProductCreation_Should_Respect_PlanLimits()
    {
        // Scenario: Free plan (max 10 products), 15 concurrent creates
        // All 15 fire simultaneously — only first 10 should succeed

        var tasks = Enumerable.Range(0, 15).Select(i =>
            _factory.CreateClient()
                .PostAsJsonAsync("/api/products", new { Name = $"Product {i}", Price = 9.99m })
        ).ToList();

        var responses = await Task.WhenAll(tasks);

        var successCount = responses.Count(r => r.StatusCode == HttpStatusCode.Created);
        var rejectedCount = responses.Count(r => r.StatusCode == HttpStatusCode.PaymentRequired);

        successCount.Should().BeLessThanOrEqualTo(10);
        rejectedCount.Should().Be(15 - successCount);
    }
}
```

---

## Step 1810: Testing Domain Events End-to-End

```csharp
// Tests/Integration/DomainEventTests.cs
public sealed class DomainEventIntegrationTests : IClassFixture<OrdersApiFactory>
{
    private readonly OrdersApiFactory _factory;

    public DomainEventIntegrationTests(OrdersApiFactory factory) => _factory = factory;

    [Fact]
    public async Task SubmitOrder_Should_Send_ConfirmationEmail()
    {
        // Arrange
        using var scope = _factory.Services.CreateScope();
        var emailService = scope.ServiceProvider.GetRequiredService<IEmailService>()
            as FakeEmailService;

        var client = _factory.CreateClient();

        // Act
        var response = await client.PostAsJsonAsync("/api/orders", new
        {
            CustomerId = Guid.NewGuid(),
            Street = "1 Test St",
            City = "City",
            PostalCode = "12345",
            Country = "US",
            Items = new[] { new { ProductId = Guid.NewGuid(), Quantity = 1 } }
        });

        // Wait briefly for async event handlers
        await Task.Delay(100);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Created);
        emailService!.SentEmails.Should().HaveCount(1);
        emailService.SentEmails.Single().Subject.Should().StartWith("Order Confirmation");
    }

    [Fact]
    public async Task CancelOrder_Should_Release_Inventory_Reservation()
    {
        // Arrange
        var productId = Guid.NewGuid();
        using var scope = _factory.Services.CreateScope();
        var inventoryService = scope.ServiceProvider.GetRequiredService<IInventoryReservationService>()
            as FakeInventoryReservationService;

        var client = _factory.CreateClient();

        // Create and submit order
        var createResponse = await client.PostAsJsonAsync("/api/orders", new
        {
            CustomerId = Guid.NewGuid(),
            Street = "1 Test St",
            City = "City",
            PostalCode = "12345",
            Country = "US",
            Items = new[] { new { ProductId = productId, Quantity = 2 } }
        });

        var orderId = (await createResponse.Content.ReadFromJsonAsync<PlaceOrderResponse>())!.OrderId;

        // Act: cancel the order
        var cancelResponse = await client.PostAsJsonAsync(
            $"/api/orders/{orderId}/cancel",
            new { Reason = "Customer changed mind" });

        await Task.Delay(100); // wait for event handler

        // Assert: inventory reservation released
        cancelResponse.StatusCode.Should().Be(HttpStatusCode.OK);
        inventoryService!.ReleasedOrderIds.Should().Contain(orderId);
    }
}
```

---

## Step 1811: CI/CD Pipeline for Testing

```yaml
# .github/workflows/test.yml
name: Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  unit-tests:
    name: Unit & Domain Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '9.0.x'

      - name: Restore
        run: dotnet restore

      - name: Build
        run: dotnet build --no-restore --configuration Release

      - name: Unit Tests
        run: |
          dotnet test tests/MyApp.Domain.Tests \
            --no-build --configuration Release \
            --logger "trx;LogFileName=unit-results.trx" \
            --collect:"XPlat Code Coverage" \
            -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.Format=opencover

      - name: Architecture Tests
        run: |
          dotnet test tests/MyApp.Architecture.Tests \
            --no-build --configuration Release

      - name: Upload Coverage
        uses: codecov/codecov-action@v4
        with:
          files: '**/coverage.opencover.xml'
          fail_ci_if_error: true

  integration-tests:
    name: Integration Tests
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: orders_test
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '9.0.x'

      - name: Integration Tests
        run: |
          dotnet test tests/MyApp.Integration.Tests \
            --configuration Release \
            --logger "trx;LogFileName=integration-results.trx"
        env:
          ConnectionStrings__DefaultConnection: "Host=localhost;Database=orders_test;Username=postgres;Password=postgres"

  contract-tests:
    name: Pact Contract Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Consumer Contract Tests
        run: dotnet test tests/MyApp.Consumer.ContractTests

      - name: Publish Pacts to Pact Broker
        run: |
          docker run --rm pactfoundation/pact-cli:latest \
            publish pacts/ \
            --broker-base-url ${{ secrets.PACT_BROKER_URL }} \
            --consumer-app-version ${{ github.sha }} \
            --tag ${{ github.ref_name }}

  mutation-tests:
    name: Mutation Tests (Stryker)
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    steps:
      - uses: actions/checkout@v4

      - name: Install Stryker
        run: dotnet tool install -g dotnet-stryker

      - name: Run Mutation Tests
        run: |
          dotnet stryker \
            --threshold-high 90 \
            --threshold-low 75 \
            --threshold-break 60 \
            --reporters json

      - name: Upload Mutation Report
        uses: actions/upload-artifact@v4
        with:
          name: mutation-report
          path: StrykerOutput/
```

---

## Step 1812: Test Quality Metrics Dashboard

```csharp
// Tests/Quality/TestQualityReport.cs
// Run this to assess overall test quality
public static class TestQualityMetrics
{
    public static TestReport GenerateReport(Assembly domainAssembly, Assembly testAssembly)
    {
        var domainTypes = domainAssembly.GetTypes()
            .Where(t => !t.IsAbstract && !t.IsInterface)
            .ToList();

        var testTypes = testAssembly.GetTypes()
            .Where(t => t.GetMethods().Any(m =>
                m.GetCustomAttribute<FactAttribute>() is not null ||
                m.GetCustomAttribute<TheoryAttribute>() is not null))
            .ToList();

        var totalTestMethods = testTypes
            .SelectMany(t => t.GetMethods())
            .Count(m =>
                m.GetCustomAttribute<FactAttribute>() is not null ||
                m.GetCustomAttribute<TheoryAttribute>() is not null);

        return new TestReport(
            DomainClassCount: domainTypes.Count,
            TestClassCount: testTypes.Count,
            TotalTests: totalTestMethods,
            TestToProductionRatio: (double)totalTestMethods / domainTypes.Count
        );
    }
}

public record TestReport(
    int DomainClassCount,
    int TestClassCount,
    int TotalTests,
    double TestToProductionRatio)
{
    public string QualityGrade => TestToProductionRatio switch
    {
        >= 5 => "A — Excellent",
        >= 3 => "B — Good",
        >= 2 => "C — Adequate",
        >= 1 => "D — Needs Improvement",
        _ => "F — Insufficient"
    };
}
```

---

## Summary: Advanced Testing Strategy Matrix

| Test Type | Tool | What It Catches | When to Run |
|---|---|---|---|
| Unit | xUnit + FluentAssertions | Logic bugs | Every commit |
| Property | FsCheck / CsCheck | Edge cases, invariant violations | Every commit |
| Architecture | NetArchTest | Layer violations | Every commit |
| Integration | WebApplicationFactory + Testcontainers | Infrastructure wiring | Every PR |
| Contract | PactNet | API breaking changes | Every PR, publish pacts |
| Snapshot | Verify.Xunit | Serialization regressions | Every PR |
| Mutation | Stryker.NET | Missing test assertions | Daily / PR |
| Load | k6 / NBomber | Performance regressions | Pre-release |
| Chaos | Simmy | Resilience failures | Pre-release |

**The golden rule**: A test that doesn't fail when it should is worse than no test — it gives false confidence. Mutation testing proves your tests actually catch bugs.

**Minimum viable test suite for production**:
- All domain logic: 100% branch coverage (unit tests)
- All HTTP endpoints: integration tests with real DB (Testcontainers)
- All inter-service contracts: Pact consumer tests
- Critical paths: load-tested against SLA thresholds

---

*Next: Part 74 — GraphQL with Hot Chocolate: Schema-First Design, Subscriptions & Batching*
