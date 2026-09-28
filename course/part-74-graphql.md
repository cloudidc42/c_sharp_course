# Part 74: GraphQL with Hot Chocolate — Schema Design, Subscriptions & DataLoader

## Steps 1813–1828 | World-Class GraphQL APIs

---

## Step 1813: Why GraphQL?

```
REST Problems that GraphQL Solves:
┌─────────────────────────────────────────────────────────────────────┐
│  Overfetching:  GET /orders returns 40 fields, client needs 5       │
│  Underfetching: Need order + customer + items = 3 round trips       │
│  Versioning:    /api/v1/ → /api/v2/ → /api/v3/ — maintenance hell  │
│  Rigid schema:  Frontend change requires backend coordination       │
└─────────────────────────────────────────────────────────────────────┘

GraphQL Solution:
┌─────────────────────────────────────────────────────────────────────┐
│  query {                                                            │
│    order(id: "abc") {                                               │
│      id status total { amount currency }                            │
│      customer { name email }      ← joined in one request          │
│      items { productName quantity unitPrice { amount } }            │
│    }                                                                │
│  }                                                                  │
│  → Single request, exactly the fields needed, no version number    │
└─────────────────────────────────────────────────────────────────────┘

Hot Chocolate Features:
• Code-first schema (C# classes → SDL)
• DataLoader (N+1 query prevention)
• Subscriptions (WebSocket/SSE)
• Persisted queries
• Relay spec (cursor pagination, node interface)
• Schema stitching / federated schemas
• Automatic filtering, sorting, pagination
• Integration with EF Core, MediatR
```

---

## Step 1814: Project Setup

```bash
dotnet new webapi -n MyApp.GraphQL -f net9.0
cd MyApp.GraphQL

# Core Hot Chocolate packages
dotnet add package HotChocolate.AspNetCore
dotnet add package HotChocolate.Data
dotnet add package HotChocolate.Data.EntityFramework
dotnet add package HotChocolate.Subscriptions.InMemory  # or Redis
dotnet add package HotChocolate.Diagnostics             # OpenTelemetry

# Optional: Relay spec support
dotnet add package HotChocolate.Types.Relay

# Database
dotnet add package Microsoft.EntityFrameworkCore.Design
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL
```

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseNpgsql(builder.Configuration.GetConnectionString("DefaultConnection")));

builder.Services
    .AddGraphQLServer()
    .AddQueryType<Query>()
    .AddMutationType<Mutation>()
    .AddSubscriptionType<Subscription>()
    .AddType<OrderType>()
    .AddType<OrderItemType>()
    .AddType<CustomerType>()
    .AddDataLoader<OrderByIdDataLoader>()
    .AddDataLoader<CustomerByIdDataLoader>()
    .AddFiltering()
    .AddSorting()
    .AddProjections()
    .AddInMemorySubscriptions()  // WebSocket subscriptions
    .RegisterDbContext<AppDbContext>();

var app = builder.Build();

app.UseWebSockets();  // Required for subscriptions
app.MapGraphQL();     // Default: /graphql
app.MapGraphQLVoyager(); // Schema visualizer: /graphql-voyager

app.Run();
```

---

## Step 1815: Domain Model — Database Entities

```csharp
// Data/AppDbContext.cs
public sealed class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    public DbSet<OrderRecord> Orders => Set<OrderRecord>();
    public DbSet<OrderItemRecord> OrderItems => Set<OrderItemRecord>();
    public DbSet<CustomerRecord> Customers => Set<CustomerRecord>();
    public DbSet<ProductRecord> Products => Set<ProductRecord>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<OrderRecord>(b =>
        {
            b.HasKey(o => o.Id);
            b.Property(o => o.Status).HasConversion<string>().HasMaxLength(50);
            b.HasMany(o => o.Items).WithOne().HasForeignKey(i => i.OrderId);
            b.HasOne(o => o.Customer).WithMany(c => c.Orders).HasForeignKey(o => o.CustomerId);
        });

        modelBuilder.Entity<CustomerRecord>(b =>
        {
            b.HasKey(c => c.Id);
            b.Property(c => c.Email).HasMaxLength(200);
        });

        modelBuilder.Entity<ProductRecord>(b => b.HasKey(p => p.Id));
    }
}

