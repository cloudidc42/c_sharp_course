# Part 49: Real-Time Applications & SignalR

## Steps 1381-1420: Real-Time Communication ระดับโลก

---

## Step 1381: SignalR Architecture

```
┌──────────────────────────────────────────────────────────┐
│                    SIGNALR TRANSPORT                      │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  Client                     Server                      │
│  ┌──────────┐               ┌──────────────┐            │
│  │ Browser  │ WebSocket     │   Hub        │            │
│  │ Mobile   │ ◄────────────►│   Methods    │            │
│  │ .NET App │               │   Groups     │            │
│  └──────────┘               │   Users      │            │
│                             └──────┬───────┘            │
│  Fallbacks (in order):            │                     │
│  1. WebSocket (preferred)         │                     │
│  2. Server-Sent Events            ├─► Redis Backplane   │
│  3. Long Polling                  │   (scale-out)       │
│                                   ├─► Azure SignalR     │
│                                   │   Service           │
│                                   └─► SQL Server        │
└──────────────────────────────────────────────────────────┘
```

---

## Step 1382: Basic SignalR Hub Setup

```xml
<ItemGroup>
  <PackageReference Include="Microsoft.AspNetCore.SignalR.StackExchangeRedis" Version="9.*" />
</ItemGroup>
```

```csharp
// Hub definition
public class OrderHub(
    IOrderRepository orders,
    ILogger<OrderHub> logger) : Hub<IOrderClient>
{
    // Called by client to join a customer's order group
    public async Task JoinCustomerGroup(Guid customerId)
    {
        var userId = Context.User?.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        if (userId != customerId.ToString())
        {
            logger.LogWarning("User {UserId} tried to join group for customer {CustomerId}",
                userId, customerId);
            throw new HubException("Unauthorized: cannot join other customer's group");
        }

        await Groups.AddToGroupAsync(Context.ConnectionId, $"customer:{customerId}");
        logger.LogInformation("Connection {ConnId} joined group customer:{CustomerId}",
            Context.ConnectionId, customerId);
    }

    public async Task LeaveCustomerGroup(Guid customerId)
    {
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, $"customer:{customerId}");
    }

    // Real-time order status request
    public async Task<OrderStatusResponse> GetOrderStatus(Guid orderId)
    {
        var order = await orders.GetByIdAsync(orderId, Context.ConnectionAborted);
        if (order is null)
            throw new HubException($"Order {orderId} not found");

        return new OrderStatusResponse(order.Id, order.Status.ToString(), order.UpdatedAt);
    }

    // Hub lifecycle callbacks
    public override async Task OnConnectedAsync()
    {
        var userId = Context.User?.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        logger.LogInformation("Client connected: {ConnId} user:{UserId}", Context.ConnectionId, userId);

        if (userId is not null)
        {
            await Groups.AddToGroupAsync(Context.ConnectionId, $"user:{userId}");
            await Clients.Caller.Connected(new ConnectedMessage(Context.ConnectionId, DateTimeOffset.UtcNow));
        }

        await base.OnConnectedAsync();
    }

    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        var userId = Context.User?.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        logger.LogInformation("Client disconnected: {ConnId} user:{UserId} reason:{Reason}",
            Context.ConnectionId, userId, exception?.Message ?? "clean");
        await base.OnDisconnectedAsync(exception);
    }
}

// Strongly-typed client interface
public interface IOrderClient
{
    Task Connected(ConnectedMessage message);
    Task OrderStatusUpdated(OrderStatusUpdate update);
    Task OrderCreated(OrderCreatedNotification notification);
    Task InventoryAlert(InventoryAlertMessage alert);
}

// Message types
public record OrderStatusUpdate(Guid OrderId, string Status, DateTimeOffset UpdatedAt);
public record OrderCreatedNotification(Guid OrderId, decimal TotalAmount, DateTimeOffset CreatedAt);
public record ConnectedMessage(string ConnectionId, DateTimeOffset ConnectedAt);
public record OrderStatusResponse(Guid OrderId, string Status, DateTimeOffset? UpdatedAt);
public record InventoryAlertMessage(Guid ProductId, string ProductName, int AvailableQuantity);
```

```csharp
// Program.cs registration
builder.Services.AddSignalR(options =>
{
    options.EnableDetailedErrors = builder.Environment.IsDevelopment();
    options.MaximumReceiveMessageSize = 32 * 1024; // 32 KB
    options.ClientTimeoutInterval = TimeSpan.FromSeconds(30);
    options.KeepAliveInterval = TimeSpan.FromSeconds(15);
    options.HandshakeTimeout = TimeSpan.FromSeconds(15);
})
.AddJsonProtocol(options =>
{
    options.PayloadSerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;
})
.AddStackExchangeRedis(builder.Configuration.GetConnectionString("Redis")!,
    options =>
    {
        options.Configuration.ChannelPrefix = RedisChannel.Literal("signalr:");
    });

// Authorization for hub
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("HubAccess", policy => policy.RequireAuthenticatedUser());
});

app.MapHub<OrderHub>("/hubs/orders", options =>
{
    options.Transports = HttpTransportType.WebSockets | HttpTransportType.ServerSentEvents;
    options.CloseOnAuthenticationExpiration = true;
})
.RequireAuthorization("HubAccess");
```

---

## Step 1383: Sending Messages from Services

