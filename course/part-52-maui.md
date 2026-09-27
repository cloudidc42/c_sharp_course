# Part 52: .NET MAUI — Cross-Platform Mobile & Desktop
## Steps 1461-1490: iOS, Android, Windows, macOS from One Codebase

---

## Step 1461: .NET MAUI Project Structure

```
MyApp/
├── MyApp.csproj              ← Multi-target: net9.0-ios, net9.0-android, net9.0-windows, net9.0-maccatalyst
├── App.xaml                  ← App-level resources and styles
├── App.xaml.cs               ← Application lifecycle
├── AppShell.xaml             ← Navigation structure
├── MauiProgram.cs            ← DI and app configuration
├── Platforms/
│   ├── Android/
│   │   ├── AndroidManifest.xml
│   │   └── MainApplication.cs
│   ├── iOS/
│   │   ├── Info.plist
│   │   └── AppDelegate.cs
│   ├── Windows/
│   │   └── Package.appxmanifest
│   └── MacCatalyst/
│       └── Info.plist
├── Resources/
│   ├── AppIcon/              ← Single SVG → all platform icon sizes
│   ├── Fonts/
│   ├── Images/               ← Single PNG/SVG → all DPI variants
│   └── Styles/
│       └── Colors.xaml
├── Views/
│   ├── OrderListPage.xaml
│   └── OrderDetailPage.xaml
├── ViewModels/
│   ├── OrderListViewModel.cs
│   └── OrderDetailViewModel.cs
└── Services/
    └── OrderApiService.cs
```

```xml
<!-- MyApp.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFrameworks>net9.0-android;net9.0-ios;net9.0-maccatalyst;net9.0-windows10.0.19041.0</TargetFrameworks>
    <OutputType>Exe</OutputType>
    <RootNamespace>MyApp</RootNamespace>
    <UseMaui>true</UseMaui>
    <SingleProject>true</SingleProject>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>

    <!-- App metadata -->
    <ApplicationTitle>MyApp</ApplicationTitle>
    <ApplicationId>com.example.myapp</ApplicationId>
    <ApplicationVersion>1.0</ApplicationVersion>
    <ApplicationDisplayVersion>1.0.0</ApplicationDisplayVersion>

    <!-- iOS signing -->
    <CodesignKey>Apple Development: dev@example.com (XXXXXXXXXX)</CodesignKey>
    <CodesignProvision>MyApp Development</CodesignProvision>
  </PropertyGroup>

  <ItemGroup>
    <!-- App icon from single SVG -->
    <MauiIcon Include="Resources\AppIcon\appicon.svg" ForegroundFile="Resources\AppIcon\appiconfg.svg" />
    <!-- Splash screen -->
    <MauiSplashScreen Include="Resources\Splash\splash.svg" Color="#512BD4" />
    <!-- Images -->
    <MauiImage Include="Resources\Images\*" />
    <!-- Fonts -->
    <MauiFont Include="Resources\Fonts\*" />
    <!-- Assets -->
    <MauiAsset Include="Resources\Raw\**" LogicalName="%(RecursiveDir)%(Filename)%(Extension)" />
  </ItemGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.Maui.Controls" Version="9.0.*" />
    <PackageReference Include="CommunityToolkit.Mvvm" Version="8.3.*" />
    <PackageReference Include="CommunityToolkit.Maui" Version="9.1.*" />
    <PackageReference Include="Microsoft.Extensions.Http" Version="9.0.*" />
  </ItemGroup>
</Project>
```

---

## Step 1462: MauiProgram — DI and Configuration

```csharp
// MauiProgram.cs
public static class MauiProgram
{
    public static MauiApp CreateMauiApp()
    {
        var builder = MauiApp.CreateBuilder();

        builder
            .UseMauiApp<App>()
            .UseMauiCommunityToolkit()
            .ConfigureFonts(fonts =>
            {
                fonts.AddFont("OpenSans-Regular.ttf", "OpenSansRegular");
                fonts.AddFont("OpenSans-Semibold.ttf", "OpenSansSemibold");
                fonts.AddFont("MaterialIcons-Regular.ttf", "MaterialIcons");
            });

        // HTTP client with base address
        builder.Services.AddHttpClient<IOrderApiService, OrderApiService>(client =>
        {
            client.BaseAddress = new Uri("https://api.example.com");
            client.DefaultRequestHeaders.Add("Accept", "application/json");
        });

        // Secure storage for auth tokens
        builder.Services.AddSingleton<ISecureTokenStore, SecureTokenStore>();
        builder.Services.AddSingleton<IAuthService, AuthService>();

        // ViewModels
        builder.Services.AddTransient<OrderListViewModel>();
        builder.Services.AddTransient<OrderDetailViewModel>();
        builder.Services.AddTransient<LoginViewModel>();
        builder.Services.AddTransient<ProfileViewModel>();

        // Pages
        builder.Services.AddTransient<OrderListPage>();
        builder.Services.AddTransient<OrderDetailPage>();
        builder.Services.AddTransient<LoginPage>();

        // Navigation
        builder.Services.AddSingleton<INavigationService, ShellNavigationService>();

        // Platform-specific services via conditional compilation
#if ANDROID
        builder.Services.AddSingleton<INotificationService, AndroidNotificationService>();
#elif IOS
        builder.Services.AddSingleton<INotificationService, iOSNotificationService>();
#elif WINDOWS
        builder.Services.AddSingleton<INotificationService, WindowsNotificationService>();
#else
        builder.Services.AddSingleton<INotificationService, NoOpNotificationService>();
#endif

        return builder.Build();
    }
}
```

---

## Step 1463: Shell Navigation