public class OrderRecord
{
    public Guid Id { get; init; } = Guid.NewGuid();
    public Guid CustomerId { get; set; }
    public string Status { get; set; } = "Draft";
    public decimal TotalAmount { get; set; }
    public string Currency { get; set; } = "USD";
    public DateTime CreatedAt { get; init; } = DateTime.UtcNow;
    public DateTime? UpdatedAt { get; set; }
    public CustomerRecord Customer { get; set; } = null!;
    public List<OrderItemRecord> Items { get; set; } = [];
}

public class OrderItemRecord
{
    public Guid Id { get; init; } = Guid.NewGuid();
    public Guid OrderId { get; set; }
    public Guid ProductId { get; set; }
    public string ProductName { get; set; } = string.Empty;
    public decimal UnitPrice { get; set; }
    public string Currency { get; set; } = "USD";
    public int Quantity { get; set; }
}

public class CustomerRecord
{
    public Guid Id { get; init; } = Guid.NewGuid();
    public string FullName { get; set; } = string.Empty;
    public string Email { get; set; } = string.Empty;
    public DateTime CreatedAt { get; init; } = DateTime.UtcNow;
    public List<OrderRecord> Orders { get; set; } = [];
}

public class ProductRecord
{
    public Guid Id { get; init; } = Guid.NewGuid();
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public string Currency { get; set; } = "USD";
    public bool IsAvailable { get; set; } = true;
}
```

---

## Step 1816: GraphQL Types — Code-First Schema

```csharp
// GraphQL/Types/MoneyType.cs
public sealed class MoneyType : ObjectType
{
    protected override void Configure(IObjectTypeDescriptor descriptor)
    {
        descriptor.Name("Money");
        descriptor.Field("amount").Type<NonNullType<DecimalType>>();
        descriptor.Field("currency").Type<NonNullType<StringType>>();
    }
}

// GraphQL/Types/OrderType.cs
[GraphQLName("Order")]
[GraphQLDescription("Represents a customer order")]
public sealed class OrderType : ObjectType<OrderRecord>
{
    protected override void Configure(IObjectTypeDescriptor<OrderRecord> descriptor)
    {
        descriptor.Field(o => o.Id).Type<NonNullType<IdType>>();
        descriptor.Field(o => o.Status).Type<NonNullType<StringType>>();
        descriptor.Field(o => o.CreatedAt).Type<NonNullType<DateTimeType>>();

        // Computed field: total as Money object
        descriptor.Field("total")
            .Type<NonNullType<ObjectType<MoneyDto>>>()
            .Resolve(ctx =>
            {
                var order = ctx.Parent<OrderRecord>();
                return new MoneyDto(order.TotalAmount, order.Currency);
            });

        // Resolve items via DataLoader (N+1 prevention)
        descriptor.Field("items")
            .Type<NonNullType<ListType<NonNullType<OrderItemType>>>>()
            .UseDbContext<AppDbContext>()
            .Resolve(async ctx =>
            {
                var order = ctx.Parent<OrderRecord>();
                var db = ctx.DbContext<AppDbContext>();
                return await db.OrderItems
                    .Where(i => i.OrderId == order.Id)
                    .ToListAsync(ctx.RequestAborted);
            });

        // Customer resolved via DataLoader
        descriptor.Field("customer")
            .Type<NonNullType<CustomerType>>()
            .ResolveWith<OrderResolvers>(r => r.GetCustomerAsync(default!, default!, default));
    }
}

public sealed class OrderResolvers
{
    public async Task<CustomerRecord?> GetCustomerAsync(
        [Parent] OrderRecord order,
        CustomerByIdDataLoader dataLoader,
        CancellationToken ct)
        => await dataLoader.LoadAsync(order.CustomerId, ct);
}

// GraphQL/Types/CustomerType.cs
[GraphQLName("Customer")]
public sealed class CustomerType : ObjectType<CustomerRecord>
{
    protected override void Configure(IObjectTypeDescriptor<CustomerRecord> descriptor)
    {
        descriptor.Field(c => c.Id).Type<NonNullType<IdType>>();
        descriptor.Field(c => c.FullName).Type<NonNullType<StringType>>();
        descriptor.Field(c => c.Email).Type<NonNullType<StringType>>();
        descriptor.Field(c => c.CreatedAt).Type<NonNullType<DateTimeType>>();

        descriptor.Field("orders")
            .Type<NonNullType<ListType<NonNullType<OrderType>>>>()
            .UsePaging<OrderType>()  // Relay-spec cursor pagination
            .UseFiltering()
            .UseSorting()
            .UseProjection()
            .UseDbContext<AppDbContext>()
            .Resolve(ctx =>
            {
                var customer = ctx.Parent<CustomerRecord>();
                return ctx.DbContext<AppDbContext>().Orders
                    .Where(o => o.CustomerId == customer.Id);
            });
    }
}

