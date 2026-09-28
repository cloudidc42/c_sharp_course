# Part 62: Blazor WebAssembly — Full-Stack C# in the Browser

## Steps 1626-1650: Production Blazor WASM Applications

---

## Step 1626: Blazor Architecture Landscape

```
Blazor Hosting Models
══════════════════════════════════════════════════════════════

  Blazor Server              Blazor WASM              Blazor United (Auto)
  ─────────────              ────────────             ────────────────────
  .NET runs on server        .NET runs in browser     Server first, then
  SignalR for UI updates     Downloaded WebAssembly   migrates to WASM
  Fast startup               Offline capable          Best of both worlds
  SEO friendly               No server required
                             Full SPA experience

  Blazor WASM Architecture:
  ┌─────────────────────────────────────────────────────────┐
  │                     Browser                              │
  │  ┌──────────────────────────────────────────────────┐   │
  │  │             WebAssembly Runtime                   │   │
  │  │  ┌─────────────┐  ┌──────────────────────────┐  │   │
  │  │  │  .NET WASM  │  │   Blazor Component Tree  │  │   │
  │  │  │   Runtime   │  │   App.razor              │  │   │
  │  │  │             │  │   ├─ Layout              │  │   │
  │  │  │  C# code    │  │   │   └─ NavMenu         │  │   │
  │  │  │  executes   │  │   └─ Pages               │  │   │
  │  │  │  here       │  │       ├─ Counter         │  │   │
  │  │  └─────────────┘  │       └─ Orders          │  │   │
  │  └──────────────────────────────────────────────────┘   │
  │          │ JS Interop                 │ HTTP             │
  │          ▼                            ▼                  │
  │  DOM Manipulation                API Server             │
  └─────────────────────────────────────────────────────────┘
```

### Project Structure

```
BlazorApp/
├── BlazorApp.Client/          ← WASM frontend
│   ├── Program.cs
│   ├── App.razor
│   ├── Pages/
│   ├── Components/
│   ├── Services/
│   └── wwwroot/
│       ├── index.html
│       └── css/
├── BlazorApp.Server/          ← API host (serves WASM + API)
│   ├── Program.cs
│   └── Controllers/
└── BlazorApp.Shared/          ← Shared models
    └── Models/
```

---

## Step 1627: Blazor WASM Setup

```csharp
// BlazorApp.Client/Program.cs
using Microsoft.AspNetCore.Components.Web;
using Microsoft.AspNetCore.Components.WebAssembly.Hosting;
using BlazorApp.Client;
using BlazorApp.Client.Services;

var builder = WebAssemblyHostBuilder.CreateDefault(args);
builder.RootComponents.Add<App>("#app");
builder.RootComponents.Add<HeadOutlet>("head::after");

// ── HTTP Client ────────────────────────────────────────────────────────────
builder.Services.AddScoped(sp => new HttpClient
{
    BaseAddress = new Uri(builder.HostEnvironment.BaseAddress)
});

builder.Services.AddHttpClient<IOrderApiClient, OrderApiClient>(c =>
    c.BaseAddress = new Uri(builder.HostEnvironment.BaseAddress + "api/"));

// ── Auth ───────────────────────────────────────────────────────────────────
builder.Services
    .AddOidcAuthentication(options =>
    {
        builder.Configuration.Bind("Auth", options.ProviderOptions);
        options.ProviderOptions.DefaultScopes.Add("openid");
        options.ProviderOptions.DefaultScopes.Add("profile");
        options.ProviderOptions.DefaultScopes.Add("api");
    });

// ── State Management ───────────────────────────────────────────────────────
builder.Services.AddScoped<CartState>();
builder.Services.AddScoped<UserState>();

// ── Blazored Libraries ─────────────────────────────────────────────────────
builder.Services.AddBlazoredLocalStorage();
builder.Services.AddBlazoredToast();

await builder.Build().RunAsync();
```

```html
<!-- BlazorApp.Client/wwwroot/index.html -->
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>BlazorApp</title>
    <base href="/" />
    <link rel="stylesheet" href="css/app.css" />
    <link rel="stylesheet" href="BlazorApp.Client.styles.css" />
    <!-- Preload hints for better performance -->
    <link rel="preconnect" href="https://fonts.googleapis.com" />
</head>
<body>
    <div id="app">
        <!-- Loading indicator before WASM loads -->
        <div class="loading-progress">
            <div class="loading-progress-text"></div>
            <div class="loading-progress-bar">
                <div class="loading-progress-bar-fill"></div>
            </div>
        </div>
    </div>

    <div id="blazor-error-ui">
        An unhandled error has occurred.
        <a href="" class="reload">Reload</a>
        <a class="dismiss">🗙</a>
    </div>

    <script src="_framework/blazor.webassembly.js"></script>
</body>
</html>
```

