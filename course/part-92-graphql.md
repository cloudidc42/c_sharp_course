# Part 92: GraphQL with Hot Chocolate

## Steps 2101–2116

---

## Step 2101: GraphQL Fundamentals

```
GraphQL vs REST:
┌──────────────────────────────────────────────────────────┐
│  REST                    │  GraphQL                      │
│  Multiple endpoints      │  Single endpoint /graphql     │
│  Fixed response shape    │  Client specifies fields      │
│  Over/under-fetching     │  Precise data fetching        │
│  N+1 requires batching   │  DataLoader built-in          │
│  Versioning via URL      │  Evolve schema with @deprecated│
│  HTTP verbs              │  Query/Mutation/Subscription  │
└──────────────────────────────────────────────────────────┘

GraphQL Operations:
- Query      → Read data (like GET)
- Mutation   → Write data (like POST/PUT/DELETE)
- Subscription → Real-time updates over WebSocket
```

```bash
dotnet add package HotChocolate.AspNetCore
dotnet add package HotChocolate.Data.EntityFramework
dotnet add package HotChocolate.AspNetCore.Authorization
dotnet add package HotChocolate.Diagnostics   # OpenTelemetry
```

---

## Step 2102: Schema-First vs Code-First — Hot Chocolate Code-First

```csharp
// Domain types
public class Order
{
    public Guid        Id          { get; set; }
    public Guid        CustomerId  { get; set; }
    public string      Status      { get; set; } = null!;
    public decimal     TotalAmount { get; set; }
    public List<OrderItem> Items   { get; set; } = [];
    public DateTimeOffset CreatedAt { get; set; }
}

public class Customer
{
    public Guid   Id       { get; set; }
    public string FullName { get; set; } = null!;
    public string Email    { get; set; } = null!;
    public List<Order> Orders { get; set; } = [];
}

public class OrderItem
{
    public Guid    Id        { get; set; }
    public Guid    ProductId { get; set; }
    public int     Quantity  { get; set; }
    public decimal UnitPrice { get; set; }
}

// GraphQL type extensions
public class OrderType : ObjectType<Order>
{
    protected override void Configure(IObjectTypeDescriptor<Order> descriptor)
    {
        descriptor.Name("Order");
        descriptor.Description("Represents a customer order");

        descriptor.Field(o => o.Id).Type<NonNullType<UuidType>>();
        descriptor.Field(o => o.TotalAmount)
            .Type<NonNullType<DecimalType>>()
            .Name("totalAmount");

        // Computed field
        descriptor.Field("itemCount")
            .Type<NonNullType<IntType>>()
            .Resolve(ctx => ctx.Parent<Order>().Items.Count);

        // Authorize specific fields
        descriptor.Field(o => o.CustomerId)
            .Authorize("admin");
    }
}

// Program.cs
builder.Services
    .AddGraphQLServer()
    .AddQueryType<Query>()
    .AddMutationType<Mutation>()
    .AddSubscriptionType<Subscription>()
    .AddType<OrderType>()
    .AddType<CustomerType>()
    .AddFiltering()
    .AddSorting()
    .AddProjections()
    .AddAuthorization()
    .RegisterDbContext<AppDbContext>(DbContextKind.Pooled)
    .AddInMemorySubscriptions()
    .AddDiagnosticEventListener<OpenTelemetryDiagnosticEventListener>();

var app = builder.Build();
app.MapGraphQL();  // default: /graphql
app.MapGraphQLWebSocket(); // for subscriptions

if (app.Environment.IsDevelopment())
    app.MapBananaCakePop(); // GraphQL IDE at /graphql/ui
```

---

## Step 2103: Query Type — Filters, Sorting & Pagination

