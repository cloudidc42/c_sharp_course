# Part 46: Microservices Patterns & Service Communication

## Steps 1321-1360: จาก Monolith สู่ Microservices ระดับโลก

---

## Step 1321: Microservices Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    MICROSERVICES ECOSYSTEM                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Client Apps                                                    │
│  ┌──────┐ ┌──────┐ ┌──────┐                                    │
│  │ Web  │ │Mobile│ │ CLI  │                                    │
│  └──┬───┘ └──┬───┘ └──┬───┘                                    │
│     └────────┴─────────┘                                        │
│              │                                                  │
│  ┌───────────▼──────────────────────────────┐                  │
│  │            API Gateway                   │                  │
│  │  (Rate Limit, Auth, Routing, LB)         │                  │
│  └───┬───────┬───────┬───────┬──────────────┘                  │
│      │       │       │       │                                  │
│  ┌───▼──┐ ┌──▼──┐ ┌──▼──┐ ┌──▼──┐                             │
│  │Order │ │User │ │Prod │ │Notif│  BFF (Backend for Frontend)  │
│  │ Svc  │ │ Svc │ │ Svc │ │ Svc │                             │
│  └───┬──┘ └──┬──┘ └──┬──┘ └──┬──┘                             │
│      │       │       │       │                                  │
│  ┌───▼───────▼───────▼───────▼──────────────┐                  │
│  │         Message Bus (Kafka/RabbitMQ)      │                  │
│  └──────────────────────────────────────────┘                  │
│                                                                 │
│  Cross-Cutting: Distributed Tracing, Service Mesh, Config      │
└─────────────────────────────────────────────────────────────────┘
```

### Decomposition Strategies

```csharp
// Domain-Driven Design Bounded Contexts
// ─────────────────────────────────────
// Order Context    → OrderService
// Catalog Context  → ProductService  
// Identity Context → UserService
// Payment Context  → PaymentService
// Notification Context → NotificationService

// Microservice Template
// dotnet new webapi -n OrderService --use-minimal-api
// Each service: own DB, own repo, own CI/CD pipeline

// Service Contract (shared via NuGet package)
public record CreateOrderRequest(
    Guid CustomerId,
    List<OrderItem> Items,
    string ShippingAddress
);

public record OrderCreatedEvent(
    Guid OrderId,
    Guid CustomerId,
    decimal TotalAmount,
    DateTimeOffset CreatedAt
);
```

---

## Step 1322: API Gateway with YARP

```xml
<!-- ApiGateway.csproj -->
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Yarp.ReverseProxy" Version="2.*" />
    <PackageReference Include="Microsoft.AspNetCore.RateLimiting" Version="*" />
    <PackageReference Include="AspNetCoreRateLimit" Version="5.*" />
  </ItemGroup>
</Project>
```

```csharp
// Program.cs - API Gateway
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"))
    .AddTransforms(ctx =>
    {
        // Add correlation ID to all upstream requests
        ctx.AddRequestTransform(async transformContext =>
        {
            var correlationId = transformContext.HttpContext.TraceIdentifier;
            transformContext.ProxyRequest.Headers.TryAddWithoutValidation(
                "X-Correlation-ID", correlationId);
        });
    });

builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer();

builder.Services.AddRateLimiter(options =>
{
    options.AddPolicy("api", context =>
    {
        var clientId = context.User.FindFirst("client_id")?.Value
            ?? context.Connection.RemoteIpAddress?.ToString()
            ?? "anonymous";
        return RateLimitPartition.GetTokenBucketLimiter(clientId, _ =>
            new TokenBucketRateLimiterOptions
            {
                TokenLimit = 1000,
                ReplenishmentPeriod = TimeSpan.FromMinutes(1),
                TokensPerPeriod = 1000,
                AutoReplenishment = true,
            });
    });
});

var app = builder.Build();
app.UseRateLimiter();
app.UseAuthentication();
app.UseAuthorization();
app.MapReverseProxy();
app.Run();
```

```json
// appsettings.json - YARP Configuration
{
  "ReverseProxy": {
    "Routes": {
      "orders-route": {
        "ClusterId": "orders-cluster",
        "Match": { "Path": "/api/orders/{**catch-all}" },
        "AuthorizationPolicy": "default",
        "RateLimiterPolicy": "api",
        "Metadata": { "ServiceName": "OrderService" }
      },
      "products-route": {
        "ClusterId": "products-cluster",
        "Match": { "Path": "/api/products/{**catch-all}" }
      },
      "users-route": {
        "ClusterId": "users-cluster",
        "Match": { "Path": "/api/users/{**catch-all}" },
        "AuthorizationPolicy": "default"
      }
    },
    "Clusters": {
      "orders-cluster": {
        "LoadBalancingPolicy": "RoundRobin",
        "Destinations": {
          "destination1": { "Address": "http://order-service:8080/" },
          "destination2": { "Address": "http://order-service-2:8080/" }
        },
        "HealthCheck": {
          "Active": {
            "Enabled": true,
            "Interval": "00:00:10",
            "Path": "/health"
          }
        }
      },
      "products-cluster": {
        "Destinations": {
          "destination1": { "Address": "http://product-service:8080/" }
        }
      },
      "users-cluster": {
        "Destinations": {
          "destination1": { "Address": "http://user-service:8080/" }
        }
      }
    }
  }
}
```

```csharp
// Custom Middleware: Request Aggregation
public class RequestAggregationMiddleware(RequestDelegate next)
{
    public async Task InvokeAsync(HttpContext context)
    {
        if (context.Request.Path == "/api/dashboard" && context.Request.Method == "GET")
        {
            await HandleDashboardAggregation(context);
            return;
        }
        await next(context);
    }

    private static async Task HandleDashboardAggregation(HttpContext context)
    {
        var httpClient = context.RequestServices
            .GetRequiredService<IHttpClientFactory>()
            .CreateClient("internal");

        var token = context.Request.Headers.Authorization.ToString();
        httpClient.DefaultRequestHeaders.Authorization =
            new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer",
                token.Replace("Bearer ", ""));

        // Parallel calls to downstream services
        var ordersTask = httpClient.GetFromJsonAsync<object>("http://order-service/api/orders/recent");
        var statsTask = httpClient.GetFromJsonAsync<object>("http://analytics-service/api/stats");
        var notifTask = httpClient.GetFromJsonAsync<object>("http://notification-service/api/unread");

        await Task.WhenAll(ordersTask, statsTask, notifTask);

        await context.Response.WriteAsJsonAsync(new
        {
            orders = ordersTask.Result,
            stats = statsTask.Result,
            notifications = notifTask.Result,
        });
    }
}
```

---

## Step 1323: Backend for Frontend (BFF) Pattern

```csharp
// MobileBff/Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddHttpClient("orders", c =>
    c.BaseAddress = new Uri("http://order-service/"));
builder.Services.AddHttpClient("products", c =>
    c.BaseAddress = new Uri("http://product-service/"));
builder.Services.AddHttpClient("users", c =>
    c.BaseAddress = new Uri("http://user-service/"));

var app = builder.Build();

// Mobile-optimized endpoint: returns only fields mobile needs
app.MapGet("/mobile/home", async (
    IHttpClientFactory factory,
    ClaimsPrincipal user) =>
{
    var ordersClient = factory.CreateClient("orders");
    var productsClient = factory.CreateClient("products");

    var userId = user.FindFirst(ClaimTypes.NameIdentifier)!.Value;

    var recentOrdersTask = ordersClient
        .GetFromJsonAsync<List<OrderDto>>($"api/orders?customerId={userId}&limit=3");
    var featuredTask = productsClient
        .GetFromJsonAsync<List<ProductDto>>("api/products/featured?limit=6");

    await Task.WhenAll(recentOrdersTask, featuredTask);

    // Shape data specifically for mobile UI
    return Results.Ok(new MobileHomeResponse(
        RecentOrders: recentOrdersTask.Result!.Select(o => new MobileOrderSummary(
            o.Id, o.Status, o.TotalAmount, o.CreatedAt)),
        FeaturedProducts: featuredTask.Result!.Select(p => new MobileProductCard(
            p.Id, p.Name, p.Price, p.ThumbnailUrl))
    ));
})
.RequireAuthorization();

app.Run();

public record MobileHomeResponse(
    IEnumerable<MobileOrderSummary> RecentOrders,
    IEnumerable<MobileProductCard> FeaturedProducts
);

public record MobileOrderSummary(Guid Id, string Status, decimal Total, DateTimeOffset Date);
public record MobileProductCard(Guid Id, string Name, decimal Price, string ThumbnailUrl);
```

---

## Step 1324: gRPC Service Communication

```xml
<!-- OrderService.csproj -->
<ItemGroup>
  <PackageReference Include="Grpc.AspNetCore" Version="2.*" />
  <PackageReference Include="Google.Protobuf" Version="3.*" />
  <PackageReference Include="Grpc.Tools" Version="2.*" PrivateAssets="All" />
</ItemGroup>

<ItemGroup>
  <Protobuf Include="Protos\orders.proto" GrpcServices="Server" />
</ItemGroup>
```

```protobuf
// Protos/orders.proto
syntax = "proto3";
option csharp_namespace = "OrderService.Grpc";
package orders;

import "google/protobuf/timestamp.proto";

service OrderGrpc {
  rpc CreateOrder (CreateOrderRequest) returns (OrderResponse);
  rpc GetOrder (GetOrderRequest) returns (OrderResponse);
  rpc ListOrders (ListOrdersRequest) returns (stream OrderResponse);
  rpc UpdateOrderStatus (UpdateStatusRequest) returns (OrderResponse);
  rpc WatchOrderUpdates (WatchRequest) returns (stream OrderEvent);
}

message CreateOrderRequest {
  string customer_id = 1;
  repeated OrderItem items = 2;
  string shipping_address = 3;
}

message OrderItem {
  string product_id = 1;
  int32 quantity = 2;
  double unit_price = 3;
}

message OrderResponse {
  string id = 1;
  string customer_id = 2;
  string status = 3;
  double total_amount = 4;
  google.protobuf.Timestamp created_at = 5;
  repeated OrderItem items = 6;
}

message GetOrderRequest { string id = 1; }
message ListOrdersRequest { string customer_id = 1; int32 page_size = 2; }
message UpdateStatusRequest { string id = 1; string status = 2; }
message WatchRequest { string customer_id = 1; }
message OrderEvent {
  string order_id = 1;
  string event_type = 2;
  google.protobuf.Timestamp occurred_at = 3;
}
```

```csharp
// Services/OrderGrpcService.cs
using Grpc.Core;
using OrderService.Grpc;

