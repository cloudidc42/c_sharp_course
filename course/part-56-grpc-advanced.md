# Part 56: gRPC Advanced — Bidirectional Streaming, Interceptors & Production Patterns

## Steps 1532-1550: World-Class gRPC in .NET

---

## Step 1532: gRPC Architecture & Protocol Deep Dive

```
gRPC Communication Patterns
═══════════════════════════════════════════════════════════

  Client                              Server
  ──────                              ──────
  
  Unary RPC (request-response)
  ┌────────────────────┐   Request    ┌─────────────────────┐
  │  await stub.Call() │ ──────────► │  Handle() → return  │
  └────────────────────┘ ◄────────── └─────────────────────┘
                           Response
  
  Server Streaming (one request, many responses)
  ┌────────────────────┐   Request    ┌─────────────────────┐
  │  await foreach     │ ──────────► │  while(hasMore)     │
  │    responseStream  │ ◄────────── │    yield response   │
  └────────────────────┘  Stream...  └─────────────────────┘
  
  Client Streaming (many requests, one response)
  ┌────────────────────┐  Stream...  ┌─────────────────────┐
  │  while(hasMore)    │ ──────────► │  await foreach req  │
  │    await Write()   │ ◄────────── │  return summary     │
  └────────────────────┘  Response  └─────────────────────┘
  
  Bidirectional Streaming (full-duplex)
  ┌────────────────────┐  Stream...  ┌─────────────────────┐
  │  Task WriteLoop()  │ ──────────► │  Read & Process     │
  │  Task ReadLoop()   │ ◄────────── │  Write Responses    │
  └────────────────────┘  Stream...  └─────────────────────┘

Protocol:
  HTTP/2 multiplexed streams → binary Protobuf encoding
  TLS by default → header compression (HPACK)
  Flow control per stream → concurrent requests on 1 connection
```

### Project Structure

```
GrpcAdvanced/
├── Protos/
│   ├── order.proto
│   ├── inventory.proto
│   └── analytics.proto
├── Services/
│   ├── OrderService.cs
│   ├── InventoryService.cs
│   └── AnalyticsService.cs
├── Interceptors/
│   ├── AuthInterceptor.cs
│   ├── LoggingInterceptor.cs
│   └── RetryInterceptor.cs
├── Clients/
│   ├── OrderGrpcClient.cs
│   └── InventoryGrpcClient.cs
└── Program.cs
```

---

## Step 1533: Proto Definitions — Full Production Schema

```protobuf
// Protos/order.proto
syntax = "proto3";

option csharp_namespace = "GrpcAdvanced.Protos";

import "google/protobuf/timestamp.proto";
import "google/protobuf/empty.proto";
import "google/protobuf/wrappers.proto";

package orders;

// ── Enums ──────────────────────────────────────────────────────────────────

enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;
  ORDER_STATUS_PENDING     = 1;
  ORDER_STATUS_CONFIRMED   = 2;
  ORDER_STATUS_PROCESSING  = 3;
  ORDER_STATUS_SHIPPED     = 4;
  ORDER_STATUS_DELIVERED   = 5;
  ORDER_STATUS_CANCELLED   = 6;
}

// ── Messages ───────────────────────────────────────────────────────────────

message Money {
  string currency_code = 1;
  int64  units         = 2;   // whole units
  int32  nanos         = 3;   // 0-999999999
}

message OrderItem {
  string product_id  = 1;
  string name        = 2;
  int32  quantity    = 3;
  Money  unit_price  = 4;
}

message Order {
  string                    id          = 1;
  string                    customer_id = 2;
  repeated OrderItem        items       = 3;
  Money                     total       = 4;
  OrderStatus               status      = 5;
  google.protobuf.Timestamp created_at  = 6;
  google.protobuf.Timestamp updated_at  = 7;
  map<string, string>       metadata    = 8;
}

message CreateOrderRequest {
  string             customer_id = 1;
  repeated OrderItem items       = 2;
  map<string,string> metadata    = 3;
}

message CreateOrderResponse {
  Order                     order    = 1;
  bool                      success  = 2;
  repeated ValidationError  errors   = 3;
}

message ValidationError {
  string field   = 1;
  string message = 2;
  string code    = 3;
}

message GetOrderRequest {
  string id = 1;
}

message ListOrdersRequest {
  string      customer_id  = 1;
  OrderStatus status       = 2;
  int32       page_size    = 3;
  string      page_token   = 4;
  string      order_by     = 5;
}

message ListOrdersResponse {
  repeated Order orders          = 1;
  string         next_page_token = 2;
  int32          total_count     = 3;
}

message UpdateOrderStatusRequest {
  string      order_id = 1;
  OrderStatus status   = 2;
  string      reason   = 3;
}

message BatchUpdateRequest {
  repeated UpdateOrderStatusRequest updates = 1;
}

message BatchUpdateResponse {
  int32          succeeded = 1;
  int32          failed    = 2;
  repeated Error errors    = 3;
}

message Error {
  string order_id = 1;
  string message  = 2;
  int32  code     = 3;
}

message OrderEvent {
  string                    order_id   = 1;
  OrderStatus               old_status = 2;
  OrderStatus               new_status = 3;
  google.protobuf.Timestamp occurred   = 4;
  string                    actor      = 5;
}

message WatchOrdersRequest {
  string customer_id = 1;
  repeated OrderStatus status_filter = 2;
}

message StreamProcessRequest {
  oneof payload {
    CreateOrderRequest create_order   = 1;
    UpdateOrderStatusRequest update   = 2;
    GetOrderRequest          get      = 3;
  }
  string request_id = 4;
}

message StreamProcessResponse {
  oneof result {
    CreateOrderResponse  created = 1;
    Order                order   = 2;
    Error                error   = 3;
  }
  string request_id = 4;
  int64  latency_ms = 5;
}

// ── Service ────────────────────────────────────────────────────────────────

service OrderService {
  // Unary
  rpc CreateOrder    (CreateOrderRequest)       returns (CreateOrderResponse);
  rpc GetOrder       (GetOrderRequest)          returns (Order);
  rpc DeleteOrder    (GetOrderRequest)          returns (google.protobuf.Empty);

  // Server streaming — client receives a stream of results
  rpc ListOrders     (ListOrdersRequest)        returns (stream Order);
  rpc WatchOrders    (WatchOrdersRequest)       returns (stream OrderEvent);

  // Client streaming — client sends a stream, gets one response
  rpc BatchUpdate    (stream UpdateOrderStatusRequest) returns (BatchUpdateResponse);

  // Bidirectional streaming — full-duplex processing
  rpc ProcessStream  (stream StreamProcessRequest) returns (stream StreamProcessResponse);
}
```

```protobuf
// Protos/inventory.proto
syntax = "proto3";
option csharp_namespace = "GrpcAdvanced.Protos";

package inventory;

message CheckStockRequest {
  string product_id = 1;
  int32  quantity   = 2;
}

message StockCheckResult {
  string product_id    = 1;
  bool   available     = 2;
  int32  current_stock = 3;
  string warehouse     = 4;
}

message ReserveStockRequest {
  string order_id    = 1;
  string product_id  = 2;
  int32  quantity    = 3;
}

message ReserveStockResponse {
  bool   success      = 1;
  string reservation  = 2;
  string message      = 3;
}

service InventoryService {
  rpc CheckStock   (CheckStockRequest)                  returns (StockCheckResult);
  rpc CheckBulk    (stream CheckStockRequest)           returns (stream StockCheckResult);
  rpc ReserveStock (ReserveStockRequest)                returns (ReserveStockResponse);
}
```

---

## Step 1534: Server Implementation — All Four Patterns

