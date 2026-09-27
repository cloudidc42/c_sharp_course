# Part 27: SignalR - Real-Time Communication

## Steps 731-770: การสร้างแอปพลิเคชัน Real-Time ด้วย SignalR

---

## Step 731: SignalR คืออะไร

```bash
# SignalR คือ library ที่ทำให้ server สามารถ push data ไปหา client ได้แบบ real-time
# Transport protocols (ลำดับ fallback):
# 1. WebSockets (fastest, full-duplex)
# 2. Server-Sent Events (SSE) - server to client only
# 3. Long Polling - fallback

# สร้างโปรเจค
dotnet new web -n SignalRDemo
cd SignalRDemo

# SignalR เป็น built-in ใน ASP.NET Core ไม่ต้องติดตั้ง package เพิ่ม
# สำหรับ Client JS:
# dotnet add package Microsoft.AspNetCore.SignalR.Client (สำหรับ .NET client)

# ติดตั้ง JS client
# npm install @microsoft/signalr
# หรือใช้ CDN: https://unpkg.com/@microsoft/signalr/dist/browser/signalr.min.js
```

```csharp
// Program.cs - SignalR Setup
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddSignalR(options =>
{
    options.EnableDetailedErrors = builder.Environment.IsDevelopment();
    options.HandshakeTimeout = TimeSpan.FromSeconds(15);
    options.KeepAliveInterval = TimeSpan.FromSeconds(15);
    options.ClientTimeoutInterval = TimeSpan.FromSeconds(60);
    options.MaximumReceiveMessageSize = 32 * 1024; // 32KB
});

builder.Services.AddCors(options =>
    options.AddDefaultPolicy(policy =>
        policy.WithOrigins("http://localhost:3000")
              .AllowAnyHeader()
              .AllowAnyMethod()
              .AllowCredentials()));

var app = builder.Build();
app.UseCors();
app.MapHub<ChatHub>("/hubs/chat");
app.MapHub<NotificationHub>("/hubs/notifications");
app.Run();
```

---

## Step 732: Hub พื้นฐาน

```csharp
// Hubs/ChatHub.cs
using Microsoft.AspNetCore.SignalR;

namespace SignalRDemo.Hubs;

public class ChatHub : Hub
{
    private readonly ILogger<ChatHub> _logger;

    public ChatHub(ILogger<ChatHub> logger)
    {
        _logger = logger;
    }

    // Client calls this method on the server
    public async Task SendMessage(string user, string message)
    {
        _logger.LogInformation("{User} sent: {Message}", user, message);
        
        // Broadcast to ALL clients including sender
        await Clients.All.SendAsync("ReceiveMessage", new
        {
            User = user,
            Message = message,
            Timestamp = DateTime.UtcNow
        });
    }

    // Send to specific client
    public async Task SendPrivateMessage(string targetConnectionId, string message)
    {
        await Clients.Client(targetConnectionId)
            .SendAsync("ReceivePrivateMessage", Context.ConnectionId, message);
    }

    // Send to all except sender
    public async Task Broadcast(string message)
    {
        await Clients.Others.SendAsync("ReceiveMessage", new
        {
            User = "System",
            Message = message,
            Timestamp = DateTime.UtcNow
        });
    }

    // Called when client connects
    public override async Task OnConnectedAsync()
    {
        _logger.LogInformation("Client connected: {ConnectionId}", Context.ConnectionId);
        
        await Clients.Others.SendAsync("UserConnected", Context.ConnectionId);
        await base.OnConnectedAsync();
    }

    // Called when client disconnects
    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        _logger.LogInformation("Client disconnected: {ConnectionId}", Context.ConnectionId);
        
        await Clients.Others.SendAsync("UserDisconnected", Context.ConnectionId);
        await base.OnDisconnectedAsync(exception);
    }
}
```

---

## Step 733: Groups และ Rooms

```csharp
// Hubs/ChatRoomHub.cs
public class ChatRoomHub : Hub
{
    private static readonly Dictionary<string, HashSet<string>> _rooms = new();
    private static readonly Dictionary<string, string> _userNames = new();
    
    private readonly ILogger<ChatRoomHub> _logger;

    public ChatRoomHub(ILogger<ChatRoomHub> logger)
    {
        _logger = logger;
    }

    public async Task JoinRoom(string roomName, string userName)
    {
        // Leave previous room if any
        var currentRoom = GetCurrentRoom();
        if (currentRoom is not null)
            await LeaveRoom(currentRoom);

        // Join new room
        await Groups.AddToGroupAsync(Context.ConnectionId, roomName);
        
        lock (_rooms)
        {
            if (!_rooms.ContainsKey(roomName))
                _rooms[roomName] = new HashSet<string>();
            _rooms[roomName].Add(Context.ConnectionId);
            _userNames[Context.ConnectionId] = userName;
        }

        // Notify room members
        await Clients.Group(roomName).SendAsync("UserJoined", new
        {
            ConnectionId = Context.ConnectionId,
            UserName = userName,
            RoomName = roomName,
            MemberCount = _rooms.GetValueOrDefault(roomName)?.Count ?? 0
        });
        
        // Send room history to the new joiner
        var history = await GetRoomHistory(roomName);
        await Clients.Caller.SendAsync("RoomHistory", history);
        
        _logger.LogInformation("{User} joined room {Room}", userName, roomName);
    }

    public async Task LeaveRoom(string roomName)
    {
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, roomName);
        
        string? userName;
        lock (_rooms)
        {
            _rooms.GetValueOrDefault(roomName)?.Remove(Context.ConnectionId);
            _userNames.TryGetValue(Context.ConnectionId, out userName);
        }

        await Clients.Group(roomName).SendAsync("UserLeft", new
        {
            ConnectionId = Context.ConnectionId,
            UserName = userName,
            RoomName = roomName
        });
    }

    public async Task SendToRoom(string roomName, string message)
    {
        var userName = _userNames.GetValueOrDefault(Context.ConnectionId, "Anonymous");
        
        var chatMessage = new ChatMessage(
            Id: Guid.NewGuid(),
            RoomName: roomName,
            UserName: userName,
            ConnectionId: Context.ConnectionId,
            Text: message,
            Timestamp: DateTime.UtcNow);
        
        await PersistMessage(chatMessage);
        
        await Clients.Group(roomName).SendAsync("ReceiveRoomMessage", chatMessage);
    }

    public async Task<IEnumerable<string>> GetRoomMembers(string roomName)
    {
        lock (_rooms)
        {
            return _rooms.GetValueOrDefault(roomName)?
                .Select(id => _userNames.GetValueOrDefault(id, "Unknown"))
                .ToList() ?? [];
        }
    }

    private string? GetCurrentRoom()
    {
        lock (_rooms)
        {
            return _rooms.FirstOrDefault(r => r.Value.Contains(Context.ConnectionId)).Key;
        }
    }

    private Task<IEnumerable<ChatMessage>> GetRoomHistory(string roomName)
        => Task.FromResult(Enumerable.Empty<ChatMessage>());
    
    private Task PersistMessage(ChatMessage message) => Task.CompletedTask;

    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        var currentRoom = GetCurrentRoom();
        if (currentRoom is not null)
            await LeaveRoom(currentRoom);
        
        lock (_rooms) { _userNames.Remove(Context.ConnectionId); }
        
        await base.OnDisconnectedAsync(exception);
    }
}

public record ChatMessage(
    Guid Id, 
    string RoomName, 
    string UserName, 
    string ConnectionId,
    string Text, 
    DateTime Timestamp);
```