```csharp
// IHubContext — send messages from anywhere (background services, API endpoints)
public class OrderNotificationService(
    IHubContext<OrderHub, IOrderClient> hubContext,
    ILogger<OrderNotificationService> logger)
{
    public async Task NotifyOrderStatusChangeAsync(
        Order order, string oldStatus, CancellationToken ct)
    {
        var update = new OrderStatusUpdate(order.Id, order.Status.ToString(), order.UpdatedAt);

        // Send to customer's group
        await hubContext.Clients
            .Group($"customer:{order.CustomerId}")
            .OrderStatusUpdated(update);

        // Also send to admin group if high-value order
        if (order.TotalAmount > 10_000)
        {
            await hubContext.Clients
                .Group("admin:orders")
                .OrderStatusUpdated(update);
        }

        logger.LogDebug("Notified order {OrderId} status change: {Old}→{New}",
            order.Id, oldStatus, order.Status);
    }

    public async Task NotifyOrderCreatedAsync(Order order, CancellationToken ct)
    {
        var notification = new OrderCreatedNotification(
            order.Id, order.TotalAmount, order.CreatedAt);

        // Notify the specific customer
        await hubContext.Clients
            .User(order.CustomerId.ToString())
            .OrderCreated(notification);

        // Broadcast to warehouse staff
        await hubContext.Clients
            .Group("warehouse:staff")
            .OrderCreated(notification);
    }

    public async Task BroadcastInventoryAlertAsync(
        Guid productId, string productName, int quantity, CancellationToken ct)
    {
        var alert = new InventoryAlertMessage(productId, productName, quantity);

        // Send to all connected clients
        await hubContext.Clients.All.InventoryAlert(alert);
    }
}

// Publish from Kafka consumer
public class OrderEventConsumer(
    OrderNotificationService notifications,
    ILogger<OrderEventConsumer> logger) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var consumer = BuildKafkaConsumer();
        consumer.Subscribe("order-events");

        while (!stoppingToken.IsCancellationRequested)
        {
            var result = consumer.Consume(TimeSpan.FromSeconds(1));
            if (result?.Message is null) continue;

            try
            {
                var @event = JsonSerializer.Deserialize<OrderEvent>(result.Message.Value);
                if (@event is not null)
                    await ProcessEventAsync(@event, stoppingToken);

                consumer.Commit(result);
            }
            catch (Exception ex)
            {
                logger.LogError(ex, "Failed to process Kafka message");
            }
        }
    }

    private async Task ProcessEventAsync(OrderEvent @event, CancellationToken ct)
    {
        if (@event.Type == "OrderStatusChanged")
        {
            // Signal connected clients in real-time
            await notifications.NotifyOrderStatusChangeAsync(
                @event.Order!, @event.OldStatus!, ct);
        }
    }

    private static IConsumer<string, string> BuildKafkaConsumer() =>
        new ConsumerBuilder<string, string>(new ConsumerConfig
        {
            BootstrapServers = "kafka:9092",
            GroupId = "signalr-relay",
            AutoOffsetReset = AutoOffsetReset.Latest,
        }).Build();
}
```

---

## Step 1384: JavaScript / TypeScript Client

```typescript
// TypeScript SignalR client
import * as signalR from "@microsoft/signalr";

interface OrderStatusUpdate {
  orderId: string;
  status: string;
  updatedAt: string;
}

interface OrderCreatedNotification {
  orderId: string;
  totalAmount: number;
  createdAt: string;
}

class OrderHubClient {
  private connection: signalR.HubConnection;

  constructor(private accessToken: () => string) {
    this.connection = new signalR.HubConnectionBuilder()
      .withUrl("/hubs/orders", {
        accessTokenFactory: accessToken,
        transport: signalR.HttpTransportType.WebSockets,
      })
      .withAutomaticReconnect({
        nextRetryDelayInMilliseconds: (retryContext) => {
          // Exponential backoff: 0, 2s, 4s, 8s, 30s, 30s, ...
          const delays = [0, 2000, 4000, 8000, 30000];
          return delays[Math.min(retryContext.previousRetryCount, delays.length - 1)];
        },
      })
      .withStatefulReconnect()
      .configureLogging(signalR.LogLevel.Information)
      .build();

    this.setupHandlers();
  }

  private setupHandlers(): void {
    this.connection.on("orderStatusUpdated", (update: OrderStatusUpdate) => {
      console.log(`Order ${update.orderId} → ${update.status}`);
      this.onOrderStatusUpdated?.(update);
    });

    this.connection.on("orderCreated", (notif: OrderCreatedNotification) => {
      this.onOrderCreated?.(notif);
    });

    this.connection.onreconnecting((error) => {
      console.warn("SignalR reconnecting...", error);
      this.onReconnecting?.();
    });

    this.connection.onreconnected((connectionId) => {
      console.info(`SignalR reconnected: ${connectionId}`);
      this.onReconnected?.();
    });

    this.connection.onclose((error) => {
      console.error("SignalR connection closed", error);
      this.onClosed?.();
    });
  }

  async start(): Promise<void> {
    await this.connection.start();
    console.info("SignalR connected:", this.connection.connectionId);
  }

  async stop(): Promise<void> {
    await this.connection.stop();
  }

  async joinCustomerGroup(customerId: string): Promise<void> {
    await this.connection.invoke("JoinCustomerGroup", customerId);
  }

  async getOrderStatus(orderId: string): Promise<{ status: string }> {
    return await this.connection.invoke("GetOrderStatus", orderId);
  }

  // Event callbacks (set by consumer)
  onOrderStatusUpdated?: (update: OrderStatusUpdate) => void;
  onOrderCreated?: (notif: OrderCreatedNotification) => void;
  onReconnecting?: () => void;
  onReconnected?: () => void;
  onClosed?: () => void;
}

// React hook usage
export function useOrderHub(customerId: string) {
  const [orderStatus, setOrderStatus] = React.useState<Record<string, string>>({});
  const hubRef = React.useRef<OrderHubClient | null>(null);

  React.useEffect(() => {
    const hub = new OrderHubClient(() => localStorage.getItem("access_token") ?? "");
    hubRef.current = hub;

    hub.onOrderStatusUpdated = (update) => {
      setOrderStatus(prev => ({ ...prev, [update.orderId]: update.status }));
    };

    hub.start()
      .then(() => hub.joinCustomerGroup(customerId))
      .catch(console.error);

    return () => { hub.stop().catch(console.error); };
  }, [customerId]);

  return { orderStatus };
}
```

---

## Step 1385: .NET SignalR Client