```xml
<!-- AppShell.xaml -->
<Shell
    x:Class="MyApp.AppShell"
    xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
    xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
    xmlns:views="clr-namespace:MyApp.Views"
    Shell.FlyoutBehavior="Flyout">

    <!-- Flyout header -->
    <Shell.FlyoutHeader>
        <Grid BackgroundColor="{StaticResource Primary}" Padding="16" RowDefinitions="Auto,Auto">
            <Image Source="avatar.png" WidthRequest="64" HeightRequest="64"
                   HorizontalOptions="Center" Aspect="AspectFill">
                <Image.Clip>
                    <EllipseGeometry Center="32,32" RadiusX="32" RadiusY="32"/>
                </Image.Clip>
            </Image>
            <Label Grid.Row="1" Text="{Binding UserName}" TextColor="White"
                   FontSize="16" FontAttributes="Bold" HorizontalOptions="Center"/>
        </Grid>
    </Shell.FlyoutHeader>

    <!-- Tab bar section -->
    <TabBar Route="main">
        <Tab Title="Orders" Icon="orders.png">
            <ShellContent
                Title="My Orders"
                ContentTemplate="{DataTemplate views:OrderListPage}"
                Route="orders"/>
        </Tab>

        <Tab Title="Products" Icon="products.png">
            <ShellContent
                ContentTemplate="{DataTemplate views:ProductListPage}"
                Route="products"/>
        </Tab>

        <Tab Title="Profile" Icon="profile.png">
            <ShellContent
                ContentTemplate="{DataTemplate views:ProfilePage}"
                Route="profile"/>
        </Tab>
    </TabBar>

    <!-- Flyout items -->
    <Shell.Items>
        <FlyoutItem Title="Settings" Icon="settings.png" Route="settings">
            <ShellContent ContentTemplate="{DataTemplate views:SettingsPage}"/>
        </FlyoutItem>
    </Shell.Items>
</Shell>
```

```csharp
// AppShell.xaml.cs — register routes for detail pages
public partial class AppShell : Shell
{
    public AppShell()
    {
        InitializeComponent();

        // Register routes for pages not in the tab structure
        Routing.RegisterRoute(nameof(OrderDetailPage), typeof(OrderDetailPage));
        Routing.RegisterRoute(nameof(ProductDetailPage), typeof(ProductDetailPage));
        Routing.RegisterRoute(nameof(LoginPage), typeof(LoginPage));

        BindingContext = new ShellViewModel();
    }
}

// Navigation helper
public class ShellNavigationService : INavigationService
{
    public Task NavigateToAsync(string route, IDictionary<string, object>? parameters = null) =>
        parameters is null
            ? Shell.Current.GoToAsync(route)
            : Shell.Current.GoToAsync(route, parameters);

    public Task GoBackAsync() => Shell.Current.GoToAsync("..");
    public Task GoToRootAsync() => Shell.Current.GoToAsync("//orders");
}
```

---

## Step 1464: MVVM with CommunityToolkit.Mvvm

```csharp
// ViewModels/OrderListViewModel.cs
[QueryProperty(nameof(Filter), "filter")]
public partial class OrderListViewModel : ObservableObject
{
    private readonly IOrderApiService _orderService;
    private readonly INavigationService _navigation;

    [ObservableProperty]
    private string _filter = "all";

    [ObservableProperty]
    [NotifyPropertyChangedFor(nameof(HasOrders))]
    private ObservableCollection<OrderSummary> _orders = [];

    [ObservableProperty]
    private bool _isLoading;

    [ObservableProperty]
    private bool _isRefreshing;

    [ObservableProperty]
    private string? _errorMessage;

    public bool HasOrders => Orders.Count > 0;

    public OrderListViewModel(IOrderApiService orderService, INavigationService navigation)
    {
        _orderService = orderService;
        _navigation = navigation;
    }

    [RelayCommand]
    private async Task LoadOrdersAsync(CancellationToken ct = default)
    {
        IsLoading = true;
        ErrorMessage = null;

        try
        {
            var orders = await _orderService.GetOrdersAsync(Filter, ct);
            Orders = new ObservableCollection<OrderSummary>(orders);
        }
        catch (Exception ex)
        {
            ErrorMessage = $"Failed to load orders: {ex.Message}";
        }
        finally
        {
            IsLoading = false;
            IsRefreshing = false;
        }
    }

    [RelayCommand]
    private Task RefreshAsync() => LoadOrdersAsync();

    [RelayCommand]
    private async Task SelectOrderAsync(OrderSummary order)
    {
        await _navigation.NavigateToAsync(
            nameof(OrderDetailPage),
            new Dictionary<string, object> { [nameof(OrderDetailViewModel.OrderId)] = order.Id }
        );
    }

    [RelayCommand]
    private async Task CreateOrderAsync()
    {
        await _navigation.NavigateToAsync(nameof(CreateOrderPage));
    }

    // Called when page appears
    public async Task OnAppearingAsync()
    {
        if (Orders.Count == 0)
            await LoadOrdersAsync();
    }
}
```

---

## Step 1465: XAML UI with Data Binding

