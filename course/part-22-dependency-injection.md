# Part 22: Dependency Injection (DI)

## ขั้นตอนที่ 624: DI คืออะไร?

Dependency Injection (DI) คือ pattern ที่ส่ง dependencies เข้ามาจากภายนอก แทนที่จะสร้างเอง

```csharp
// ❌ Without DI: hard-coded dependencies
public class OrderService_NoDI
{
    // สร้าง dependencies เอง - ไม่ testable, ไม่ flexible
    private readonly OrderRepository _repo = new OrderRepository(
        new SqlConnection("Server=localhost;Database=Orders"));
    private readonly EmailSender _email = new EmailSender("smtp.gmail.com", 587);
    
    public void ProcessOrder(Order order)
    {
        _repo.Save(order);
        _email.Send(order.CustomerEmail, "Confirmed!");
    }
}

// ✅ With DI: inject dependencies
public class OrderService
{
    private readonly IOrderRepository _repo;
    private readonly IEmailSender _email;
    
    // Dependencies injected from outside
    public OrderService(IOrderRepository repo, IEmailSender email)
    {
        _repo = repo;
        _email = email;
    }
    
    public async Task ProcessOrderAsync(Order order)
    {
        await _repo.SaveAsync(order);
        await _email.SendAsync(order.CustomerEmail, "Order Confirmed!");
    }
}

// Benefits:
// 1. Unit testable (inject mock)
// 2. Swap implementation (SQL -> Redis -> MongoDB)
// 3. Configure in one place (Program.cs)
// 4. Lifetime management (Singleton, Scoped, Transient)
```

## ขั้นตอนที่ 625: Service Lifetimes

```csharp
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

// Singleton: สร้างครั้งเดียว ใช้ตลอด application lifetime
// Scoped:    สร้างใหม่ต่อ HTTP request (ต่อ scope)
// Transient: สร้างใหม่ทุกครั้งที่ขอ

var builder = Host.CreateApplicationBuilder(args);

// Singleton: ใช้สำหรับ stateless services, expensive-to-create objects
builder.Services.AddSingleton<IConfiguration>(builder.Configuration);
builder.Services.AddSingleton<ICacheService, MemoryCacheService>();
builder.Services.AddSingleton<IHttpClientFactory, DefaultHttpClientFactory>();

// Scoped: ใช้สำหรับ DB connections, unit of work, authenticated user context
builder.Services.AddScoped<IDbConnection>(sp =>
{
    var conn = new SqlConnection(builder.Configuration.GetConnectionString("Default"));
    conn.Open();
    return conn;
});
builder.Services.AddScoped<IOrderRepository, SqlOrderRepository>();
builder.Services.AddScoped<IUnitOfWork, UnitOfWork>();

// Transient: ใช้สำหรับ lightweight, stateless utilities
builder.Services.AddTransient<IEmailValidator, EmailValidator>();
builder.Services.AddTransient<IGuidGenerator, GuidGenerator>();

// การดู lifetime ด้วย ServiceDescriptor
foreach (var descriptor in builder.Services)
{
    Console.WriteLine($"{descriptor.ServiceType.Name}: {descriptor.Lifetime}");
}
```

### ตัวอย่างที่แสดงความแตกต่าง

```csharp
// RequestCounter จะนับต่างกันตาม lifetime
public class RequestCounter
{
    private static int _singletonCount = 0;
    private static int _scopedCount = 0;
    private static int _transientCount = 0;
    
    public readonly string InstanceId = Guid.NewGuid().ToString("N")[..8];
    public int Count;
    
    public int Increment() => ++Count;
}

// ใน Controller:
// Singleton: Count เพิ่มขึ้นเรื่อยๆ ตลอด app
// Scoped: Count เริ่มที่ 0 ทุก request (แต่ instance เดียวกันใน request เดียว)
// Transient: Count เป็น 1 เสมอ (instance ใหม่ทุกครั้ง)

public class DemoController
{
    private readonly RequestCounter _singleton;
    private readonly RequestCounter _scoped;
    private readonly RequestCounter _transient1;
    private readonly RequestCounter _transient2;
    
    public DemoController(
        [FromKeyedServices("singleton")] RequestCounter singleton,
        [FromKeyedServices("scoped")] RequestCounter scoped,
        [FromKeyedServices("transient1")] RequestCounter transient1,
        [FromKeyedServices("transient2")] RequestCounter transient2)
    {
        _singleton = singleton;
        _scoped = scoped;
        _transient1 = transient1;
        _transient2 = transient2;
    }
    
    public void Demo()
    {
        Console.WriteLine($"Singleton: {_singleton.InstanceId} Count={_singleton.Increment()}");
        Console.WriteLine($"Scoped:    {_scoped.InstanceId}");
        Console.WriteLine($"Transient1:{_transient1.InstanceId}"); // ต่างจาก Transient2
        Console.WriteLine($"Transient2:{_transient2.InstanceId}"); // ต่างจาก Transient1
    }
}
```