```csharp
public class Query
{
    // EF Core integration with filtering/sorting/pagination
    [UseDbContext(typeof(AppDbContext))]
    [UsePaging(IncludeTotalCount = true, MaxPageSize = 100)]
    [UseProjection]
    [UseFiltering]
    [UseSorting]
    public IQueryable<Order> GetOrders([ScopedService] AppDbContext db)
        => db.Orders.AsNoTracking();

    [UseDbContext(typeof(AppDbContext))]
    [UseFirstOrDefault]
    [UseProjection]
    public IQueryable<Order> GetOrderById(Guid id, [ScopedService] AppDbContext db)
        => db.Orders.Where(o => o.Id == id);

    [UseDbContext(typeof(AppDbContext))]
    [UsePaging(IncludeTotalCount = true)]
    [UseProjection]
    [UseFiltering]
    [UseSorting]
    [Authorize]
    public IQueryable<Customer> GetCustomers([ScopedService] AppDbContext db)
        => db.Customers.AsNoTracking();

    // Complex query with data loader (N+1 prevention)
    public async Task<IReadOnlyList<OrderSummary>> GetOrderSummariesAsync(
        [Service] IOrderSummaryService summaryService,
        CancellationToken ct)
        => await summaryService.GetSummariesAsync(ct);
}

/* GraphQL query:
query GetOrders($first: Int, $where: OrderFilterInput, $order: [OrderSortInput!]) {
  orders(first: $first, where: $where, order: $order) {
    nodes {
      id
      status
      totalAmount
      itemCount
      createdAt
    }
    pageInfo {
      hasNextPage
      endCursor
    }
    totalCount
  }
}

Variables:
{
  "first": 10,
  "where": { "status": { "eq": "Pending" }, "totalAmount": { "gte": 100 } },
  "order": [{ "createdAt": "DESC" }]
}
*/
```

---

## Step 2104: Mutation Type

```csharp
public class Mutation
{
    [Authorize]
    [Error<ValidationException>]
    [Error<NotFoundException>]
    public async Task<MutationResult<PlaceOrderPayload>> PlaceOrderAsync(
        PlaceOrderInput input,
        [Service] IOrderService service,
        [Service] IHttpContextAccessor httpCtx,
        CancellationToken ct)
    {
        var customerId = httpCtx.HttpContext!.User.GetUserId();
        var order      = await service.PlaceOrderAsync(customerId, input.Items, ct);

        return new PlaceOrderPayload(order);
    }

    [Authorize]
    [Error<NotFoundException>]
    [Error<InvalidOperationException>]
    public async Task<MutationResult<CancelOrderPayload>> CancelOrderAsync(
        CancelOrderInput input,
        [Service] IOrderService service,
        [Service] IAuthorizationService authz,
        [Service] IHttpContextAccessor httpCtx,
        CancellationToken ct)
    {
        var order = await service.GetByIdAsync(input.OrderId, ct);
        if (order is null) throw new NotFoundException($"Order {input.OrderId} not found");

        var result = await authz.AuthorizeAsync(httpCtx.HttpContext!.User, order, "Cancel");
        if (!result.Succeeded) throw new UnauthorizedAccessException("Not authorized to cancel this order");

        await service.CancelAsync(input.OrderId, input.Reason, ct);

        return new CancelOrderPayload(input.OrderId, "Cancelled");
    }
}

// Input types
public record PlaceOrderInput(
    [property: GraphQLNonNullType] IReadOnlyList<OrderItemInput> Items);

public record OrderItemInput(
    [property: GraphQLNonNullType] Guid ProductId,
    [property: GraphQLNonNullType] int Quantity,
    [property: GraphQLNonNullType] decimal UnitPrice);

public record CancelOrderInput(
    [property: GraphQLNonNullType] Guid OrderId,
    string? Reason);

// Payload types
public record PlaceOrderPayload(Order Order);
public record CancelOrderPayload(Guid OrderId, string Status);

/* GraphQL mutation:
mutation PlaceOrder($input: PlaceOrderInput!) {
  placeOrder(input: $input) {
    order {
      id
      status
      totalAmount
    }
    errors {
      ... on ValidationError {
        message
        field
      }
    }
  }
}
*/
```