---

## Step 734: Strongly Typed Hubs

```csharp
// Hubs/INotificationClient.cs - Client interface
public interface INotificationClient
{
    Task ReceiveNotification(NotificationDto notification);
    Task ReceiveSystemAlert(SystemAlertDto alert);
    Task OrderStatusChanged(OrderStatusDto status);
    Task InventoryUpdated(InventoryUpdateDto update);
}

// Hubs/NotificationHub.cs - Strongly typed hub
public class NotificationHub : Hub<INotificationClient>
{
    private readonly ILogger<NotificationHub> _logger;

    public NotificationHub(ILogger<NotificationHub> logger)
    {
        _logger = logger;
    }

    public async Task SubscribeToOrder(int orderId)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, $"order_{orderId}");
        _logger.LogInformation("Client {Id} subscribed to order {OrderId}", 
            Context.ConnectionId, orderId);
    }

    public async Task SubscribeToCategory(string category)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, $"category_{category}");
    }

    // Type-safe: no "magic strings" for method names
    public async Task SendNotification(string targetUserId, string title, string body)
    {
        await Clients.Group($"user_{targetUserId}").ReceiveNotification(new NotificationDto
        {
            Title = title,
            Body = body,
            CreatedAt = DateTime.UtcNow
        });
    }
}

// DTOs
public record NotificationDto(string Title, string Body, DateTime CreatedAt, 
    string? ActionUrl = null, string Type = "info");

public record SystemAlertDto(string Level, string Message, DateTime Timestamp);

public record OrderStatusDto(int OrderId, string Status, string? Note, DateTime UpdatedAt);

public record InventoryUpdateDto(int ProductId, string ProductName, int OldStock, int NewStock);
```

---

## Step 735: Server-side Hub Invocation

```csharp
// Services/NotificationService.cs
using Microsoft.AspNetCore.SignalR;

namespace SignalRDemo.Services;

public interface INotificationService
{
    Task NotifyUserAsync(string userId, NotificationDto notification);
    Task NotifyGroupAsync(string groupName, NotificationDto notification);
    Task BroadcastAlertAsync(SystemAlertDto alert);
    Task NotifyOrderStatusAsync(int orderId, OrderStatusDto status);
}

public class SignalRNotificationService : INotificationService
{
    private readonly IHubContext<NotificationHub, INotificationClient> _hubContext;
    private readonly ILogger<SignalRNotificationService> _logger;

    public SignalRNotificationService(
        IHubContext<NotificationHub, INotificationClient> hubContext,
        ILogger<SignalRNotificationService> logger)
    {
        _hubContext = hubContext;
        _logger = logger;
    }

    public async Task NotifyUserAsync(string userId, NotificationDto notification)
    {
        await _hubContext.Clients.Group($"user_{userId}")
            .ReceiveNotification(notification);
        _logger.LogInformation("Notified user {UserId}", userId);
    }

    public async Task NotifyGroupAsync(string groupName, NotificationDto notification)
    {
        await _hubContext.Clients.Group(groupName).ReceiveNotification(notification);
    }

    public async Task BroadcastAlertAsync(SystemAlertDto alert)
    {
        await _hubContext.Clients.All.ReceiveSystemAlert(alert);
    }

    public async Task NotifyOrderStatusAsync(int orderId, OrderStatusDto status)
    {
        await _hubContext.Clients.Group($"order_{orderId}")
            .OrderStatusChanged(status);
    }
}

// Using from a Controller
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    private readonly IOrderService _orderService;
    private readonly INotificationService _notificationService;

    public OrdersController(IOrderService orderService, INotificationService notificationService)
    {
        _orderService = orderService;
        _notificationService = notificationService;
    }

    [HttpPut("{id}/status")]
    public async Task<IActionResult> UpdateStatus(int id, UpdateOrderStatusRequest request)
    {
        var order = await _orderService.UpdateStatusAsync(id, request.Status);
        
        // Push real-time update to all subscribers
        await _notificationService.NotifyOrderStatusAsync(id, new OrderStatusDto(
            id, request.Status, request.Note, DateTime.UtcNow));
        
        return Ok(order);
    }
}
```

---

## Step 736: Authentication ใน SignalR

```csharp
// Program.cs - Authenticated SignalR
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Events = new JwtBearerEvents
        {
            // Read JWT from query string for WebSocket connections
            OnMessageReceived = context =>
            {
                var accessToken = context.Request.Query["access_token"];
                var path = context.HttpContext.Request.Path;
                
                if (!string.IsNullOrEmpty(accessToken) && 
                    path.StartsWithSegments("/hubs"))
                {
                    context.Token = accessToken;
                }
                return Task.CompletedTask;
            }
        };
    });

// Hubs/SecureHub.cs
[Authorize]
public class SecureHub : Hub<INotificationClient>
{
    public override async Task OnConnectedAsync()
    {
        var userId = Context.UserIdentifier; // From ClaimTypes.NameIdentifier
        var userName = Context.User?.Identity?.Name;
        
        // Add to user-specific group
        if (userId is not null)
            await Groups.AddToGroupAsync(Context.ConnectionId, $"user_{userId}");
        
        await base.OnConnectedAsync();
    }

    [Authorize(Roles = "Admin")]
    public async Task AdminBroadcast(string message)
    {
        await Clients.All.ReceiveNotification(new NotificationDto(
            "Admin Broadcast", message, DateTime.UtcNow));
    }
}

// UserIdProvider - map ClaimType to user ID
public class EmailUserIdProvider : IUserIdProvider
{
    public string? GetUserId(HubConnectionContext connection)
    {
        return connection.User?.FindFirst(ClaimTypes.Email)?.Value;
    }
}

// Register custom provider
builder.Services.AddSingleton<IUserIdProvider, EmailUserIdProvider>();
```