## ขั้นตอนที่ 626: Microsoft.Extensions.DI - Built-in DI Container

```csharp
// Program.cs ใน ASP.NET Core / Generic Host
var builder = WebApplication.CreateBuilder(args);

// --- Register Services ---

// 1. Interface -> Implementation
builder.Services.AddScoped<IUserRepository, SqlUserRepository>();

// 2. Implementation only (ไม่มี interface)
builder.Services.AddScoped<UserNotificationService>();

// 3. Factory method
builder.Services.AddScoped<IDbConnection>(sp =>
{
    var config = sp.GetRequiredService<IConfiguration>();
    var conn = new SqlConnection(config.GetConnectionString("Default"));
    conn.Open();
    return conn;
});

// 4. Instance
var settings = new AppSettings { Theme = "Dark" };
builder.Services.AddSingleton(settings);

// 5. Generic types
builder.Services.AddScoped(typeof(IRepository<>), typeof(GenericRepository<>));

// 6. Multiple implementations
builder.Services.AddScoped<IPaymentGateway, StripeGateway>();
builder.Services.AddScoped<IPaymentGateway, PayPalGateway>(); // เพิ่ม implementation

// การ resolve multiple: IEnumerable<IPaymentGateway>
// services.GetServices<IPaymentGateway>() → ได้ทั้ง Stripe และ PayPal

// 7. Decorator pattern
builder.Services.AddScoped<IUserRepository, SqlUserRepository>();
builder.Services.Decorate<IUserRepository, CachedUserRepository>();
// ต้อง dotnet add package Scrutor

// 8. Named/Keyed Services (.NET 8+)
builder.Services.AddKeyedScoped<IPaymentGateway, StripeGateway>("stripe");
builder.Services.AddKeyedScoped<IPaymentGateway, PayPalGateway>("paypal");

// Resolve keyed service
// var stripe = sp.GetRequiredKeyedService<IPaymentGateway>("stripe");

// --- Build and Use ---
var app = builder.Build();

// Manual resolution (avoid in production - use constructor injection instead)
using var scope = app.Services.CreateScope();
var repo = scope.ServiceProvider.GetRequiredService<IUserRepository>();
var optRepo = scope.ServiceProvider.GetService<IOptionalRepo>(); // null if not registered
```

## ขั้นตอนที่ 627: Constructor Injection (Primary Pattern)