```csharp
// Services/OrderService.cs
using Grpc.Core;
using GrpcAdvanced.Protos;
using Google.Protobuf.WellKnownTypes;

namespace GrpcAdvanced.Services;

public class OrderGrpcService : OrderService.OrderServiceBase
{
    private readonly IOrderRepository _repo;
    private readonly ILogger<OrderGrpcService> _log;
    private readonly IEventBus _events;

    public OrderGrpcService(
        IOrderRepository repo,
        ILogger<OrderGrpcService> log,
        IEventBus events)
    {
        _repo = repo;
        _log = log;
        _events = events;
    }

    // ── Unary ─────────────────────────────────────────────────────────────

    public override async Task<CreateOrderResponse> CreateOrder(
        CreateOrderRequest request,
        ServerCallContext context)
    {
        var cancellation = context.CancellationToken;

        // Validate
        var errors = ValidateCreateOrder(request);
        if (errors.Any())
        {
            return new CreateOrderResponse
            {
                Success = false,
                Errors  = { errors }
            };
        }

        try
        {
            var order = await _repo.CreateAsync(MapToOrder(request), cancellation);

            // Add custom trailer headers
            context.ResponseTrailers.Add("x-order-id", order.Id);

            return new CreateOrderResponse
            {
                Order   = MapToProto(order),
                Success = true
            };
        }
        catch (DomainException ex)
        {
            // Rich status with details
            var status = new Grpc.Core.Status(
                StatusCode.FailedPrecondition,
                ex.Message);
            throw new RpcException(status, ex.Message);
        }
    }

    public override async Task<Order> GetOrder(
        GetOrderRequest request,
        ServerCallContext context)
    {
        if (!Guid.TryParse(request.Id, out _))
            throw new RpcException(new Grpc.Core.Status(
                StatusCode.InvalidArgument, "Invalid order id format"));

        var order = await _repo.GetByIdAsync(request.Id, context.CancellationToken);

        if (order is null)
            throw new RpcException(new Grpc.Core.Status(
                StatusCode.NotFound, $"Order {request.Id} not found"));

        return MapToProto(order);
    }

    // ── Server Streaming ──────────────────────────────────────────────────

    public override async Task ListOrders(
        ListOrdersRequest request,
        IServerStreamWriter<Order> responseStream,
        ServerCallContext context)
    {
        var ct = context.CancellationToken;

        var filter = new OrderFilter(
            CustomerId: request.CustomerId,
            Status: (OrderStatus?)request.Status,
            PageSize: request.PageSize > 0 ? request.PageSize : 100);

        // Stream results page by page
        string? cursor = request.PageToken;
        do
        {
            var page = await _repo.ListAsync(filter with { Cursor = cursor }, ct);

            foreach (var order in page.Items)
            {
                ct.ThrowIfCancellationRequested();
                await responseStream.WriteAsync(MapToProto(order), ct);
            }

            cursor = page.NextCursor;
        }
        while (cursor is not null);
    }

    public override async Task WatchOrders(
        WatchOrdersRequest request,
        IServerStreamWriter<OrderEvent> responseStream,
        ServerCallContext context)
    {
        var ct = context.CancellationToken;
        _log.LogInformation("Client watching orders for customer {Id}", request.CustomerId);

        var statusFilter = request.StatusFilter.Select(s => (OrderStatus)s).ToHashSet();

        // Subscribe to event bus
        await foreach (var evt in _events.SubscribeAsync<OrderStatusChangedEvent>(ct))
        {
            if (evt.CustomerId != request.CustomerId)
                continue;

            if (statusFilter.Count > 0 && !statusFilter.Contains(evt.NewStatus))
                continue;

            await responseStream.WriteAsync(new OrderEvent
            {
                OrderId    = evt.OrderId,
                OldStatus  = (Protos.OrderStatus)evt.OldStatus,
                NewStatus  = (Protos.OrderStatus)evt.NewStatus,
                OccurredAt = Timestamp.FromDateTime(evt.OccurredAt),
                Actor      = evt.Actor
            }, ct);
        }
    }

    // ── Client Streaming ──────────────────────────────────────────────────

    public override async Task<BatchUpdateResponse> BatchUpdate(
        IAsyncStreamReader<UpdateOrderStatusRequest> requestStream,
        ServerCallContext context)
    {
        var ct = context.CancellationToken;
        int succeeded = 0, failed = 0;
        var errors = new List<Error>();

        await foreach (var req in requestStream.ReadAllAsync(ct))
        {
            try
            {
                await _repo.UpdateStatusAsync(req.OrderId, (OrderStatus)req.Status, req.Reason, ct);
                succeeded++;
            }
            catch (Exception ex)
            {
                failed++;
                errors.Add(new Error
                {
                    OrderId = req.OrderId,
                    Message = ex.Message,
                    Code    = ex is NotFoundException ? 404 : 500
                });
                _log.LogWarning(ex, "Batch update failed for order {Id}", req.OrderId);
            }
        }

        return new BatchUpdateResponse
        {
            Succeeded = succeeded,
            Failed    = failed,
            Errors    = { errors }
        };
    }

    // ── Bidirectional Streaming ───────────────────────────────────────────

    public override async Task ProcessStream(
        IAsyncStreamReader<StreamProcessRequest> requestStream,
        IServerStreamWriter<StreamProcessResponse> responseStream,
        ServerCallContext context)
    {
        var ct = context.CancellationToken;

        // Process requests concurrently with bounded parallelism
        var semaphore = new SemaphoreSlim(10, 10); // max 10 concurrent
        var pending   = new List<Task>();

        await foreach (var request in requestStream.ReadAllAsync(ct))
        {
            await semaphore.WaitAsync(ct);

            var task = ProcessRequestAsync(request, responseStream, semaphore, ct);
            pending.Add(task);

            // Clean up completed tasks
            pending.RemoveAll(t => t.IsCompleted);
        }

        await Task.WhenAll(pending);
    }

    private async Task ProcessRequestAsync(
        StreamProcessRequest request,
        IServerStreamWriter<StreamProcessResponse> writer,
        SemaphoreSlim semaphore,
        CancellationToken ct)
    {
        var sw = System.Diagnostics.Stopwatch.StartNew();
        StreamProcessResponse response;

        try
        {
            response = request.PayloadCase switch
            {
                StreamProcessRequest.PayloadOneofCase.CreateOrder =>
                    await HandleCreateAsync(request.CreateOrder, request.RequestId, ct),

                StreamProcessRequest.PayloadOneofCase.Update =>
                    await HandleUpdateAsync(request.Update, request.RequestId, ct),

                StreamProcessRequest.PayloadOneofCase.Get =>
                    await HandleGetAsync(request.Get, request.RequestId, ct),

                _ => new StreamProcessResponse
                {
                    RequestId = request.RequestId,
                    Error = new Error { Message = "Unknown request type", Code = 400 }
                }
            };
        }
        catch (Exception ex)
        {
            response = new StreamProcessResponse
            {
                RequestId = request.RequestId,
                Error     = new Error { Message = ex.Message, Code = 500 }
            };
        }
        finally
        {
            semaphore.Release();
            sw.Stop();
        }

        response.LatencyMs = sw.ElapsedMilliseconds;
        await writer.WriteAsync(response, ct);
    }

    // ── Helpers ───────────────────────────────────────────────────────────

    private List<ValidationError> ValidateCreateOrder(CreateOrderRequest req)
    {
        var errors = new List<ValidationError>();
        if (string.IsNullOrEmpty(req.CustomerId))
            errors.Add(new ValidationError { Field = "customer_id", Message = "Required", Code = "REQUIRED" });
        if (req.Items.Count == 0)
            errors.Add(new ValidationError { Field = "items", Message = "At least one item required", Code = "MIN_ITEMS" });
        foreach (var item in req.Items.Where(i => i.Quantity <= 0))
            errors.Add(new ValidationError { Field = "items.quantity", Message = $"Quantity must be positive for {item.ProductId}", Code = "POSITIVE_INT" });
        return errors;
    }

    private static Order MapToProto(OrderEntity order) => new()
    {
        Id         = order.Id.ToString(),
        CustomerId = order.CustomerId,
        Status     = (Protos.OrderStatus)order.Status,
        Total      = new Money
        {
            CurrencyCode = "USD",
            Units        = (long)order.Total,
            Nanos        = (int)((order.Total % 1) * 1_000_000_000)
        },
        CreatedAt = Timestamp.FromDateTime(order.CreatedAt.ToUniversalTime()),
        Items     = { order.Items.Select(i => new OrderItem
        {
            ProductId = i.ProductId,
            Name      = i.Name,
            Quantity  = i.Quantity,
            UnitPrice = new Money { CurrencyCode = "USD", Units = (long)i.UnitPrice }
        })}
    };
}
```

