# Part 91: gRPC Advanced — Streaming, Interceptors & Code-First

## Steps 2085–2100

---

## Step 2085: gRPC Fundamentals & Service Types

```
gRPC Service Types:
┌────────────────────────────────────────────────────────────┐
│  Unary          │  Client sends 1 request → 1 response     │
│  Server Stream  │  Client sends 1 request → stream back    │
│  Client Stream  │  Client streams → 1 response             │
│  Bidirectional  │  Full-duplex streaming both ways         │
└────────────────────────────────────────────────────────────┘

Benefits over REST:
- Binary Protocol Buffers (~5x smaller, faster serialize)
- HTTP/2 multiplexing (no head-of-line blocking)
- Strongly typed contracts via .proto
- Built-in deadline/cancellation propagation
- Code generation for client + server
```

```protobuf
// orders.proto
syntax = "proto3";
option csharp_namespace = "OrderService.Grpc";

package orders;

import "google/protobuf/timestamp.proto";
import "google/protobuf/empty.proto";

service OrderService {
  // Unary
  rpc GetOrder        (GetOrderRequest)        returns (OrderResponse);
  rpc PlaceOrder      (PlaceOrderRequest)      returns (PlaceOrderResponse);

  // Server streaming
  rpc StreamOrderEvents (StreamOrderEventsRequest) returns (stream OrderEvent);

  // Client streaming
  rpc BatchImportOrders (stream ImportOrderRequest) returns (BatchImportResponse);

  // Bidirectional streaming
  rpc TrackOrders (stream TrackOrderRequest) returns (stream OrderStatusUpdate);
}

message GetOrderRequest  { string order_id = 1; }
message PlaceOrderRequest {
  string customer_id = 1;
  repeated OrderLineItem items = 2;
  string currency = 3;
}
message PlaceOrderResponse {
  string order_id = 1;
  string status   = 2;
  google.protobuf.Timestamp created_at = 3;
}
message OrderResponse {
  string order_id    = 1;
  string customer_id = 2;
  string status      = 3;
  double total_amount = 4;
  string currency    = 5;
  repeated OrderLineItem items = 6;
  google.protobuf.Timestamp created_at = 7;
}
message OrderLineItem {
  string product_id = 1;
  int32  quantity   = 2;
  double unit_price = 3;
}
message OrderEvent {
  string order_id   = 1;
  string event_type = 2;
  string payload    = 3;
  google.protobuf.Timestamp occurred_at = 4;
}
message StreamOrderEventsRequest { string order_id = 1; }
message ImportOrderRequest  { OrderLineItem item = 1; string customer_id = 2; }
message BatchImportResponse { int32 imported_count = 1; repeated string errors = 2; }
message TrackOrderRequest   { string order_id = 1; }
message OrderStatusUpdate   { string order_id = 1; string status = 2; }
```

---

## Step 2086: gRPC Server Implementation

```xml
<!-- .csproj -->
<ItemGroup>
  <PackageReference Include="Grpc.AspNetCore" Version="2.65.0" />
  <PackageReference Include="Google.Protobuf" Version="3.27.2" />
  <PackageReference Include="Grpc.Tools" Version="2.65.0" PrivateAssets="All" />
</ItemGroup>
<ItemGroup>
  <Protobuf Include="Protos\orders.proto" GrpcServices="Server" />
</ItemGroup>
```