```csharp
// ✅ Best Practice: Constructor Injection
public class OrderApplicationService
{
    private readonly IOrderRepository _orders;
    private readonly IProductRepository _products;
    private readonly IPaymentGateway _payment;
    private readonly IEventBus _events;
    private readonly ILogger<OrderApplicationService> _logger;
    
    public OrderApplicationService(
        IOrderRepository orders,
        IProductRepository products,
        IPaymentGateway payment,
        IEventBus events,
        ILogger<OrderApplicationService> logger)
    {
        // Validate required dependencies
        _orders = orders ?? throw new ArgumentNullException(nameof(orders));
        _products = products ?? throw new ArgumentNullException(nameof(products));
        _payment = payment ?? throw new ArgumentNullException(nameof(payment));
        _events = events ?? throw new ArgumentNullException(nameof(events));
        _logger = logger ?? throw new ArgumentNullException(nameof(logger));
    }
    
    public async Task<Order> PlaceOrderAsync(PlaceOrderCommand command)
    {
        // Use injected services
        _logger.LogInformation("Placing order for customer {CustomerId}", command.CustomerId);
        
        var order = new Order
        {
            CustomerId = command.CustomerId,
            Lines = command.Items.Select(i => new OrderLine(i.ProductId, i.Quantity)).ToList()
        };
        
        var paymentResult = await _payment.ChargeAsync(
            command.CustomerId, order.Total, "THB");
        
        if (!paymentResult.Success)
            throw new PaymentException(paymentResult.ErrorMessage!);
        
        var savedOrder = await _orders.CreateAsync(order);
        
        await _events.PublishAsync(new OrderPlacedEvent(savedOrder.Id, command.CustomerId));
        
        return savedOrder;
    }
}

// Primary Constructor (C# 12) - ทำได้เหมือนกัน แต่สั้นกว่า
public class ProductService(
    IProductRepository repo,
    ILogger<ProductService> logger,
    IMemoryCache cache)
{
    public async Task<Product?> GetByIdAsync(int id)
    {
        var key = $"product:{id}";
        
        if (cache.TryGetValue(key, out Product? cached))
            return cached;
        
        var product = await repo.GetByIdAsync(id);
        
        if (product is not null)
            cache.Set(key, product, TimeSpan.FromMinutes(5));
        
        logger.LogDebug("Product {Id} loaded from database", id);
        return product;
    }
}
```

## ขั้นตอนที่ 628: IOptions<T> Pattern

```csharp
// appsettings.json
/*
{
  "Email": {
    "SmtpHost": "smtp.gmail.com",
    "SmtpPort": 587,
    "SenderName": "My App",
    "SenderEmail": "noreply@myapp.com",
    "UseSSL": true
  },
  "Cache": {
    "DefaultExpiryMinutes": 30,
    "MaxSize": 1000
  }
}
*/

// Configuration classes
public class EmailOptions
{
    public const string SectionName = "Email";
    
    [Required]
    public string SmtpHost { get; set; } = "";
    
    [Range(1, 65535)]
    public int SmtpPort { get; set; } = 587;
    
    [Required]
    public string SenderName { get; set; } = "";
    
    [Required, EmailAddress]
    public string SenderEmail { get; set; } = "";
    
    public bool UseSSL { get; set; } = true;
}

public class CacheOptions
{
    public const string SectionName = "Cache";
    
    public int DefaultExpiryMinutes { get; set; } = 30;
    public int MaxSize { get; set; } = 1000;
}

// Registration
builder.Services.Configure<EmailOptions>(
    builder.Configuration.GetSection(EmailOptions.SectionName));
builder.Services.Configure<CacheOptions>(
    builder.Configuration.GetSection(CacheOptions.SectionName));

// Validate on startup
builder.Services.AddOptions<EmailOptions>()
    .BindConfiguration(EmailOptions.SectionName)
    .ValidateDataAnnotations()
    .ValidateOnStart();

// Injection patterns:

// 1. IOptions<T>: ค่าคงที่ตลอด lifetime ของ service
public class EmailService(IOptions<EmailOptions> options)
{
    private readonly EmailOptions _options = options.Value;
    
    public async Task SendAsync(string to, string subject, string body)
    {
        using var client = new System.Net.Mail.SmtpClient(_options.SmtpHost, _options.SmtpPort);
        client.EnableSsl = _options.UseSSL;
        // ...
    }
}

// 2. IOptionsSnapshot<T>: อ่านใหม่ต่อ scope (per-request)
public class DynamicConfigService(IOptionsSnapshot<CacheOptions> options)
{
    // options.Value อ่านค่า config ที่อาจเปลี่ยนแปลงได้ per-request
    public int GetCacheExpiry() => options.Value.DefaultExpiryMinutes;
}

// 3. IOptionsMonitor<T>: reactive updates ได้ real-time
public class HotReloadService : IDisposable
{
    private readonly IOptionsMonitor<EmailOptions> _monitor;
    private readonly IDisposable? _changeToken;
    
    public HotReloadService(IOptionsMonitor<EmailOptions> monitor)
    {
        _monitor = monitor;
        _changeToken = _monitor.OnChange(OnOptionsChanged);
    }
    
    private void OnOptionsChanged(EmailOptions newOptions, string? name)
    {
        Console.WriteLine($"Email config changed: {newOptions.SmtpHost}");
    }
    
    public EmailOptions Current => _monitor.CurrentValue;
    
    public void Dispose() => _changeToken?.Dispose();
}

// 4. Named Options (หลาย instances ของ options เดียวกัน)
builder.Services.Configure<EmailOptions>("transactional", opts =>
{
    opts.SmtpHost = "smtp-transactional.sendgrid.com";
    opts.SmtpPort = 465;
});

builder.Services.Configure<EmailOptions>("marketing", opts =>
{
    opts.SmtpHost = "smtp-marketing.mailchimp.com";
    opts.SmtpPort = 587;
});

// Resolve named options
public class EmailRouter(IOptionsMonitor<EmailOptions> options)
{
    public EmailOptions GetForType(string emailType)
        => options.Get(emailType); // "transactional" or "marketing"
}
```

