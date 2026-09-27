# Part 26: Blazor

## Steps 701-750: การสร้างเว็บแอปด้วย Blazor (.NET 9)

---

## Step 701: Blazor คืออะไร และโมเดลการทำงาน

```bash
# สร้าง Blazor Web App (.NET 9 - Unified)
dotnet new blazor -n MyBlazorApp
cd MyBlazorApp

# Blazor WebAssembly (Standalone)
dotnet new blazorwasm -n MyBlazorWasm
cd MyBlazorWasm

# โครงสร้าง Blazor Web App
# Components/        - Razor Components (.razor)
# Components/Layout/ - Layout components
# Components/Pages/  - Page components
# wwwroot/           - Static assets
# Program.cs         - Entry point
```

```csharp
// Program.cs - Blazor Web App (.NET 9)
var builder = WebApplication.CreateBuilder(args);

// Add services for Blazor
builder.Services.AddRazorComponents()
    .AddInteractiveServerComponents()    // Blazor Server
    .AddInteractiveWebAssemblyComponents(); // Blazor WASM

builder.Services.AddScoped<HttpClient>(sp =>
    new HttpClient { BaseAddress = new Uri("https://localhost:7001") });

var app = builder.Build();

app.UseStaticFiles();
app.UseAntiforgery();

app.MapRazorComponents<App>()
    .AddInteractiveServerRenderMode()
    .AddInteractiveWebAssemblyRenderMode()
    .AddAdditionalAssemblies(typeof(MyBlazorApp.Client._Imports).Assembly);

app.Run();
```

```razor
@* Components/App.razor *@
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <base href="/" />
    <link rel="stylesheet" href="bootstrap/bootstrap.min.css" />
    <link rel="stylesheet" href="app.css" />
    <HeadOutlet />
</head>
<body>
    <Routes />
    <script src="_framework/blazor.web.js"></script>
</body>
</html>
```

---

## Step 702: Component พื้นฐาน

```razor
@* Components/Pages/Counter.razor *@
@page "/counter"
@rendermode InteractiveServer

<PageTitle>Counter</PageTitle>

<h1>Counter</h1>

<p role="status">Current count: <strong>@_count</strong></p>

<div class="d-flex gap-2">
    <button class="btn btn-primary" @onclick="IncrementCount">Click me</button>
    <button class="btn btn-secondary" @onclick="DecrementCount" disabled="@(_count <= 0)">Decrement</button>
    <button class="btn btn-danger" @onclick="Reset">Reset</button>
</div>

@code {
    private int _count = 0;

    private void IncrementCount() => _count++;
    private void DecrementCount() => _count--;
    private void Reset() => _count = 0;
}
```

```razor
@* Components/HelloWorld.razor *@
<div class="greeting @CssClass">
    <h2>Hello, @Name!</h2>
    @if (!string.IsNullOrEmpty(Message))
    {
        <p>@Message</p>
    }
    @ChildContent
</div>

@code {
    [Parameter] public string Name { get; set; } = "World";
    [Parameter] public string? Message { get; set; }
    [Parameter] public string CssClass { get; set; } = "";
    [Parameter] public RenderFragment? ChildContent { get; set; }
}
```

```razor
@* Usage of HelloWorld component *@
@page "/hello"

<HelloWorld Name="Alice" CssClass="highlight">
    <p>This is child content passed to the component.</p>
</HelloWorld>

<HelloWorld Name="@_userName" Message="@_greeting" />

@code {
    private string _userName = "Bob";
    private string _greeting = "Welcome to Blazor!";
}
```

---

## Step 703: Data Binding

```razor
@* Components/Pages/DataBinding.razor *@
@page "/binding"
@rendermode InteractiveServer

<h1>Data Binding Demo</h1>

@* One-way binding *@
<p>Current time: @_currentTime.ToString("HH:mm:ss")</p>

@* Two-way binding with @bind *@
<div class="mb-3">
    <label>Name:</label>
    <input @bind="_name" class="form-control" />
    <p>Hello, @_name!</p>
</div>

@* Two-way with event binding *@
<div class="mb-3">
    <label>Search (live):</label>
    <input @bind="_search" @bind:event="oninput" class="form-control" />
    <p>Searching for: <em>@_search</em></p>
</div>

@* Checkbox binding *@
<div class="form-check">
    <input type="checkbox" @bind="_isActive" class="form-check-input" id="active" />
    <label class="form-check-label" for="active">Is Active: @_isActive</label>
</div>

@* Select binding *@
<select @bind="_selectedColor" class="form-select">
    <option value="">Select color...</option>
    @foreach (var color in _colors)
    {
        <option value="@color">@color</option>
    }
</select>
<p>Selected: @_selectedColor</p>

@* Number binding *@
<input type="range" @bind="_slider" min="0" max="100" class="form-range" />
<p>Value: @_slider</p>

@code {
    private DateTime _currentTime = DateTime.Now;
    private string _name = string.Empty;
    private string _search = string.Empty;
    private bool _isActive = true;
    private string _selectedColor = string.Empty;
    private int _slider = 50;
    private readonly string[] _colors = ["Red", "Green", "Blue", "Yellow"];

    protected override void OnInitialized()
    {
        // Timer to update time
        var timer = new System.Timers.Timer(1000);
        timer.Elapsed += (_, _) =>
        {
            _currentTime = DateTime.Now;
            InvokeAsync(StateHasChanged);
        };
        timer.Start();
    }
}
```

---

## Step 704: Event Handling

```razor
@* Components/Pages/Events.razor *@
@page "/events"
@rendermode InteractiveServer

<h1>Event Handling</h1>

@* Basic click event *@
<button @onclick="HandleClick">Click Me</button>
<button @onclick="() => HandleClickWithArgs(42, "hello")">Click with Args</button>

@* Mouse events *@
<div @onmousemove="HandleMouseMove" 
     @onmouseenter="() => _isHovered = true"
     @onmouseleave="() => _isHovered = false"
     class="border p-3 @(_isHovered ? "bg-light" : "")">
    Mouse position: @_mouseX, @_mouseY
</div>

@* Keyboard events *@
<input @onkeydown="HandleKeyDown"
       @onkeyup="HandleKeyUp"
       placeholder="Type something..." 
       class="form-control" />
<p>Last key: @_lastKey</p>

@* Form submit event *@
<form @onsubmit="HandleSubmit" @onsubmit:preventDefault>
    <input @bind="_formInput" class="form-control" />
    <button type="submit" class="btn btn-primary">Submit</button>
</form>

@* Event with stopPropagation *@
<div @onclick="HandleOuterClick">
    Outer div clicked: @_outerCount
    <button @onclick="HandleInnerClick" @onclick:stopPropagation>
        Inner button: @_innerCount
    </button>
</div>

@* Lambda with async *@
<button @onclick="async _ => await LoadDataAsync()">Load Data Async</button>

@code {
    private double _mouseX, _mouseY;
    private bool _isHovered;
    private string _lastKey = string.Empty;
    private string _formInput = string.Empty;
    private int _outerCount, _innerCount;

    private void HandleClick() => Console.WriteLine("Clicked!");
    
    private void HandleClickWithArgs(int number, string text)
        => Console.WriteLine($"Clicked with {number} and {text}");

    private void HandleMouseMove(MouseEventArgs e)
    {
        _mouseX = e.ClientX;
        _mouseY = e.ClientY;
    }

    private void HandleKeyDown(KeyboardEventArgs e)
        => _lastKey = $"Down: {e.Key}";

    private void HandleKeyUp(KeyboardEventArgs e)
        => _lastKey = $"Up: {e.Key}";

    private void HandleSubmit()
        => Console.WriteLine($"Submitted: {_formInput}");

    private void HandleOuterClick() => _outerCount++;
    private void HandleInnerClick() => _innerCount++;

    private async Task LoadDataAsync()
    {
        await Task.Delay(1000);
        Console.WriteLine("Data loaded");
    }
}
```

---

## Step 705: Component Lifecycle