---

## Step 1628: Component Architecture

```razor
@* App.razor *@
<Router AppAssembly="@typeof(App).Assembly">
    <Found Context="routeData">
        <AuthorizeRouteView RouteData="@routeData" DefaultLayout="@typeof(MainLayout)">
            <NotAuthorized>
                <RedirectToLogin />
            </NotAuthorized>
            <Authorizing>
                <LoadingSpinner />
            </Authorizing>
        </AuthorizeRouteView>
        <FocusOnNavigate RouteData="@routeData" Selector="h1" />
    </Found>
    <NotFound>
        <PageTitle>Not Found</PageTitle>
        <LayoutView Layout="@typeof(MainLayout)">
            <NotFoundPage />
        </LayoutView>
    </NotFound>
</Router>
```

```razor
@* Shared/MainLayout.razor *@
@inherits LayoutComponentBase
@inject NavigationManager Nav
@inject CartState Cart

<div class="app-shell">
    <header class="top-nav">
        <NavLink href="/" class="brand">ShopCo</NavLink>
        <nav>
            <NavLink href="/products">Products</NavLink>
            <NavLink href="/orders">Orders</NavLink>
        </nav>
        <div class="nav-actions">
            <CartButton Count="@Cart.ItemCount" />
            <AuthDisplay />
        </div>
    </header>

    <main>
        @Body
    </main>

    <footer>© 2025 ShopCo</footer>
</div>

<BlazoredToasts Position="ToastPosition.TopRight"
                Timeout="5"
                MaxToastCount="3" />
```

---

## Step 1629: Pages with Forms & Validation

```razor
@* Pages/Orders/PlaceOrder.razor *@
@page "/orders/new"
@attribute [Authorize]
@inject IOrderApiClient OrderApi
@inject NavigationManager Nav
@inject IToastService Toast

<PageTitle>Place Order</PageTitle>

<h1>Place New Order</h1>

<EditForm Model="@_model" OnValidSubmit="HandleValidSubmitAsync">
    <DataAnnotationsValidator />
    <ValidationSummary class="alert alert-danger" />

    <div class="form-section">
        <h3>Shipping Address</h3>
        <div class="form-group">
            <label for="street">Street Address</label>
            <InputText id="street" class="form-control" @bind-Value="_model.Street"
                       placeholder="123 Main St" />
            <ValidationMessage For="@(() => _model.Street)" />
        </div>
        <div class="form-row">
            <div class="form-group col-md-8">
                <label for="city">City</label>
                <InputText id="city" class="form-control" @bind-Value="_model.City" />
                <ValidationMessage For="@(() => _model.City)" />
            </div>
            <div class="form-group col-md-4">
                <label for="zip">ZIP</label>
                <InputText id="zip" class="form-control" @bind-Value="_model.Zip" />
                <ValidationMessage For="@(() => _model.Zip)" />
            </div>
        </div>
    </div>

    <div class="form-section">
        <h3>Order Items</h3>
        <OrderItemsEditor Items="_model.Items" OnItemsChanged="HandleItemsChanged" />
        @if (_model.Items.Count == 0)
        {
            <div class="alert alert-warning">Add at least one item to your order.</div>
        }
    </div>

    <div class="order-summary">
        <h3>Summary</h3>
        <p>Subtotal: @_model.Items.Sum(i => i.UnitPrice * i.Quantity).ToString("C")</p>
        <p>Shipping: @(_model.Items.Any() ? "$5.99" : "$0.00")</p>
        <p><strong>Total: @(_model.Items.Sum(i => i.UnitPrice * i.Quantity) + (_model.Items.Any() ? 5.99m : 0)).ToString("C")</strong></p>
    </div>

    <button type="submit" class="btn btn-primary" disabled="@_isSubmitting">
        @if (_isSubmitting)
        {
            <span class="spinner-border spinner-border-sm me-2"></span>
        }
        Place Order
    </button>
    <button type="button" class="btn btn-outline-secondary ms-2" @onclick="Cancel">Cancel</button>
</EditForm>

@code {
    private PlaceOrderModel _model = new();
    private bool _isSubmitting;

    protected override void OnInitialized()
    {
        // Pre-populate from cart if available
        _model = PlaceOrderModel.FromCart(CartState.Items);
    }

    private async Task HandleValidSubmitAsync()
    {
        if (!_model.Items.Any())
        {
            Toast.ShowError("Please add at least one item");
            return;
        }

        _isSubmitting = true;
        try
        {
            var result = await OrderApi.CreateOrderAsync(new CreateOrderRequest
            {
                Street   = _model.Street,
                City     = _model.City,
                Zip      = _model.Zip,
                Items    = _model.Items.Select(i => new OrderItemRequest(
                    i.ProductId, i.Name, i.Quantity, i.UnitPrice)).ToList()
            });

            Toast.ShowSuccess($"Order #{result.OrderId} placed successfully!");
            Nav.NavigateTo($"/orders/{result.OrderId}");
        }
        catch (HttpRequestException ex)
        {
            Toast.ShowError($"Failed to place order: {ex.Message}");
        }
        finally
        {
            _isSubmitting = false;
        }
    }

    private void HandleItemsChanged(List<OrderItemModel> items)
        => _model.Items = items;

    private void Cancel() => Nav.NavigateTo("/orders");
}
```