## ขั้นตอนที่ 629: Extension Methods สำหรับ Registration

```csharp
// Pattern ที่ดี: แยก registration ออกเป็น extension methods

// Infrastructure layer registration
public static class InfrastructureServiceExtensions
{
    public static IServiceCollection AddInfrastructure(
        this IServiceCollection services, 
        IConfiguration configuration)
    {
        // Database
        services.AddDbContext<AppDbContext>(options =>
            options.UseSqlServer(configuration.GetConnectionString("Default")));
        
        // Repositories
        services.AddScoped<IUserRepository, SqlUserRepository>();
        services.AddScoped<IOrderRepository, SqlOrderRepository>();
        services.AddScoped<IProductRepository, SqlProductRepository>();
        
        // Caching
        services.AddStackExchangeRedisCache(options =>
        {
            options.Configuration = configuration.GetConnectionString("Redis");
            options.InstanceName = "MyApp:";
        });
        
        // External APIs
        services.AddHttpClient<IGitHubClient, GitHubClient>(c =>
        {
            c.BaseAddress = new Uri("https://api.github.com");
            c.DefaultRequestHeaders.Add("User-Agent", "MyApp/1.0");
        });
        
        return services;
    }
}

// Application layer registration
public static class ApplicationServiceExtensions
{
    public static IServiceCollection AddApplication(
        this IServiceCollection services)
    {
        // Application Services
        services.AddScoped<OrderApplicationService>();
        services.AddScoped<ProductApplicationService>();
        services.AddScoped<UserApplicationService>();
        
        // Domain Services
        services.AddScoped<IDiscountCalculator, DiscountCalculator>();
        services.AddScoped<IShippingCalculator, ShippingCalculator>();
        
        // Validators
        services.AddScoped<IOrderValidator, OrderValidator>();
        services.AddValidatorsFromAssembly(Assembly.GetExecutingAssembly());
        
        // AutoMapper
        services.AddAutoMapper(Assembly.GetExecutingAssembly());
        
        // MediatR
        services.AddMediatR(cfg =>
            cfg.RegisterServicesFromAssembly(Assembly.GetExecutingAssembly()));
        
        return services;
    }
}

// Program.cs - clean registration
var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddInfrastructure(builder.Configuration)
    .AddApplication()
    .AddPresentation();

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
```

## ขั้นตอนที่ 630: Assembly Scanning