public class OrderGrpcService(
    IOrderRepository repository,
    ILogger<OrderGrpcService> logger) : OrderGrpc.OrderGrpcBase
{
    public override async Task<OrderResponse> CreateOrder(
        CreateOrderRequest request,
        ServerCallContext context)
    {
        var order = new Order
        {
            Id = Guid.NewGuid(),
            CustomerId = Guid.Parse(request.CustomerId),
            Items = request.Items.Select(i => new OrderItem
            {
                ProductId = Guid.Parse(i.ProductId),
                Quantity = i.Quantity,
                UnitPrice = (decimal)i.UnitPrice,
            }).ToList(),
            ShippingAddress = request.ShippingAddress,
            Status = OrderStatus.Pending,
            CreatedAt = DateTimeOffset.UtcNow,
        };

        order.TotalAmount = order.Items.Sum(i => i.Quantity * i.UnitPrice);
        await repository.AddAsync(order, context.CancellationToken);

        logger.LogInformation("Order {OrderId} created via gRPC", order.Id);
        return MapToResponse(order);
    }

    public override async Task ListOrders(
        ListOrdersRequest request,
        IServerStreamWriter<OrderResponse> responseStream,
        ServerCallContext context)
    {
        var customerId = Guid.Parse(request.CustomerId);
        await foreach (var order in repository.StreamByCustomerAsync(
            customerId, context.CancellationToken))
        {
            await responseStream.WriteAsync(MapToResponse(order));
        }
    }

    // Server-side streaming for real-time order updates
    public override async Task WatchOrderUpdates(
        WatchRequest request,
        IServerStreamWriter<OrderEvent> responseStream,
        ServerCallContext context)
    {
        var eventChannel = context.RequestServices
            .GetRequiredService<IOrderEventChannel>();

        await foreach (var evt in eventChannel.ReadAllAsync(
            Guid.Parse(request.CustomerId), context.CancellationToken))
        {
            await responseStream.WriteAsync(new OrderEvent
            {
                OrderId = evt.OrderId.ToString(),
                EventType = evt.Type,
                OccurredAt = Google.Protobuf.WellKnownTypes.Timestamp.FromDateTimeOffset(evt.OccurredAt),
            });
        }
    }

    private static OrderResponse MapToResponse(Order order) => new()
    {
        Id = order.Id.ToString(),
        CustomerId = order.CustomerId.ToString(),
        Status = order.Status.ToString(),
        TotalAmount = (double)order.TotalAmount,
        CreatedAt = Google.Protobuf.WellKnownTypes.Timestamp.FromDateTimeOffset(order.CreatedAt),
        Items = { order.Items.Select(i => new Grpc.OrderItem
        {
            ProductId = i.ProductId.ToString(),
            Quantity = i.Quantity,
            UnitPrice = (double)i.UnitPrice,
        })},
    };
}
```

```csharp
// gRPC Client in another service
// InventoryService.csproj references OrderService.Grpc NuGet

public class OrderGrpcClient(GrpcChannel channel)
{
    private readonly OrderGrpc.OrderGrpcClient _client = new(channel);

    public async Task<OrderDto?> GetOrderAsync(Guid orderId, CancellationToken ct = default)
    {
        try
        {
            var response = await _client.GetOrderAsync(
                new GetOrderRequest { Id = orderId.ToString() },
                cancellationToken: ct);
            return new OrderDto(Guid.Parse(response.Id), response.Status, (decimal)response.TotalAmount);
        }
        catch (RpcException ex) when (ex.StatusCode == StatusCode.NotFound)
        {
            return null;
        }
    }

    public async IAsyncEnumerable<OrderDto> StreamCustomerOrdersAsync(
        Guid customerId,
        [EnumeratorCancellation] CancellationToken ct = default)
    {
        using var call = _client.ListOrders(
            new ListOrdersRequest { CustomerId = customerId.ToString(), PageSize = 100 });

        await foreach (var response in call.ResponseStream.ReadAllAsync(ct))
        {
            yield return new OrderDto(Guid.Parse(response.Id), response.Status, (decimal)response.TotalAmount);
        }
    }
}

// Registration
builder.Services.AddGrpcClient<OrderGrpc.OrderGrpcClient>(options =>
{
    options.Address = new Uri(builder.Configuration["Services:OrderService:GrpcUrl"]!);
})
.AddInterceptor<TracingInterceptor>()
.AddCallCredentials((context, metadata, serviceProvider) =>
{
    var tokenProvider = serviceProvider.GetRequiredService<ITokenProvider>();
    metadata.Add("Authorization", $"Bearer {tokenProvider.GetToken()}");
    return Task.CompletedTask;
});
```

```csharp
// gRPC Interceptor for tracing
public class TracingInterceptor(ILogger<TracingInterceptor> logger) : Interceptor
{
    public override async Task<TResponse> UnaryServerHandler<TRequest, TResponse>(
        TRequest request,
        ServerCallContext context,
        UnaryServerMethod<TRequest, TResponse> continuation)
    {
        using var activity = Activity.Current?.Source.StartActivity(
            $"grpc.{context.Method}",
            ActivityKind.Server);

        activity?.SetTag("rpc.system", "grpc");
        activity?.SetTag("rpc.method", context.Method);

        try
        {
            var response = await continuation(request, context);
            activity?.SetStatus(ActivityStatusCode.Ok);
            return response;
        }
        catch (Exception ex)
        {
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            logger.LogError(ex, "gRPC call {Method} failed", context.Method);
            throw;
        }
    }
}
```

---

## Step 1325: GraphQL with Hot Chocolate

```xml
<ItemGroup>
  <PackageReference Include="HotChocolate.AspNetCore" Version="14.*" />
  <PackageReference Include="HotChocolate.Data.EntityFramework" Version="14.*" />
  <PackageReference Include="HotChocolate.Subscriptions.InMemory" Version="14.*" />
</ItemGroup>
```

```csharp
// Program.cs - GraphQL setup
builder.Services
    .AddGraphQLServer()
    .AddQueryType<Query>()
    .AddMutationType<Mutation>()
    .AddSubscriptionType<Subscription>()
    .AddType<OrderType>()
    .AddType<ProductType>()
    .AddType<UserType>()
    .AddProjections()
    .AddFiltering()
    .AddSorting()
    .AddPagingArguments()
    .AddInMemorySubscriptions()
    .AddAuthorization();

app.MapGraphQL();
app.MapGraphQLWebSocket();
```

```csharp
// GraphQL Types
public class OrderType : ObjectType<Order>
{
    protected override void Configure(IObjectTypeDescriptor<Order> descriptor)
    {
        descriptor.Name("Order");
        descriptor.Field(o => o.Id).Type<NonNullType<IdType>>();
        descriptor.Field(o => o.Status).Type<NonNullType<StringType>>();
        descriptor.Field(o => o.TotalAmount).Type<NonNullType<DecimalType>>();
        descriptor.Field(o => o.CreatedAt).Type<NonNullType<DateTimeType>>();

        // Resolver for nested customer (N+1 → DataLoader)
        descriptor
            .Field("customer")
            .Type<NonNullType<UserType>>()
            .ResolveWith<OrderResolvers>(r => r.GetCustomerAsync(default!, default!, default!));
    }
}

public class OrderResolvers
{
    public async Task<User?> GetCustomerAsync(
        [Parent] Order order,
        UserDataLoader userLoader,
        CancellationToken ct)
        => await userLoader.LoadAsync(order.CustomerId, ct);
}

// DataLoader prevents N+1 problem
public class UserDataLoader(IBatchScheduler scheduler, IUserRepository repo)
    : BatchDataLoader<Guid, User>(scheduler)
{
    protected override async Task<IReadOnlyDictionary<Guid, User>> LoadBatchAsync(
        IReadOnlyList<Guid> keys,
        CancellationToken ct)
    {
        var users = await repo.GetByIdsAsync(keys, ct);
        return users.ToDictionary(u => u.Id);
    }
}
```

```csharp
// Query type
public class Query
{
    [UseProjection]
    [UseFiltering]
    [UseSorting]
    [UsePaging(IncludeTotalCount = true)]
    public IQueryable<Order> GetOrders([Service] AppDbContext db)
        => db.Orders.AsNoTracking();

    [GraphQLNonNull]
    public async Task<Order?> GetOrder(
        Guid id,
        [Service] IOrderRepository repo,
        CancellationToken ct)
        => await repo.GetByIdAsync(id, ct);

    [Authorize]
    [UseProjection]
    [UsePaging]
    public IQueryable<Order> GetMyOrders(
        ClaimsPrincipal user,
        [Service] AppDbContext db)
    {
        var userId = Guid.Parse(user.FindFirst(ClaimTypes.NameIdentifier)!.Value);
        return db.Orders.Where(o => o.CustomerId == userId).AsNoTracking();
    }
}

// Mutation type
public class Mutation
{
    [Authorize]
    [Error<ValidationException>]
    [Error<InsufficientInventoryException>]
    public async Task<MutationResult<Order>> CreateOrderAsync(
        CreateOrderInput input,
        [Service] IOrderService orderService,
        [Service] ITopicEventSender eventSender,
        ClaimsPrincipal user,
        CancellationToken ct)
    {
        var customerId = Guid.Parse(user.FindFirst(ClaimTypes.NameIdentifier)!.Value);
        var order = await orderService.CreateAsync(customerId, input, ct);

        // Publish to GraphQL subscription
        await eventSender.SendAsync(
            $"ORDER_UPDATES_{customerId}",
            order, ct);

        return order;
    }
}

// Subscription type
public class Subscription
{
    [Subscribe]
    [Topic("ORDER_UPDATES_{customerId}")]
    [Authorize]
    public Order OnOrderUpdate(
        Guid customerId,
        [EventMessage] Order order)
        => order;
}

// Input types
public record CreateOrderInput(
    List<OrderItemInput> Items,
    string ShippingAddress
);

public record OrderItemInput(Guid ProductId, int Quantity);
```

---

## Step 1326: Saga Pattern — Choreography

```csharp
// Choreography Saga: events drive state machine
// Order Flow: OrderPlaced → InventoryReserved → PaymentProcessed → OrderConfirmed
//             OR InventoryFailed/PaymentFailed → OrderCancelled (compensating)