---

## Step 1535: Server-Side Interceptors

```csharp
// Interceptors/LoggingInterceptor.cs
using Grpc.Core;
using Grpc.Core.Interceptors;
using System.Diagnostics;

namespace GrpcAdvanced.Interceptors;

public class LoggingInterceptor : Interceptor
{
    private readonly ILogger<LoggingInterceptor> _log;

    public LoggingInterceptor(ILogger<LoggingInterceptor> log) => _log = log;

    // Intercept unary calls
    public override async Task<TResponse> UnaryServerHandler<TRequest, TResponse>(
        TRequest request,
        ServerCallContext context,
        UnaryServerMethod<TRequest, TResponse> continuation)
    {
        var method = context.Method;
        var sw     = Stopwatch.StartNew();

        _log.LogInformation("gRPC {Method} started | Peer: {Peer}",
            method, context.Peer);

        try
        {
            var response = await continuation(request, context);
            _log.LogInformation("gRPC {Method} succeeded in {Ms}ms", method, sw.ElapsedMilliseconds);
            return response;
        }
        catch (RpcException ex)
        {
            _log.LogWarning(ex, "gRPC {Method} failed with {Status} in {Ms}ms",
                method, ex.StatusCode, sw.ElapsedMilliseconds);
            throw;
        }
        catch (Exception ex)
        {
            _log.LogError(ex, "gRPC {Method} threw unhandled exception in {Ms}ms",
                method, sw.ElapsedMilliseconds);
            throw new RpcException(
                new Grpc.Core.Status(StatusCode.Internal, "Internal server error"),
                ex.Message);
        }
    }

    // Intercept server streaming calls
    public override async Task ServerStreamingServerHandler<TRequest, TResponse>(
        TRequest request,
        IServerStreamWriter<TResponse> responseStream,
        ServerCallContext context,
        ServerStreamingServerMethod<TRequest, TResponse> continuation)
    {
        var sw = Stopwatch.StartNew();
        _log.LogInformation("gRPC stream {Method} started", context.Method);

        try
        {
            await continuation(request, responseStream, context);
            _log.LogInformation("gRPC stream {Method} completed in {Ms}ms",
                context.Method, sw.ElapsedMilliseconds);
        }
        catch (Exception ex)
        {
            _log.LogError(ex, "gRPC stream {Method} failed", context.Method);
            throw;
        }
    }

    // Intercept client streaming
    public override async Task<TResponse> ClientStreamingServerHandler<TRequest, TResponse>(
        IAsyncStreamReader<TRequest> requestStream,
        ServerCallContext context,
        ClientStreamingServerMethod<TRequest, TResponse> continuation)
    {
        _log.LogInformation("gRPC client stream {Method} started", context.Method);
        return await continuation(requestStream, context);
    }

    // Intercept bidirectional streaming
    public override async Task DuplexStreamingServerHandler<TRequest, TResponse>(
        IAsyncStreamReader<TRequest> requestStream,
        IServerStreamWriter<TResponse> responseStream,
        ServerCallContext context,
        DuplexStreamingServerMethod<TRequest, TResponse> continuation)
    {
        _log.LogInformation("gRPC bidi stream {Method} started", context.Method);
        await continuation(requestStream, responseStream, context);
    }
}
```

```csharp
// Interceptors/AuthInterceptor.cs
using Grpc.Core;
using Grpc.Core.Interceptors;
using Microsoft.AspNetCore.Authorization;
using System.Security.Claims;

namespace GrpcAdvanced.Interceptors;

public class AuthInterceptor : Interceptor
{
    private readonly IAuthorizationService _authz;
    private readonly IHttpContextAccessor _http;

    public AuthInterceptor(IAuthorizationService authz, IHttpContextAccessor http)
    {
        _authz = authz;
        _http  = http;
    }

    public override async Task<TResponse> UnaryServerHandler<TRequest, TResponse>(
        TRequest request,
        ServerCallContext context,
        UnaryServerMethod<TRequest, TResponse> continuation)
    {
        // Extract JWT from gRPC metadata
        var auth = context.RequestHeaders.GetValue("authorization");
        if (auth is null)
            throw new RpcException(new Grpc.Core.Status(StatusCode.Unauthenticated, "Missing token"));

        // Validate and set user principal (delegated to ASP.NET Core middleware)
        var user = _http.HttpContext?.User;
        if (user?.Identity?.IsAuthenticated != true)
            throw new RpcException(new Grpc.Core.Status(StatusCode.Unauthenticated, "Invalid token"));

        // Check required policy
        var policy = GetRequiredPolicy(context.Method);
        if (policy is not null)
        {
            var result = await _authz.AuthorizeAsync(user, policy);
            if (!result.Succeeded)
                throw new RpcException(
                    new Grpc.Core.Status(StatusCode.PermissionDenied, "Insufficient permissions"));
        }

        // Add caller info to context items
        context.UserState["user_id"]   = user?.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        context.UserState["user_role"] = user?.FindFirst(ClaimTypes.Role)?.Value;

        return await continuation(request, context);
    }

    private static string? GetRequiredPolicy(string method) => method switch
    {
        var m when m.Contains("Create") => "WritePolicy",
        var m when m.Contains("Delete") => "AdminPolicy",
        _                               => null   // no auth required
    };
}
```

```csharp
// Interceptors/RetryInterceptor.cs — client-side
using Grpc.Core;
using Grpc.Core.Interceptors;

namespace GrpcAdvanced.Interceptors;

public class RetryInterceptor : Interceptor
{
    private readonly int _maxRetries;
    private readonly TimeSpan _baseDelay;

    public RetryInterceptor(int maxRetries = 3, TimeSpan? baseDelay = null)
    {
        _maxRetries = maxRetries;
        _baseDelay  = baseDelay ?? TimeSpan.FromMilliseconds(100);
    }

    public override AsyncUnaryCall<TResponse> AsyncUnaryCall<TRequest, TResponse>(
        TRequest request,
        ClientInterceptorContext<TRequest, TResponse> context,
        AsyncUnaryCallContinuation<TRequest, TResponse> continuation)
    {
        return new AsyncUnaryCall<TResponse>(
            RetryAsync(request, context, continuation),
            GetResponseHeadersAsync(request, context, continuation),
            () => Grpc.Core.Status.DefaultSuccess,
            () => new Grpc.Core.Metadata(),
            () => { });
    }

    private async Task<TResponse> RetryAsync<TRequest, TResponse>(
        TRequest request,
        ClientInterceptorContext<TRequest, TResponse> context,
        AsyncUnaryCallContinuation<TRequest, TResponse> continuation)
        where TRequest : class
        where TResponse : class
    {
        for (int attempt = 0; attempt <= _maxRetries; attempt++)
        {
            try
            {
                var call = continuation(request, context);
                return await call.ResponseAsync;
            }
            catch (RpcException ex) when (IsRetryable(ex.StatusCode) && attempt < _maxRetries)
            {
                var delay = _baseDelay * Math.Pow(2, attempt);
                await Task.Delay(delay);
            }
        }

        throw new RpcException(new Grpc.Core.Status(StatusCode.Unavailable, "All retries exhausted"));
    }

    private static async Task<Grpc.Core.Metadata> GetResponseHeadersAsync<TRequest, TResponse>(
        TRequest request,
        ClientInterceptorContext<TRequest, TResponse> context,
        AsyncUnaryCallContinuation<TRequest, TResponse> continuation)
        where TRequest : class
        where TResponse : class
    {
        try
        {
            var call = continuation(request, context);
            return await call.ResponseHeadersAsync;
        }
        catch
        {
            return new Grpc.Core.Metadata();
        }
    }

    private static bool IsRetryable(StatusCode code) =>
        code is StatusCode.Unavailable
             or StatusCode.DeadlineExceeded
             or StatusCode.ResourceExhausted;
}
```

