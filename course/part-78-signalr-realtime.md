# Part 78: Real-Time Features — SignalR, SSE & WebSockets

## Steps 1877–1892

---

## Step 1877: SignalR Core Concepts

Real-time bidirectional communication — server pushes to clients without polling.

**Transport negotiation order**: WebSocket → Server-Sent Events → Long Polling  
**Hub**: central abstraction — methods clients call and server invokes on clients

```bash
dotnet new webapi -n RealtimeApp
cd RealtimeApp
dotnet add package Microsoft.AspNetCore.SignalR.Client
```

---

## Step 1878: Basic Hub Implementation

```csharp
// Hubs/OrderHub.cs
using Microsoft.AspNetCore.SignalR;

namespace RealtimeApp.Hubs;

public class OrderHub : Hub
{
    // Client calls this method
    public async Task SubscribeToOrder(string orderId)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, $"order-{orderId}");
    }

    public async Task UnsubscribeFromOrder(string orderId)
    {
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, $"order-{orderId}");
    }

    // Server-to-client — called via IHubContext
    // Client method: connection.on("OrderStatusChanged", handler)

    public override async Task OnConnectedAsync()
    {
        var userId = Context.User?.FindFirst("sub")?.Value;
        if (userId is not null)
            await Groups.AddToGroupAsync(Context.ConnectionId, $"user-{userId}");

        await base.OnConnectedAsync();
    }

    public override Task OnDisconnectedAsync(Exception? exception)
    {
        // Cleanup if needed
        return base.OnDisconnectedAsync(exception);
    }
}
```

```csharp
// Program.cs
builder.Services.AddSignalR(opts =>
{
    opts.EnableDetailedErrors = builder.Environment.IsDevelopment();
    opts.KeepAliveInterval = TimeSpan.FromSeconds(15);
    opts.ClientTimeoutInterval = TimeSpan.FromSeconds(30);
    opts.HandshakeTimeout = TimeSpan.FromSeconds(15);
    opts.MaximumReceiveMessageSize = 64 * 1024; // 64 KB
});

app.MapHub<OrderHub>("/hubs/orders");
```

---

## Step 1879: Strongly-Typed Hubs

Eliminate magic strings — compile-time safety for client method calls.

```csharp
// Hubs/Contracts/IOrderHubClient.cs
namespace RealtimeApp.Hubs.Contracts;

public interface IOrderHubClient
{
    Task OrderStatusChanged(OrderStatusChangedMessage message);
    Task OrderItemAdded(OrderItemAddedMessage message);
    Task PaymentProcessed(PaymentProcessedMessage message);
    Task ErrorOccurred(string errorCode, string message);
}

public record OrderStatusChangedMessage(
    Guid OrderId,
    string OldStatus,
    string NewStatus,
    DateTimeOffset ChangedAt,
    string? Note = null);

public record OrderItemAddedMessage(
    Guid OrderId,
    Guid ItemId,
    string ProductName,
    int Quantity,
    decimal UnitPrice);

public record PaymentProcessedMessage(
    Guid OrderId,
    Guid PaymentId,
    decimal Amount,
    string Currency,
    bool Success,
    string? FailureReason = null);
```

```csharp
// Hubs/OrderHub.cs
using Microsoft.AspNetCore.SignalR;
using RealtimeApp.Hubs.Contracts;

namespace RealtimeApp.Hubs;

public class OrderHub : Hub<IOrderHubClient>
{
    private readonly ILogger<OrderHub> _logger;

    public OrderHub(ILogger<OrderHub> logger)
    {
        _logger = logger;
    }

    public async Task SubscribeToOrder(Guid orderId)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, GroupName(orderId));
        _logger.LogInformation("Connection {ConnectionId} subscribed to order {OrderId}",
            Context.ConnectionId, orderId);
    }

    public async Task UnsubscribeFromOrder(Guid orderId)
    {
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, GroupName(orderId));
    }

    // Strongly-typed: Clients.Group().OrderStatusChanged(...)
    // Not Clients.Group().SendAsync("OrderStatusChanged", ...)

    public static string GroupName(Guid orderId) => $"order:{orderId}";
    public static string UserGroup(string userId) => $"user:{userId}";
}
```

---

## Step 1880: IHubContext — Server-Side Push

Inject and use from application services, not just the hub itself.