```csharp
// Models/PlaceOrderModel.cs — validation model
using System.ComponentModel.DataAnnotations;

public class PlaceOrderModel
{
    [Required(ErrorMessage = "Street address is required")]
    [StringLength(200, MinimumLength = 5)]
    public string Street { get; set; } = string.Empty;

    [Required(ErrorMessage = "City is required")]
    [StringLength(100)]
    public string City { get; set; } = string.Empty;

    [Required(ErrorMessage = "ZIP code is required")]
    [RegularExpression(@"^\d{5}(-\d{4})?$", ErrorMessage = "Enter a valid ZIP code")]
    public string Zip { get; set; } = string.Empty;

    public List<OrderItemModel> Items { get; set; } = [];

    public static PlaceOrderModel FromCart(IEnumerable<CartItem> cartItems) => new()
    {
        Items = cartItems.Select(i => new OrderItemModel
        {
            ProductId = i.ProductId,
            Name      = i.Name,
            Quantity  = i.Quantity,
            UnitPrice = i.Price
        }).ToList()
    };
}

public class OrderItemModel
{
    [Required] public string ProductId { get; set; } = string.Empty;
    [Required] public string Name      { get; set; } = string.Empty;
    [Range(1, 100, ErrorMessage = "Quantity must be between 1 and 100")]
    public int Quantity    { get; set; } = 1;
    [Range(0.01, 100000)]
    public decimal UnitPrice { get; set; }
}
```

---

## Step 1630: Component Library — Reusable UI

```razor
@* Components/OrderCard.razor *@
@using BlazorApp.Shared.Models

<div class="order-card @StatusClass" @onclick="OnClick">
    <div class="order-card-header">
        <span class="order-id">Order #@Order.Id.ToString()[..8]</span>
        <span class="order-status badge @BadgeClass">@Order.Status</span>
    </div>
    <div class="order-card-body">
        <p class="order-date text-muted">@Order.PlacedAt.ToString("MMM dd, yyyy")</p>
        <p class="order-items">@Order.ItemCount item(s)</p>
        <p class="order-total fw-bold">@Order.Total.ToString("C")</p>
    </div>
    @if (ShowActions)
    {
        <div class="order-card-actions">
            <a href="/orders/@Order.Id" class="btn btn-sm btn-outline-primary">View Details</a>
            @if (Order.Status == "Pending" || Order.Status == "Confirmed")
            {
                <button class="btn btn-sm btn-outline-danger ms-2"
                        @onclick:stopPropagation="true"
                        @onclick="HandleCancelAsync">
                    Cancel
                </button>
            }
        </div>
    }
</div>

@code {
    [Parameter, EditorRequired] public OrderSummary Order { get; set; } = null!;
    [Parameter] public bool ShowActions { get; set; } = true;
    [Parameter] public EventCallback<OrderSummary> OnCancelRequested { get; set; }
    [Parameter] public EventCallback<OrderSummary> OnCardClicked { get; set; }

    private string StatusClass => Order.Status switch
    {
        "Delivered"  => "order-card--delivered",
        "Cancelled"  => "order-card--cancelled",
        "Shipped"    => "order-card--shipped",
        _            => "order-card--active"
    };

    private string BadgeClass => Order.Status switch
    {
        "Delivered"  => "bg-success",
        "Cancelled"  => "bg-secondary",
        "Shipped"    => "bg-info",
        "Confirmed"  => "bg-primary",
        _            => "bg-warning text-dark"
    };

    private async Task HandleCancelAsync()
        => await OnCancelRequested.InvokeAsync(Order);

    private async Task OnClick()
        => await OnCardClicked.InvokeAsync(Order);
}
```