```xml
<!-- Views/OrderListPage.xaml -->
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage
    x:Class="MyApp.Views.OrderListPage"
    xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
    xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
    xmlns:vm="clr-namespace:MyApp.ViewModels"
    xmlns:models="clr-namespace:MyApp.Models"
    xmlns:toolkit="http://schemas.microsoft.com/dotnet/2022/maui/toolkit"
    Title="My Orders">

    <ContentPage.Behaviors>
        <toolkit:EventToCommandBehavior
            EventName="Appearing"
            Command="{Binding LoadOrdersCommand}"/>
    </ContentPage.Behaviors>

    <Grid RowDefinitions="Auto,*">

        <!-- Filter chips -->
        <HorizontalStackLayout Grid.Row="0" Padding="16,8" Spacing="8">
            <Button Text="All" Command="{Binding SetFilterCommand}" CommandParameter="all"
                    Style="{StaticResource FilterChipStyle}"/>
            <Button Text="Pending" Command="{Binding SetFilterCommand}" CommandParameter="pending"
                    Style="{StaticResource FilterChipStyle}"/>
            <Button Text="Delivered" Command="{Binding SetFilterCommand}" CommandParameter="delivered"
                    Style="{StaticResource FilterChipStyle}"/>
        </HorizontalStackLayout>

        <!-- Loading state -->
        <ActivityIndicator Grid.Row="1" IsRunning="{Binding IsLoading}"
                           IsVisible="{Binding IsLoading}"
                           HorizontalOptions="Center" VerticalOptions="Center"/>

        <!-- Empty state -->
        <VerticalStackLayout Grid.Row="1" IsVisible="{Binding HasOrders, Converter={StaticResource InvertBoolConverter}}"
                             VerticalOptions="Center" HorizontalOptions="Center" Spacing="16">
            <Image Source="empty_orders.png" WidthRequest="120" Opacity="0.5"/>
            <Label Text="No orders yet" FontSize="18" TextColor="{StaticResource Gray400}"/>
            <Button Text="Place Your First Order" Command="{Binding CreateOrderCommand}"/>
        </VerticalStackLayout>

        <!-- Order list with pull-to-refresh -->
        <RefreshView Grid.Row="1"
                     IsRefreshing="{Binding IsRefreshing}"
                     Command="{Binding RefreshCommand}"
                     IsVisible="{Binding HasOrders}">

            <CollectionView ItemsSource="{Binding Orders}"
                            SelectionMode="None">

                <CollectionView.ItemsLayout>
                    <LinearItemsLayout Orientation="Vertical" ItemSpacing="4"/>
                </CollectionView.ItemsLayout>

                <CollectionView.ItemTemplate>
                    <DataTemplate x:DataType="models:OrderSummary">
                        <SwipeView>
                            <SwipeView.RightItems>
                                <SwipeItems>
                                    <SwipeItem Text="Cancel" BackgroundColor="Red"
                                               Command="{Binding Source={RelativeSource AncestorType={x:Type vm:OrderListViewModel}},
                                                         Path=CancelOrderCommand}"
                                               CommandParameter="{Binding .}"/>
                                </SwipeItems>
                            </SwipeView.RightItems>

                            <Border Style="{StaticResource CardBorderStyle}">
                                <Grid ColumnDefinitions="*,Auto" RowDefinitions="Auto,Auto" Padding="16">
                                    <Label Text="{Binding OrderNumber}" FontSize="16" FontAttributes="Bold"/>
                                    <Label Grid.Column="1" Text="{Binding Total, StringFormat='{0:C}'}"
                                           FontSize="16" FontAttributes="Bold" TextColor="{StaticResource Primary}"/>
                                    <Label Grid.Row="1" Text="{Binding CreatedAt, StringFormat='{0:MMM dd, yyyy}'}"
                                           TextColor="{StaticResource Gray500}" FontSize="13"/>
                                    <Label Grid.Row="1" Grid.Column="1"
                                           Text="{Binding Status}"
                                           TextColor="{Binding Status, Converter={StaticResource StatusToColorConverter}}"
                                           FontSize="13" FontAttributes="Bold"/>
                                </Grid>

                                <Border.GestureRecognizers>
                                    <TapGestureRecognizer
                                        Command="{Binding Source={RelativeSource AncestorType={x:Type vm:OrderListViewModel}},
                                                  Path=SelectOrderCommand}"
                                        CommandParameter="{Binding .}"/>
                                </Border.GestureRecognizers>
                            </Border>
                        </SwipeView>
                    </DataTemplate>
                </CollectionView.ItemTemplate>
            </CollectionView>
        </RefreshView>

        <!-- Error message -->
        <Border Grid.Row="1" BackgroundColor="{StaticResource Error}"
                IsVisible="{Binding ErrorMessage, Converter={StaticResource NullToBoolConverter}}"
                Padding="16" VerticalOptions="End" Margin="16">
            <Label Text="{Binding ErrorMessage}" TextColor="White"/>
        </Border>
    </Grid>

    <ContentPage.ToolbarItems>
        <ToolbarItem Text="Add" IconImageSource="add.png" Command="{Binding CreateOrderCommand}"/>
    </ContentPage.ToolbarItems>
</ContentPage>
```

---

## Step 1466: HTTP Service with Polly Resilience

```csharp
// Services/OrderApiService.cs
public class OrderApiService(HttpClient httpClient, ISecureTokenStore tokenStore) : IOrderApiService
{
    private readonly ResiliencePipeline<HttpResponseMessage> _pipeline =
        new ResiliencePipelineBuilder<HttpResponseMessage>()
            .AddRetry(new RetryStrategyOptions<HttpResponseMessage>
            {
                MaxRetryAttempts = 3,
                Delay = TimeSpan.FromSeconds(1),
                BackoffType = DelayBackoffType.Exponential,
                ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
                    .HandleResult(r => r.StatusCode >= HttpStatusCode.InternalServerError)
                    .Handle<HttpRequestException>()
            })
            .AddTimeout(TimeSpan.FromSeconds(30))
            .Build();

    public async Task<List<OrderSummary>> GetOrdersAsync(string filter, CancellationToken ct = default)
    {
        var token = await tokenStore.GetTokenAsync(ct);
        httpClient.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", token);

        var response = await _pipeline.ExecuteAsync(
            async pipeCt => await httpClient.GetAsync($"api/v1/orders?filter={filter}", pipeCt),
            ct);

        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<List<OrderSummary>>(ct) ?? [];
    }

    public async Task<OrderDetail?> GetOrderByIdAsync(Guid orderId, CancellationToken ct = default)
    {
        var response = await httpClient.GetAsync($"api/v1/orders/{orderId}", ct);

        if (response.StatusCode == HttpStatusCode.NotFound) return null;
        response.EnsureSuccessStatusCode();

        return await response.Content.ReadFromJsonAsync<OrderDetail>(ct);
    }

    public async Task<OrderSummary> CreateOrderAsync(CreateOrderRequest request, CancellationToken ct = default)
    {
        var response = await httpClient.PostAsJsonAsync("api/v1/orders", request, ct);
        response.EnsureSuccessStatusCode();
        return (await response.Content.ReadFromJsonAsync<OrderSummary>(ct))!;
    }
}
```