```csharp
// .NET client (for microservice-to-microservice real-time)
using Microsoft.AspNetCore.SignalR.Client;

public class OrderHubClient(ILogger<OrderHubClient> logger) : IAsyncDisposable
{
    private HubConnection? _connection;

    public async Task ConnectAsync(string hubUrl, string accessToken, CancellationToken ct)
    {
        _connection = new HubConnectionBuilder()
            .WithUrl(hubUrl, options =>
            {
                options.AccessTokenProvider = () => Task.FromResult(accessToken)!;
                options.Transports = HttpTransportType.WebSockets;
            })
            .WithAutomaticReconnect(new ExponentialBackoffReconnectPolicy())
            .AddJsonProtocol()
            .Build();

        // Register handlers
        _connection.On<OrderStatusUpdate>("orderStatusUpdated", update =>
        {
            logger.LogInformation("Order {OrderId} status → {Status}", update.OrderId, update.Status);
            OrderStatusUpdated?.Invoke(update);
        });

        _connection.Reconnecting += error =>
        {
            logger.LogWarning("SignalR reconnecting: {Error}", error?.Message);
            return Task.CompletedTask;
        };

        _connection.Reconnected += connectionId =>
        {
            logger.LogInformation("SignalR reconnected: {ConnId}", connectionId);
            return Task.CompletedTask;
        };

        await _connection.StartAsync(ct);
        logger.LogInformation("Connected to hub: {ConnId}", _connection.ConnectionId);
    }

    public event Action<OrderStatusUpdate>? OrderStatusUpdated;

    public async Task JoinGroupAsync(string groupName, CancellationToken ct)
        => await _connection!.InvokeAsync("JoinGroup", groupName, ct);

    public async ValueTask DisposeAsync()
    {
        if (_connection is not null)
            await _connection.DisposeAsync();
    }
}

// Exponential backoff reconnect policy
public class ExponentialBackoffReconnectPolicy : IRetryPolicy
{
    private static readonly TimeSpan[] Delays =
    [
        TimeSpan.Zero,
        TimeSpan.FromSeconds(2),
        TimeSpan.FromSeconds(4),
        TimeSpan.FromSeconds(8),
        TimeSpan.FromSeconds(30),
    ];

    public TimeSpan? NextRetryDelay(RetryContext retryContext)
    {
        if (retryContext.PreviousRetryCount >= 10) return null; // Give up
        var index = Math.Min(retryContext.PreviousRetryCount, Delays.Length - 1);
        return Delays[index];
    }
}
```

---

## Step 1386: Server-Sent Events (SSE)

```csharp
// SSE — simpler than WebSocket for one-way server→client streaming
// Great for: live feeds, progress updates, log streaming

app.MapGet("/api/orders/stream", async (
    Guid customerId,
    IOrderEventStream eventStream,
    HttpContext context,
    CancellationToken ct) =>
{
    context.Response.Headers.ContentType = "text/event-stream";
    context.Response.Headers.CacheControl = "no-cache";
    context.Response.Headers.Connection = "keep-alive";
    context.Response.Headers["X-Accel-Buffering"] = "no"; // Disable Nginx buffering

    await context.Response.Body.FlushAsync(ct);

    await foreach (var update in eventStream.SubscribeAsync(customerId, ct))
    {
        var data = JsonSerializer.Serialize(update);
        await context.Response.WriteAsync($"id: {Guid.NewGuid()}\n", ct);
        await context.Response.WriteAsync($"event: orderUpdate\n", ct);
        await context.Response.WriteAsync($"data: {data}\n\n", ct);
        await context.Response.Body.FlushAsync(ct);
    }
});

// SSE with retry hint
app.MapGet("/api/events", async (HttpContext context, CancellationToken ct) =>
{
    context.Response.Headers.ContentType = "text/event-stream";

    // Tell client to retry after 3s on disconnect
    await context.Response.WriteAsync("retry: 3000\n\n", ct);
    await context.Response.Body.FlushAsync(ct);

    while (!ct.IsCancellationRequested)
    {
        var timestamp = DateTimeOffset.UtcNow;
        await context.Response.WriteAsync(
            $"data: {{\"time\":\"{timestamp:O}\"}}\n\n", ct);
        await context.Response.Body.FlushAsync(ct);
        await Task.Delay(TimeSpan.FromSeconds(1), ct);
    }
});

// Typed SSE helper
public static class SseHelper
{
    public static async Task WriteEventAsync(
        HttpContext context,
        string eventName,
        object data,
        string? id = null,
        CancellationToken ct = default)
    {
        var json = JsonSerializer.Serialize(data, new JsonSerializerOptions
        {
            PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
        });

        var sb = new StringBuilder();
        if (id is not null) sb.AppendLine($"id: {id}");
        sb.AppendLine($"event: {eventName}");
        sb.AppendLine($"data: {json}");
        sb.AppendLine(); // Empty line terminates event

        await context.Response.WriteAsync(sb.ToString(), ct);
        await context.Response.Body.FlushAsync(ct);
    }
}
```

---

## Step 1387: WebSocket Raw API

```csharp
// Raw WebSocket for custom protocols (lower overhead than SignalR)
app.MapGet("/ws/trades", async (HttpContext context) =>
{
    if (!context.WebSockets.IsWebSocketRequest)
    {
        context.Response.StatusCode = 400;
        return;
    }

    using var webSocket = await context.WebSockets.AcceptWebSocketAsync();
    var logger = context.RequestServices.GetRequiredService<ILogger<Program>>();

    logger.LogInformation("WebSocket connected from {IP}", context.Connection.RemoteIpAddress);

    await HandleWebSocketAsync(webSocket, context.RequestAborted);
});

private static async Task HandleWebSocketAsync(WebSocket ws, CancellationToken ct)
{
    var buffer = new byte[4096];

    while (ws.State == WebSocketState.Open && !ct.IsCancellationRequested)
    {
        WebSocketReceiveResult result;
        var received = new MemoryStream();

        do
        {
            result = await ws.ReceiveAsync(buffer.AsMemory(), ct);
            if (result.MessageType == WebSocketMessageType.Close)
            {
                await ws.CloseAsync(WebSocketCloseStatus.NormalClosure, "Closing", ct);
                return;
            }
            await received.WriteAsync(buffer.AsMemory(0, result.Count), ct);
        } while (!result.EndOfMessage);

        var messageBytes = received.ToArray();
        var message = Encoding.UTF8.GetString(messageBytes);

        // Process message and send response
        var response = ProcessTrade(message);
        var responseBytes = JsonSerializer.SerializeToUtf8Bytes(response);

        await ws.SendAsync(
            responseBytes.AsMemory(),
            WebSocketMessageType.Text,
            endOfMessage: true,
            ct);
    }
}

static object ProcessTrade(string json) => new { processed = true, timestamp = DateTimeOffset.UtcNow };

// WebSocket middleware for all /ws/* paths
public class WebSocketMiddleware(RequestDelegate next)
{
    public async Task InvokeAsync(HttpContext context)
    {
        if (!context.Request.Path.StartsWithSegments("/ws"))
        {
            await next(context);
            return;
        }

        if (!context.WebSockets.IsWebSocketRequest)
        {
            context.Response.StatusCode = StatusCodes.Status400BadRequest;
            return;
        }

        var socket = await context.WebSockets.AcceptWebSocketAsync();
        using var cts = CancellationTokenSource.CreateLinkedTokenSource(
            context.RequestAborted);

        await HandleWebSocketAsync(socket, cts.Token);
    }
}
```

---

## Step 1388: Real-Time Dashboard