---

## Step 737: SignalR .NET Client

```csharp
// Client/NotificationClient.cs
using Microsoft.AspNetCore.SignalR.Client;

namespace SignalRDemo.Client;

public class NotificationClient : IAsyncDisposable
{
    private HubConnection? _connection;
    private readonly string _hubUrl;
    private readonly ILogger<NotificationClient> _logger;

    public event Func<NotificationDto, Task>? OnNotification;
    public event Func<string, Task>? OnOrderStatusChanged;
    
    public HubConnectionState State => _connection?.State ?? HubConnectionState.Disconnected;
    public bool IsConnected => _connection?.State == HubConnectionState.Connected;

    public NotificationClient(string hubUrl, ILogger<NotificationClient> logger)
    {
        _hubUrl = hubUrl;
        _logger = logger;
    }

    public async Task ConnectAsync(string accessToken, CancellationToken ct = default)
    {
        _connection = new HubConnectionBuilder()
            .WithUrl(_hubUrl, options =>
            {
                options.AccessTokenProvider = () => Task.FromResult<string?>(accessToken);
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
                logging.SetMinimumLevel(LogLevel.Warning);
            })
            .Build();

        _connection.On<NotificationDto>("ReceiveNotification", async notification =>
        {
            if (OnNotification is not null)
                await OnNotification(notification);
        });

        _connection.On<OrderStatusDto>("OrderStatusChanged", async status =>
        {
            if (OnOrderStatusChanged is not null)
                await OnOrderStatusChanged($"Order {status.OrderId}: {status.Status}");
        });

        _connection.Reconnecting += error =>
        {
            _logger.LogWarning("Reconnecting: {Error}", error?.Message);
            return Task.CompletedTask;
        };

        _connection.Reconnected += connectionId =>
        {
            _logger.LogInformation("Reconnected: {ConnectionId}", connectionId);
            return Task.CompletedTask;
        };

        _connection.Closed += error =>
        {
            _logger.LogError("Connection closed: {Error}", error?.Message);
            return Task.CompletedTask;
        };

        await _connection.StartAsync(ct);
        _logger.LogInformation("Connected to {Url}", _hubUrl);
    }

    public async Task SubscribeToOrderAsync(int orderId)
    {
        if (_connection is not null)
            await _connection.InvokeAsync("SubscribeToOrder", orderId);
    }

    public async Task<IEnumerable<string>> GetRoomMembersAsync(string room)
    {
        if (_connection is null) return [];
        return await _connection.InvokeAsync<IEnumerable<string>>("GetRoomMembers", room);
    }

    // Streaming from server
    public IAsyncEnumerable<StockPrice> StreamStockPricesAsync(
        string symbol, CancellationToken ct = default)
    {
        return _connection!.StreamAsync<StockPrice>("StreamStockPrices", symbol, ct);
    }

    public async ValueTask DisposeAsync()
    {
        if (_connection is not null)
        {
            await _connection.DisposeAsync();
        }
    }
}
```

---

## Step 738: Streaming ใน SignalR

```csharp
// Hubs/StreamingHub.cs
public class StreamingHub : Hub
{
    private readonly IStockService _stockService;
    private readonly ILogger<StreamingHub> _logger;

    public StreamingHub(IStockService stockService, ILogger<StreamingHub> logger)
    {
        _stockService = stockService;
        _logger = logger;
    }

    // Server -> Client streaming (IAsyncEnumerable)
    public async IAsyncEnumerable<StockPrice> StreamStockPrices(
        string symbol,
        [System.Runtime.CompilerServices.EnumeratorCancellation] CancellationToken ct)
    {
        _logger.LogInformation("Starting stock stream for {Symbol}", symbol);
        
        var rng = new Random();
        var price = 100.0m;

        while (!ct.IsCancellationRequested)
        {
            price += (decimal)(rng.NextDouble() - 0.5) * 2;
            price = Math.Max(1, price);
            
            yield return new StockPrice(symbol, Math.Round(price, 2), DateTime.UtcNow);
            
            await Task.Delay(500, ct);
        }
        
        _logger.LogInformation("Stock stream ended for {Symbol}", symbol);
    }

    // Server -> Client streaming with ChannelReader
    public ChannelReader<LogEntry> StreamLogs(
        string logLevel, CancellationToken ct)
    {
        var channel = Channel.CreateUnbounded<LogEntry>();
        
        _ = Task.Run(async () =>
        {
            try
            {
                await foreach (var entry in _stockService.GetLogStreamAsync(logLevel, ct))
                {
                    await channel.Writer.WriteAsync(entry, ct);
                }
            }
            finally
            {
                channel.Writer.Complete();
            }
        }, ct);

        return channel.Reader;
    }

    // Client -> Server streaming
    public async Task UploadData(IAsyncEnumerable<DataChunk> chunks)
    {
        await foreach (var chunk in chunks)
        {
            _logger.LogInformation("Received chunk {Index}: {Size} bytes", 
                chunk.Index, chunk.Data.Length);
            await _stockService.ProcessChunkAsync(chunk);
        }
        
        _logger.LogInformation("Upload complete");
    }
}

public record StockPrice(string Symbol, decimal Price, DateTime Timestamp);
public record LogEntry(string Level, string Message, DateTime Timestamp);
public record DataChunk(int Index, byte[] Data);
```

```javascript
// Client streaming usage (JavaScript)
const connection = new signalR.HubConnectionBuilder()
    .withUrl("/hubs/streaming")
    .build();

// Receive server stream
const stockStream = connection.stream("StreamStockPrices", "AAPL");

stockStream.subscribe({
    next: (price) => {
        console.log(`${price.symbol}: $${price.price}`);
        updateChart(price);
    },
    error: (err) => console.error(err),
    complete: () => console.log("Stream completed")
});

// Stop streaming
setTimeout(() => stockStream.dispose(), 10000); // Stop after 10s

// Send client stream to server
async function uploadFile(file) {
    const chunkSize = 8192;
    
    async function* generateChunks() {
        const reader = new FileReader();
        let offset = 0;
        let index = 0;
        
        while (offset < file.size) {
            const chunk = file.slice(offset, offset + chunkSize);
            const buffer = await chunk.arrayBuffer();
            yield { index: index++, data: Array.from(new Uint8Array(buffer)) };
            offset += chunkSize;
        }
    }
    
    await connection.send("UploadData", generateChunks());
}
```