```razor
@* Components/VirtualOrderList.razor — virtualized list for large datasets *@
@using Microsoft.AspNetCore.Components.Web.Virtualization

<div class="order-list" style="height: 600px; overflow-y: auto;">
    <Virtualize Items="Orders"
                Context="order"
                ItemSize="120"
                OverscanCount="3">
        <OrderCard Order="@order"
                   OnCancelRequested="@HandleCancel"
                   OnCardClicked="@HandleCardClick" />
    </Virtualize>
</div>

@code {
    [Parameter, EditorRequired] public IReadOnlyList<OrderSummary> Orders { get; set; } = [];
    [Parameter] public EventCallback<OrderSummary> OnOrderCancelled { get; set; }
    [Parameter] public EventCallback<OrderSummary> OnOrderSelected  { get; set; }

    private async Task HandleCancel(OrderSummary order)
        => await OnOrderCancelled.InvokeAsync(order);

    private async Task HandleCardClick(OrderSummary order)
        => await OnOrderSelected.InvokeAsync(order);
}
```

---

## Step 1631: State Management

```csharp
// Services/CartState.cs — reactive state container
using System.Text.Json;

public class CartState
{
    private readonly Blazored.LocalStorage.ILocalStorageService _storage;
    private List<CartItem> _items = [];

    public IReadOnlyList<CartItem> Items => _items;
    public int    ItemCount  => _items.Sum(i => i.Quantity);
    public decimal Total     => _items.Sum(i => i.Price * i.Quantity);

    public event Action? OnChange;

    public CartState(Blazored.LocalStorage.ILocalStorageService storage)
        => _storage = storage;

    public async Task InitializeAsync()
    {
        var stored = await _storage.GetItemAsync<List<CartItem>>("cart");
        _items = stored ?? [];
        NotifyStateChanged();
    }

    public async Task AddItemAsync(CartItem item)
    {
        var existing = _items.FirstOrDefault(i => i.ProductId == item.ProductId);
        if (existing is not null)
            existing.Quantity += item.Quantity;
        else
            _items.Add(item);

        await PersistAsync();
        NotifyStateChanged();
    }

    public async Task RemoveItemAsync(string productId)
    {
        _items.RemoveAll(i => i.ProductId == productId);
        await PersistAsync();
        NotifyStateChanged();
    }

    public async Task UpdateQuantityAsync(string productId, int newQuantity)
    {
        var item = _items.FirstOrDefault(i => i.ProductId == productId);
        if (item is null) return;

        if (newQuantity <= 0)
            await RemoveItemAsync(productId);
        else
        {
            item.Quantity = newQuantity;
            await PersistAsync();
            NotifyStateChanged();
        }
    }

    public async Task ClearAsync()
    {
        _items.Clear();
        await _storage.RemoveItemAsync("cart");
        NotifyStateChanged();
    }

    private async Task PersistAsync()
        => await _storage.SetItemAsync("cart", _items);

    private void NotifyStateChanged()
        => OnChange?.Invoke();
}

// Component subscribing to state changes
@code {
    [Inject] CartState Cart { get; set; } = null!;

    protected override void OnInitialized()
    {
        Cart.OnChange += StateHasChanged;  // re-render on change
    }

    public void Dispose()
    {
        Cart.OnChange -= StateHasChanged;  // prevent memory leak
    }
}
```

---

## Step 1632: JavaScript Interop