```csharp
// Dashboard hub — streaming aggregated metrics in real time
public class DashboardHub(
    IMetricsService metrics,
    IOrderRepository orders,
    ILogger<DashboardHub> logger) : Hub
{
    public async Task SubscribeDashboard()
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, "dashboard");
        logger.LogInformation("Dashboard client subscribed: {ConnId}", Context.ConnectionId);
    }

    // Streaming: server pushes updates to client as they arrive
    public async IAsyncEnumerable<DashboardSnapshot> StreamMetrics(
        [EnumeratorCancellation] CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            yield return new DashboardSnapshot(
                Timestamp: DateTimeOffset.UtcNow,
                OrdersToday: await orders.CountTodayAsync(ct),
                RevenueToday: await orders.RevenueTodayAsync(ct),
                ActiveUsers: await metrics.GetActiveUsersAsync(ct),
                OrdersPerMinute: await metrics.GetOrderRateAsync(ct),
                ErrorRate: await metrics.GetErrorRateAsync(ct)
            );

            await Task.Delay(TimeSpan.FromSeconds(5), ct);
        }
    }
}

// Background service to push periodic updates to dashboard
public class DashboardBroadcaster(
    IHubContext<DashboardHub> hubContext,
    IMetricsService metrics,
    IOrderRepository orders,
    ILogger<DashboardBroadcaster> logger) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromSeconds(5));

        while (await timer.WaitForNextTickAsync(stoppingToken))
        {
            try
            {
                var snapshot = new DashboardSnapshot(
                    Timestamp: DateTimeOffset.UtcNow,
                    OrdersToday: await orders.CountTodayAsync(stoppingToken),
                    RevenueToday: await orders.RevenueTodayAsync(stoppingToken),
                    ActiveUsers: await metrics.GetActiveUsersAsync(stoppingToken),
                    OrdersPerMinute: await metrics.GetOrderRateAsync(stoppingToken),
                    ErrorRate: await metrics.GetErrorRateAsync(stoppingToken)
                );

                await hubContext.Clients
                    .Group("dashboard")
                    .SendAsync("dashboardUpdate", snapshot, stoppingToken);
            }
            catch (OperationCanceledException) { break; }
            catch (Exception ex)
            {
                logger.LogError(ex, "Failed to broadcast dashboard update");
            }
        }
    }
}

public record DashboardSnapshot(
    DateTimeOffset Timestamp,
    int OrdersToday,
    decimal RevenueToday,
    int ActiveUsers,
    double OrdersPerMinute,
    double ErrorRate
);
```

---

## Step 1389: Presence System

```csharp
// Track which users are currently online
public class PresenceService(IConnectionMultiplexer redis, ILogger<PresenceService> logger)
{
    private readonly IDatabase _db = redis.GetDatabase();
    private const string OnlineUsersKey = "presence:online";

    public async Task UserConnectedAsync(string userId, string connectionId, CancellationToken ct)
    {
        await _db.SetAddAsync(OnlineUsersKey, userId);
        await _db.StringSetAsync($"presence:{userId}:conn", connectionId,
            TimeSpan.FromHours(24));
        logger.LogDebug("User {UserId} online", userId);
    }

    public async Task UserDisconnectedAsync(string userId, CancellationToken ct)
    {
        await _db.SetRemoveAsync(OnlineUsersKey, userId);
        await _db.KeyDeleteAsync($"presence:{userId}:conn");
        logger.LogDebug("User {UserId} offline", userId);
    }

    public async Task<bool> IsOnlineAsync(string userId)
        => await _db.SetContainsAsync(OnlineUsersKey, userId);

    public async Task<List<string>> GetOnlineUsersAsync()
    {
        var members = await _db.SetMembersAsync(OnlineUsersKey);
        return members.Select(m => m.ToString()).ToList();
    }

    public async Task<int> GetOnlineCountAsync()
        => (int)await _db.SetLengthAsync(OnlineUsersKey);
}

// Presence Hub
public class PresenceHub(PresenceService presence) : Hub
{
    public override async Task OnConnectedAsync()
    {
        var userId = Context.User?.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        if (userId is not null)
        {
            await presence.UserConnectedAsync(userId, Context.ConnectionId, Context.ConnectionAborted);

            // Notify friends they're online
            await Clients.All.SendAsync("userOnline", new { userId, timestamp = DateTimeOffset.UtcNow });
        }
        await base.OnConnectedAsync();
    }

    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        var userId = Context.User?.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        if (userId is not null)
        {
            await presence.UserDisconnectedAsync(userId, Context.ConnectionAborted);
            await Clients.All.SendAsync("userOffline", new { userId, timestamp = DateTimeOffset.UtcNow });
        }
        await base.OnDisconnectedAsync(exception);
    }

    public async Task<int> GetOnlineCount()
        => await presence.GetOnlineCountAsync();
}
```

---

## Step 1390: Chat Application

