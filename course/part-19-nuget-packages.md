# Part 19: NuGet Packages และการจัดการ Package

## ขั้นตอนที่ 581: NuGet คืออะไร?

NuGet คือ Package Manager สำหรับ .NET ที่ช่วยให้คุณสามารถแชร์และใช้โค้ดที่พัฒนาโดยผู้อื่น

```bash
# ค้นหา package
dotnet search Newtonsoft.Json

# เพิ่ม package เข้าโปรเจกต์
dotnet add package Newtonsoft.Json

# เพิ่ม package เวอร์ชันที่ระบุ
dotnet add package Newtonsoft.Json --version 13.0.3

# ลบ package
dotnet remove package Newtonsoft.Json

# รายการ packages ที่ติดตั้งอยู่
dotnet list package

# อัปเดต packages ทั้งหมด
dotnet list package --outdated
```

## ขั้นตอนที่ 582: .csproj และ Package References

```xml
<!-- MyProject.csproj -->
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net9.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

  <ItemGroup>
    <!-- Package พื้นฐาน -->
    <PackageReference Include="Newtonsoft.Json" Version="13.0.3" />
    
    <!-- เวอร์ชัน range -->
    <PackageReference Include="Serilog" Version="[3.0.0, 4.0.0)" />
    
    <!-- เวอร์ชัน wildcard -->
    <PackageReference Include="AutoMapper" Version="12.*" />
    
    <!-- PrivateAssets: ไม่ส่งต่อ dependency ไปยัง project ที่ reference -->
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.8.0">
      <PrivateAssets>all</PrivateAssets>
      <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
    </PackageReference>
    
    <!-- Development-only package -->
    <PackageReference Include="xunit" Version="2.6.4">
      <PrivateAssets>all</PrivateAssets>
    </PackageReference>
  </ItemGroup>

</Project>
```

## ขั้นตอนที่ 583: NuGet.config - การกำหนดค่า NuGet

```xml
<!-- NuGet.config ที่ root ของ solution -->
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <!-- Official NuGet.org -->
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" protocolVersion="3" />
    
    <!-- Private feed (Azure Artifacts, GitHub Packages, etc.) -->
    <add key="MyCompanyFeed" value="https://pkgs.dev.azure.com/myorg/_packaging/myFeed/nuget/v3/index.json" />
    
    <!-- Local folder feed -->
    <add key="LocalPackages" value="C:\LocalNuGet" />
  </packageSources>
  
  <packageSourceCredentials>
    <MyCompanyFeed>
      <add key="Username" value="%AZURE_DEVOPS_USER%" />
      <add key="ClearTextPassword" value="%AZURE_DEVOPS_PAT%" />
    </MyCompanyFeed>
  </packageSourceCredentials>
  
  <config>
    <!-- Directory สำหรับเก็บ global packages -->
    <add key="globalPackagesFolder" value="C:\NuGetPackages" />
    
    <!-- HTTP cache lifetime (นาที) -->
    <add key="maxHttpRequestsPerSource" value="64" />
  </config>
  
  <!-- ปิด source ที่ไม่ต้องการ -->
  <disabledPackageSources>
    <add key="nuget.org" value="false" />
  </disabledPackageSources>
</configuration>
```

## ขั้นตอนที่ 584: Package Lock File

```bash
# สร้าง packages.lock.json เพื่อ reproducible builds
dotnet restore --use-lock-file

# ใช้ lock file (CI/CD)
dotnet restore --locked-mode
```

```xml
<!-- เปิดใช้งาน lock file ใน .csproj -->
<PropertyGroup>
  <RestorePackagesWithLockFile>true</RestorePackagesWithLockFile>
  <!-- ใน CI: ล้มเหลวถ้า lock file ไม่ตรงกัน -->
  <!-- <RestoreLockedMode>true</RestoreLockedMode> -->
</PropertyGroup>
```

```json
// packages.lock.json (generated automatically)
{
  "version": 1,
  "dependencies": {
    "net9.0": {
      "Newtonsoft.Json": {
        "type": "Direct",
        "requested": "[13.0.3, )",
        "resolved": "13.0.3",
        "contentHash": "HrC5BXdl00IP9zeV+0Z848QWPAoCr9P3bDEZguI+gkLcBkAZWD/D/noWjuhKoiGzDsGKp0Un8uf5egMPnvKSA=="
      }
    }
  }
}
```

## ขั้นตอนที่ 585: Popular NuGet Packages - Logging

### Serilog

```csharp
// dotnet add package Serilog.AspNetCore
// dotnet add package Serilog.Sinks.Console
// dotnet add package Serilog.Sinks.File
// dotnet add package Serilog.Enrichers.Environment

using Serilog;
using Serilog.Events;

// กำหนดค่า Serilog
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Debug()
    .MinimumLevel.Override("Microsoft", LogEventLevel.Information)
    .MinimumLevel.Override("System", LogEventLevel.Warning)
    .Enrich.FromLogContext()
    .Enrich.WithMachineName()
    .Enrich.WithThreadId()
    .WriteTo.Console(outputTemplate: 
        "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj}{NewLine}{Exception}")
    .WriteTo.File(
        path: "logs/app-.txt",
        rollingInterval: RollingInterval.Day,
        retainedFileCountLimit: 30,
        outputTemplate: "{Timestamp:yyyy-MM-dd HH:mm:ss.fff zzz} [{Level:u3}] {Message:lj}{NewLine}{Exception}")
    .WriteTo.Seq("http://localhost:5341") // Structured log viewer
    .CreateLogger();

try
{
    // การใช้งาน Structured Logging
    Log.Information("Application starting up");
    
    var userId = 42;
    var action = "Login";
    Log.Information("User {UserId} performed {Action}", userId, action);
    
    // Destructuring objects
    var user = new { Id = 1, Name = "Alice", Email = "alice@example.com" };
    Log.Debug("Processing user {@User}", user); // @ = destructure
    
    // Exception logging
    try
    {
        throw new InvalidOperationException("Something went wrong");
    }
    catch (Exception ex)
    {
        Log.Error(ex, "Error processing request for user {UserId}", userId);
    }
    
    // Performance timing
    using var timer = new System.Diagnostics.Stopwatch();
    timer.Start();
    // ... some work ...
    timer.Stop();
    Log.Information("Operation completed in {ElapsedMs}ms", timer.ElapsedMilliseconds);
}
finally
{
    Log.CloseAndFlush();
}
```

### Microsoft.Extensions.Logging