---

## Step 1536: Client Implementation with Resilience

```csharp
// Clients/OrderGrpcClient.cs
using Grpc.Core;
using GrpcAdvanced.Protos;
using Google.Protobuf.WellKnownTypes;

namespace GrpcAdvanced.Clients;

public interface IOrderGrpcClient
{
    Task<CreateOrderResponse> CreateOrderAsync(CreateOrderRequest req, CancellationToken ct = default);
    IAsyncEnumerable<Order> ListOrdersAsync(ListOrdersRequest req, CancellationToken ct = default);
    IAsyncEnumerable<OrderEvent> WatchOrdersAsync(WatchOrdersRequest req, CancellationToken ct = default);
    Task<BatchUpdateResponse> BatchUpdateAsync(IEnumerable<UpdateOrderStatusRequest> updates, CancellationToken ct = default);
    IAsyncEnumerable<StreamProcessResponse> ProcessStreamAsync(IAsyncEnumerable<StreamProcessRequest> requests, CancellationToken ct = default);
}

public class OrderGrpcClient : IOrderGrpcClient
{
    private readonly OrderService.OrderServiceClient _client;
    private readonly ILogger<OrderGrpcClient> _log;

    public OrderGrpcClient(
        OrderService.OrderServiceClient client,
        ILogger<OrderGrpcClient> log)
    {
        _client = client;
        _log    = log;
    }

    // ── Unary ─────────────────────────────────────────────────────────────

    public async Task<CreateOrderResponse> CreateOrderAsync(
        CreateOrderRequest req,
        CancellationToken ct = default)
    {
        // Set deadline
        var options = new CallOptions(
            deadline: DateTime.UtcNow.AddSeconds(30),
            cancellationToken: ct,
            headers: new Grpc.Core.Metadata
            {
                { "x-request-id", Guid.NewGuid().ToString() }
            });

        try
        {
            return await _client.CreateOrderAsync(req, options);
        }
        catch (RpcException ex) when (ex.StatusCode == StatusCode.InvalidArgument)
        {
            _log.LogWarning("Validation failed: {Msg}", ex.Message);
            throw new ValidationException(ex.Message);
        }
        catch (RpcException ex) when (ex.StatusCode == StatusCode.DeadlineExceeded)
        {
            _log.LogError("CreateOrder timed out after 30s");
            throw new TimeoutException("Order creation timed out");
        }
    }

    // ── Server Streaming ──────────────────────────────────────────────────

    public async IAsyncEnumerable<Order> ListOrdersAsync(
        ListOrdersRequest req,
        [System.Runtime.CompilerServices.EnumeratorCancellation] CancellationToken ct = default)
    {
        using var call = _client.ListOrders(req,
            new CallOptions(deadline: DateTime.UtcNow.AddMinutes(5), cancellationToken: ct));

        await foreach (var order in call.ResponseStream.ReadAllAsync(ct))
        {
            yield return order;
        }
    }

    public async IAsyncEnumerable<OrderEvent> WatchOrdersAsync(
        WatchOrdersRequest req,
        [System.Runtime.CompilerServices.EnumeratorCancellation] CancellationToken ct = default)
    {
        while (!ct.IsCancellationRequested)
        {
            using var call = _client.WatchOrders(req,
                new CallOptions(cancellationToken: ct));

            bool connected = false;
            try
            {
                await foreach (var evt in call.ResponseStream.ReadAllAsync(ct))
                {
                    connected = true;
                    yield return evt;
                }
                break; // server closed gracefully
            }
            catch (RpcException ex) when (ex.StatusCode == StatusCode.Unavailable && !connected)
            {
                _log.LogWarning("Watch stream disconnected, reconnecting in 5s...");
                await Task.Delay(TimeSpan.FromSeconds(5), ct);
            }
        }
    }

    // ── Client Streaming ──────────────────────────────────────────────────

    public async Task<BatchUpdateResponse> BatchUpdateAsync(
        IEnumerable<UpdateOrderStatusRequest> updates,
        CancellationToken ct = default)
    {
        using var call = _client.BatchUpdate(
            new CallOptions(deadline: DateTime.UtcNow.AddMinutes(2), cancellationToken: ct));

        foreach (var update in updates)
        {
            await call.RequestStream.WriteAsync(update, ct);
        }

        await call.RequestStream.CompleteAsync();
        return await call.ResponseAsync;
    }

    // ── Bidirectional Streaming ───────────────────────────────────────────

    public async IAsyncEnumerable<StreamProcessResponse> ProcessStreamAsync(
        IAsyncEnumerable<StreamProcessRequest> requests,
        [System.Runtime.CompilerServices.EnumeratorCancellation] CancellationToken ct = default)
    {
        using var call = _client.ProcessStream(
            new CallOptions(cancellationToken: ct));

        // Producer task: write all requests
        var producerTask = Task.Run(async () =>
        {
            await foreach (var req in requests.WithCancellation(ct))
                await call.RequestStream.WriteAsync(req, ct);
            await call.RequestStream.CompleteAsync();
        }, ct);

        // Consumer: read responses as they arrive
        await foreach (var response in call.ResponseStream.ReadAllAsync(ct))
            yield return response;

        await producerTask;
    }
}
```

---

## Step 1537: Program.cs — Server & Client Registration

```csharp
// Server Program.cs
using GrpcAdvanced.Interceptors;
using GrpcAdvanced.Services;
using Microsoft.AspNetCore.Authentication.JwtBearer;

var builder = WebApplication.CreateBuilder(args);

// ── gRPC Server ───────────────────────────────────────────────────────────
builder.Services.AddGrpc(options =>
{
    options.EnableDetailedErrors          = builder.Environment.IsDevelopment();
    options.MaxReceiveMessageSize         = 4 * 1024 * 1024;  // 4 MB
    options.MaxSendMessageSize            = 4 * 1024 * 1024;
    options.ResponseCompressionLevel      = System.IO.Compression.CompressionLevel.Optimal;
    options.ResponseCompressionAlgorithm  = "gzip";

    // Global interceptors (applied to all services)
    options.Interceptors.Add<LoggingInterceptor>();
    options.Interceptors.Add<AuthInterceptor>();
});

builder.Services.AddGrpcReflection();  // for grpcui / grpcurl in development

// ── gRPC Health Checks ─────────────────────────────────────────────────────
builder.Services.AddGrpcHealthChecks()
    .AddCheck<DatabaseHealthCheck>("database")
    .AddCheck<CacheHealthCheck>("cache");

// ── Auth ───────────────────────────────────────────────────────────────────
builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(opts =>
    {
        opts.Authority = builder.Configuration["Auth:Authority"];
        opts.Audience  = builder.Configuration["Auth:Audience"];
    });
builder.Services.AddAuthorization();
builder.Services.AddHttpContextAccessor();

// ── DI ─────────────────────────────────────────────────────────────────────
builder.Services.AddScoped<IOrderRepository, OrderRepository>();
builder.Services.AddSingleton<IEventBus, InMemoryEventBus>();
builder.Services.AddScoped<LoggingInterceptor>();
builder.Services.AddScoped<AuthInterceptor>();

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();

app.MapGrpcService<OrderGrpcService>();
app.MapGrpcService<InventoryGrpcService>();
app.MapGrpcHealthChecksService();

if (app.Environment.IsDevelopment())
    app.MapGrpcReflectionService();

// ── REST gateway for browsers ───────────────────────────────────────────────
app.MapGrpcReflectionService();
app.MapGet("/",
    () => "gRPC server running. Use a gRPC client.");

app.Run();
```