```razor
@* Components/LifecycleDemo.razor *@
@implements IDisposable
@implements IAsyncDisposable

<div>
    <h3>Lifecycle Demo</h3>
    <p>Status: @_status</p>
    <p>Render count: @_renderCount</p>
</div>

@code {
    [Parameter] public int Id { get; set; }
    
    private string _status = "Initializing";
    private int _renderCount = 0;
    private System.Timers.Timer? _timer;

    // 1. Called once when component is first created
    protected override void OnInitialized()
    {
        _status = "Initialized";
        _timer = new System.Timers.Timer(5000);
        _timer.Elapsed += OnTimerElapsed;
        _timer.Start();
    }

    // 2. Async version - awaited before rendering
    protected override async Task OnInitializedAsync()
    {
        await LoadInitialDataAsync();
        _status = "Data Loaded";
    }

    // 3. Called when parameters change (including first render)
    protected override void OnParametersSet()
    {
        _status = $"Parameters set (Id={Id})";
    }

    // 4. Async version
    protected override async Task OnParametersSetAsync()
    {
        if (Id > 0)
            await LoadByIdAsync(Id);
    }

    // 5. Called after each render
    protected override void OnAfterRender(bool firstRender)
    {
        _renderCount++;
        if (firstRender)
        {
            // DOM is ready - can call JS here
        }
    }

    // 6. Async version
    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender)
        {
            // await jsRuntime.InvokeVoidAsync("initializeComponent");
        }
    }

    // 7. Prevent re-renders when state hasn't changed
    protected override bool ShouldRender()
    {
        // Return false to skip rendering
        return true;
    }

    private async Task LoadInitialDataAsync() => await Task.Delay(100);
    private async Task LoadByIdAsync(int id) => await Task.Delay(100);

    private void OnTimerElapsed(object? sender, System.Timers.ElapsedEventArgs e)
    {
        InvokeAsync(StateHasChanged);
    }

    // 8. Cleanup
    public void Dispose()
    {
        _timer?.Stop();
        _timer?.Dispose();
    }

    public async ValueTask DisposeAsync()
    {
        _timer?.Stop();
        _timer?.Dispose();
        await ValueTask.CompletedTask;
    }
}
```

---

## Step 706: Component Communication

```razor
@* Components/ParentComponent.razor *@
<h2>Parent Component</h2>
<p>Message from child: @_childMessage</p>

<ChildComponent 
    Title="Hello from Parent"
    Count="@_count"
    OnCountChanged="HandleCountChanged"
    OnMessageSent="HandleMessage" />

<button @onclick="() => _count++">Increment from Parent</button>

@code {
    private int _count = 0;
    private string _childMessage = string.Empty;

    private void HandleCountChanged(int newCount) => _count = newCount;
    private void HandleMessage(string message) => _childMessage = message;
}
```

```razor
@* Components/ChildComponent.razor *@
<div class="child-component border p-3">
    <h3>Child: @Title</h3>
    <p>Count: @Count</p>
    
    <button @onclick="IncrementAndNotify">+1 (notify parent)</button>
    <button @onclick="SendMessage">Send Message to Parent</button>
</div>

@code {
    [Parameter] public string Title { get; set; } = string.Empty;
    [Parameter] public int Count { get; set; }
    [Parameter] public EventCallback<int> OnCountChanged { get; set; }
    [Parameter] public EventCallback<string> OnMessageSent { get; set; }

    private async Task IncrementAndNotify()
    {
        await OnCountChanged.InvokeAsync(Count + 1);
    }

    private async Task SendMessage()
    {
        await OnMessageSent.InvokeAsync($"Hello from {Title} at {DateTime.Now:HH:mm:ss}");
    }
}
```

```razor
@* Cascading Parameters *@

@* CascadingValue Provider *@
<CascadingValue Value="@_theme" Name="Theme">
    <CascadingValue Value="@_currentUser">
        @Body
    </CascadingValue>
</CascadingValue>

@code {
    private string _theme = "dark";
    private UserInfo _currentUser = new("Alice", "Admin");
}

@* Consuming CascadingParameter *@
<div class="themed @Theme">
    <p>Hello, @CurrentUser?.Name (@CurrentUser?.Role)</p>
</div>

@code {
    [CascadingParameter(Name = "Theme")] 
    public string Theme { get; set; } = "light";
    
    [CascadingParameter] 
    public UserInfo? CurrentUser { get; set; }
}
```

---

## Step 707: State Management

```csharp
// Services/AppStateService.cs
namespace MyBlazorApp.Services;

public class AppStateService
{
    private string _userName = string.Empty;
    private List<CartItem> _cartItems = [];
    private bool _isDarkMode = false;

    public string UserName
    {
        get => _userName;
        set
        {
            _userName = value;
            NotifyStateChanged();
        }
    }

    public IReadOnlyList<CartItem> CartItems => _cartItems.AsReadOnly();
    public int CartCount => _cartItems.Sum(i => i.Quantity);
    public decimal CartTotal => _cartItems.Sum(i => i.Price * i.Quantity);
    
    public bool IsDarkMode
    {
        get => _isDarkMode;
        set
        {
            _isDarkMode = value;
            NotifyStateChanged();
        }
    }

    public event Action? OnChange;

    public void AddToCart(Product product, int quantity = 1)
    {
        var existing = _cartItems.FirstOrDefault(i => i.ProductId == product.Id);
        if (existing is not null)
            existing.Quantity += quantity;
        else
            _cartItems.Add(new CartItem(product.Id, product.Name, product.Price, quantity));
        
        NotifyStateChanged();
    }

    public void RemoveFromCart(int productId)
    {
        _cartItems.RemoveAll(i => i.ProductId == productId);
        NotifyStateChanged();
    }

    public void ClearCart()
    {
        _cartItems.Clear();
        NotifyStateChanged();
    }

    private void NotifyStateChanged() => OnChange?.Invoke();
}

public record CartItem(int ProductId, string Name, decimal Price, int Quantity)
{
    public int Quantity { get; set; } = Quantity;
}
```

```razor
@* Components/CartBadge.razor *@
@inject AppStateService AppState
@implements IDisposable

<a href="/cart" class="btn btn-outline-primary position-relative">
    Cart
    @if (AppState.CartCount > 0)
    {
        <span class="position-absolute top-0 start-100 translate-middle badge rounded-pill bg-danger">
            @AppState.CartCount
        </span>
    }
</a>

@code {
    protected override void OnInitialized()
    {
        AppState.OnChange += OnStateChanged;
    }

    private void OnStateChanged() => InvokeAsync(StateHasChanged);

    public void Dispose() => AppState.OnChange -= OnStateChanged;
}
```

---

## Step 708: Forms และ Validation

```razor
@* Components/Pages/ProductForm.razor *@
@page "/products/create"
@rendermode InteractiveServer
@inject IProductService ProductService
@inject NavigationManager Navigation

<h1>Create Product</h1>

<EditForm Model="@_model" OnValidSubmit="HandleSubmit" FormName="create-product">
    <DataAnnotationsValidator />
    <ValidationSummary class="alert alert-danger" />

    <div class="mb-3">
        <label class="form-label">Name</label>
        <InputText @bind-Value="_model.Name" class="form-control" />
        <ValidationMessage For="() => _model.Name" class="text-danger" />
    </div>

    <div class="mb-3">
        <label class="form-label">Description</label>
        <InputTextArea @bind-Value="_model.Description" class="form-control" rows="3" />
        <ValidationMessage For="() => _model.Description" />
    </div>

    <div class="row">
        <div class="col-md-6 mb-3">
            <label class="form-label">Price</label>
            <InputNumber @bind-Value="_model.Price" class="form-control" />
            <ValidationMessage For="() => _model.Price" />
        </div>
        <div class="col-md-6 mb-3">
            <label class="form-label">Stock</label>
            <InputNumber @bind-Value="_model.Stock" class="form-control" />
            <ValidationMessage For="() => _model.Stock" />
        </div>
    </div>

    <div class="mb-3">
        <label class="form-label">Category</label>
        <InputSelect @bind-Value="_model.CategoryId" class="form-select">
            <option value="0">-- Select --</option>
            @foreach (var cat in _categories)
            {
                <option value="@cat.Id">@cat.Name</option>
            }
        </InputSelect>
        <ValidationMessage For="() => _model.CategoryId" />
    </div>

    <div class="mb-3">
        <label class="form-label">Release Date</label>
        <InputDate @bind-Value="_model.ReleaseDate" class="form-control" />
        <ValidationMessage For="() => _model.ReleaseDate" />
    </div>

    <div class="form-check mb-3">
        <InputCheckbox @bind-Value="_model.IsActive" class="form-check-input" id="active" />
        <label class="form-check-label" for="active">Active</label>
    </div>

    <button type="submit" class="btn btn-primary" disabled="@_isSubmitting">
        @if (_isSubmitting)
        {
            <span class="spinner-border spinner-border-sm"></span>
        }
        Create Product
    </button>
    <a href="/products" class="btn btn-secondary">Cancel</a>
</EditForm>

@code {
    private CreateProductModel _model = new();
    private List<Category> _categories = [];
    private bool _isSubmitting = false;

    protected override async Task OnInitializedAsync()
    {
        _categories = await ProductService.GetCategoriesAsync();
    }

    private async Task HandleSubmit()
    {
        _isSubmitting = true;
        try
        {
            await ProductService.CreateAsync(_model);
            Navigation.NavigateTo("/products");
        }
        finally
        {
            _isSubmitting = false;
        }
    }
}
```