```csharp
// ใน ASP.NET Core / Generic Host
// การใช้งาน ILogger<T> (built-in)
public class OrderService
{
    private readonly ILogger<OrderService> _logger;
    
    public OrderService(ILogger<OrderService> logger)
    {
        _logger = logger;
    }
    
    public async Task<Order> CreateOrderAsync(CreateOrderRequest request)
    {
        _logger.LogInformation(
            "Creating order for customer {CustomerId} with {ItemCount} items",
            request.CustomerId,
            request.Items.Count);
        
        using (_logger.BeginScope("OrderId: {OrderId}", Guid.NewGuid()))
        {
            try
            {
                var order = await ProcessOrderAsync(request);
                _logger.LogInformation("Order {OrderId} created successfully", order.Id);
                return order;
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to create order for customer {CustomerId}", 
                    request.CustomerId);
                throw;
            }
        }
    }
    
    // High-performance logging ด้วย LoggerMessage
    private static readonly Action<ILogger, string, int, Exception?> _logOrderProcessed =
        LoggerMessage.Define<string, int>(
            LogLevel.Information,
            new EventId(1001, "OrderProcessed"),
            "Order {OrderId} processed with {ItemCount} items");
    
    private void LogOrderProcessed(string orderId, int itemCount)
        => _logOrderProcessed(_logger, orderId, itemCount, null);
}
```

## ขั้นตอนที่ 586: Popular NuGet Packages - Mapping

### AutoMapper

```csharp
// dotnet add package AutoMapper

using AutoMapper;

// Domain Models
public class User
{
    public int Id { get; set; }
    public string FirstName { get; set; } = "";
    public string LastName { get; set; } = "";
    public DateTime BirthDate { get; set; }
    public Address Address { get; set; } = new();
    public List<string> Roles { get; set; } = [];
}

public class Address
{
    public string Street { get; set; } = "";
    public string City { get; set; } = "";
    public string Country { get; set; } = "";
}

// DTOs
public class UserDto
{
    public int Id { get; set; }
    public string FullName { get; set; } = "";
    public int Age { get; set; }
    public string City { get; set; } = "";
    public bool IsAdmin { get; set; }
}

// Mapping Profile
public class UserMappingProfile : Profile
{
    public UserMappingProfile()
    {
        CreateMap<User, UserDto>()
            .ForMember(dest => dest.FullName, 
                opt => opt.MapFrom(src => $"{src.FirstName} {src.LastName}"))
            .ForMember(dest => dest.Age, 
                opt => opt.MapFrom(src => DateTime.Now.Year - src.BirthDate.Year))
            .ForMember(dest => dest.City, 
                opt => opt.MapFrom(src => src.Address.City))
            .ForMember(dest => dest.IsAdmin, 
                opt => opt.MapFrom(src => src.Roles.Contains("Admin")));
        
        // Reverse mapping
        CreateMap<UserDto, User>()
            .ForMember(dest => dest.FirstName, 
                opt => opt.MapFrom(src => src.FullName.Split(' ')[0]))
            .ForMember(dest => dest.LastName, 
                opt => opt.MapFrom(src => src.FullName.Split(' ').Last()));
        
        // Collection mapping
        CreateMap<Address, AddressDto>().ReverseMap();
    }
}

// Setup (Dependency Injection)
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddAutoMapper(typeof(UserMappingProfile));

// การใช้งาน
public class UserController
{
    private readonly IMapper _mapper;
    private readonly IUserRepository _repo;
    
    public UserController(IMapper mapper, IUserRepository repo)
    {
        _mapper = mapper;
        _repo = repo;
    }
    
    public async Task<UserDto> GetUserAsync(int id)
    {
        var user = await _repo.GetByIdAsync(id);
        return _mapper.Map<UserDto>(user);
    }
    
    public async Task<List<UserDto>> GetAllUsersAsync()
    {
        var users = await _repo.GetAllAsync();
        return _mapper.Map<List<UserDto>>(users);
    }
}
```

### Mapster (เร็วกว่า AutoMapper)

```csharp
// dotnet add package Mapster

using Mapster;

// Simple mapping (no config needed for matching properties)
var userDto = user.Adapt<UserDto>();
var users = userList.Adapt<List<UserDto>>();

// Custom mapping
TypeAdapterConfig<User, UserDto>
    .NewConfig()
    .Map(dest => dest.FullName, src => $"{src.FirstName} {src.LastName}")
    .Map(dest => dest.City, src => src.Address.City);

// Global config
TypeAdapterConfig.GlobalSettings.Scan(Assembly.GetExecutingAssembly());
```

## ขั้นตอนที่ 587: Popular NuGet Packages - Validation

### FluentValidation

```csharp
// dotnet add package FluentValidation
// dotnet add package FluentValidation.DependencyInjectionExtensions

using FluentValidation;

// Models
public class CreateProductRequest
{
    public string Name { get; set; } = "";
    public string Description { get; set; } = "";
    public decimal Price { get; set; }
    public int Stock { get; set; }
    public string Category { get; set; } = "";
    public string Email { get; set; } = "";
    public DateTime? ExpiryDate { get; set; }
    public List<string> Tags { get; set; } = [];
}

// Validator
public class CreateProductValidator : AbstractValidator<CreateProductRequest>
{
    private static readonly string[] ValidCategories = ["Electronics", "Clothing", "Food", "Books"];
    
    public CreateProductValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("Product name is required")
            .MinimumLength(3).WithMessage("Name must be at least 3 characters")
            .MaximumLength(100)
            .Matches(@"^[a-zA-Z0-9\s\-]+$").WithMessage("Name contains invalid characters");
        
        RuleFor(x => x.Description)
            .MaximumLength(500)
            .When(x => !string.IsNullOrEmpty(x.Description));
        
        RuleFor(x => x.Price)
            .GreaterThan(0).WithMessage("Price must be greater than 0")
            .LessThanOrEqualTo(999999.99m).WithMessage("Price is too high")
            .PrecisionScale(8, 2, false).WithMessage("Price can have at most 2 decimal places");
        
        RuleFor(x => x.Stock)
            .GreaterThanOrEqualTo(0)
            .LessThanOrEqualTo(100000);
        
        RuleFor(x => x.Category)
            .NotEmpty()
            .Must(c => ValidCategories.Contains(c))
            .WithMessage($"Category must be one of: {string.Join(", ", ValidCategories)}");
        
        RuleFor(x => x.Email)
            .EmailAddress().When(x => !string.IsNullOrEmpty(x.Email));
        
        RuleFor(x => x.ExpiryDate)
            .GreaterThan(DateTime.Now).WithMessage("Expiry date must be in the future")
            .When(x => x.ExpiryDate.HasValue);
        
        RuleFor(x => x.Tags)
            .Must(tags => tags.Count <= 10).WithMessage("Maximum 10 tags allowed")
            .ForEach(tag => tag
                .NotEmpty()
                .MaximumLength(20));
        
        // Custom rule
        RuleFor(x => x)
            .Must(x => !(x.Category == "Food" && !x.ExpiryDate.HasValue))
            .WithMessage("Food products must have an expiry date")
            .WithName("Category");
    }
}

// การใช้งาน
var validator = new CreateProductValidator();
var request = new CreateProductRequest
{
    Name = "A",  // Too short
    Price = -10, // Invalid
    Category = "Unknown" // Invalid category
};

var result = await validator.ValidateAsync(request);

if (!result.IsValid)
{
    foreach (var error in result.Errors)
    {
        Console.WriteLine($"Property: {error.PropertyName}");
        Console.WriteLine($"Error: {error.ErrorMessage}");
        Console.WriteLine($"Code: {error.ErrorCode}");
        Console.WriteLine();
    }
}

// ใน ASP.NET Core
builder.Services.AddValidatorsFromAssembly(Assembly.GetExecutingAssembly());

// Automatic validation ด้วย FluentValidation.AspNetCore
builder.Services.AddFluentValidationAutoValidation();
```