```csharp
// Client registration (in consuming service)
builder.Services
    .AddGrpcClient<OrderService.OrderServiceClient>(o =>
    {
        o.Address = new Uri(builder.Configuration["Grpc:OrderServiceUrl"]!);
    })
    .ConfigurePrimaryHttpMessageHandler(() =>
    {
        var handler = new HttpClientHandler();
        // In dev: trust self-signed cert
        if (builder.Environment.IsDevelopment())
            handler.ServerCertificateCustomValidationCallback =
                HttpClientHandler.DangerousAcceptAnyServerCertificateValidator;
        return handler;
    })
    .AddInterceptor<RetryInterceptor>()          // client-side retry
    .AddInterceptor<ClientLoggingInterceptor>()  // client-side logging
    .EnableCallContextPropagation()              // propagate deadlines & cancellation
    .ConfigureChannel(o =>
    {
        o.MaxRetryAttempts               = 5;
        o.MaxSendMessageSize             = 4 * 1024 * 1024;
        o.ServiceConfig = new Grpc.Net.Client.Configuration.ServiceConfig
        {
            MethodConfigs =
            {
                new Grpc.Net.Client.Configuration.MethodConfig
                {
                    Names = { MethodName.Default },
                    RetryPolicy = new Grpc.Net.Client.Configuration.RetryPolicy
                    {
                        MaxAttempts          = 5,
                        InitialBackoff       = TimeSpan.FromSeconds(0.5),
                        MaxBackoff           = TimeSpan.FromSeconds(5),
                        BackoffMultiplier    = 1.5,
                        RetryableStatusCodes = { StatusCode.Unavailable }
                    }
                }
            }
        };
    });

builder.Services.AddScoped<RetryInterceptor>();
builder.Services.AddScoped<IOrderGrpcClient, OrderGrpcClient>();
```

---

## Step 1538: gRPC Health Checks & Reflection

```csharp
// Health check for gRPC services
using Grpc.Health.V1;
using Grpc.HealthCheck;
using Microsoft.Extensions.Diagnostics.HealthChecks;

// server registers health service
builder.Services.AddGrpcHealthChecks()
    .AddAsyncCheck("self", () => ValueTask.FromResult(HealthCheckResult.Healthy()));

// Kubernetes liveness/readiness probes via gRPC
// grpc-health-probe -addr=:5001 -service=""
// grpc-health-probe -addr=:5001 -service="orders.OrderService"
```

```csharp
// DatabaseHealthCheck.cs
public class DatabaseHealthCheck : IHealthCheck
{
    private readonly IDbConnectionFactory _db;

    public DatabaseHealthCheck(IDbConnectionFactory db) => _db = db;

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken ct = default)
    {
        try
        {
            await using var conn = _db.Create();
            await conn.OpenAsync(ct);
            await conn.ExecuteScalarAsync<int>("SELECT 1", ct);
            return HealthCheckResult.Healthy("Database reachable");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("Database unreachable", ex);
        }
    }
}
```

---

## Step 1539: gRPC-Web & Transcoding (REST ↔ gRPC)

```csharp
// Enable gRPC-Web for browser clients
builder.Services.AddGrpcWeb(o => o.GrpcWebEnabled = true);

app.UseGrpcWeb(new GrpcWebOptions { DefaultEnabled = true });

app.MapGrpcService<OrderGrpcService>().EnableGrpcWeb();
```

```protobuf
// HTTP transcoding annotation in proto (grpc-gateway style)
import "google/api/annotations.proto";

service OrderService {
  rpc CreateOrder (CreateOrderRequest) returns (CreateOrderResponse) {
    option (google.api.http) = {
      post: "/v1/orders"
      body: "*"
    };
  }

  rpc GetOrder (GetOrderRequest) returns (Order) {
    option (google.api.http) = {
      get: "/v1/orders/{id}"
    };
  }

  rpc ListOrders (ListOrdersRequest) returns (ListOrdersResponse) {
    option (google.api.http) = {
      get: "/v1/orders"
    };
  }
}
```

```csharp
// ASP.NET Core gRPC HTTP/1 JSON transcoding
builder.Services.AddGrpc().AddJsonTranscoding();

// grpc-gateway.yaml for complex route rules
// curl -X POST http://localhost:5000/v1/orders \
//   -H "Content-Type: application/json" \
//   -d '{"customer_id":"cust-1","items":[{"product_id":"p1","quantity":2}]}'
```

---

## Step 1540: Load Balancing & Channel Management

```csharp
// Round-robin load balancing across multiple servers
using Grpc.Net.Client.Balancer;
using Grpc.Net.Client.Configuration;

builder.Services
    .AddGrpcClient<OrderService.OrderServiceClient>(o =>
    {
        // Use DNS or static list
        o.Address = new Uri("static:///order-service");
    })
    .ConfigureChannel(o =>
    {
        o.Credentials = ChannelCredentials.Insecure;
        o.ServiceConfig = new ServiceConfig
        {
            LoadBalancingConfigs = { new RoundRobinConfig() }
        };
    });

// Static resolver for service discovery
builder.Services.AddSingleton<ResolverFactory>(
    new StaticResolverFactory(addr => new[]
    {
        new BalancerAddress("order-service-1", 5001),
        new BalancerAddress("order-service-2", 5001),
        new BalancerAddress("order-service-3", 5001),
    }));
```

```csharp
// Channel pooling for high-throughput scenarios
public class GrpcChannelPool : IDisposable
{
    private readonly GrpcChannel[] _channels;
    private int _index;

    public GrpcChannelPool(string address, int poolSize = 4)
    {
        _channels = Enumerable.Range(0, poolSize)
            .Select(_ => GrpcChannel.ForAddress(address, new GrpcChannelOptions
            {
                HttpHandler = new SocketsHttpHandler
                {
                    PooledConnectionIdleTimeout    = TimeSpan.FromMinutes(5),
                    KeepAlivePingDelay             = TimeSpan.FromSeconds(60),
                    KeepAlivePingTimeout           = TimeSpan.FromSeconds(30),
                    EnableMultipleHttp2Connections = true
                }
            }))
            .ToArray();
    }

    public GrpcChannel GetChannel()
    {
        var idx = Interlocked.Increment(ref _index) % _channels.Length;
        return _channels[idx];
    }

    public void Dispose()
    {
        foreach (var ch in _channels) ch.Dispose();
    }
}
```

---

## Step 1541: Inventory Service — Client Streaming

```csharp
// Services/InventoryGrpcService.cs
public class InventoryGrpcService : InventoryService.InventoryServiceBase
{
    private readonly IInventoryRepository _repo;

    public InventoryGrpcService(IInventoryRepository repo) => _repo = repo;

    public override async Task<StockCheckResult> CheckStock(
        CheckStockRequest request,
        ServerCallContext context)
    {
        var stock = await _repo.GetStockAsync(request.ProductId, context.CancellationToken);

        return new StockCheckResult
        {
            ProductId    = request.ProductId,
            Available    = stock.Current >= request.Quantity,
            CurrentStock = stock.Current,
            Warehouse    = stock.Warehouse
        };
    }

    // Bidirectional: check many products concurrently
    public override async Task CheckBulk(
        IAsyncStreamReader<CheckStockRequest> requestStream,
        IServerStreamWriter<StockCheckResult> responseStream,
        ServerCallContext context)
    {
        var ct = context.CancellationToken;

        // Fan-out: check all in parallel, stream results as they complete
        var semaphore = new SemaphoreSlim(20);
        var tasks = new List<Task>();

        await foreach (var req in requestStream.ReadAllAsync(ct))
        {
            await semaphore.WaitAsync(ct);

            var task = Task.Run(async () =>
            {
                try
                {
                    var stock = await _repo.GetStockAsync(req.ProductId, ct);
                    var result = new StockCheckResult
                    {
                        ProductId    = req.ProductId,
                        Available    = stock.Current >= req.Quantity,
                        CurrentStock = stock.Current,
                        Warehouse    = stock.Warehouse
                    };
                    await responseStream.WriteAsync(result, ct);
                }
                finally
                {
                    semaphore.Release();
                }
            }, ct);

            tasks.Add(task);
        }

        await Task.WhenAll(tasks);
    }
}
```