```csharp
// Models/CreateProductModel.cs
using System.ComponentModel.DataAnnotations;

public class CreateProductModel
{
    [Required(ErrorMessage = "Name is required")]
    [StringLength(200, MinimumLength = 2)]
    public string Name { get; set; } = string.Empty;

    [StringLength(1000)]
    public string Description { get; set; } = string.Empty;

    [Required]
    [Range(0.01, 999999)]
    public decimal Price { get; set; }

    [Range(0, int.MaxValue)]
    public int Stock { get; set; }

    [Required(ErrorMessage = "Please select a category")]
    [Range(1, int.MaxValue, ErrorMessage = "Please select a category")]
    public int CategoryId { get; set; }

    public DateTime ReleaseDate { get; set; } = DateTime.Today;
    public bool IsActive { get; set; } = true;
}
```

---

## Step 709: JavaScript Interop

```razor
@* Components/Pages/JsInterop.razor *@
@page "/js-interop"
@rendermode InteractiveServer
@inject IJSRuntime JSRuntime

<h1>JavaScript Interop</h1>

<button @onclick="ShowAlert">Show Alert</button>
<button @onclick="GetWindowSize">Get Window Size</button>
<button @onclick="SetLocalStorage">Save to LocalStorage</button>
<button @onclick="GetFromLocalStorage">Load from LocalStorage</button>
<button @onclick="FocusInput">Focus Input</button>

<p>Window: @_windowSize</p>
<p>Storage value: @_storageValue</p>

<input @ref="_inputRef" class="form-control" placeholder="Click 'Focus Input'" />

@code {
    private string _windowSize = string.Empty;
    private string _storageValue = string.Empty;
    private ElementReference _inputRef;

    private async Task ShowAlert()
    {
        await JSRuntime.InvokeVoidAsync("alert", "Hello from Blazor!");
    }

    private async Task GetWindowSize()
    {
        var size = await JSRuntime.InvokeAsync<WindowSize>("getWindowSize");
        _windowSize = $"{size.Width}x{size.Height}";
    }

    private async Task SetLocalStorage()
    {
        await JSRuntime.InvokeVoidAsync("localStorage.setItem", "key", "Blazor value");
    }

    private async Task GetFromLocalStorage()
    {
        _storageValue = await JSRuntime.InvokeAsync<string>("localStorage.getItem", "key") ?? "null";
    }

    private async Task FocusInput()
    {
        await _inputRef.FocusAsync();
    }

    private record WindowSize(double Width, double Height);
}
```

```javascript
// wwwroot/js/app.js
window.getWindowSize = () => ({
    width: window.innerWidth,
    height: window.innerHeight
});

window.scrollToTop = () => window.scrollTo({ top: 0, behavior: 'smooth' });

window.downloadFile = (filename, content) => {
    const blob = new Blob([content], { type: 'text/plain' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = filename;
    a.click();
    URL.revokeObjectURL(url);
};

// .NET -> JS -> .NET callback
window.setupResizeObserver = (dotNetRef, elementId) => {
    const element = document.getElementById(elementId);
    const observer = new ResizeObserver(entries => {
        for (const entry of entries) {
            dotNetRef.invokeMethodAsync('OnResize', entry.contentRect.width, entry.contentRect.height);
        }
    });
    observer.observe(element);
    return observer;
};
```

```razor
@* Using JS Object Reference *@
@implements IAsyncDisposable
@inject IJSRuntime JSRuntime

<div id="resizable-@_id">Resizable content</div>

@code {
    private string _id = Guid.NewGuid().ToString("N")[..8];
    private IJSObjectReference? _observer;
    private DotNetObjectReference<JsInteropPage>? _dotNetRef;

    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender)
        {
            _dotNetRef = DotNetObjectReference.Create(this);
            _observer = await JSRuntime.InvokeAsync<IJSObjectReference>(
                "setupResizeObserver", _dotNetRef, $"resizable-{_id}");
        }
    }

    [JSInvokable]
    public void OnResize(double width, double height)
    {
        Console.WriteLine($"Resized to {width}x{height}");
        InvokeAsync(StateHasChanged);
    }

    public async ValueTask DisposeAsync()
    {
        if (_observer is not null)
            await _observer.DisposeAsync();
        _dotNetRef?.Dispose();
    }
}
```

---

## Step 710: Routing ใน Blazor

```razor
@* Components/Pages/Products/Details.razor *@
@page "/products/{Id:int}"
@page "/products/{Id:int}/details"

<h1>Product @Id</h1>
<p>Category: @Category</p>

@code {
    [Parameter] public int Id { get; set; }
    
    [SupplyParameterFromQuery(Name = "category")]
    public string? Category { get; set; }
}

@* With optional parameter *@
@page "/products/{Id:int?}"
@code {
    [Parameter] public int? Id { get; set; }
}

@* Catch-all route *@
@page "/docs/{*path}"
@code {
    [Parameter] public string? Path { get; set; }
}
```

```razor
@* Components/Layout/NavMenu.razor *@
<nav class="sidebar">
    <NavLink href="/" Match="NavLinkMatch.All" class="nav-link" ActiveClass="active">
        Home
    </NavLink>
    <NavLink href="/products" class="nav-link" ActiveClass="active">
        Products
    </NavLink>
    <NavLink href="/about" class="nav-link" ActiveClass="active">
        About
    </NavLink>
</nav>

@* Programmatic Navigation *@
@inject NavigationManager Navigation

@code {
    private void GoToProduct(int id)
    {
        Navigation.NavigateTo($"/products/{id}");
    }

    private void GoWithQueryString()
    {
        var uri = Navigation.GetUriWithQueryParameter("page", 2);
        Navigation.NavigateTo(uri);
    }

    // Listen for location changes
    protected override void OnInitialized()
    {
        Navigation.LocationChanged += OnLocationChanged;
    }

    private void OnLocationChanged(object? sender, LocationChangedEventArgs e)
    {
        Console.WriteLine($"Navigated to: {e.Location}");
        InvokeAsync(StateHasChanged);
    }

    public void Dispose()
    {
        Navigation.LocationChanged -= OnLocationChanged;
    }
}
```

---

## Step 711: Dependency Injection ใน Blazor

```csharp
// Program.cs - Service Registration
builder.Services.AddSingleton<WeatherService>();
builder.Services.AddScoped<CartService>();
builder.Services.AddTransient<NotificationService>();

// Scoped in Blazor Server = per circuit (user connection)
// Scoped in Blazor WASM = per lifetime of app (singleton-like)
```

```razor
@* Injecting services in components *@
@inject IProductService ProductService
@inject NavigationManager Navigation
@inject ILogger<ProductsPage> Logger
@inject IJSRuntime JS
@inject IOptions<AppSettings> AppSettings
@inject AppStateService AppState

@* Or using [Inject] attribute in @code *@
@code {
    [Inject] private IProductService ProductService { get; set; } = default!;
    [Inject] private ILogger<ProductsPage> Logger { get; set; } = default!;
}
```

---

## Step 712: HTTP Calls ใน Blazor

```csharp
// Services/ProductApiService.cs
using System.Net.Http.Json;

namespace MyBlazorApp.Services;

public class ProductApiService
{
    private readonly HttpClient _http;
    private readonly ILogger<ProductApiService> _logger;

    public ProductApiService(HttpClient http, ILogger<ProductApiService> logger)
    {
        _http = http;
        _logger = logger;
    }

    public async Task<PagedResult<ProductDto>?> GetPagedAsync(int page = 1, int size = 10, string? search = null)
    {
        var url = $"api/products?page={page}&size={size}";
        if (!string.IsNullOrEmpty(search))
            url += $"&search={Uri.EscapeDataString(search)}";
        
        return await _http.GetFromJsonAsync<PagedResult<ProductDto>>(url);
    }

    public async Task<ProductDto?> GetByIdAsync(int id)
    {
        try
        {
            return await _http.GetFromJsonAsync<ProductDto>($"api/products/{id}");
        }
        catch (HttpRequestException ex) when (ex.StatusCode == System.Net.HttpStatusCode.NotFound)
        {
            return null;
        }
    }

    public async Task<ProductDto?> CreateAsync(CreateProductDto dto)
    {
        var response = await _http.PostAsJsonAsync("api/products", dto);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<ProductDto>();
    }

    public async Task UpdateAsync(int id, UpdateProductDto dto)
    {
        var response = await _http.PutAsJsonAsync($"api/products/{id}", dto);
        response.EnsureSuccessStatusCode();
    }

    public async Task DeleteAsync(int id)
    {
        var response = await _http.DeleteAsync($"api/products/{id}");
        response.EnsureSuccessStatusCode();
    }
}
```