---

## Step 2105: Subscriptions — Real-Time Updates

```csharp
public class Subscription
{
    [Subscribe]
    [Topic("{orderId}")]
    public Order OnOrderStatusChanged(
        [EventMessage] Order order,
        Guid orderId) => order;

    [Subscribe]
    [Topic("new-orders")]
    [Authorize(Roles = "admin,support")]
    public Order OnNewOrder([EventMessage] Order order) => order;
}

// Publishing events from a command handler
public class PlaceOrderCommandHandler
{
    private readonly ITopicEventSender _sender;
    private readonly IOrderRepository _repo;

    public PlaceOrderCommandHandler(ITopicEventSender sender, IOrderRepository repo)
    {
        _sender = sender;
        _repo   = repo;
    }

    public async Task<Order> HandleAsync(PlaceOrderCommand cmd, CancellationToken ct)
    {
        var order = await _repo.CreateAsync(cmd, ct);

        // Notify subscribers
        await _sender.SendAsync("new-orders", order, ct);

        return order;
    }
}

// Update status and notify
public class UpdateOrderStatusHandler
{
    private readonly ITopicEventSender _sender;
    private readonly IOrderRepository _repo;

    public UpdateOrderStatusHandler(ITopicEventSender sender, IOrderRepository repo)
    {
        _sender = sender;
        _repo   = repo;
    }

    public async Task HandleAsync(Guid orderId, string newStatus, CancellationToken ct)
    {
        var order = await _repo.UpdateStatusAsync(orderId, newStatus, ct);

        // Topic matches subscription pattern "{orderId}"
        await _sender.SendAsync(orderId.ToString(), order, ct);
    }
}

// Redis pub/sub for distributed subscriptions
builder.Services
    .AddGraphQLServer()
    .AddRedisSubscriptions(sp =>
        sp.GetRequiredService<IConnectionMultiplexer>());

/* GraphQL subscription:
subscription TrackOrder($orderId: UUID!) {
  onOrderStatusChanged(orderId: $orderId) {
    id
    status
    updatedAt
  }
}
*/
```

---

## Step 2106: DataLoader — N+1 Prevention

```csharp
// DataLoader batches multiple loads into a single query
public class CustomerByIdDataLoader : BatchDataLoader<Guid, Customer>
{
    private readonly IDbContextFactory<AppDbContext> _dbFactory;

    public CustomerByIdDataLoader(
        IDbContextFactory<AppDbContext> dbFactory,
        IBatchScheduler batchScheduler,
        DataLoaderOptions options)
        : base(batchScheduler, options)
    {
        _dbFactory = dbFactory;
    }

    protected override async Task<IReadOnlyDictionary<Guid, Customer>> LoadBatchAsync(
        IReadOnlyList<Guid> keys, CancellationToken ct)
    {
        await using var db = await _dbFactory.CreateDbContextAsync(ct);

        return await db.Customers
            .Where(c => keys.Contains(c.Id))
            .ToDictionaryAsync(c => c.Id, ct);
    }
}

// GroupedDataLoader: load multiple items per key
public class OrdersByCustomerDataLoader : GroupedDataLoader<Guid, Order>
{
    private readonly IDbContextFactory<AppDbContext> _dbFactory;

    public OrdersByCustomerDataLoader(
        IDbContextFactory<AppDbContext> dbFactory,
        IBatchScheduler batchScheduler,
        DataLoaderOptions options)
        : base(batchScheduler, options)
    {
        _dbFactory = dbFactory;
    }

    protected override async Task<ILookup<Guid, Order>> LoadGroupedBatchAsync(
        IReadOnlyList<Guid> keys, CancellationToken ct)
    {
        await using var db = await _dbFactory.CreateDbContextAsync(ct);

        var orders = await db.Orders
            .Where(o => keys.Contains(o.CustomerId))
            .ToListAsync(ct);

        return orders.ToLookup(o => o.CustomerId);
    }
}

// Using DataLoader in type resolvers
public class CustomerType : ObjectType<Customer>
{
    protected override void Configure(IObjectTypeDescriptor<Customer> descriptor)
    {
        descriptor.Field("orders")
            .ResolveWith<CustomerResolvers>(r => r.GetOrdersAsync(default!, default!, default!));
    }
}

public class CustomerResolvers
{
    public async Task<IReadOnlyList<Order>> GetOrdersAsync(
        [Parent] Customer customer,
        OrdersByCustomerDataLoader ordersLoader,
        CancellationToken ct)
        => await ordersLoader.LoadAsync(customer.Id, ct);
}

// Register DataLoaders
builder.Services
    .AddGraphQLServer()
    .AddDataLoader<CustomerByIdDataLoader>()
    .AddDataLoader<OrdersByCustomerDataLoader>();
```