public record MoneyDto(decimal Amount, string Currency);
```

---

## Step 1817: Query Root

```csharp
// GraphQL/Queries/Query.cs
[QueryType]
public sealed class Query
{
    [UseDbContext(typeof(AppDbContext))]
    [UsePaging]
    [UseProjection]
    [UseFiltering]
    [UseSorting]
    public IQueryable<OrderRecord> GetOrders(AppDbContext db)
        => db.Orders;

    [UseDbContext(typeof(AppDbContext))]
    [UseFirstOrDefault]
    [UseProjection]
    [UseFiltering]
    public IQueryable<OrderRecord> GetOrderById(
        [ID(nameof(OrderRecord))] Guid id,
        AppDbContext db)
        => db.Orders.Where(o => o.Id == id);

    [UseDbContext(typeof(AppDbContext))]
    [UsePaging]
    [UseProjection]
    [UseFiltering]
    [UseSorting]
    public IQueryable<CustomerRecord> GetCustomers(AppDbContext db)
        => db.Customers;

    [UseDbContext(typeof(AppDbContext))]
    [UseFirstOrDefault]
    [UseProjection]
    public IQueryable<CustomerRecord> GetCustomerById(
        [ID(nameof(CustomerRecord))] Guid id,
        AppDbContext db)
        => db.Customers.Where(c => c.Id == id);

    [UseDbContext(typeof(AppDbContext))]
    [UsePaging]
    [UseProjection]
    [UseFiltering]
    [UseSorting]
    public IQueryable<ProductRecord> GetProducts(AppDbContext db)
        => db.Products;
}
```

Generated SDL (subset):
```graphql
type Query {
  orders(
    first: Int
    after: String
    last: Int
    before: String
    where: OrderFilterInput
    order: [OrderSortInput!]
  ): OrdersConnection

  orderById(id: ID!): Order

  customers(
    first: Int
    after: String
    where: CustomerFilterInput
  ): CustomersConnection
}

type OrdersConnection {
  pageInfo: PageInfo!
  edges: [OrdersEdge!]
  nodes: [Order!]
  totalCount: Int!
}
```

---

## Step 1818: DataLoader — Solving N+1

```csharp
// GraphQL/DataLoaders/CustomerByIdDataLoader.cs
public sealed class CustomerByIdDataLoader : BatchDataLoader<Guid, CustomerRecord?>
{
    private readonly IDbContextFactory<AppDbContext> _dbContextFactory;

    public CustomerByIdDataLoader(
        IBatchScheduler scheduler,
        IDbContextFactory<AppDbContext> dbContextFactory,
        DataLoaderOptions? options = null)
        : base(scheduler, options)
    {
        _dbContextFactory = dbContextFactory;
    }

    protected override async Task<IReadOnlyDictionary<Guid, CustomerRecord?>> LoadBatchAsync(
        IReadOnlyList<Guid> keys,
        CancellationToken ct)
    {
        await using var db = await _dbContextFactory.CreateDbContextAsync(ct);

        // Single query for ALL customers in the batch — no N+1
        var customers = await db.Customers
            .Where(c => keys.Contains(c.Id))
            .ToDictionaryAsync(c => c.Id, ct);

        // Return null for missing keys
        return keys.ToDictionary(key => key, key =>
            customers.TryGetValue(key, out var customer) ? customer : null);
    }
}

// GraphQL/DataLoaders/OrdersByCustomerDataLoader.cs
public sealed class OrdersByCustomerDataLoader : GroupedDataLoader<Guid, OrderRecord>
{
    private readonly IDbContextFactory<AppDbContext> _dbContextFactory;

    public OrdersByCustomerDataLoader(
        IBatchScheduler scheduler,
        IDbContextFactory<AppDbContext> dbContextFactory)
        : base(scheduler)
    {
        _dbContextFactory = dbContextFactory;
    }