```razor
@* Components/Pages/Products/Index.razor *@
@page "/products"
@rendermode InteractiveServer
@inject ProductApiService ProductService

<h1>Products</h1>

@if (_loading)
{
    <div class="text-center">
        <div class="spinner-border" role="status">
            <span class="visually-hidden">Loading...</span>
        </div>
    </div>
}
else if (_error is not null)
{
    <div class="alert alert-danger">@_error</div>
}
else
{
    <div class="row">
        @foreach (var product in _products)
        {
            <div class="col-md-4">
                <ProductCard Product="product" OnDelete="DeleteProduct" />
            </div>
        }
    </div>
    
    @* Pagination *@
    <Pagination CurrentPage="_page" TotalPages="_totalPages" OnPageChanged="LoadPage" />
}

@code {
    private List<ProductDto> _products = [];
    private int _page = 1, _totalPages = 1;
    private bool _loading = true;
    private string? _error;

    protected override async Task OnInitializedAsync()
    {
        await LoadPage(1);
    }

    private async Task LoadPage(int page)
    {
        _loading = true;
        _error = null;
        
        try
        {
            var result = await ProductService.GetPagedAsync(page);
            _products = result?.Items.ToList() ?? [];
            _totalPages = result?.TotalPages ?? 1;
            _page = page;
        }
        catch (Exception ex)
        {
            _error = "Failed to load products. Please try again.";
        }
        finally
        {
            _loading = false;
        }
    }

    private async Task DeleteProduct(int id)
    {
        await ProductService.DeleteAsync(id);
        await LoadPage(_page);
    }
}
```

---

## Step 713: Render Modes (.NET 8+)

```razor
@* Static SSR - no interactivity, fastest *@
@page "/static-page"
@* No @rendermode = Static SSR by default *@

<h1>Static Page (SSR)</h1>
<p>This renders on the server and sends HTML to the browser</p>

@* Interactive Server - SignalR connection *@
@page "/server-page"
@rendermode InteractiveServer

<button @onclick="() => _count++">Count: @_count</button>
@code { private int _count = 0; }

@* Interactive WebAssembly - runs in browser *@
@page "/wasm-page"
@rendermode InteractiveWebAssembly

<button @onclick="() => _count++">Count: @_count</button>
@code { private int _count = 0; }

@* Auto - starts as Server, switches to WASM when loaded *@
@page "/auto-page"
@rendermode InteractiveAuto

<button @onclick="() => _count++">Count: @_count</button>
@code { private int _count = 0; }
```

```csharp
// Per-component render mode
// In App.razor or parent component:
<MyComponent @rendermode="InteractiveServer" />
<MyComponent @rendermode="new InteractiveServerRenderMode(prerender: false)" />
<MyComponent @rendermode="InteractiveWebAssembly" />

// Programmatic render mode
@code {
    private IComponentRenderMode? _renderMode = InteractiveServer.Instance;
    
    // Can switch render mode based on condition
    private void SwitchToWasm() 
    {
        _renderMode = InteractiveWebAssembly.Instance;
    }
}
```

---

## Step 714: Streaming Rendering

```razor
@* Streaming rendering - render immediately with placeholders *@
@page "/streaming"
@attribute [StreamRendering]

<h1>Streaming Demo</h1>

@if (_forecast is null)
{
    <p>Loading weather data...</p>
}
else
{
    <table class="table">
        <thead>
            <tr><th>Date</th><th>Temp (C)</th><th>Summary</th></tr>
        </thead>
        <tbody>
            @foreach (var f in _forecast)
            {
                <tr>
                    <td>@f.Date.ToShortDateString()</td>
                    <td>@f.TemperatureC</td>
                    <td>@f.Summary</td>
                </tr>
            }
        </tbody>
    </table>
}

@code {
    private WeatherForecast[]? _forecast;

    protected override async Task OnInitializedAsync()
    {
        // This delay is visible to the user via streaming
        await Task.Delay(2000);
        _forecast = GenerateForecast();
    }

    private WeatherForecast[] GenerateForecast()
    {
        var rng = new Random();
        return Enumerable.Range(1, 5).Select(i => new WeatherForecast
        {
            Date = DateOnly.FromDateTime(DateTime.Now.AddDays(i)),
            TemperatureC = rng.Next(-20, 55),
            Summary = "Sunny"
        }).ToArray();
    }
}
```

---

## Step 715: Blazor Components Library

```csharp
// Creating a reusable component library
// MyBlazorComponents.csproj
<Project Sdk="Microsoft.NET.Sdk.Razor">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <AddRazorSupportForMvc>true</AddRazorSupportForMvc>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.Components.Web" Version="9.*" />
  </ItemGroup>
</Project>
```

```razor
@* Components/DataTable.razor in component library *@
@typeparam TItem

<table class="table @TableClass">
    <thead class="table-@HeaderStyle">
        <tr>
            @foreach (var col in Columns)
            {
                <th @onclick="() => SortBy(col)" style="cursor: pointer">
                    @col.Header
                    @if (_sortColumn == col.Property)
                    {
                        <span>@(_sortAscending ? "↑" : "↓")</span>
                    }
                </th>
            }
            @if (ShowActions)
            {
                <th>Actions</th>
            }
        </tr>
    </thead>
    <tbody>
        @foreach (var item in SortedItems)
        {
            <tr>
                @foreach (var col in Columns)
                {
                    <td>@col.Render(item)</td>
                }
                @if (ShowActions && RowActions is not null)
                {
                    <td>@RowActions(item)</td>
                }
            </tr>
        }
        @if (!SortedItems.Any())
        {
            <tr>
                <td colspan="@(Columns.Count + (ShowActions ? 1 : 0))" class="text-center text-muted">
                    @EmptyMessage
                </td>
            </tr>
        }
    </tbody>
</table>

@code {
    [Parameter, EditorRequired] 
    public IEnumerable<TItem> Items { get; set; } = [];
    
    [Parameter, EditorRequired]
    public List<TableColumn<TItem>> Columns { get; set; } = [];
    
    [Parameter] public RenderFragment<TItem>? RowActions { get; set; }
    [Parameter] public bool ShowActions => RowActions is not null;
    [Parameter] public string TableClass { get; set; } = "table-striped table-hover";
    [Parameter] public string HeaderStyle { get; set; } = "dark";
    [Parameter] public string EmptyMessage { get; set; } = "No data available";

    private string? _sortColumn;
    private bool _sortAscending = true;

    private IEnumerable<TItem> SortedItems =>
        _sortColumn is null ? Items : _sortAscending
            ? Items.OrderBy(x => GetValue(x, _sortColumn))
            : Items.OrderByDescending(x => GetValue(x, _sortColumn));

    private void SortBy(TableColumn<TItem> col)
    {
        if (_sortColumn == col.Property)
            _sortAscending = !_sortAscending;
        else
        {
            _sortColumn = col.Property;
            _sortAscending = true;
        }
    }

    private object? GetValue(TItem item, string property)
        => typeof(TItem).GetProperty(property)?.GetValue(item);
}

public class TableColumn<TItem>
{
    public string Header { get; set; } = string.Empty;
    public string Property { get; set; } = string.Empty;
    public Func<TItem, string>? Format { get; set; }
    
    public string Render(TItem item)
    {
        var value = typeof(TItem).GetProperty(Property)?.GetValue(item);
        return Format is not null ? Format(item) : value?.ToString() ?? string.Empty;
    }
}
```

```razor
@* Usage of DataTable component *@
@page "/products"

<DataTable TItem="ProductDto" Items="_products">
    <Columns>
        @(new TableColumn<ProductDto> { Header = "Name", Property = "Name" })
        @(new TableColumn<ProductDto> { Header = "Price", Property = "Price", 
            Format = p => p.Price.ToString("C") })
        @(new TableColumn<ProductDto> { Header = "Stock", Property = "Stock" })
    </Columns>
    <RowActions Context="product">
        <a href="/products/@product.Id" class="btn btn-sm btn-info">View</a>
    </RowActions>
</DataTable>
```

---

## Step 716: Error Boundaries

```razor
@* ErrorBoundary usage *@
<ErrorBoundary>
    <ChildContent>
        <ProductList />
    </ChildContent>
    <ErrorContent Context="ex">
        <div class="alert alert-danger">
            <h4>Something went wrong!</h4>
            <p>@ex.Message</p>
            <button class="btn btn-sm btn-warning" @onclick="context.Recover">Try Again</button>
        </div>
    </ErrorContent>
</ErrorBoundary>

@* Custom Error Boundary *@
public class CustomErrorBoundary : ErrorBoundaryBase
{
    [Inject] private ILogger<CustomErrorBoundary> Logger { get; set; } = default!;

    protected override async Task OnErrorAsync(Exception exception)
    {
        Logger.LogError(exception, "Unhandled error in component tree");
        // Could notify an error monitoring service
        await base.OnErrorAsync(exception);
    }
}
```