---

## Step 739: Scale-out ด้วย Redis Backplane

```bash
# Install Redis backplane
dotnet add package Microsoft.AspNetCore.SignalR.StackExchangeRedis
```

```csharp
// Program.cs - Scale-out with Redis
builder.Services.AddSignalR()
    .AddStackExchangeRedis(options =>
    {
        options.Configuration.EndPoints.Add("localhost:6379");
        options.Configuration.Password = "your-password";
        options.Configuration.ChannelPrefix = RedisChannel.Literal("signalr_");
        options.Configuration.ReconnectRetryPolicy = new ExponentialRetry(5000, 30000);
    });

// Azure SignalR Service (cloud scale-out)
builder.Services.AddSignalR()
    .AddAzureSignalR(builder.Configuration.GetConnectionString("AzureSignalR")!);
```

---

## Step 740: Real-time Dashboard

```csharp
// Hubs/DashboardHub.cs
[Authorize]
public class DashboardHub : Hub<IDashboardClient>
{
    private readonly IDashboardService _dashboardService;
    private static readonly Timer _broadcastTimer;

    static DashboardHub()
    {
        // This approach requires IHubContext - better done via BackgroundService
    }

    public DashboardHub(IDashboardService dashboardService)
    {
        _dashboardService = dashboardService;
    }

    public async Task SubscribeToDashboard(string dashboardId)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, dashboardId);
        
        // Send current state to new subscriber
        var snapshot = await _dashboardService.GetSnapshotAsync(dashboardId);
        await Clients.Caller.DashboardUpdated(snapshot);
    }

    public async Task UnsubscribeFromDashboard(string dashboardId)
    {
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, dashboardId);
    }
}

public interface IDashboardClient
{
    Task DashboardUpdated(DashboardSnapshot snapshot);
    Task MetricUpdated(string metricName, double value, DateTime timestamp);
    Task AlertTriggered(DashboardAlert alert);
}

public record DashboardSnapshot(
    int ActiveUsers,
    int TotalOrders,
    decimal Revenue,
    double CpuUsage,
    double MemoryUsage,
    DateTime LastUpdated);

public record DashboardAlert(string Level, string Message, DateTime Timestamp);

// Services/DashboardBroadcastService.cs
public class DashboardBroadcastService : BackgroundService
{
    private readonly IHubContext<DashboardHub, IDashboardClient> _hubContext;
    private readonly IDashboardService _dashboardService;
    private readonly ILogger<DashboardBroadcastService> _logger;

    public DashboardBroadcastService(
        IHubContext<DashboardHub, IDashboardClient> hubContext,
        IDashboardService dashboardService,
        ILogger<DashboardBroadcastService> logger)
    {
        _hubContext = hubContext;
        _dashboardService = dashboardService;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            try
            {
                var snapshot = await _dashboardService.GetCurrentSnapshotAsync();
                await _hubContext.Clients.All.DashboardUpdated(snapshot);
                
                // Check for alerts
                var alerts = await _dashboardService.GetActiveAlertsAsync();
                foreach (var alert in alerts)
                {
                    await _hubContext.Clients.All.AlertTriggered(alert);
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error broadcasting dashboard");
            }

            await Task.Delay(TimeSpan.FromSeconds(5), ct);
        }
    }
}
```

---

## Step 741: Rate Limiting และ Throttling

```csharp
// Hubs/ThrottledHub.cs
public class ThrottledChatHub : Hub
{
    private static readonly Dictionary<string, DateTime> _lastMessageTimes = new();
    private static readonly TimeSpan MinMessageInterval = TimeSpan.FromMilliseconds(500);
    
    private readonly IHubContext<ThrottledChatHub> _hubContext;

    public async Task SendMessage(string message)
    {
        var connectionId = Context.ConnectionId;
        
        // Rate limiting
        if (_lastMessageTimes.TryGetValue(connectionId, out var lastTime))
        {
            if (DateTime.UtcNow - lastTime < MinMessageInterval)
            {
                await Clients.Caller.SendAsync("Error", "Too many messages. Please slow down.");
                return;
            }
        }
        
        _lastMessageTimes[connectionId] = DateTime.UtcNow;
        
        // Input validation
        if (string.IsNullOrWhiteSpace(message))
            return;
        
        if (message.Length > 500)
        {
            await Clients.Caller.SendAsync("Error", "Message too long (max 500 chars).");
            return;
        }
        
        // Sanitize
        var sanitized = System.Web.HttpUtility.HtmlEncode(message);
        
        var userName = Context.User?.Identity?.Name ?? "Anonymous";
        await Clients.All.SendAsync("ReceiveMessage", userName, sanitized, DateTime.UtcNow);
    }

    public override Task OnDisconnectedAsync(Exception? exception)
    {
        _lastMessageTimes.Remove(Context.ConnectionId);
        return base.OnDisconnectedAsync(exception);
    }
}
```

---

## Step 742: Presence System

```csharp
// Services/PresenceTracker.cs
public class PresenceTracker
{
    private static readonly Dictionary<string, HashSet<string>> _userConnections = new();
    private static readonly SemaphoreSlim _lock = new(1, 1);

    public async Task<bool> UserConnected(string username, string connectionId)
    {
        await _lock.WaitAsync();
        try
        {
            if (!_userConnections.ContainsKey(username))
            {
                _userConnections[username] = new HashSet<string>();
            }
            _userConnections[username].Add(connectionId);
            return _userConnections[username].Count == 1; // First connection?
        }
        finally
        {
            _lock.Release();
        }
    }

    public async Task<bool> UserDisconnected(string username, string connectionId)
    {
        await _lock.WaitAsync();
        try
        {
            if (!_userConnections.ContainsKey(username))
                return false;
            
            _userConnections[username].Remove(connectionId);
            if (_userConnections[username].Count == 0)
            {
                _userConnections.Remove(username);
                return true; // Last connection
            }
            return false;
        }
        finally
        {
            _lock.Release();
        }
    }

    public async Task<IEnumerable<string>> GetOnlineUsers()
    {
        await _lock.WaitAsync();
        try
        {
            return _userConnections.Keys.ToList();
        }
        finally
        {
            _lock.Release();
        }
    }

    public async Task<bool> IsUserOnline(string username)
    {
        await _lock.WaitAsync();
        try
        {
            return _userConnections.ContainsKey(username);
        }
        finally
        {
            _lock.Release();
        }
    }
}

// Hubs/PresenceHub.cs
[Authorize]
public class PresenceHub : Hub
{
    private readonly PresenceTracker _tracker;

    public PresenceHub(PresenceTracker tracker)
    {
        _tracker = tracker;
    }

    public override async Task OnConnectedAsync()
    {
        var username = Context.User?.Identity?.Name;
        if (username is null) return;

        var isFirstConnection = await _tracker.UserConnected(username, Context.ConnectionId);
        
        if (isFirstConnection)
        {
            // Only notify when user truly comes online
            await Clients.Others.SendAsync("UserOnline", username);
        }

        var onlineUsers = await _tracker.GetOnlineUsers();
        await Clients.Caller.SendAsync("GetOnlineUsers", onlineUsers);

        await base.OnConnectedAsync();
    }

    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        var username = Context.User?.Identity?.Name;
        if (username is null) return;

        var isLastConnection = await _tracker.UserDisconnected(username, Context.ConnectionId);
        
        if (isLastConnection)
        {
            // Only notify when user truly goes offline
            await Clients.Others.SendAsync("UserOffline", username);
        }

        await base.OnDisconnectedAsync(exception);
    }
}
```