```csharp
public class ChatHub(
    IChatRepository chatRepo,
    IPresenceService presence,
    ILogger<ChatHub> logger) : Hub<IChatClient>
{
    public async Task JoinRoom(string roomId)
    {
        var userId = GetUserId();
        await Groups.AddToGroupAsync(Context.ConnectionId, $"room:{roomId}");

        // Load recent history
        var history = await chatRepo.GetRecentMessagesAsync(roomId, limit: 50, Context.ConnectionAborted);
        await Clients.Caller.ChatHistory(history);

        // Notify room members
        await Clients.Group($"room:{roomId}").UserJoined(new UserJoinedMessage(userId, roomId, DateTimeOffset.UtcNow));
        logger.LogInformation("User {UserId} joined room {RoomId}", userId, roomId);
    }

    public async Task LeaveRoom(string roomId)
    {
        var userId = GetUserId();
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, $"room:{roomId}");
        await Clients.Group($"room:{roomId}").UserLeft(new UserLeftMessage(userId, roomId, DateTimeOffset.UtcNow));
    }

    public async Task<string> SendMessage(string roomId, string content)
    {
        var userId = GetUserId();

        if (string.IsNullOrWhiteSpace(content) || content.Length > 4000)
            throw new HubException("Invalid message content");

        // Sanitize content
        var sanitized = HtmlEncoder.Default.Encode(content.Trim());

        var message = new ChatMessage
        {
            Id = Guid.NewGuid().ToString(),
            RoomId = roomId,
            UserId = userId,
            Content = sanitized,
            SentAt = DateTimeOffset.UtcNow,
        };

        await chatRepo.SaveMessageAsync(message, Context.ConnectionAborted);

        // Broadcast to room
        await Clients.Group($"room:{roomId}").MessageReceived(message);

        return message.Id;
    }

    // Typing indicator (no persistence)
    public async Task StartTyping(string roomId)
    {
        var userId = GetUserId();
        await Clients.OthersInGroup($"room:{roomId}")
            .UserTyping(new TypingIndicator(userId, roomId, true));
    }

    public async Task StopTyping(string roomId)
    {
        var userId = GetUserId();
        await Clients.OthersInGroup($"room:{roomId}")
            .UserTyping(new TypingIndicator(userId, roomId, false));
    }

    // Message reactions
    public async Task AddReaction(string messageId, string reaction)
    {
        var userId = GetUserId();
        await chatRepo.AddReactionAsync(messageId, userId, reaction, Context.ConnectionAborted);

        var message = await chatRepo.GetMessageAsync(messageId, Context.ConnectionAborted);
        if (message is not null)
            await Clients.Group($"room:{message.RoomId}")
                .ReactionAdded(new ReactionUpdate(messageId, userId, reaction));
    }

    private string GetUserId()
        => Context.User?.FindFirst(ClaimTypes.NameIdentifier)?.Value
            ?? throw new HubException("Not authenticated");
}

public interface IChatClient
{
    Task MessageReceived(ChatMessage message);
    Task ChatHistory(List<ChatMessage> messages);
    Task UserJoined(UserJoinedMessage message);
    Task UserLeft(UserLeftMessage message);
    Task UserTyping(TypingIndicator indicator);
    Task ReactionAdded(ReactionUpdate update);
}

public record ChatMessage
{
    public string Id { get; init; } = string.Empty;
    public string RoomId { get; init; } = string.Empty;
    public string UserId { get; init; } = string.Empty;
    public string Content { get; init; } = string.Empty;
    public DateTimeOffset SentAt { get; init; }
}

public record UserJoinedMessage(string UserId, string RoomId, DateTimeOffset Timestamp);
public record UserLeftMessage(string UserId, string RoomId, DateTimeOffset Timestamp);
public record TypingIndicator(string UserId, string RoomId, bool IsTyping);
public record ReactionUpdate(string MessageId, string UserId, string Reaction);
```

---

## Step 1391: Live Order Tracking

```csharp
// Real-time order tracking with GPS-like updates
public class OrderTrackingHub(
    IDeliveryService delivery,
    ILogger<OrderTrackingHub> logger) : Hub<ITrackingClient>
{
    public async Task TrackOrder(Guid orderId)
    {
        // Validate access
        var userId = GetUserId();
        var order = await delivery.GetOrderAsync(orderId, Context.ConnectionAborted);
        if (order?.CustomerId.ToString() != userId)
            throw new HubException("Access denied");

        await Groups.AddToGroupAsync(Context.ConnectionId, $"tracking:{orderId}");
        logger.LogInformation("User {UserId} tracking order {OrderId}", userId, orderId);

        // Send current location immediately
        var location = await delivery.GetCurrentLocationAsync(orderId, Context.ConnectionAborted);
        if (location is not null)
            await Clients.Caller.LocationUpdated(location);
    }

    public async Task StopTracking(Guid orderId)
    {
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, $"tracking:{orderId}");
    }

    private string GetUserId() =>
        Context.User?.FindFirst(ClaimTypes.NameIdentifier)?.Value
            ?? throw new HubException("Not authenticated");
}

public interface ITrackingClient
{
    Task LocationUpdated(LocationUpdate location);
    Task DeliveryStatusChanged(DeliveryStatus status);
    Task EstimatedTimeUpdated(TimeEstimate estimate);
}

// Delivery driver pushes location updates
public class DeliveryDriverHub(
    IHubContext<OrderTrackingHub, ITrackingClient> trackingHub,
    IDeliveryService delivery) : Hub
{
    [Authorize(Roles = "Driver")]
    public async Task UpdateLocation(Guid orderId, double latitude, double longitude)
    {
        var driverId = Context.User!.FindFirst(ClaimTypes.NameIdentifier)!.Value;

        // Save to database
        var location = new LocationUpdate(orderId, latitude, longitude,
            driverId, DateTimeOffset.UtcNow);
        await delivery.SaveLocationAsync(location, Context.ConnectionAborted);

        // Push to customers tracking this order
        await trackingHub.Clients
            .Group($"tracking:{orderId}")
            .LocationUpdated(location);

        // Update ETA estimate
        var eta = await delivery.CalculateEtaAsync(orderId, latitude, longitude, Context.ConnectionAborted);
        await trackingHub.Clients
            .Group($"tracking:{orderId}")
            .EstimatedTimeUpdated(eta);
    }
}

public record LocationUpdate(
    Guid OrderId,
    double Latitude,
    double Longitude,
    string DriverId,
    DateTimeOffset Timestamp
);

public record DeliveryStatus(Guid OrderId, string Status, string? Note);
public record TimeEstimate(Guid OrderId, int MinutesRemaining, DateTimeOffset EstimatedArrival);
```

---

## Step 1392: Scale-Out with Redis Backplane

```csharp
// Redis backplane ensures messages reach all instances
// Even when clients connect to different server instances

// All Server instances publish to Redis channel
// Redis delivers to all subscribed instances
// Each instance delivers to its local connections

// Single vs Multi-instance comparison:
//
// Single instance: Client A → Hub → Client B ✓ (direct)
//
// Multi-instance WITHOUT backplane:
//   Client A (on Server 1) → Hub (Server 1)
//   Client B (on Server 2) → never receives! ✗
//
// Multi-instance WITH Redis backplane:
//   Client A (on Server 1) → Hub (Server 1)
//   → Redis channel
//   Server 2 → Client B ✓

builder.Services.AddSignalR()
    .AddStackExchangeRedis(connectionString, options =>
    {
        options.Configuration.ChannelPrefix = RedisChannel.Literal("myapp:signalr");
        options.Configuration.ConnectTimeout = 5000;
        options.Configuration.SyncTimeout = 3000;
        options.Configuration.ReconnectRetryPolicy = new LinearRetry(500);
    });

// Azure SignalR Service (fully managed scale-out, 100K+ concurrent)
builder.Services.AddSignalR()
    .AddAzureSignalR(options =>
    {
        options.ConnectionString = builder.Configuration["AzureSignalR:ConnectionString"];
        options.ServerStickyMode = ServerStickyMode.Required; // for stateful scenarios
    });

// Connection tracking across instances (via Redis)
public class MultiInstanceConnectionTracker(IConnectionMultiplexer redis)
{
    private readonly IDatabase _db = redis.GetDatabase();

    public async Task TrackConnectionAsync(string userId, string connectionId, string instanceId)
    {
        var key = $"connections:{userId}";
        await _db.HashSetAsync(key, connectionId, instanceId);
        await _db.KeyExpireAsync(key, TimeSpan.FromHours(24));
    }

    public async Task RemoveConnectionAsync(string userId, string connectionId)
    {
        await _db.HashDeleteAsync($"connections:{userId}", connectionId);
    }

    public async Task<List<string>> GetConnectionsForUserAsync(string userId)
    {
        var fields = await _db.HashGetAllAsync($"connections:{userId}");
        return fields.Select(f => f.Name.ToString()).ToList();
    }

    public async Task<bool> IsUserOnlineAsync(string userId)
    {
        var count = await _db.HashLengthAsync($"connections:{userId}");
        return count > 0;
    }
}
```