```csharp
// ลงทะเบียนทุก repository/service ในครั้งเดียว ด้วย convention-based registration

// 1. ด้วย .NET built-in (manual แต่ flexible)
var assembly = Assembly.GetExecutingAssembly();

// หา types ทั้งหมดที่ implement IRepository<>
var repositoryTypes = assembly.GetTypes()
    .Where(t => t.IsClass && !t.IsAbstract)
    .Where(t => t.GetInterfaces()
        .Any(i => i.IsGenericType && i.GetGenericTypeDefinition() == typeof(IRepository<>)))
    .ToList();

foreach (var repoType in repositoryTypes)
{
    var interfaceType = repoType.GetInterfaces()
        .First(i => i.IsGenericType && i.GetGenericTypeDefinition() == typeof(IRepository<>));
    
    builder.Services.AddScoped(interfaceType, repoType);
    Console.WriteLine($"Registered: {interfaceType.Name} -> {repoType.Name}");
}

// 2. ด้วย Scrutor (dotnet add package Scrutor)
builder.Services.Scan(scan => scan
    .FromAssemblyOf<Program>()
    
    // Register all IRepository<T> implementations as Scoped
    .AddClasses(classes => classes.AssignableTo(typeof(IRepository<>)))
        .AsImplementedInterfaces()
        .WithScopedLifetime()
    
    // Register all IHandler implementations as Transient
    .AddClasses(classes => classes.AssignableTo(typeof(IHandler<>)))
        .AsImplementedInterfaces()
        .WithTransientLifetime()
    
    // Register all Validators
    .AddClasses(classes => classes.AssignableTo(typeof(AbstractValidator<>)))
        .AsImplementedInterfaces()
        .WithTransientLifetime()
    
    // Register services by name convention (ends with "Service")
    .AddClasses(classes => classes
        .InNamespaceOf<UserService>()
        .Where(t => t.Name.EndsWith("Service")))
        .AsSelf()
        .AsImplementedInterfaces()
        .WithScopedLifetime());
```

## ขั้นตอนที่ 631: Conditional Registration

```csharp
// ลงทะเบียน service ตาม environment หรือ configuration

// 1. Environment-based
if (builder.Environment.IsDevelopment())
{
    // Development: ใช้ fake implementations
    builder.Services.AddSingleton<IEmailSender, ConsoleEmailSender>();
    builder.Services.AddSingleton<IPaymentGateway, FakePaymentGateway>();
    
    // Enable detailed errors
    builder.Services.Configure<RouteHandlerOptions>(o => o.ThrowOnBadRequest = true);
}
else
{
    // Production: ใช้ real implementations
    builder.Services.AddScoped<IEmailSender, SendGridEmailSender>();
    builder.Services.AddScoped<IPaymentGateway, StripePaymentGateway>();
}

// 2. Feature flags
var featureFlags = builder.Configuration.GetSection("FeatureFlags");
if (featureFlags.GetValue<bool>("UseRedisCache"))
    builder.Services.AddStackExchangeRedisCache(o => 
        o.Configuration = builder.Configuration.GetConnectionString("Redis"));
else
    builder.Services.AddMemoryCache();

// 3. Decorator pattern with conditions
builder.Services.AddScoped<IUserRepository, SqlUserRepository>();

if (featureFlags.GetValue<bool>("EnableCaching"))
    builder.Services.Decorate<IUserRepository, CachedUserRepository>();

if (featureFlags.GetValue<bool>("EnableLogging"))
    builder.Services.Decorate<IUserRepository, LoggingUserRepository>();

// 4. TryAdd - ลงทะเบียนเฉพาะถ้ายังไม่มี
builder.Services.TryAddScoped<IEmailSender, SmtpEmailSender>(); // ถ้ายังไม่ register

// 5. Replace - แทนที่ registration ที่มีอยู่
builder.Services.Replace(
    ServiceDescriptor.Scoped<IEmailSender, MockEmailSender>());

// 6. RemoveAll - ลบ registrations ทั้งหมดสำหรับ type นั้น
builder.Services.RemoveAll<IEmailSender>();
```

## ขั้นตอนที่ 632: Autofac - Advanced DI Container