// OrderService publishes
public class OrderPlacedEventPublisher(IEventBus eventBus)
{
    public async Task PublishAsync(Order order, CancellationToken ct)
    {
        await eventBus.PublishAsync(new OrderPlacedEvent(
            OrderId: order.Id,
            CustomerId: order.CustomerId,
            Items: order.Items.Select(i => new OrderItemData(i.ProductId, i.Quantity, i.UnitPrice)).ToList(),
            TotalAmount: order.TotalAmount,
            OccurredAt: DateTimeOffset.UtcNow
        ), ct);
    }
}

// InventoryService listens
public class OrderPlacedHandler(
    IInventoryRepository inventory,
    IEventBus eventBus,
    ILogger<OrderPlacedHandler> logger) : IEventHandler<OrderPlacedEvent>
{
    public async Task HandleAsync(OrderPlacedEvent evt, CancellationToken ct)
    {
        logger.LogInformation("Reserving inventory for order {OrderId}", evt.OrderId);

        try
        {
            // Check and reserve atomically
            foreach (var item in evt.Items)
            {
                var available = await inventory.GetAvailableQuantityAsync(item.ProductId, ct);
                if (available < item.Quantity)
                    throw new InsufficientInventoryException(item.ProductId, item.Quantity, available);
            }

            // Reserve all items
            foreach (var item in evt.Items)
                await inventory.ReserveAsync(item.ProductId, item.Quantity, evt.OrderId, ct);

            await eventBus.PublishAsync(new InventoryReservedEvent(
                evt.OrderId, evt.CustomerId, evt.TotalAmount, DateTimeOffset.UtcNow), ct);
        }
        catch (InsufficientInventoryException ex)
        {
            logger.LogWarning("Inventory reservation failed for order {OrderId}: {Reason}",
                evt.OrderId, ex.Message);

            await eventBus.PublishAsync(new InventoryReservationFailedEvent(
                evt.OrderId, ex.Message, DateTimeOffset.UtcNow), ct);
        }
    }
}

// PaymentService listens to InventoryReserved
public class InventoryReservedHandler(
    IPaymentGateway gateway,
    IEventBus eventBus,
    ILogger<InventoryReservedHandler> logger) : IEventHandler<InventoryReservedEvent>
{
    public async Task HandleAsync(InventoryReservedEvent evt, CancellationToken ct)
    {
        try
        {
            var result = await gateway.ChargeAsync(evt.CustomerId, evt.TotalAmount, evt.OrderId, ct);
            if (result.Success)
            {
                await eventBus.PublishAsync(new PaymentProcessedEvent(
                    evt.OrderId, result.TransactionId, DateTimeOffset.UtcNow), ct);
            }
            else
            {
                await eventBus.PublishAsync(new PaymentFailedEvent(
                    evt.OrderId, result.FailureReason, DateTimeOffset.UtcNow), ct);
            }
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "Payment processing failed for order {OrderId}", evt.OrderId);
            await eventBus.PublishAsync(new PaymentFailedEvent(
                evt.OrderId, ex.Message, DateTimeOffset.UtcNow), ct);
        }
    }
}

// OrderService listens to PaymentFailed → compensate
public class PaymentFailedHandler(
    IOrderRepository orders,
    IEventBus eventBus) : IEventHandler<PaymentFailedEvent>
{
    public async Task HandleAsync(PaymentFailedEvent evt, CancellationToken ct)
    {
        var order = await orders.GetByIdAsync(evt.OrderId, ct)
            ?? throw new InvalidOperationException($"Order {evt.OrderId} not found");

        order.Status = OrderStatus.Cancelled;
        order.CancellationReason = $"Payment failed: {evt.Reason}";
        await orders.UpdateAsync(order, ct);

        // Trigger compensating transaction to release inventory
        await eventBus.PublishAsync(new ReleaseInventoryCommand(
            evt.OrderId, DateTimeOffset.UtcNow), ct);
    }
}
```

---

## Step 1327: Saga Pattern — Orchestration with MassTransit

```xml
<ItemGroup>
  <PackageReference Include="MassTransit" Version="8.*" />
  <PackageReference Include="MassTransit.RabbitMQ" Version="8.*" />
  <PackageReference Include="MassTransit.EntityFrameworkCore" Version="8.*" />
</ItemGroup>
```

```csharp
// Orchestration Saga: central coordinator
public class OrderSaga : MassTransitStateMachine<OrderSagaState>
{
    public State WaitingForInventory { get; private set; } = null!;
    public State WaitingForPayment { get; private set; } = null!;
    public State Completed { get; private set; } = null!;
    public State Cancelled { get; private set; } = null!;

    public Event<OrderPlacedEvent> OrderPlaced { get; private set; } = null!;
    public Event<InventoryReservedEvent> InventoryReserved { get; private set; } = null!;
    public Event<InventoryReservationFailedEvent> InventoryFailed { get; private set; } = null!;
    public Event<PaymentProcessedEvent> PaymentProcessed { get; private set; } = null!;
    public Event<PaymentFailedEvent> PaymentFailed { get; private set; } = null!;

    public Schedule<OrderSagaState, OrderTimeoutExpired> OrderTimeout { get; private set; } = null!;

    public OrderSaga()
    {
        InstanceState(s => s.CurrentState);

        Schedule(() => OrderTimeout, s => s.TimeoutTokenId, schedule =>
        {
            schedule.Delay = TimeSpan.FromMinutes(30);
            schedule.Received = r => r.CorrelateById(ctx => ctx.Message.OrderId);
        });

        Event(() => OrderPlaced, e => e.CorrelateById(ctx => ctx.Message.OrderId));
        Event(() => InventoryReserved, e => e.CorrelateById(ctx => ctx.Message.OrderId));
        Event(() => InventoryFailed, e => e.CorrelateById(ctx => ctx.Message.OrderId));
        Event(() => PaymentProcessed, e => e.CorrelateById(ctx => ctx.Message.OrderId));
        Event(() => PaymentFailed, e => e.CorrelateById(ctx => ctx.Message.OrderId));

        Initially(
            When(OrderPlaced)
                .Then(ctx =>
                {
                    ctx.Saga.OrderId = ctx.Message.OrderId;
                    ctx.Saga.CustomerId = ctx.Message.CustomerId;
                    ctx.Saga.TotalAmount = ctx.Message.TotalAmount;
                })
                .Schedule(OrderTimeout, ctx => new OrderTimeoutExpired { OrderId = ctx.Saga.OrderId })
                .Publish(ctx => new ReserveInventoryCommand(ctx.Saga.OrderId, ctx.Message.Items))
                .TransitionTo(WaitingForInventory)
        );

        During(WaitingForInventory,
            When(InventoryReserved)
                .Publish(ctx => new ProcessPaymentCommand(
                    ctx.Saga.OrderId, ctx.Saga.CustomerId, ctx.Saga.TotalAmount))
                .TransitionTo(WaitingForPayment),

            When(InventoryFailed)
                .Then(ctx => ctx.Saga.FailureReason = ctx.Message.Reason)
                .Publish(ctx => new CancelOrderCommand(ctx.Saga.OrderId, ctx.Saga.FailureReason))
                .TransitionTo(Cancelled)
                .Finalize()
        );

        During(WaitingForPayment,
            When(PaymentProcessed)
                .Unschedule(OrderTimeout)
                .Publish(ctx => new ConfirmOrderCommand(ctx.Saga.OrderId))
                .TransitionTo(Completed)
                .Finalize(),

            When(PaymentFailed)
                .Then(ctx => ctx.Saga.FailureReason = ctx.Message.Reason)
                .Publish(ctx => new ReleaseInventoryCommand(ctx.Saga.OrderId))
                .Publish(ctx => new CancelOrderCommand(ctx.Saga.OrderId, ctx.Saga.FailureReason))
                .TransitionTo(Cancelled)
                .Finalize()
        );

        // Timeout compensation
        DuringAny(
            When(OrderTimeout!.Received)
                .Publish(ctx => new CancelOrderCommand(ctx.Saga.OrderId, "Order timeout"))
                .TransitionTo(Cancelled)
                .Finalize()
        );

        SetCompletedWhenFinalized();
    }
}

public class OrderSagaState : SagaStateMachineInstance
{
    public Guid CorrelationId { get; set; }
    public string CurrentState { get; set; } = string.Empty;
    public Guid OrderId { get; set; }
    public Guid CustomerId { get; set; }
    public decimal TotalAmount { get; set; }
    public string? FailureReason { get; set; }
    public Guid? TimeoutTokenId { get; set; }
}

// Registration
builder.Services.AddMassTransit(x =>
{
    x.AddSagaStateMachine<OrderSaga, OrderSagaState>()
        .EntityFrameworkRepository(r =>
        {
            r.ConcurrencyMode = ConcurrencyMode.Optimistic;
            r.AddDbContext<SagaDbContext>((provider, optionsBuilder) =>
                optionsBuilder.UseNpgsql(connectionString));
        });

    x.UsingRabbitMq((ctx, cfg) =>
    {
        cfg.Host(rabbitmqHost);
        cfg.ConfigureEndpoints(ctx);
    });
});
```

---

## Step 1328: Event Sourcing

```csharp
// Domain Events
public abstract record DomainEvent(Guid AggregateId, int Version, DateTimeOffset OccurredAt);

public record OrderCreated(
    Guid AggregateId, int Version, DateTimeOffset OccurredAt,
    Guid CustomerId, List<OrderItem> Items, string ShippingAddress
) : DomainEvent(AggregateId, Version, OccurredAt);

public record OrderStatusChanged(
    Guid AggregateId, int Version, DateTimeOffset OccurredAt,
    string OldStatus, string NewStatus
) : DomainEvent(AggregateId, Version, OccurredAt);

public record OrderItemAdded(
    Guid AggregateId, int Version, DateTimeOffset OccurredAt,
    Guid ProductId, int Quantity, decimal UnitPrice
) : DomainEvent(AggregateId, Version, OccurredAt);

// Event-sourced aggregate
public class OrderAggregate
{
    public Guid Id { get; private set; }
    public Guid CustomerId { get; private set; }
    public string Status { get; private set; } = "Draft";
    public List<OrderItem> Items { get; private set; } = [];
    public string ShippingAddress { get; private set; } = string.Empty;
    public int Version { get; private set; }

    private readonly List<DomainEvent> _uncommittedEvents = [];
    public IReadOnlyList<DomainEvent> UncommittedEvents => _uncommittedEvents;

    // Factory: create new order
    public static OrderAggregate Create(Guid customerId, string shippingAddress)
    {
        var aggregate = new OrderAggregate();
        aggregate.Apply(new OrderCreated(
            Guid.NewGuid(), 1, DateTimeOffset.UtcNow,
            customerId, [], shippingAddress));
        return aggregate;
    }