```csharp
// OrderGrpcService.cs
public class OrderGrpcService : OrderService.OrderServiceBase
{
    private readonly IOrderRepository _repo;
    private readonly IOrderEventStream _eventStream;
    private readonly ILogger<OrderGrpcService> _logger;

    public OrderGrpcService(IOrderRepository repo, IOrderEventStream eventStream, ILogger<OrderGrpcService> logger)
    {
        _repo        = repo;
        _eventStream = eventStream;
        _logger      = logger;
    }

    // Unary
    public override async Task<OrderResponse> GetOrder(GetOrderRequest request, ServerCallContext ctx)
    {
        var order = await _repo.GetByIdAsync(Guid.Parse(request.OrderId), ctx.CancellationToken);

        if (order is null)
            throw new RpcException(new Status(StatusCode.NotFound, $"Order {request.OrderId} not found"));

        return MapToResponse(order);
    }

    public override async Task<PlaceOrderResponse> PlaceOrder(PlaceOrderRequest request, ServerCallContext ctx)
    {
        var order = await _repo.CreateAsync(new CreateOrderModel
        {
            CustomerId = Guid.Parse(request.CustomerId),
            Items      = request.Items.Select(i => new OrderItem(
                Guid.Parse(i.ProductId), i.Quantity, (decimal)i.UnitPrice)).ToList()
        }, ctx.CancellationToken);

        return new PlaceOrderResponse
        {
            OrderId   = order.Id.ToString(),
            Status    = order.Status.ToString(),
            CreatedAt = Timestamp.FromDateTimeOffset(order.CreatedAt)
        };
    }

    // Server streaming
    public override async Task StreamOrderEvents(
        StreamOrderEventsRequest request,
        IServerStreamWriter<OrderEvent> responseStream,
        ServerCallContext ctx)
    {
        var orderId = Guid.Parse(request.OrderId);
        _logger.LogInformation("Streaming events for order {OrderId}", orderId);

        await foreach (var evt in _eventStream.SubscribeAsync(orderId, ctx.CancellationToken))
        {
            await responseStream.WriteAsync(new OrderEvent
            {
                OrderId    = evt.OrderId.ToString(),
                EventType  = evt.Type,
                Payload    = JsonSerializer.Serialize(evt.Payload),
                OccurredAt = Timestamp.FromDateTimeOffset(evt.OccurredAt)
            }, ctx.CancellationToken);
        }
    }

    // Client streaming
    public override async Task<BatchImportResponse> BatchImportOrders(
        IAsyncStreamReader<ImportOrderRequest> requestStream,
        ServerCallContext ctx)
    {
        var imported = 0;
        var errors   = new List<string>();

        await foreach (var request in requestStream.ReadAllAsync(ctx.CancellationToken))
        {
            try
            {
                await _repo.ImportItemAsync(request, ctx.CancellationToken);
                imported++;
            }
            catch (Exception ex)
            {
                errors.Add($"Item {request.Item?.ProductId}: {ex.Message}");
            }
        }

        return new BatchImportResponse
        {
            ImportedCount = imported,
            Errors        = { errors }
        };
    }

    // Bidirectional streaming
    public override async Task TrackOrders(
        IAsyncStreamReader<TrackOrderRequest> requestStream,
        IServerStreamWriter<OrderStatusUpdate> responseStream,
        ServerCallContext ctx)
    {
        var subscriptions = new ConcurrentDictionary<string, IDisposable>();

        try
        {
            await foreach (var request in requestStream.ReadAllAsync(ctx.CancellationToken))
            {
                var orderId = request.OrderId;

                // Subscribe to status changes for this order
                var sub = _eventStream.SubscribeToStatus(orderId, async status =>
                {
                    await responseStream.WriteAsync(new OrderStatusUpdate
                    {
                        OrderId = orderId,
                        Status  = status
                    }, ctx.CancellationToken);
                });

                subscriptions[orderId] = sub;
            }
        }
        finally
        {
            foreach (var sub in subscriptions.Values)
                sub.Dispose();
        }
    }

    private static OrderResponse MapToResponse(Order order) => new()
    {
        OrderId     = order.Id.ToString(),
        CustomerId  = order.CustomerId.ToString(),
        Status      = order.Status.ToString(),
        TotalAmount = (double)order.TotalAmount,
        Currency    = order.Currency,
        CreatedAt   = Timestamp.FromDateTimeOffset(order.CreatedAt),
        Items       =
        {
            order.Items.Select(i => new OrderLineItem
            {
                ProductId = i.ProductId.ToString(),
                Quantity  = i.Quantity,
                UnitPrice = (double)i.UnitPrice
            })
        }
    };
}

// Program.cs
builder.Services.AddGrpc(options =>
{
    options.MaxReceiveMessageSize = 4 * 1024 * 1024; // 4 MB
    options.MaxSendMessageSize    = 4 * 1024 * 1024;
    options.EnableDetailedErrors  = builder.Environment.IsDevelopment();
    options.CompressionProviders  = [new GzipCompressionProvider(CompressionLevel.Fastest)];
    options.ResponseCompressionAlgorithm = "gzip";
});

builder.Services.AddGrpcReflection(); // enables grpcurl / Postman

var app = builder.Build();
app.MapGrpcService<OrderGrpcService>();

if (app.Environment.IsDevelopment())
    app.MapGrpcReflectionService();
```

---

## Step 2087: gRPC Client — Named HttpClient with Deadline