---

## Step 717: Virtualization

```razor
@* Efficient rendering of large lists *@
@page "/virtual-list"
@rendermode InteractiveServer
@inject ProductApiService ProductService

<h1>Virtual Scrolling (10,000 items)</h1>

<div style="height: 600px; overflow-y: auto;" class="border">
    <Virtualize Items="@_allProducts" Context="product" ItemSize="60"
                SpacerElement="tr">
        <ItemContent>
            <div class="border-bottom p-2 d-flex align-items-center">
                <div class="flex-grow-1">
                    <strong>@product.Name</strong>
                    <small class="text-muted d-block">@product.CategoryName</small>
                </div>
                <span class="text-primary fw-bold">@product.Price.ToString("C")</span>
            </div>
        </ItemContent>
        <Placeholder>
            <div class="border-bottom p-2">
                <div class="placeholder-glow">
                    <span class="placeholder col-8"></span>
                    <span class="placeholder col-4"></span>
                </div>
            </div>
        </Placeholder>
    </Virtualize>
</div>

@* Virtualize with async data loading *@
<Virtualize Context="product" 
             ItemsProvider="LoadProductsAsync"
             ItemSize="60">
    <ItemContent>
        <div>@product.Name</div>
    </ItemContent>
</Virtualize>

@code {
    private List<ProductDto> _allProducts = [];

    protected override async Task OnInitializedAsync()
    {
        // Load all 10,000 items - Virtualize only renders visible ones
        _allProducts = await ProductService.GetAllAsync();
    }

    private async ValueTask<ItemsProviderResult<ProductDto>> LoadProductsAsync(
        ItemsProviderRequest request)
    {
        var result = await ProductService.GetPagedAsync(
            page: request.StartIndex / 20 + 1,
            size: request.Count);
        
        return new ItemsProviderResult<ProductDto>(
            result?.Items ?? [],
            result?.TotalCount ?? 0);
    }
}
```

---

## Step 718: Authentication ใน Blazor

```csharp
// Services/AuthStateProvider.cs
using Microsoft.AspNetCore.Components.Authorization;
using System.Security.Claims;
using System.IdentityModel.Tokens.Jwt;

namespace MyBlazorApp.Services;

public class JwtAuthStateProvider : AuthenticationStateProvider
{
    private readonly ILocalStorageService _localStorage;
    private readonly HttpClient _http;

    public JwtAuthStateProvider(ILocalStorageService localStorage, HttpClient http)
    {
        _localStorage = localStorage;
        _http = http;
    }

    public override async Task<AuthenticationState> GetAuthenticationStateAsync()
    {
        var token = await _localStorage.GetItemAsync<string>("authToken");

        if (string.IsNullOrEmpty(token))
            return new AuthenticationState(new ClaimsPrincipal(new ClaimsIdentity()));

        _http.DefaultRequestHeaders.Authorization =
            new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", token);

        var claims = ParseTokenClaims(token);
        var identity = new ClaimsIdentity(claims, "jwt");
        return new AuthenticationState(new ClaimsPrincipal(identity));
    }

    public void NotifyUserAuthenticated(string token)
    {
        var claims = ParseTokenClaims(token);
        var identity = new ClaimsIdentity(claims, "jwt");
        var user = new ClaimsPrincipal(identity);
        NotifyAuthenticationStateChanged(Task.FromResult(new AuthenticationState(user)));
    }

    public void NotifyUserLoggedOut()
    {
        var anonymous = new ClaimsPrincipal(new ClaimsIdentity());
        NotifyAuthenticationStateChanged(Task.FromResult(new AuthenticationState(anonymous)));
    }

    private static IEnumerable<Claim> ParseTokenClaims(string token)
    {
        var handler = new JwtSecurityTokenHandler();
        var jwt = handler.ReadJwtToken(token);
        return jwt.Claims;
    }
}
```

```razor
@* Components/Pages/Login.razor *@
@page "/login"
@inject IAuthService AuthService
@inject NavigationManager Navigation

<h1>Login</h1>

<EditForm Model="_model" OnValidSubmit="HandleLogin">
    <DataAnnotationsValidator />
    
    @if (_error is not null)
    {
        <div class="alert alert-danger">@_error</div>
    }
    
    <div class="mb-3">
        <label>Email</label>
        <InputText @bind-Value="_model.Email" class="form-control" />
        <ValidationMessage For="() => _model.Email" />
    </div>
    
    <div class="mb-3">
        <label>Password</label>
        <InputText @bind-Value="_model.Password" type="password" class="form-control" />
    </div>
    
    <button type="submit" class="btn btn-primary" disabled="@_loading">
        @(_loading ? "Logging in..." : "Login")
    </button>
</EditForm>

@* Protected content *@
<AuthorizeView>
    <Authorized>
        <p>Welcome, @context.User.Identity?.Name!</p>
    </Authorized>
    <NotAuthorized>
        <p>Please login to continue.</p>
    </NotAuthorized>
</AuthorizeView>

@code {
    private LoginModel _model = new();
    private bool _loading = false;
    private string? _error;

    private async Task HandleLogin()
    {
        _loading = true;
        _error = null;
        
        try
        {
            var result = await AuthService.LoginAsync(_model.Email, _model.Password);
            if (result.Success)
                Navigation.NavigateTo("/");
            else
                _error = result.Error ?? "Login failed";
        }
        finally
        {
            _loading = false;
        }
    }
}
```

```razor
@* Route Authorization *@
@page "/admin"
@attribute [Authorize(Roles = "Admin")]

<h1>Admin Dashboard</h1>

@* Or with AuthorizeView *@
<AuthorizeView Roles="Admin,Manager">
    <Authorized>
        <p>Admin content here</p>
    </Authorized>
    <NotAuthorized>
        <p>Access denied.</p>
    </NotAuthorized>
    <Authorizing>
        <p>Checking permissions...</p>
    </Authorizing>
</AuthorizeView>

@* CascadingAuthenticationState in App.razor *@
@* 
<CascadingAuthenticationState>
    <Router AppAssembly="typeof(App).Assembly">
        <Found Context="routeData">
            <AuthorizeRouteView RouteData="routeData" DefaultLayout="typeof(MainLayout)">
                <NotAuthorized>
                    <h3>Sorry, you're not authorized to reach this page.</h3>
                </NotAuthorized>
            </AuthorizeRouteView>
        </Found>
    </Router>
</CascadingAuthenticationState>
*@
```

---

## Step 719: Real-time ด้วย SignalR ใน Blazor

```csharp
// Hubs/ChatHub.cs
using Microsoft.AspNetCore.SignalR;

public class ChatHub : Hub
{
    public async Task SendMessage(string user, string message)
    {
        await Clients.All.SendAsync("ReceiveMessage", user, message, DateTime.UtcNow);
    }

    public async Task JoinRoom(string roomName)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, roomName);
        await Clients.Group(roomName).SendAsync("UserJoined", Context.User?.Identity?.Name);
    }
}
```

```razor
@* Components/Pages/Chat.razor *@
@page "/chat"
@rendermode InteractiveServer
@inject NavigationManager Navigation

<h1>Real-time Chat</h1>

<div class="mb-3">
    <div class="border rounded p-2" style="height: 300px; overflow-y: auto;" id="chat-box">
        @foreach (var msg in _messages)
        {
            <div class="mb-1">
                <span class="text-primary fw-bold">@msg.User:</span>
                <span>@msg.Text</span>
                <small class="text-muted">@msg.Time.ToString("HH:mm")</small>
            </div>
        }
    </div>
</div>

<div class="input-group">
    <input @bind="_messageInput" @onkeydown="HandleKeyDown"
           class="form-control" placeholder="Type a message..." />
    <button @onclick="SendMessage" class="btn btn-primary"
            disabled="@(!_hubConnection?.State.Equals(HubConnectionState.Connected) ?? true)">
        Send
    </button>
</div>

<p>Status: @_connectionStatus</p>

@code {
    private HubConnection? _hubConnection;
    private List<ChatMessage> _messages = [];
    private string _messageInput = string.Empty;
    private string _connectionStatus = "Disconnected";

    protected override async Task OnInitializedAsync()
    {
        _hubConnection = new HubConnectionBuilder()
            .WithUrl(Navigation.ToAbsoluteUri("/chathub"))
            .WithAutomaticReconnect()
            .Build();

        _hubConnection.On<string, string, DateTime>("ReceiveMessage", 
            (user, message, time) =>
            {
                _messages.Add(new ChatMessage(user, message, time));
                InvokeAsync(StateHasChanged);
            });

        _hubConnection.Reconnecting += error =>
        {
            _connectionStatus = "Reconnecting...";
            InvokeAsync(StateHasChanged);
            return Task.CompletedTask;
        };

        _hubConnection.Reconnected += _ =>
        {
            _connectionStatus = "Connected";
            InvokeAsync(StateHasChanged);
            return Task.CompletedTask;
        };

        await _hubConnection.StartAsync();
        _connectionStatus = "Connected";
    }

    private async Task SendMessage()
    {
        if (!string.IsNullOrWhiteSpace(_messageInput) && _hubConnection is not null)
        {
            await _hubConnection.SendAsync("SendMessage", "User", _messageInput);
            _messageInput = string.Empty;
        }
    }

    private async Task HandleKeyDown(KeyboardEventArgs e)
    {
        if (e.Key == "Enter") await SendMessage();
    }

    public async ValueTask DisposeAsync()
    {
        if (_hubConnection is not null)
            await _hubConnection.DisposeAsync();
    }

    private record ChatMessage(string User, string Text, DateTime Time);
}
```