    // Rehydrate from event stream
    public static OrderAggregate Rehydrate(IEnumerable<DomainEvent> events)
    {
        var aggregate = new OrderAggregate();
        foreach (var @event in events)
            aggregate.ApplyEvent(@event, isNew: false);
        return aggregate;
    }

    public void AddItem(Guid productId, int quantity, decimal unitPrice)
    {
        if (Status != "Draft")
            throw new InvalidOperationException("Can only add items to draft orders");

        Apply(new OrderItemAdded(Id, Version + 1, DateTimeOffset.UtcNow,
            productId, quantity, unitPrice));
    }

    public void Submit()
    {
        if (Status != "Draft")
            throw new InvalidOperationException("Order already submitted");
        if (!Items.Any())
            throw new InvalidOperationException("Cannot submit empty order");

        Apply(new OrderStatusChanged(Id, Version + 1, DateTimeOffset.UtcNow, Status, "Pending"));
    }

    private void Apply(DomainEvent @event)
    {
        ApplyEvent(@event, isNew: true);
        if (@event is { } newEvent)
            _uncommittedEvents.Add(newEvent);
    }

    private void ApplyEvent(DomainEvent @event, bool isNew)
    {
        Version = @event.Version;
        switch (@event)
        {
            case OrderCreated e:
                Id = e.AggregateId;
                CustomerId = e.CustomerId;
                Items = e.Items.ToList();
                ShippingAddress = e.ShippingAddress;
                Status = "Draft";
                break;

            case OrderItemAdded e:
                Items.Add(new OrderItem
                {
                    ProductId = e.ProductId,
                    Quantity = e.Quantity,
                    UnitPrice = e.UnitPrice,
                });
                break;

            case OrderStatusChanged e:
                Status = e.NewStatus;
                break;
        }
    }

    public void ClearUncommittedEvents() => _uncommittedEvents.Clear();
}

// Event Store
public interface IEventStore
{
    Task AppendEventsAsync(Guid aggregateId, int expectedVersion, IEnumerable<DomainEvent> events, CancellationToken ct);
    Task<List<DomainEvent>> GetEventsAsync(Guid aggregateId, CancellationToken ct);
    Task<List<DomainEvent>> GetEventsFromVersionAsync(Guid aggregateId, int fromVersion, CancellationToken ct);
}

public class PostgresEventStore(NpgsqlDataSource dataSource, IEventSerializer serializer) : IEventStore
{
    public async Task AppendEventsAsync(
        Guid aggregateId, int expectedVersion,
        IEnumerable<DomainEvent> events, CancellationToken ct)
    {
        await using var conn = await dataSource.OpenConnectionAsync(ct);
        await using var tx = await conn.BeginTransactionAsync(ct);

        // Optimistic concurrency check
        var currentVersion = await conn.ExecuteScalarAsync<int>(
            "SELECT COALESCE(MAX(version), 0) FROM events WHERE aggregate_id = @id",
            new { id = aggregateId });

        if (currentVersion != expectedVersion)
            throw new OptimisticConcurrencyException(aggregateId, expectedVersion, currentVersion);

        foreach (var @event in events)
        {
            await conn.ExecuteAsync(
                """
                INSERT INTO events (aggregate_id, version, event_type, data, occurred_at)
                VALUES (@AggregateId, @Version, @EventType, @Data::jsonb, @OccurredAt)
                """,
                new
                {
                    @event.AggregateId,
                    @event.Version,
                    EventType = @event.GetType().Name,
                    Data = serializer.Serialize(@event),
                    @event.OccurredAt,
                });
        }

        await tx.CommitAsync(ct);
    }

    public async Task<List<DomainEvent>> GetEventsAsync(Guid aggregateId, CancellationToken ct)
    {
        await using var conn = await dataSource.OpenConnectionAsync(ct);
        var rows = await conn.QueryAsync<EventRow>(
            "SELECT event_type, data FROM events WHERE aggregate_id = @id ORDER BY version",
            new { id = aggregateId });

        return rows.Select(r => serializer.Deserialize(r.EventType, r.Data)).ToList();
    }
}

// Repository using event store
public class OrderAggregateRepository(IEventStore eventStore, IEventBus eventBus)
{
    public async Task<OrderAggregate?> GetByIdAsync(Guid id, CancellationToken ct)
    {
        var events = await eventStore.GetEventsAsync(id, ct);
        return events.Count == 0 ? null : OrderAggregate.Rehydrate(events);
    }

    public async Task SaveAsync(OrderAggregate aggregate, CancellationToken ct)
    {
        var uncommitted = aggregate.UncommittedEvents;
        if (!uncommitted.Any()) return;

        var expectedVersion = aggregate.Version - uncommitted.Count;
        await eventStore.AppendEventsAsync(aggregate.Id, expectedVersion, uncommitted, ct);

        // Publish domain events to message bus
        foreach (var @event in uncommitted)
            await eventBus.PublishAsync(@event, ct);

        aggregate.ClearUncommittedEvents();
    }
}
```

---

## Step 1329: CQRS with Projections

```csharp
// Commands go to aggregate, Queries go to read model
// Read model is built by projecting events

// Write side
public class CreateOrderCommandHandler(OrderAggregateRepository repository) 
    : IRequestHandler<CreateOrderCommand, Guid>
{
    public async Task<Guid> Handle(CreateOrderCommand request, CancellationToken ct)
    {
        var order = OrderAggregate.Create(request.CustomerId, request.ShippingAddress);
        foreach (var item in request.Items)
            order.AddItem(item.ProductId, item.Quantity, item.UnitPrice);
        order.Submit();

        await repository.SaveAsync(order, ct);
        return order.Id;
    }
}

// Read model (projection)
public class OrderReadModel
{
    public Guid Id { get; set; }
    public Guid CustomerId { get; set; }
    public string CustomerName { get; set; } = string.Empty;
    public string Status { get; set; } = string.Empty;
    public decimal TotalAmount { get; set; }
    public int ItemCount { get; set; }
    public DateTimeOffset CreatedAt { get; set; }
    public DateTimeOffset? LastUpdatedAt { get; set; }
}

// Projection: rebuilds read model from events
public class OrderProjection(AppReadDbContext db) :
    IEventHandler<OrderCreated>,
    IEventHandler<OrderItemAdded>,
    IEventHandler<OrderStatusChanged>
{
    public async Task HandleAsync(OrderCreated evt, CancellationToken ct)
    {
        var readModel = new OrderReadModel
        {
            Id = evt.AggregateId,
            CustomerId = evt.CustomerId,
            Status = "Draft",
            TotalAmount = 0,
            ItemCount = 0,
            CreatedAt = evt.OccurredAt,
        };
        db.Orders.Add(readModel);
        await db.SaveChangesAsync(ct);
    }

    public async Task HandleAsync(OrderItemAdded evt, CancellationToken ct)
    {
        var order = await db.Orders.FindAsync([evt.AggregateId], ct)
            ?? throw new InvalidOperationException($"Order {evt.AggregateId} not found in read model");

        order.ItemCount++;
        order.TotalAmount += evt.Quantity * evt.UnitPrice;
        order.LastUpdatedAt = evt.OccurredAt;
        await db.SaveChangesAsync(ct);
    }

    public async Task HandleAsync(OrderStatusChanged evt, CancellationToken ct)
    {
        var order = await db.Orders.FindAsync([evt.AggregateId], ct)
            ?? throw new InvalidOperationException($"Order {evt.AggregateId} not found in read model");

        order.Status = evt.NewStatus;
        order.LastUpdatedAt = evt.OccurredAt;
        await db.SaveChangesAsync(ct);
    }
}

// Query side — fast reads from denormalized read model
public class GetOrderSummaryQueryHandler(AppReadDbContext db)
    : IRequestHandler<GetOrderSummaryQuery, List<OrderSummaryDto>>
{
    public async Task<List<OrderSummaryDto>> Handle(
        GetOrderSummaryQuery request, CancellationToken ct)
    {
        return await db.Orders
            .Where(o => o.CustomerId == request.CustomerId)
            .Where(o => request.Status == null || o.Status == request.Status)
            .OrderByDescending(o => o.CreatedAt)
            .Take(request.Limit)
            .Select(o => new OrderSummaryDto(
                o.Id, o.Status, o.TotalAmount, o.ItemCount, o.CreatedAt))
            .ToListAsync(ct);
    }
}
```

---

## Step 1330: Outbox Pattern — Reliable Event Publishing

```csharp
// Transactional Outbox: DB write + event in one transaction
public class OutboxMessage
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public string EventType { get; set; } = string.Empty;
    public string Payload { get; set; } = string.Empty;
    public DateTimeOffset CreatedAt { get; set; } = DateTimeOffset.UtcNow;
    public DateTimeOffset? ProcessedAt { get; set; }
    public int RetryCount { get; set; }
    public string? Error { get; set; }
}

// Write order + outbox message atomically
public class TransactionalOrderService(
    AppDbContext db,
    IEventSerializer serializer,
    ILogger<TransactionalOrderService> logger)
{
    public async Task<Order> CreateOrderAsync(CreateOrderRequest request, CancellationToken ct)
    {
        await using var tx = await db.Database.BeginTransactionAsync(ct);

        var order = new Order
        {
            Id = Guid.NewGuid(),
            CustomerId = request.CustomerId,
            Items = request.Items.Select(i => new OrderItem
            {
                ProductId = i.ProductId,
                Quantity = i.Quantity,
                UnitPrice = i.UnitPrice,
            }).ToList(),
            Status = OrderStatus.Pending,
            CreatedAt = DateTimeOffset.UtcNow,
        };
        order.TotalAmount = order.Items.Sum(i => i.Quantity * i.UnitPrice);
        db.Orders.Add(order);

        // Write outbox message in same transaction
        var outboxMsg = new OutboxMessage
        {
            EventType = nameof(OrderCreatedEvent),
            Payload = serializer.Serialize(new OrderCreatedEvent(
                order.Id, order.CustomerId, order.TotalAmount, order.CreatedAt)),
        };
        db.OutboxMessages.Add(outboxMsg);

        await db.SaveChangesAsync(ct);
        await tx.CommitAsync(ct);

        logger.LogInformation("Order {OrderId} created with outbox message {MessageId}",
            order.Id, outboxMsg.Id);
        return order;
    }
}