```xml
<!-- Client project .csproj -->
<ItemGroup>
  <PackageReference Include="Grpc.Net.Client" Version="2.65.0" />
  <PackageReference Include="Grpc.Net.ClientFactory" Version="2.65.0" />
  <Protobuf Include="Protos\orders.proto" GrpcServices="Client" />
</ItemGroup>
```

```csharp
// Register gRPC client
builder.Services.AddGrpcClient<OrderService.OrderServiceClient>(options =>
{
    options.Address = new Uri(builder.Configuration["Services:OrderService:GrpcUrl"]!);
})
.ConfigurePrimaryHttpMessageHandler(() => new SocketsHttpHandler
{
    PooledConnectionIdleTimeout    = TimeSpan.FromMinutes(5),
    KeepAlivePingDelay             = TimeSpan.FromSeconds(60),
    KeepAlivePingTimeout           = TimeSpan.FromSeconds(30),
    EnableMultipleHttp2Connections = true
})
.AddCallCredentials(async (ctx, metadata, sp) =>
{
    // Attach JWT token to every gRPC call
    var tokenService = sp.GetRequiredService<ITokenService>();
    var token        = await tokenService.GetAccessTokenAsync();
    metadata.Add("Authorization", $"Bearer {token}");
})
.AddInterceptor<ClientLoggingInterceptor>()
.AddInterceptor<RetryInterceptor>();

// Service using gRPC client with deadline
public class OrderGrpcClient : IOrderClient
{
    private readonly OrderService.OrderServiceClient _client;
    private readonly ILogger<OrderGrpcClient> _logger;

    public OrderGrpcClient(OrderService.OrderServiceClient client, ILogger<OrderGrpcClient> logger)
    {
        _client = client;
        _logger = logger;
    }

    public async Task<OrderDto?> GetOrderAsync(Guid id, CancellationToken ct = default)
    {
        try
        {
            var deadline  = DateTime.UtcNow.AddSeconds(10); // absolute deadline
            var callOptions = new CallOptions(deadline: deadline, cancellationToken: ct);

            var response = await _client.GetOrderAsync(
                new GetOrderRequest { OrderId = id.ToString() },
                callOptions);

            return MapToDto(response);
        }
        catch (RpcException ex) when (ex.StatusCode == StatusCode.NotFound)
        {
            return null;
        }
        catch (RpcException ex) when (ex.StatusCode == StatusCode.DeadlineExceeded)
        {
            _logger.LogWarning("gRPC call timed out for order {OrderId}", id);
            throw new TimeoutException($"Order {id} retrieval timed out", ex);
        }
    }

    public async IAsyncEnumerable<OrderEventDto> StreamEventsAsync(
        Guid orderId,
        [EnumeratorCancellation] CancellationToken ct = default)
    {
        var call = _client.StreamOrderEvents(
            new StreamOrderEventsRequest { OrderId = orderId.ToString() },
            new CallOptions(cancellationToken: ct));

        await foreach (var evt in call.ResponseStream.ReadAllAsync(ct))
        {
            yield return new OrderEventDto(evt.EventType, evt.Payload,
                evt.OccurredAt.ToDateTimeOffset());
        }
    }

    private static OrderDto MapToDto(OrderResponse r) => new(
        Id:         Guid.Parse(r.OrderId),
        CustomerId: Guid.Parse(r.CustomerId),
        Status:     r.Status,
        Total:      (decimal)r.TotalAmount,
        Currency:   r.Currency);
}
```

---

## Step 2088: gRPC Interceptors