```csharp
// Services/ClipboardService.cs — call JS from C#
using Microsoft.JSInterop;

public class ClipboardService
{
    private readonly IJSRuntime _js;

    public ClipboardService(IJSRuntime js) => _js = js;

    public async Task CopyToClipboardAsync(string text)
    {
        await _js.InvokeVoidAsync("navigator.clipboard.writeText", text);
    }

    public async Task<string> ReadFromClipboardAsync()
        => await _js.InvokeAsync<string>("navigator.clipboard.readText");
}

// Chart.js integration
public class ChartService
{
    private readonly IJSRuntime _js;

    public ChartService(IJSRuntime js) => _js = js;

    public async Task RenderOrderChartAsync(
        string canvasId,
        IEnumerable<string> labels,
        IEnumerable<decimal> values)
    {
        await _js.InvokeVoidAsync("renderOrderChart", canvasId,
            labels.ToArray(), values.ToArray());
    }
}
```

```javascript
// wwwroot/js/interop.js — JS functions called from C#
window.renderOrderChart = function (canvasId, labels, values) {
    const ctx = document.getElementById(canvasId)?.getContext('2d');
    if (!ctx) return;

    new Chart(ctx, {
        type: 'bar',
        data: {
            labels: labels,
            datasets: [{
                label: 'Revenue',
                data: values,
                backgroundColor: 'rgba(54, 162, 235, 0.6)'
            }]
        },
        options: { responsive: true, plugins: { legend: { position: 'top' } } }
    });
};

window.downloadFile = function (filename, content, mimeType) {
    const blob = new Blob([content], { type: mimeType });
    const url  = URL.createObjectURL(blob);
    const a    = document.createElement('a');
    a.href     = url;
    a.download = filename;
    a.click();
    URL.revokeObjectURL(url);
};

// C# called from JavaScript
DotNet.invokeMethodAsync('BlazorApp.Client', 'OnNetworkStatusChanged', navigator.onLine);
window.addEventListener('online',  () => DotNet.invokeMethodAsync('BlazorApp.Client', 'OnNetworkStatusChanged', true));
window.addEventListener('offline', () => DotNet.invokeMethodAsync('BlazorApp.Client', 'OnNetworkStatusChanged', false));
```

```csharp
// Services/NetworkStatusService.cs — called from JavaScript
using Microsoft.JSInterop;

public class NetworkStatusService
{
    public bool IsOnline { get; private set; } = true;
    public event Action<bool>? NetworkStatusChanged;

    [JSInvokable]
    public static Task OnNetworkStatusChanged(bool isOnline)
    {
        Current?.SetStatus(isOnline);
        return Task.CompletedTask;
    }

    private static NetworkStatusService? Current;

    public NetworkStatusService() => Current = this;

    private void SetStatus(bool isOnline)
    {
        IsOnline = isOnline;
        NetworkStatusChanged?.Invoke(isOnline);
    }
}
```

---

## Step 1633: Authentication & Authorization

```razor
@* Components/AuthDisplay.razor *@
<AuthorizeView>
    <Authorized>
        <div class="user-menu dropdown">
            <button class="btn btn-outline-light dropdown-toggle" type="button"
                    data-bs-toggle="dropdown">
                <img src="@context.User.FindFirst("picture")?.Value" class="avatar"
                     alt="Profile" onerror="this.src='/img/default-avatar.png'" />
                @context.User.FindFirst("name")?.Value
            </button>
            <ul class="dropdown-menu">
                <li><a class="dropdown-item" href="/profile">Profile</a></li>
                <li><a class="dropdown-item" href="/orders">My Orders</a></li>
                <li><hr class="dropdown-divider" /></li>
                <li>
                    <button class="dropdown-item" @onclick="LogoutAsync">Sign Out</button>
                </li>
            </ul>
        </div>
    </Authorized>
    <NotAuthorized>
        <a href="/login" class="btn btn-outline-light">Sign In</a>
    </NotAuthorized>
    <Authorizing>
        <div class="spinner-border spinner-border-sm text-light"></div>
    </Authorizing>
</AuthorizeView>

@code {
    [Inject] SignOutSessionStateManager SignOutManager { get; set; } = null!;
    [Inject] NavigationManager Nav { get; set; } = null!;

    async Task LogoutAsync()
    {
        await SignOutManager.SetSignOutState();
        Nav.NavigateTo("authentication/logout");
    }
}
```

```razor
@* Pages/Authentication.razor *@
@page "/authentication/{action}"
<RemoteAuthenticatorView Action="@Action" />

@code {
    [Parameter] public string? Action { get; set; }
}
```