---

## Step 1393: Rate Limiting & Security for SignalR

```csharp
// Hub filters — cross-cutting concerns for hubs
public class RateLimitHubFilter(IConnectionMultiplexer redis) : IHubFilter
{
    private readonly IDatabase _db = redis.GetDatabase();
    private const int MaxMessagesPerMinute = 60;

    public async ValueTask<object?> InvokeMethodAsync(
        HubInvocationContext invocationContext,
        Func<HubInvocationContext, ValueTask<object?>> next)
    {
        var userId = invocationContext.Context.User?
            .FindFirst(ClaimTypes.NameIdentifier)?.Value ?? "anonymous";

        var key = $"hub:ratelimit:{userId}:{invocationContext.HubMethodName}";
        var count = await _db.StringIncrementAsync(key);
        if (count == 1)
            await _db.KeyExpireAsync(key, TimeSpan.FromMinutes(1));

        if (count > MaxMessagesPerMinute)
        {
            throw new HubException($"Rate limit exceeded for {invocationContext.HubMethodName}. " +
                $"Max {MaxMessagesPerMinute} calls/minute.");
        }

        return await next(invocationContext);
    }
}

// Logging filter
public class LoggingHubFilter(ILogger<LoggingHubFilter> logger) : IHubFilter
{
    public async ValueTask<object?> InvokeMethodAsync(
        HubInvocationContext invocationContext,
        Func<HubInvocationContext, ValueTask<object?>> next)
    {
        var userId = invocationContext.Context.User?.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        using var scope = logger.BeginScope(new Dictionary<string, object?>
        {
            ["HubMethod"] = invocationContext.HubMethodName,
            ["UserId"] = userId,
            ["ConnectionId"] = invocationContext.Context.ConnectionId,
        });

        logger.LogDebug("Hub method invoked: {Method}", invocationContext.HubMethodName);
        var sw = Stopwatch.StartNew();

        try
        {
            var result = await next(invocationContext);
            logger.LogDebug("Hub method {Method} completed in {Ms}ms",
                invocationContext.HubMethodName, sw.ElapsedMilliseconds);
            return result;
        }
        catch (Exception ex)
        {
            logger.LogWarning(ex, "Hub method {Method} failed after {Ms}ms",
                invocationContext.HubMethodName, sw.ElapsedMilliseconds);
            throw;
        }
    }
}

// Registration
builder.Services.AddSignalR(options =>
{
    options.AddFilter<RateLimitHubFilter>();
    options.AddFilter<LoggingHubFilter>();
});
```

---

## Step 1394: Testing SignalR Hubs

```csharp
// Integration test with SignalR test client
public class OrderHubTests : IAsyncLifetime
{
    private WebApplicationFactory<Program> _factory = null!;
    private HubConnection _connection = null!;
    private readonly List<OrderStatusUpdate> _received = [];

    public async Task InitializeAsync()
    {
        _factory = new WebApplicationFactory<Program>()
            .WithWebHostBuilder(builder =>
            {
                builder.UseUrls("http://localhost:0");
                builder.ConfigureServices(services =>
                {
                    // Replace Redis with in-memory for testing
                    services.AddSignalR(); // No Redis backplane
                });
            });

        var server = _factory.Server;
        var baseUrl = server.BaseAddress;

        _connection = new HubConnectionBuilder()
            .WithUrl($"http://localhost/hubs/orders", options =>
            {
                options.HttpMessageHandlerFactory = _ => server.CreateHandler();
            })
            .Build();

        _connection.On<OrderStatusUpdate>("orderStatusUpdated", update =>
            _received.Add(update));

        await _connection.StartAsync();
    }

    [Fact]
    public async Task SendMessage_ClientReceivesNotification()
    {
        // Arrange: subscribe to order updates via hub
        await _connection.InvokeAsync("JoinCustomerGroup", Guid.NewGuid().ToString());

        // Act: simulate order status change via service
        using var scope = _factory.Services.CreateScope();
        var notificationService = scope.ServiceProvider
            .GetRequiredService<OrderNotificationService>();
        var order = new Order { Id = Guid.NewGuid(), Status = OrderStatus.Shipped };
        await notificationService.NotifyOrderStatusChangeAsync(order, "Pending", CancellationToken.None);

        // Assert: message received within 2 seconds
        await Task.Delay(200); // Allow message to propagate
        _received.Should().ContainSingle(u => u.OrderId == order.Id);
    }

    [Fact]
    public async Task InvokeMethod_ReturnsExpectedResult()
    {
        var orderId = Guid.NewGuid();
        // Seed test data...

        var result = await _connection.InvokeAsync<OrderStatusResponse>(
            "GetOrderStatus", orderId);

        result.Should().NotBeNull();
        result.OrderId.Should().Be(orderId);
    }

    public async Task DisposeAsync()
    {
        await _connection.DisposeAsync();
        await _factory.DisposeAsync();
    }
}
```

---

## Step 1395: gRPC Streaming for Real-Time