```csharp
// dotnet add package Autofac
// dotnet add package Autofac.Extensions.DependencyInjection

using Autofac;
using Autofac.Extensions.DependencyInjection;

// Program.cs
builder.Host.UseServiceProviderFactory(new AutofacServiceProviderFactory());
builder.Host.ConfigureContainer<ContainerBuilder>(containerBuilder =>
{
    // Register module (grouping of registrations)
    containerBuilder.RegisterModule<InfrastructureModule>();
    containerBuilder.RegisterModule<ApplicationModule>();
});

// Autofac Module
public class InfrastructureModule : Autofac.Module
{
    protected override void Load(ContainerBuilder builder)
    {
        // Assembly scanning
        builder.RegisterAssemblyTypes(Assembly.GetExecutingAssembly())
            .Where(t => t.Name.EndsWith("Repository"))
            .AsImplementedInterfaces()
            .InstancePerLifetimeScope();
        
        // Named registration
        builder.RegisterType<SmtpEmailSender>()
            .Named<IEmailSender>("smtp")
            .InstancePerLifetimeScope();
        
        builder.RegisterType<SendGridEmailSender>()
            .Named<IEmailSender>("sendgrid")
            .InstancePerLifetimeScope();
        
        // Decorators
        builder.RegisterDecorator<CachedUserRepository, IUserRepository>();
        builder.RegisterDecorator<LoggingUserRepository, IUserRepository>();
        
        // Factory
        builder.Register(ctx =>
        {
            var config = ctx.Resolve<IConfiguration>();
            var conn = new SqlConnection(config.GetConnectionString("Default"));
            conn.Open();
            return conn;
        }).As<IDbConnection>().InstancePerLifetimeScope();
    }
}

public class ApplicationModule : Autofac.Module
{
    protected override void Load(ContainerBuilder builder)
    {
        // Generic registration
        builder.RegisterGeneric(typeof(GenericRepository<>))
            .As(typeof(IRepository<>))
            .InstancePerLifetimeScope();
        
        // Properties injection
        builder.RegisterType<EmailNotificationService>()
            .As<INotificationService>()
            .PropertiesAutowired()
            .InstancePerLifetimeScope();
        
        // Interceptors (AOP)
        builder.RegisterType<LoggingInterceptor>();
        builder.RegisterType<OrderService>()
            .As<IOrderService>()
            .EnableInterfaceInterceptors()
            .InterceptedBy(typeof(LoggingInterceptor));
    }
}
```

## ขั้นตอนที่ 633: Service Locator Anti-Pattern

```csharp
// ❌ Service Locator Anti-Pattern (ควรหลีกเลี่ยง)
public class OrderService_ServiceLocator
{
    private readonly IServiceProvider _serviceProvider; // ❌ dependency on container
    
    public OrderService_ServiceLocator(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }
    
    public async Task ProcessOrderAsync(Order order)
    {
        // ❌ ไม่ดี: dependencies ซ่อนอยู่ใน method body
        var repo = _serviceProvider.GetRequiredService<IOrderRepository>();
        var email = _serviceProvider.GetRequiredService<IEmailSender>();
        
        await repo.SaveAsync(order);
        await email.SendAsync(order.CustomerEmail, "Confirmed");
    }
}

// ✅ ยกเว้นกรณีที่จำเป็นต้องใช้ (lazy resolution, dynamic services)
public class PluginManager
{
    private readonly IServiceProvider _serviceProvider;
    
    public PluginManager(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }
    
    // ✅ OK: resolve at runtime based on dynamic type
    public IPlugin? GetPlugin(string pluginType)
    {
        return pluginType switch
        {
            "pdf" => _serviceProvider.GetService<IPdfPlugin>(),
            "excel" => _serviceProvider.GetService<IExcelPlugin>(),
            _ => null
        };
    }
}

// ✅ Factory pattern เพื่อ create scoped services จาก singleton
public class OrderProcessorFactory
{
    private readonly IServiceScopeFactory _scopeFactory;
    
    public OrderProcessorFactory(IServiceScopeFactory scopeFactory)
    {
        _scopeFactory = scopeFactory;
    }
    
    public async Task ProcessInNewScopeAsync(Order order)
    {
        // สร้าง scope ใหม่เพื่อ resolve scoped services จาก singleton
        await using var scope = _scopeFactory.CreateAsyncScope();
        var processor = scope.ServiceProvider.GetRequiredService<IOrderProcessor>();
        await processor.ProcessAsync(order);
    }
}
```

## ขั้นตอนที่ 634: IHostedService และ BackgroundService