```razor
@* Protecting routes *@
@page "/orders"
@attribute [Authorize]

@* Role-based *@
@attribute [Authorize(Roles = "admin,manager")]

@* In template *@
<AuthorizeView Policy="CanViewOrders">
    <Authorized>
        <OrderList />
    </Authorized>
    <NotAuthorized>
        <p>You don't have permission to view orders.</p>
    </NotAuthorized>
</AuthorizeView>
```

---

## Step 1634: HTTP Client & Error Handling

```csharp
// Services/OrderApiClient.cs
using System.Net.Http.Json;
using BlazorApp.Shared.Models;

public interface IOrderApiClient
{
    Task<OrderSummary[]> GetOrdersAsync(CancellationToken ct = default);
    Task<OrderDetails?> GetOrderAsync(Guid id, CancellationToken ct = default);
    Task<CreateOrderResult> CreateOrderAsync(CreateOrderRequest req, CancellationToken ct = default);
    Task CancelOrderAsync(Guid id, string reason, CancellationToken ct = default);
}

public class OrderApiClient : IOrderApiClient
{
    private readonly HttpClient _http;
    private readonly IToastService _toast;
    private readonly ILogger<OrderApiClient> _log;

    public OrderApiClient(
        HttpClient http,
        IToastService toast,
        ILogger<OrderApiClient> log)
    {
        _http  = http;
        _toast = toast;
        _log   = log;
    }

    public async Task<OrderSummary[]> GetOrdersAsync(CancellationToken ct = default)
    {
        try
        {
            return await _http.GetFromJsonAsync<OrderSummary[]>("orders", ct)
                ?? [];
        }
        catch (HttpRequestException ex)
        {
            _log.LogError(ex, "Failed to fetch orders");
            _toast.ShowError("Failed to load orders. Please try again.");
            return [];
        }
    }

    public async Task<OrderDetails?> GetOrderAsync(Guid id, CancellationToken ct = default)
    {
        var response = await _http.GetAsync($"orders/{id}", ct);

        if (response.StatusCode == System.Net.HttpStatusCode.NotFound)
            return null;

        if (!response.IsSuccessStatusCode)
        {
            var error = await response.Content.ReadFromJsonAsync<ProblemDetails>(ct);
            throw new ApiException(error?.Detail ?? "Unknown error", response.StatusCode);
        }

        return await response.Content.ReadFromJsonAsync<OrderDetails>(ct);
    }

    public async Task<CreateOrderResult> CreateOrderAsync(
        CreateOrderRequest req,
        CancellationToken ct = default)
    {
        var response = await _http.PostAsJsonAsync("orders", req, ct);

        if (!response.IsSuccessStatusCode)
        {
            var error = await response.Content.ReadFromJsonAsync<ValidationProblemDetails>(ct);
            throw new ValidationException(error?.Errors);
        }

        return await response.Content.ReadFromJsonAsync<CreateOrderResult>(ct)
            ?? throw new InvalidOperationException("Empty response");
    }

    public async Task CancelOrderAsync(Guid id, string reason, CancellationToken ct = default)
    {
        var response = await _http.PostAsJsonAsync(
            $"orders/{id}/cancel",
            new { reason },
            ct);

        if (!response.IsSuccessStatusCode)
        {
            var error = await response.Content.ReadFromJsonAsync<ProblemDetails>(ct);
            throw new ApiException(error?.Detail ?? "Cannot cancel order", response.StatusCode);
        }
    }
}

public class ApiException : Exception
{
    public System.Net.HttpStatusCode StatusCode { get; }
    public ApiException(string message, System.Net.HttpStatusCode code)
        : base(message) => StatusCode = code;
}
```

---

## Step 1635: Real-Time Updates with SignalR