```csharp
// Server-side interceptor — authentication, logging, error handling
public class ServerAuthInterceptor : Interceptor
{
    private readonly IServiceProvider _sp;

    public ServerAuthInterceptor(IServiceProvider sp) => _sp = sp;

    public override async Task<TResponse> UnaryServerHandler<TRequest, TResponse>(
        TRequest request,
        ServerCallContext context,
        UnaryServerMethod<TRequest, TResponse> continuation)
    {
        await AuthenticateAsync(context);
        return await continuation(request, context);
    }

    public override async Task ServerStreamingServerHandler<TRequest, TResponse>(
        TRequest request,
        IServerStreamWriter<TResponse> responseStream,
        ServerCallContext context,
        ServerStreamingServerMethod<TRequest, TResponse> continuation)
    {
        await AuthenticateAsync(context);
        await continuation(request, responseStream, context);
    }

    private async Task AuthenticateAsync(ServerCallContext context)
    {
        var authHeader = context.RequestHeaders.GetValue("authorization");
        if (string.IsNullOrEmpty(authHeader))
            throw new RpcException(new Status(StatusCode.Unauthenticated, "Missing authorization"));

        var token = authHeader.Replace("Bearer ", "", StringComparison.OrdinalIgnoreCase);

        using var scope    = _sp.CreateScope();
        var validator      = scope.ServiceProvider.GetRequiredService<IJwtValidator>();
        var principal      = await validator.ValidateAsync(token);

        if (principal is null)
            throw new RpcException(new Status(StatusCode.Unauthenticated, "Invalid token"));

        context.UserState["user"] = principal;
    }
}

// Server-side global exception interceptor
public class ExceptionInterceptor : Interceptor
{
    private readonly ILogger<ExceptionInterceptor> _logger;

    public ExceptionInterceptor(ILogger<ExceptionInterceptor> logger) => _logger = logger;

    public override async Task<TResponse> UnaryServerHandler<TRequest, TResponse>(
        TRequest request,
        ServerCallContext context,
        UnaryServerMethod<TRequest, TResponse> continuation)
    {
        try
        {
            return await continuation(request, context);
        }
        catch (RpcException)
        {
            throw; // already formatted
        }
        catch (ValidationException ex)
        {
            throw new RpcException(new Status(StatusCode.InvalidArgument, ex.Message));
        }
        catch (NotFoundException ex)
        {
            throw new RpcException(new Status(StatusCode.NotFound, ex.Message));
        }
        catch (UnauthorizedAccessException ex)
        {
            throw new RpcException(new Status(StatusCode.PermissionDenied, ex.Message));
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unhandled error in gRPC call {Method}", context.Method);
            throw new RpcException(new Status(StatusCode.Internal, "An internal error occurred"));
        }
    }
}

// Client-side retry interceptor
public class RetryInterceptor : Interceptor
{
    private readonly ILogger<RetryInterceptor> _logger;

    public RetryInterceptor(ILogger<RetryInterceptor> logger) => _logger = logger;

    public override AsyncUnaryCall<TResponse> AsyncUnaryCall<TRequest, TResponse>(
        TRequest request,
        ClientInterceptorContext<TRequest, TResponse> context,
        AsyncUnaryCallContinuation<TRequest, TResponse> continuation)
    {
        return new AsyncUnaryCall<TResponse>(
            ExecuteWithRetryAsync(request, context, continuation),
            continuation(request, context).ResponseHeadersAsync,
            continuation(request, context).GetStatus,
            continuation(request, context).GetTrailers,
            continuation(request, context).Dispose);
    }

    private async Task<TResponse> ExecuteWithRetryAsync<TRequest, TResponse>(
        TRequest request,
        ClientInterceptorContext<TRequest, TResponse> context,
        AsyncUnaryCallContinuation<TRequest, TResponse> continuation)
        where TRequest : class
        where TResponse : class
    {
        const int maxRetries = 3;

        for (var attempt = 0; attempt < maxRetries; attempt++)
        {
            try
            {
                return await continuation(request, context).ResponseAsync;
            }
            catch (RpcException ex) when (IsRetryable(ex.StatusCode) && attempt < maxRetries - 1)
            {
                var delay = TimeSpan.FromMilliseconds(100 * Math.Pow(2, attempt));
                _logger.LogWarning("gRPC call failed ({Status}), retry {Attempt} in {Delay}ms",
                    ex.StatusCode, attempt + 1, delay.TotalMilliseconds);
                await Task.Delay(delay);
            }
        }

        return await continuation(request, context).ResponseAsync;
    }

    private static bool IsRetryable(StatusCode code) =>
        code is StatusCode.Unavailable or StatusCode.ResourceExhausted or StatusCode.DeadlineExceeded;
}

// Register interceptors
builder.Services.AddGrpc(o => { });
builder.Services.AddSingleton<ExceptionInterceptor>();
builder.Services.AddSingleton<ServerAuthInterceptor>();
app.MapGrpcService<OrderGrpcService>()
   .AddInterceptors<ExceptionInterceptor, ServerAuthInterceptor>();
```

---

## Step 2089: Code-First gRPC with protobuf-net

```bash
dotnet add package protobuf-net.Grpc.AspNetCore
dotnet add package protobuf-net.Grpc
```