```csharp
// Services/OrderNotificationService.cs
using Microsoft.AspNetCore.SignalR;
using RealtimeApp.Hubs;
using RealtimeApp.Hubs.Contracts;

namespace RealtimeApp.Services;

public interface IOrderNotificationService
{
    Task NotifyStatusChangedAsync(Guid orderId, string oldStatus, string newStatus,
        string? note = null, CancellationToken ct = default);
    Task NotifyPaymentProcessedAsync(Guid orderId, Guid paymentId,
        decimal amount, string currency, bool success, string? failureReason = null,
        CancellationToken ct = default);
    Task NotifyUserAsync(string userId, Func<IOrderHubClient, Task> notification,
        CancellationToken ct = default);
}

public sealed class OrderNotificationService : IOrderNotificationService
{
    private readonly IHubContext<OrderHub, IOrderHubClient> _hubContext;

    public OrderNotificationService(IHubContext<OrderHub, IOrderHubClient> hubContext)
    {
        _hubContext = hubContext;
    }

    public async Task NotifyStatusChangedAsync(Guid orderId, string oldStatus,
        string newStatus, string? note = null, CancellationToken ct = default)
    {
        var message = new OrderStatusChangedMessage(
            orderId, oldStatus, newStatus, DateTimeOffset.UtcNow, note);

        // Notify all subscribers of this order
        await _hubContext.Clients
            .Group(OrderHub.GroupName(orderId))
            .OrderStatusChanged(message);
    }

    public async Task NotifyPaymentProcessedAsync(Guid orderId, Guid paymentId,
        decimal amount, string currency, bool success, string? failureReason = null,
        CancellationToken ct = default)
    {
        var message = new PaymentProcessedMessage(
            orderId, paymentId, amount, currency, success, failureReason);

        await _hubContext.Clients
            .Group(OrderHub.GroupName(orderId))
            .PaymentProcessed(message);
    }

    public async Task NotifyUserAsync(string userId,
        Func<IOrderHubClient, Task> notification, CancellationToken ct = default)
    {
        var client = _hubContext.Clients.Group(OrderHub.UserGroup(userId));
        await notification(client);
    }
}
```

```csharp
// Registration
builder.Services.AddScoped<IOrderNotificationService, OrderNotificationService>();
```

```csharp
// Usage in a command handler / domain event handler
public class OrderStatusChangedHandler : INotificationHandler<OrderStatusChangedDomainEvent>
{
    private readonly IOrderNotificationService _notifications;

    public OrderStatusChangedHandler(IOrderNotificationService notifications)
    {
        _notifications = notifications;
    }

    public async Task Handle(OrderStatusChangedDomainEvent notification,
        CancellationToken ct)
    {
        await _notifications.NotifyStatusChangedAsync(
            notification.OrderId,
            notification.OldStatus.ToString(),
            notification.NewStatus.ToString(),
            ct: ct);
    }
}
```

---

## Step 1881: SignalR Authentication & Authorization

```csharp
// Program.cs — JWT in SignalR (WebSocket can't send headers, so token goes in query string)
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(opts =>
    {
        opts.Events = new JwtBearerEvents
        {
            OnMessageReceived = context =>
            {
                // Read token from query string for WebSocket/SSE connections
                var accessToken = context.Request.Query["access_token"];
                var path = context.HttpContext.Request.Path;

                if (!string.IsNullOrEmpty(accessToken) && path.StartsWithSegments("/hubs"))
                    context.Token = accessToken;

                return Task.CompletedTask;
            }
        };
        // ... standard JWT validation params
    });
```

```csharp
// Hub with authorization
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.SignalR;

[Authorize]
public class OrderHub : Hub<IOrderHubClient>
{
    public async Task SubscribeToOrder(Guid orderId)
    {
        // Access the authenticated user
        var userId = Context.UserIdentifier; // Uses IUserIdProvider

        // Check order belongs to user (avoid IDOR)
        // var order = await _orderService.GetAsync(orderId, ct);
        // if (order.CustomerId.ToString() != userId) throw new HubException("Forbidden");

        await Groups.AddToGroupAsync(Context.ConnectionId, GroupName(orderId));
    }

    // Per-method authorization
    [Authorize(Policy = "AdminOnly")]
    public async Task BroadcastSystemAlert(string message)
    {
        await Clients.All.ErrorOccurred("SYSTEM_ALERT", message);
    }
}
```

```csharp
// Custom IUserIdProvider — maps connection to user identity
public class UserIdProvider : IUserIdProvider
{
    public string? GetUserId(HubConnectionContext connection)
    {
        // Default uses ClaimTypes.NameIdentifier
        // Custom: use "sub" claim from JWT
        return connection.User?.FindFirst("sub")?.Value;
    }
}

// Register
builder.Services.AddSingleton<IUserIdProvider, UserIdProvider>();
```

---

## Step 1882: Real-Time Dashboard with Live Metrics

```csharp
// Hubs/MetricsHub.cs
using Microsoft.AspNetCore.SignalR;

namespace RealtimeApp.Hubs;

public interface IMetricsHubClient
{
    Task MetricsSnapshot(MetricsSnapshot snapshot);
    Task MetricUpdated(string metricName, double value, DateTimeOffset timestamp);
}

public record MetricsSnapshot(
    int ActiveUsers,
    int OrdersPerMinute,
    decimal RevenueToday,
    double AvgResponseTimeMs,
    DateTimeOffset AsOf);

[Authorize]
public class MetricsHub : Hub<IMetricsHubClient>
{
    private readonly IMetricsCollector _collector;

    public MetricsHub(IMetricsCollector collector)
    {
        _collector = collector;
    }

    public override async Task OnConnectedAsync()
    {
        // Send current snapshot immediately on connection
        var snapshot = await _collector.GetCurrentSnapshotAsync();
        await Clients.Caller.MetricsSnapshot(snapshot);
        await base.OnConnectedAsync();
    }
}
```