// Outbox Relay: polls and publishes pending messages
public class OutboxRelayWorker(
    IServiceProvider services,
    IEventBus eventBus,
    IEventSerializer serializer,
    ILogger<OutboxRelayWorker> logger) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await ProcessBatchAsync(stoppingToken);
            await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
        }
    }

    private async Task ProcessBatchAsync(CancellationToken ct)
    {
        using var scope = services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();

        // Fetch unprocessed messages with advisory lock (PostgreSQL)
        var messages = await db.OutboxMessages
            .Where(m => m.ProcessedAt == null && m.RetryCount < 5)
            .OrderBy(m => m.CreatedAt)
            .Take(50)
            .ToListAsync(ct);

        foreach (var message in messages)
        {
            try
            {
                var @event = serializer.Deserialize(message.EventType, message.Payload);
                await eventBus.PublishAsync(@event, ct);

                message.ProcessedAt = DateTimeOffset.UtcNow;
                logger.LogDebug("Outbox message {Id} published", message.Id);
            }
            catch (Exception ex)
            {
                message.RetryCount++;
                message.Error = ex.Message;
                logger.LogWarning(ex, "Failed to publish outbox message {Id} (attempt {Retry})",
                    message.Id, message.RetryCount);
            }
        }

        await db.SaveChangesAsync(ct);
    }
}
```

---

## Step 1331: Service Mesh — Resilience Patterns

```csharp
// Polly v8 — Resilience Pipelines
using Polly;
using Polly.Retry;
using Polly.CircuitBreaker;
using Polly.Timeout;
using Polly.Bulkhead;

public static class ResilienceExtensions
{
    public static IHttpClientBuilder AddStandardResilience(this IHttpClientBuilder builder)
    {
        return builder.AddResilienceHandler("standard", pipeline =>
        {
            // 1. Timeout per attempt
            pipeline.AddTimeout(new TimeoutStrategyOptions
            {
                Timeout = TimeSpan.FromSeconds(30),
            });

            // 2. Retry with jitter
            pipeline.AddRetry(new RetryStrategyOptions<HttpResponseMessage>
            {
                MaxRetryAttempts = 3,
                Delay = TimeSpan.FromSeconds(1),
                BackoffType = DelayBackoffType.Exponential,
                UseJitter = true,
                ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
                    .Handle<HttpRequestException>()
                    .HandleResult(r => r.StatusCode >= System.Net.HttpStatusCode.InternalServerError)
                    .HandleResult(r => r.StatusCode == System.Net.HttpStatusCode.TooManyRequests),
                OnRetry = static args =>
                {
                    var logger = args.Context.ServiceProvider?.GetService<ILogger>();
                    logger?.LogWarning("Retry attempt {Attempt}", args.AttemptNumber);
                    return ValueTask.CompletedTask;
                },
            });

            // 3. Circuit Breaker
            pipeline.AddCircuitBreaker(new CircuitBreakerStrategyOptions<HttpResponseMessage>
            {
                SamplingDuration = TimeSpan.FromSeconds(30),
                MinimumThroughput = 10,
                FailureRatio = 0.5,
                BreakDuration = TimeSpan.FromSeconds(15),
                ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
                    .Handle<HttpRequestException>()
                    .HandleResult(r => (int)r.StatusCode >= 500),
                OnOpened = static args =>
                {
                    var logger = args.Context.ServiceProvider?.GetService<ILogger>();
                    logger?.LogError("Circuit breaker opened for {Duration}", args.BreakDuration);
                    return ValueTask.CompletedTask;
                },
                OnClosed = static args =>
                {
                    var logger = args.Context.ServiceProvider?.GetService<ILogger>();
                    logger?.LogInformation("Circuit breaker closed");
                    return ValueTask.CompletedTask;
                },
            });
        });
    }
}

// Registration
builder.Services
    .AddHttpClient<IOrderServiceClient, OrderServiceClient>(c =>
        c.BaseAddress = new Uri(builder.Configuration["Services:OrderService:Url"]!))
    .AddStandardResilience();
```

---

## Step 1332: Distributed Caching with Cache-Aside Pattern

```csharp
public interface ICacheService
{
    Task<T?> GetAsync<T>(string key, CancellationToken ct = default);
    Task SetAsync<T>(string key, T value, TimeSpan? ttl = null, CancellationToken ct = default);
    Task RemoveAsync(string key, CancellationToken ct = default);
    Task<T> GetOrCreateAsync<T>(string key, Func<CancellationToken, Task<T>> factory,
        TimeSpan? ttl = null, CancellationToken ct = default);
}

public class RedisCacheService(IConnectionMultiplexer redis, ILogger<RedisCacheService> logger)
    : ICacheService
{
    private readonly IDatabase _db = redis.GetDatabase();
    private static readonly JsonSerializerOptions _jsonOptions = new()
    {
        PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
    };

    public async Task<T?> GetAsync<T>(string key, CancellationToken ct = default)
    {
        try
        {
            var value = await _db.StringGetAsync(key);
            if (value.IsNullOrEmpty) return default;
            return JsonSerializer.Deserialize<T>(value!, _jsonOptions);
        }
        catch (Exception ex)
        {
            logger.LogWarning(ex, "Cache get failed for key {Key}", key);
            return default; // Cache failures are non-fatal
        }
    }

    public async Task SetAsync<T>(string key, T value, TimeSpan? ttl = null, CancellationToken ct = default)
    {
        try
        {
            var serialized = JsonSerializer.Serialize(value, _jsonOptions);
            await _db.StringSetAsync(key, serialized, ttl ?? TimeSpan.FromMinutes(30));
        }
        catch (Exception ex)
        {
            logger.LogWarning(ex, "Cache set failed for key {Key}", key);
        }
    }

    public async Task<T> GetOrCreateAsync<T>(
        string key, Func<CancellationToken, Task<T>> factory,
        TimeSpan? ttl = null, CancellationToken ct = default)
    {
        var cached = await GetAsync<T>(key, ct);
        if (cached is not null) return cached;

        var value = await factory(ct);
        await SetAsync(key, value, ttl, ct);
        return value;
    }

    public async Task RemoveAsync(string key, CancellationToken ct = default)
    {
        try { await _db.KeyDeleteAsync(key); }
        catch (Exception ex) { logger.LogWarning(ex, "Cache delete failed for key {Key}", key); }
    }
}

// Usage with cache-aside in service
public class ProductService(
    IProductRepository repository,
    ICacheService cache,
    ILogger<ProductService> logger)
{
    private static string CacheKey(Guid id) => $"product:{id}";
    private static string ListCacheKey(int page) => $"products:page:{page}";

    public async Task<ProductDto?> GetByIdAsync(Guid id, CancellationToken ct)
    {
        return await cache.GetOrCreateAsync(
            CacheKey(id),
            async ct => await repository.GetDtoByIdAsync(id, ct),
            ttl: TimeSpan.FromMinutes(10),
            ct: ct);
    }

    public async Task UpdateAsync(UpdateProductCommand command, CancellationToken ct)
    {
        await repository.UpdateAsync(command, ct);

        // Invalidate cache
        await cache.RemoveAsync(CacheKey(command.Id), ct);

        // Also invalidate any list cache (pattern delete)
        var server = cache.GetServer(); // extension
        await foreach (var key in server.KeysAsync(pattern: "products:page:*"))
            await cache.RemoveAsync(key, ct);

        logger.LogInformation("Product {Id} updated, cache invalidated", command.Id);
    }
}
```

---

## Step 1333: Service Discovery & Load Balancing

```csharp
// Consul-based service discovery
using Consul;

public class ConsulServiceRegistry(IConsulClient consul, ILogger<ConsulServiceRegistry> logger)
    : IHostedService
{
    private string? _serviceId;

    public async Task StartAsync(CancellationToken ct)
    {
        _serviceId = $"order-service-{Guid.NewGuid():N}";

        var registration = new AgentServiceRegistration
        {
            ID = _serviceId,
            Name = "order-service",
            Address = Environment.GetEnvironmentVariable("POD_IP") ?? "localhost",
            Port = 8080,
            Tags = ["v2", "dotnet", "orders"],
            Check = new AgentServiceCheck
            {
                HTTP = $"http://localhost:8080/health",
                Interval = "10s",
                Timeout = "5s",
                DeregisterCriticalServiceAfter = "60s",
            },
            Meta = new Dictionary<string, string>
            {
                ["version"] = Assembly.GetExecutingAssembly().GetName().Version?.ToString() ?? "0.0.1",
                ["environment"] = Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT") ?? "Production",
            },
        };

        await consul.Agent.ServiceRegister(registration, ct);
        logger.LogInformation("Registered with Consul as {ServiceId}", _serviceId);
    }

    public async Task StopAsync(CancellationToken ct)
    {
        if (_serviceId is not null)
        {
            await consul.Agent.ServiceDeregister(_serviceId, ct);
            logger.LogInformation("Deregistered from Consul: {ServiceId}", _serviceId);
        }
    }
}

// Custom IServiceEndpointResolver for named clients
public class ConsulServiceEndpointResolver(IConsulClient consul) : IHostedService
{
    public async Task<Uri?> ResolveAsync(string serviceName, CancellationToken ct)
    {
        var result = await consul.Health.Service(serviceName, tag: null, passingOnly: true, ct);
        var services = result.Response;
        if (!services.Any()) return null;

        // Simple random selection (production: use weighted round-robin)
        var service = services[Random.Shared.Next(services.Length)];
        return new Uri($"http://{service.Service.Address}:{service.Service.Port}");
    }
}
```

---

## Step 1334: Distributed Tracing Across Services

```csharp
// OpenTelemetry propagation ensures trace context flows across service boundaries

// Sending service: context is automatically propagated by HttpClient
public class OrderHttpClient(HttpClient httpClient, ILogger<OrderHttpClient> logger)
{
    public async Task<OrderDto?> GetOrderAsync(Guid orderId, CancellationToken ct)
    {
        // Activity context is automatically injected into request headers
        // by the OpenTelemetry HttpClient instrumentation
        using var activity = OrdersActivitySource.StartActivity(
            "GetOrder",
            ActivityKind.Client,
            tags: new[] { new KeyValuePair<string, object?>("order.id", orderId) });

        var response = await httpClient.GetAsync($"api/orders/{orderId}", ct);
        response.EnsureSuccessStatusCode();

        activity?.SetTag("http.status_code", (int)response.StatusCode);
        return await response.Content.ReadFromJsonAsync<OrderDto>(ct);
    }
}