```csharp
// gRPC bi-directional streaming (alternative to WebSocket)

// Proto definition
/*
service LiveUpdates {
  rpc Subscribe (SubscribeRequest) returns (stream UpdateResponse);
  rpc BidirectionalChat (stream ChatMessage) returns (stream ChatMessage);
}
*/

// Server-side streaming: server pushes updates to client
public class LiveUpdatesService : LiveUpdates.LiveUpdatesBase
{
    public override async Task Subscribe(
        SubscribeRequest request,
        IServerStreamWriter<UpdateResponse> responseStream,
        ServerCallContext context)
    {
        // Subscribe to real-time event channel
        await foreach (var update in GetUpdateStreamAsync(request.TopicId, context.CancellationToken))
        {
            await responseStream.WriteAsync(new UpdateResponse
            {
                EventId = update.Id.ToString(),
                Data = update.Payload,
                Timestamp = Google.Protobuf.WellKnownTypes.Timestamp.FromDateTimeOffset(update.OccurredAt),
            });
        }
    }

    private async IAsyncEnumerable<DomainUpdate> GetUpdateStreamAsync(
        string topicId,
        [EnumeratorCancellation] CancellationToken ct)
    {
        var channel = Channel.CreateUnbounded<DomainUpdate>();
        // Subscribe to event bus and write to channel...

        await foreach (var update in channel.Reader.ReadAllAsync(ct))
            yield return update;
    }

    // Bidirectional streaming
    public override async Task BidirectionalChat(
        IAsyncStreamReader<ChatMessage> requestStream,
        IServerStreamWriter<ChatMessage> responseStream,
        ServerCallContext context)
    {
        var userId = context.RequestHeaders.GetValue("user-id");
        await foreach (var message in requestStream.ReadAllAsync(context.CancellationToken))
        {
            // Echo message to all participants (simplified)
            var reply = new ChatMessage
            {
                UserId = "server",
                Content = $"Echo: {message.Content}",
                Timestamp = Google.Protobuf.WellKnownTypes.Timestamp.FromDateTimeOffset(DateTimeOffset.UtcNow),
            };
            await responseStream.WriteAsync(reply);
        }
    }
}
```

---

## Step 1396: Real-Time Notifications with Push API

```csharp
// Web Push Notifications (browser push even when tab closed)
using WebPush;

public class PushNotificationService(
    IConfiguration config,
    ISubscriptionRepository subscriptions,
    ILogger<PushNotificationService> logger)
{
    private readonly VapidDetails _vapidDetails = new(
        config["WebPush:Subject"]!,
        config["WebPush:PublicKey"]!,
        config["WebPush:PrivateKey"]!);

    public async Task SendToUserAsync(string userId, PushPayload payload, CancellationToken ct)
    {
        var userSubscriptions = await subscriptions.GetByUserIdAsync(userId, ct);

        var tasks = userSubscriptions.Select(async sub =>
        {
            try
            {
                var client = new WebPushClient();
                await client.SendNotificationAsync(
                    new PushSubscription(sub.Endpoint, sub.P256dh, sub.Auth),
                    JsonSerializer.Serialize(payload),
                    _vapidDetails);
            }
            catch (WebPushException ex) when (ex.StatusCode == HttpStatusCode.Gone)
            {
                // Subscription expired — remove it
                await subscriptions.DeleteAsync(sub.Id, ct);
                logger.LogInformation("Removed expired subscription {SubId} for user {UserId}",
                    sub.Id, userId);
            }
            catch (Exception ex)
            {
                logger.LogError(ex, "Failed to send push to subscription {SubId}", sub.Id);
            }
        });

        await Task.WhenAll(tasks);
    }
}

// API endpoint to save push subscription from browser
app.MapPost("/api/push/subscribe", async (
    PushSubscriptionDto dto,
    ISubscriptionRepository repo,
    ClaimsPrincipal user,
    CancellationToken ct) =>
{
    var userId = user.FindFirst(ClaimTypes.NameIdentifier)!.Value;
    await repo.SaveAsync(new PushSubscription
    {
        UserId = userId,
        Endpoint = dto.Endpoint,
        P256dh = dto.Keys.P256dh,
        Auth = dto.Keys.Auth,
        CreatedAt = DateTimeOffset.UtcNow,
    }, ct);
    return Results.Ok();
})
.RequireAuthorization();

public record PushPayload(string Title, string Body, string? Icon = null, string? Url = null);
public record PushSubscriptionDto(string Endpoint, PushKeys Keys);
public record PushKeys(string P256dh, string Auth);
```

---

## Step 1397: SignalR vs SSE vs WebSocket Decision Guide

```
REAL-TIME TECHNOLOGY SELECTION GUIDE
══════════════════════════════════════

                 SignalR         SSE             WebSocket       gRPC Streaming
─────────────────────────────────────────────────────────────────────────────
Direction        Bi-directional  Server→Client   Bi-directional  Bi-directional
Protocol         WS/SSE/LP auto  HTTP            WS              HTTP/2
.NET support     Excellent       Built-in        Built-in        Excellent
Browser support  Via JS SDK      Native          Native          Via grpc-web
Binary support   Yes (MsgPack)   No (text only)  Yes             Yes (Protobuf)
Groups/Users     Built-in        Manual          Manual          Manual
Scale-out        Redis backplane Manual          Manual          Manual
Overhead         Medium (lib)    Low             Low             Low (proto)
Use case         Apps/Games/Chat News feeds      Trading/Games   Microservices

CHOOSE:
├── SignalR    → Rich apps, groups, auth, scale-out needed
├── SSE        → Simple server pushes, no client→server needed
├── WebSocket  → Custom protocol, low-level control, binary data
├── gRPC       → Microservice streaming, strongly-typed, high perf
└── Polling    → Simple, rare updates, no connection management needed
```

---

## Step 1398: Complete Real-Time Order System