---

## Step 743: Collaborative Real-time Editor

```csharp
// Hubs/DocumentHub.cs
public class DocumentHub : Hub
{
    private static readonly Dictionary<string, DocumentState> _documents = new();
    private static readonly SemaphoreSlim _documentLock = new(1, 1);
    
    public async Task JoinDocument(string documentId)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, documentId);
        
        var state = await GetOrCreateDocumentAsync(documentId);
        await Clients.Caller.SendAsync("DocumentState", state);
        
        await Clients.OthersInGroup(documentId).SendAsync("UserJoinedDocument", 
            Context.ConnectionId);
    }

    public async Task SendOperation(string documentId, DocumentOperation operation)
    {
        // Apply operation to document state
        await _documentLock.WaitAsync();
        try
        {
            if (_documents.TryGetValue(documentId, out var state))
            {
                var transformed = TransformOperation(state, operation);
                state.ApplyOperation(transformed);
                state.Version++;
                
                // Broadcast transformed operation to other clients
                await Clients.OthersInGroup(documentId).SendAsync("ReceiveOperation", transformed);
                
                // Acknowledge to sender
                await Clients.Caller.SendAsync("OperationAcknowledged", operation.Id, state.Version);
            }
        }
        finally
        {
            _documentLock.Release();
        }
    }

    public async Task SendCursorPosition(string documentId, CursorPosition position)
    {
        await Clients.OthersInGroup(documentId).SendAsync("CursorMoved", 
            Context.ConnectionId, position);
    }

    private async Task<DocumentState> GetOrCreateDocumentAsync(string documentId)
    {
        await _documentLock.WaitAsync();
        try
        {
            if (!_documents.ContainsKey(documentId))
                _documents[documentId] = new DocumentState(documentId);
            return _documents[documentId];
        }
        finally
        {
            _documentLock.Release();
        }
    }

    private static DocumentOperation TransformOperation(DocumentState state, DocumentOperation op)
    {
        // Operational Transformation (OT) - simplified version
        // In production, use a full OT or CRDT library
        return op;
    }
}

public class DocumentState
{
    public string Id { get; }
    public string Content { get; private set; } = string.Empty;
    public long Version { get; set; } = 0;
    public List<DocumentOperation> History { get; } = [];

    public DocumentState(string id) => Id = id;

    public void ApplyOperation(DocumentOperation operation)
    {
        switch (operation.Type)
        {
            case "insert":
                Content = Content[..operation.Position] + 
                          operation.Text + 
                          Content[operation.Position..];
                break;
            case "delete":
                Content = Content[..operation.Position] + 
                          Content[(operation.Position + operation.Length)..];
                break;
        }
        History.Add(operation);
    }
}

public record DocumentOperation(
    string Id,
    string Type,           // "insert" | "delete"
    int Position,
    string Text = "",
    int Length = 0,
    long BaseVersion = 0);

public record CursorPosition(int Start, int End, string? Color = null);
```

---

## Step 744: Game: Real-time Multiplayer

```csharp
// Hubs/GameHub.cs
public class GameHub : Hub
{
    private static readonly Dictionary<string, GameRoom> _rooms = new();
    private static readonly Dictionary<string, string> _playerRooms = new();

    public async Task<string> CreateRoom(string playerName)
    {
        var roomId = GenerateRoomCode();
        var room = new GameRoom(roomId);
        room.AddPlayer(new Player(Context.ConnectionId, playerName));
        
        _rooms[roomId] = room;
        _playerRooms[Context.ConnectionId] = roomId;
        
        await Groups.AddToGroupAsync(Context.ConnectionId, roomId);
        await Clients.Caller.SendAsync("RoomCreated", roomId, room.State);
        
        return roomId;
    }

    public async Task<bool> JoinRoom(string roomId, string playerName)
    {
        if (!_rooms.TryGetValue(roomId, out var room) || room.IsFull)
        {
            await Clients.Caller.SendAsync("Error", "Room not found or full");
            return false;
        }

        var player = new Player(Context.ConnectionId, playerName);
        room.AddPlayer(player);
        _playerRooms[Context.ConnectionId] = roomId;
        
        await Groups.AddToGroupAsync(Context.ConnectionId, roomId);
        await Clients.Group(roomId).SendAsync("PlayerJoined", player, room.State);
        
        if (room.IsReadyToStart)
            await StartGame(roomId);
        
        return true;
    }

    public async Task MakeMove(string action, object? data = null)
    {
        if (!_playerRooms.TryGetValue(Context.ConnectionId, out var roomId)) return;
        if (!_rooms.TryGetValue(roomId, out var room)) return;

        var result = room.ProcessMove(Context.ConnectionId, action, data);
        
        await Clients.Group(roomId).SendAsync("GameStateUpdated", room.State);
        
        if (result.IsGameOver)
        {
            await Clients.Group(roomId).SendAsync("GameOver", result.Winner);
            _rooms.Remove(roomId);
        }
    }

    private async Task StartGame(string roomId)
    {
        if (!_rooms.TryGetValue(roomId, out var room)) return;
        room.Start();
        await Clients.Group(roomId).SendAsync("GameStarted", room.State);
    }

    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        if (_playerRooms.TryGetValue(Context.ConnectionId, out var roomId))
        {
            _playerRooms.Remove(Context.ConnectionId);
            if (_rooms.TryGetValue(roomId, out var room))
            {
                room.RemovePlayer(Context.ConnectionId);
                await Clients.Group(roomId).SendAsync("PlayerLeft", Context.ConnectionId);
                
                if (room.PlayerCount == 0)
                    _rooms.Remove(roomId);
            }
        }
        await base.OnDisconnectedAsync(exception);
    }

    private static string GenerateRoomCode()
    {
        var random = new Random();
        const string chars = "ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";
        return new string(Enumerable.Repeat(chars, 6)
            .Select(s => s[random.Next(s.Length)]).ToArray());
    }
}

public class GameRoom
{
    public string Id { get; }
    public List<Player> Players { get; } = [];
    public GameState State { get; private set; }
    public bool IsFull => Players.Count >= 4;
    public bool IsReadyToStart => Players.Count >= 2;
    public int PlayerCount => Players.Count;

    public GameRoom(string id)
    {
        Id = id;
        State = new GameState(id, "waiting", [], 0);
    }

    public void AddPlayer(Player player) => Players.Add(player);
    public void RemovePlayer(string connectionId) => Players.RemoveAll(p => p.ConnectionId == connectionId);

    public void Start()
    {
        State = State with { Status = "playing" };
    }

    public MoveResult ProcessMove(string connectionId, string action, object? data)
    {
        // Game logic here
        return new MoveResult(false, null);
    }
}

public record Player(string ConnectionId, string Name);
public record GameState(string RoomId, string Status, IEnumerable<Player> Players, int Turn);
public record MoveResult(bool IsGameOver, string? Winner);
```