---

## Step 720: Blazor WASM Progressive Web App (PWA)

```json
// wwwroot/manifest.json
{
    "name": "My Blazor App",
    "short_name": "BlazorApp",
    "start_url": "/",
    "display": "standalone",
    "background_color": "#ffffff",
    "theme_color": "#0d6efd",
    "icons": [
        {
            "src": "icon-192.png",
            "sizes": "192x192",
            "type": "image/png"
        },
        {
            "src": "icon-512.png",
            "sizes": "512x512",
            "type": "image/png"
        }
    ]
}
```

```javascript
// wwwroot/service-worker.js
const CACHE_NAME = 'blazor-pwa-v1';
const STATIC_ASSETS = [
    '/',
    '/index.html',
    '/_framework/blazor.webassembly.js',
    '/css/app.css'
];

self.addEventListener('install', event => {
    event.waitUntil(
        caches.open(CACHE_NAME).then(cache => cache.addAll(STATIC_ASSETS))
    );
});

self.addEventListener('fetch', event => {
    event.respondWith(
        caches.match(event.request).then(response => {
            if (response) return response;
            
            return fetch(event.request).then(networkResponse => {
                if (networkResponse.ok) {
                    const clonedResponse = networkResponse.clone();
                    caches.open(CACHE_NAME).then(cache => 
                        cache.put(event.request, clonedResponse));
                }
                return networkResponse;
            }).catch(() => {
                // Return offline page if network fails
                if (event.request.mode === 'navigate')
                    return caches.match('/offline.html');
            });
        })
    );
});
```

```csharp
// Offline data sync with IndexedDB
// Using Blazored.LocalStorage or writing JS interop
public class OfflineSyncService
{
    private readonly IJSRuntime _js;
    private readonly HttpClient _http;

    public OfflineSyncService(IJSRuntime js, HttpClient http)
    {
        _js = js;
        _http = http;
    }

    public async Task<bool> IsOnlineAsync()
        => await _js.InvokeAsync<bool>("eval", "navigator.onLine");

    public async Task SaveOfflineAsync<T>(string key, T data)
    {
        var json = JsonSerializer.Serialize(data);
        await _js.InvokeVoidAsync("indexedDB_set", key, json);
    }

    public async Task<T?> LoadOfflineAsync<T>(string key)
    {
        var json = await _js.InvokeAsync<string>("indexedDB_get", key);
        return json is null ? default : JsonSerializer.Deserialize<T>(json);
    }

    public async Task SyncWhenOnlineAsync()
    {
        var isOnline = await IsOnlineAsync();
        if (!isOnline) return;
        
        var pendingChanges = await LoadOfflineAsync<List<PendingChange>>("pending_changes");
        if (pendingChanges is null) return;
        
        foreach (var change in pendingChanges)
        {
            await _http.PostAsJsonAsync($"api/{change.Resource}", change.Data);
        }
        
        await _js.InvokeVoidAsync("indexedDB_remove", "pending_changes");
    }
}
```

---

## Step 721: State Management with Fluxor

```csharp
// Install: dotnet add package Fluxor.Blazor.Web

// Program.cs
builder.Services.AddFluxor(options =>
    options.ScanAssemblies(typeof(Program).Assembly));

// State/Products/ProductsState.cs
using Fluxor;

[FeatureState]
public record ProductsState
{
    public bool IsLoading { get; init; } = false;
    public string? ErrorMessage { get; init; }
    public ImmutableList<ProductDto> Products { get; init; } = ImmutableList<ProductDto>.Empty;
    public int TotalCount { get; init; }
    public int CurrentPage { get; init; } = 1;
}

// Actions
public record LoadProductsAction(int Page = 1, string? Search = null);
public record LoadProductsSuccessAction(IEnumerable<ProductDto> Products, int Total);
public record LoadProductsFailureAction(string Error);
public record DeleteProductAction(int Id);
public record DeleteProductSuccessAction(int Id);

// Reducers
public static class ProductsReducers
{
    [ReducerMethod]
    public static ProductsState OnLoadProducts(ProductsState state, LoadProductsAction action)
        => state with { IsLoading = true, ErrorMessage = null };

    [ReducerMethod]
    public static ProductsState OnLoadProductsSuccess(
        ProductsState state, LoadProductsSuccessAction action)
        => state with 
        { 
            IsLoading = false,
            Products = action.Products.ToImmutableList(),
            TotalCount = action.Total
        };

    [ReducerMethod]
    public static ProductsState OnLoadProductsFailure(
        ProductsState state, LoadProductsFailureAction action)
        => state with { IsLoading = false, ErrorMessage = action.Error };

    [ReducerMethod]
    public static ProductsState OnDeleteProductSuccess(
        ProductsState state, DeleteProductSuccessAction action)
        => state with
        {
            Products = state.Products.RemoveAll(p => p.Id == action.Id),
            TotalCount = state.TotalCount - 1
        };
}

// Effects
public class ProductsEffects
{
    private readonly IProductApiService _productService;

    public ProductsEffects(IProductApiService productService)
    {
        _productService = productService;
    }

    [EffectMethod]
    public async Task HandleLoadProducts(LoadProductsAction action, IDispatcher dispatcher)
    {
        try
        {
            var result = await _productService.GetPagedAsync(action.Page, search: action.Search);
            dispatcher.Dispatch(new LoadProductsSuccessAction(result!.Items, result.TotalCount));
        }
        catch (Exception ex)
        {
            dispatcher.Dispatch(new LoadProductsFailureAction(ex.Message));
        }
    }
}
```

```razor
@* Using Fluxor in Components *@
@page "/products"
@inherits Fluxor.Blazor.Web.Components.FluxorComponent
@inject IState<ProductsState> ProductsState
@inject IDispatcher Dispatcher

<h1>Products</h1>

@if (ProductsState.Value.IsLoading)
{
    <p>Loading...</p>
}
else if (ProductsState.Value.ErrorMessage is not null)
{
    <div class="alert alert-danger">@ProductsState.Value.ErrorMessage</div>
}
else
{
    @foreach (var product in ProductsState.Value.Products)
    {
        <div>
            @product.Name 
            <button @onclick="() => Delete(product.Id)">Delete</button>
        </div>
    }
}

@code {
    protected override void OnInitialized()
    {
        base.OnInitialized();
        Dispatcher.Dispatch(new LoadProductsAction());
    }

    private void Delete(int id)
        => Dispatcher.Dispatch(new DeleteProductAction(id));
}
```

---

## Step 722: Mudblazor Component Library

```bash
# Install MudBlazor
dotnet add package MudBlazor
```

```csharp
// Program.cs
builder.Services.AddMudServices();
```

```html
<!-- In App.razor or index.html -->
<link href="https://fonts.googleapis.com/css?family=Roboto:300,400,500,700&display=swap" rel="stylesheet" />
<link href="_content/MudBlazor/MudBlazor.min.css" rel="stylesheet" />
<script src="_content/MudBlazor/MudBlazor.min.js"></script>
```