```csharp
// Background service that pushes metrics every 5 seconds
public class MetricsBroadcastService : BackgroundService
{
    private readonly IHubContext<MetricsHub, IMetricsHubClient> _hubContext;
    private readonly IMetricsCollector _collector;
    private readonly ILogger<MetricsBroadcastService> _logger;

    public MetricsBroadcastService(
        IHubContext<MetricsHub, IMetricsHubClient> hubContext,
        IMetricsCollector collector,
        ILogger<MetricsBroadcastService> logger)
    {
        _hubContext = hubContext;
        _collector = collector;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromSeconds(5));

        while (await timer.WaitForNextTickAsync(ct))
        {
            try
            {
                var snapshot = await _collector.GetCurrentSnapshotAsync(ct);
                await _hubContext.Clients.All.MetricsSnapshot(snapshot);
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                _logger.LogError(ex, "Error broadcasting metrics");
            }
        }
    }
}
```

---

## Step 1883: Client-Side SignalR (.NET Client)

```csharp
// Clients/OrderServiceClient.cs
using Microsoft.AspNetCore.SignalR.Client;
using Microsoft.Extensions.Logging;

namespace RealtimeApp.Clients;

public sealed class OrderHubClient : IAsyncDisposable
{
    private readonly HubConnection _connection;
    private readonly ILogger<OrderHubClient> _logger;

    public event Action<OrderStatusChangedMessage>? OrderStatusChanged;
    public event Action<PaymentProcessedMessage>? PaymentProcessed;

    public OrderHubClient(string hubUrl, string accessToken,
        ILogger<OrderHubClient> logger)
    {
        _logger = logger;

        _connection = new HubConnectionBuilder()
            .WithUrl(hubUrl, opts =>
            {
                opts.AccessTokenProvider = () => Task.FromResult<string?>(accessToken);
            })
            .WithAutomaticReconnect(new[]
            {
                TimeSpan.Zero,
                TimeSpan.FromSeconds(2),
                TimeSpan.FromSeconds(10),
                TimeSpan.FromSeconds(30)
            })
            .ConfigureLogging(logging =>
            {
                logging.AddConsole();
                logging.SetMinimumLevel(LogLevel.Information);
            })
            .Build();

        RegisterHandlers();
        RegisterConnectionEvents();
    }

    private void RegisterHandlers()
    {
        _connection.On<OrderStatusChangedMessage>("OrderStatusChanged",
            msg => OrderStatusChanged?.Invoke(msg));

        _connection.On<PaymentProcessedMessage>("PaymentProcessed",
            msg => PaymentProcessed?.Invoke(msg));
    }

    private void RegisterConnectionEvents()
    {
        _connection.Reconnecting += error =>
        {
            _logger.LogWarning("Reconnecting due to: {Error}", error?.Message);
            return Task.CompletedTask;
        };

        _connection.Reconnected += connectionId =>
        {
            _logger.LogInformation("Reconnected with connectionId: {ConnectionId}", connectionId);
            return Task.CompletedTask;
        };

        _connection.Closed += error =>
        {
            if (error is not null)
                _logger.LogError(error, "Connection closed with error");
            return Task.CompletedTask;
        };
    }

    public async Task StartAsync(CancellationToken ct = default)
    {
        await _connection.StartAsync(ct);
        _logger.LogInformation("Connected to hub. ConnectionId: {ConnectionId}",
            _connection.ConnectionId);
    }

    public async Task SubscribeToOrderAsync(Guid orderId, CancellationToken ct = default)
    {
        await _connection.InvokeAsync("SubscribeToOrder", orderId, ct);
    }

    public async Task UnsubscribeFromOrderAsync(Guid orderId, CancellationToken ct = default)
    {
        await _connection.InvokeAsync("UnsubscribeFromOrder", orderId, ct);
    }

    public HubConnectionState State => _connection.State;

    public async ValueTask DisposeAsync()
    {
        await _connection.DisposeAsync();
    }
}
```

---

## Step 1884: Server-Sent Events (SSE) with IAsyncEnumerable

SSE is one-directional (server → client), simpler than WebSocket. Perfect for event streams.