---

## Step 2107: Error Handling & Validation

```csharp
// Domain errors as GraphQL union types
[GraphQLDescription("Represents an error")]
public interface IUserError
{
    string Message { get; }
}

public class ValidationError : IUserError
{
    public ValidationError(string message, string field)
    {
        Message = message;
        Field   = field;
    }

    public string Message { get; }
    public string Field   { get; }
}

public class NotFoundError : IUserError
{
    public NotFoundError(string entityType, Guid id)
    {
        Message    = $"{entityType} with ID {id} was not found";
        EntityType = entityType;
        Id         = id;
    }

    public string Message    { get; }
    public string EntityType { get; }
    public Guid   Id         { get; }
}

// Mutation using OneOf result pattern
public class Mutation
{
    public async Task<UpdateOrderResult> UpdateOrderAsync(
        UpdateOrderInput input,
        [Service] IOrderService service,
        CancellationToken ct)
    {
        var validationErrors = ValidateInput(input);
        if (validationErrors.Count > 0)
            return new UpdateOrderResult(validationErrors.First());

        var order = await service.GetByIdAsync(input.OrderId, ct);
        if (order is null)
            return new UpdateOrderResult(new NotFoundError("Order", input.OrderId));

        var updated = await service.UpdateAsync(input, ct);
        return new UpdateOrderResult(updated);
    }

    private static List<ValidationError> ValidateInput(UpdateOrderInput input)
    {
        var errors = new List<ValidationError>();
        if (input.Items.Count == 0)
            errors.Add(new ValidationError("At least one item required", "items"));
        return errors;
    }
}

[UnionType("UpdateOrderResult")]
public class UpdateOrderResult
{
    public Order?         Order           { get; }
    public ValidationError? ValidationError { get; }
    public NotFoundError?   NotFoundError   { get; }

    public UpdateOrderResult(Order order)           => Order = order;
    public UpdateOrderResult(ValidationError error) => ValidationError = error;
    public UpdateOrderResult(NotFoundError error)   => NotFoundError = error;
}

// Global error filter
public class OrderErrorFilter : IErrorFilter
{
    private readonly ILogger<OrderErrorFilter> _logger;

    public OrderErrorFilter(ILogger<OrderErrorFilter> logger) => _logger = logger;

    public IError OnError(IError error)
    {
        if (error.Exception is UnauthorizedAccessException)
            return error.WithCode("UNAUTHORIZED").WithMessage("Access denied");

        if (error.Exception is not null)
        {
            _logger.LogError(error.Exception, "GraphQL error: {Message}", error.Message);
            return error.WithMessage("An internal error occurred").RemoveExtensions();
        }

        return error;
    }
}

builder.Services.AddGraphQLServer()
    .AddErrorFilter<OrderErrorFilter>();
```

---

## Step 2108: Authorization, Persisted Queries & Security