## ขั้นตอนที่ 588: Popular NuGet Packages - HTTP Client

### Refit (Type-safe HTTP Client)

```csharp
// dotnet add package Refit
// dotnet add package Refit.HttpClientFactory

using Refit;

// กำหนด API Interface
public interface IGitHubApi
{
    [Get("/users/{username}")]
    Task<GitHubUser> GetUserAsync(string username);
    
    [Get("/users/{username}/repos")]
    Task<List<GitHubRepo>> GetReposAsync(
        string username, 
        [Query] string sort = "updated",
        [Query] int per_page = 30);
    
    [Post("/user/repos")]
    Task<GitHubRepo> CreateRepoAsync([Body] CreateRepoRequest request);
    
    [Put("/user/starred/{owner}/{repo}")]
    Task StarRepoAsync(string owner, string repo);
    
    [Delete("/user/starred/{owner}/{repo}")]
    Task UnstarRepoAsync(string owner, string repo);
    
    [Get("/search/repositories")]
    Task<SearchResult<GitHubRepo>> SearchReposAsync(
        [Query] string q,
        [Query] string sort = "stars",
        [Query] int page = 1);
    
    [Multipart]
    [Post("/user/avatar")]
    Task UpdateAvatarAsync([AliasAs("avatar")] StreamPart image);
}

public class GitHubUser
{
    [JsonPropertyName("login")]
    public string Login { get; set; } = "";
    
    [JsonPropertyName("name")]
    public string Name { get; set; } = "";
    
    [JsonPropertyName("public_repos")]
    public int PublicRepos { get; set; }
    
    [JsonPropertyName("followers")]
    public int Followers { get; set; }
}

// การลงทะเบียนใน DI
builder.Services
    .AddRefitClient<IGitHubApi>(new RefitSettings
    {
        ContentSerializer = new SystemTextJsonContentSerializer(new JsonSerializerOptions
        {
            PropertyNameCaseInsensitive = true
        })
    })
    .ConfigureHttpClient(c =>
    {
        c.BaseAddress = new Uri("https://api.github.com");
        c.DefaultRequestHeaders.Add("Accept", "application/vnd.github.v3+json");
        c.DefaultRequestHeaders.Add("User-Agent", "MyApp/1.0");
        c.DefaultRequestHeaders.Authorization = 
            new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", 
                Environment.GetEnvironmentVariable("GITHUB_TOKEN") ?? "");
    });

// การใช้งาน
public class GitHubService(IGitHubApi api)
{
    public async Task<UserSummary> GetUserSummaryAsync(string username)
    {
        var user = await api.GetUserAsync(username);
        var repos = await api.GetReposAsync(username, "stars", 5);
        
        return new UserSummary
        {
            Username = user.Login,
            DisplayName = user.Name,
            PublicRepos = user.PublicRepos,
            Followers = user.Followers,
            TopRepos = repos.Select(r => r.Name).ToList()
        };
    }
}
```

### Polly (Resilience & Retry)

```csharp
// dotnet add package Polly
// dotnet add package Microsoft.Extensions.Http.Polly

using Polly;
using Polly.Extensions.Http;
using Polly.Retry;
using Polly.CircuitBreaker;
using Polly.Timeout;

// Retry Policy
var retryPolicy = HttpPolicyExtensions
    .HandleTransientHttpError()  // 5xx, 408, network errors
    .OrResult(msg => msg.StatusCode == System.Net.HttpStatusCode.TooManyRequests)
    .WaitAndRetryAsync(
        retryCount: 3,
        sleepDurationProvider: retryAttempt => 
            TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)),  // Exponential backoff
        onRetry: (outcome, timespan, retryAttempt, context) =>
        {
            Console.WriteLine($"Retry {retryAttempt} after {timespan.TotalSeconds}s. " +
                $"Status: {outcome.Result?.StatusCode}");
        });

// Circuit Breaker Policy
var circuitBreakerPolicy = HttpPolicyExtensions
    .HandleTransientHttpError()
    .CircuitBreakerAsync(
        handledEventsAllowedBeforeBreaking: 5,
        durationOfBreak: TimeSpan.FromSeconds(30),
        onBreak: (outcome, breakDelay) =>
        {
            Console.WriteLine($"Circuit OPEN for {breakDelay.TotalSeconds}s");
        },
        onReset: () => Console.WriteLine("Circuit CLOSED"),
        onHalfOpen: () => Console.WriteLine("Circuit HALF-OPEN"));

// Timeout Policy
var timeoutPolicy = Policy
    .TimeoutAsync<HttpResponseMessage>(
        seconds: 10,
        timeoutStrategy: TimeoutStrategy.Optimistic);

// Combined Policy (จากซ้ายไปขวา: timeout > retry > circuit breaker)
var combinedPolicy = Policy.WrapAsync(timeoutPolicy, retryPolicy, circuitBreakerPolicy);

// ลงทะเบียนกับ HttpClient
builder.Services
    .AddHttpClient<IWeatherService, WeatherService>()
    .AddPolicyHandler(retryPolicy)
    .AddPolicyHandler(circuitBreakerPolicy)
    .AddPolicyHandler(Policy.TimeoutAsync<HttpResponseMessage>(10));

// ใช้กับ Polly v8 (ResiliencePipeline API)
var pipeline = new ResiliencePipelineBuilder<HttpResponseMessage>()
    .AddRetry(new RetryStrategyOptions<HttpResponseMessage>
    {
        ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
            .Handle<HttpRequestException>()
            .HandleResult(r => !r.IsSuccessStatusCode),
        MaxRetryAttempts = 3,
        Delay = TimeSpan.FromSeconds(1),
        BackoffType = DelayBackoffType.Exponential,
        OnRetry = args =>
        {
            Console.WriteLine($"Retry {args.AttemptNumber}");
            return default;
        }
    })
    .AddCircuitBreaker(new CircuitBreakerStrategyOptions<HttpResponseMessage>
    {
        FailureRatio = 0.5,
        SamplingDuration = TimeSpan.FromSeconds(30),
        MinimumThroughput = 10,
        BreakDuration = TimeSpan.FromSeconds(30)
    })
    .AddTimeout(TimeSpan.FromSeconds(10))
    .Build();
```

## ขั้นตอนที่ 589: Popular NuGet Packages - Database

### Dapper (Micro ORM)