```csharp
// Endpoints/EventStreamEndpoints.cs
using System.Runtime.CompilerServices;

namespace RealtimeApp.Endpoints;

public static class EventStreamEndpoints
{
    public static void MapEventStreamEndpoints(this IEndpointRouteBuilder app)
    {
        app.MapGet("/api/orders/{orderId}/events", StreamOrderEvents)
            .RequireAuthorization()
            .WithName("StreamOrderEvents")
            .Produces<OrderEvent>(200, "text/event-stream");

        app.MapGet("/api/metrics/stream", StreamMetrics)
            .RequireAuthorization()
            .Produces<MetricsSnapshot>(200, "text/event-stream");
    }

    // IAsyncEnumerable → ASP.NET Core streams as SSE automatically with Accept: text/event-stream
    private static async IAsyncEnumerable<OrderEvent> StreamOrderEvents(
        Guid orderId,
        IOrderEventStore eventStore,
        [EnumeratorCancellation] CancellationToken ct)
    {
        // Stream historical events first
        await foreach (var evt in eventStore.GetHistoricalEventsAsync(orderId, ct))
            yield return evt;

        // Then stream live events
        await foreach (var evt in eventStore.SubscribeToLiveEventsAsync(orderId, ct))
            yield return evt;
    }

    private static async IAsyncEnumerable<MetricsSnapshot> StreamMetrics(
        IMetricsCollector collector,
        [EnumeratorCancellation] CancellationToken ct)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromSeconds(5));

        while (await timer.WaitForNextTickAsync(ct))
        {
            yield return await collector.GetCurrentSnapshotAsync(ct);
        }
    }
}
```

```csharp
// Manual SSE endpoint with custom formatting
app.MapGet("/api/stream/manual", async (HttpContext context, CancellationToken ct) =>
{
    context.Response.Headers.ContentType = "text/event-stream";
    context.Response.Headers.CacheControl = "no-cache";
    context.Response.Headers.Connection = "keep-alive";
    context.Response.Headers["X-Accel-Buffering"] = "no"; // Disable nginx buffering

    await context.Response.Body.FlushAsync(ct);

    var eventId = 0;
    using var timer = new PeriodicTimer(TimeSpan.FromSeconds(1));

    while (await timer.WaitForNextTickAsync(ct))
    {
        var data = JsonSerializer.Serialize(new { time = DateTimeOffset.UtcNow });

        await context.Response.WriteAsync($"id: {eventId++}\n", ct);
        await context.Response.WriteAsync($"event: heartbeat\n", ct);
        await context.Response.WriteAsync($"data: {data}\n\n", ct);
        await context.Response.Body.FlushAsync(ct);
    }
});
```

---

## Step 1885: WebSocket Middleware

Low-level WebSocket for cases where SignalR overhead is undesirable.

```csharp
// Middleware/WebSocketMiddleware.cs
namespace RealtimeApp.Middleware;

public class OrderWebSocketMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<OrderWebSocketMiddleware> _logger;
    private readonly ConcurrentDictionary<string, WebSocket> _connections = new();

    public OrderWebSocketMiddleware(RequestDelegate next,
        ILogger<OrderWebSocketMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        if (!context.Request.Path.StartsWithSegments("/ws/orders"))
        {
            await _next(context);
            return;
        }

        if (!context.WebSockets.IsWebSocketRequest)
        {
            context.Response.StatusCode = 400;
            return;
        }

        var ws = await context.WebSockets.AcceptWebSocketAsync();
        var connectionId = Guid.NewGuid().ToString();
        _connections[connectionId] = ws;

        _logger.LogInformation("WebSocket connected: {ConnectionId}", connectionId);

        try
        {
            await HandleConnectionAsync(ws, connectionId, context.RequestAborted);
        }
        finally
        {
            _connections.TryRemove(connectionId, out _);
            _logger.LogInformation("WebSocket disconnected: {ConnectionId}", connectionId);
        }
    }

    private async Task HandleConnectionAsync(WebSocket ws, string connectionId,
        CancellationToken ct)
    {
        var buffer = new ArraySegment<byte>(new byte[4096]);

        while (ws.State == WebSocketState.Open)
        {
            WebSocketReceiveResult result;
            using var ms = new MemoryStream();

            do
            {
                result = await ws.ReceiveAsync(buffer, ct);
                if (result.Count > 0)
                    ms.Write(buffer.Array!, buffer.Offset, result.Count);
            }
            while (!result.EndOfMessage);

            if (result.MessageType == WebSocketMessageType.Close)
            {
                await ws.CloseAsync(WebSocketCloseStatus.NormalClosure, "Closing", ct);
                break;
            }

            if (result.MessageType == WebSocketMessageType.Text)
            {
                var message = Encoding.UTF8.GetString(ms.ToArray());
                await ProcessMessageAsync(ws, connectionId, message, ct);
            }
        }
    }

    private async Task ProcessMessageAsync(WebSocket ws, string connectionId,
        string message, CancellationToken ct)
    {
        var request = JsonSerializer.Deserialize<WsRequest>(message);
        if (request is null) return;

        var response = request.Type switch
        {
            "ping" => new WsResponse("pong", null),
            "subscribe" => new WsResponse("subscribed", new { connectionId }),
            _ => new WsResponse("error", new { message = "Unknown message type" })
        };

        var bytes = JsonSerializer.SerializeToUtf8Bytes(response);
        await ws.SendAsync(bytes, WebSocketMessageType.Text, true, ct);
    }

    public async Task BroadcastAsync(string message, CancellationToken ct = default)
    {
        var bytes = Encoding.UTF8.GetBytes(message);
        var segment = new ArraySegment<byte>(bytes);

        var tasks = _connections.Values
            .Where(ws => ws.State == WebSocketState.Open)
            .Select(ws => ws.SendAsync(segment, WebSocketMessageType.Text, true, ct));

        await Task.WhenAll(tasks);
    }

    private record WsRequest(string Type, JsonElement? Payload);
    private record WsResponse(string Type, object? Data);
}
```