```csharp
// Code-first: define contracts in C#, no .proto file needed
[ServiceContract]
public interface IInventoryService
{
    [OperationContract]
    ValueTask<StockLevelResponse> GetStockLevelAsync(StockLevelRequest request, CallContext ctx = default);

    [OperationContract]
    IAsyncEnumerable<StockUpdate> SubscribeToUpdatesAsync(SubscribeRequest request, CallContext ctx = default);
}

[ProtoContract]
public class StockLevelRequest
{
    [ProtoMember(1)] public Guid ProductId { get; set; }
}

[ProtoContract]
public class StockLevelResponse
{
    [ProtoMember(1)] public Guid   ProductId     { get; set; }
    [ProtoMember(2)] public int    Available     { get; set; }
    [ProtoMember(3)] public int    Reserved      { get; set; }
    [ProtoMember(4)] public string WarehouseCode { get; set; } = null!;
}

[ProtoContract]
public class StockUpdate
{
    [ProtoMember(1)] public Guid   ProductId { get; set; }
    [ProtoMember(2)] public int    NewLevel  { get; set; }
    [ProtoMember(3)] public DateTime UpdatedAt { get; set; }
}

[ProtoContract]
public class SubscribeRequest
{
    [ProtoMember(1)] public Guid ProductId { get; set; }
}

// Server implementation
public class InventoryGrpcService : IInventoryService
{
    private readonly IInventoryRepository _repo;
    private readonly IInventoryEventStream _stream;

    public InventoryGrpcService(IInventoryRepository repo, IInventoryEventStream stream)
    {
        _repo   = repo;
        _stream = stream;
    }

    public async ValueTask<StockLevelResponse> GetStockLevelAsync(
        StockLevelRequest request, CallContext ctx = default)
    {
        var stock = await _repo.GetAsync(request.ProductId, ctx.CancellationToken);
        return new StockLevelResponse
        {
            ProductId     = request.ProductId,
            Available     = stock.Available,
            Reserved      = stock.Reserved,
            WarehouseCode = stock.WarehouseCode
        };
    }

    public async IAsyncEnumerable<StockUpdate> SubscribeToUpdatesAsync(
        SubscribeRequest request,
        [EnumeratorCancellation] CallContext ctx = default)
    {
        await foreach (var update in _stream.SubscribeAsync(request.ProductId, ctx.CancellationToken))
        {
            yield return new StockUpdate
            {
                ProductId = request.ProductId,
                NewLevel  = update.Level,
                UpdatedAt = update.At.UtcDateTime
            };
        }
    }
}

// Program.cs — code-first setup
builder.Services.AddCodeFirstGrpc();

var app = builder.Build();
app.MapGrpcService<InventoryGrpcService>();
```

---

## Step 2090: gRPC-Web & Transcoding (REST bridge)

```csharp
// gRPC-Web: allows browser JS to call gRPC services
builder.Services.AddGrpc();
builder.Services.AddGrpcWeb(options => options.DefaultEnabled = true);

var app = builder.Build();
app.UseGrpcWeb();
app.MapGrpcService<OrderGrpcService>().EnableGrpcWeb();

// HTTP/JSON Transcoding: expose gRPC as REST automatically
// Requires Google.Api.CommonProtos package
```

```protobuf
// With transcoding annotations in .proto
import "google/api/annotations.proto";

service OrderService {
  rpc GetOrder (GetOrderRequest) returns (OrderResponse) {
    option (google.api.http) = {
      get: "/v1/orders/{order_id}"
    };
  }

  rpc PlaceOrder (PlaceOrderRequest) returns (PlaceOrderResponse) {
    option (google.api.http) = {
      post: "/v1/orders"
      body: "*"
    };
  }
}
```

```csharp
// Enable transcoding
builder.Services.AddGrpc().AddJsonTranscoding();
builder.Services.AddGrpcSwagger(); // Swagger UI for transcoded endpoints

// appsettings.json for transcoding
{
  "Grpc": {
    "Services": {
      "OrderService.Grpc.OrderService": {
        "JsonTranscoding": { "Enable": true }
      }
    }
  }
}
```

---

## Step 2091: gRPC Health Checks & Load Balancing