    protected override async Task<ILookup<Guid, OrderRecord>> LoadGroupedBatchAsync(
        IReadOnlyList<Guid> customerIds,
        CancellationToken ct)
    {
        await using var db = await _dbContextFactory.CreateDbContextAsync(ct);

        var orders = await db.Orders
            .Where(o => customerIds.Contains(o.CustomerId))
            .ToListAsync(ct);

        // Returns ILookup: one key → many values
        return orders.ToLookup(o => o.CustomerId);
    }
}

// Product data loader with cache (same product referenced by many orders)
public sealed class ProductByIdDataLoader : CacheDataLoader<Guid, ProductRecord?>
{
    private readonly IDbContextFactory<AppDbContext> _dbContextFactory;

    public ProductByIdDataLoader(
        IDbContextFactory<AppDbContext> dbContextFactory,
        DataLoaderOptions? options = null)
        : base(options)
    {
        _dbContextFactory = dbContextFactory;
    }

    protected override async Task<ProductRecord?> LoadSingleAsync(
        Guid productId,
        CancellationToken ct)
    {
        await using var db = await _dbContextFactory.CreateDbContextAsync(ct);
        return await db.Products.FindAsync([productId], ct);
    }
}
```

---

## Step 1819: Mutations

```csharp
// GraphQL/Mutations/Mutation.cs
[MutationType]
public sealed class Mutation
{
    public async Task<PlaceOrderPayload> PlaceOrderAsync(
        PlaceOrderInput input,
        [Service] IMediator mediator,
        CancellationToken ct)
    {
        var command = new PlaceOrderCommand(
            input.CustomerId,
            input.Street,
            input.City,
            input.PostalCode,
            input.Country,
            input.Items.Select(i => new OrderLineRequest(i.ProductId, i.Quantity)).ToList());

        try
        {
            var orderId = await mediator.Send(command, ct);
            return new PlaceOrderPayload(orderId, null);
        }
        catch (DomainException ex)
        {
            return new PlaceOrderPayload(null, new UserError(ex.Message, ex.Code));
        }
    }

    public async Task<CancelOrderPayload> CancelOrderAsync(
        CancelOrderInput input,
        [Service] IMediator mediator,
        CancellationToken ct)
    {
        var command = new CancelOrderCommand(input.OrderId, input.Reason);

        try
        {
            await mediator.Send(command, ct);
            return new CancelOrderPayload(input.OrderId, null);
        }
        catch (DomainException ex)
        {
            return new CancelOrderPayload(null, new UserError(ex.Message, ex.Code));
        }
    }

    public async Task<UpdateProductPayload> UpdateProductPriceAsync(
        UpdateProductPriceInput input,
        AppDbContext db,
        [Service] ITopicEventSender sender,
        CancellationToken ct)
    {
        var product = await db.Products.FindAsync([input.ProductId], ct)
            ?? throw new GraphQLException(
                ErrorBuilder.New()
                    .SetMessage("Product not found.")
                    .SetCode("PRODUCT_NOT_FOUND")
                    .Build());

        var oldPrice = product.Price;
        product.Price = input.NewPrice;
        await db.SaveChangesAsync(ct);

        // Publish to subscription topic
        await sender.SendAsync(
            $"product_price_{input.ProductId}",
            new ProductPriceUpdated(product.Id, oldPrice, product.Price, product.Currency),
            ct);

        return new UpdateProductPayload(product);
    }
}

// Input types
public record PlaceOrderInput(
    [ID(nameof(CustomerRecord))] Guid CustomerId,
    string Street,
    string City,
    string PostalCode,
    string Country,
    IReadOnlyList<OrderItemInput> Items);

public record OrderItemInput(
    [ID(nameof(ProductRecord))] Guid ProductId,
    int Quantity);

public record CancelOrderInput(
    [ID(nameof(OrderRecord))] Guid OrderId,
    string Reason);

public record UpdateProductPriceInput(
    [ID(nameof(ProductRecord))] Guid ProductId,
    decimal NewPrice);

// Payload types with error union pattern
public record PlaceOrderPayload(
    [property: ID(nameof(OrderRecord))] Guid? OrderId,
    UserError? Error);