---

## Step 745: TypeScript Client

```typescript
// signalr-client.ts
import * as signalR from "@microsoft/signalr";

interface NotificationDto {
    title: string;
    body: string;
    createdAt: string;
    type: string;
}

interface StockPrice {
    symbol: string;
    price: number;
    timestamp: string;
}

class NotificationHubClient {
    private connection: signalR.HubConnection;
    private reconnectAttempts = 0;

    constructor(private readonly hubUrl: string, private readonly getToken: () => Promise<string>) {
        this.connection = new signalR.HubConnectionBuilder()
            .withUrl(hubUrl, {
                accessTokenFactory: getToken
            })
            .withAutomaticReconnect({
                nextRetryDelayInMilliseconds: (context) => {
                    const delays = [0, 2000, 10000, 30000];
                    return delays[Math.min(context.previousRetryCount, delays.length - 1)];
                }
            })
            .configureLogging(signalR.LogLevel.Warning)
            .build();

        this.setupEventHandlers();
    }

    private setupEventHandlers() {
        this.connection.onreconnecting(() => {
            console.warn("SignalR: Reconnecting...");
            document.dispatchEvent(new CustomEvent("signalr:reconnecting"));
        });

        this.connection.onreconnected(() => {
            console.info("SignalR: Reconnected");
            document.dispatchEvent(new CustomEvent("signalr:reconnected"));
        });

        this.connection.onclose((error) => {
            console.error("SignalR: Connection closed", error);
            document.dispatchEvent(new CustomEvent("signalr:closed", { detail: error }));
        });
    }

    async start(): Promise<void> {
        try {
            await this.connection.start();
            console.info("SignalR: Connected");
        } catch (error) {
            console.error("Failed to connect:", error);
            setTimeout(() => this.start(), 5000);
        }
    }

    onNotification(handler: (notification: NotificationDto) => void): void {
        this.connection.on("ReceiveNotification", handler);
    }

    offNotification(handler: (notification: NotificationDto) => void): void {
        this.connection.off("ReceiveNotification", handler);
    }

    async subscribeToOrder(orderId: number): Promise<void> {
        await this.connection.invoke("SubscribeToOrder", orderId);
    }

    // Streaming
    async streamStockPrices(
        symbol: string, 
        onPrice: (price: StockPrice) => void,
        onComplete?: () => void
    ): Promise<signalR.ISubscription<StockPrice>> {
        const subscription = this.connection.stream<StockPrice>("StreamStockPrices", symbol)
            .subscribe({
                next: onPrice,
                error: (err) => console.error("Stream error:", err),
                complete: onComplete ?? (() => {})
            });
        return subscription;
    }

    async stop(): Promise<void> {
        await this.connection.stop();
    }

    get state(): signalR.HubConnectionState {
        return this.connection.state;
    }
}

// Usage
const client = new NotificationHubClient(
    "/hubs/notifications",
    async () => localStorage.getItem("authToken") ?? ""
);

await client.start();

client.onNotification((notification) => {
    showToast(notification.title, notification.body, notification.type);
});

await client.subscribeToOrder(123);

const stockSubscription = await client.streamStockPrices("AAPL", (price) => {
    updateStockChart(price.symbol, price.price);
});

// Stop streaming after 30 seconds
setTimeout(() => stockSubscription.dispose(), 30000);
```

---

## Step 746: SignalR với React

```typescript
// hooks/useSignalR.ts
import { useEffect, useRef, useCallback, useState } from 'react';
import * as signalR from '@microsoft/signalr';

type SignalRState = 'connecting' | 'connected' | 'reconnecting' | 'disconnected';

export function useSignalRHub(hubUrl: string, token?: string) {
    const connectionRef = useRef<signalR.HubConnection | null>(null);
    const [state, setState] = useState<SignalRState>('disconnected');

    useEffect(() => {
        const connection = new signalR.HubConnectionBuilder()
            .withUrl(hubUrl, {
                accessTokenFactory: token ? () => token : undefined
            })
            .withAutomaticReconnect()
            .build();

        connection.onreconnecting(() => setState('reconnecting'));
        connection.onreconnected(() => setState('connected'));
        connection.onclose(() => setState('disconnected'));

        setState('connecting');
        connection.start()
            .then(() => setState('connected'))
            .catch(() => setState('disconnected'));

        connectionRef.current = connection;

        return () => {
            connection.stop();
        };
    }, [hubUrl, token]);

    const on = useCallback(<T>(event: string, handler: (data: T) => void) => {
        connectionRef.current?.on(event, handler);
        return () => connectionRef.current?.off(event, handler);
    }, []);

    const invoke = useCallback(<T>(method: string, ...args: unknown[]): Promise<T> => {
        return connectionRef.current!.invoke<T>(method, ...args);
    }, []);

    return { state, on, invoke, connection: connectionRef.current };
}

// Component using the hook
function ChatComponent() {
    const { state, on, invoke } = useSignalRHub('/hubs/chat', token);
    const [messages, setMessages] = useState<ChatMessage[]>([]);
    const [input, setInput] = useState('');

    useEffect(() => {
        const unsubscribe = on<ChatMessage>('ReceiveMessage', (msg) => {
            setMessages(prev => [...prev, msg]);
        });
        return unsubscribe;
    }, [on]);

    const sendMessage = async () => {
        if (input.trim()) {
            await invoke('SendMessage', 'User', input);
            setInput('');
        }
    };

    return (
        <div>
            <p>Status: {state}</p>
            <div>
                {messages.map(m => (
                    <div key={m.id}><b>{m.user}:</b> {m.text}</div>
                ))}
            </div>
            <input value={input} onChange={e => setInput(e.target.value)} />
            <button onClick={sendMessage} disabled={state !== 'connected'}>Send</button>
        </div>
    );
}
```