---

## Step 1467: Secure Storage for Authentication

```csharp
// Services/AuthService.cs
public class AuthService(
    IOrderApiService apiService,
    ISecureTokenStore tokenStore,
    IConnectivity connectivity) : IAuthService
{
    private const string TokenKey = "auth_token";
    private const string RefreshTokenKey = "refresh_token";

    public async Task<bool> LoginAsync(string email, string password, CancellationToken ct = default)
    {
        if (!connectivity.NetworkAccess.HasFlag(NetworkAccess.Internet))
            throw new NoInternetException("No internet connection available");

        var result = await apiService.LoginAsync(new LoginRequest(email, password), ct);

        await SecureStorage.Default.SetAsync(TokenKey, result.AccessToken);
        await SecureStorage.Default.SetAsync(RefreshTokenKey, result.RefreshToken);

        return true;
    }

    public async Task LogoutAsync()
    {
        SecureStorage.Default.Remove(TokenKey);
        SecureStorage.Default.Remove(RefreshTokenKey);
        await Shell.Current.GoToAsync("//login");
    }

    public async Task<string?> GetTokenAsync(CancellationToken ct = default)
    {
        var token = await SecureStorage.Default.GetAsync(TokenKey);
        if (string.IsNullOrEmpty(token)) return null;

        // Check if token is expired and refresh if needed
        if (IsTokenExpired(token))
        {
            var refreshToken = await SecureStorage.Default.GetAsync(RefreshTokenKey);
            if (string.IsNullOrEmpty(refreshToken)) return null;

            var refreshed = await apiService.RefreshTokenAsync(refreshToken, ct);
            await SecureStorage.Default.SetAsync(TokenKey, refreshed.AccessToken);
            await SecureStorage.Default.SetAsync(RefreshTokenKey, refreshed.RefreshToken);
            return refreshed.AccessToken;
        }

        return token;
    }

    private bool IsTokenExpired(string token)
    {
        try
        {
            var handler = new JwtSecurityTokenHandler();
            var jwt = handler.ReadJwtToken(token);
            return jwt.ValidTo < DateTime.UtcNow.AddMinutes(5);
        }
        catch { return true; }
    }
}
```

---

## Step 1468: Platform-Specific Features

```csharp
// Platform interfaces for abstraction
public interface INotificationService
{
    Task RequestPermissionAsync();
    Task ScheduleLocalNotificationAsync(string title, string body, DateTimeOffset when);
    Task ShowImmediateNotificationAsync(string title, string body);
}

// Android implementation
#if ANDROID
using Android.App;
using AndroidX.Core.App;

public class AndroidNotificationService(Context context) : INotificationService
{
    private const string ChannelId = "orders";

    public async Task RequestPermissionAsync()
    {
        if (OperatingSystem.IsAndroidVersionAtLeast(33))
        {
            var status = await Permissions.RequestAsync<Permissions.PostNotifications>();
            if (status != PermissionStatus.Granted)
                throw new PermissionException("Notification permission denied");
        }
    }

    public Task ScheduleLocalNotificationAsync(string title, string body, DateTimeOffset when)
    {
        CreateNotificationChannel();
        // Use AlarmManager for scheduled notifications
        return Task.CompletedTask;
    }

    public Task ShowImmediateNotificationAsync(string title, string body)
    {
        CreateNotificationChannel();

        var notification = new NotificationCompat.Builder(context, ChannelId)
            .SetContentTitle(title)
            .SetContentText(body)
            .SetSmallIcon(Resource.Drawable.ic_notification)
            .SetAutoCancel(true)
            .Build();

        var manager = NotificationManagerCompat.From(context);
        manager.Notify(Random.Shared.Next(), notification);

        return Task.CompletedTask;
    }

    private void CreateNotificationChannel()
    {
        if (!OperatingSystem.IsAndroidVersionAtLeast(26)) return;

        var channel = new NotificationChannel(ChannelId, "Orders", NotificationImportance.Default);
        var manager = (NotificationManager)context.GetSystemService(Context.NotificationService)!;
        manager.CreateNotificationChannel(channel);
    }
}
#endif

// Geolocation
public class LocationService(IGeolocation geolocation)
{
    public async Task<Location?> GetCurrentLocationAsync(CancellationToken ct = default)
    {
        var status = await Permissions.CheckStatusAsync<Permissions.LocationWhenInUse>();
        if (status != PermissionStatus.Granted)
        {
            status = await Permissions.RequestAsync<Permissions.LocationWhenInUse>();
            if (status != PermissionStatus.Granted) return null;
        }

        return await geolocation.GetLocationAsync(new GeolocationRequest
        {
            DesiredAccuracy = GeolocationAccuracy.High,
            Timeout = TimeSpan.FromSeconds(10)
        }, ct);
    }
}

// Camera
public class DocumentScannerService
{
    public async Task<byte[]?> CapturePhotoAsync()
    {
        if (!MediaPicker.Default.IsCaptureSupported)
            throw new NotSupportedException("Camera not available");

        var photo = await MediaPicker.Default.CapturePhotoAsync();
        if (photo is null) return null;

        await using var stream = await photo.OpenReadAsync();
        var buffer = new MemoryStream();
        await stream.CopyToAsync(buffer);
        return buffer.ToArray();
    }
}
```