```razor
@* Using MudBlazor components *@
@page "/mud-demo"
@inject ISnackbar Snackbar
@inject IDialogService DialogService

<MudText Typo="Typo.h4">MudBlazor Demo</MudText>

<MudGrid>
    <MudItem xs="12" md="6">
        <MudCard>
            <MudCardHeader>
                <CardHeaderContent>
                    <MudText Typo="Typo.h6">Product Form</MudText>
                </CardHeaderContent>
            </MudCardHeader>
            <MudCardContent>
                <MudTextField @bind-Value="_name" Label="Product Name" 
                              Required="true" RequiredError="Name is required" />
                <MudNumericField @bind-Value="_price" Label="Price" Adornment="Adornment.Start" 
                                 AdornmentText="$" Format="N2" />
                <MudSelect @bind-Value="_category" Label="Category">
                    <MudSelectItem Value="1">Electronics</MudSelectItem>
                    <MudSelectItem Value="2">Clothing</MudSelectItem>
                    <MudSelectItem Value="3">Books</MudSelectItem>
                </MudSelect>
                <MudSwitch @bind-Checked="_isActive" Label="Active" Color="Color.Primary" />
            </MudCardContent>
            <MudCardActions>
                <MudButton Variant="Variant.Filled" Color="Color.Primary" 
                           OnClick="Save">Save</MudButton>
                <MudButton Variant="Variant.Text" OnClick="Cancel">Cancel</MudButton>
            </MudCardActions>
        </MudCard>
    </MudItem>
    
    <MudItem xs="12" md="6">
        <MudDataGrid Items="@_products" Hover="true" Striped="true">
            <Columns>
                <PropertyColumn Property="p => p.Name" Title="Name" />
                <PropertyColumn Property="p => p.Price" Title="Price" Format="C" />
                <TemplateColumn Title="Actions">
                    <CellTemplate>
                        <MudIconButton Icon="@Icons.Material.Filled.Edit" 
                                       Color="Color.Warning" Size="Size.Small" />
                        <MudIconButton Icon="@Icons.Material.Filled.Delete" 
                                       Color="Color.Error" Size="Size.Small"
                                       OnClick="() => ConfirmDelete(context.Item)" />
                    </CellTemplate>
                </TemplateColumn>
            </Columns>
        </MudDataGrid>
    </MudItem>
</MudGrid>

@code {
    private string _name = string.Empty;
    private decimal _price;
    private int _category;
    private bool _isActive = true;
    private List<ProductDto> _products = [];

    private void Save()
    {
        Snackbar.Add("Product saved!", Severity.Success);
    }

    private void Cancel() { }

    private async Task ConfirmDelete(ProductDto product)
    {
        var result = await DialogService.ShowMessageBox(
            "Delete Product",
            $"Are you sure you want to delete '{product.Name}'?",
            yesText: "Delete",
            cancelText: "Cancel");

        if (result == true)
        {
            _products.Remove(product);
            Snackbar.Add($"'{product.Name}' deleted", Severity.Warning);
        }
    }
}
```

---

## Step 723: Performance Optimization ใน Blazor

```razor
@* Preventing unnecessary re-renders *@
@* Option 1: ShouldRender *@
@code {
    private bool _dataChanged = false;

    protected override bool ShouldRender()
    {
        if (_dataChanged)
        {
            _dataChanged = false;
            return true;
        }
        return false;
    }
}

@* Option 2: Immutable parameters prevent re-render *@
@code {
    // ComponentBase only re-renders when parameters change
    // For reference types, implement IEquatable<T>
    [Parameter] public ProductDto? Product { get; set; }
}

@* Memoization with computed properties *@
@code {
    private IEnumerable<ProductDto> _allProducts = [];
    private string _filter = string.Empty;

    private IEnumerable<ProductDto> FilteredProducts =>
        string.IsNullOrEmpty(_filter)
            ? _allProducts
            : _allProducts.Where(p => p.Name.Contains(_filter, StringComparison.OrdinalIgnoreCase));
}

@* RenderFragment caching *@
@code {
    // Use @key to preserve component identity during list re-renders
    @foreach (var product in _products)
    {
        @* Adding @key helps Blazor match components to data *@
    }
}

@* In Razor template *@
@foreach (var product in _products)
{
    <ProductCard @key="product.Id" Product="product" />
}
```

```csharp
// Lazy loading WASM assemblies
// Program.cs (Client)
builder.Services.AddSingleton<LazyAssemblyLoader>();

// In component
@inject LazyAssemblyLoader AssemblyLoader

@code {
    private bool _chartsLoaded = false;

    private async Task LoadChartsFeature()
    {
        if (!_chartsLoaded)
        {
            await AssemblyLoader.LoadAssembliesAsync(["MyBlazorApp.Charts.dll"]);
            _chartsLoaded = true;
            StateHasChanged();
        }
    }
}
```

---

## Step 724: Testing Blazor Components (bUnit)

```csharp
// Install: dotnet add package bunit

// Tests/Components/CounterTests.cs
using Bunit;
using Xunit;

namespace MyBlazorApp.Tests;

public class CounterTests : TestContext
{
    [Fact]
    public void Counter_RendersInitialCount()
    {
        var cut = RenderComponent<Counter>();
        
        cut.Find("p[role='status']").MarkupMatches("<p role=\"status\">Current count: <strong>0</strong></p>");
    }

    [Fact]
    public void Counter_IncreasesCountOnClick()
    {
        var cut = RenderComponent<Counter>();
        
        cut.Find("button.btn-primary").Click();
        
        cut.Find("p[role='status'] strong").TextContent.Should().Be("1");
    }

    [Fact]
    public void Counter_DecrementDisabledWhenZero()
    {
        var cut = RenderComponent<Counter>();
        
        var decrementBtn = cut.Find("button.btn-secondary");
        decrementBtn.HasAttribute("disabled").Should().BeTrue();
    }
}

public class ProductFormTests : TestContext
{
    [Fact]
    public async Task ProductForm_ShowsValidationErrors_WhenNameEmpty()
    {
        // Arrange
        Services.AddSingleton(Substitute.For<IProductService>());
        var cut = RenderComponent<ProductForm>();
        
        // Act - submit empty form
        cut.Find("form").Submit();
        
        // Assert
        cut.FindAll(".validation-message").Should().NotBeEmpty();
    }

    [Fact]
    public async Task ProductForm_CallsCreate_OnValidSubmit()
    {
        // Arrange
        var mockService = Substitute.For<IProductService>();
        Services.AddSingleton(mockService);
        
        var cut = RenderComponent<ProductForm>();
        
        // Fill form
        cut.Find("input[id='name']").Change("Test Product");
        cut.Find("input[id='price']").Change("9.99");
        
        // Act
        cut.Find("form").Submit();
        await cut.InvokeAsync(() => Task.CompletedTask);
        
        // Assert
        await mockService.Received(1).CreateAsync(Arg.Any<CreateProductModel>());
    }
}
```

---

## Step 725: Globalization ใน Blazor

```csharp
// Program.cs
builder.Services.AddLocalization(options => options.ResourcesPath = "Resources");
builder.Services.Configure<RequestLocalizationOptions>(options =>
{
    var cultures = new[] { "en-US", "th-TH" };
    options.SetDefaultCulture(cultures[0])
        .AddSupportedCultures(cultures)
        .AddSupportedUICultures(cultures);
});

app.UseRequestLocalization();
```

```razor
@* Components/LanguageSwitcher.razor *@
@inject NavigationManager Navigation
@inject IStringLocalizer<SharedResource> Localizer

<select class="form-select form-select-sm" style="width: auto"
        @onchange="ChangeLanguage">
    <option value="en-US" selected="@(CurrentCulture == "en-US")">English</option>
    <option value="th-TH" selected="@(CurrentCulture == "th-TH")">ภาษาไทย</option>
</select>

@code {
    private string CurrentCulture => 
        System.Threading.Thread.CurrentThread.CurrentCulture.Name;

    private void ChangeLanguage(ChangeEventArgs e)
    {
        var culture = e.Value?.ToString() ?? "en-US";
        var uri = new Uri(Navigation.Uri).GetComponents(
            UriComponents.PathAndQuery, UriFormat.Unescaped);
        var query = $"?culture={Uri.EscapeDataString(culture)}&redirectUri={Uri.EscapeDataString(uri)}";
        Navigation.NavigateTo("/Culture/Set" + query, forceLoad: true);
    }
}
```

---

## Step 726: Server-Side Prerendering

```razor
@* Components/Pages/Products/Details.razor *@
@page "/products/{Id:int}"
@* Prerendering happens by default in .NET 9 Blazor Web App *@

<PageTitle>@(_product?.Name ?? "Loading...")</PageTitle>

@if (_product is null)
{
    @* This shows during SSR, then replaced after WASM/Server loads *@
    <div class="placeholder-glow">
        <h1><span class="placeholder col-6"></span></h1>
        <p><span class="placeholder col-12"></span></p>
    </div>
}
else
{
    <h1>@_product.Name</h1>
    <p>@_product.Description</p>
    <p class="price h3 text-primary">@_product.Price.ToString("C")</p>
    
    <button @onclick="AddToCart" class="btn btn-primary">Add to Cart</button>
}

@code {
    [Parameter] public int Id { get; set; }
    private ProductDto? _product;

    protected override async Task OnInitializedAsync()
    {
        // This runs both during SSR prerender AND after hydration
        _product = await ProductService.GetByIdAsync(Id);
    }

    private void AddToCart()
    {
        // Only runs after interactive rendering
    }
}
```

---

## Step 727: Upload Files ใน Blazor