---

## Step 747: Monitoring และ Diagnostics

```csharp
// Program.cs - SignalR monitoring
builder.Services.AddSignalR()
    .AddHubOptions<ChatHub>(options =>
    {
        options.EnableDetailedErrors = true;
    });

// Custom Hub Filter for monitoring
public class HubMetricsFilter : IHubFilter
{
    private readonly IMetricsService _metrics;

    public HubMetricsFilter(IMetricsService metrics)
    {
        _metrics = metrics;
    }

    public async ValueTask<object?> InvokeMethodAsync(
        HubInvocationContext invocationContext,
        Func<HubInvocationContext, ValueTask<object?>> next)
    {
        var hubName = invocationContext.Hub.GetType().Name;
        var methodName = invocationContext.HubMethodName;
        var stopwatch = System.Diagnostics.Stopwatch.StartNew();
        
        try
        {
            var result = await next(invocationContext);
            _metrics.RecordHubCall(hubName, methodName, stopwatch.Elapsed, success: true);
            return result;
        }
        catch (Exception ex)
        {
            _metrics.RecordHubCall(hubName, methodName, stopwatch.Elapsed, success: false);
            throw;
        }
    }

    public Task OnConnectedAsync(HubLifetimeContext context, Func<HubLifetimeContext, Task> next)
    {
        _metrics.IncrementConnections(context.Hub.GetType().Name);
        return next(context);
    }

    public Task OnDisconnectedAsync(
        HubLifetimeContext context, Exception? exception, 
        Func<HubLifetimeContext, Exception?, Task> next)
    {
        _metrics.DecrementConnections(context.Hub.GetType().Name);
        return next(context, exception);
    }
}

// Register filter
builder.Services.AddSignalR()
    .AddHubOptions<ChatHub>(options =>
    {
        options.AddFilter<HubMetricsFilter>();
    });
```

---

## Step 748: Live Notifications System

```csharp
// Complete notification system combining all concepts

// Models
public record Notification(
    Guid Id,
    string UserId,
    string Title,
    string Body,
    NotificationType Type,
    string? ActionUrl,
    DateTime CreatedAt,
    bool IsRead = false);

public enum NotificationType { Info, Success, Warning, Error, OrderUpdate, Message }

// Hub
[Authorize]
public class LiveNotificationHub : Hub<ILiveNotificationClient>
{
    private readonly INotificationRepository _repository;
    private readonly PresenceTracker _presence;

    public LiveNotificationHub(
        INotificationRepository repository, 
        PresenceTracker presence)
    {
        _repository = repository;
        _presence = presence;
    }

    public override async Task OnConnectedAsync()
    {
        var userId = Context.UserIdentifier!;
        
        await Groups.AddToGroupAsync(Context.ConnectionId, $"user_{userId}");
        await _presence.UserConnected(userId, Context.ConnectionId);

        // Send unread notifications on connect
        var unread = await _repository.GetUnreadAsync(userId);
        if (unread.Any())
            await Clients.Caller.ReceiveUnreadNotifications(unread);

        await base.OnConnectedAsync();
    }

    public async Task MarkAsRead(Guid notificationId)
    {
        var userId = Context.UserIdentifier!;
        await _repository.MarkAsReadAsync(notificationId, userId);
        
        var unreadCount = await _repository.GetUnreadCountAsync(userId);
        await Clients.Caller.UnreadCountChanged(unreadCount);
    }

    public async Task MarkAllAsRead()
    {
        var userId = Context.UserIdentifier!;
        await _repository.MarkAllAsReadAsync(userId);
        await Clients.Caller.UnreadCountChanged(0);
    }
}

public interface ILiveNotificationClient
{
    Task ReceiveNotification(Notification notification);
    Task ReceiveUnreadNotifications(IEnumerable<Notification> notifications);
    Task UnreadCountChanged(int count);
}

// Sending notifications from anywhere in the app
public class NotificationSender
{
    private readonly IHubContext<LiveNotificationHub, ILiveNotificationClient> _hub;
    private readonly INotificationRepository _repository;

    public NotificationSender(
        IHubContext<LiveNotificationHub, ILiveNotificationClient> hub,
        INotificationRepository repository)
    {
        _hub = hub;
        _repository = repository;
    }

    public async Task SendToUserAsync(string userId, string title, string body, 
        NotificationType type = NotificationType.Info, string? actionUrl = null)
    {
        var notification = new Notification(
            Guid.NewGuid(), userId, title, body, type, actionUrl, DateTime.UtcNow);
        
        await _repository.SaveAsync(notification);
        await _hub.Clients.Group($"user_{userId}").ReceiveNotification(notification);
    }

    public async Task BroadcastAsync(string title, string body, NotificationType type = NotificationType.Info)
    {
        var notification = new Notification(
            Guid.NewGuid(), "*", title, body, type, null, DateTime.UtcNow);
        
        await _hub.Clients.All.ReceiveNotification(notification);
    }
}
```

---

## Step 749: Testing SignalR Hubs