```csharp
// Complete system: Order created → Kafka → SignalR → Browser

// 1. API: Create order (writes to DB + Kafka)
app.MapPost("/api/orders", async (
    CreateOrderRequest request,
    IOrderService orderService,
    CancellationToken ct) =>
{
    var order = await orderService.CreateAsync(request, ct);
    return TypedResults.Created($"/api/orders/{order.Id}", order);
});

// 2. OrderService: creates order and publishes event
public class OrderService(
    IOrderRepository repo,
    IKafkaProducer<string, string> producer,
    ILogger<OrderService> logger)
{
    public async Task<Order> CreateAsync(CreateOrderRequest request, CancellationToken ct)
    {
        var order = /* create order */ null!;
        await repo.AddAsync(order, ct);

        // Publish to Kafka (outbox for reliability in production)
        var evt = new OrderCreatedEvent(order.Id, order.CustomerId, order.TotalAmount, order.CreatedAt);
        await producer.ProduceAsync("order-events",
            order.Id.ToString(),
            JsonSerializer.Serialize(evt), ct);

        return order;
    }
}

// 3. Kafka consumer → SignalR notification
public class OrderEventConsumerWorker(
    IServiceProvider services,
    ILogger<OrderEventConsumerWorker> logger) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        using var consumer = BuildConsumer();
        consumer.Subscribe("order-events");

        while (!ct.IsCancellationRequested)
        {
            var result = consumer.Consume(TimeSpan.FromMilliseconds(100));
            if (result?.Message is null) continue;

            using var scope = services.CreateScope();
            var hubContext = scope.ServiceProvider
                .GetRequiredService<IHubContext<OrderHub, IOrderClient>>();

            var evt = JsonSerializer.Deserialize<OrderCreatedEvent>(result.Message.Value);
            if (evt is not null)
            {
                await hubContext.Clients
                    .User(evt.CustomerId.ToString())
                    .OrderCreated(new OrderCreatedNotification(
                        evt.OrderId, evt.TotalAmount, evt.CreatedAt));
            }

            consumer.Commit(result);
        }
    }

    private static IConsumer<string, string> BuildConsumer() =>
        new ConsumerBuilder<string, string>(new ConsumerConfig
        {
            BootstrapServers = "kafka:9092",
            GroupId = "signalr-relay",
            AutoOffsetReset = AutoOffsetReset.Latest,
        }).Build();
}

// 4. Browser receives real-time notification without polling
// connection.on("orderCreated", notification => {
//   showToast(`Order ${notification.orderId} confirmed! Total: ${notification.totalAmount}`);
//   updateOrderList(notification);
// });
```

---

## Step 1399: Monitoring Real-Time Connections

```csharp
// Track SignalR connection metrics
public class SignalRMetrics
{
    private static readonly Meter _meter = new("SignalR.Metrics", "1.0.0");

    public static readonly UpDownCounter<int> ActiveConnections =
        _meter.CreateUpDownCounter<int>("signalr.connections.active", "connections");

    public static readonly Counter<long> MessagesReceived =
        _meter.CreateCounter<long>("signalr.messages.received", "messages");

    public static readonly Counter<long> MessagesSent =
        _meter.CreateCounter<long>("signalr.messages.sent", "messages");

    public static readonly Histogram<double> MessageLatency =
        _meter.CreateHistogram<double>("signalr.message.latency", "ms");
}

public class MetricsHubFilter : IHubFilter
{
    public async Task OnConnectedAsync(
        HubLifetimeContext context, Func<HubLifetimeContext, Task> next)
    {
        SignalRMetrics.ActiveConnections.Add(1);
        await next(context);
    }

    public async Task OnDisconnectedAsync(
        HubLifetimeContext context, Exception? exception,
        Func<HubLifetimeContext, Exception?, Task> next)
    {
        SignalRMetrics.ActiveConnections.Add(-1);
        await next(context, exception);
    }

    public async ValueTask<object?> InvokeMethodAsync(
        HubInvocationContext context,
        Func<HubInvocationContext, ValueTask<object?>> next)
    {
        var sw = Stopwatch.StartNew();
        SignalRMetrics.MessagesReceived.Add(1);

        var result = await next(context);

        SignalRMetrics.MessageLatency.Record(sw.Elapsed.TotalMilliseconds,
            new KeyValuePair<string, object?>("hub.method", context.HubMethodName));

        return result;
    }
}
```

---

## Step 1400: Summary — Real-Time Applications Checklist

```
REAL-TIME APPLICATIONS CHECKLIST
══════════════════════════════════

SIGNALR
├── Strongly-typed hubs (Hub<IClient>)
├── Groups for targeted broadcasts
├── IHubContext for server-side notifications
├── Redis backplane for multi-instance scale
├── Hub filters for cross-cutting concerns
├── Automatic reconnect with exponential backoff
└── Connection lifecycle (OnConnected/OnDisconnected)

SECURITY
├── Authentication via JWT on WebSocket upgrade
├── Authorization policy on hub endpoints
├── Rate limiting via hub filter
├── Input validation inside hub methods
└── CloseOnAuthenticationExpiration = true

PATTERNS
├── Dashboard broadcasting with PeriodicTimer
├── Presence system (online/offline tracking)
├── Chat (rooms, typing indicators, reactions)
├── Live order tracking (GPS updates)
├── Push notifications (Web Push API)
└── IAsyncEnumerable streaming from hub

SCALE-OUT
├── Redis backplane (self-hosted)
├── Azure SignalR Service (managed, 100K+)
├── Connection tracking across instances
└── Sticky sessions when stateful (Azure ASR)

MONITORING
├── Active connections gauge
├── Messages received/sent counters
├── Message latency histogram
└── Reconnection rate tracking

ALTERNATIVES
├── SSE for simple one-way streaming
├── Raw WebSocket for custom binary protocols
├── gRPC streaming for microservice comms
└── Web Push for background notifications
```

---

## สรุป Part 49

Part นี้ครอบคลุม Real-Time Applications & SignalR อย่างครบถ้วน:

**Steps 1381-1400:**
- **SignalR Setup** — strongly-typed hubs, Redis backplane, JWT auth
- **Hub Context** — IHubContext for server-initiated messages from background services
- **TypeScript/JavaScript Client** — HubConnectionBuilder, auto-reconnect, exponential backoff
- **.NET Client** — microservice-to-microservice real-time communication
- **Server-Sent Events** — simple server→client streaming with retry hints
- **Raw WebSocket** — custom binary protocols, WebSocket middleware
- **Real-Time Dashboard** — PeriodicTimer broadcasts, IAsyncEnumerable streaming
- **Presence System** — Redis-backed online/offline tracking
- **Chat Application** — rooms, typing indicators, message reactions, history
- **Live Order Tracking** — GPS location updates, driver hub, customer tracking hub
- **Redis Scale-Out** — backplane, Azure SignalR Service, multi-instance connection tracking
- **Security** — rate limiting hub filter, logging filter, CloseOnAuthenticationExpiration
- **Testing** — WebApplicationFactory with test HTTP handler for hub connections
- **gRPC Streaming** — server-side streaming, bidirectional chat over gRPC
- **Web Push API** — VAPID keys, browser push even when tab closed
- **Decision Guide** — SignalR vs SSE vs WebSocket vs gRPC comparison
- **Complete Pipeline** — Order API → Kafka → Consumer → SignalR → Browser
- **Monitoring** — custom metrics for connections, messages, latency

เนื้อหาทั้งหมดเป็น Production-ready code ที่ใช้ได้จริงใน .NET 9.0