---

## Step 1469: Offline Support with SQLite

```csharp
// Install: sqlite-net-pcl, SQLitePCLRaw.bundle_green

public class LocalOrderRepository
{
    private readonly SQLiteAsyncConnection _db;

    public LocalOrderRepository()
    {
        var dbPath = Path.Combine(FileSystem.AppDataDirectory, "orders.db3");
        _db = new SQLiteAsyncConnection(dbPath, SQLiteOpenFlags.ReadWrite | SQLiteOpenFlags.Create | SQLiteOpenFlags.SharedCache);
    }

    public async Task InitializeAsync()
    {
        await _db.CreateTableAsync<OrderEntity>();
        await _db.CreateTableAsync<SyncQueueEntity>();
    }

    public Task<List<OrderEntity>> GetAllAsync() =>
        _db.Table<OrderEntity>().OrderByDescending(o => o.CreatedAt).ToListAsync();

    public Task<OrderEntity?> GetByIdAsync(string id) =>
        _db.Table<OrderEntity>().Where(o => o.Id == id).FirstOrDefaultAsync();

    public Task UpsertAsync(OrderEntity order) =>
        _db.InsertOrReplaceAsync(order);

    public Task<int> DeleteAsync(OrderEntity order) => _db.DeleteAsync(order);

    // Queue actions for sync when back online
    public Task EnqueueSyncActionAsync(SyncAction action, string payload) =>
        _db.InsertAsync(new SyncQueueEntity
        {
            Id = Guid.NewGuid().ToString(),
            Action = action.ToString(),
            Payload = payload,
            CreatedAt = DateTime.UtcNow
        });

    public Task<List<SyncQueueEntity>> GetPendingSyncActionsAsync() =>
        _db.Table<SyncQueueEntity>()
           .OrderBy(s => s.CreatedAt)
           .ToListAsync();
}

[Table("orders")]
public class OrderEntity
{
    [PrimaryKey] public string Id { get; set; } = string.Empty;
    public string OrderNumber { get; set; } = string.Empty;
    public decimal Total { get; set; }
    public string Status { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; }
    public bool IsSynced { get; set; } = true;
    // Store items as JSON in SQLite
    public string ItemsJson { get; set; } = "[]";
    [Ignore] public List<OrderItem> Items
    {
        get => JsonSerializer.Deserialize<List<OrderItem>>(ItemsJson) ?? [];
        set => ItemsJson = JsonSerializer.Serialize(value);
    }
}

// Connectivity-aware repository
public class SmartOrderRepository(
    IOrderApiService apiService,
    LocalOrderRepository localRepo,
    IConnectivity connectivity) : IOrderRepository
{
    public async Task<List<OrderSummary>> GetOrdersAsync(CancellationToken ct = default)
    {
        if (connectivity.NetworkAccess.HasFlag(NetworkAccess.Internet))
        {
            try
            {
                var orders = await apiService.GetOrdersAsync("all", ct);

                // Update local cache
                foreach (var order in orders)
                    await localRepo.UpsertAsync(order.ToEntity());

                return orders;
            }
            catch
            {
                // Fall through to local cache
            }
        }

        // Offline: return local data
        var local = await localRepo.GetAllAsync();
        return local.Select(e => e.ToSummary()).ToList();
    }
}
```

---

## Step 1470: Custom Controls and Handlers

```csharp
// Custom control — gradient button
public class GradientButton : Button
{
    public static readonly BindableProperty StartColorProperty =
        BindableProperty.Create(nameof(StartColor), typeof(Color), typeof(GradientButton), Colors.Blue);

    public static readonly BindableProperty EndColorProperty =
        BindableProperty.Create(nameof(EndColor), typeof(Color), typeof(GradientButton), Colors.Purple);

    public Color StartColor
    {
        get => (Color)GetValue(StartColorProperty);
        set => SetValue(StartColorProperty, value);
    }

    public Color EndColor
    {
        get => (Color)GetValue(EndColorProperty);
        set => SetValue(EndColorProperty, value);
    }
}

// Custom handler for Android
#if ANDROID
public class GradientButtonHandler : ButtonHandler
{
    protected override void ConnectHandler(Android.Widget.Button platformView)
    {
        base.ConnectHandler(platformView);
        UpdateGradient(platformView);
    }

    private void UpdateGradient(Android.Widget.Button button)
    {
        if (VirtualView is not GradientButton gradientButton) return;

        var startColor = gradientButton.StartColor.ToAndroid();
        var endColor = gradientButton.EndColor.ToAndroid();

        var gradient = new Android.Graphics.Drawables.GradientDrawable(
            Android.Graphics.Drawables.GradientDrawable.Orientation.LeftRight,
            [startColor, endColor]);
        gradient.SetCornerRadius(gradientButton.CornerRadius);

        button.Background = gradient;
    }
}
#endif

// Register handler
public static MauiApp CreateMauiApp()
{
    var builder = MauiApp.CreateBuilder();
    builder.ConfigureMauiHandlers(handlers =>
    {
        handlers.AddHandler<GradientButton, GradientButtonHandler>();
    });
    return builder.Build();
}

// XAML usage
// <controls:GradientButton Text="Place Order" StartColor="#6200EE" EndColor="#B39DDB"/>
```

---

## Step 1471: Animations and Visual States