```csharp
// gRPC health checks (standard protocol)
builder.Services.AddGrpcHealthChecks()
    .AddCheck("orders-db", () =>
    {
        // check DB connectivity
        return HealthCheckResult.Healthy();
    });

var app = builder.Build();
app.MapGrpcHealthChecksService();

// Client-side load balancing
builder.Services.AddGrpcClient<OrderService.OrderServiceClient>(o =>
{
    // Multiple addresses for round-robin
    o.Address = new Uri("static:///order-service");
})
.ConfigureChannel(channel =>
{
    channel.Credentials = ChannelCredentials.SecureSsl;
    channel.ServiceConfig = new ServiceConfig
    {
        MethodConfigs =
        {
            new MethodConfig
            {
                Names = { MethodName.Default },
                RetryPolicy = new RetryPolicy
                {
                    MaxAttempts         = 5,
                    InitialBackoff      = TimeSpan.FromSeconds(1),
                    MaxBackoff          = TimeSpan.FromSeconds(5),
                    BackoffMultiplier   = 1.5,
                    RetryableStatusCodes = { StatusCode.Unavailable }
                }
            }
        },
        LoadBalancingConfigs = { new RoundRobinConfig() }
    };
});

// Static resolver for multiple addresses
builder.Services.AddSingleton<ResolverFactory>(new StaticResolverFactory(addr =>
[
    new BalancerAddress("order-svc-1.internal", 443),
    new BalancerAddress("order-svc-2.internal", 443),
    new BalancerAddress("order-svc-3.internal", 443),
]));
```

---

## Step 2092: gRPC Deadlines & Cancellation Propagation

```csharp
// Deadline propagation across service calls
public class OrderFulfillmentService
{
    private readonly OrderService.OrderServiceClient _orderClient;
    private readonly InventoryService.InventoryServiceClient _inventoryClient;

    public OrderFulfillmentService(
        OrderService.OrderServiceClient orderClient,
        InventoryService.InventoryServiceClient inventoryClient)
    {
        _orderClient     = orderClient;
        _inventoryClient = inventoryClient;
    }

    public async Task<FulfillmentResult> FulfillAsync(
        Guid orderId, DateTime overallDeadline, CancellationToken ct = default)
    {
        // Use the lesser of the overall deadline or per-operation deadline
        var orderDeadline = new DateTime[]
        {
            overallDeadline,
            DateTime.UtcNow.AddSeconds(5) // max 5s for order lookup
        }.Min();

        var order = await _orderClient.GetOrderAsync(
            new GetOrderRequest { OrderId = orderId.ToString() },
            new CallOptions(deadline: orderDeadline, cancellationToken: ct));

        // Remaining time for inventory check
        var remaining = overallDeadline - DateTime.UtcNow;
        if (remaining <= TimeSpan.Zero)
            throw new RpcException(new Status(StatusCode.DeadlineExceeded, "Overall deadline exceeded"));

        var inventoryDeadline = DateTime.UtcNow.Add(remaining);

        // Parallel gRPC calls with shared deadline
        var tasks = order.Items.Select(item => _inventoryClient.CheckStockAsync(
            new CheckStockRequest { ProductId = item.ProductId, Quantity = item.Quantity },
            new CallOptions(deadline: inventoryDeadline, cancellationToken: ct)).ResponseAsync);

        var stockResults = await Task.WhenAll(tasks);

        return new FulfillmentResult(orderId, stockResults.All(r => r.IsAvailable));
    }
}

// Propagate context (deadline + metadata) from incoming gRPC call to outgoing
public class DeadlinePropagationInterceptor : Interceptor
{
    public override AsyncUnaryCall<TResponse> AsyncUnaryCall<TRequest, TResponse>(
        TRequest request,
        ClientInterceptorContext<TRequest, TResponse> context,
        AsyncUnaryCallContinuation<TRequest, TResponse> continuation)
    {
        // If the current server call has a deadline, propagate it to outgoing calls
        var serverContext = GrpcServerCallContext.Current;
        if (serverContext?.Deadline != DateTime.MaxValue)
        {
            var existingOptions = context.Options;
            var deadline        = serverContext?.Deadline ?? DateTime.UtcNow.AddSeconds(30);

            context = new ClientInterceptorContext<TRequest, TResponse>(
                context.Method,
                context.Host,
                existingOptions.WithDeadline(deadline));
        }

        return continuation(request, context);
    }
}
```

---

## Step 2093: gRPC Authentication & Authorization