```csharp
// Register
app.UseWebSockets(new WebSocketOptions
{
    KeepAliveInterval = TimeSpan.FromMinutes(2)
});
app.UseMiddleware<OrderWebSocketMiddleware>();
```

---

## Step 1886: SignalR Redis Backplane — Horizontal Scaling

Without a backplane, SignalR only works on a single server — groups and user connections are local.

```bash
dotnet add package Microsoft.AspNetCore.SignalR.StackExchangeRedis
```

```csharp
// Program.cs
builder.Services.AddSignalR()
    .AddStackExchangeRedis(builder.Configuration.GetConnectionString("Redis")!, opts =>
    {
        opts.Configuration.ChannelPrefix = RedisChannel.Literal("myapp");
        opts.Configuration.ConnectTimeout = 5000;
        opts.Configuration.SyncTimeout = 3000;
        opts.Configuration.ConnectRetry = 3;
        opts.Configuration.ReconnectRetryPolicy = new LinearRetry(1000);
    });
```

```csharp
// appsettings.json
{
  "ConnectionStrings": {
    "Redis": "localhost:6379,abortConnect=false,ssl=false,password=yourpassword"
  }
}
```

With Redis backplane:
- Server A has connection for User1
- Server B wants to send to User1's group
- Message goes via Redis pub/sub → Server A delivers it

---

## Step 1887: Presence Tracking — Online/Offline Users

```csharp
// Services/PresenceTracker.cs
using Microsoft.Extensions.Caching.Distributed;

namespace RealtimeApp.Services;

public interface IPresenceTracker
{
    Task<bool> UserConnectedAsync(string userId, string connectionId);
    Task<bool> UserDisconnectedAsync(string userId, string connectionId);
    Task<IReadOnlyList<string>> GetOnlineUsersAsync();
    Task<bool> IsOnlineAsync(string userId);
}

public sealed class RedisPresenceTracker : IPresenceTracker
{
    private readonly IConnectionMultiplexer _redis;
    private const string OnlineUsersKey = "presence:online";
    private const string ConnectionsKeyPrefix = "presence:connections:";

    public RedisPresenceTracker(IConnectionMultiplexer redis)
    {
        _redis = redis;
    }

    public async Task<bool> UserConnectedAsync(string userId, string connectionId)
    {
        var db = _redis.GetDatabase();
        var key = $"{ConnectionsKeyPrefix}{userId}";

        await db.SetAddAsync(key, connectionId);
        var isNewUser = await db.SetAddAsync(OnlineUsersKey, userId);
        await db.KeyExpireAsync(key, TimeSpan.FromHours(24));

        return isNewUser; // true if user was previously offline
    }

    public async Task<bool> UserDisconnectedAsync(string userId, string connectionId)
    {
        var db = _redis.GetDatabase();
        var key = $"{ConnectionsKeyPrefix}{userId}";

        await db.SetRemoveAsync(key, connectionId);
        var remainingConnections = await db.SetLengthAsync(key);

        if (remainingConnections == 0)
        {
            await db.KeyDeleteAsync(key);
            await db.SetRemoveAsync(OnlineUsersKey, userId);
            return true; // user went fully offline
        }

        return false;
    }

    public async Task<IReadOnlyList<string>> GetOnlineUsersAsync()
    {
        var db = _redis.GetDatabase();
        var members = await db.SetMembersAsync(OnlineUsersKey);
        return members.Select(m => m.ToString()).ToList();
    }

    public async Task<bool> IsOnlineAsync(string userId)
    {
        var db = _redis.GetDatabase();
        return await db.SetContainsAsync(OnlineUsersKey, userId);
    }
}
```

```csharp
// Hub with presence tracking
public class ChatHub : Hub<IChatHubClient>
{
    private readonly IPresenceTracker _presence;
    private readonly IHubContext<ChatHub, IChatHubClient> _hubContext;

    public ChatHub(IPresenceTracker presence,
        IHubContext<ChatHub, IChatHubClient> hubContext)
    {
        _presence = presence;
        _hubContext = hubContext;
    }

    public override async Task OnConnectedAsync()
    {
        var userId = Context.UserIdentifier!;
        var isNewlyOnline = await _presence.UserConnectedAsync(userId, Context.ConnectionId);

        if (isNewlyOnline)
        {
            // Notify others that this user came online
            await _hubContext.Clients.Others.UserCameOnline(new UserPresenceMessage(userId, true));
        }

        await base.OnConnectedAsync();
    }

    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        var userId = Context.UserIdentifier!;
        var isNowOffline = await _presence.UserDisconnectedAsync(userId, Context.ConnectionId);

        if (isNowOffline)
        {
            await _hubContext.Clients.Others.UserWentOffline(new UserPresenceMessage(userId, false));
        }

        await base.OnDisconnectedAsync(exception);
    }
}
```