```xml
<!-- Animated status badge -->
<ContentView x:Name="StatusBadge">
    <VisualStateManager.VisualStateGroups>
        <VisualStateGroup Name="OrderStatus">
            <VisualState Name="Pending">
                <VisualState.Setters>
                    <Setter Property="BackgroundColor" Value="{StaticResource Warning}"/>
                </VisualState.Setters>
            </VisualState>
            <VisualState Name="Confirmed">
                <VisualState.Setters>
                    <Setter Property="BackgroundColor" Value="{StaticResource Info}"/>
                </VisualState.Setters>
            </VisualState>
            <VisualState Name="Delivered">
                <VisualState.Setters>
                    <Setter Property="BackgroundColor" Value="{StaticResource Success}"/>
                </VisualState.Setters>
            </VisualState>
        </VisualStateGroup>
    </VisualStateManager.VisualStateGroups>
</ContentView>
```

```csharp
// Animations in code-behind
public partial class OrderDetailPage : ContentPage
{
    private async Task AnimateStatusChangeAsync(string newStatus)
    {
        // Scale down
        await StatusBadge.ScaleTo(0.8, 150, Easing.CubicIn);

        // Update status
        VisualStateManager.GoToState(StatusBadge, newStatus);

        // Scale back up with bounce
        await StatusBadge.ScaleTo(1.1, 150, Easing.BounceOut);
        await StatusBadge.ScaleTo(1.0, 100);
    }

    private async Task ShowSuccessAnimationAsync()
    {
        // Fade in checkmark
        CheckmarkIcon.Opacity = 0;
        CheckmarkIcon.IsVisible = true;
        await CheckmarkIcon.FadeTo(1, 400, Easing.CubicIn);

        // Rotate
        await CheckmarkIcon.RotateTo(360, 600, Easing.SpringOut);

        // Then fade out the whole overlay
        await Task.Delay(1500);
        await SuccessOverlay.FadeTo(0, 300);
        SuccessOverlay.IsVisible = false;
    }
}
```

---

## Step 1472: MAUI Blazor Hybrid

```xml
<!-- MyBlazorApp.csproj — Hybrid: MAUI + Blazor WebView -->
<Project Sdk="Microsoft.NET.Sdk.Razor">
  <PropertyGroup>
    <TargetFrameworks>net9.0-android;net9.0-ios;net9.0-maccatalyst;net9.0-windows10.0.19041.0</TargetFrameworks>
    <UseMauiBlazorWebView>true</UseMauiBlazorWebView>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.Components.WebView.Maui" Version="9.0.*"/>
  </ItemGroup>
</Project>
```

```csharp
// MauiProgram.cs
builder.Services.AddMauiBlazorWebView();
#if DEBUG
builder.Services.AddBlazorWebViewDeveloperTools();
#endif
```

```xml
<!-- MainPage.xaml — embed Blazor inside MAUI -->
<ContentPage>
    <BlazorWebView HostPage="wwwroot/index.html">
        <BlazorWebView.RootComponents>
            <RootComponent Selector="#app" ComponentType="{x:Type local:Routes}"/>
        </BlazorWebView.RootComponents>
    </BlazorWebView>
</ContentPage>
```

```razor
@* Components/Pages/Orders.razor *@
@page "/orders"
@inject IOrderApiService OrderService
@inject NavigationManager Nav

<h1>Orders</h1>

@if (_orders is null)
{
    <div class="loading-spinner">Loading...</div>
}
else if (!_orders.Any())
{
    <EmptyState Message="No orders yet" Icon="📦"/>
}
else
{
    <div class="order-list">
        @foreach (var order in _orders)
        {
            <div class="order-card" @onclick="() => Nav.NavigateTo($"/orders/{order.Id}")">
                <span class="order-number">@order.OrderNumber</span>
                <span class="order-total">@order.Total.ToString("C")</span>
                <OrderStatusBadge Status="@order.Status"/>
            </div>
        }
    </div>
}

@code {
    private List<OrderSummary>? _orders;

    protected override async Task OnInitializedAsync()
    {
        _orders = await OrderService.GetOrdersAsync("all");
    }
}
```

---

## Step 1473: Push Notifications with Firebase

```csharp
// Platforms/Android/FirebaseMessagingService.cs
#if ANDROID
[Service(Exported = false)]
[IntentFilter(["com.google.firebase.MESSAGING_EVENT"])]
public class MyFirebaseMessagingService : FirebaseMessagingService
{
    public override void OnNewToken(string token)
    {
        base.OnNewToken(token);
        // Send token to your server
        Task.Run(async () =>
        {
            var service = MauiApplication.Current.Services.GetRequiredService<IDeviceTokenService>();
            await service.RegisterTokenAsync(token);
        });
    }

    public override void OnMessageReceived(RemoteMessage message)
    {
        base.OnMessageReceived(message);

        var notification = message.GetNotification();
        if (notification is null) return;

        var notificationService = MauiApplication.Current.Services
            .GetRequiredService<INotificationService>();

        MainThread.BeginInvokeOnMainThread(async () =>
        {
            await notificationService.ShowImmediateNotificationAsync(
                notification.Title ?? "New notification",
                notification.Body ?? string.Empty
            );
        });
    }
}
#endif
```

---

## Step 1474: App Lifecycle and Background Tasks

