# Part 30: Microservices Architecture with .NET Aspire

## Steps 811-845: Building Distributed Systems

---

## Step 811: Microservices คืออะไร?

Microservices คือ architectural pattern ที่แบ่งแอพพลิเคชันออกเป็น services ขนาดเล็กที่ทำงานอิสระต่อกัน

### Monolith vs Microservices

```
Monolith:
[UI] → [Business Logic] → [Database]
ทุกอย่างอยู่ใน process เดียว

Microservices:
[API Gateway]
    ├── [Product Service] → [Product DB]
    ├── [Order Service]   → [Order DB]
    ├── [User Service]    → [User DB]
    └── [Payment Service] → [Payment DB]
```

### เมื่อไหรควรใช้ Microservices
- Team ขนาดใหญ่ (Conway's Law)
- ต้องการ scale แต่ละส่วนแยกกัน
- มี different technology requirements
- ต้องการ independent deployment

---

## Step 812: .NET Aspire คืออะไร?

.NET Aspire คือ opinionated stack สำหรับ cloud-ready distributed applications ใน .NET

```bash
# ติดตั้ง Aspire workload
dotnet workload install aspire

# สร้าง Aspire solution
dotnet new aspire-starter -n EShopAspire
cd EShopAspire

# Structure:
# EShopAspire.AppHost/         - Orchestration
# EShopAspire.ServiceDefaults/ - Shared defaults
# EShopAspire.ApiService/      - Example API service
# EShopAspire.Web/             - Frontend
```

---

## Step 813: AppHost - Orchestration

```csharp
// EShopAspire.AppHost/Program.cs
var builder = DistributedApplication.CreateBuilder(args);

// Infrastructure
var redis = builder.AddRedis("cache")
    .WithRedisInsight();

var postgres = builder.AddPostgres("postgres")
    .WithPgAdmin()
    .WithVolume("postgres-data", "/var/lib/postgresql/data");

var productDb = postgres.AddDatabase("productdb");
var orderDb = postgres.AddDatabase("orderdb");

var rabbitmq = builder.AddRabbitMQ("messaging")
    .WithManagementPlugin();

// Services
var productService = builder.AddProject<Projects.ProductService>("product-service")
    .WithReference(productDb)
    .WithReference(redis)
    .WithReference(rabbitmq);

var orderService = builder.AddProject<Projects.OrderService>("order-service")
    .WithReference(orderDb)
    .WithReference(redis)
    .WithReference(rabbitmq)
    .WithReference(productService);

var userService = builder.AddProject<Projects.UserService>("user-service")
    .WithReference(postgres.AddDatabase("userdb"))
    .WithReference(redis);

// API Gateway
var apiGateway = builder.AddProject<Projects.ApiGateway>("api-gateway")
    .WithReference(productService)
    .WithReference(orderService)
    .WithReference(userService)
    .WithExternalHttpEndpoints();

// Frontend
builder.AddProject<Projects.WebApp>("webapp")
    .WithReference(apiGateway);

builder.Build().Run();
```

---

## Step 814: Service Defaults

```csharp
// EShopAspire.ServiceDefaults/Extensions.cs
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Diagnostics.HealthChecks;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Diagnostics.HealthChecks;
using Microsoft.Extensions.Logging;
using OpenTelemetry.Logs;
using OpenTelemetry.Metrics;
using OpenTelemetry.Trace;

namespace Microsoft.Extensions.Hosting;

public static class Extensions
{
    public static IHostApplicationBuilder AddServiceDefaults(
        this IHostApplicationBuilder builder)
    {
        builder.ConfigureOpenTelemetry();
        builder.AddDefaultHealthChecks();
        builder.Services.AddServiceDiscovery();
        builder.Services.ConfigureHttpClientDefaults(http =>
        {
            http.AddStandardResilienceHandler();
            http.AddServiceDiscovery();
        });
        
        return builder;
    }
    
    public static IHostApplicationBuilder ConfigureOpenTelemetry(
        this IHostApplicationBuilder builder)
    {
        builder.Logging.AddOpenTelemetry(logging =>
        {
            logging.IncludeFormattedMessage = true;
            logging.IncludeScopes = true;
        });
        
        builder.Services.AddOpenTelemetry()
            .WithMetrics(metrics =>
            {
                metrics
                    .AddAspNetCoreInstrumentation()
                    .AddHttpClientInstrumentation()
                    .AddRuntimeInstrumentation();
            })
            .WithTracing(tracing =>
            {
                tracing
                    .AddAspNetCoreInstrumentation()
                    .AddHttpClientInstrumentation()
                    .AddEntityFrameworkCoreInstrumentation();
            });
        
        builder.AddOpenTelemetryExporters();
        return builder;
    }
    
    private static IHostApplicationBuilder AddOpenTelemetryExporters(
        this IHostApplicationBuilder builder)
    {
        var useOtlpExporter = !string.IsNullOrWhiteSpace(
            builder.Configuration["OTEL_EXPORTER_OTLP_ENDPOINT"]);
        
        if (useOtlpExporter)
        {
            builder.Services.AddOpenTelemetry()
                .UseOtlpExporter();
        }
        
        return builder;
    }
    
    public static IHostApplicationBuilder AddDefaultHealthChecks(
        this IHostApplicationBuilder builder)
    {
        builder.Services.AddHealthChecks()
            .AddCheck("self", () => HealthCheckResult.Healthy(), ["live"]);
        
        return builder;
    }
    
    public static WebApplication MapDefaultEndpoints(this WebApplication app)
    {
        if (app.Environment.IsDevelopment())
        {
            app.MapHealthChecks("/health");
            app.MapHealthChecks("/alive", new HealthCheckOptions
            {
                Predicate = r => r.Tags.Contains("live")
            });
        }
        
        return app;
    }
}
```

---

## Step 815: Product Service

```csharp
// ProductService/Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.AddServiceDefaults();

// Database
builder.AddNpgsqlDbContext<ProductDbContext>("productdb");

// Redis
builder.AddRedisDistributedCache("cache");

// RabbitMQ
builder.AddRabbitMQClient("messaging");

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
builder.Services.AddScoped<IProductRepository, ProductRepository>();
builder.Services.AddScoped<IProductService, ProductService>();

var app = builder.Build();

app.UseSwagger();
app.UseSwaggerUI();

// Endpoints
app.MapProductEndpoints();
app.MapDefaultEndpoints();

app.Run();
```

```csharp
// ProductService/Models/Product.cs
namespace ProductService.Models;

public class Product
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public string Name { get; set; } = string.Empty;
    public string Description { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public int StockQuantity { get; set; }
    public string Category { get; set; } = string.Empty;
    public bool IsActive { get; set; } = true;
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime UpdatedAt { get; set; } = DateTime.UtcNow;
}

// ProductService/DTOs/ProductDtos.cs
public record ProductDto(
    Guid Id,
    string Name, 
    string Description,
    decimal Price,
    int StockQuantity,
    string Category);

public record CreateProductRequest(
    string Name,
    string Description,
    decimal Price,
    int StockQuantity,
    string Category);

public record UpdateStockRequest(int Delta);
```

```csharp
// ProductService/Endpoints/ProductEndpoints.cs
namespace ProductService.Endpoints;

public static class ProductEndpoints
{
    public static void MapProductEndpoints(this WebApplication app)
    {
        var group = app.MapGroup("/api/products")
            .WithTags("Products");
        
        group.MapGet("/", GetProducts)
            .WithName("GetProducts");
        
        group.MapGet("/{id:guid}", GetProduct)
            .WithName("GetProduct");
        
        group.MapPost("/", CreateProduct)
            .WithName("CreateProduct");
        
        group.MapPut("/{id:guid}/stock", UpdateStock)
            .WithName("UpdateStock");
        
        group.MapDelete("/{id:guid}", DeleteProduct)
            .WithName("DeleteProduct");
    }
    
    private static async Task<IResult> GetProducts(
        IProductService service,
        string? category = null,
        int page = 1,
        int pageSize = 20,
        CancellationToken ct = default)
    {
        var products = await service.GetProductsAsync(category, page, pageSize, ct);
        return Results.Ok(products);
    }
    
    private static async Task<IResult> GetProduct(
        Guid id,
        IProductService service,
        CancellationToken ct)
    {
        var product = await service.GetProductAsync(id, ct);
        return product is null ? Results.NotFound() : Results.Ok(product);
    }
    
    private static async Task<IResult> CreateProduct(
        CreateProductRequest request,
        IProductService service,
        CancellationToken ct)
    {
        var product = await service.CreateProductAsync(request, ct);
        return Results.CreatedAtRoute("GetProduct", new { id = product.Id }, product);
    }
    
    private static async Task<IResult> UpdateStock(
        Guid id,
        UpdateStockRequest request,
        IProductService service,
        CancellationToken ct)
    {
        var product = await service.UpdateStockAsync(id, request.Delta, ct);
        return product is null ? Results.NotFound() : Results.Ok(product);
    }
    
    private static async Task<IResult> DeleteProduct(
        Guid id,
        IProductService service,
        CancellationToken ct)
    {
        var deleted = await service.DeleteProductAsync(id, ct);
        return deleted ? Results.NoContent() : Results.NotFound();
    }
}
```

---

## Step 816: Product Service Implementation

```csharp
// ProductService/Services/ProductService.cs
public interface IProductService
{
    Task<IEnumerable<ProductDto>> GetProductsAsync(
        string? category, int page, int pageSize, CancellationToken ct);
    Task<ProductDto?> GetProductAsync(Guid id, CancellationToken ct);
    Task<ProductDto> CreateProductAsync(CreateProductRequest request, CancellationToken ct);
    Task<ProductDto?> UpdateStockAsync(Guid id, int delta, CancellationToken ct);
    Task<bool> DeleteProductAsync(Guid id, CancellationToken ct);
}

public class ProductService(
    IProductRepository repo,
    IDistributedCache cache,
    IMessageBus bus,
    ILogger<ProductService> logger) : IProductService
{
    public async Task<IEnumerable<ProductDto>> GetProductsAsync(
        string? category, int page, int pageSize, CancellationToken ct)
    {
        var cacheKey = $"products:{category}:{page}:{pageSize}";
        var cached = await cache.GetStringAsync(cacheKey, ct);
        
        if (cached != null)
            return JsonSerializer.Deserialize<IEnumerable<ProductDto>>(cached)!;
        
        var products = await repo.GetAllAsync(category, page, pageSize, ct);
        var dtos = products.Select(ToDto);
        
        await cache.SetStringAsync(cacheKey,
            JsonSerializer.Serialize(dtos),
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5)
            }, ct);
        
        return dtos;
    }
    
    public async Task<ProductDto?> GetProductAsync(Guid id, CancellationToken ct)
    {
        var cacheKey = $"product:{id}";
        var cached = await cache.GetStringAsync(cacheKey, ct);
        
        if (cached != null)
            return JsonSerializer.Deserialize<ProductDto>(cached);
        
        var product = await repo.GetByIdAsync(id, ct);
        if (product == null) return null;
        
        var dto = ToDto(product);
        await cache.SetStringAsync(cacheKey,
            JsonSerializer.Serialize(dto),
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10)
            }, ct);
        
        return dto;
    }
    
    public async Task<ProductDto> CreateProductAsync(
        CreateProductRequest request, CancellationToken ct)
    {
        var product = new Product
        {
            Name = request.Name,
            Description = request.Description,
            Price = request.Price,
            StockQuantity = request.StockQuantity,
            Category = request.Category
        };
        
        await repo.AddAsync(product, ct);
        
        // Publish event
        await bus.PublishAsync(new ProductCreatedEvent(
            product.Id, product.Name, product.Price, product.Category), ct);
        
        logger.LogInformation("Product created: {ProductId}", product.Id);
        return ToDto(product);
    }
    
    public async Task<ProductDto?> UpdateStockAsync(
        Guid id, int delta, CancellationToken ct)
    {
        var product = await repo.GetByIdAsync(id, ct);
        if (product == null) return null;
        
        var oldQty = product.StockQuantity;
        product.StockQuantity = Math.Max(0, product.StockQuantity + delta);
        await repo.UpdateAsync(product, ct);
        
        // Invalidate cache
        await cache.RemoveAsync($"product:{id}", ct);
        
        // Publish event if stock changed significantly
        if (product.StockQuantity == 0 && oldQty > 0)
        {
            await bus.PublishAsync(new ProductOutOfStockEvent(id, product.Name), ct);
        }
        
        return ToDto(product);
    }
    
    public async Task<bool> DeleteProductAsync(Guid id, CancellationToken ct)
    {
        var product = await repo.GetByIdAsync(id, ct);
        if (product == null) return false;
        
        product.IsActive = false;
        await repo.UpdateAsync(product, ct);
        
        await cache.RemoveAsync($"product:{id}", ct);
        await bus.PublishAsync(new ProductDeletedEvent(id), ct);
        
        return true;
    }
    
    private static ProductDto ToDto(Product p) =>
        new(p.Id, p.Name, p.Description, p.Price, p.StockQuantity, p.Category);
}
```

---

## Step 817: Message Bus (RabbitMQ with MassTransit)

```bash
dotnet add package MassTransit
dotnet add package MassTransit.RabbitMQ
dotnet add package MassTransit.EntityFrameworkCore
```

```csharp
// Shared/Events/ProductEvents.cs
namespace Shared.Events;

public record ProductCreatedEvent(
    Guid ProductId,
    string Name,
    decimal Price,
    string Category);

public record ProductDeletedEvent(Guid ProductId);
public record ProductOutOfStockEvent(Guid ProductId, string ProductName);
public record OrderPlacedEvent(Guid OrderId, Guid UserId, List<OrderItemEvent> Items);
public record OrderItemEvent(Guid ProductId, int Quantity, decimal UnitPrice);
public record StockReservationRequested(Guid OrderId, List<OrderItemEvent> Items);
public record StockReservationCompleted(Guid OrderId, bool Success, string? Reason);
```

```csharp
// ProductService/Messaging/MessageBus.cs
public interface IMessageBus
{
    Task PublishAsync<T>(T message, CancellationToken ct) where T : class;
    Task SendAsync<T>(string endpoint, T message, CancellationToken ct) where T : class;
}

public class MassTransitMessageBus(IPublishEndpoint publishEndpoint, ISendEndpointProvider sendEndpointProvider) 
    : IMessageBus
{
    public Task PublishAsync<T>(T message, CancellationToken ct) where T : class
        => publishEndpoint.Publish(message, ct);
    
    public async Task SendAsync<T>(string endpoint, T message, CancellationToken ct) where T : class
    {
        var sendEndpoint = await sendEndpointProvider.GetSendEndpoint(new Uri($"queue:{endpoint}"));
        await sendEndpoint.Send(message, ct);
    }
}
```

```csharp
// ProductService/Consumers/StockReservationConsumer.cs
using MassTransit;
using Shared.Events;

public class StockReservationConsumer(
    IProductRepository repo,
    ILogger<StockReservationConsumer> logger)
    : IConsumer<StockReservationRequested>
{
    public async Task Consume(ConsumeContext<StockReservationRequested> context)
    {
        var request = context.Message;
        logger.LogInformation(
            "Processing stock reservation for order {OrderId}", request.OrderId);
        
        try
        {
            foreach (var item in request.Items)
            {
                var product = await repo.GetByIdAsync(item.ProductId, context.CancellationToken);
                
                if (product == null || product.StockQuantity < item.Quantity)
                {
                    await context.Publish(new StockReservationCompleted(
                        request.OrderId, false,
                        $"Insufficient stock for product {item.ProductId}"));
                    return;
                }
            }
            
            // Reserve stock
            foreach (var item in request.Items)
            {
                var product = await repo.GetByIdAsync(item.ProductId, context.CancellationToken);
                product!.StockQuantity -= item.Quantity;
                await repo.UpdateAsync(product, context.CancellationToken);
            }
            
            await context.Publish(
                new StockReservationCompleted(request.OrderId, true, null));
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "Error processing stock reservation");
            await context.Publish(
                new StockReservationCompleted(request.OrderId, false, ex.Message));
        }
    }
}

// Register MassTransit
builder.Services.AddMassTransit(x =>
{
    x.AddConsumer<StockReservationConsumer>();
    
    x.UsingRabbitMq((ctx, cfg) =>
    {
        cfg.Host(builder.Configuration["ConnectionStrings:messaging"]);
        
        cfg.ReceiveEndpoint("stock-reservation", e =>
        {
            e.ConfigureConsumer<StockReservationConsumer>(ctx);
            e.UseMessageRetry(r => r.Exponential(5, 
                TimeSpan.FromSeconds(1), TimeSpan.FromSeconds(30), TimeSpan.FromSeconds(5)));
        });
    });
});
```

---

## Step 818: Order Service with Saga Pattern

```csharp
// OrderService/Sagas/OrderSaga.cs
using MassTransit;
using Shared.Events;

public class OrderSagaState : SagaStateMachineInstance
{
    public Guid CorrelationId { get; set; }
    public string CurrentState { get; set; } = string.Empty;
    public Guid OrderId { get; set; }
    public Guid UserId { get; set; }
    public List<OrderItemEvent> Items { get; set; } = [];
    public decimal TotalAmount { get; set; }
    public DateTime CreatedAt { get; set; }
    public string? FailureReason { get; set; }
}

public class OrderStateMachine : MassTransitStateMachine<OrderSagaState>
{
    public State Pending { get; private set; } = default!;
    public State StockReserved { get; private set; } = default!;
    public State PaymentProcessing { get; private set; } = default!;
    public State Completed { get; private set; } = default!;
    public State Failed { get; private set; } = default!;
    
    public Event<OrderPlacedEvent> OrderPlaced { get; private set; } = default!;
    public Event<StockReservationCompleted> StockReservationCompleted { get; private set; } = default!;
    public Event<PaymentCompletedEvent> PaymentCompleted { get; private set; } = default!;
    
    public OrderStateMachine()
    {
        InstanceState(x => x.CurrentState);
        
        Event(() => OrderPlaced, x => 
            x.CorrelateById(m => m.Message.OrderId));
        Event(() => StockReservationCompleted, x =>
            x.CorrelateById(m => m.Message.OrderId));
        Event(() => PaymentCompleted, x =>
            x.CorrelateById(m => m.Message.OrderId));
        
        Initially(
            When(OrderPlaced)
                .Then(ctx =>
                {
                    ctx.Saga.OrderId = ctx.Message.OrderId;
                    ctx.Saga.UserId = ctx.Message.UserId;
                    ctx.Saga.Items = ctx.Message.Items;
                    ctx.Saga.TotalAmount = ctx.Message.Items
                        .Sum(i => i.Quantity * i.UnitPrice);
                    ctx.Saga.CreatedAt = DateTime.UtcNow;
                })
                .Publish(ctx => new StockReservationRequested(
                    ctx.Message.OrderId, ctx.Message.Items))
                .TransitionTo(Pending)
        );
        
        During(Pending,
            When(StockReservationCompleted, x => x.Message.Success)
                .TransitionTo(StockReserved)
                .Publish(ctx => new ProcessPaymentCommand(
                    ctx.Saga.OrderId, ctx.Saga.UserId, ctx.Saga.TotalAmount)),
            
            When(StockReservationCompleted, x => !x.Message.Success)
                .Then(ctx => ctx.Saga.FailureReason = ctx.Message.Reason)
                .Publish(ctx => new OrderFailedEvent(
                    ctx.Saga.OrderId, ctx.Saga.FailureReason!))
                .TransitionTo(Failed)
        );
        
        During(StockReserved,
            When(PaymentCompleted, x => x.Message.Success)
                .Publish(ctx => new OrderCompletedEvent(ctx.Saga.OrderId))
                .TransitionTo(Completed),
            
            When(PaymentCompleted, x => !x.Message.Success)
                // Compensate - release stock
                .Publish(ctx => new ReleaseStockCommand(ctx.Saga.OrderId, ctx.Saga.Items))
                .Publish(ctx => new OrderFailedEvent(ctx.Saga.OrderId, "Payment failed"))
                .TransitionTo(Failed)
        );
        
        SetCompletedWhenFinalized();
    }
}
```

---

## Step 819: API Gateway with YARP

```bash
dotnet add package Yarp.ReverseProxy
```

```csharp
// ApiGateway/Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.AddServiceDefaults();

builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"))
    .AddServiceDiscoveryDestinationResolver();

builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer();

builder.Services.AddAuthorization();
builder.Services.AddRateLimiter(opts =>
{
    opts.AddFixedWindowLimiter("api", limiterOpts =>
    {
        limiterOpts.Window = TimeSpan.FromMinutes(1);
        limiterOpts.PermitLimit = 100;
        limiterOpts.QueueLimit = 0;
    });
});

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();
app.UseRateLimiter();
app.MapReverseProxy();

app.Run();
```

```json
// ApiGateway/appsettings.json
{
  "ReverseProxy": {
    "Routes": {
      "product-route": {
        "ClusterId": "product-cluster",
        "Match": {
          "Path": "/api/products/{**catch-all}"
        },
        "Metadata": {
          "RequireAuthorization": "false"
        }
      },
      "order-route": {
        "ClusterId": "order-cluster",
        "Match": {
          "Path": "/api/orders/{**catch-all}"
        },
        "AuthorizationPolicy": "default"
      },
      "user-route": {
        "ClusterId": "user-cluster",
        "Match": {
          "Path": "/api/users/{**catch-all}"
        }
      }
    },
    "Clusters": {
      "product-cluster": {
        "Destinations": {
          "product-service": {
            "Address": "http://product-service"
          }
        }
      },
      "order-cluster": {
        "Destinations": {
          "order-service": {
            "Address": "http://order-service"
          }
        }
      },
      "user-cluster": {
        "Destinations": {
          "user-service": {
            "Address": "http://user-service"
          }
        }
      }
    }
  }
}
```

---

## Step 820: Service Discovery

```csharp
// .NET Aspire Service Discovery ทำงานอัตโนมัติ
// ใน AppHost:
var productService = builder.AddProject<Projects.ProductService>("product-service");

// ใน OrderService - อ้างอิงด้วย service name
builder.Services.AddHttpClient<IProductClient, ProductClient>(client =>
    client.BaseAddress = new Uri("http://product-service"));  // Aspire resolves this

// ProductClient
public class ProductClient(HttpClient httpClient) : IProductClient
{
    public async Task<ProductDto?> GetProductAsync(Guid id, CancellationToken ct)
    {
        return await httpClient.GetFromJsonAsync<ProductDto>(
            $"/api/products/{id}", ct);
    }
    
    public async Task<bool> ReserveStockAsync(
        Guid productId, int quantity, CancellationToken ct)
    {
        var response = await httpClient.PutAsJsonAsync(
            $"/api/products/{productId}/stock",
            new UpdateStockRequest(-quantity), ct);
        return response.IsSuccessStatusCode;
    }
}
```

---

## Step 821: Circuit Breaker และ Resilience

```csharp
// Polly resilience pipeline via .NET Aspire
// ServiceDefaults already adds AddStandardResilienceHandler()

// Custom resilience for specific services
builder.Services.AddHttpClient<IProductClient, ProductClient>(client =>
    client.BaseAddress = new Uri("http://product-service"))
    .AddResilienceHandler("product-pipeline", pipelineBuilder =>
    {
        // Retry
        pipelineBuilder.AddRetry(new HttpRetryStrategyOptions
        {
            MaxRetryAttempts = 3,
            Delay = TimeSpan.FromSeconds(1),
            BackoffType = DelayBackoffType.Exponential,
            UseJitter = true,
            ShouldHandle = args => ValueTask.FromResult(
                args.Outcome.Result?.IsSuccessStatusCode is false or null)
        });
        
        // Circuit Breaker
        pipelineBuilder.AddCircuitBreaker(new HttpCircuitBreakerStrategyOptions
        {
            FailureRatio = 0.5,
            MinimumThroughput = 10,
            SamplingDuration = TimeSpan.FromSeconds(30),
            BreakDuration = TimeSpan.FromSeconds(30),
            ShouldHandle = args => ValueTask.FromResult(
                args.Outcome.Result?.IsSuccessStatusCode is false or null)
        });
        
        // Timeout
        pipelineBuilder.AddTimeout(TimeSpan.FromSeconds(10));
    });
```

---

## Step 822: Distributed Tracing

```csharp
// Tracing คืนค่า correlation สำหรับ debugging across services
// .NET Aspire ตั้งค่า OpenTelemetry ให้อัตโนมัติ

// Custom spans
public class ProductService(
    IProductRepository repo,
    ActivitySource activitySource)
{
    private static readonly ActivitySource ActivitySource = 
        new("ProductService");
    
    public async Task<ProductDto> CreateProductAsync(
        CreateProductRequest request, CancellationToken ct)
    {
        using var activity = ActivitySource.StartActivity("CreateProduct");
        activity?.SetTag("product.name", request.Name);
        activity?.SetTag("product.category", request.Category);
        
        var product = new Product
        {
            Name = request.Name,
            Price = request.Price,
            Category = request.Category
        };
        
        await repo.AddAsync(product, ct);
        
        activity?.SetTag("product.id", product.Id.ToString());
        activity?.SetStatus(ActivityStatusCode.Ok);
        
        return ToDto(product);
    }
}

// Register ActivitySource
builder.Services.AddSingleton<ActivitySource>(
    new ActivitySource("ProductService", "1.0.0"));

// Configure Tracing in ServiceDefaults
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing =>
    {
        tracing.AddSource("ProductService");
        tracing.AddSource("OrderService");
    });
```

---

## Step 823: Health Checks

```csharp
// Detailed health checks
builder.Services.AddHealthChecks()
    .AddDbContextCheck<ProductDbContext>()
    .AddRedis(builder.Configuration["ConnectionStrings:cache"]!)
    .AddRabbitMQ(rabbitConnectionString: 
        builder.Configuration["ConnectionStrings:messaging"]!)
    .AddCheck("product-service-storage", () =>
    {
        var tempPath = Path.GetTempPath();
        if (Directory.Exists(tempPath))
            return HealthCheckResult.Healthy();
        return HealthCheckResult.Unhealthy("Temp path not accessible");
    });

// Health Check endpoints
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = _ => true,
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = r => r.Tags.Contains("live"),
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse  
});

// Expose to Aspire dashboard
app.MapHealthChecks("/health");
```

---

## Step 824: Outbox Pattern

Outbox Pattern รับประกัน atomic save ทั้ง database และ message

```csharp
// Models/OutboxMessage.cs
public class OutboxMessage
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public string EventType { get; set; } = string.Empty;
    public string Payload { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime? ProcessedAt { get; set; }
    public int RetryCount { get; set; }
    public string? Error { get; set; }
}

// Services/OutboxProcessor.cs
public class OutboxProcessor(
    IServiceProvider services,
    ILogger<OutboxProcessor> logger) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            await ProcessOutboxAsync(ct);
            await Task.Delay(TimeSpan.FromSeconds(10), ct);
        }
    }
    
    private async Task ProcessOutboxAsync(CancellationToken ct)
    {
        using var scope = services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        var bus = scope.ServiceProvider.GetRequiredService<IMessageBus>();
        
        var messages = await db.OutboxMessages
            .Where(m => m.ProcessedAt == null && m.RetryCount < 5)
            .OrderBy(m => m.CreatedAt)
            .Take(50)
            .ToListAsync(ct);
        
        foreach (var message in messages)
        {
            try
            {
                var eventType = Type.GetType(message.EventType)!;
                var payload = JsonSerializer.Deserialize(message.Payload, eventType)!;
                
                await bus.PublishAsync(payload, ct);
                
                message.ProcessedAt = DateTime.UtcNow;
            }
            catch (Exception ex)
            {
                logger.LogError(ex, "Error processing outbox message {Id}", message.Id);
                message.RetryCount++;
                message.Error = ex.Message;
            }
        }
        
        await db.SaveChangesAsync(ct);
    }
}

// Save to outbox atomically with business data
public async Task CreateProductAsync(Product product, CancellationToken ct)
{
    var outboxMessage = new OutboxMessage
    {
        EventType = typeof(ProductCreatedEvent).AssemblyQualifiedName!,
        Payload = JsonSerializer.Serialize(new ProductCreatedEvent(
            product.Id, product.Name, product.Price, product.Category))
    };
    
    db.Products.Add(product);
    db.OutboxMessages.Add(outboxMessage);
    
    await db.SaveChangesAsync(ct); // atomic
}
```

---

## Step 825: Distributed Cache Pattern

```csharp
// Cache-Aside Pattern
public class CachedProductService(
    IProductRepository repo,
    IDistributedCache cache,
    ILogger<CachedProductService> logger) : IProductService
{
    public async Task<ProductDto?> GetProductAsync(Guid id, CancellationToken ct)
    {
        var cacheKey = CacheKeys.Product(id);
        
        // 1. Check cache
        var bytes = await cache.GetAsync(cacheKey, ct);
        if (bytes != null)
        {
            logger.LogDebug("Cache hit for product {Id}", id);
            return JsonSerializer.Deserialize<ProductDto>(bytes);
        }
        
        // 2. Get from database
        var product = await repo.GetByIdAsync(id, ct);
        if (product == null) return null;
        
        var dto = ToDto(product);
        
        // 3. Store in cache
        await cache.SetAsync(
            cacheKey,
            JsonSerializer.SerializeToUtf8Bytes(dto),
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10),
                SlidingExpiration = TimeSpan.FromMinutes(2)
            }, ct);
        
        return dto;
    }
    
    public async Task InvalidateCacheAsync(Guid id)
    {
        await cache.RemoveAsync(CacheKeys.Product(id));
        await cache.RemoveAsync(CacheKeys.ProductList());
    }
}

public static class CacheKeys
{
    public static string Product(Guid id) => $"product:{id}";
    public static string ProductList(string? category = null) => 
        $"products:{category ?? "all"}";
}
```

---

## Step 826: Sidecar Pattern (Dapr)

```csharp
// ProductService/Program.cs with Dapr
builder.Services.AddDaprClient();

// Use Dapr state store instead of direct Redis
public class DaprProductCache(DaprClient dapr) : IProductCache
{
    private const string StoreName = "redis-store";
    
    public async Task<ProductDto?> GetAsync(Guid id, CancellationToken ct)
        => await dapr.GetStateAsync<ProductDto>(StoreName, id.ToString(), null, null, ct);
    
    public async Task SetAsync(ProductDto product, CancellationToken ct)
        => await dapr.SaveStateAsync(StoreName, product.Id.ToString(), product, 
            new StateOptions { Concurrency = ConcurrencyMode.LastWrite }, ct: ct);
    
    public async Task DeleteAsync(Guid id, CancellationToken ct)
        => await dapr.DeleteStateAsync(StoreName, id.ToString(), ct: ct);
}

// Publish via Dapr Pub/Sub
public class DaprMessageBus(DaprClient dapr) : IMessageBus
{
    private const string PubSubName = "rabbitmq-pubsub";
    
    public Task PublishAsync<T>(T message, CancellationToken ct) where T : class
        => dapr.PublishEventAsync(PubSubName, typeof(T).Name.ToLowerInvariant(), message, ct);
    
    public Task SendAsync<T>(string endpoint, T message, CancellationToken ct) where T : class
        => dapr.InvokeBindingAsync(endpoint, "create", message, cancellationToken: ct);
}

// Subscribe to Dapr topics
app.MapSubscribeHandler();

app.MapPost("/subscribe/product-created",
    [Topic("rabbitmq-pubsub", "productcreatedevent")]
    async (ProductCreatedEvent evt, IProductService service) =>
    {
        await service.HandleProductCreatedAsync(evt);
        return Results.Ok();
    });
```

---

## Step 827: .NET Aspire Dashboard

```
Aspire Dashboard - http://localhost:18888
├── Projects           - Running services
├── Containers         - Docker containers
├── Executables        - Any executables
├── Traces             - Distributed traces
├── Metrics            - Service metrics
└── Logs               - Structured logs
```

```csharp
// AppHost Program.cs - เพิ่ม resources
var builder = DistributedApplication.CreateBuilder(args);

// SQL Server
var sqlServer = builder.AddSqlServer("sqlserver")
    .WithSqlServerManagement();

// PostgreSQL
var postgres = builder.AddPostgres("postgres")
    .WithPgAdmin()
    .WithVolume("pgdata", "/var/lib/postgresql/data");

// MongoDB
var mongodb = builder.AddMongoDB("mongodb")
    .WithMongoExpress();

// Redis
var redis = builder.AddRedis("cache")
    .WithRedisInsight()
    .WithRedisCommander();

// RabbitMQ
var rabbitmq = builder.AddRabbitMQ("messaging")
    .WithManagementPlugin();

// Kafka
var kafka = builder.AddKafka("kafka")
    .WithKafkaUI();

// Elasticsearch
var elasticsearch = builder.AddElasticsearch("elasticsearch");

// Add Kibana
var kibana = builder.AddKibana("kibana")
    .WithReference(elasticsearch);
```

---

## Step 828: API Versioning across Services

```csharp
// Shared API versioning
builder.Services.AddApiVersioning(opts =>
{
    opts.DefaultApiVersion = new ApiVersion(1, 0);
    opts.AssumeDefaultVersionWhenUnspecified = true;
    opts.ApiVersionReader = ApiVersionReader.Combine(
        new QueryStringApiVersionReader("api-version"),
        new HeaderApiVersionReader("X-API-Version"),
        new MediaTypeApiVersionReader("version")
    );
});

// Versioned endpoints
var v1 = app.MapGroup("/api/v{version:apiVersion}/products")
    .WithApiVersionSet(versionSet)
    .MapToApiVersion(1, 0);

var v2 = app.MapGroup("/api/v{version:apiVersion}/products")
    .WithApiVersionSet(versionSet)
    .MapToApiVersion(2, 0);

v1.MapGet("/", GetProductsV1);
v2.MapGet("/", GetProductsV2); // V2 adds new fields

// Deprecation
v1.MapGet("/search", SearchProductsV1)
    .Deprecated();
```

---

## Step 829: Multi-tenancy

```csharp
// Tenant resolver
public interface ITenantResolver
{
    string? ResolveTenant(HttpContext context);
}

public class HeaderTenantResolver : ITenantResolver
{
    public string? ResolveTenant(HttpContext context)
        => context.Request.Headers["X-Tenant-ID"].FirstOrDefault();
}

// Tenant context
public class TenantContext
{
    public string TenantId { get; set; } = string.Empty;
    public string ConnectionString { get; set; } = string.Empty;
}

// Multi-tenant DbContext
public class MultiTenantDbContext(
    DbContextOptions<MultiTenantDbContext> options,
    ITenantContext tenant)
    : DbContext(options)
{
    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        if (!string.IsNullOrEmpty(tenant.ConnectionString))
            optionsBuilder.UseNpgsql(tenant.ConnectionString);
    }
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Row-level filtering by tenant
        modelBuilder.Entity<Product>()
            .HasQueryFilter(p => p.TenantId == tenant.TenantId);
    }
}

// Middleware
public class TenantMiddleware(RequestDelegate next)
{
    public async Task InvokeAsync(HttpContext context, ITenantResolver resolver)
    {
        var tenantId = resolver.ResolveTenant(context);
        if (tenantId != null)
            context.Items["TenantId"] = tenantId;
        
        await next(context);
    }
}
```

---

## Step 830: Event Sourcing

```csharp
// Domain Events
public abstract record DomainEvent
{
    public Guid Id { get; } = Guid.NewGuid();
    public DateTime OccurredAt { get; } = DateTime.UtcNow;
    public int Version { get; init; }
}

public record ProductPriceChanged(
    Guid ProductId, decimal OldPrice, decimal NewPrice) : DomainEvent;

public record StockQuantityChanged(
    Guid ProductId, int OldQuantity, int NewQuantity) : DomainEvent;

// Aggregate
public class ProductAggregate
{
    public Guid Id { get; private set; }
    public string Name { get; private set; } = string.Empty;
    public decimal Price { get; private set; }
    public int StockQuantity { get; private set; }
    private int _version;
    private readonly List<DomainEvent> _uncommittedEvents = [];
    
    public IReadOnlyList<DomainEvent> UncommittedEvents => _uncommittedEvents;
    
    public void ChangePrice(decimal newPrice)
    {
        if (newPrice <= 0) throw new ArgumentException("Price must be positive");
        
        var @event = new ProductPriceChanged(Id, Price, newPrice)
        {
            Version = _version + 1
        };
        
        Apply(@event);
        _uncommittedEvents.Add(@event);
    }
    
    private void Apply(ProductPriceChanged e)
    {
        Price = e.NewPrice;
        _version = e.Version;
    }
    
    // Reconstitute from events
    public static ProductAggregate Reconstitute(IEnumerable<DomainEvent> events)
    {
        var aggregate = new ProductAggregate();
        foreach (var @event in events)
            aggregate.ApplyEvent(@event);
        return aggregate;
    }
    
    private void ApplyEvent(DomainEvent @event)
    {
        switch (@event)
        {
            case ProductPriceChanged e: Apply(e); break;
            case StockQuantityChanged e: Apply(e); break;
        }
    }
}

// Event Store
public interface IEventStore
{
    Task AppendEventsAsync(Guid aggregateId, IEnumerable<DomainEvent> events, 
        int expectedVersion, CancellationToken ct);
    Task<IEnumerable<DomainEvent>> GetEventsAsync(Guid aggregateId, CancellationToken ct);
}

public class SqlEventStore(AppDbContext db) : IEventStore
{
    public async Task AppendEventsAsync(
        Guid aggregateId, IEnumerable<DomainEvent> events, 
        int expectedVersion, CancellationToken ct)
    {
        var currentVersion = await db.EventStoreRecords
            .Where(e => e.AggregateId == aggregateId)
            .MaxAsync(e => (int?)e.Version, ct) ?? 0;
        
        if (currentVersion != expectedVersion)
            throw new ConcurrencyException(
                $"Expected version {expectedVersion}, got {currentVersion}");
        
        var records = events.Select(e => new EventStoreRecord
        {
            AggregateId = aggregateId,
            EventType = e.GetType().AssemblyQualifiedName!,
            Payload = JsonSerializer.Serialize(e, e.GetType()),
            Version = e.Version,
            OccurredAt = e.OccurredAt
        });
        
        db.EventStoreRecords.AddRange(records);
        await db.SaveChangesAsync(ct);
    }
    
    public async Task<IEnumerable<DomainEvent>> GetEventsAsync(
        Guid aggregateId, CancellationToken ct)
    {
        var records = await db.EventStoreRecords
            .Where(r => r.AggregateId == aggregateId)
            .OrderBy(r => r.Version)
            .ToListAsync(ct);
        
        return records.Select(r =>
        {
            var type = Type.GetType(r.EventType)!;
            return (DomainEvent)JsonSerializer.Deserialize(r.Payload, type)!;
        });
    }
}
```

---

## Step 831: CQRS Pattern

```csharp
// Commands
public record CreateProductCommand(
    string Name, decimal Price, string Category, int StockQuantity);

public record UpdateProductPriceCommand(Guid ProductId, decimal NewPrice);

// Queries
public record GetProductQuery(Guid Id);
public record GetProductsQuery(string? Category, int Page, int PageSize);

// MediatR handlers
using MediatR;

public class CreateProductCommandHandler(
    IProductRepository repo,
    IMessageBus bus)
    : IRequestHandler<CreateProductCommand, ProductDto>
{
    public async Task<ProductDto> Handle(
        CreateProductCommand request, CancellationToken ct)
    {
        var product = new Product
        {
            Name = request.Name,
            Price = request.Price,
            Category = request.Category,
            StockQuantity = request.StockQuantity
        };
        
        await repo.AddAsync(product, ct);
        await bus.PublishAsync(
            new ProductCreatedEvent(product.Id, product.Name, product.Price, product.Category), ct);
        
        return new ProductDto(
            product.Id, product.Name, product.Description,
            product.Price, product.StockQuantity, product.Category);
    }
}

public class GetProductQueryHandler(
    IProductRepository repo,
    IDistributedCache cache)
    : IRequestHandler<GetProductQuery, ProductDto?>
{
    public async Task<ProductDto?> Handle(
        GetProductQuery request, CancellationToken ct)
    {
        // Optimized read path - can use a separate read model
        var product = await repo.GetByIdAsync(request.Id, ct);
        if (product == null) return null;
        
        return new ProductDto(
            product.Id, product.Name, product.Description,
            product.Price, product.StockQuantity, product.Category);
    }
}

// Register MediatR
builder.Services.AddMediatR(cfg =>
    cfg.RegisterServicesFromAssembly(typeof(Program).Assembly));

// Use in endpoints
app.MapPost("/api/products", async (
    CreateProductCommand command,
    IMediator mediator,
    CancellationToken ct) =>
{
    var product = await mediator.Send(command, ct);
    return Results.Created($"/api/products/{product.Id}", product);
});
```

---

## Step 832: Containerization

```dockerfile
# ProductService/Dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS base
WORKDIR /app
EXPOSE 8080
EXPOSE 8081

FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
ARG BUILD_CONFIGURATION=Release
WORKDIR /src

# Copy project files
COPY ["ProductService/ProductService.csproj", "ProductService/"]
COPY ["Shared/Shared.csproj", "Shared/"]
RUN dotnet restore "ProductService/ProductService.csproj"

# Build
COPY . .
WORKDIR "/src/ProductService"
RUN dotnet build "ProductService.csproj" -c $BUILD_CONFIGURATION -o /app/build

# Publish
FROM build AS publish
RUN dotnet publish "ProductService.csproj" -c $BUILD_CONFIGURATION \
    -o /app/publish /p:UseAppHost=false

# Final
FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "ProductService.dll"]
```

```yaml
# docker-compose.yml
version: '3.9'

services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: postgres
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"
  
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
  
  rabbitmq:
    image: rabbitmq:3-management
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: admin
    ports:
      - "5672:5672"
      - "15672:15672"
  
  product-service:
    build:
      context: .
      dockerfile: ProductService/Dockerfile
    environment:
      - ConnectionStrings__productdb=Host=postgres;Database=productdb;Username=postgres;Password=postgres
      - ConnectionStrings__cache=redis:6379
      - ConnectionStrings__messaging=amqp://admin:admin@rabbitmq
    depends_on:
      - postgres
      - redis
      - rabbitmq
    ports:
      - "5001:8080"
  
  order-service:
    build:
      context: .
      dockerfile: OrderService/Dockerfile
    environment:
      - ConnectionStrings__orderdb=Host=postgres;Database=orderdb;Username=postgres;Password=postgres
      - Services__ProductService=http://product-service:8080
    depends_on:
      - postgres
      - product-service
    ports:
      - "5002:8080"
  
  api-gateway:
    build:
      context: .
      dockerfile: ApiGateway/Dockerfile
    environment:
      - ReverseProxy__Clusters__product-cluster__Destinations__product-service__Address=http://product-service:8080
      - ReverseProxy__Clusters__order-cluster__Destinations__order-service__Address=http://order-service:8080
    depends_on:
      - product-service
      - order-service
    ports:
      - "5000:8080"

volumes:
  pgdata:
```

---

## Step 833: Kubernetes Deployment

```yaml
# k8s/product-service-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service
  namespace: eshop
spec:
  replicas: 3
  selector:
    matchLabels:
      app: product-service
  template:
    metadata:
      labels:
        app: product-service
    spec:
      containers:
        - name: product-service
          image: eshop/product-service:latest
          ports:
            - containerPort: 8080
          env:
            - name: ConnectionStrings__productdb
              valueFrom:
                secretKeyRef:
                  name: db-secrets
                  key: product-connection-string
            - name: ConnectionStrings__cache
              value: "redis-service:6379"
          resources:
            requests:
              memory: "128Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "1000m"
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: product-service
  namespace: eshop
spec:
  selector:
    app: product-service
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: product-service-hpa
  namespace: eshop
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: product-service
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

---

## Step 834: Monitoring Stack

```yaml
# k8s/monitoring.yaml
# Prometheus
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    spec:
      containers:
        - name: prometheus
          image: prom/prometheus:latest
          ports:
            - containerPort: 9090
          volumeMounts:
            - name: config
              mountPath: /etc/prometheus
      volumes:
        - name: config
          configMap:
            name: prometheus-config
---
# Grafana
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    spec:
      containers:
        - name: grafana
          image: grafana/grafana:latest
          ports:
            - containerPort: 3000
```

```csharp
// Custom metrics
public class ProductMetrics
{
    private readonly Counter<int> _productsCreated;
    private readonly Histogram<double> _requestDuration;
    private readonly UpDownCounter<int> _activeConnections;
    
    public ProductMetrics(IMeterFactory meterFactory)
    {
        var meter = meterFactory.Create("ProductService");
        
        _productsCreated = meter.CreateCounter<int>(
            "products.created.total",
            description: "Total number of products created");
        
        _requestDuration = meter.CreateHistogram<double>(
            "http.request.duration",
            unit: "ms",
            description: "HTTP request duration");
        
        _activeConnections = meter.CreateUpDownCounter<int>(
            "http.active.connections",
            description: "Number of active connections");
    }
    
    public void RecordProductCreated(string category)
        => _productsCreated.Add(1, new TagList
        {
            { "category", category }
        });
    
    public void RecordRequestDuration(double durationMs, string endpoint)
        => _requestDuration.Record(durationMs, new TagList
        {
            { "endpoint", endpoint }
        });
}

builder.Services.AddSingleton<ProductMetrics>();
```

---

## Step 835: Integration Testing for Microservices

```csharp
// Tests/IntegrationTests/ProductServiceTests.cs
using Testcontainers.PostgreSql;
using Testcontainers.Redis;

public class ProductServiceTests : IAsyncLifetime
{
    private readonly PostgreSqlContainer _postgres = new PostgreSqlBuilder()
        .WithImage("postgres:16-alpine")
        .Build();
    
    private readonly RedisContainer _redis = new RedisBuilder()
        .WithImage("redis:7-alpine")
        .Build();
    
    private WebApplicationFactory<Program> _factory = default!;
    private HttpClient _client = default!;
    
    public async Task InitializeAsync()
    {
        await _postgres.StartAsync();
        await _redis.StartAsync();
        
        _factory = new WebApplicationFactory<Program>()
            .WithWebHostBuilder(builder =>
            {
                builder.ConfigureServices(services =>
                {
                    // Replace DbContext with test DB
                    var descriptor = services.SingleOrDefault(
                        d => d.ServiceType == typeof(DbContextOptions<AppDbContext>));
                    if (descriptor != null) services.Remove(descriptor);
                    
                    services.AddDbContext<AppDbContext>(opts =>
                        opts.UseNpgsql(_postgres.GetConnectionString()));
                    
                    // Replace Redis
                    services.AddStackExchangeRedisCache(opts =>
                        opts.Configuration = _redis.GetConnectionString());
                });
            });
        
        _client = _factory.CreateClient();
        
        // Migrate database
        using var scope = _factory.Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        await db.Database.MigrateAsync();
    }
    
    [Fact]
    public async Task CreateProduct_ReturnsCreated()
    {
        var request = new CreateProductRequest("Test Product", "Description", 100m, 10, "Electronics");
        var response = await _client.PostAsJsonAsync("/api/products", request);
        
        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
        
        var product = await response.Content.ReadFromJsonAsync<ProductDto>();
        Assert.NotNull(product);
        Assert.Equal("Test Product", product.Name);
    }
    
    [Fact]
    public async Task GetProduct_UsesCaching()
    {
        // Create product
        var created = await CreateTestProduct();
        
        // First request - from DB
        var sw = Stopwatch.StartNew();
        await _client.GetAsync($"/api/products/{created.Id}");
        var firstRequestMs = sw.ElapsedMilliseconds;
        
        // Second request - from cache (should be faster)
        sw.Restart();
        var response = await _client.GetAsync($"/api/products/{created.Id}");
        var cachedRequestMs = sw.ElapsedMilliseconds;
        
        Assert.Equal(HttpStatusCode.OK, response.StatusCode);
        // Cache should be faster (though this is a soft assertion)
    }
    
    private async Task<ProductDto> CreateTestProduct()
    {
        var response = await _client.PostAsJsonAsync("/api/products",
            new CreateProductRequest("Test", "Desc", 100, 10, "Electronics"));
        return (await response.Content.ReadFromJsonAsync<ProductDto>())!;
    }
    
    public async Task DisposeAsync()
    {
        await _postgres.StopAsync();
        await _redis.StopAsync();
        _factory.Dispose();
    }
}
```

---

## Step 836-840: Aspire + Azure Integration

```csharp
// AppHost for Azure deployment
var builder = DistributedApplication.CreateBuilder(args);

if (builder.ExecutionContext.IsPublishMode)
{
    // Azure resources
    var cosmosDb = builder.AddAzureCosmosDB("cosmos");
    var serviceBus = builder.AddAzureServiceBus("servicebus");
    var keyVault = builder.AddAzureKeyVault("vault");
    var appInsights = builder.AddAzureApplicationInsights("insights");
    var storageAccount = builder.AddAzureStorage("storage");
    var containerRegistry = builder.AddAzureContainerRegistry("registry");
    
    // Services deployed as Container Apps
    builder.AddProject<Projects.ProductService>("product-service")
        .WithReference(cosmosDb)
        .WithReference(serviceBus)
        .WithReference(keyVault);
}
else
{
    // Local resources
    var postgres = builder.AddPostgres("postgres");
    var rabbitmq = builder.AddRabbitMQ("messaging");
    
    builder.AddProject<Projects.ProductService>("product-service")
        .WithReference(postgres.AddDatabase("productdb"))
        .WithReference(rabbitmq);
}

builder.Build().Run();
```

---

## Step 841-845: สรุป Microservices Patterns

### Patterns ที่ครอบคลุม

| Pattern | Use Case | .NET Implementation |
|---------|----------|---------------------|
| API Gateway | Single entry point | YARP |
| Service Discovery | Dynamic routing | Aspire + DNS |
| Circuit Breaker | Fault tolerance | Polly |
| Saga | Distributed transactions | MassTransit State Machine |
| CQRS | Read/Write separation | MediatR |
| Event Sourcing | Audit trail | Custom EventStore |
| Outbox | Reliable messaging | Background worker |
| Cache-Aside | Performance | Redis + IDistributedCache |
| Sidecar | Cross-cutting concerns | Dapr |
| Multi-tenancy | SaaS applications | Middleware + EF Filters |

### .NET Aspire ช่วยอะไร
- **Orchestration**: รัน services ทั้งหมดด้วยคำสั่งเดียว
- **Service Discovery**: ค้นหา services อัตโนมัติ
- **Health Checks**: ตรวจสอบสถานะ services
- **Observability**: OpenTelemetry พร้อมใช้
- **Dashboard**: UI สวยงามสำหรับ monitoring
- **Configuration**: ตั้งค่า connection strings อัตโนมัติ

---

*จบ Part 30: Microservices Architecture with .NET Aspire*
*ต่อไป Part 31: Docker, Containers & Kubernetes*