```csharp
// Services ที่ทำงานเป็น background tasks

// 1. Simple background service
public class CleanupHostedService : BackgroundService
{
    private readonly ILogger<CleanupHostedService> _logger;
    private readonly IServiceScopeFactory _scopeFactory;
    
    public CleanupHostedService(
        ILogger<CleanupHostedService> logger,
        IServiceScopeFactory scopeFactory)
    {
        _logger = logger;
        _scopeFactory = scopeFactory;
    }
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Cleanup service started");
        
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await DoCleanupAsync(stoppingToken);
            }
            catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested)
            {
                break;
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error during cleanup");
            }
            
            // รอ 1 ชั่วโมง
            await Task.Delay(TimeSpan.FromHours(1), stoppingToken);
        }
        
        _logger.LogInformation("Cleanup service stopped");
    }
    
    private async Task DoCleanupAsync(CancellationToken ct)
    {
        // ต้องใช้ scope ใหม่เพราะ BackgroundService เป็น Singleton
        await using var scope = _scopeFactory.CreateAsyncScope();
        var repo = scope.ServiceProvider.GetRequiredService<IOrderRepository>();
        
        var count = await repo.DeleteExpiredAsync(ct);
        _logger.LogInformation("Deleted {Count} expired orders", count);
    }
}

// 2. Queue-based background worker
public class EmailQueueProcessor : BackgroundService
{
    private readonly ILogger<EmailQueueProcessor> _logger;
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly System.Threading.Channels.Channel<EmailJob> _channel;
    
    public EmailQueueProcessor(
        ILogger<EmailQueueProcessor> logger,
        IServiceScopeFactory scopeFactory,
        System.Threading.Channels.Channel<EmailJob> channel)
    {
        _logger = logger;
        _scopeFactory = scopeFactory;
        _channel = channel;
    }
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await foreach (var job in _channel.Reader.ReadAllAsync(stoppingToken))
        {
            try
            {
                await using var scope = _scopeFactory.CreateAsyncScope();
                var emailSender = scope.ServiceProvider.GetRequiredService<IEmailSender>();
                await emailSender.SendAsync(job.To, job.Subject, job.Body);
                _logger.LogInformation("Email sent to {Recipient}", job.To);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to send email to {Recipient}", job.To);
            }
        }
    }
}

public record EmailJob(string To, string Subject, string Body);

// Registration
builder.Services.AddHostedService<CleanupHostedService>();
builder.Services.AddSingleton(System.Threading.Channels.Channel.CreateUnbounded<EmailJob>());
builder.Services.AddHostedService<EmailQueueProcessor>();
```

## ขั้นตอนที่ 635: Testing with DI

```csharp
// Unit Test - Mock dependencies
public class OrderServiceTests
{
    private readonly Mock<IOrderRepository> _mockRepo;
    private readonly Mock<IEmailSender> _mockEmail;
    private readonly Mock<ILogger<OrderService>> _mockLogger;
    private readonly OrderService _sut; // System Under Test
    
    public OrderServiceTests()
    {
        _mockRepo = new Mock<IOrderRepository>();
        _mockEmail = new Mock<IEmailSender>();
        _mockLogger = new Mock<ILogger<OrderService>>();
        
        _sut = new OrderService(_mockRepo.Object, _mockEmail.Object, _mockLogger.Object);
    }
    
    [Fact]
    public async Task ProcessOrder_ValidOrder_SavesAndSendsEmail()
    {
        // Arrange
        var order = new Order { Id = 1, CustomerEmail = "test@example.com" };
        _mockRepo.Setup(r => r.SaveAsync(It.IsAny<Order>())).ReturnsAsync(1);
        _mockEmail.Setup(e => e.SendAsync(It.IsAny<string>(), It.IsAny<string>(), It.IsAny<string>()))
            .Returns(Task.CompletedTask);
        
        // Act
        await _sut.ProcessOrderAsync(order);
        
        // Assert
        _mockRepo.Verify(r => r.SaveAsync(order), Times.Once);
        _mockEmail.Verify(e => e.SendAsync("test@example.com", It.IsAny<string>(), It.IsAny<string>()), Times.Once);
    }
}

// Integration Test - Real DI container
public class OrderIntegrationTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;
    
    public OrderIntegrationTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                // Replace real implementations with test doubles
                services.RemoveAll<IOrderRepository>();
                services.AddSingleton<IOrderRepository, InMemoryOrderRepository>();
                
                services.RemoveAll<IEmailSender>();
                services.AddSingleton<IEmailSender, FakeEmailSender>();
                
                // Override configuration
                services.Configure<EmailOptions>(opts =>
                {
                    opts.SmtpHost = "localhost";
                    opts.SmtpPort = 1025;
                });
            });
        });
    }
    
    [Fact]
    public async Task PlaceOrder_ReturnsCreated()
    {
        var client = _factory.CreateClient();
        
        var response = await client.PostAsJsonAsync("/api/orders", new
        {
            customerId = "C001",
            items = new[] { new { productId = 1, quantity = 2 } }
        });
        
        response.EnsureSuccessStatusCode();
        Assert.Equal(System.Net.HttpStatusCode.Created, response.StatusCode);
    }
}

// Custom ServiceProvider สำหรับ Unit Testing
public static class ServiceProviderFactory
{
    public static IServiceProvider CreateTestProvider(
        Action<IServiceCollection>? configure = null)
    {
        var services = new ServiceCollection();
        
        // Register fakes by default
        services.AddSingleton<IOrderRepository, InMemoryOrderRepository>();
        services.AddSingleton<IEmailSender, FakeEmailSender>();
        services.AddLogging(b => b.AddConsole());
        
        // Allow caller to override
        configure?.Invoke(services);
        
        return services.BuildServiceProvider();
    }
}

// Usage in tests
[Fact]
public async Task Test_With_Custom_Provider()
{
    var provider = ServiceProviderFactory.CreateTestProvider(services =>
    {
        services.AddScoped<OrderService>();
    });
    
    await using var scope = provider.CreateAsyncScope();
    var service = scope.ServiceProvider.GetRequiredService<OrderService>();
    // test...
}
```