public record CancelOrderPayload(
    [property: ID(nameof(OrderRecord))] Guid? OrderId,
    UserError? Error);

public record UpdateProductPayload(ProductRecord Product);

public record UserError(string Message, string Code);

public record ProductPriceUpdated(Guid ProductId, decimal OldPrice, decimal NewPrice, string Currency);
```

---

## Step 1820: Subscriptions

```csharp
// GraphQL/Subscriptions/Subscription.cs
[SubscriptionType]
public sealed class Subscription
{
    // Subscribe to order status changes for a specific order
    [Subscribe]
    [Topic("{orderId}")]
    public OrderRecord OnOrderStatusChanged(
        [EventMessage] OrderRecord order,
        [ID(nameof(OrderRecord))] Guid orderId)
        => order;

    // Subscribe to all product price changes
    [Subscribe]
    [Topic("product_price_{productId}")]
    public ProductPriceUpdated OnProductPriceChanged(
        [EventMessage] ProductPriceUpdated update,
        [ID(nameof(ProductRecord))] Guid productId)
        => update;

    // Subscribe to new orders (for admin dashboards)
    [Subscribe]
    [Topic("new_orders")]
    public OrderRecord OnNewOrderPlaced([EventMessage] OrderRecord order)
        => order;
}

// Sending subscription events from the application layer
public sealed class OrderStatusChangedHandler
    : INotificationHandler<OrderSubmittedEvent>
{
    private readonly ITopicEventSender _sender;
    private readonly AppDbContext _db;

    public OrderStatusChangedHandler(ITopicEventSender sender, AppDbContext db)
    {
        _sender = sender;
        _db = db;
    }

    public async Task Handle(OrderSubmittedEvent notification, CancellationToken ct)
    {
        var order = await _db.Orders
            .Include(o => o.Items)
            .Include(o => o.Customer)
            .FirstOrDefaultAsync(o => o.Id == notification.OrderId.Value, ct);

        if (order is null) return;

        // Notify subscriber watching THIS specific order
        await _sender.SendAsync(
            notification.OrderId.Value.ToString(),
            order,
            ct);

        // Notify admin dashboard subscribers
        await _sender.SendAsync("new_orders", order, ct);
    }
}
```

Client subscription (JavaScript):
```graphql
subscription {
  onOrderStatusChanged(orderId: "T3JkZXI6YWJjMTIz") {
    id
    status
    updatedAt
    total {
      amount
      currency
    }
  }
}
```

---

## Step 1821: Persisted Queries

```csharp
// Persisted queries: store query on server, client sends only hash
// Prevents arbitrary queries in production

// Program.cs additions
builder.Services
    .AddGraphQLServer()
    // ... other config ...
    .UsePersistedQueryPipeline()
    .AddFileSystemQueryStorage("./persisted-queries")  // Store queries on disk
    // OR for production:
    .AddRedisQueryStorage(configuration.GetConnectionString("Redis")!);

// For automatic query export during development:
builder.Services
    .AddGraphQLServer()
    .AddInstrumentation(o => o.RenameRootActivity = true)
    .UseDocumentCache()
    .UseDocumentParser()
    .UseDocumentValidation()
    .ExecutePersistedQuery()
    .UseOperationCache()
    .UseOperationComplexityAnalyzer()
    .UseOperationResolver()
    .UseOperationVariableCoercion()
    .UseOperationExecution();
```

```bash
# Extract persisted queries from Relay/Apollo client build
# This prevents clients from sending arbitrary queries to your API
relay-compiler --persist-output ./persisted-queries/relay.json

# queries look like: { "id": "abc123hash", "variables": { ... } }
# Server looks up "abc123hash" → full query document
```

---

## Step 1822: Query Complexity and Depth Limiting

```csharp
// Prevent expensive nested queries like:
// { orders { customer { orders { customer { orders { ... } } } } } }

builder.Services
    .AddGraphQLServer()
    .AddMaxExecutionDepthRule(maxAllowedExecutionDepth: 8)
    .ModifyRequestOptions(opt =>
    {
        opt.MaxAllowedComplexity = 500; // Each field costs 1, nested lists multiply
        opt.Complexity.DefaultComplexity = 1;
        opt.Complexity.DefaultResolverComplexity = 5;
        opt.IncludeExceptionDetails = builder.Environment.IsDevelopment();
    })
    .SetPagingOptions(new PagingOptions
    {
        MaxPageSize = 100,
        DefaultPageSize = 20,
        IncludeTotalCount = true
    });