```csharp
// JWT bearer authentication for gRPC
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = "https://identity.example.com";
        options.Audience  = "grpc-api";

        // gRPC uses HTTP/2 — handle metadata
        options.Events = new JwtBearerEvents
        {
            OnMessageReceived = ctx =>
            {
                // For gRPC, token comes in Authorization metadata
                ctx.Token = ctx.Request.Headers.Authorization
                    .FirstOrDefault()?.Replace("Bearer ", "");
                return Task.CompletedTask;
            }
        };
    });

builder.Services.AddAuthorization();

var app = builder.Build();
app.UseAuthentication();
app.UseAuthorization();
app.MapGrpcService<OrderGrpcService>()
   .RequireAuthorization(); // all methods require auth

// Per-method authorization
public class OrderGrpcService : OrderService.OrderServiceBase
{
    [Authorize(Policy = "orders:read")]
    public override Task<OrderResponse> GetOrder(GetOrderRequest request, ServerCallContext ctx)
        => base.GetOrder(request, ctx);

    [Authorize(Policy = "orders:write")]
    public override Task<PlaceOrderResponse> PlaceOrder(PlaceOrderRequest request, ServerCallContext ctx)
        => base.PlaceOrder(request, ctx);

    [Authorize(Roles = "admin")]
    public override Task<BatchImportResponse> BatchImportOrders(
        IAsyncStreamReader<ImportOrderRequest> requestStream, ServerCallContext ctx)
        => base.BatchImportOrders(requestStream, ctx);
}

// TLS configuration for gRPC (mutual TLS for service-to-service)
builder.WebHost.ConfigureKestrel(options =>
{
    options.ListenAnyIP(5001, listenOptions =>
    {
        listenOptions.Protocols = HttpProtocols.Http2;
        listenOptions.UseHttps(httpsOptions =>
        {
            httpsOptions.ServerCertificate = LoadCertificate();
            // For mTLS:
            httpsOptions.ClientCertificateMode = ClientCertificateMode.RequireCertificate;
        });
    });
});
```

---

## Step 2094: gRPC Compression & Performance Tuning

```csharp
// Compression on server
builder.Services.AddGrpc(options =>
{
    options.CompressionProviders = new List<ICompressionProvider>
    {
        new GzipCompressionProvider(CompressionLevel.Optimal),
        new BrotliCompressionProvider(CompressionLevel.Optimal)
    };
    options.ResponseCompressionAlgorithm      = "br";  // brotli for responses
    options.ResponseCompressionLevel          = CompressionLevel.Optimal;
    options.ReceiveMessageSize                = 16 * 1024 * 1024;  // 16 MB max inbound
    options.SendMessageSize                   = 16 * 1024 * 1024;  // 16 MB max outbound
});

// HTTP/2 keep-alive tuning
builder.WebHost.ConfigureKestrel(options =>
{
    options.Limits.Http2.InitialConnectionWindowSize      = 131072;      // 128 KB
    options.Limits.Http2.InitialStreamWindowSize          = 98304;       // 96 KB
    options.Limits.Http2.MaxStreamsPerConnection           = 100;
    options.Limits.Http2.KeepAlivePingDelay               = TimeSpan.FromSeconds(30);
    options.Limits.Http2.KeepAlivePingTimeout             = TimeSpan.FromSeconds(5);
});

// Client compression
builder.Services.AddGrpcClient<OrderService.OrderServiceClient>(o =>
{
    o.Address = new Uri("https://order-service");
})
.ConfigureChannel(channel =>
{
    channel.CompressionProviders =
    [
        new GzipCompressionProvider(CompressionLevel.Fastest)
    ];
});

// Benchmark: gRPC vs HTTP/JSON
// Typical results:
// gRPC (protobuf):  ~0.5ms latency, ~5KB payload
// REST (JSON):      ~2ms latency,  ~25KB payload
// gRPC (protobuf) is 5-10x more efficient for structured data
```

---

## Step 2095: gRPC with Service Discovery