## ขั้นตอนที่ 636: สรุป Dependency Injection

```
Dependency Injection Summary
═══════════════════════════════════════════════════════════════

1. SERVICE LIFETIMES:
   Singleton  → 1 instance per application (stateless, heavy init)
   Scoped     → 1 instance per request (DB connections, user context)
   Transient  → new instance every time (lightweight utilities)
   
   ⚠️  Captive dependency: Singleton → Scoped/Transient = BUG!

2. REGISTRATION PATTERNS:
   AddSingleton<IFoo, Foo>()
   AddScoped<IFoo, Foo>()
   AddTransient<IFoo, Foo>()
   AddKeyedScoped<IFoo, Foo>("key")  // .NET 8+
   TryAdd...()                        // only if not registered

3. INJECTION PATTERNS:
   Constructor injection (preferred)
   IOptions<T> / IOptionsSnapshot<T> / IOptionsMonitor<T>
   [FromKeyedServices("key")] attribute  // .NET 8+

4. BEST PRACTICES:
   ✓ Inject interfaces, not concretions
   ✓ Use extension methods to group registrations
   ✓ Validate options with .ValidateDataAnnotations()
   ✓ Use IServiceScopeFactory in background services
   ✗ Avoid Service Locator anti-pattern
   ✗ Avoid new-ing concrete dependencies
   ✗ Avoid injecting IServiceProvider directly

5. ADVANCED:
   Assembly scanning (Scrutor)
   Decorators: Wrap implementations
   Background services: BackgroundService
   Keyed services: Named registrations

═══════════════════════════════════════════════════════════════
```

## สรุป Part 22

ใน Part นี้คุณได้เรียนรู้:
- **DI Fundamentals**: Constructor injection, Service lifetimes
- **Built-in DI**: IServiceCollection, registration patterns
- **IOptions<T>**: Options pattern สำหรับ configuration
- **Extension methods**: Clean service registration
- **Assembly scanning**: Convention-based registration
- **Autofac**: Advanced container features
- **BackgroundService**: Long-running background tasks
- **Testing**: Unit tests ด้วย Mock, Integration tests ด้วย WebApplicationFactory

## แบบฝึกหัด

1. สร้าง `IEmailSender` ที่มี 3 implementations: SMTP, SendGrid, Console (สำหรับ dev)
2. ลงทะเบียน implementations ด้วย `builder.Environment.IsDevelopment()` check
3. ใช้ `IOptions<T>` เพื่อกำหนดค่า SMTP settings จาก `appsettings.json`
4. สร้าง `BackgroundService` ที่ส่ง emails จาก queue ทุก 30 วินาที
5. เขียน integration test ที่ replace `IEmailSender` ด้วย `FakeEmailSender`