---

## Step 1888: SignalR with Channels — Streaming

Server → Client streaming: server sends multiple results progressively.

```csharp
// Hub streaming method
using System.Threading.Channels;

public class ReportHub : Hub
{
    private readonly IReportService _reportService;

    public ReportHub(IReportService reportService)
    {
        _reportService = reportService;
    }

    // ChannelReader<T> — server streams to client
    public async IAsyncEnumerable<ReportRow> StreamReport(
        ReportRequest request,
        [EnumeratorCancellation] CancellationToken ct)
    {
        await foreach (var row in _reportService.GenerateReportAsync(request, ct))
        {
            yield return row;
        }
    }

    // Channel-based approach (older API, still valid)
    public ChannelReader<ReportRow> StreamReportViaChannel(ReportRequest request)
    {
        var channel = Channel.CreateUnbounded<ReportRow>();

        _ = GenerateAsync(channel.Writer, request);

        return channel.Reader;
    }

    private async Task GenerateAsync(ChannelWriter<ReportRow> writer, ReportRequest request)
    {
        try
        {
            await foreach (var row in _reportService.GenerateReportAsync(request))
                await writer.WriteAsync(row);
        }
        catch (Exception ex)
        {
            writer.TryComplete(ex);
            return;
        }

        writer.TryComplete();
    }
}
```

```csharp
// Client receives the stream
var hubConnection = new HubConnectionBuilder()
    .WithUrl("https://localhost:5001/hubs/reports")
    .Build();

await hubConnection.StartAsync();

await foreach (var row in hubConnection.StreamAsync<ReportRow>(
    "StreamReport", new ReportRequest { Year = 2024 }))
{
    Console.WriteLine($"Row: {row}");
}
```

---

## Step 1889: Client → Server Streaming

Client streams data to server (e.g., file upload with progress, bulk inserts).

```csharp
// Hub accepts IAsyncEnumerable from client
public class UploadHub : Hub
{
    private readonly IBulkImportService _importService;

    public UploadHub(IBulkImportService importService)
    {
        _importService = importService;
    }

    public async Task<ImportResult> BulkImportProducts(
        IAsyncEnumerable<ProductImportRow> rows)
    {
        var processed = 0;
        var errors = new List<string>();

        await foreach (var row in rows)
        {
            try
            {
                await _importService.ImportAsync(row);
                processed++;
            }
            catch (Exception ex)
            {
                errors.Add($"Row {processed + 1}: {ex.Message}");
            }
        }

        return new ImportResult(processed, errors);
    }
}

public record ProductImportRow(string Sku, string Name, decimal Price, int Stock);
public record ImportResult(int Processed, List<string> Errors);
```

```csharp
// Client streams data
var channel = Channel.CreateBounded<ProductImportRow>(10);

// Producer task
_ = Task.Run(async () =>
{
    await foreach (var row in ReadCsvAsync("products.csv"))
        await channel.Writer.WriteAsync(row);
    channel.Writer.Complete();
});

var result = await connection.InvokeAsync<ImportResult>(
    "BulkImportProducts",
    channel.Reader.ReadAllAsync());

Console.WriteLine($"Imported {result.Processed} rows with {result.Errors.Count} errors");
```

---

## Step 1890: Real-Time Notifications System

```csharp
// Domain/Notifications/Notification.cs
namespace RealtimeApp.Domain;

public sealed class Notification
{
    public Guid Id { get; private set; }
    public string UserId { get; private set; }
    public NotificationType Type { get; private set; }
    public string Title { get; private set; }
    public string Body { get; private set; }
    public string? ActionUrl { get; private set; }
    public bool IsRead { get; private set; }
    public DateTimeOffset CreatedAt { get; private set; }
    public DateTimeOffset? ReadAt { get; private set; }

    private Notification() { }

    public static Notification Create(
        string userId,
        NotificationType type,
        string title,
        string body,
        string? actionUrl = null)
    {
        return new Notification
        {
            Id = Guid.NewGuid(),
            UserId = userId,
            Type = type,
            Title = title,
            Body = body,
            ActionUrl = actionUrl,
            IsRead = false,
            CreatedAt = DateTimeOffset.UtcNow
        };
    }

    public void MarkAsRead()
    {
        IsRead = true;
        ReadAt = DateTimeOffset.UtcNow;
    }
}

public enum NotificationType
{
    Info,
    Success,
    Warning,
    Error,
    OrderUpdate,
    PaymentAlert,
    SystemAnnouncement
}
```