---

## Step 1542: OpenTelemetry Integration with gRPC

```csharp
// gRPC tracing & metrics via OpenTelemetry
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing =>
    {
        tracing
            .AddAspNetCoreInstrumentation()
            .AddGrpcCoreInstrumentation()  // server
            .AddGrpcClientInstrumentation() // client
            .AddOtlpExporter();
    })
    .WithMetrics(metrics =>
    {
        metrics
            .AddAspNetCoreInstrumentation()
            .AddMeter("Grpc.AspNetCore.Server")
            .AddMeter("Grpc.Net.Client")
            .AddPrometheusExporter();
    });
```

```csharp
// Custom gRPC metrics interceptor
public class MetricsInterceptor : Interceptor
{
    private static readonly Meter Meter = new("GrpcAdvanced.Server", "1.0.0");
    private static readonly Counter<long>    TotalCalls    = Meter.CreateCounter<long>("grpc.server.calls.total");
    private static readonly Histogram<double> CallDuration  = Meter.CreateHistogram<double>("grpc.server.call.duration", "ms");

    public override async Task<TResponse> UnaryServerHandler<TRequest, TResponse>(
        TRequest request,
        ServerCallContext context,
        UnaryServerMethod<TRequest, TResponse> continuation)
    {
        var tags = new TagList
        {
            { "grpc.method", context.Method },
            { "grpc.service", context.Host }
        };

        var sw = System.Diagnostics.Stopwatch.StartNew();
        TotalCalls.Add(1, tags);

        try
        {
            var resp = await continuation(request, context);
            tags.Add("grpc.status", "OK");
            return resp;
        }
        catch (RpcException ex)
        {
            tags.Add("grpc.status", ex.StatusCode.ToString());
            throw;
        }
        finally
        {
            CallDuration.Record(sw.Elapsed.TotalMilliseconds, tags);
        }
    }
}
```

---

## Step 1543: Testing gRPC Services

```csharp
// Tests/OrderGrpcServiceTests.cs
using Grpc.Core;
using Grpc.Core.Testing;
using GrpcAdvanced.Protos;
using GrpcAdvanced.Services;
using Microsoft.Extensions.Logging.Abstractions;
using Moq;

namespace GrpcAdvanced.Tests;

public class OrderGrpcServiceTests
{
    private readonly Mock<IOrderRepository> _repo = new();
    private readonly Mock<IEventBus>        _bus  = new();
    private readonly OrderGrpcService       _sut;

    public OrderGrpcServiceTests()
    {
        _sut = new OrderGrpcService(
            _repo.Object,
            NullLogger<OrderGrpcService>.Instance,
            _bus.Object);
    }

    [Fact]
    public async Task CreateOrder_ValidRequest_ReturnsSuccess()
    {
        var request = new CreateOrderRequest
        {
            CustomerId = "cust-1",
            Items =
            {
                new OrderItem { ProductId = "prod-1", Quantity = 2, Name = "Widget" }
            }
        };

        var order = new OrderEntity { Id = Guid.NewGuid(), CustomerId = "cust-1", Total = 50m };
        _repo.Setup(r => r.CreateAsync(It.IsAny<OrderEntity>(), It.IsAny<CancellationToken>()))
             .ReturnsAsync(order);

        var context = TestServerCallContext.Create(
            method: "CreateOrder",
            host: "localhost",
            deadline: DateTime.UtcNow.AddMinutes(1),
            requestHeaders: new Grpc.Core.Metadata(),
            cancellationToken: CancellationToken.None,
            peer: "127.0.0.1",
            authContext: null,
            contextPropagationToken: null,
            writeHeadersFunc: _ => Task.CompletedTask,
            writeOptionsGetter: () => null,
            writeOptionsSetter: _ => { });

        var result = await _sut.CreateOrder(request, context);

        Assert.True(result.Success);
        Assert.Empty(result.Errors);
        Assert.Equal("cust-1", result.Order.CustomerId);
    }

    [Fact]
    public async Task CreateOrder_InvalidRequest_ReturnsValidationErrors()
    {
        var request = new CreateOrderRequest { CustomerId = "" }; // missing items too

        var context = TestServerCallContext.Create(
            method: "CreateOrder",
            host: "localhost",
            deadline: DateTime.UtcNow.AddMinutes(1),
            requestHeaders: new Grpc.Core.Metadata(),
            cancellationToken: CancellationToken.None,
            peer: "127.0.0.1",
            authContext: null,
            contextPropagationToken: null,
            writeHeadersFunc: _ => Task.CompletedTask,
            writeOptionsGetter: () => null,
            writeOptionsSetter: _ => { });

        var result = await _sut.CreateOrder(request, context);

        Assert.False(result.Success);
        Assert.Equal(2, result.Errors.Count);
    }

    [Fact]
    public async Task GetOrder_NotFound_ThrowsRpcException()
    {
        _repo.Setup(r => r.GetByIdAsync(It.IsAny<string>(), It.IsAny<CancellationToken>()))
             .ReturnsAsync((OrderEntity?)null);

        var context = TestServerCallContext.Create(
            method: "GetOrder",
            host: "localhost",
            deadline: DateTime.UtcNow.AddMinutes(1),
            requestHeaders: new Grpc.Core.Metadata(),
            cancellationToken: CancellationToken.None,
            peer: "127.0.0.1",
            authContext: null,
            contextPropagationToken: null,
            writeHeadersFunc: _ => Task.CompletedTask,
            writeOptionsGetter: () => null,
            writeOptionsSetter: _ => { });

        var ex = await Assert.ThrowsAsync<RpcException>(
            () => _sut.GetOrder(new GetOrderRequest { Id = Guid.NewGuid().ToString() }, context));

        Assert.Equal(StatusCode.NotFound, ex.StatusCode);
    }

    [Fact]
    public async Task ListOrders_StreamsResults()
    {
        var orders = Enumerable.Range(1, 50)
            .Select(i => new OrderEntity
            {
                Id = Guid.NewGuid(),
                CustomerId = "cust-1",
                Total = i * 10m
            })
            .ToList();

        _repo.Setup(r => r.ListAsync(It.IsAny<OrderFilter>(), It.IsAny<CancellationToken>()))
             .ReturnsAsync(new Page<OrderEntity>(orders, null));

        var context = TestServerCallContext.Create(
            method: "ListOrders",
            host: "localhost",
            deadline: DateTime.UtcNow.AddMinutes(1),
            requestHeaders: new Grpc.Core.Metadata(),
            cancellationToken: CancellationToken.None,
            peer: "127.0.0.1",
            authContext: null,
            contextPropagationToken: null,
            writeHeadersFunc: _ => Task.CompletedTask,
            writeOptionsGetter: () => null,
            writeOptionsSetter: _ => { });

        var written = new List<Order>();
        var mockWriter = new MockServerStreamWriter<Order>(written);

        await _sut.ListOrders(
            new ListOrdersRequest { CustomerId = "cust-1" },
            mockWriter,
            context);

        Assert.Equal(50, written.Count);
    }
}

public class MockServerStreamWriter<T> : IServerStreamWriter<T>
{
    private readonly List<T> _items;

    public MockServerStreamWriter(List<T> items) => _items = items;

    public WriteOptions? WriteOptions { get; set; }

    public Task WriteAsync(T message)
    {
        _items.Add(message);
        return Task.CompletedTask;
    }

    public Task WriteAsync(T message, CancellationToken cancellationToken)
    {
        _items.Add(message);
        return Task.CompletedTask;
    }
}
```