// Receiving service: trace context is automatically extracted from headers
// by the ASP.NET Core OpenTelemetry instrumentation

// Custom baggage propagation
public class CorrelationMiddleware(RequestDelegate next)
{
    public async Task InvokeAsync(HttpContext context)
    {
        // Extract tenant from JWT and add to baggage for all child spans
        var tenantId = context.User.FindFirst("tenant_id")?.Value;
        if (tenantId is not null)
        {
            Activity.Current?.SetBaggage("tenant.id", tenantId);
            Activity.Current?.SetTag("tenant.id", tenantId);
        }

        // Add to response so clients can correlate
        var traceId = Activity.Current?.TraceId.ToString();
        if (traceId is not null)
            context.Response.Headers["X-Trace-ID"] = traceId;

        await next(context);
    }
}

// Activity source per service
public static class OrdersActivitySource
{
    private static readonly ActivitySource _source = new("OrderService", "1.0.0");

    public static Activity? StartActivity(
        string name,
        ActivityKind kind = ActivityKind.Internal,
        IEnumerable<KeyValuePair<string, object?>>? tags = null)
        => _source.StartActivity(name, kind, tags: tags);
}
```

---

## Step 1335: API Versioning

```csharp
// API Versioning in ASP.NET Core
using Asp.Versioning;

builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;
    options.ApiVersionReader = ApiVersionReader.Combine(
        new UrlSegmentApiVersionReader(),
        new HeaderApiVersionReader("X-API-Version"),
        new QueryStringApiVersionReader("api-version")
    );
}).AddApiExplorer(options =>
{
    options.GroupNameFormat = "'v'VVV";
    options.SubstituteApiVersionInUrl = true;
});

// V1 Controller
[ApiController]
[ApiVersion("1.0")]
[Route("api/v{version:apiVersion}/orders")]
public class OrdersV1Controller(IOrderService service) : ControllerBase
{
    [HttpGet("{id:guid}")]
    public async Task<IActionResult> GetOrder(Guid id, CancellationToken ct)
    {
        var order = await service.GetByIdAsync(id, ct);
        return order is null ? NotFound() : Ok(new OrderResponseV1(order.Id, order.Status, order.TotalAmount));
    }
}

// V2 Controller — adds items detail and customer info
[ApiController]
[ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/orders")]
public class OrdersV2Controller(IOrderService service, IUserService userService) : ControllerBase
{
    [HttpGet("{id:guid}")]
    public async Task<IActionResult> GetOrder(Guid id, CancellationToken ct)
    {
        var order = await service.GetByIdAsync(id, ct);
        if (order is null) return NotFound();

        var customer = await userService.GetByIdAsync(order.CustomerId, ct);
        return Ok(new OrderResponseV2(
            order.Id,
            order.Status,
            order.TotalAmount,
            order.Items,
            customer is null ? null : new CustomerSummary(customer.Id, customer.Name, customer.Email)
        ));
    }

    [HttpGet]
    [MapToApiVersion("2.0")]
    public async Task<IActionResult> ListOrders(
        [FromQuery] OrderFilterV2 filter,
        CancellationToken ct)
    {
        var result = await service.ListAsync(filter, ct);
        return Ok(result);
    }
}

// Versioned records
public record OrderResponseV1(Guid Id, string Status, decimal TotalAmount);
public record OrderResponseV2(Guid Id, string Status, decimal TotalAmount,
    List<OrderItem> Items, CustomerSummary? Customer);
public record CustomerSummary(Guid Id, string Name, string Email);
```

---

## Step 1336: Health Checks & Readiness Probes

```csharp
builder.Services
    .AddHealthChecks()
    .AddNpgsql(connectionString, name: "postgres",
        failureStatus: HealthStatus.Unhealthy,
        tags: ["db", "data"])
    .AddRedis(redisConnectionString, name: "redis",
        failureStatus: HealthStatus.Degraded,
        tags: ["cache"])
    .AddRabbitMQ(rabbitMQUri, name: "rabbitmq",
        failureStatus: HealthStatus.Degraded,
        tags: ["messaging"])
    .AddCheck<ExternalApiHealthCheck>("payment-api",
        failureStatus: HealthStatus.Degraded,
        tags: ["external"])
    .AddCheck<DiskSpaceHealthCheck>("disk-space",
        failureStatus: HealthStatus.Unhealthy)
    .AddCheck("startup", () => AppStartup.IsReady
        ? HealthCheckResult.Healthy()
        : HealthCheckResult.Unhealthy("Not yet ready"));

// Separate endpoints for k8s probes
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false, // Only checks if app is alive
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse,
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = hc => hc.Tags.Contains("db") || hc.Tags.Contains("messaging"),
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse,
});

app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse,
});

// Custom health check
public class ExternalApiHealthCheck(IHttpClientFactory factory) : IHealthCheck
{
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context, CancellationToken ct)
    {
        try
        {
            var client = factory.CreateClient("payment-api");
            var response = await client.GetAsync("/health", ct);
            return response.IsSuccessStatusCode
                ? HealthCheckResult.Healthy($"Payment API is healthy: {response.StatusCode}")
                : HealthCheckResult.Degraded($"Payment API returned {response.StatusCode}");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("Payment API is unreachable", ex);
        }
    }
}

public class DiskSpaceHealthCheck : IHealthCheck
{
    private const long MinFreeBytes = 1024L * 1024 * 500; // 500 MB

    public Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context, CancellationToken ct)
    {
        var drive = DriveInfo.GetDrives().FirstOrDefault(d => d.IsReady && d.DriveType == DriveType.Fixed);
        if (drive is null) return Task.FromResult(HealthCheckResult.Healthy("No fixed drive found"));

        var free = drive.AvailableFreeSpace;
        var data = new Dictionary<string, object>
        {
            ["free_bytes"] = free,
            ["free_gb"] = free / (1024.0 * 1024 * 1024),
        };

        return Task.FromResult(free > MinFreeBytes
            ? HealthCheckResult.Healthy($"Disk space OK: {free / 1024 / 1024} MB free", data)
            : HealthCheckResult.Unhealthy($"Low disk space: {free / 1024 / 1024} MB free", data: data));
    }
}
```

---

## Step 1337: Inter-Service Contract Testing with Pact

```xml
<ItemGroup>
  <PackageReference Include="PactNet" Version="5.*" />
  <PackageReference Include="PactNet.Native" Version="5.*" />
</ItemGroup>
```

```csharp
// Consumer test (OrderService consuming ProductService)
public class ProductServiceConsumerTests : IAsyncLifetime
{
    private readonly IPactBuilderV4 _pact;
    private IMockServer _mockServer = null!;

    public ProductServiceConsumerTests()
    {
        var config = new PactConfig
        {
            PactDir = "../../../pacts",
            LogLevel = PactLogLevel.Debug,
        };
        _pact = Pact.V4("OrderService", "ProductService", config).WithHttpInteractions();
    }

    public async Task InitializeAsync()
    {
        _mockServer = _pact.MockServer.Start();
        await Task.CompletedTask;
    }

    [Fact]
    public async Task GetProduct_ExistingProduct_ReturnsProductDto()
    {
        var productId = Guid.NewGuid();

        _pact
            .UponReceiving("A request for an existing product")
            .Given("Product exists", new Dictionary<string, string>
            {
                ["productId"] = productId.ToString(),
                ["productName"] = "Widget A",
                ["price"] = "29.99",
            })
            .WithRequest(HttpMethod.Get, $"/api/products/{productId}")
            .WillRespond()
            .WithStatus(200)
            .WithHeader("Content-Type", "application/json; charset=utf-8")
            .WithJsonBody(new
            {
                id = productId.ToString(),
                name = Match.Type("Widget A"),
                price = Match.Decimal(29.99m),
                inStock = Match.Type(true),
            });

        await _pact.VerifyAsync(async ctx =>
        {
            var client = new ProductHttpClient(new HttpClient
            {
                BaseAddress = new Uri(ctx.MockServerUri),
            });
            var product = await client.GetByIdAsync(productId);
            product.Should().NotBeNull();
            product!.Name.Should().Be("Widget A");
        });
    }

    public async Task DisposeAsync()
    {
        _mockServer.Dispose();
        await Task.CompletedTask;
    }
}

// Provider test (ProductService verifying contracts)
public class ProductServiceProviderTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;

    public ProductServiceProviderTests(WebApplicationFactory<Program> factory)
        => _factory = factory;

    [Fact]
    public async Task VerifyPactWithOrderService()
    {
        var verifier = new PactVerifier("ProductService", new PactVerifierConfig
        {
            LogLevel = PactLogLevel.Information,
        });

        await verifier
            .WithHttpEndpoint(new Uri(_factory.Server.BaseAddress))
            .WithFileSource(new FileInfo("../../../pacts/OrderService-ProductService.json"))
            .WithStateHandlers(new Dictionary<string, Func<IDictionary<string, string>, Task>>
            {
                ["Product exists"] = async parameters =>
                {
                    var productId = Guid.Parse(parameters["productId"]);
                    // Seed test data
                    using var scope = _factory.Services.CreateScope();
                    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
                    db.Products.Add(new Product
                    {
                        Id = productId,
                        Name = parameters["productName"],
                        Price = decimal.Parse(parameters["price"]),
                        InStock = true,
                    });
                    await db.SaveChangesAsync();
                },
            })
            .VerifyAsync();
    }
}
```

---

## Step 1338: Microservice Testing Strategy

```csharp
// Integration test with Testcontainers — full microservice test
public class OrderServiceIntegrationTests : IAsyncLifetime
{
    private PostgreSqlContainer _postgres = null!;
    private RedisContainer _redis = null!;
    private RabbitMqContainer _rabbitmq = null!;
    private WebApplicationFactory<Program> _factory = null!;
    private HttpClient _client = null!;

    public async Task InitializeAsync()
    {
        _postgres = new PostgreSqlBuilder()
            .WithImage("postgres:16-alpine")
            .WithDatabase("orders_test")
            .Build();

        _redis = new RedisBuilder()
            .WithImage("redis:7-alpine")
            .Build();

        _rabbitmq = new RabbitMqBuilder()
            .WithImage("rabbitmq:3.13-management-alpine")
            .Build();

        await Task.WhenAll(
            _postgres.StartAsync(),
            _redis.StartAsync(),
            _rabbitmq.StartAsync());

        _factory = new WebApplicationFactory<Program>()
            .WithWebHostBuilder(builder =>
            {
                builder.ConfigureAppConfiguration((_, config) =>
                {
                    config.AddInMemoryCollection(new Dictionary<string, string?>
                    {
                        ["ConnectionStrings:DefaultConnection"] = _postgres.GetConnectionString(),
                        ["ConnectionStrings:Redis"] = _redis.GetConnectionString(),
                        ["RabbitMQ:Host"] = _rabbitmq.Hostname,
                        ["RabbitMQ:Port"] = _rabbitmq.GetMappedPublicPort(5672).ToString(),
                    });
                });
            });

        _client = _factory.CreateClient();

        // Run migrations
        using var scope = _factory.Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        await db.Database.MigrateAsync();
    }