```csharp
// dotnet add package Dapper
// dotnet add package Microsoft.Data.SqlClient

using Dapper;
using Microsoft.Data.SqlClient;

public class ProductRepository
{
    private readonly string _connectionString;
    
    public ProductRepository(string connectionString)
    {
        _connectionString = connectionString;
    }
    
    private SqlConnection GetConnection() => 
        new SqlConnection(_connectionString);
    
    // Query เดียว
    public async Task<Product?> GetByIdAsync(int id)
    {
        await using var conn = GetConnection();
        return await conn.QueryFirstOrDefaultAsync<Product>(
            "SELECT * FROM Products WHERE Id = @Id AND IsDeleted = 0",
            new { Id = id });
    }
    
    // Query หลายรายการ
    public async Task<IEnumerable<Product>> GetByCategoryAsync(string category)
    {
        await using var conn = GetConnection();
        return await conn.QueryAsync<Product>(
            @"SELECT p.*, c.Name as CategoryName 
              FROM Products p
              JOIN Categories c ON p.CategoryId = c.Id
              WHERE c.Name = @Category
              ORDER BY p.Name",
            new { Category = category });
    }
    
    // Multi-mapping (JOIN)
    public async Task<IEnumerable<Order>> GetOrdersWithItemsAsync(int customerId)
    {
        await using var conn = GetConnection();
        
        var orderDict = new Dictionary<int, Order>();
        
        var orders = await conn.QueryAsync<Order, OrderItem, Order>(
            @"SELECT o.*, oi.*
              FROM Orders o
              JOIN OrderItems oi ON o.Id = oi.OrderId
              WHERE o.CustomerId = @CustomerId",
            (order, item) =>
            {
                if (!orderDict.TryGetValue(order.Id, out var existingOrder))
                {
                    existingOrder = order;
                    existingOrder.Items = [];
                    orderDict[order.Id] = existingOrder;
                }
                existingOrder.Items.Add(item);
                return existingOrder;
            },
            new { CustomerId = customerId },
            splitOn: "OrderItemId");
        
        return orderDict.Values;
    }
    
    // Multiple result sets
    public async Task<(IEnumerable<Product> Products, int Total)> GetPagedAsync(
        int page, int pageSize)
    {
        await using var conn = GetConnection();
        
        using var multi = await conn.QueryMultipleAsync(
            @"SELECT * FROM Products ORDER BY Id OFFSET @Offset ROWS FETCH NEXT @PageSize ROWS ONLY;
              SELECT COUNT(*) FROM Products;",
            new { Offset = (page - 1) * pageSize, PageSize = pageSize });
        
        var products = await multi.ReadAsync<Product>();
        var total = await multi.ReadFirstAsync<int>();
        
        return (products, total);
    }
    
    // Insert
    public async Task<int> CreateAsync(Product product)
    {
        await using var conn = GetConnection();
        return await conn.ExecuteScalarAsync<int>(
            @"INSERT INTO Products (Name, Price, Stock, CategoryId)
              OUTPUT INSERTED.Id
              VALUES (@Name, @Price, @Stock, @CategoryId)",
            product);
    }
    
    // Bulk insert
    public async Task BulkInsertAsync(IEnumerable<Product> products)
    {
        await using var conn = GetConnection();
        await conn.ExecuteAsync(
            @"INSERT INTO Products (Name, Price, Stock) VALUES (@Name, @Price, @Stock)",
            products);
    }
    
    // Transaction
    public async Task TransferStockAsync(int fromId, int toId, int quantity)
    {
        await using var conn = GetConnection();
        await conn.OpenAsync();
        using var tx = conn.BeginTransaction();
        
        try
        {
            await conn.ExecuteAsync(
                "UPDATE Products SET Stock = Stock - @Qty WHERE Id = @Id",
                new { Id = fromId, Qty = quantity },
                tx);
            
            await conn.ExecuteAsync(
                "UPDATE Products SET Stock = Stock + @Qty WHERE Id = @Id",
                new { Id = toId, Qty = quantity },
                tx);
            
            await tx.CommitAsync();
        }
        catch
        {
            await tx.RollbackAsync();
            throw;
        }
    }
    
    // Stored Procedure
    public async Task<IEnumerable<Product>> SearchAsync(string keyword, string? category)
    {
        await using var conn = GetConnection();
        return await conn.QueryAsync<Product>(
            "sp_SearchProducts",
            new { Keyword = keyword, Category = category },
            commandType: CommandType.StoredProcedure);
    }
}
```

## ขั้นตอนที่ 590: Popular NuGet Packages - MediatR

### MediatR (CQRS Pattern)

```csharp
// dotnet add package MediatR
// dotnet add package MediatR.Extensions.Microsoft.DependencyInjection

using MediatR;

// Commands (Write operations)
public record CreateProductCommand(
    string Name,
    decimal Price,
    int Stock,
    string Category) : IRequest<int>;

// Queries (Read operations)
public record GetProductByIdQuery(int Id) : IRequest<ProductDto?>;
public record GetProductsQuery(string? Category, int Page, int PageSize) 
    : IRequest<PagedResult<ProductDto>>;

// Notifications (Events)
public record ProductCreatedNotification(int ProductId, string Name) : INotification;

// Command Handler
public class CreateProductCommandHandler : IRequestHandler<CreateProductCommand, int>
{
    private readonly IProductRepository _repo;
    private readonly IMediator _mediator;
    
    public CreateProductCommandHandler(IProductRepository repo, IMediator mediator)
    {
        _repo = repo;
        _mediator = mediator;
    }
    
    public async Task<int> Handle(CreateProductCommand request, CancellationToken ct)
    {
        var product = new Product
        {
            Name = request.Name,
            Price = request.Price,
            Stock = request.Stock,
            Category = request.Category,
            CreatedAt = DateTime.UtcNow
        };
        
        var id = await _repo.CreateAsync(product);
        
        // Publish notification (ไม่รอผล)
        await _mediator.Publish(new ProductCreatedNotification(id, product.Name), ct);
        
        return id;
    }
}

// Query Handler
public class GetProductByIdQueryHandler : IRequestHandler<GetProductByIdQuery, ProductDto?>
{
    private readonly IProductRepository _repo;
    private readonly IMapper _mapper;
    
    public GetProductByIdQueryHandler(IProductRepository repo, IMapper mapper)
    {
        _repo = repo;
        _mapper = mapper;
    }
    
    public async Task<ProductDto?> Handle(GetProductByIdQuery request, CancellationToken ct)
    {
        var product = await _repo.GetByIdAsync(request.Id);
        return product is null ? null : _mapper.Map<ProductDto>(product);
    }
}

// Notification Handler (Multiple handlers allowed)
public class SendProductCreatedEmailHandler : INotificationHandler<ProductCreatedNotification>
{
    private readonly IEmailService _emailService;
    
    public SendProductCreatedEmailHandler(IEmailService emailService)
    {
        _emailService = emailService;
    }
    
    public async Task Handle(ProductCreatedNotification notification, CancellationToken ct)
    {
        await _emailService.SendAsync(
            "admin@shop.com",
            "New Product Created",
            $"Product '{notification.Name}' (ID: {notification.ProductId}) was created");
    }
}

public class UpdateSearchIndexHandler : INotificationHandler<ProductCreatedNotification>
{
    public async Task Handle(ProductCreatedNotification notification, CancellationToken ct)
    {
        // อัปเดต search index
        await Task.Delay(100, ct);
        Console.WriteLine($"Search index updated for product {notification.ProductId}");
    }
}

// Pipeline Behavior (Middleware สำหรับ MediatR)
public class LoggingBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    private readonly ILogger<LoggingBehavior<TRequest, TResponse>> _logger;
    
    public LoggingBehavior(ILogger<LoggingBehavior<TRequest, TResponse>> logger)
    {
        _logger = logger;
    }
    
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken ct)
    {
        var requestName = typeof(TRequest).Name;
        _logger.LogInformation("Handling {RequestName}: {@Request}", requestName, request);
        
        var sw = System.Diagnostics.Stopwatch.StartNew();
        var response = await next();
        sw.Stop();
        
        _logger.LogInformation("Handled {RequestName} in {ElapsedMs}ms", 
            requestName, sw.ElapsedMilliseconds);
        
        return response;
    }
}

public class ValidationBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;
    
    public ValidationBehavior(IEnumerable<IValidator<TRequest>> validators)
    {
        _validators = validators;
    }
    
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken ct)
    {
        if (_validators.Any())
        {
            var context = new ValidationContext<TRequest>(request);
            var results = await Task.WhenAll(
                _validators.Select(v => v.ValidateAsync(context, ct)));
            
            var failures = results
                .SelectMany(r => r.Errors)
                .Where(f => f is not null)
                .ToList();
            
            if (failures.Count > 0)
                throw new ValidationException(failures);
        }
        
        return await next();
    }
}

// Registration
builder.Services.AddMediatR(cfg =>
{
    cfg.RegisterServicesFromAssembly(Assembly.GetExecutingAssembly());
    cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
    cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
});

// การใช้งาน ใน Controller
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IMediator _mediator;
    
    public ProductsController(IMediator mediator) => _mediator = mediator;
    
    [HttpPost]
    public async Task<IActionResult> Create(CreateProductCommand command)
    {
        var id = await _mediator.Send(command);
        return CreatedAtAction(nameof(GetById), new { id }, new { id });
    }
    
    [HttpGet("{id}")]
    public async Task<IActionResult> GetById(int id)
    {
        var product = await _mediator.Send(new GetProductByIdQuery(id));
        return product is null ? NotFound() : Ok(product);
    }
}
```