---

## Step 1544: Integration Testing with TestServer

```csharp
// Tests/GrpcIntegrationTests.cs
using Grpc.Net.Client;
using GrpcAdvanced.Protos;
using Microsoft.AspNetCore.Mvc.Testing;

namespace GrpcAdvanced.Tests;

public class GrpcIntegrationTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;

    public GrpcIntegrationTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                // Replace DB with in-memory
                services.AddSingleton<IOrderRepository, InMemoryOrderRepository>();
            });
        });
    }

    private OrderService.OrderServiceClient CreateClient()
    {
        var httpClient = _factory.CreateClient();

        // gRPC over HTTP/1.1 requires special handler for test server
        var channel = GrpcChannel.ForAddress(httpClient.BaseAddress!, new GrpcChannelOptions
        {
            HttpClient = httpClient
        });

        return new OrderService.OrderServiceClient(channel);
    }

    [Fact]
    public async Task CreateAndGetOrder_EndToEnd()
    {
        var client = CreateClient();

        var created = await client.CreateOrderAsync(new CreateOrderRequest
        {
            CustomerId = "integration-test",
            Items =
            {
                new OrderItem { ProductId = "p1", Quantity = 3, Name = "Widget" }
            }
        });

        Assert.True(created.Success);
        Assert.NotEmpty(created.Order.Id);

        var fetched = await client.GetOrderAsync(new GetOrderRequest { Id = created.Order.Id });

        Assert.Equal(created.Order.Id, fetched.Id);
        Assert.Equal("integration-test", fetched.CustomerId);
    }

    [Fact]
    public async Task ListOrders_ServerStreaming()
    {
        var client = CreateClient();

        // Seed data
        for (int i = 0; i < 10; i++)
        {
            await client.CreateOrderAsync(new CreateOrderRequest
            {
                CustomerId = "stream-test",
                Items = { new OrderItem { ProductId = "p1", Quantity = 1, Name = "Item" } }
            });
        }

        var received = new List<Order>();
        using var call = client.ListOrders(new ListOrdersRequest { CustomerId = "stream-test" });

        await foreach (var order in call.ResponseStream.ReadAllAsync())
            received.Add(order);

        Assert.Equal(10, received.Count);
    }

    [Fact]
    public async Task BatchUpdate_ClientStreaming()
    {
        var client = CreateClient();

        // Create orders first
        var ids = new List<string>();
        for (int i = 0; i < 5; i++)
        {
            var r = await client.CreateOrderAsync(new CreateOrderRequest
            {
                CustomerId = "batch-test",
                Items = { new OrderItem { ProductId = "p1", Quantity = 1, Name = "Item" } }
            });
            ids.Add(r.Order.Id);
        }

        // Stream updates
        using var call = client.BatchUpdate();

        foreach (var id in ids)
        {
            await call.RequestStream.WriteAsync(new UpdateOrderStatusRequest
            {
                OrderId = id,
                Status  = Protos.OrderStatus.Confirmed,
                Reason  = "integration-test"
            });
        }

        await call.RequestStream.CompleteAsync();
        var result = await call.ResponseAsync;

        Assert.Equal(5, result.Succeeded);
        Assert.Equal(0, result.Failed);
    }
}
```

---

## Step 1545: Docker & Deployment Configuration

```yaml
# docker-compose.yml
version: '3.9'

services:
  grpc-server:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "5001:5001"
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - ASPNETCORE_URLS=http://+:5001
      - Grpc__MaxReceiveMessageSize=4194304
      - ConnectionStrings__Default=Host=postgres;Database=orders;Username=app;Password=secret
    depends_on:
      postgres:
        condition: service_healthy
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '0.5'
          memory: 512M

  envoy:
    image: envoyproxy/envoy:v1.29-latest
    ports:
      - "8080:8080"   # HTTP/1.1 REST (transcoded from gRPC)
      - "9090:9090"   # gRPC (HTTP/2)
    volumes:
      - ./envoy.yaml:/etc/envoy/envoy.yaml:ro
    depends_on:
      - grpc-server

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: orders
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d orders"]
      interval: 5s
      timeout: 5s
      retries: 5
```

```yaml
# envoy.yaml — gRPC load balancing & transcoding
static_resources:
  listeners:
    - name: grpc_listener
      address:
        socket_address: { address: 0.0.0.0, port_value: 9090 }
      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                codec_type: HTTP2
                stat_prefix: grpc
                route_config:
                  virtual_hosts:
                    - name: grpc_services
                      domains: ["*"]
                      routes:
                        - match: { prefix: "/orders.OrderService" }
                          route:
                            cluster: order_grpc
                            timeout: 30s
                            retry_policy:
                              retry_on: "connect-failure,reset,5xx"
                              num_retries: 3

  clusters:
    - name: order_grpc
      type: STRICT_DNS
      lb_policy: ROUND_ROBIN
      http2_protocol_options: {}
      load_assignment:
        cluster_name: order_grpc
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address: { address: grpc-server, port_value: 5001 }
```

```dockerfile
# Dockerfile
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /src

COPY *.csproj .
RUN dotnet restore

COPY . .
RUN dotnet publish -c Release -o /app --no-restore

FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS runtime
WORKDIR /app

# gRPC needs HTTP/2 — ensure TLS or h2c is configured
ENV ASPNETCORE_HTTP_PORTS=5001

COPY --from=build /app .
ENTRYPOINT ["dotnet", "GrpcAdvanced.dll"]
```

---

## Step 1546: Advanced Proto Patterns

```protobuf
// Protos/analytics.proto — advanced patterns
syntax = "proto3";
option csharp_namespace = "GrpcAdvanced.Protos";

import "google/protobuf/duration.proto";
import "google/protobuf/field_mask.proto";
import "google/protobuf/struct.proto";

package analytics;

// Field masks for partial updates
message UpdateAnalyticsRequest {
  AnalyticsRecord record     = 1;
  google.protobuf.FieldMask update_mask = 2;
}

// Struct for dynamic JSON-like data
message EventPayload {
  string                    event_type = 1;
  google.protobuf.Struct    properties = 2;
  google.protobuf.Duration  duration   = 3;
}

// oneOf for sum types
message MetricValue {
  oneof value {
    int64  integer_value  = 1;
    double double_value   = 2;
    string string_value   = 3;
    bool   boolean_value  = 4;
  }
}

// Any for polymorphic payloads
import "google/protobuf/any.proto";

message Command {
  string               id      = 1;
  string               type    = 2;
  google.protobuf.Any  payload = 3;
}

service AnalyticsService {
  // Long-running operation pattern
  rpc ExportData  (ExportRequest)  returns (stream ExportChunk);
  rpc IngestEvents(stream EventPayload) returns (IngestSummary);
  rpc UpdatePartial(UpdateAnalyticsRequest) returns (AnalyticsRecord);
}

message ExportRequest {
  string start_date = 1;
  string end_date   = 2;
  string format     = 3;  // "csv", "json", "parquet"
}

message ExportChunk {
  bytes  data       = 1;
  int32  chunk_num  = 2;
  bool   is_last    = 3;
}

message IngestSummary {
  int64 accepted = 1;
  int64 rejected = 2;
  repeated string errors = 3;
}

message AnalyticsRecord {
  string id    = 1;
  string name  = 2;
  double value = 3;
}
```

---

## Step 1547: Error Handling with Rich Status Details