// Custom complexity calculation
public class OrderComplexityEstimator : IComplexityEstimator
{
    public int? GetComplexity(IComplexityEstimatorContext context)
    {
        if (context.Field.Name == "orders" &&
            context.Field.DeclaringType.Name == "Customer")
        {
            // Orders on Customer is expensive — charge more
            return 10;
        }
        return null; // Use default
    }
}
```

---

## Step 1823: Authorization in GraphQL

```csharp
// Install: dotnet add package HotChocolate.Authorization

// Program.cs
builder.Services
    .AddGraphQLServer()
    .AddAuthorization()  // HC authorization middleware
    // ...

// On types
[GraphQLName("Order")]
[Authorize(Policy = "ReadOrders")] // whole type requires policy
public sealed class OrderType : ObjectType<OrderRecord>
{
    protected override void Configure(IObjectTypeDescriptor<OrderRecord> descriptor)
    {
        // Fields can also have their own auth
        descriptor.Field(o => o.TotalAmount)
            .Authorize("OrderAdmin"); // Only admins see raw total
    }
}

// On mutations
[MutationType]
public sealed class Mutation
{
    [Authorize(Roles = new[] { "admin", "order-manager" })]
    public async Task<CancelOrderPayload> CancelOrderAsync(/* ... */) { /* ... */ }

    [Authorize(Policy = "WriteOrders")]
    public async Task<PlaceOrderPayload> PlaceOrderAsync(/* ... */) { /* ... */ }
}

// On queries
[QueryType]
public sealed class Query
{
    [Authorize] // Just needs to be authenticated
    [UseDbContext(typeof(AppDbContext))]
    [UsePaging]
    public IQueryable<OrderRecord> GetMyOrders(
        AppDbContext db,
        [GlobalState("currentUser")] ClaimsPrincipal user)
    {
        var userId = Guid.Parse(user.FindFirst(ClaimTypes.NameIdentifier)!.Value);
        return db.Orders.Where(o => o.CustomerId == userId);
    }

    [Authorize(Roles = new[] { "admin" })]
    [UseDbContext(typeof(AppDbContext))]
    [UsePaging]
    public IQueryable<OrderRecord> GetAllOrders(AppDbContext db)
        => db.Orders;
}
```

---

## Step 1824: Error Handling

```csharp
// Errors in GraphQL should be handled at two levels:
// 1. Field-level errors (business logic) — return in payload
// 2. Unexpected errors — handled by error filter

// Error filter — translate exceptions to GraphQL errors
public sealed class AppErrorFilter : IErrorFilter
{
    private readonly ILogger<AppErrorFilter> _logger;
    private readonly IHostEnvironment _env;

    public AppErrorFilter(ILogger<AppErrorFilter> logger, IHostEnvironment env)
    {
        _logger = logger;
        _env = env;
    }

    public IError OnError(IError error)
    {
        if (error.Exception is DomainException domainEx)
        {
            return error
                .WithMessage(domainEx.Message)
                .WithCode(domainEx.Code)
                .RemoveExtension("stacktrace");
        }

        if (error.Exception is not null)
        {
            _logger.LogError(error.Exception, "Unhandled GraphQL error: {Message}", error.Message);

            return _env.IsProduction()
                ? error.WithMessage("An unexpected error occurred.").RemoveExtension("stacktrace")
                : error; // Show full details in dev
        }

        return error;
    }
}

// Register
builder.Services
    .AddGraphQLServer()
    .AddErrorFilter<AppErrorFilter>()
    // ...

// Mutation error result (GraphQL Error Union pattern)
// SDL:
// type PlaceOrderPayload {
//   order: Order          # Success case
//   errors: [UserError!]  # Business error case
// }
//
// type UserError {
//   message: String!
//   code: String!
//   field: [String!]      # Which field caused the error
// }

public sealed class PlaceOrderPayload
{
    public PlaceOrderPayload(OrderRecord order)
    {
        Order = order;
        Errors = [];
    }

    public PlaceOrderPayload(IReadOnlyList<UserError> errors)
    {
        Errors = errors;
    }