## ขั้นตอนที่ 591: Popular NuGet Packages - Caching

### MemoryCache และ Redis

```csharp
// dotnet add package Microsoft.Extensions.Caching.Memory
// dotnet add package StackExchange.Redis
// dotnet add package Microsoft.Extensions.Caching.StackExchangeRedis

using Microsoft.Extensions.Caching.Memory;
using Microsoft.Extensions.Caching.Distributed;
using StackExchange.Redis;

// In-Memory Cache
public class ProductCacheService
{
    private readonly IMemoryCache _cache;
    private readonly IProductRepository _repo;
    private static readonly TimeSpan DefaultExpiry = TimeSpan.FromMinutes(5);
    
    public ProductCacheService(IMemoryCache cache, IProductRepository repo)
    {
        _cache = cache;
        _repo = repo;
    }
    
    public async Task<Product?> GetProductAsync(int id)
    {
        var cacheKey = $"product:{id}";
        
        if (_cache.TryGetValue(cacheKey, out Product? cached))
            return cached;
        
        var product = await _repo.GetByIdAsync(id);
        
        if (product is not null)
        {
            var options = new MemoryCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = DefaultExpiry,
                SlidingExpiration = TimeSpan.FromMinutes(1),
                Priority = CacheItemPriority.Normal,
                Size = 1
            };
            options.RegisterPostEvictionCallback((key, value, reason, state) =>
            {
                Console.WriteLine($"Cache evicted: {key}, reason: {reason}");
            });
            
            _cache.Set(cacheKey, product, options);
        }
        
        return product;
    }
    
    // GetOrCreate pattern
    public async Task<List<Product>> GetByCategoryAsync(string category)
    {
        return await _cache.GetOrCreateAsync($"products:category:{category}", async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10);
            return await _repo.GetByCategoryAsync(category);
        }) ?? [];
    }
    
    public void InvalidateProduct(int id)
    {
        _cache.Remove($"product:{id}");
    }
}

// Distributed Cache (Redis)
public class DistributedCacheService
{
    private readonly IDistributedCache _cache;
    private static readonly JsonSerializerOptions JsonOptions = new()
    {
        PropertyNameCaseInsensitive = true
    };
    
    public DistributedCacheService(IDistributedCache cache)
    {
        _cache = cache;
    }
    
    public async Task<T?> GetAsync<T>(string key, CancellationToken ct = default)
    {
        var bytes = await _cache.GetAsync(key, ct);
        if (bytes is null) return default;
        
        return JsonSerializer.Deserialize<T>(bytes, JsonOptions);
    }
    
    public async Task SetAsync<T>(
        string key, 
        T value, 
        TimeSpan? absoluteExpiry = null,
        TimeSpan? slidingExpiry = null,
        CancellationToken ct = default)
    {
        var bytes = JsonSerializer.SerializeToUtf8Bytes(value, JsonOptions);
        
        var options = new DistributedCacheEntryOptions();
        if (absoluteExpiry.HasValue)
            options.AbsoluteExpirationRelativeToNow = absoluteExpiry;
        if (slidingExpiry.HasValue)
            options.SlidingExpiration = slidingExpiry;
        
        await _cache.SetAsync(key, bytes, options, ct);
    }
    
    public async Task<T> GetOrCreateAsync<T>(
        string key,
        Func<Task<T>> factory,
        TimeSpan? expiry = null,
        CancellationToken ct = default)
    {
        var cached = await GetAsync<T>(key, ct);
        if (cached is not null) return cached;
        
        var value = await factory();
        await SetAsync(key, value, expiry ?? TimeSpan.FromMinutes(5), ct: ct);
        return value;
    }
    
    public async Task RemoveAsync(string key, CancellationToken ct = default)
        => await _cache.RemoveAsync(key, ct);
}

// Registration
builder.Services.AddMemoryCache(options =>
{
    options.SizeLimit = 1024;  // 1024 items max
});

builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
    options.InstanceName = "MyApp:";
});
```

## ขั้นตอนที่ 592: สร้าง NuGet Package ของตัวเอง

### โครงสร้างไฟล์

```bash
MyLibrary/
├── src/
│   └── MyLibrary/
│       ├── MyLibrary.csproj
│       ├── Extensions/
│       │   └── StringExtensions.cs
│       └── Utilities/
│           └── Guard.cs
├── tests/
│   └── MyLibrary.Tests/
│       ├── MyLibrary.Tests.csproj
│       └── StringExtensionsTests.cs
├── MyLibrary.sln
├── README.md
└── CHANGELOG.md
```

### Library .csproj