```csharp
// Services/OrderHubClient.cs — real-time order updates
using Microsoft.AspNetCore.SignalR.Client;

public class OrderHubClient : IAsyncDisposable
{
    private HubConnection? _connection;
    private readonly IAccessTokenProvider _tokenProvider;

    public event Action<OrderStatusUpdate>? OrderStatusChanged;
    public event Action? ConnectionStateChanged;
    public HubConnectionState State => _connection?.State ?? HubConnectionState.Disconnected;

    public OrderHubClient(IAccessTokenProvider tokenProvider)
        => _tokenProvider = tokenProvider;

    public async Task StartAsync(CancellationToken ct = default)
    {
        var tokenResult = await _tokenProvider.RequestAccessToken();
        tokenResult.TryGetToken(out var token);

        _connection = new HubConnectionBuilder()
            .WithUrl("/hubs/orders", options =>
            {
                options.AccessTokenProvider = async () => token?.Value;
            })
            .WithAutomaticReconnect(new[] { 0, 2, 5, 15, 30, 60 })
            .Build();

        _connection.On<OrderStatusUpdate>("OrderStatusChanged", update =>
            OrderStatusChanged?.Invoke(update));

        _connection.Reconnecting    += _ => { ConnectionStateChanged?.Invoke(); return Task.CompletedTask; };
        _connection.Reconnected     += _ => { ConnectionStateChanged?.Invoke(); return Task.CompletedTask; };
        _connection.Closed          += _ => { ConnectionStateChanged?.Invoke(); return Task.CompletedTask; };

        await _connection.StartAsync(ct);
        ConnectionStateChanged?.Invoke();
    }

    public async Task WatchOrderAsync(Guid orderId)
    {
        if (_connection?.State == HubConnectionState.Connected)
            await _connection.InvokeAsync("WatchOrder", orderId.ToString());
    }

    public async Task UnwatchOrderAsync(Guid orderId)
    {
        if (_connection?.State == HubConnectionState.Connected)
            await _connection.InvokeAsync("UnwatchOrder", orderId.ToString());
    }

    public async ValueTask DisposeAsync()
    {
        if (_connection is not null)
            await _connection.DisposeAsync();
    }
}
```

```razor
@* Pages/Orders/OrderDetails.razor — live updates *@
@page "/orders/{Id:guid}"
@attribute [Authorize]
@implements IAsyncDisposable
@inject IOrderApiClient Api
@inject OrderHubClient Hub
@inject IToastService Toast

<PageTitle>Order @Id.ToString()[..8]</PageTitle>

@if (_order is null)
{
    <LoadingSpinner Message="Loading order..." />
}
else
{
    <div class="order-details">
        <h1>Order #@_order.Id.ToString()[..8]</h1>

        <div class="status-badge @_statusClass">@_order.Status</div>

        @if (_connectionState != HubConnectionState.Connected)
        {
            <div class="alert alert-warning small">
                Live updates unavailable — @_connectionState.ToString().ToLower()
            </div>
        }

        <OrderTimeline Events="@_order.Timeline" />
        <OrderItemsList Items="@_order.Items" />

        @if (_order.TrackingNumber is not null)
        {
            <ShippingTracker TrackingNumber="@_order.TrackingNumber" Carrier="@_order.Carrier" />
        }
    </div>
}

@code {
    [Parameter] public Guid Id { get; set; }

    private OrderDetails? _order;
    private HubConnectionState _connectionState = HubConnectionState.Disconnected;

    protected override async Task OnInitializedAsync()
    {
        _order = await Api.GetOrderAsync(Id);

        // Subscribe to SignalR updates
        Hub.OrderStatusChanged += HandleStatusChanged;
        Hub.ConnectionStateChanged += HandleConnectionChanged;

        await Hub.StartAsync();
        await Hub.WatchOrderAsync(Id);
    }

    private void HandleStatusChanged(OrderStatusUpdate update)
    {
        if (update.OrderId != Id) return;

        _order = _order! with { Status = update.NewStatus };

        if (update.NewStatus == "Shipped")
            Toast.ShowInfo($"Your order has been shipped! Tracking: {update.TrackingNumber}");

        InvokeAsync(StateHasChanged);
    }

    private void HandleConnectionChanged()
    {
        _connectionState = Hub.State;
        InvokeAsync(StateHasChanged);
    }

    public async ValueTask DisposeAsync()
    {
        Hub.OrderStatusChanged    -= HandleStatusChanged;
        Hub.ConnectionStateChanged -= HandleConnectionChanged;
        await Hub.UnwatchOrderAsync(Id);
    }
}
```

---

## Step 1636: Performance Optimization