```csharp
// Consul-based service discovery for gRPC
builder.Services.AddSingleton<ResolverFactory>(sp =>
{
    var consulClient = sp.GetRequiredService<IConsulClient>();
    return new ConsulGrpcResolverFactory(consulClient);
});

builder.Services.AddGrpcClient<OrderService.OrderServiceClient>(o =>
{
    o.Address = new Uri("consul://order-service"); // resolved by Consul
});

// Custom resolver factory
public class ConsulGrpcResolverFactory : ResolverFactory
{
    private readonly IConsulClient _consul;
    public ConsulGrpcResolverFactory(IConsulClient consul) => _consul = consul;

    public override string Name => "consul";

    public override Resolver Create(ResolverOptions options)
        => new ConsulGrpcResolver(options.Address, _consul);
}

public class ConsulGrpcResolver : PollingResolver
{
    private readonly string _serviceName;
    private readonly IConsulClient _consul;

    public ConsulGrpcResolver(Uri address, IConsulClient consul)
        : base(loggerFactory: null)
    {
        _serviceName = address.Host;
        _consul      = consul;
    }

    protected override async Task ResolveAsync(CancellationToken ct)
    {
        var result = await _consul.Health.Service(_serviceName, tag: "grpc", passingOnly: true, ct);

        var addresses = result.Response
            .Select(entry => new BalancerAddress(entry.Service.Address, entry.Service.Port))
            .ToList();

        Listener(ResolverResult.ForResult(addresses));
    }
}
```

---

## Step 2096: gRPC Testing

```csharp
// Unit testing gRPC services
public class OrderGrpcServiceTests
{
    private readonly Mock<IOrderRepository> _repoMock;
    private readonly OrderGrpcService _service;

    public OrderGrpcServiceTests()
    {
        _repoMock = new Mock<IOrderRepository>();
        _service  = new OrderGrpcService(
            _repoMock.Object,
            Mock.Of<IOrderEventStream>(),
            Mock.Of<ILogger<OrderGrpcService>>());
    }

    [Fact]
    public async Task GetOrder_ExistingOrder_ReturnsOrder()
    {
        // Arrange
        var orderId = Guid.NewGuid();
        var order   = CreateTestOrder(orderId);
        _repoMock.Setup(r => r.GetByIdAsync(orderId, It.IsAny<CancellationToken>()))
                 .ReturnsAsync(order);

        var request = new GetOrderRequest { OrderId = orderId.ToString() };
        var ctx     = TestServerCallContext.Create();

        // Act
        var response = await _service.GetOrder(request, ctx);

        // Assert
        response.OrderId.Should().Be(orderId.ToString());
        response.Status.Should().Be("Pending");
    }

    [Fact]
    public async Task GetOrder_NotFound_ThrowsRpcException()
    {
        _repoMock.Setup(r => r.GetByIdAsync(It.IsAny<Guid>(), It.IsAny<CancellationToken>()))
                 .ReturnsAsync((Order?)null);

        var act = () => _service.GetOrder(
            new GetOrderRequest { OrderId = Guid.NewGuid().ToString() },
            TestServerCallContext.Create());

        await act.Should().ThrowAsync<RpcException>()
            .Where(ex => ex.StatusCode == StatusCode.NotFound);
    }

    private static Order CreateTestOrder(Guid id) => new()
    {
        Id         = id,
        CustomerId = Guid.NewGuid(),
        Status     = OrderStatus.Pending,
        TotalAmount = 100m,
        Currency   = "USD",
        Items      = [],
        CreatedAt  = DateTimeOffset.UtcNow
    };
}

// Integration test with TestServer
public class GrpcIntegrationTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;

    public GrpcIntegrationTests(WebApplicationFactory<Program> factory)
        => _factory = factory;

    [Fact]
    public async Task PlaceOrder_ValidRequest_ReturnsCreated()
    {
        var client  = _factory.CreateDefaultClient(new GrpcWebHandler(GrpcWebMode.GrpcWeb));
        var channel = GrpcChannel.ForAddress(_factory.Server.BaseAddress,
            new GrpcChannelOptions { HttpClient = client });

        var grpc    = new OrderService.OrderServiceClient(channel);
        var request = new PlaceOrderRequest
        {
            CustomerId = Guid.NewGuid().ToString(),
            Items      = { new OrderLineItem { ProductId = Guid.NewGuid().ToString(), Quantity = 2, UnitPrice = 50 } },
            Currency   = "USD"
        };

        var response = await grpc.PlaceOrderAsync(request);

        response.OrderId.Should().NotBeNullOrEmpty();
        response.Status.Should().Be("Pending");
    }
}
```

**Summary**: Part 91 covers gRPC service types (unary/streaming), full .proto definition, server implementation with all 4 service types, typed gRPC client with deadline support, server/client interceptors (auth, exception mapping, retry), code-first gRPC with protobuf-net, gRPC-Web and HTTP transcoding, health checks, load balancing, deadline propagation, mTLS, compression tuning, Consul service discovery, and unit/integration testing. Steps 2085–2100 complete.