```razor
@* Components/FileUpload.razor *@
@rendermode InteractiveServer

<h3>File Upload</h3>

<InputFile OnChange="OnFilesChanged" multiple accept="image/*" />

@if (_uploadedFiles.Any())
{
    <ul>
        @foreach (var file in _uploadedFiles)
        {
            <li>
                @file.Name (@FormatSize(file.Size))
                @if (file.Preview is not null)
                {
                    <img src="@file.Preview" width="100" alt="@file.Name" />
                }
            </li>
        }
    </ul>
    
    <button @onclick="UploadAll" class="btn btn-primary">Upload All</button>
}

@code {
    private List<UploadFile> _uploadedFiles = [];
    private const long MaxFileSize = 10 * 1024 * 1024; // 10MB

    private async Task OnFilesChanged(InputFileChangeEventArgs e)
    {
        _uploadedFiles.Clear();
        
        foreach (var file in e.GetMultipleFiles(maxAllowedFiles: 10))
        {
            if (file.Size > MaxFileSize)
            {
                Console.WriteLine($"File {file.Name} is too large");
                continue;
            }

            string? preview = null;
            if (file.ContentType.StartsWith("image/"))
            {
                var imageFile = await file.RequestImageFileAsync(file.ContentType, 100, 100);
                var buffer = new byte[imageFile.Size];
                await imageFile.OpenReadStream(MaxFileSize).ReadExactlyAsync(buffer);
                preview = $"data:{file.ContentType};base64,{Convert.ToBase64String(buffer)}";
            }

            _uploadedFiles.Add(new UploadFile(file, preview));
        }
    }

    private async Task UploadAll()
    {
        using var content = new MultipartFormDataContent();
        
        foreach (var uploadFile in _uploadedFiles)
        {
            var stream = uploadFile.File.OpenReadStream(MaxFileSize);
            content.Add(new StreamContent(stream), "files", uploadFile.File.Name);
        }

        var response = await Http.PostAsync("api/files/upload", content);
        if (response.IsSuccessStatusCode)
        {
            _uploadedFiles.Clear();
            Snackbar.Add("Files uploaded!", Severity.Success);
        }
    }

    private static string FormatSize(long bytes) =>
        bytes switch
        {
            < 1024 => $"{bytes} B",
            < 1024 * 1024 => $"{bytes / 1024:F1} KB",
            _ => $"{bytes / (1024 * 1024):F1} MB"
        };

    private record UploadFile(IBrowserFile File, string? Preview);
}
```

---

## Step 728: Custom Render Fragment Patterns

```razor
@* Reusable Modal Component *@

@* Components/Modal.razor *@
@if (_isVisible)
{
    <div class="modal show d-block" tabindex="-1" @onclick="HandleBackdropClick">
        <div class="modal-dialog @SizeClass" @onclick:stopPropagation>
            <div class="modal-content">
                <div class="modal-header">
                    <h5 class="modal-title">@Title</h5>
                    <button type="button" class="btn-close" @onclick="Close"></button>
                </div>
                <div class="modal-body">
                    @BodyContent
                </div>
                @if (FooterContent is not null)
                {
                    <div class="modal-footer">
                        @FooterContent
                    </div>
                }
            </div>
        </div>
    </div>
    <div class="modal-backdrop fade show"></div>
}

@code {
    [Parameter] public string Title { get; set; } = "Modal";
    [Parameter] public RenderFragment? BodyContent { get; set; }
    [Parameter] public RenderFragment? FooterContent { get; set; }
    [Parameter] public string Size { get; set; } = "md"; // sm, lg, xl
    [Parameter] public bool CloseOnBackdrop { get; set; } = true;
    [Parameter] public EventCallback OnClose { get; set; }

    private bool _isVisible;
    private string SizeClass => Size == "md" ? "" : $"modal-{Size}";

    public void Show() { _isVisible = true; StateHasChanged(); }
    public void Close() 
    { 
        _isVisible = false; 
        OnClose.InvokeAsync();
        StateHasChanged(); 
    }

    private void HandleBackdropClick()
    {
        if (CloseOnBackdrop) Close();
    }
}

@* Usage *@
@page "/modal-demo"

<button @onclick="() => _modal!.Show()">Open Modal</button>

<Modal @ref="_modal" Title="Confirm Delete" Size="sm">
    <BodyContent>
        Are you sure you want to delete this item?
    </BodyContent>
    <FooterContent>
        <button class="btn btn-danger" @onclick="Confirm">Yes, Delete</button>
        <button class="btn btn-secondary" @onclick="() => _modal!.Close()">Cancel</button>
    </FooterContent>
</Modal>

@code {
    private Modal? _modal;

    private void Confirm()
    {
        // Do delete
        _modal?.Close();
    }
}
```

---

## Step 729: Integration Testing

```csharp
// Tests/Integration/ProductsPageTests.cs
using Microsoft.Playwright;
using Xunit;

public class ProductsPageTests : IClassFixture<BlazorPlaywrightFixture>
{
    private readonly IPage _page;

    public ProductsPageTests(BlazorPlaywrightFixture fixture)
    {
        _page = fixture.Page;
    }

    [Fact]
    public async Task ProductsPage_DisplaysProducts()
    {
        await _page.GotoAsync("/products");
        
        var heading = await _page.TextContentAsync("h1");
        Assert.Equal("Products", heading);
        
        // Wait for products to load
        await _page.WaitForSelectorAsync(".product-card");
        
        var cards = await _page.QuerySelectorAllAsync(".product-card");
        Assert.True(cards.Count > 0);
    }

    [Fact]
    public async Task Search_FiltersProducts()
    {
        await _page.GotoAsync("/products");
        
        await _page.FillAsync("input[placeholder='Search...']", "laptop");
        await _page.WaitForTimeoutAsync(500);
        
        var results = await _page.QuerySelectorAllAsync(".product-card");
        foreach (var card in results)
        {
            var name = await card.TextContentAsync();
            Assert.Contains("laptop", name, StringComparison.OrdinalIgnoreCase);
        }
    }
}
```

---

## Step 730: Deployment Blazor WASM

```bash
# Publish Blazor WASM
dotnet publish -c Release -o dist

# Static file hosting on GitHub Pages
# In GitHub Actions: .github/workflows/deploy.yml

# Serve with NGINX
# nginx.conf
# location / {
#     root /usr/share/nginx/html;
#     try_files $uri $uri/ /index.html;
# }

# Environment-specific settings
# wwwroot/appsettings.Production.json
{
    "ApiBaseUrl": "https://api.myapp.com"
}
```

```csharp
// Program.cs - WASM Startup
var builder = WebAssemblyHostBuilder.CreateDefault(args);

builder.RootComponents.Add<App>("#app");
builder.RootComponents.Add<HeadOutlet>("head::after");

// Read settings from wwwroot/appsettings.json (or environment-specific)
var apiBaseUrl = builder.Configuration["ApiBaseUrl"] ?? builder.HostEnvironment.BaseAddress;

builder.Services.AddScoped(sp => new HttpClient 
{ 
    BaseAddress = new Uri(apiBaseUrl) 
});

await builder.Build().RunAsync();
```

---

## สรุป Part 26: Blazor

ในส่วนนี้เราได้เรียนรู้:

| หัวข้อ | ขั้นตอน |
|--------|---------|
| Blazor Models (Server/WASM/Auto) | 701 |
| Components พื้นฐาน | 702 |
| Data Binding (one-way, two-way) | 703 |
| Event Handling | 704 |
| Component Lifecycle | 705 |
| Component Communication (Parameters, EventCallback, Cascading) | 706 |
| State Management (AppStateService) | 707 |
| Forms และ Validation (EditForm, InputText) | 708 |
| JavaScript Interop (IJSRuntime) | 709 |
| Routing | 710 |
| Dependency Injection | 711 |
| HTTP Calls (HttpClient, GetFromJsonAsync) | 712 |
| Render Modes (.NET 8/9) | 713 |
| Streaming Rendering | 714 |
| Component Library | 715 |
| Error Boundaries | 716 |
| Virtualization | 717 |
| Authentication (JWT, AuthorizeView) | 718 |
| Real-time SignalR | 719 |
| Progressive Web App (PWA) | 720 |
| Fluxor State Management | 721 |
| MudBlazor Component Library | 722 |
| Performance Optimization | 723 |
| Testing (bUnit) | 724 |
| Globalization | 725 |
| Prerendering | 726 |
| File Upload | 727 |
| RenderFragment Patterns (Modal) | 728 |
| Integration Testing (Playwright) | 729 |
| Deployment | 730 |

**ขั้นตอนต่อไป**: Part 27 จะเรียนรู้ **SignalR** อย่างลึกซึ้ง - Real-time communication, Hubs, Groups, WebSockets