```csharp
// Lazy loading heavy pages
// Program.cs
builder.Services.AddLazyAssemblyLoader();

// App.razor — lazy loaded routes
@using Microsoft.AspNetCore.Components.WebAssembly.Services

<Router AppAssembly="@typeof(App).Assembly"
        AdditionalAssemblies="@_lazyLoadedAssemblies"
        OnNavigateAsync="@OnNavigateAsync">
    ...
</Router>

@code {
    [Inject] LazyAssemblyLoader LazyLoader { get; set; } = null!;
    private List<Assembly> _lazyLoadedAssemblies = [];

    async Task OnNavigateAsync(NavigationContext ctx)
    {
        if (ctx.Path == "admin/reports")
        {
            var assemblies = await LazyLoader.LoadAssembliesAsync(
                ["BlazorApp.Admin.dll"]);
            _lazyLoadedAssemblies.AddRange(assemblies);
        }
    }
}
```

```razor
@* Optimize re-renders with ShouldRender *@
@code {
    private int _previousCount = -1;

    protected override bool ShouldRender()
    {
        if (_count == _previousCount) return false;
        _previousCount = _count;
        return true;
    }
}

@* Freeze pure display components with @key *@
@foreach (var order in Orders)
{
    <OrderCard @key="order.Id" Order="order" />
}
```

---

## Step 1637: PWA — Progressive Web App

```json
// wwwroot/manifest.json
{
  "name": "ShopCo",
  "short_name": "ShopCo",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#1a73e8",
  "icons": [
    { "src": "icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "icon-512.png", "sizes": "512x512", "type": "image/png" }
  ]
}
```

```javascript
// wwwroot/service-worker.published.js
const CACHE_NAME = 'shopco-v1';
const offlineAssets = [
    '/',
    '/css/app.css',
    '/js/app.js',
    '/_framework/blazor.webassembly.js',
    '/offline.html'
];

self.addEventListener('install', event => {
    event.waitUntil(
        caches.open(CACHE_NAME).then(cache => cache.addAll(offlineAssets))
    );
});

self.addEventListener('fetch', event => {
    if (event.request.mode === 'navigate') {
        event.respondWith(
            fetch(event.request).catch(() =>
                caches.match('/offline.html')
            )
        );
        return;
    }

    event.respondWith(
        caches.match(event.request).then(cached =>
            cached ?? fetch(event.request).then(response => {
                const clone = response.clone();
                caches.open(CACHE_NAME).then(cache => cache.put(event.request, clone));
                return response;
            })
        ).catch(() => new Response('Offline', { status: 503 }))
    );
});
```

---

## Step 1638: Blazor WASM Production Checklist

```
Blazor WASM Production Checklist
═══════════════════════════════════════════════════════════

Performance
✅ Lazy load assemblies for infrequently visited pages
✅ Use @key on list items to minimize DOM diffing
✅ ShouldRender() override for pure display components
✅ Virtualize<T> for lists with >50 items
✅ Compress WASM files: Brotli pre-compressed
✅ Tree-shaking unused .NET libraries (PublishTrimmed)
✅ Ahead-of-time compilation (WasmEnableAot=true for perf)

Architecture
✅ Shared models library (Server + Client share types)
✅ HTTP client registered as scoped (not singleton)
✅ State containers for cross-component data
✅ IAsyncDisposable on components with subscriptions
✅ Event handler cleanup to prevent memory leaks

Forms
✅ DataAnnotationsValidator + ValidationSummary on all forms
✅ Client-side validation matches server-side
✅ Disable submit button while processing
✅ Show loading indicator on async operations
✅ Toast/alert notifications for success/failure

Auth
✅ [Authorize] attribute on all protected pages
✅ AuthorizeView for conditional UI rendering
✅ Access token sent with every API call
✅ Token refresh handled by MSAL/OIDC library
✅ Unauthorized → redirect to login

Offline/PWA
✅ Service worker for asset caching
✅ Offline fallback page
✅ manifest.json with icons
✅ Handle network status changes gracefully
✅ Queue actions when offline, sync when back online

Real-Time
✅ SignalR hub client with automatic reconnect
✅ Show connection status to user
✅ Handle reconnect gracefully (re-subscribe)
✅ IAsyncDisposable to clean up connections
```

---

**Part 62 ครอบคลุม:**
- Blazor WASM setup ด้วย OIDC auth, lazy loading, PWA
- Component architecture: forms, validation, virtualized lists
- State management ด้วย CartState + localStorage
- JavaScript interop สองทิศทาง
- SignalR integration สำหรับ real-time order updates
- Performance: `@key`, `ShouldRender`, lazy assembly loading
- Service worker + offline support