```csharp
// Services/NotificationDispatcher.cs
public class NotificationDispatcher
{
    private readonly IHubContext<NotificationHub, INotificationHubClient> _hubContext;
    private readonly INotificationRepository _repository;
    private readonly ILogger<NotificationDispatcher> _logger;

    public NotificationDispatcher(
        IHubContext<NotificationHub, INotificationHubClient> hubContext,
        INotificationRepository repository,
        ILogger<NotificationDispatcher> logger)
    {
        _hubContext = hubContext;
        _repository = repository;
        _logger = logger;
    }

    public async Task SendToUserAsync(
        string userId,
        NotificationType type,
        string title,
        string body,
        string? actionUrl = null,
        CancellationToken ct = default)
    {
        var notification = Notification.Create(userId, type, title, body, actionUrl);
        await _repository.SaveAsync(notification, ct);

        var dto = new NotificationDto(
            notification.Id,
            notification.Type,
            notification.Title,
            notification.Body,
            notification.ActionUrl,
            notification.CreatedAt);

        // Push in real-time (fire-and-forget if user is offline — they'll get it on reconnect)
        await _hubContext.Clients
            .User(userId)
            .ReceiveNotification(dto);

        _logger.LogInformation("Notification {NotificationId} sent to user {UserId}",
            notification.Id, userId);
    }

    public async Task SendToAllAsync(
        NotificationType type,
        string title,
        string body,
        CancellationToken ct = default)
    {
        var dto = new NotificationDto(Guid.NewGuid(), type, title, body, null,
            DateTimeOffset.UtcNow);

        await _hubContext.Clients.All.ReceiveNotification(dto);
    }
}
```

```csharp
// Notification Hub — with unread count on connect
public interface INotificationHubClient
{
    Task ReceiveNotification(NotificationDto notification);
    Task NotificationRead(Guid notificationId);
    Task UnreadCountChanged(int count);
}

public class NotificationHub : Hub<INotificationHubClient>
{
    private readonly INotificationRepository _repository;

    public NotificationHub(INotificationRepository repository)
    {
        _repository = repository;
    }

    public override async Task OnConnectedAsync()
    {
        var userId = Context.UserIdentifier!;
        var unreadCount = await _repository.GetUnreadCountAsync(userId);

        if (unreadCount > 0)
            await Clients.Caller.UnreadCountChanged(unreadCount);

        await base.OnConnectedAsync();
    }

    public async Task MarkAsRead(Guid notificationId)
    {
        var userId = Context.UserIdentifier!;
        await _repository.MarkAsReadAsync(notificationId, userId);

        var newCount = await _repository.GetUnreadCountAsync(userId);
        await Clients.Caller.UnreadCountChanged(newCount);
        await Clients.Caller.NotificationRead(notificationId);
    }

    public async Task<IReadOnlyList<NotificationDto>> GetUnread()
    {
        var userId = Context.UserIdentifier!;
        return await _repository.GetUnreadAsync(userId);
    }
}
```

---

## Step 1891: SignalR Filters & Middleware

```csharp
// Filters/HubActivityFilter.cs
using Microsoft.AspNetCore.SignalR;

namespace RealtimeApp.Filters;

public class HubActivityFilter : IHubFilter
{
    private readonly ILogger<HubActivityFilter> _logger;
    private readonly ActivitySource _activitySource = new("RealtimeApp.SignalR");

    public HubActivityFilter(ILogger<HubActivityFilter> logger)
    {
        _logger = logger;
    }

    public async ValueTask<object?> InvokeMethodAsync(
        HubInvocationContext invocationContext,
        Func<HubInvocationContext, ValueTask<object?>> next)
    {
        var hubName = invocationContext.Hub.GetType().Name;
        var methodName = invocationContext.HubMethodName;
        var userId = invocationContext.Context.UserIdentifier ?? "anonymous";

        using var activity = _activitySource.StartActivity($"{hubName}/{methodName}");
        activity?.SetTag("hub.user_id", userId);
        activity?.SetTag("hub.connection_id", invocationContext.Context.ConnectionId);

        try
        {
            var result = await next(invocationContext);
            activity?.SetStatus(ActivityStatusCode.Ok);
            return result;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Hub method {HubName}/{MethodName} failed for user {UserId}",
                hubName, methodName, userId);
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            throw new HubException($"An error occurred executing {methodName}");
        }
    }

    public Task OnConnectedAsync(HubLifetimeContext context,
        Func<HubLifetimeContext, Task> next)
    {
        _logger.LogInformation("Hub connected: {ConnectionId} ({HubType})",
            context.Context.ConnectionId, context.Hub.GetType().Name);
        return next(context);
    }

    public Task OnDisconnectedAsync(HubLifetimeContext context,
        Exception? exception,
        Func<HubLifetimeContext, Exception?, Task> next)
    {
        _logger.LogInformation("Hub disconnected: {ConnectionId}", context.Context.ConnectionId);
        return next(context, exception);
    }
}
```

```csharp
// Register filter
builder.Services.AddSignalR(opts =>
{
    opts.AddFilter<HubActivityFilter>();
});

builder.Services.AddSingleton<HubActivityFilter>();
```

---

## Step 1892: Complete Program.cs & Integration Test