```csharp
// Fine-grained GraphQL authorization
builder.Services
    .AddGraphQLServer()
    .AddAuthorization();

// Type-level authorization
[Authorize]
public class ProtectedQuery
{
    [Authorize(Policy = "admin")]
    public IQueryable<User> GetUsers([ScopedService] AppDbContext db)
        => db.Users.AsNoTracking();
}

// Field-level authorization
public class OrderType : ObjectType<Order>
{
    protected override void Configure(IObjectTypeDescriptor<Order> descriptor)
    {
        descriptor.Field(o => o.CustomerId)
            .Authorize("admin"); // only admins see customer ID

        descriptor.Field(o => o.InternalNotes)
            .Authorize(Roles: ["admin", "support"]);
    }
}

// Persisted queries (prevent arbitrary query attacks)
builder.Services
    .AddGraphQLServer()
    .UsePersistedQueryPipeline()
    .AddReadOnlyFileSystemQueryStorage("./persisted-queries"); // pre-approved queries only

// Request size limits
builder.Services.AddGraphQLServer(maxAllowedRequestSize: 1024 * 1024) // 1 MB
    .ModifyRequestOptions(o =>
    {
        o.IncludeExceptionDetails   = false; // production safety
        o.ExecutionTimeout          = TimeSpan.FromSeconds(30);
    });

// Query complexity limits (anti-DoS)
builder.Services
    .AddGraphQLServer()
    .AddMaxExecutionDepthRule(maxAllowedExecutionDepth: 15)
    .SetMaxAllowedValidationErrors(10)
    .ModifyParserOptions(o =>
    {
        o.MaxAllowedNodes = 1000;
        o.MaxAllowedFields = 500;
    });

// Introspection disable in production
if (!builder.Environment.IsDevelopment())
{
    builder.Services
        .AddGraphQLServer()
        .AllowIntrospection(false);
}
```

---

## Step 2109: Complete GraphQL Setup

```csharp
// Program.cs — production-ready GraphQL
builder.Services
    .AddGraphQLServer()
    // Types
    .AddQueryType<Query>()
    .AddMutationType<Mutation>()
    .AddSubscriptionType<Subscription>()
    // Extensions
    .AddFiltering()
    .AddSorting()
    .AddProjections()
    // Auth
    .AddAuthorization()
    // DataLoaders
    .AddDataLoader<CustomerByIdDataLoader>()
    .AddDataLoader<OrdersByCustomerDataLoader>()
    // Error handling
    .AddErrorFilter<OrderErrorFilter>()
    // DB
    .RegisterDbContext<AppDbContext>(DbContextKind.Pooled)
    // Subscriptions (Redis for distributed)
    .AddRedisSubscriptions(sp => sp.GetRequiredService<IConnectionMultiplexer>())
    // Observability
    .AddDiagnosticEventListener<OpenTelemetryDiagnosticEventListener>()
    // Security
    .AddMaxExecutionDepthRule(15)
    .ModifyRequestOptions(o =>
    {
        o.IncludeExceptionDetails = builder.Environment.IsDevelopment();
        o.ExecutionTimeout        = TimeSpan.FromSeconds(30);
    });

var app = builder.Build();
app.UseAuthentication();
app.UseAuthorization();
app.MapGraphQL("/graphql");
app.MapGraphQLWebSocket("/graphql");

if (app.Environment.IsDevelopment())
    app.MapBananaCakePop("/graphql/ui");

app.Run();
```

**Summary**: Part 92 covers GraphQL fundamentals vs REST, Hot Chocolate setup, code-first schema with ObjectType descriptors, Query type with EF Core filtering/sorting/cursor pagination, Mutation type with union result types and authorization, Subscriptions over WebSocket with Redis pub/sub for distributed scenarios, DataLoader pattern for N+1 prevention (BatchDataLoader + GroupedDataLoader), error handling with IUserError union types, global error filters, persisted queries for security, query complexity limits, introspection control, and complete production GraphQL configuration. Steps 2101–2116 complete.