```csharp
// App.xaml.cs
public partial class App : Application
{
    private readonly ISyncService _syncService;
    private readonly IAuthService _authService;

    public App(ISyncService syncService, IAuthService authService)
    {
        InitializeComponent();
        _syncService = syncService;
        _authService = authService;
    }

    protected override Window CreateWindow(IActivationState? activationState)
    {
        var window = new Window(new AppShell());

        window.Created += (s, e) => OnWindowCreated();
        window.Activated += async (s, e) => await OnWindowActivatedAsync();
        window.Deactivated += (s, e) => OnWindowDeactivated();
        window.Destroying += (s, e) => OnWindowDestroying();

        return window;
    }

    private async Task OnWindowActivatedAsync()
    {
        // App came to foreground — sync data
        if (await _authService.IsAuthenticatedAsync())
            _ = _syncService.SyncAsync();
    }

    private void OnWindowDeactivated()
    {
        // App went to background — save state
    }

    private void OnWindowDestroying()
    {
        // App is closing — cleanup
    }

    private void OnWindowCreated()
    {
        // Check if user is logged in and navigate accordingly
        MainThread.BeginInvokeOnMainThread(async () =>
        {
            if (!await _authService.IsAuthenticatedAsync())
                await Shell.Current.GoToAsync("//login");
        });
    }
}

// Background sync service
public class SyncService(
    LocalOrderRepository localRepo,
    IOrderApiService apiService,
    IConnectivity connectivity) : ISyncService
{
    public async Task SyncAsync(CancellationToken ct = default)
    {
        if (!connectivity.NetworkAccess.HasFlag(NetworkAccess.Internet)) return;

        var pendingActions = await localRepo.GetPendingSyncActionsAsync();

        foreach (var action in pendingActions)
        {
            try
            {
                await ProcessSyncActionAsync(action, ct);
                await localRepo.DeleteSyncActionAsync(action.Id);
            }
            catch (Exception ex)
            {
                // Log and continue with next action
                Console.WriteLine($"Sync failed for action {action.Id}: {ex.Message}");
            }
        }

        // Pull latest data
        var orders = await apiService.GetOrdersAsync("all", ct);
        foreach (var order in orders)
            await localRepo.UpsertAsync(order.ToEntity());
    }

    private async Task ProcessSyncActionAsync(SyncQueueEntity action, CancellationToken ct)
    {
        switch (Enum.Parse<SyncAction>(action.Action))
        {
            case SyncAction.CreateOrder:
                var createRequest = JsonSerializer.Deserialize<CreateOrderRequest>(action.Payload)!;
                await apiService.CreateOrderAsync(createRequest, ct);
                break;

            case SyncAction.CancelOrder:
                var orderId = JsonSerializer.Deserialize<Guid>(action.Payload);
                await apiService.CancelOrderAsync(orderId, ct);
                break;
        }
    }
}
```

---

## Step 1475: Performance — CollectionView vs ListView

```xml
<!-- CollectionView: virtualized, better performance than ListView -->
<CollectionView
    ItemsSource="{Binding Orders}"
    ItemsUpdatingScrollMode="KeepLastItemInView"
    RemainingItemsThreshold="5"
    RemainingItemsThresholdReachedCommand="{Binding LoadMoreCommand}">

    <!-- GroupHeader -->
    <CollectionView.GroupHeaderTemplate>
        <DataTemplate x:DataType="models:OrderGroup">
            <Label Text="{Binding GroupKey}" FontAttributes="Bold"
                   BackgroundColor="{StaticResource Gray100}" Padding="16,8"/>
        </DataTemplate>
    </CollectionView.GroupHeaderTemplate>

    <!-- Item template -->
    <CollectionView.ItemTemplate>
        <DataTemplate x:DataType="models:OrderSummary">
            <!-- Reuse this view — minimize allocations -->
            <Grid Padding="16,8" ColumnDefinitions="*,Auto">
                <Label Text="{Binding OrderNumber}" FontSize="14"/>
                <Label Grid.Column="1" Text="{Binding Total, StringFormat='{0:C}'}" FontSize="14"/>
            </Grid>
        </DataTemplate>
    </CollectionView.ItemTemplate>
</CollectionView>
```

```csharp
// Infinite scrolling with cursor pagination
public partial class OrderListViewModel
{
    private string? _nextCursor;
    private bool _hasMoreItems = true;

    [RelayCommand]
    private async Task LoadMoreAsync(CancellationToken ct = default)
    {
        if (!_hasMoreItems || IsLoading) return;

        IsLoading = true;
        try
        {
            var (orders, cursor, hasMore) = await _orderService.GetOrdersPageAsync(_nextCursor, ct);

            foreach (var order in orders)
                Orders.Add(order);

            _nextCursor = cursor;
            _hasMoreItems = hasMore;
        }
        finally
        {
            IsLoading = false;
        }
    }
}
```

---

## Step 1476: Testing MAUI ViewModels

```csharp
// ViewModels are testable — pure C#, no UI dependencies
public class OrderListViewModelTests
{
    [Fact]
    public async Task LoadOrders_OnSuccess_PopulatesOrdersCollection()
    {
        var orders = Enumerable.Range(1, 5).Select(i => new OrderSummary
        {
            Id = Guid.NewGuid(),
            OrderNumber = $"ORD-{i:000}",
            Total = i * 10m,
            Status = "Pending"
        }).ToList();

        var serviceMock = new Mock<IOrderApiService>();
        serviceMock.Setup(x => x.GetOrdersAsync(It.IsAny<string>(), It.IsAny<CancellationToken>()))
                   .ReturnsAsync(orders);

        var navMock = new Mock<INavigationService>();
        var vm = new OrderListViewModel(serviceMock.Object, navMock.Object);

        await vm.LoadOrdersCommand.ExecuteAsync(null);

        vm.Orders.Should().HaveCount(5);
        vm.IsLoading.Should().BeFalse();
        vm.ErrorMessage.Should().BeNull();
    }

    [Fact]
    public async Task LoadOrders_OnFailure_SetsErrorMessage()
    {
        var serviceMock = new Mock<IOrderApiService>();
        serviceMock.Setup(x => x.GetOrdersAsync(It.IsAny<string>(), It.IsAny<CancellationToken>()))
                   .ThrowsAsync(new HttpRequestException("Network error"));

        var vm = new OrderListViewModel(serviceMock.Object, Mock.Of<INavigationService>());

        await vm.LoadOrdersCommand.ExecuteAsync(null);

        vm.Orders.Should().BeEmpty();
        vm.ErrorMessage.Should().Contain("Network error");
        vm.IsLoading.Should().BeFalse();
    }

    [Fact]
    public async Task SelectOrder_NavigatesToDetailPage()
    {
        var navMock = new Mock<INavigationService>();
        var serviceMock = new Mock<IOrderApiService>();
        serviceMock.Setup(x => x.GetOrdersAsync(It.IsAny<string>(), It.IsAny<CancellationToken>()))
                   .ReturnsAsync([new OrderSummary { Id = Guid.NewGuid(), OrderNumber = "ORD-001" }]);

        var vm = new OrderListViewModel(serviceMock.Object, navMock.Object);
        await vm.LoadOrdersCommand.ExecuteAsync(null);

        await vm.SelectOrderCommand.ExecuteAsync(vm.Orders[0]);

        navMock.Verify(x => x.NavigateToAsync(
            nameof(OrderDetailPage),
            It.IsAny<Dictionary<string, object>>()
        ), Times.Once);
    }
}
```