```csharp
// Tests/ChatHubTests.cs
using Microsoft.AspNetCore.SignalR;
using Moq;
using Xunit;

public class ChatHubTests
{
    [Fact]
    public async Task SendMessage_BroadcastsToAllClients()
    {
        // Arrange
        var mockClients = new Mock<IHubCallerClients>();
        var mockAllClients = new Mock<IClientProxy>();
        
        mockClients.Setup(c => c.All).Returns(mockAllClients.Object);
        
        var hub = new ChatHub(Mock.Of<ILogger<ChatHub>>())
        {
            Clients = mockClients.Object,
            Context = MockHubContext("conn1")
        };

        // Act
        await hub.SendMessage("Alice", "Hello, world!");

        // Assert
        mockAllClients.Verify(
            c => c.SendCoreAsync(
                "ReceiveMessage",
                It.Is<object[]>(args => 
                    args[0] is { } msg && 
                    msg.GetType().GetProperty("User")?.GetValue(msg)?.ToString() == "Alice"),
                default),
            Times.Once);
    }

    [Fact]
    public async Task OnConnectedAsync_NotifiesOtherClients()
    {
        var mockClients = new Mock<IHubCallerClients>();
        var mockOthers = new Mock<IClientProxy>();
        
        mockClients.Setup(c => c.Others).Returns(mockOthers.Object);
        
        var hub = new ChatHub(Mock.Of<ILogger<ChatHub>>())
        {
            Clients = mockClients.Object,
            Context = MockHubContext("new-conn")
        };

        await hub.OnConnectedAsync();

        mockOthers.Verify(
            c => c.SendCoreAsync("UserConnected", It.IsAny<object[]>(), default),
            Times.Once);
    }

    private static HubCallerContext MockHubContext(string connectionId)
    {
        var mockContext = new Mock<HubCallerContext>();
        mockContext.Setup(c => c.ConnectionId).Returns(connectionId);
        return mockContext.Object;
    }
}

// Integration test with TestServer
public class ChatHubIntegrationTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;

    public ChatHubIntegrationTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory;
    }

    [Fact]
    public async Task ChatHub_SendMessage_ReceivedByAllClients()
    {
        var client1 = _factory.Server.CreateWebSocketClient();
        var client2 = _factory.Server.CreateWebSocketClient();

        var connection1 = new HubConnectionBuilder()
            .WithUrl("http://localhost/hubs/chat", opts =>
                opts.HttpMessageHandlerFactory = _ => _factory.Server.CreateHandler())
            .Build();

        var connection2 = new HubConnectionBuilder()
            .WithUrl("http://localhost/hubs/chat", opts =>
                opts.HttpMessageHandlerFactory = _ => _factory.Server.CreateHandler())
            .Build();

        var receivedMessages = new List<object>();
        connection2.On("ReceiveMessage", (object msg) => receivedMessages.Add(msg));

        await connection1.StartAsync();
        await connection2.StartAsync();

        await connection1.InvokeAsync("SendMessage", "Alice", "Hello!");

        await Task.Delay(500);

        Assert.Single(receivedMessages);

        await connection1.StopAsync();
        await connection2.StopAsync();
    }
}
```

---

## Step 750: Production Best Practices

```csharp
// Program.cs - Production SignalR Configuration
builder.Services.AddSignalR(options =>
{
    // Timeouts
    options.HandshakeTimeout = TimeSpan.FromSeconds(15);
    options.KeepAliveInterval = TimeSpan.FromSeconds(15);
    options.ClientTimeoutInterval = TimeSpan.FromSeconds(30);
    
    // Message size limit
    options.MaximumReceiveMessageSize = 64 * 1024; // 64KB
    
    // Streaming
    options.StreamBufferCapacity = 10;
    
    // Errors
    options.EnableDetailedErrors = false; // Disable in production!
})
.AddStackExchangeRedis(redisConnection) // Scale-out
.AddJsonProtocol(options =>
{
    options.PayloadSerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;
    options.PayloadSerializerOptions.DefaultIgnoreCondition = 
        JsonIgnoreCondition.WhenWritingNull;
})
.AddMessagePackProtocol(); // Binary protocol for performance

// Sticky sessions for load balancers (required for non-Redis backplane)
// Use Application Request Routing (ARR) cookie in IIS
// Or use Azure SignalR Service

// appsettings.Production.json
/*
{
    "SignalR": {
        "Azure": {
            "ConnectionString": "Endpoint=https://....;"
        }
    }
}
*/

// Health check for SignalR
builder.Services.AddHealthChecks()
    .AddCheck("signalr", () => 
    {
        // Custom check logic
        return HealthCheckResult.Healthy("SignalR is running");
    });
```

```javascript
// Client Production Setup
const connection = new signalR.HubConnectionBuilder()
    .withUrl("/hubs/chat", {
        // Use WebSockets only (no fallback) in production for best performance
        transport: signalR.HttpTransportType.WebSockets,
        // Or fallback: signalR.HttpTransportType.WebSockets | signalR.HttpTransportType.LongPolling
        
        // Custom headers
        headers: { "X-Client-Version": "1.0" },
        
        // Skip negotiation for WebSockets-only (reduces 1 HTTP round trip)
        skipNegotiation: true
    })
    .withAutomaticReconnect({
        nextRetryDelayInMilliseconds: (retryContext) => {
            if (retryContext.elapsedMilliseconds < 60000) {
                // Retry within first 60 seconds
                return Math.random() * 10000;
            }
            // Stop retrying after 60 seconds
            return null;
        }
    })
    .configureLogging(signalR.LogLevel.Warning)
    .build();

// Handle connection state
connection.onclose(async () => {
    // Show "offline" indicator to user
    showConnectionStatus("offline");
    await sleep(5000);
    await startConnection();
});

connection.onreconnecting(() => showConnectionStatus("reconnecting"));
connection.onreconnected(() => showConnectionStatus("connected"));

async function startConnection() {
    try {
        await connection.start();
        showConnectionStatus("connected");
    } catch (err) {
        console.error(err);
        setTimeout(startConnection, 5000);
    }
}

startConnection();
```

---

## สรุป Part 27: SignalR

ในส่วนนี้เราได้เรียนรู้:

| หัวข้อ | ขั้นตอน |
|--------|---------|
| SignalR Overview และ Setup | 731 |
| Hub พื้นฐาน (Send, Broadcast, Events) | 732 |
| Groups และ Chat Rooms | 733 |
| Strongly Typed Hubs | 734 |
| Server-side Hub Invocation (IHubContext) | 735 |
| Authentication ใน SignalR | 736 |
| .NET Client | 737 |
| Streaming (Server→Client, Client→Server) | 738 |
| Scale-out ด้วย Redis Backplane | 739 |
| Real-time Dashboard + BackgroundService | 740 |
| Rate Limiting และ Throttling | 741 |
| Presence System (Online/Offline tracking) | 742 |
| Collaborative Real-time Editor (OT) | 743 |
| Multiplayer Game | 744 |
| TypeScript Client | 745 |
| React Integration | 746 |
| Monitoring และ HubFilter | 747 |
| Complete Notification System | 748 |
| Testing Hubs | 749 |
| Production Best Practices | 750 |

**ขั้นตอนต่อไป**: Part 28 จะเรียนรู้ **gRPC** - High-performance RPC framework