    [Fact]
    public async Task CreateOrder_ValidRequest_ReturnsCreated()
    {
        var request = new CreateOrderRequest(
            CustomerId: Guid.NewGuid(),
            Items: [new OrderItemRequest(ProductId: Guid.NewGuid(), Quantity: 2, UnitPrice: 19.99m)],
            ShippingAddress: "123 Test Street"
        );

        var response = await _client.PostAsJsonAsync("/api/v1/orders", request);

        response.StatusCode.Should().Be(HttpStatusCode.Created);

        var created = await response.Content.ReadFromJsonAsync<OrderDto>();
        created.Should().NotBeNull();
        created!.Status.Should().Be("Pending");
        created.TotalAmount.Should().Be(39.98m);
    }

    [Fact]
    public async Task GetOrder_NonExistent_Returns404()
    {
        var response = await _client.GetAsync($"/api/v1/orders/{Guid.NewGuid()}");
        response.StatusCode.Should().Be(HttpStatusCode.NotFound);
    }

    [Fact]
    public async Task CreateOrder_Then_MessagePublished()
    {
        // Arrange: set up RabbitMQ consumer to capture message
        var messageReceived = new TaskCompletionSource<OrderCreatedEvent>(
            TaskCreationOptions.RunContinuationsAsynchronously);

        using var scope = _factory.Services.CreateScope();
        var consumer = scope.ServiceProvider.GetRequiredService<ITestConsumer<OrderCreatedEvent>>();
        consumer.OnMessage = msg => messageReceived.TrySetResult(msg);

        // Act
        var response = await _client.PostAsJsonAsync("/api/v1/orders", new CreateOrderRequest(
            CustomerId: Guid.NewGuid(),
            Items: [new OrderItemRequest(Guid.NewGuid(), 1, 10.00m)],
            ShippingAddress: "Test"));
        response.EnsureSuccessStatusCode();

        // Assert: message published within 5 seconds
        var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
        var message = await messageReceived.Task.WaitAsync(cts.Token);
        message.TotalAmount.Should().Be(10.00m);
    }

    public async Task DisposeAsync()
    {
        _client.Dispose();
        await _factory.DisposeAsync();
        await Task.WhenAll(
            _postgres.DisposeAsync().AsTask(),
            _redis.DisposeAsync().AsTask(),
            _rabbitmq.DisposeAsync().AsTask());
    }
}
```

---

## Step 1339: Microservice Configuration Management

```csharp
// Centralized config with validation
public class OrderServiceOptions
{
    [Required]
    public string PaymentServiceUrl { get; set; } = string.Empty;

    [Required]
    public string InventoryServiceUrl { get; set; } = string.Empty;

    [Range(1, 100)]
    public int MaxRetryAttempts { get; set; } = 3;

    [Range(100, 10000)]
    public int TimeoutMs { get; set; } = 5000;

    public bool EnableDistributedTracing { get; set; } = true;
    public bool EnableCircuitBreaker { get; set; } = true;
}

// Registration with validation
builder.Services
    .AddOptions<OrderServiceOptions>()
    .Bind(builder.Configuration.GetSection("OrderService"))
    .ValidateDataAnnotations()
    .ValidateOnStart();

// Dynamic config reload via IOptionsMonitor
public class DynamicConfigService(IOptionsMonitor<OrderServiceOptions> options)
{
    public int GetTimeout() => options.CurrentValue.TimeoutMs;
    public bool IsCircuitBreakerEnabled() => options.CurrentValue.EnableCircuitBreaker;
}

// Feature flags
public class FeatureFlags
{
    public bool NewCheckoutFlow { get; set; }
    public bool GraphQlEnabled { get; set; }
    public bool AsyncOrderProcessing { get; set; }
    public double CanaryPercentage { get; set; } = 0.0;
}

builder.Services
    .AddOptions<FeatureFlags>()
    .Bind(builder.Configuration.GetSection("FeatureFlags"))
    .ValidateDataAnnotations()
    .ValidateOnStart();

// Azure App Configuration with feature flags
builder.Configuration.AddAzureAppConfiguration(options =>
{
    options.Connect(builder.Configuration["AzureAppConfig:ConnectionString"])
        .ConfigureRefresh(refresh =>
        {
            refresh.Register("FeatureFlags:Sentinel", refreshAll: true);
            refresh.SetCacheExpiration(TimeSpan.FromSeconds(30));
        })
        .UseFeatureFlags(flags =>
            flags.SetCachingInterval(TimeSpan.FromSeconds(30)));
});

builder.Services.AddAzureAppConfiguration();
app.UseAzureAppConfiguration();
```

---

## Step 1340: Complete Microservices Solution Structure

```
ecommerce-platform/
├── services/
│   ├── order-service/
│   │   ├── src/
│   │   │   ├── OrderService.Api/          # ASP.NET Core Web API
│   │   │   ├── OrderService.Application/  # CQRS handlers, commands, queries
│   │   │   ├── OrderService.Domain/       # Aggregates, events, value objects
│   │   │   └── OrderService.Infrastructure/ # EF Core, Kafka, Redis
│   │   ├── tests/
│   │   │   ├── OrderService.UnitTests/
│   │   │   ├── OrderService.IntegrationTests/
│   │   │   └── OrderService.ContractTests/
│   │   ├── Dockerfile
│   │   └── docker-compose.test.yml
│   ├── product-service/
│   ├── user-service/
│   ├── payment-service/
│   └── notification-service/
├── api-gateway/
│   ├── src/ApiGateway/                    # YARP-based gateway
│   └── Dockerfile
├── shared/
│   ├── Contracts/                         # Shared event contracts (NuGet)
│   ├── BuildingBlocks/                    # Common utilities (NuGet)
│   └── ServiceDefaults/                   # Aspire service defaults
├── infrastructure/
│   ├── k8s/                               # Kubernetes manifests
│   ├── helm/                              # Helm charts
│   └── terraform/                         # IaC
├── .github/
│   └── workflows/
│       ├── service-ci.yml                 # Per-service CI
│       └── integration-tests.yml
└── docker-compose.yml                     # Local development
```

```csharp
// ServiceDefaults (Aspire pattern) — shared across all services
// ServiceDefaults/Extensions.cs
public static class Extensions
{
    public static IHostApplicationBuilder AddServiceDefaults(
        this IHostApplicationBuilder builder)
    {
        builder.ConfigureOpenTelemetry();
        builder.AddDefaultHealthChecks();
        builder.Services.AddServiceDiscovery();
        builder.Services.ConfigureHttpClientDefaults(http =>
        {
            http.AddStandardResilienceHandler();
            http.AddServiceDiscovery();
        });
        return builder;
    }

    public static IHostApplicationBuilder ConfigureOpenTelemetry(
        this IHostApplicationBuilder builder)
    {
        builder.Logging.AddOpenTelemetry(logging =>
        {
            logging.IncludeFormattedMessage = true;
            logging.IncludeScopes = true;
        });

        builder.Services.AddOpenTelemetry()
            .WithMetrics(metrics =>
            {
                metrics
                    .AddAspNetCoreInstrumentation()
                    .AddHttpClientInstrumentation()
                    .AddRuntimeInstrumentation()
                    .AddProcessInstrumentation();
            })
            .WithTracing(tracing =>
            {
                tracing
                    .AddAspNetCoreInstrumentation()
                    .AddHttpClientInstrumentation()
                    .AddGrpcClientInstrumentation()
                    .AddNpgsql()
                    .AddRedisInstrumentation();
            });

        builder.AddOpenTelemetryExporters();
        return builder;
    }

    private static IHostApplicationBuilder AddOpenTelemetryExporters(
        this IHostApplicationBuilder builder)
    {
        var useOtlpExporter = !string.IsNullOrWhiteSpace(
            builder.Configuration["OTEL_EXPORTER_OTLP_ENDPOINT"]);

        if (useOtlpExporter)
        {
            builder.Services.AddOpenTelemetry().UseOtlpExporter();
        }
        return builder;
    }
}

// Each service's Program.cs
var builder = WebApplication.CreateBuilder(args);
builder.AddServiceDefaults();  // One line for all cross-cutting concerns

builder.Services.AddProblemDetails();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// Service-specific registrations
builder.Services.AddDbContext<AppDbContext>(opt =>
    opt.UseNpgsql(builder.Configuration.GetConnectionString("DefaultConnection")));

var app = builder.Build();
app.MapDefaultEndpoints(); // /health/live, /health/ready
app.UseExceptionHandler();
app.MapControllers();
app.Run();
```

---

## Step 1341: .NET Aspire — Local Microservices Orchestration

```xml
<!-- AppHost.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net9.0</TargetFramework>
    <IsAspireHost>true</IsAspireHost>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Aspire.Hosting.AppHost" Version="9.*" />
    <PackageReference Include="Aspire.Hosting.PostgreSQL" Version="9.*" />
    <PackageReference Include="Aspire.Hosting.Redis" Version="9.*" />
    <PackageReference Include="Aspire.Hosting.RabbitMQ" Version="9.*" />
    <PackageReference Include="Aspire.Hosting.Kafka" Version="9.*" />
  </ItemGroup>
</Project>
```

```csharp
// AppHost/Program.cs
var builder = DistributedApplication.CreateBuilder(args);

// Infrastructure
var postgres = builder.AddPostgres("postgres")
    .WithDataVolume()
    .WithPgAdmin();

var redis = builder.AddRedis("redis")
    .WithRedisCommander();

var rabbitmq = builder.AddRabbitMQ("rabbitmq")
    .WithManagementPlugin();

var kafka = builder.AddKafka("kafka")
    .WithKafkaUI();

// Databases per service (isolation)
var ordersDb = postgres.AddDatabase("ordersdb");
var productsDb = postgres.AddDatabase("productsdb");
var usersDb = postgres.AddDatabase("usersdb");

// Services
var userService = builder.AddProject<Projects.UserService>("user-service")
    .WithReference(usersDb)
    .WithReference(redis)
    .WithExternalHttpEndpoints();