---

## Step 1477: App Distribution

```yaml
# .github/workflows/maui-publish.yml
name: MAUI Publish

on:
  push:
    tags: ['v*']

jobs:
  android:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: 9.0.x
      - uses: android-actions/setup-android@v3

      - name: Decode keystore
        run: echo "${{ secrets.KEYSTORE_BASE64 }}" | base64 -d > release.keystore

      - name: Build Android APK
        run: |
          dotnet publish MyApp/MyApp.csproj \
            -f net9.0-android \
            -c Release \
            -p:AndroidKeyStore=true \
            -p:AndroidSigningKeyStore=release.keystore \
            -p:AndroidSigningKeyAlias=${{ secrets.KEY_ALIAS }} \
            -p:AndroidSigningKeyPass=${{ secrets.KEY_PASSWORD }} \
            -p:AndroidSigningStorePass=${{ secrets.STORE_PASSWORD }}

      - name: Upload to Play Store
        uses: r0adkll/upload-google-play@v1
        with:
          serviceAccountJsonPlainText: ${{ secrets.PLAY_STORE_JSON }}
          packageName: com.example.myapp
          releaseFiles: '**/*.aab'
          track: internal

  ios:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: 9.0.x

      - name: Import certificate and provisioning profile
        env:
          CERTIFICATE_BASE64: ${{ secrets.IOS_CERTIFICATE_BASE64 }}
          PROVISIONING_PROFILE_BASE64: ${{ secrets.IOS_PROVISIONING_PROFILE_BASE64 }}
        run: |
          echo "$CERTIFICATE_BASE64" | base64 -d > certificate.p12
          echo "$PROVISIONING_PROFILE_BASE64" | base64 -d > profile.mobileprovision
          security import certificate.p12 -P "${{ secrets.CERTIFICATE_PASSWORD }}" -k ~/Library/Keychains/login.keychain
          mkdir -p ~/Library/MobileDevice/Provisioning\ Profiles
          cp profile.mobileprovision ~/Library/MobileDevice/Provisioning\ Profiles/

      - name: Build iOS IPA
        run: |
          dotnet publish MyApp/MyApp.csproj \
            -f net9.0-ios \
            -c Release \
            -p:ArchiveOnBuild=true \
            -p:CodesignKey="${{ secrets.SIGNING_IDENTITY }}" \
            -p:CodesignProvision="${{ secrets.PROVISIONING_PROFILE_NAME }}"

      - name: Upload to TestFlight
        run: xcrun altool --upload-app -f **/*.ipa -u ${{ secrets.APPLE_ID }} -p ${{ secrets.APP_SPECIFIC_PASSWORD }}
```

---

## Summary: .NET MAUI Architecture

```
╔═══════════════════════════════════════════════════════════╗
║           .NET MAUI Cross-Platform Architecture           ║
╠═══════════════════════════════════════════════════════════╣
║  Shared Code (C#/XAML)                                    ║
║  ┌────────────────┐  ┌──────────────────┐                 ║
║  │   ViewModels   │  │    Services      │                 ║
║  │ (MVVM Toolkit) │  │ (HTTP, SQLite)   │                 ║
║  └────────────────┘  └──────────────────┘                 ║
║  ┌────────────────────────────────────┐                   ║
║  │         XAML Pages & Controls      │                   ║
║  └────────────────────────────────────┘                   ║
╠═══════════════════════════════════════════════════════════╣
║  .NET MAUI Abstraction Layer                              ║
║  (IConnectivity, IGeolocation, SecureStorage, etc.)       ║
╠══════════════╦══════════════╦══════════════╦══════════════╣
║   Android    ║     iOS      ║   Windows    ║   macOS      ║
║  (Handlers)  ║  (Handlers)  ║  (Handlers)  ║  (Handlers)  ║
╚══════════════╩══════════════╩══════════════╩══════════════╝

Platforms built from ONE codebase:
  ✅ Android (API 21+)
  ✅ iOS (12.2+)
  ✅ Windows (10.0.17763.0+)
  ✅ macOS (12+, via Mac Catalyst)
```

---

*Part 52 complete — Steps 1461-1477. Topics covered: multi-target project structure, MauiProgram DI, Shell navigation, MVVM with CommunityToolkit.Mvvm (ObservableObject, RelayCommand, ObservableProperty), XAML data binding with CollectionView, SwipeView, RefreshView, HTTP service with Polly, SecureStorage auth, platform-specific features (notifications, geolocation, camera), offline support with SQLite and sync queue, custom controls and handlers, animations and visual states, MAUI Blazor Hybrid, Firebase push notifications, app lifecycle, CollectionView infinite scroll, ViewModel unit tests, CI/CD for App Store / Play Store distribution.*