    public OrderRecord? Order { get; }
    public IReadOnlyList<UserError> Errors { get; }
    [GraphQLIgnore]
    public bool IsSuccess => !Errors.Any();
}
```

---

## Step 1825: Real-Time Dashboard Example

```graphql
# Client query — get initial data
query GetDashboard {
  orders(
    first: 20
    where: { status: { eq: "Submitted" } }
    order: [{ createdAt: DESC }]
  ) {
    totalCount
    nodes {
      id
      status
      createdAt
      total { amount currency }
      customer { fullName email }
      items {
        productName
        quantity
        unitPrice { amount }
      }
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}

# Subscribe to live updates
subscription WatchNewOrders {
  onNewOrderPlaced {
    id
    status
    createdAt
    total { amount currency }
    customer { fullName }
  }
}

# Pagination (load more)
query GetMoreOrders($cursor: String!) {
  orders(
    first: 20
    after: $cursor
    order: [{ createdAt: DESC }]
  ) {
    nodes {
      id
      status
      total { amount currency }
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}

# Mutation with error handling
mutation PlaceOrder($input: PlaceOrderInput!) {
  placeOrder(input: $input) {
    order {
      id
      status
      total { amount currency }
    }
    errors {
      message
      code
    }
  }
}
```

---

## Step 1826: Schema Stitching — Federated Architecture

```csharp
// Gateway service that stitches multiple GraphQL services

// Orders.GraphQL service — exposes its piece
[QueryType]
public sealed class OrderQuery
{
    [NodeResolver]  // Relay node interface
    public static async Task<OrderRecord?> GetOrderNodeAsync(
        Guid id,
        OrderByIdDataLoader dataLoader,
        CancellationToken ct)
        => await dataLoader.LoadAsync(id, ct);

    [UseDbContext(typeof(AppDbContext))]
    [UsePaging]
    [UseFiltering]
    public IQueryable<OrderRecord> GetOrders(AppDbContext db)
        => db.Orders;
}

// Gateway / BFF — stitches Order + Catalog + Customer services
// Program.cs for the gateway
builder.Services
    .AddGraphQLServer()
    .AddRemoteSchema("orders", ignoreRootTypes: false)
    .AddRemoteSchema("catalog", ignoreRootTypes: false)
    .AddRemoteSchema("customers", ignoreRootTypes: false)
    .AddTypeExtensionsFromFile("./stitching.graphql");

// stitching.graphql — extends remote types
// extend type Order {
//   product: Product @delegate(schema: "catalog", path: "product(id: $fields:productId)")
// }
```

---

## Step 1827: Hot Chocolate with EF Core — Best Practices

```csharp
// Use IDbContextFactory for DataLoaders (one context per batch, not per request)
builder.Services.AddDbContextFactory<AppDbContext>(options =>
    options.UseNpgsql(connectionString));

// Projections — let HC optimize the SQL query to only fetch needed columns
[QueryType]
public sealed class Query
{
    [UseDbContext(typeof(AppDbContext))]
    [UsePaging]
    [UseProjection]    // ← HC analyzes the GraphQL query, builds optimal SQL projection
    [UseFiltering]
    [UseSorting]
    public IQueryable<OrderRecord> GetOrders(AppDbContext db)
        => db.Orders;
    // Query: { orders { nodes { id status } } }
    // → SELECT "Id", "Status" FROM "Orders" (only requested columns!)
    //   (not SELECT * FROM "Orders")
}

// Avoid N+1 — always use DataLoader for related entities
// BAD:
[GraphQLName("Order")]
public sealed class OrderType : ObjectType<OrderRecord>
{
    protected override void Configure(IObjectTypeDescriptor<OrderRecord> descriptor)
    {
        // This causes N+1: one DB call per order!
        descriptor.Field("customer")
            .UseDbContext<AppDbContext>()
            .Resolve(async ctx =>
            {
                var order = ctx.Parent<OrderRecord>();
                var db = ctx.DbContext<AppDbContext>();
                return await db.Customers.FindAsync(order.CustomerId); // N+1!
            });
    }
}

// GOOD: use DataLoader
public sealed class OrderResolvers
{
    // One SQL query per batch of customer IDs
    public Task<CustomerRecord?> GetCustomerAsync(
        [Parent] OrderRecord order,
        CustomerByIdDataLoader dataLoader,
        CancellationToken ct)
        => dataLoader.LoadAsync(order.CustomerId, ct);
}
```

---

## Step 1828: Testing GraphQL APIs

```csharp
// Tests/GraphQL/OrderQueryTests.cs
public sealed class OrderQueryTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;

    public OrderQueryTests(WebApplicationFactory<Program> factory)
        => _factory = factory;

    [Fact]
    public async Task GetOrders_Should_Return_Paginated_Results()
    {
        var client = _factory.CreateClient();

        var query = """
            query {
              orders(first: 5) {
                totalCount
                nodes {
                  id
                  status
                  total { amount currency }
                }
                pageInfo {
                  hasNextPage
                  endCursor
                }
              }
            }
            """;

        var response = await client.PostAsJsonAsync("/graphql", new { query });
        var result = await response.Content.ReadFromJsonAsync<GraphQLResponse>();

        response.StatusCode.Should().Be(HttpStatusCode.OK);
        result!.Errors.Should().BeNullOrEmpty();
        result.Data.GetProperty("orders").GetProperty("nodes").GetArrayLength()
            .Should().BeLessThanOrEqualTo(5);
    }

    [Fact]
    public async Task PlaceOrder_Mutation_Should_Return_OrderId()
    {
        var client = _factory.CreateClient();

        var mutation = """
            mutation {
              placeOrder(input: {
                customerId: "Q3VzdG9tZXI6c29tZS1pZA=="
                street: "1 Test St"
                city: "Austin"
                postalCode: "78701"
                country: "US"
                items: [{ productId: "UHJvZHVjdDpzb21lLWlk", quantity: 2 }]
              }) {
                order { id status }
                errors { message code }
              }
            }
            """;

        var response = await client.PostAsJsonAsync("/graphql", new { query = mutation });
        var result = await response.Content.ReadFromJsonAsync<GraphQLResponse>();

        result!.Errors.Should().BeNullOrEmpty();
        var payload = result.Data.GetProperty("placeOrder");
        payload.GetProperty("errors").GetArrayLength().Should().Be(0);
        payload.GetProperty("order").GetProperty("id").GetString().Should().NotBeNullOrEmpty();
    }

    [Fact]
    public async Task GetOrders_InvalidDepth_Should_Return_Error()
    {
        var client = _factory.CreateClient();

        // Exceeds max depth of 8
        var deepQuery = """
            query {
              orders {
                nodes {
                  customer {
                    orders {
                      nodes {
                        customer {
                          orders {
                            nodes { id }
                          }
                        }
                      }
                    }
                  }
                }
              }
            }
            """;

        var response = await client.PostAsJsonAsync("/graphql", new { query = deepQuery });
        var result = await response.Content.ReadFromJsonAsync<GraphQLResponse>();

        result!.Errors.Should().NotBeNullOrEmpty();
        result.Errors!.First().Message.Should().Contain("depth");
    }
}

public record GraphQLResponse(
    JsonElement Data,
    GraphQLError[]? Errors);

public record GraphQLError(string Message, string? Code);
```

---

## Summary: GraphQL with Hot Chocolate

| Feature | Pattern | Key API |
|---|---|---|
| Schema | Code-first | `ObjectType<T>`, `[QueryType]`, `[MutationType]` |
| N+1 Prevention | DataLoader | `BatchDataLoader<K,V>`, `GroupedDataLoader<K,V>` |
| Pagination | Relay cursor | `[UsePaging]`, `Connection<T>` |
| Filtering | Auto-generated | `[UseFiltering]`, `IFilterInputType` |
| Subscriptions | WebSocket | `[SubscriptionType]`, `ITopicEventSender` |
| Authorization | HC auth | `[Authorize(Policy)]`, `IAuthorizationHandler` |
| Complexity | Built-in | `AddMaxExecutionDepthRule`, `MaxAllowedComplexity` |
| Projections | EF integration | `[UseProjection]` → optimized SQL |

**DataLoader is mandatory** for any GraphQL field that loads related data. Every `Customer → Orders` or `Order → Customer` relationship must go through a DataLoader, otherwise your API will execute hundreds of SQL queries for a single GraphQL request.

---

*Next: Part 75 — NuGet Package Publishing: Creating Reusable Libraries, Source Generators as NuGet & Package Governance*