```xml
<!-- src/MyLibrary/MyLibrary.csproj -->
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFrameworks>net8.0;net9.0</TargetFrameworks>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <LangVersion>latest</LangVersion>
    
    <!-- NuGet Package Metadata -->
    <PackageId>MyCompany.MyLibrary</PackageId>
    <Version>1.0.0</Version>
    <Authors>Your Name</Authors>
    <Company>My Company</Company>
    <Product>My Library</Product>
    <Description>A utility library for .NET applications with string extensions and guard clauses</Description>
    <PackageTags>utilities;extensions;dotnet</PackageTags>
    <PackageProjectUrl>https://github.com/mycompany/mylibrary</PackageProjectUrl>
    <RepositoryUrl>https://github.com/mycompany/mylibrary.git</RepositoryUrl>
    <RepositoryType>git</RepositoryType>
    <PackageLicenseExpression>MIT</PackageLicenseExpression>
    <PackageReadmeFile>README.md</PackageReadmeFile>
    <PackageIcon>icon.png</PackageIcon>
    
    <!-- Build settings -->
    <GeneratePackageOnBuild>false</GeneratePackageOnBuild>
    <IncludeSymbols>true</IncludeSymbols>
    <SymbolPackageFormat>snupkg</SymbolPackageFormat>
    
    <!-- API Documentation -->
    <GenerateDocumentationFile>true</GenerateDocumentationFile>
    <NoWarn>$(NoWarn);1591</NoWarn>
    
    <!-- Strong naming (optional) -->
    <!-- <SignAssembly>true</SignAssembly>
    <AssemblyOriginatorKeyFile>MyLibrary.snk</AssemblyOriginatorKeyFile> -->
  </PropertyGroup>
  
  <ItemGroup>
    <None Include="../../README.md" Pack="true" PackagePath="/" />
    <None Include="../../icon.png" Pack="true" PackagePath="/" />
  </ItemGroup>

</Project>
```

### Library Code

```csharp
// Extensions/StringExtensions.cs
namespace MyCompany.MyLibrary.Extensions;

/// <summary>
/// Extension methods สำหรับ string
/// </summary>
public static class StringExtensions
{
    /// <summary>
    /// แปลง string เป็น camelCase
    /// </summary>
    /// <param name="value">Input string</param>
    /// <returns>camelCase string</returns>
    /// <example>
    /// <code>
    /// var result = "hello world".ToCamelCase(); // "helloWorld"
    /// </code>
    /// </example>
    public static string ToCamelCase(this string value)
    {
        ArgumentNullException.ThrowIfNull(value);
        if (string.IsNullOrWhiteSpace(value)) return value;
        
        var words = value.Split([' ', '_', '-'], StringSplitOptions.RemoveEmptyEntries);
        if (words.Length == 0) return value;
        
        return string.Concat(
            words[0].ToLower(),
            words.Skip(1).Select(w => char.ToUpper(w[0]) + w[1..].ToLower()));
    }
    
    /// <summary>
    /// ตรวจสอบว่า string เป็น valid email address
    /// </summary>
    public static bool IsValidEmail(this string value)
    {
        if (string.IsNullOrWhiteSpace(value)) return false;
        
        try
        {
            var addr = new System.Net.Mail.MailAddress(value);
            return addr.Address == value;
        }
        catch
        {
            return false;
        }
    }
    
    /// <summary>
    /// ตัด string ให้มีความยาวไม่เกิน maxLength และต่อด้วย suffix
    /// </summary>
    public static string Truncate(this string value, int maxLength, string suffix = "...")
    {
        ArgumentNullException.ThrowIfNull(value);
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(maxLength);
        
        if (value.Length <= maxLength) return value;
        
        return value[..(maxLength - suffix.Length)] + suffix;
    }
    
    /// <summary>
    /// แปลง string เป็น slug (URL-friendly)
    /// </summary>
    public static string ToSlug(this string value)
    {
        ArgumentNullException.ThrowIfNull(value);
        
        return System.Text.RegularExpressions.Regex.Replace(
                value.ToLower().Trim(),
                @"[^a-z0-9\s-]", "")
            .Replace(' ', '-');
    }
}

// Utilities/Guard.cs
namespace MyCompany.MyLibrary.Utilities;

/// <summary>
/// Guard clause helpers สำหรับ parameter validation
/// </summary>
public static class Guard
{
    /// <summary>
    /// ตรวจสอบว่า value ไม่เป็น null
    /// </summary>
    public static T NotNull<T>(T? value, [System.Runtime.CompilerServices.CallerArgumentExpression(nameof(value))] string? paramName = null)
        where T : class
    {
        if (value is null)
            throw new ArgumentNullException(paramName);
        return value;
    }
    
    /// <summary>
    /// ตรวจสอบว่า string ไม่เป็น null หรือ whitespace
    /// </summary>
    public static string NotNullOrWhiteSpace(string? value, [System.Runtime.CompilerServices.CallerArgumentExpression(nameof(value))] string? paramName = null)
    {
        if (string.IsNullOrWhiteSpace(value))
            throw new ArgumentException("Value cannot be null or whitespace.", paramName);
        return value;
    }
    
    /// <summary>
    /// ตรวจสอบว่า value อยู่ในช่วงที่กำหนด
    /// </summary>
    public static T InRange<T>(T value, T min, T max, [System.Runtime.CompilerServices.CallerArgumentExpression(nameof(value))] string? paramName = null)
        where T : IComparable<T>
    {
        if (value.CompareTo(min) < 0 || value.CompareTo(max) > 0)
            throw new ArgumentOutOfRangeException(paramName, $"Value must be between {min} and {max}.");
        return value;
    }
    
    /// <summary>
    /// ตรวจสอบว่า value เป็น positive number
    /// </summary>
    public static T Positive<T>(T value, [System.Runtime.CompilerServices.CallerArgumentExpression(nameof(value))] string? paramName = null)
        where T : IComparable<T>
    {
        if (value.CompareTo(default(T)) <= 0)
            throw new ArgumentOutOfRangeException(paramName, "Value must be positive.");
        return value;
    }
}
```

### สร้างและเผยแพร่ Package

```bash
# Build
dotnet build --configuration Release

# สร้าง package (.nupkg)
dotnet pack --configuration Release --output ./nupkg

# ตรวจสอบ package ก่อน publish
dotnet tool install -g dotnet-validate
dotnet validate package local ./nupkg/MyCompany.MyLibrary.1.0.0.nupkg

# Publish ไปยัง NuGet.org
dotnet nuget push ./nupkg/MyCompany.MyLibrary.1.0.0.nupkg \
    --api-key $NUGET_API_KEY \
    --source https://api.nuget.org/v3/index.json

# Publish Symbol Package
dotnet nuget push ./nupkg/MyCompany.MyLibrary.1.0.0.snupkg \
    --api-key $NUGET_API_KEY \
    --source https://api.nuget.org/v3/index.json

# Publish ไปยัง GitHub Packages
dotnet nuget push ./nupkg/MyCompany.MyLibrary.1.0.0.nupkg \
    --api-key $GITHUB_TOKEN \
    --source "https://nuget.pkg.github.com/myorg/index.json"

# Publish ไปยัง Azure Artifacts
dotnet nuget push ./nupkg/MyCompany.MyLibrary.1.0.0.nupkg \
    --source MyCompanyFeed
```