var productService = builder.AddProject<Projects.ProductService>("product-service")
    .WithReference(productsDb)
    .WithReference(redis)
    .WithExternalHttpEndpoints();

var orderService = builder.AddProject<Projects.OrderService>("order-service")
    .WithReference(ordersDb)
    .WithReference(redis)
    .WithReference(rabbitmq)
    .WithReference(kafka)
    .WithReference(userService)     // Service discovery
    .WithReference(productService)
    .WithExternalHttpEndpoints();

var apiGateway = builder.AddProject<Projects.ApiGateway>("api-gateway")
    .WithReference(orderService)
    .WithReference(productService)
    .WithReference(userService)
    .WithExternalHttpEndpoints();

builder.Build().Run();
```

```csharp
// Service registration using Aspire service defaults
// OrderService/Program.cs — simplified with Aspire

var builder = WebApplication.CreateBuilder(args);
builder.AddServiceDefaults(); // OpenTelemetry + health checks + service discovery

// Named HTTP clients with automatic service discovery
// "http://user-service" resolves via Aspire
builder.Services.AddHttpClient<IUserServiceClient, UserServiceClient>(
    client => client.BaseAddress = new Uri("http://user-service"));

builder.Services.AddHttpClient<IProductServiceClient, ProductServiceClient>(
    client => client.BaseAddress = new Uri("http://product-service"));

// Or gRPC
builder.Services.AddGrpcClient<UserGrpc.UserGrpcClient>(
    options => options.Address = new Uri("http://user-service"));

var app = builder.Build();
app.MapDefaultEndpoints();
app.MapControllers();
app.Run();
```

---

## Step 1342: Message Schema Evolution

```csharp
// Schema Registry pattern for Kafka with Avro/JSON Schema
// Using SchemaRegistry NuGet (Confluent.SchemaRegistry)

// V1 Event Schema
public record OrderCreatedV1(
    Guid OrderId,
    Guid CustomerId,
    decimal TotalAmount,
    DateTimeOffset CreatedAt
);

// V2 Event Schema — backward compatible (new optional fields)
public record OrderCreatedV2(
    Guid OrderId,
    Guid CustomerId,
    decimal TotalAmount,
    DateTimeOffset CreatedAt,
    string? ShippingAddress = null,  // New in V2
    string? Currency = "THB"          // New in V2 with default
);

// Versioned event envelope
public record EventEnvelope<T>
{
    public required string EventId { get; init; } = Guid.NewGuid().ToString();
    public required string EventType { get; init; }
    public required int SchemaVersion { get; init; }
    public required T Payload { get; init; }
    public required DateTimeOffset OccurredAt { get; init; }
    public string CorrelationId { get; init; } = Activity.Current?.TraceId.ToString() ?? string.Empty;
}

// Consumer with version handling
public class VersionAwareOrderConsumer(ILogger<VersionAwareOrderConsumer> logger)
{
    public void Handle(string rawMessage)
    {
        var envelope = JsonSerializer.Deserialize<EventEnvelope<JsonElement>>(rawMessage)
            ?? throw new InvalidOperationException("Invalid event envelope");

        switch (envelope.SchemaVersion)
        {
            case 1:
                var v1 = envelope.Payload.Deserialize<OrderCreatedV1>();
                HandleV1(v1!);
                break;
            case 2:
                var v2 = envelope.Payload.Deserialize<OrderCreatedV2>();
                HandleV2(v2!);
                break;
            default:
                logger.LogWarning("Unknown schema version {Version} for {EventType}",
                    envelope.SchemaVersion, envelope.EventType);
                break;
        }
    }

    private void HandleV1(OrderCreatedV1 evt) =>
        HandleV2(new OrderCreatedV2(evt.OrderId, evt.CustomerId, evt.TotalAmount, evt.CreatedAt));

    private void HandleV2(OrderCreatedV2 evt)
    {
        logger.LogInformation("Processing order {OrderId} (currency: {Currency})",
            evt.OrderId, evt.Currency);
        // ... business logic
    }
}
```

---

## Step 1343: Microservice Deployment with Helm

```yaml
# helm/order-service/Chart.yaml
apiVersion: v2
name: order-service
description: Order Service Helm Chart
type: application
version: 0.1.0
appVersion: "2.1.0"
dependencies:
  - name: common
    version: "1.x.x"
    repository: "https://charts.my-company.com"
```

```yaml
# helm/order-service/values.yaml
replicaCount: 3

image:
  repository: myacr.azurecr.io/order-service
  pullPolicy: IfNotPresent
  tag: ""  # Overridden by CI

nameOverride: ""
fullnameOverride: ""

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/rate-limit: "1000"
  hosts:
    - host: api.mycompany.com
      paths:
        - path: /api/orders
          pathType: Prefix
  tls:
    - secretName: api-tls
      hosts: [api.mycompany.com]

resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "512Mi"
    cpu: "500m"

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

env:
  - name: ASPNETCORE_ENVIRONMENT
    value: Production
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: "http://otel-collector:4317"

envFrom:
  - secretRef:
      name: order-service-secrets
  - configMapRef:
      name: order-service-config

livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5

podDisruptionBudget:
  enabled: true
  minAvailable: 2

topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app.kubernetes.io/name: order-service
```

```yaml
# GitHub Actions — per-service deploy
# .github/workflows/deploy-order-service.yml
name: Deploy Order Service
on:
  push:
    paths:
      - 'services/order-service/**'
      - '.github/workflows/deploy-order-service.yml'
    branches: [main]

jobs:
  build-push:
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      sha-tag: ${{ steps.sha.outputs.tag }}
    steps:
      - uses: actions/checkout@v4
      - name: Set SHA tag
        id: sha
        run: echo "tag=$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT
      - name: Login to ACR
        uses: azure/docker-login@v1
        with:
          login-server: ${{ vars.ACR_LOGIN_SERVER }}
          username: ${{ secrets.ACR_USERNAME }}
          password: ${{ secrets.ACR_PASSWORD }}
      - name: Build and push
        run: |
          docker build -f services/order-service/Dockerfile \
            -t ${{ vars.ACR_LOGIN_SERVER }}/order-service:${{ steps.sha.outputs.tag }} \
            -t ${{ vars.ACR_LOGIN_SERVER }}/order-service:latest \
            services/order-service/
          docker push ${{ vars.ACR_LOGIN_SERVER }}/order-service:${{ steps.sha.outputs.tag }}
          docker push ${{ vars.ACR_LOGIN_SERVER }}/order-service:latest

  deploy-staging:
    needs: build-push
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4
      - uses: azure/setup-kubectl@v3
      - name: Deploy to staging
        run: |
          helm upgrade --install order-service ./helm/order-service \
            --namespace staging \
            --set image.tag=${{ needs.build-push.outputs.sha-tag }} \
            --set replicaCount=1 \
            --wait --timeout=5m

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
      - uses: azure/setup-kubectl@v3
      - name: Deploy to production
        run: |
          helm upgrade --install order-service ./helm/order-service \
            --namespace production \
            --set image.tag=${{ needs.build-push.outputs.sha-tag }} \
            --wait --timeout=10m
```

---

## Step 1344: Summary — Microservices Patterns Checklist

```
MICROSERVICES PATTERNS REFERENCE
═══════════════════════════════

COMMUNICATION
├── Synchronous: REST (YARP gateway), gRPC (streaming), GraphQL (BFF)
├── Asynchronous: Kafka (event streaming), RabbitMQ (task queues)
└── BFF: Mobile BFF, Web BFF, Admin BFF

RESILIENCE
├── Circuit Breaker (Polly v8 pipelines)
├── Retry with Exponential Backoff + Jitter
├── Timeout per attempt
├── Bulkhead isolation
└── Health checks (live/ready probes)

DATA PATTERNS  
├── Database-per-Service (polyglot persistence)
├── Event Sourcing (append-only event log)
├── CQRS (separate read/write models)
├── Saga Pattern (choreography + orchestration)
└── Outbox Pattern (guaranteed event delivery)

OBSERVABILITY
├── Distributed Tracing (OpenTelemetry + Jaeger/Tempo)
├── Structured Logging (Serilog + correlation IDs)
├── Metrics (Prometheus + Grafana)
└── Alerts (PagerDuty / Alertmanager)

DEPLOYMENT
├── Container (Docker multi-stage)
├── Kubernetes (HPA, PDB, topology spread)
├── Helm Charts (parameterized deployments)
├── GitOps (ArgoCD + progressive rollout)
└── .NET Aspire (local orchestration)

TESTING
├── Unit Tests (domain logic)
├── Integration Tests (Testcontainers)
├── Contract Tests (Pact)
└── Load Tests (NBomber / k6)

ANTI-PATTERNS TO AVOID
├── Distributed Monolith (shared DB across services)
├── Chatty services (too many sync calls → latency)
├── Missing idempotency (duplicate events → data corruption)
├── No correlation IDs (impossible to trace)
└── Sync-only communication (tight coupling)
```

---

## สรุป Part 46

Part นี้ครอบคลุม Microservices Patterns & Service Communication อย่างครบถ้วน:

**Steps 1321-1344:**
- **API Gateway** (YARP) — routing, rate limiting, auth, request aggregation
- **BFF Pattern** — Mobile BFF with data shaping
- **gRPC** — unary + server streaming + client interceptors
- **GraphQL** (Hot Chocolate) — queries, mutations, subscriptions, DataLoader
- **Saga Patterns** — choreography (event-driven) and orchestration (MassTransit)
- **Event Sourcing** — append-only event log, aggregate rehydration
- **CQRS** — write side with aggregates, read side with projections
- **Outbox Pattern** — guaranteed-once event delivery
- **Resilience** (Polly v8) — retry, circuit breaker, timeout, bulkhead
- **Distributed Caching** — cache-aside with Redis
- **Service Discovery** — Consul registration and resolution
- **Distributed Tracing** — OpenTelemetry across service boundaries
- **API Versioning** — URL segment + header + query string
- **Health Checks** — live/ready probes for Kubernetes
- **Contract Testing** — Pact for consumer/provider verification
- **Integration Testing** — Testcontainers with full service stack
- **Configuration** — options validation, Azure App Configuration
- **.NET Aspire** — local microservices orchestration
- **Schema Evolution** — versioned event envelopes
- **Helm Deployment** — chart + CI/CD pipeline

เนื้อหาทั้งหมดเป็น Production-ready code ที่ใช้ได้จริงใน .NET 9.0