```csharp
// Rich error details using google.rpc.Status extensions
using Google.Rpc;
using Grpc.Core;
using Grpc.StatusProto;

public static class GrpcErrors
{
    public static RpcException ValidationFailed(IEnumerable<(string field, string msg)> violations)
    {
        var badRequest = new BadRequest();
        foreach (var (field, msg) in violations)
            badRequest.FieldViolations.Add(new BadRequest.Types.FieldViolation
            {
                Field       = field,
                Description = msg
            });

        var status = new Google.Rpc.Status
        {
            Code    = (int)Code.InvalidArgument,
            Message = "Request validation failed"
        };
        status.Details.Add(Google.Protobuf.WellKnownTypes.Any.Pack(badRequest));

        return status.ToRpcException();
    }

    public static RpcException ResourceExhausted(string quotaMetric, long limit)
    {
        var info = new QuotaFailure();
        info.Violations.Add(new QuotaFailure.Types.Violation
        {
            Subject     = quotaMetric,
            Description = $"Quota exceeded. Limit: {limit}"
        });

        var status = new Google.Rpc.Status
        {
            Code    = (int)Code.ResourceExhausted,
            Message = "Rate limit exceeded"
        };
        status.Details.Add(Google.Protobuf.WellKnownTypes.Any.Pack(info));

        return status.ToRpcException();
    }

    public static RpcException NotFound(string resourceType, string id)
    {
        var info = new ResourceInfo
        {
            ResourceType = resourceType,
            ResourceName = id,
            Description  = $"The {resourceType} with id '{id}' was not found"
        };

        var status = new Google.Rpc.Status
        {
            Code    = (int)Code.NotFound,
            Message = $"{resourceType} not found"
        };
        status.Details.Add(Google.Protobuf.WellKnownTypes.Any.Pack(info));

        return status.ToRpcException();
    }
}

// Client-side: extract rich error details
try
{
    await client.CreateOrderAsync(request);
}
catch (RpcException ex)
{
    var rpcStatus = ex.GetRpcStatus();
    if (rpcStatus is not null)
    {
        foreach (var detail in rpcStatus.Details)
        {
            if (detail.TryUnpack<BadRequest>(out var badReq))
            {
                foreach (var v in badReq.FieldViolations)
                    Console.WriteLine($"Field '{v.Field}': {v.Description}");
            }
            else if (detail.TryUnpack<ResourceInfo>(out var info))
            {
                Console.WriteLine($"Resource '{info.ResourceName}' of type '{info.ResourceType}' not found");
            }
        }
    }
}
```

---

## Step 1548: Deadline Propagation & Context

```csharp
// Deadline propagation across service calls
public class OrderService : OrderService.OrderServiceBase
{
    private readonly InventoryService.InventoryServiceClient _inventory;

    public override async Task<CreateOrderResponse> CreateOrder(
        CreateOrderRequest request,
        ServerCallContext context)
    {
        // Propagate deadline to downstream calls
        // context.Deadline is the client's deadline — use it as-is
        var options = new CallOptions(
            deadline:           context.Deadline,
            cancellationToken:  context.CancellationToken,
            headers:            PropagateHeaders(context));

        // This will fail fast if we're near the deadline
        var stock = await _inventory.CheckStockAsync(
            new CheckStockRequest { ProductId = "p1", Quantity = 1 },
            options);

        if (!stock.Available)
            throw new RpcException(new Grpc.Core.Status(
                StatusCode.FailedPrecondition, "Insufficient stock"));

        // Continue order creation...
        return new CreateOrderResponse { Success = true };
    }

    private static Grpc.Core.Metadata PropagateHeaders(ServerCallContext context)
    {
        var headers = new Grpc.Core.Metadata();

        // Propagate tracing headers
        foreach (var header in context.RequestHeaders
            .Where(h => h.Key.StartsWith("x-b3-") || h.Key == "traceparent"))
        {
            headers.Add(header.Key, header.Value);
        }

        return headers;
    }
}
```

---

## Step 1549: Proto Code Generation & Build Integration

```xml
<!-- GrpcAdvanced.csproj -->
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

  <ItemGroup>
    <!-- Server-side proto compilation -->
    <Protobuf Include="Protos/order.proto"
              GrpcServices="Server"
              Access="Public"
              ProtoRoot="Protos" />

    <Protobuf Include="Protos/inventory.proto"
              GrpcServices="Both"
              Access="Public" />

    <Protobuf Include="Protos/analytics.proto"
              GrpcServices="Server"
              Access="Public" />
  </ItemGroup>

  <ItemGroup>
    <PackageReference Include="Grpc.AspNetCore"                   Version="2.67.*" />
    <PackageReference Include="Grpc.AspNetCore.Web"               Version="2.67.*" />
    <PackageReference Include="Grpc.AspNetCore.Server.Reflection" Version="2.67.*" />
    <PackageReference Include="Grpc.HealthCheck"                  Version="2.67.*" />
    <PackageReference Include="Grpc.Net.ClientFactory"            Version="2.67.*" />
    <PackageReference Include="Google.Protobuf"                   Version="3.29.*" />
    <PackageReference Include="Google.Api.CommonProtos"           Version="2.15.*" />
    <PackageReference Include="Grpc.StatusProto"                  Version="1.67.*" />
    <PackageReference Include="OpenTelemetry.Instrumentation.GrpcCore" Version="1.9.*" />
  </ItemGroup>
</Project>
```

---

## Step 1550: Summary & Architecture Checklist

```
gRPC Advanced Production Checklist
═══════════════════════════════════════════════════════════

Proto Design
✅ Protobuf 3 with proper field numbering (never reuse!)
✅ Google well-known types: Timestamp, Duration, Empty
✅ oneOf for discriminated unions
✅ map<K,V> for dictionary fields
✅ FieldMask for partial updates
✅ Rich error details (BadRequest, ResourceInfo, QuotaFailure)
✅ HTTP transcoding annotations for REST bridges

Service Patterns
✅ Unary: standard request-response with deadline
✅ Server Streaming: pagination, event feeds
✅ Client Streaming: batch operations
✅ Bidirectional: real-time processing pipelines
✅ Deadline propagation to downstream services
✅ Cancellation token passed everywhere

Server
✅ EnableDetailedErrors=true in development only
✅ Compression (gzip/deflate) enabled
✅ Message size limits (default 4MB)
✅ Authentication via JWT header
✅ Authorization policies per method
✅ Health checks (gRPC native protocol)
✅ gRPC reflection for tooling (dev only)
✅ gRPC-Web for browser clients

Client
✅ Typed client via IHttpClientFactory
✅ Retry policy (service config)
✅ Circuit breaker via Polly
✅ Connection pooling (SocketsHttpHandler)
✅ Channel reuse (not per-request)
✅ Deadline set on every call
✅ Client-side interceptors (logging, retry, auth)
✅ Reconnection for streaming calls

Observability
✅ OpenTelemetry tracing (AddGrpcClientInstrumentation)
✅ Server metrics (Grpc.AspNetCore.Server meter)
✅ Client metrics (Grpc.Net.Client meter)
✅ Structured logging per method/status

Testing
✅ Unit tests with TestServerCallContext
✅ Integration tests with WebApplicationFactory
✅ In-memory repository for fast tests
✅ Load testing with NBomber gRPC scenarios

Deployment
✅ Docker + envoy for load balancing
✅ Kubernetes readiness/liveness via health checks
✅ Horizontal scaling with connection pooling
✅ TLS termination at load balancer
✅ h2c (HTTP/2 cleartext) inside cluster
```

---

**ยินดีด้วย!** Part 56 ครอบคลุม gRPC ขั้นสูงครบทุกรูปแบบ:
- **4 Communication Patterns**: Unary, Server Streaming, Client Streaming, Bidirectional
- **Interceptors**: Server-side auth/logging, client-side retry
- **Rich Errors**: google.rpc.Status details (BadRequest, ResourceInfo)
- **Load Balancing**: Round-robin, channel pooling, Envoy proxy
- **Testing**: Unit + Integration patterns
- **OpenTelemetry**: Built-in gRPC instrumentation