## ขั้นตอนที่ 593: Versioning Strategy (SemVer)

```csharp
// Semantic Versioning: MAJOR.MINOR.PATCH[-prerelease][+build]
// 1.0.0         - Stable release
// 1.0.1         - Patch (bug fix)
// 1.1.0         - Minor (new feature, backward compatible)
// 2.0.0         - Major (breaking change)
// 2.0.0-alpha.1 - Alpha pre-release
// 2.0.0-beta.2  - Beta pre-release
// 2.0.0-rc.1    - Release candidate

// MinVer - Automatic versioning ด้วย Git tags
// dotnet add package MinVer

// ใน .csproj
// <MinVerDefaultPreReleaseIdentifiers>preview</MinVerDefaultPreReleaseIdentifiers>
// <MinVerTagPrefix>v</MinVerTagPrefix>

// เมื่อ tag v1.2.3 ใน git -> version = 1.2.3
// ระหว่าง tags -> version = 1.2.4-preview.0+4.sha.abc

// GitHub Actions สำหรับ auto-publish
/*
name: Publish NuGet Package

on:
  push:
    tags: ['v*.*.*']

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0  # MinVer needs full history
    
    - uses: actions/setup-dotnet@v4
      with:
        dotnet-version: '9.x'
    
    - name: Build
      run: dotnet build --configuration Release
    
    - name: Test
      run: dotnet test --configuration Release --no-build
    
    - name: Pack
      run: dotnet pack --configuration Release --no-build --output ./nupkg
    
    - name: Push to NuGet.org
      run: dotnet nuget push ./nupkg/*.nupkg --api-key ${{ secrets.NUGET_API_KEY }} --source https://api.nuget.org/v3/index.json
*/
```

## ขั้นตอนที่ 594: Dependency Injection ด้วย Package

```csharp
// Library ควร expose extension methods สำหรับ DI registration
// เพื่อให้ผู้ใช้ integrate ได้ง่าย

namespace MyCompany.MyLibrary;

// Service interface
public interface IMyLibraryService
{
    Task<string> ProcessAsync(string input);
}

// Options
public class MyLibraryOptions
{
    public int MaxLength { get; set; } = 100;
    public bool EnableLogging { get; set; } = true;
    public string DefaultLanguage { get; set; } = "en";
}

// Implementation
public class MyLibraryService : IMyLibraryService
{
    private readonly MyLibraryOptions _options;
    private readonly ILogger<MyLibraryService> _logger;
    
    public MyLibraryService(
        IOptions<MyLibraryOptions> options,
        ILogger<MyLibraryService> logger)
    {
        _options = options.Value;
        _logger = logger;
    }
    
    public Task<string> ProcessAsync(string input)
    {
        _logger.LogInformation("Processing: {Input}", input);
        return Task.FromResult(input.Truncate(_options.MaxLength));
    }
}

// Extension method สำหรับ DI registration
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddMyLibrary(
        this IServiceCollection services,
        Action<MyLibraryOptions>? configureOptions = null)
    {
        if (configureOptions is not null)
            services.Configure(configureOptions);
        else
            services.Configure<MyLibraryOptions>(_ => { });
        
        services.AddSingleton<IMyLibraryService, MyLibraryService>();
        
        return services;
    }
    
    public static IServiceCollection AddMyLibrary(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.Configure<MyLibraryOptions>(configuration.GetSection("MyLibrary"));
        services.AddSingleton<IMyLibraryService, MyLibraryService>();
        return services;
    }
}

// ผู้ใช้ library เรียกใช้ง่ายๆ
// builder.Services.AddMyLibrary(options =>
// {
//     options.MaxLength = 200;
//     options.EnableLogging = false;
// });
```

## ขั้นตอนที่ 595: Analyzers และ Code Quality Packages

```xml
<!-- Analyzer packages -->
<ItemGroup>
  <!-- Microsoft's official analyzers (included in .NET SDK) -->
  <!-- ไม่ต้อง add แยก สำหรับ .NET 5+ -->
  
  <!-- Roslynator - รวม 500+ analyzers และ refactorings -->
  <PackageReference Include="Roslynator.Analyzers" Version="4.7.0">
    <PrivateAssets>all</PrivateAssets>
    <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
  </PackageReference>
  
  <!-- StyleCop - Coding style enforcement -->
  <PackageReference Include="StyleCop.Analyzers" Version="1.2.0-beta.556">
    <PrivateAssets>all</PrivateAssets>
    <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
  </PackageReference>
  
  <!-- SonarAnalyzer - Security and quality -->
  <PackageReference Include="SonarAnalyzer.CSharp" Version="9.18.0.87209">
    <PrivateAssets>all</PrivateAssets>
    <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
  </PackageReference>
</ItemGroup>
```

```json
// .editorconfig สำหรับกำหนด analyzer rules
[*.cs]
dotnet_analyzer_diagnostic.category-Style.severity = suggestion
dotnet_analyzer_diagnostic.category-Design.severity = warning
dotnet_analyzer_diagnostic.category-Performance.severity = warning
dotnet_analyzer_diagnostic.category-Security.severity = error

# CS8618: Non-nullable field must contain a non-null value when exiting constructor
dotnet_diagnostic.CS8618.severity = warning

# IDE0058: Expression value is never used
dotnet_diagnostic.IDE0058.severity = none

# CA1062: Validate arguments of public methods
dotnet_diagnostic.CA1062.severity = suggestion
```

## ขั้นตอนที่ 596: Global Using และ Implicit Usings

```xml
<!-- ใน .csproj -->
<PropertyGroup>
  <ImplicitUsings>enable</ImplicitUsings>
</PropertyGroup>
```

```csharp
// GlobalUsings.cs - Custom global usings สำหรับทั้งโปรเจกต์
global using System;
global using System.Collections.Generic;
global using System.Linq;
global using System.Threading;
global using System.Threading.Tasks;
global using Microsoft.Extensions.Logging;
global using MyApp.Core.Interfaces;
global using MyApp.Core.Models;

// ผลลัพธ์: ทุกไฟล์ .cs ในโปรเจกต์นี้จะ using เหล่านี้โดยอัตโนมัติ
// ไม่ต้องเขียน using ซ้ำในทุกไฟล์
```

## ขั้นตอนที่ 597: Central Package Management

```xml
<!-- Directory.Packages.props ที่ root ของ solution -->
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>
  
  <ItemGroup>
    <!-- กำหนดเวอร์ชัน package ที่นี่เพียงที่เดียว -->
    <PackageVersion Include="Newtonsoft.Json" Version="13.0.3" />
    <PackageVersion Include="Serilog" Version="3.1.1" />
    <PackageVersion Include="Serilog.Sinks.Console" Version="5.0.1" />
    <PackageVersion Include="AutoMapper" Version="12.0.1" />
    <PackageVersion Include="FluentValidation" Version="11.9.0" />
    <PackageVersion Include="MediatR" Version="12.2.0" />
    <PackageVersion Include="Dapper" Version="2.1.28" />
    <PackageVersion Include="xunit" Version="2.6.4" />
    <PackageVersion Include="Moq" Version="4.20.70" />
  </ItemGroup>
</Project>
```