```csharp
// Program.cs — complete setup
using RealtimeApp.Hubs;
using RealtimeApp.Services;
using RealtimeApp.Filters;
using StackExchange.Redis;

var builder = WebApplication.CreateBuilder(args);

// SignalR
builder.Services.AddSignalR(opts =>
{
    opts.EnableDetailedErrors = builder.Environment.IsDevelopment();
    opts.KeepAliveInterval = TimeSpan.FromSeconds(15);
    opts.ClientTimeoutInterval = TimeSpan.FromSeconds(30);
    opts.AddFilter<HubActivityFilter>();
})
.AddStackExchangeRedis(builder.Configuration.GetConnectionString("Redis")!);

// Redis for presence tracking
builder.Services.AddSingleton<IConnectionMultiplexer>(sp =>
    ConnectionMultiplexer.Connect(builder.Configuration.GetConnectionString("Redis")!));

// Services
builder.Services.AddSingleton<IPresenceTracker, RedisPresenceTracker>();
builder.Services.AddScoped<IOrderNotificationService, OrderNotificationService>();
builder.Services.AddScoped<NotificationDispatcher>();
builder.Services.AddHostedService<MetricsBroadcastService>();
builder.Services.AddSingleton<HubActivityFilter>();

// Auth
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(opts =>
    {
        opts.Events = new JwtBearerEvents
        {
            OnMessageReceived = context =>
            {
                var token = context.Request.Query["access_token"];
                if (!string.IsNullOrEmpty(token) &&
                    context.Request.Path.StartsWithSegments("/hubs"))
                    context.Token = token;
                return Task.CompletedTask;
            }
        };
    });

builder.Services.AddSingleton<IUserIdProvider, UserIdProvider>();

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();
app.UseWebSockets();
app.UseMiddleware<OrderWebSocketMiddleware>();

// Map hubs
app.MapHub<OrderHub>("/hubs/orders");
app.MapHub<MetricsHub>("/hubs/metrics");
app.MapHub<NotificationHub>("/hubs/notifications");
app.MapHub<ChatHub>("/hubs/chat");

// Map SSE endpoints
app.MapEventStreamEndpoints();

app.Run();
```

```csharp
// Integration test
using Microsoft.AspNetCore.SignalR.Client;
using Microsoft.AspNetCore.TestHost;

public class OrderHubIntegrationTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;

    public OrderHubIntegrationTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory;
    }

    [Fact]
    public async Task SubscribeAndReceiveOrderStatusChange()
    {
        var client = _factory.CreateClient();
        var server = _factory.Server;

        var connection = new HubConnectionBuilder()
            .WithUrl("http://localhost/hubs/orders", opts =>
            {
                opts.HttpMessageHandlerFactory = _ => server.CreateHandler();
                opts.AccessTokenProvider = () => Task.FromResult<string?>(
                    GenerateTestToken("user-1"));
            })
            .Build();

        var received = new List<OrderStatusChangedMessage>();
        var tcs = new TaskCompletionSource<bool>();

        connection.On<OrderStatusChangedMessage>("OrderStatusChanged", msg =>
        {
            received.Add(msg);
            tcs.TrySetResult(true);
        });

        await connection.StartAsync();
        var orderId = Guid.NewGuid();
        await connection.InvokeAsync("SubscribeToOrder", orderId);

        // Trigger notification via IHubContext
        var notifier = _factory.Services.GetRequiredService<IOrderNotificationService>();
        await notifier.NotifyStatusChangedAsync(orderId, "Pending", "Confirmed");

        await tcs.Task.WaitAsync(TimeSpan.FromSeconds(5));

        received.Should().HaveCount(1);
        received[0].NewStatus.Should().Be("Confirmed");
        received[0].OldStatus.Should().Be("Pending");

        await connection.DisposeAsync();
    }

    private static string GenerateTestToken(string userId)
    {
        // Generate a test JWT for the given userId
        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes("test-secret-key-32chars!!!!!!!!"));
        var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);
        var token = new JwtSecurityToken(
            claims: [new Claim("sub", userId)],
            expires: DateTime.UtcNow.AddHours(1),
            signingCredentials: creds);
        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}
```

---

## Summary

| Concept | Key Type | Purpose |
|---------|----------|---------|
| Hub | `Hub<T>` | Server-side connection manager |
| Strongly-typed | `IOrderHubClient` | Compile-time client method safety |
| Server push | `IHubContext<THub,T>` | Push from services, not just hub |
| SSE | `IAsyncEnumerable<T>` | Simple one-way server stream |
| WebSocket | `WebSocketMiddleware` | Low-level full-duplex |
| Scaling | Redis backplane | Multi-server message relay |
| Presence | `IPresenceTracker` | Online/offline user state |
| Streaming | `ChannelReader<T>` | Progressive server-to-client data |
| Filters | `IHubFilter` | Cross-cutting hub concerns |
| Auth | `IUserIdProvider` + JWT query | WebSocket token handling |

**Next**: Part 79 — Microservices Architecture: Service Mesh, API Gateway, Service Discovery