```xml
<!-- Project.csproj - ระบุแค่ชื่อ ไม่ต้องใส่ Version -->
<ItemGroup>
  <PackageReference Include="Newtonsoft.Json" />
  <PackageReference Include="Serilog" />
  <PackageReference Include="AutoMapper" />
</ItemGroup>

<!-- Override เวอร์ชัน สำหรับ project เฉพาะ -->
<ItemGroup>
  <PackageReference Include="Newtonsoft.Json" VersionOverride="12.0.3" />
</ItemGroup>
```

## ขั้นตอนที่ 598: Directory.Build.props และ Directory.Build.targets

```xml
<!-- Directory.Build.props ที่ root - ใช้กับทุก project ใน solution -->
<Project>
  <PropertyGroup>
    <!-- Common settings -->
    <LangVersion>latest</LangVersion>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <AnalysisLevel>latest-recommended</AnalysisLevel>
    
    <!-- Assembly info -->
    <Company>My Company</Company>
    <Copyright>Copyright © My Company 2025</Copyright>
    <AssemblyVersion>1.0.0</AssemblyVersion>
  </PropertyGroup>
  
  <!-- ใช้กับทุก project ใน Tests/ folder -->
  <!-- ตรวจจากชื่อ project -->
  <PropertyGroup Condition="$(MSBuildProjectName.EndsWith('.Tests'))">
    <IsPackable>false</IsPackable>
    <IsPublishable>false</IsPublishable>
  </PropertyGroup>
</Project>
```

```xml
<!-- Directory.Build.targets - runs after every project's targets -->
<Project>
  <Target Name="ValidatePackageMetadata" 
          BeforeTargets="GenerateNuspec"
          Condition="'$(IsPackable)' == 'true'">
    <Error 
      Condition="'$(Authors)' == ''"
      Text="PackageId '$(PackageId)' is missing Authors metadata" />
    <Error 
      Condition="'$(Description)' == ''"
      Text="PackageId '$(PackageId)' is missing Description metadata" />
  </Target>
</Project>
```

## ขั้นตอนที่ 599: dotnet tool - Global Tools

```bash
# ติดตั้ง global tool
dotnet tool install --global dotnet-ef          # Entity Framework CLI
dotnet tool install --global dotnet-outdated    # ตรวจ packages ที่ล้าสมัย
dotnet tool install --global dotnet-format      # Auto-format code
dotnet tool install --global csharpier          # Code formatter
dotnet tool install --global reportgenerator    # Coverage reports
dotnet tool install --global dotnet-coverage    # Coverage tool

# รายการ tools
dotnet tool list --global

# อัปเดต tool
dotnet tool update --global dotnet-ef

# ถอนการติดตั้ง
dotnet tool uninstall --global dotnet-ef

# Local tools (เฉพาะ project/repo)
dotnet new tool-manifest  # สร้าง .config/dotnet-tools.json
dotnet tool install --local dotnet-ef
dotnet tool restore         # ติดตั้ง tools จาก manifest

# dotnet-tools.json
/*
{
  "version": 1,
  "isRoot": true,
  "tools": {
    "dotnet-ef": {
      "version": "9.0.0",
      "commands": ["dotnet-ef"]
    },
    "csharpier": {
      "version": "0.28.1",
      "commands": ["dotnet-csharpier"]
    }
  }
}
*/
```

## ขั้นตอนที่ 600: สรุปภาพรวม NuGet Ecosystem

```
NuGet Ecosystem Overview
══════════════════════════════════════════════════════════
                         ┌─────────────────┐
                         │   NuGet.org     │
                         │ (Central repo)  │
                         └────────┬────────┘
                                  │
           ┌──────────────────────┼──────────────────────┐
           │                      │                      │
    ┌──────▼──────┐      ┌───────▼──────┐      ┌───────▼──────┐
    │  Your App   │      │  Your Lib    │      │  Your Tools  │
    │  (.csproj)  │      │  (Package)   │      │  (dotnet-x)  │
    └──────┬──────┘      └──────┬───────┘      └──────────────┘
           │                   │
    ┌──────▼──────────────────▼──────┐
    │       packages.lock.json       │
    │   (Reproducible builds)        │
    └────────────────────────────────┘

Package Resolution:
  Requested: Serilog >= 3.0.0
  → NuGet resolves to: Serilog 3.1.1 (latest matching)
  → Downloads to: ~/.nuget/packages/serilog/3.1.1/
  → References from: obj/project.assets.json

Key Commands:
  dotnet add package <name>     - เพิ่ม package
  dotnet remove package <name>  - ลบ package
  dotnet list package           - แสดง packages
  dotnet restore                - restore packages
  dotnet pack                   - สร้าง .nupkg
  dotnet nuget push             - publish package

Important Packages by Category:
  Logging:      Serilog, NLog, Microsoft.Extensions.Logging
  ORM:          EF Core, Dapper, NHibernate
  Mapping:      AutoMapper, Mapster
  Validation:   FluentValidation, DataAnnotations
  HTTP:         Refit, RestSharp, Flurl
  Resilience:   Polly, Microsoft.Extensions.Resilience
  CQRS:         MediatR, Wolverine
  DI:           Microsoft.Extensions.DI, Autofac, SimpleInjector
  Testing:      xUnit, NUnit, MSTest, FluentAssertions, Moq, NSubstitute
  Caching:      IMemoryCache, Redis, Garnet
  Messaging:    MassTransit, NServiceBus, Rebus
  Serialization: System.Text.Json, Newtonsoft.Json, MessagePack
══════════════════════════════════════════════════════════
```

## สรุป Part 19

ใน Part นี้คุณได้เรียนรู้:
- **NuGet CLI**: add, remove, list, restore, pack, push
- **Package References**: version ranges, PrivateAssets, lock files
- **Popular Packages**: Serilog, AutoMapper, FluentValidation, Refit, Polly, Dapper, MediatR
- **สร้าง NuGet Package**: metadata, packaging, publishing, versioning (SemVer)
- **Central Package Management**: Directory.Packages.props
- **Global Tools**: dotnet-ef, csharpier, dotnet-format
- **DI Integration Pattern**: extension methods สำหรับ library registration

## แบบฝึกหัด

1. สร้าง library project ที่มี extension methods อย่างน้อย 5 methods พร้อม XML documentation
2. เขียน unit tests ให้ครบ 100% coverage สำหรับ library
3. ใช้ Serilog เพื่อ log ด้วย structured logging ใน Console app
4. สร้าง API endpoint ที่ใช้ MediatR (Command + Query) พร้อม FluentValidation
5. ใช้ Polly เพื่อทำ retry policy กับ HttpClient ที่มี exponential backoff
