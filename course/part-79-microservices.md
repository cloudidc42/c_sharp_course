# Part 79: Microservices Architecture — API Gateway, Service Discovery & Communication

## Steps 1893–1908

---

## Step 1893: Microservices Fundamentals

Each service owns its domain, database, and deployment lifecycle.

```
Microservices Principles:
├── Single Responsibility — one bounded context per service
├── Own Data — no shared databases (polyglot persistence OK)
├── Communicate via API or Events — no shared libraries for logic
├── Independent Deploy — CI/CD per service
├── Design for Failure — circuit breakers, retries, timeouts
└── Observability — distributed tracing across services
```

**Domain decomposition** (example e-commerce):
```
OrderService     → Orders, OrderItems, Fulfillment
ProductService   → Catalog, Inventory, Pricing
CustomerService  → Accounts, Addresses, Preferences
PaymentService   → Transactions, Refunds, Methods
NotifyService    → Emails, SMS, Push notifications
SearchService    → Full-text search, Recommendations
```

---

## Step 1894: Solution Structure

```
ECommerce/
├── src/
│   ├── Services/
│   │   ├── Orders/
│   │   │   ├── Orders.Api/
│   │   │   ├── Orders.Application/
│   │   │   ├── Orders.Domain/
│   │   │   └── Orders.Infrastructure/
│   │   ├── Products/
│   │   │   ├── Products.Api/
│   │   │   ├── Products.Application/
│   │   │   └── ...
│   │   └── Customers/
│   │       └── ...
│   ├── ApiGateway/
│   │   └── Gateway.Api/
│   └── Shared/
│       ├── Shared.Contracts/        ← Integration event types
│       ├── Shared.Infrastructure/   ← Common infra (messaging, etc.)
│       └── Shared.Observability/    ← Tracing, metrics, logging
├── tests/
│   ├── Integration/
│   └── E2E/
├── docker-compose.yml
├── docker-compose.override.yml
└── ECommerce.sln
```

```xml
<!-- Directory.Build.props — shared across all services -->
<Project>
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
  </PropertyGroup>
</Project>
```

---

## Step 1895: YARP — Yet Another Reverse Proxy (API Gateway)

```bash
cd src/ApiGateway/Gateway.Api
dotnet add package Yarp.ReverseProxy
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
```

```json
// appsettings.json
{
  "ReverseProxy": {
    "Routes": {
      "orders-route": {
        "ClusterId": "orders-cluster",
        "AuthorizationPolicy": "default",
        "Match": {
          "Path": "/api/orders/{**catch-all}"
        },
        "Transforms": [
          { "PathRemovePrefix": "/api" },
          { "RequestHeader": "X-Gateway-Version", "Set": "1.0" }
        ]
      },
      "products-route": {
        "ClusterId": "products-cluster",
        "Match": {
          "Path": "/api/products/{**catch-all}"
        },
        "Transforms": [
          { "PathRemovePrefix": "/api" }
        ]
      },
      "customers-route": {
        "ClusterId": "customers-cluster",
        "AuthorizationPolicy": "default",
        "Match": {
          "Path": "/api/customers/{**catch-all}"
        },
        "Transforms": [
          { "PathRemovePrefix": "/api" }
        ]
      }
    },
    "Clusters": {
      "orders-cluster": {
        "LoadBalancingPolicy": "RoundRobin",
        "Destinations": {
          "orders-1": { "Address": "http://orders-api:8080/" },
          "orders-2": { "Address": "http://orders-api-2:8080/" }
        },
        "HealthCheck": {
          "Active": {
            "Enabled": true,
            "Interval": "00:00:10",
            "Timeout": "00:00:10",
            "Policy": "ConsecutiveFailures",
            "Path": "/health"
          }
        }
      },
      "products-cluster": {
        "Destinations": {
          "products-1": { "Address": "http://products-api:8080/" }
        }
      },
      "customers-cluster": {
        "Destinations": {
          "customers-1": { "Address": "http://customers-api:8080/" }
        }
      }
    }
  }
}
```

```csharp
// Program.cs (Gateway)
using Yarp.ReverseProxy.Transforms;

var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"))
    .AddTransforms(context =>
    {
        // Propagate correlation ID
        context.AddRequestTransform(async transformContext =>
        {
            var correlationId = transformContext.HttpContext.Request.Headers["X-Correlation-ID"]
                .FirstOrDefault() ?? Guid.NewGuid().ToString();

            transformContext.ProxyRequest.Headers.TryAddWithoutValidation(
                "X-Correlation-ID", correlationId);
        });
    });

builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(opts => { /* ... JWT validation */ });

builder.Services.AddAuthorization();
builder.Services.AddHealthChecks();

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();
app.MapHealthChecks("/health");
app.MapReverseProxy();

app.Run();
```

---

## Step 1896: Service-to-Service Communication — HttpClient with Polly

```csharp
// Shared.Infrastructure/Http/ServiceHttpClientFactory.cs
namespace Shared.Infrastructure.Http;

public static class ServiceHttpClientExtensions
{
    public static IHttpClientBuilder AddServiceClient<TClient, TImplementation>(
        this IServiceCollection services,
        string serviceBaseUrl)
        where TClient : class
        where TImplementation : class, TClient
    {
        return services.AddHttpClient<TClient, TImplementation>(client =>
        {
            client.BaseAddress = new Uri(serviceBaseUrl);
            client.DefaultRequestHeaders.Add("Accept", "application/json");
            client.Timeout = TimeSpan.FromSeconds(30);
        })
        .AddStandardResilienceHandler(opts =>
        {
            opts.Retry.MaxRetryAttempts = 3;
            opts.Retry.Delay = TimeSpan.FromMilliseconds(200);
            opts.CircuitBreaker.SamplingDuration = TimeSpan.FromSeconds(30);
            opts.CircuitBreaker.FailureRatio = 0.5;
        })
        .AddHttpMessageHandler<CorrelationIdHandler>()
        .AddHttpMessageHandler<ServiceAuthHandler>();
    }
}
```

```csharp
// Handlers/CorrelationIdHandler.cs
public class CorrelationIdHandler : DelegatingHandler
{
    private readonly IHttpContextAccessor _accessor;

    public CorrelationIdHandler(IHttpContextAccessor accessor)
    {
        _accessor = accessor;
    }

    protected override Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, CancellationToken ct)
    {
        var correlationId = _accessor.HttpContext?.Request.Headers["X-Correlation-ID"]
            .FirstOrDefault() ?? Guid.NewGuid().ToString();

        request.Headers.TryAddWithoutValidation("X-Correlation-ID", correlationId);
        return base.SendAsync(request, ct);
    }
}
```

```csharp
// Handlers/ServiceAuthHandler.cs — M2M auth via client credentials
public class ServiceAuthHandler : DelegatingHandler
{
    private readonly ITokenProvider _tokenProvider;

    public ServiceAuthHandler(ITokenProvider tokenProvider)
    {
        _tokenProvider = tokenProvider;
    }

    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, CancellationToken ct)
    {
        var token = await _tokenProvider.GetAccessTokenAsync(ct);
        request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", token);
        return await base.SendAsync(request, ct);
    }
}
```

```csharp
// Services/ProductServiceClient.cs
public interface IProductServiceClient
{
    Task<ProductDto?> GetProductAsync(Guid productId, CancellationToken ct = default);
    Task<IReadOnlyList<ProductDto>> GetProductsAsync(IEnumerable<Guid> productIds,
        CancellationToken ct = default);
    Task<bool> ReserveStockAsync(Guid productId, int quantity, CancellationToken ct = default);
}

public class ProductServiceClient : IProductServiceClient
{
    private readonly HttpClient _http;
    private readonly ILogger<ProductServiceClient> _logger;

    public ProductServiceClient(HttpClient http, ILogger<ProductServiceClient> logger)
    {
        _http = http;
        _logger = logger;
    }

    public async Task<ProductDto?> GetProductAsync(Guid productId, CancellationToken ct = default)
    {
        try
        {
            return await _http.GetFromJsonAsync<ProductDto>($"/products/{productId}", ct);
        }
        catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.NotFound)
        {
            return null;
        }
    }

    public async Task<IReadOnlyList<ProductDto>> GetProductsAsync(
        IEnumerable<Guid> productIds, CancellationToken ct = default)
    {
        var ids = string.Join(",", productIds);
        var result = await _http.GetFromJsonAsync<List<ProductDto>>(
            $"/products?ids={ids}", ct);
        return result ?? [];
    }

    public async Task<bool> ReserveStockAsync(Guid productId, int quantity,
        CancellationToken ct = default)
    {
        var response = await _http.PostAsJsonAsync(
            $"/products/{productId}/reserve",
            new { Quantity = quantity },
            ct);
        return response.IsSuccessStatusCode;
    }
}
```

---

## Step 1897: Integration Events (Async Messaging via MassTransit)

```bash
dotnet add package MassTransit
dotnet add package MassTransit.RabbitMQ
dotnet add package MassTransit.EntityFrameworkCore
```

```csharp
// Shared.Contracts/Events/OrderEvents.cs
namespace Shared.Contracts.Events;

// Immutable event contracts shared across services
public record OrderPlaced(
    Guid OrderId,
    Guid CustomerId,
    IReadOnlyList<OrderPlacedItem> Items,
    decimal TotalAmount,
    string Currency,
    DateTimeOffset PlacedAt);

public record OrderPlacedItem(
    Guid ProductId,
    string Sku,
    int Quantity,
    decimal UnitPrice);

public record OrderCancelled(
    Guid OrderId,
    Guid CustomerId,
    string Reason,
    DateTimeOffset CancelledAt);

public record PaymentProcessed(
    Guid OrderId,
    Guid PaymentId,
    decimal Amount,
    bool Success,
    string? FailureCode = null);

public record StockReserved(
    Guid OrderId,
    Guid ReservationId,
    IReadOnlyList<Guid> ProductIds,
    DateTimeOffset ExpiresAt);

public record StockReservationFailed(
    Guid OrderId,
    Guid ProductId,
    int RequestedQuantity,
    int AvailableQuantity);
```

```csharp
// Orders.Infrastructure/Messaging/MassTransitConfiguration.cs
namespace Orders.Infrastructure.Messaging;

public static class MassTransitConfiguration
{
    public static IServiceCollection AddOrdersMessaging(
        this IServiceCollection services,
        IConfiguration config)
    {
        services.AddMassTransit(x =>
        {
            x.SetKebabCaseEndpointNameFormatter();

            // Register consumers
            x.AddConsumer<PaymentProcessedConsumer>();
            x.AddConsumer<StockReservedConsumer>();
            x.AddConsumer<StockReservationFailedConsumer>();

            // Saga for order fulfillment workflow
            x.AddSagaStateMachine<OrderFulfillmentStateMachine, OrderFulfillmentState>()
                .EntityFrameworkRepository(r =>
                {
                    r.ExistingDbContext<OrdersDbContext>();
                    r.UsePostgres();
                });

            x.UsingRabbitMq((context, cfg) =>
            {
                cfg.Host(config.GetConnectionString("RabbitMQ"), h =>
                {
                    h.Username(config["RabbitMQ:Username"] ?? "guest");
                    h.Password(config["RabbitMQ:Password"] ?? "guest");
                });

                cfg.UseMessageRetry(r => r.Exponential(5,
                    TimeSpan.FromSeconds(1), TimeSpan.FromSeconds(30),
                    TimeSpan.FromSeconds(2)));

                cfg.UseInMemoryOutbox(context);

                cfg.ConfigureEndpoints(context);
            });
        });

        return services;
    }
}
```

```csharp
// Orders.Application/Commands/PlaceOrderCommandHandler.cs
public class PlaceOrderCommandHandler : IRequestHandler<PlaceOrderCommand, PlaceOrderResult>
{
    private readonly IOrderRepository _orders;
    private readonly IPublishEndpoint _publish;
    private readonly IUnitOfWork _uow;

    public PlaceOrderCommandHandler(
        IOrderRepository orders,
        IPublishEndpoint publish,
        IUnitOfWork uow)
    {
        _orders = orders;
        _publish = publish;
        _uow = uow;
    }

    public async Task<PlaceOrderResult> Handle(PlaceOrderCommand cmd, CancellationToken ct)
    {
        var order = Order.Create(cmd.CustomerId, cmd.ShippingAddress);

        foreach (var item in cmd.Items)
            order.AddItem(item.ProductId, item.Sku, item.Quantity, item.UnitPrice);

        order.Submit();

        await _orders.AddAsync(order, ct);

        // Publish within same transaction (outbox pattern)
        await _publish.Publish(new OrderPlaced(
            order.Id,
            order.CustomerId,
            order.Items.Select(i => new OrderPlacedItem(
                i.ProductId, i.Sku, i.Quantity, i.UnitPrice)).ToList(),
            order.TotalAmount,
            "USD",
            order.PlacedAt), ct);

        await _uow.CommitAsync(ct);

        return new PlaceOrderResult(order.Id, order.Status.ToString());
    }
}
```

---

## Step 1898: Saga Pattern — Distributed Workflow Coordination

```csharp
// Orders.Application/Sagas/OrderFulfillmentStateMachine.cs
using MassTransit;

namespace Orders.Application.Sagas;

public class OrderFulfillmentState : SagaStateMachineInstance
{
    public Guid CorrelationId { get; set; }
    public string CurrentState { get; set; } = null!;
    public Guid OrderId { get; set; }
    public Guid CustomerId { get; set; }
    public decimal TotalAmount { get; set; }
    public Guid? ReservationId { get; set; }
    public Guid? PaymentId { get; set; }
    public DateTimeOffset? StockReservedAt { get; set; }
    public DateTimeOffset? PaymentProcessedAt { get; set; }
    public int RetryCount { get; set; }
}

public class OrderFulfillmentStateMachine : MassTransitStateMachine<OrderFulfillmentState>
{
    public State WaitingForStock { get; private set; } = null!;
    public State WaitingForPayment { get; private set; } = null!;
    public State Completed { get; private set; } = null!;
    public State Failed { get; private set; } = null!;

    public Event<OrderPlaced> OrderPlaced { get; private set; } = null!;
    public Event<StockReserved> StockReserved { get; private set; } = null!;
    public Event<StockReservationFailed> StockReservationFailed { get; private set; } = null!;
    public Event<PaymentProcessed> PaymentProcessed { get; private set; } = null!;

    public OrderFulfillmentStateMachine()
    {
        InstanceState(x => x.CurrentState);

        Event(() => OrderPlaced, x => x.CorrelateById(m => m.Message.OrderId));
        Event(() => StockReserved, x => x.CorrelateById(m => m.Message.OrderId));
        Event(() => StockReservationFailed, x => x.CorrelateById(m => m.Message.OrderId));
        Event(() => PaymentProcessed, x => x.CorrelateById(m => m.Message.OrderId));

        Initially(
            When(OrderPlaced)
                .Then(ctx =>
                {
                    ctx.Saga.OrderId = ctx.Message.OrderId;
                    ctx.Saga.CustomerId = ctx.Message.CustomerId;
                    ctx.Saga.TotalAmount = ctx.Message.TotalAmount;
                })
                .Publish(ctx => new ReserveStock(
                    ctx.Message.OrderId,
                    ctx.Message.Items.Select(i => new ReserveStockItem(
                        i.ProductId, i.Quantity)).ToList()))
                .TransitionTo(WaitingForStock));

        During(WaitingForStock,
            When(StockReserved)
                .Then(ctx =>
                {
                    ctx.Saga.ReservationId = ctx.Message.ReservationId;
                    ctx.Saga.StockReservedAt = ctx.Message.ExpiresAt;
                })
                .Publish(ctx => new ProcessPayment(
                    ctx.Saga.OrderId,
                    ctx.Saga.CustomerId,
                    ctx.Saga.TotalAmount))
                .TransitionTo(WaitingForPayment),

            When(StockReservationFailed)
                .Then(ctx => ctx.Saga.RetryCount++)
                .If(ctx => ctx.Saga.RetryCount < 3,
                    binder => binder.Publish(ctx => new CancelOrder(
                        ctx.Saga.OrderId, "Stock unavailable")))
                .TransitionTo(Failed));

        During(WaitingForPayment,
            When(PaymentProcessed, ctx => ctx.Message.Success)
                .Then(ctx => ctx.Saga.PaymentId = ctx.Message.PaymentId)
                .Publish(ctx => new ConfirmOrder(ctx.Saga.OrderId))
                .TransitionTo(Completed)
                .Finalize(),

            When(PaymentProcessed, ctx => !ctx.Message.Success)
                .Publish(ctx => new ReleaseStock(
                    ctx.Saga.OrderId, ctx.Saga.ReservationId!.Value))
                .Publish(ctx => new CancelOrder(
                    ctx.Saga.OrderId, $"Payment failed: {ctx.Message.FailureCode}"))
                .TransitionTo(Failed)
                .Finalize());

        SetCompletedWhenFinalized();
    }
}
```

---

## Step 1899: Service Discovery with Consul

```bash
dotnet add package Consul
dotnet add package Winton.Extensions.Configuration.Consul
```

```csharp
// Infrastructure/ServiceDiscovery/ConsulServiceRegistration.cs
using Consul;

namespace Orders.Infrastructure.ServiceDiscovery;

public class ConsulServiceRegistration : IHostedService
{
    private readonly IConsulClient _consulClient;
    private readonly IConfiguration _config;
    private readonly ILogger<ConsulServiceRegistration> _logger;
    private string? _registrationId;

    public ConsulServiceRegistration(
        IConsulClient consulClient,
        IConfiguration config,
        ILogger<ConsulServiceRegistration> logger)
    {
        _consulClient = consulClient;
        _config = config;
        _logger = logger;
    }

    public async Task StartAsync(CancellationToken ct)
    {
        var serviceName = "orders-api";
        var serviceId = $"{serviceName}-{Guid.NewGuid():N}";
        _registrationId = serviceId;

        var port = int.Parse(_config["PORT"] ?? "8080");

        var registration = new AgentServiceRegistration
        {
            ID = serviceId,
            Name = serviceName,
            Address = _config["SERVICE_HOST"] ?? "localhost",
            Port = port,
            Tags = ["orders", "api", "v1"],
            Check = new AgentServiceCheck
            {
                HTTP = $"http://localhost:{port}/health",
                Interval = TimeSpan.FromSeconds(10),
                Timeout = TimeSpan.FromSeconds(5),
                DeregisterCriticalServiceAfter = TimeSpan.FromMinutes(1)
            },
            Meta = new Dictionary<string, string>
            {
                ["version"] = typeof(ConsulServiceRegistration).Assembly.GetName().Version?.ToString() ?? "1.0.0"
            }
        };

        await _consulClient.Agent.ServiceRegister(registration, ct);
        _logger.LogInformation("Registered service {ServiceId} with Consul", serviceId);
    }

    public async Task StopAsync(CancellationToken ct)
    {
        if (_registrationId is not null)
        {
            await _consulClient.Agent.ServiceDeregister(_registrationId, ct);
            _logger.LogInformation("Deregistered service {ServiceId} from Consul", _registrationId);
        }
    }
}
```

```csharp
// ConsulHttpClient — discovers service endpoint dynamically
public class ConsulHttpClientFactory
{
    private readonly IConsulClient _consul;

    public ConsulHttpClientFactory(IConsulClient consul)
    {
        _consul = consul;
    }

    public async Task<HttpClient> CreateForServiceAsync(string serviceName)
    {
        var result = await _consul.Health.Service(serviceName, tag: null,
            passingOnly: true, QueryOptions.Default);

        var services = result.Response;
        if (services.Length == 0)
            throw new InvalidOperationException($"No healthy instances of '{serviceName}' found");

        // Simple random load balancing
        var service = services[Random.Shared.Next(services.Length)];
        var baseUrl = $"http://{service.Service.Address}:{service.Service.Port}";

        return new HttpClient { BaseAddress = new Uri(baseUrl) };
    }
}
```

---

## Step 1900: gRPC for Internal Service Communication

```bash
dotnet add package Grpc.AspNetCore
dotnet add package Grpc.Net.Client
dotnet add package Google.Protobuf
```

```protobuf
// Protos/inventory.proto
syntax = "proto3";
option csharp_namespace = "Products.Grpc";
package inventory;

service InventoryService {
  rpc GetStock (GetStockRequest) returns (GetStockResponse);
  rpc ReserveStock (ReserveStockRequest) returns (ReserveStockResponse);
  rpc ReleaseStock (ReleaseStockRequest) returns (ReleaseStockResponse);
  rpc StreamInventoryChanges (StreamInventoryRequest) returns (stream InventoryChange);
}

message GetStockRequest {
  repeated string product_ids = 1;
}

message GetStockResponse {
  repeated StockEntry items = 1;
}

message StockEntry {
  string product_id = 1;
  int32 available = 2;
  int32 reserved = 3;
}

message ReserveStockRequest {
  string order_id = 1;
  repeated StockReservationItem items = 2;
}

message StockReservationItem {
  string product_id = 1;
  int32 quantity = 2;
}

message ReserveStockResponse {
  bool success = 1;
  string reservation_id = 2;
  string failure_reason = 3;
}

message ReleaseStockRequest {
  string reservation_id = 1;
}

message ReleaseStockResponse {
  bool success = 1;
}

message StreamInventoryRequest {
  repeated string product_ids = 1;
}

message InventoryChange {
  string product_id = 1;
  int32 available = 2;
  int64 timestamp = 3;
}
```

```csharp
// Products.Api/Grpc/InventoryGrpcService.cs
using Grpc.Core;
using Products.Grpc;

namespace Products.Api.Grpc;

public class InventoryGrpcService : InventoryService.InventoryServiceBase
{
    private readonly IInventoryRepository _inventory;
    private readonly ILogger<InventoryGrpcService> _logger;

    public InventoryGrpcService(
        IInventoryRepository inventory,
        ILogger<InventoryGrpcService> logger)
    {
        _inventory = inventory;
        _logger = logger;
    }

    public override async Task<GetStockResponse> GetStock(
        GetStockRequest request, ServerCallContext context)
    {
        var productIds = request.ProductIds.Select(Guid.Parse).ToList();
        var stocks = await _inventory.GetStocksAsync(productIds, context.CancellationToken);

        var response = new GetStockResponse();
        response.Items.AddRange(stocks.Select(s => new StockEntry
        {
            ProductId = s.ProductId.ToString(),
            Available = s.Available,
            Reserved = s.Reserved
        }));

        return response;
    }

    public override async Task<ReserveStockResponse> ReserveStock(
        ReserveStockRequest request, ServerCallContext context)
    {
        try
        {
            var orderId = Guid.Parse(request.OrderId);
            var items = request.Items.Select(i => new ReservationItem(
                Guid.Parse(i.ProductId), i.Quantity)).ToList();

            var reservationId = await _inventory.ReserveAsync(orderId, items,
                context.CancellationToken);

            return new ReserveStockResponse
            {
                Success = true,
                ReservationId = reservationId.ToString()
            };
        }
        catch (InsufficientStockException ex)
        {
            return new ReserveStockResponse
            {
                Success = false,
                FailureReason = ex.Message
            };
        }
    }

    public override async Task StreamInventoryChanges(
        StreamInventoryRequest request,
        IServerStreamWriter<InventoryChange> responseStream,
        ServerCallContext context)
    {
        var productIds = request.ProductIds.Select(Guid.Parse).ToHashSet();

        await foreach (var change in _inventory.SubscribeToChangesAsync(
            productIds, context.CancellationToken))
        {
            await responseStream.WriteAsync(new InventoryChange
            {
                ProductId = change.ProductId.ToString(),
                Available = change.Available,
                Timestamp = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds()
            });
        }
    }
}
```

```csharp
// Orders service consumes gRPC
public class InventoryGrpcClient
{
    private readonly InventoryService.InventoryServiceClient _client;

    public InventoryGrpcClient(InventoryService.InventoryServiceClient client)
    {
        _client = client;
    }

    public async Task<bool> ReserveStockAsync(Guid orderId,
        IEnumerable<(Guid ProductId, int Quantity)> items, CancellationToken ct = default)
    {
        var request = new ReserveStockRequest { OrderId = orderId.ToString() };
        request.Items.AddRange(items.Select(i => new StockReservationItem
        {
            ProductId = i.ProductId.ToString(),
            Quantity = i.Quantity
        }));

        var response = await _client.ReserveStockAsync(request,
            cancellationToken: ct);

        return response.Success;
    }
}

// Registration
builder.Services.AddGrpcClient<InventoryService.InventoryServiceClient>(o =>
{
    o.Address = new Uri(builder.Configuration["Services:Products:GrpcUrl"]!);
})
.ConfigurePrimaryHttpMessageHandler(() => new HttpClientHandler
{
    ServerCertificateCustomValidationCallback =
        HttpClientHandler.DangerousAcceptAnyServerCertificateValidator
})
.AddStandardResilienceHandler();
```

---

## Step 1901: API Versioning

```bash
dotnet add package Asp.Versioning.Mvc
dotnet add package Asp.Versioning.Mvc.ApiExplorer
```

```csharp
// Program.cs
builder.Services.AddApiVersioning(opts =>
{
    opts.DefaultApiVersion = new ApiVersion(1, 0);
    opts.AssumeDefaultVersionWhenUnspecified = true;
    opts.ReportApiVersions = true;
    opts.ApiVersionReader = ApiVersionReader.Combine(
        new UrlSegmentApiVersionReader(),
        new HeaderApiVersionReader("X-API-Version"),
        new QueryStringApiVersionReader("api-version")
    );
})
.AddApiExplorer(opts =>
{
    opts.GroupNameFormat = "'v'VVV";
    opts.SubstituteApiVersionInUrl = true;
});
```

```csharp
// V1 Controller
[ApiController]
[ApiVersion("1.0")]
[Route("v{version:apiVersion}/orders")]
public class OrdersV1Controller : ControllerBase
{
    [HttpGet("{id:guid}")]
    public async Task<ActionResult<OrderDtoV1>> GetOrder(Guid id, CancellationToken ct)
    {
        // V1 response
        return Ok(new OrderDtoV1(id, "Pending", 100.00m));
    }
}

// V2 Controller — richer response
[ApiController]
[ApiVersion("2.0")]
[Route("v{version:apiVersion}/orders")]
public class OrdersV2Controller : ControllerBase
{
    [HttpGet("{id:guid}")]
    public async Task<ActionResult<OrderDtoV2>> GetOrder(Guid id, CancellationToken ct)
    {
        // V2 — includes timeline and items
        return Ok(new OrderDtoV2(id, "Pending", 100.00m, [], []));
    }
}

public record OrderDtoV1(Guid Id, string Status, decimal Total);
public record OrderDtoV2(Guid Id, string Status, decimal Total,
    IReadOnlyList<OrderItemDto> Items,
    IReadOnlyList<OrderTimelineEvent> Timeline);
```

---

## Step 1902: Distributed Tracing with OpenTelemetry

```bash
dotnet add package OpenTelemetry.Extensions.Hosting
dotnet add package OpenTelemetry.Instrumentation.AspNetCore
dotnet add package OpenTelemetry.Instrumentation.Http
dotnet add package OpenTelemetry.Exporter.Otlp
dotnet add package OpenTelemetry.Instrumentation.EntityFrameworkCore
```

```csharp
// Shared.Observability/TracingExtensions.cs
using OpenTelemetry.Resources;
using OpenTelemetry.Trace;

namespace Shared.Observability;

public static class TracingExtensions
{
    public static IServiceCollection AddDistributedTracing(
        this IServiceCollection services,
        IConfiguration config,
        string serviceName,
        Action<TracerProviderBuilder>? configure = null)
    {
        var version = typeof(TracingExtensions).Assembly.GetName().Version?.ToString() ?? "1.0.0";

        services.AddOpenTelemetry()
            .ConfigureResource(r => r
                .AddService(serviceName, serviceVersion: version)
                .AddAttributes(new Dictionary<string, object>
                {
                    ["deployment.environment"] = config["ASPNETCORE_ENVIRONMENT"] ?? "Production"
                }))
            .WithTracing(tracing =>
            {
                tracing
                    .AddAspNetCoreInstrumentation(opts =>
                    {
                        opts.RecordException = true;
                        opts.Filter = ctx => !ctx.Request.Path.StartsWithSegments("/health");
                    })
                    .AddHttpClientInstrumentation(opts =>
                    {
                        opts.RecordException = true;
                        opts.FilterHttpRequestMessage = req =>
                            req.RequestUri?.Host != "localhost" ||
                            req.RequestUri.Port != 9411; // exclude zipkin self-reporting
                    })
                    .AddEntityFrameworkCoreInstrumentation(opts =>
                    {
                        opts.SetDbStatementForText = true;
                    });

                configure?.Invoke(tracing);

                var otlpEndpoint = config["OTEL_EXPORTER_OTLP_ENDPOINT"];
                if (!string.IsNullOrEmpty(otlpEndpoint))
                {
                    tracing.AddOtlpExporter(opts =>
                    {
                        opts.Endpoint = new Uri(otlpEndpoint);
                    });
                }
                else
                {
                    tracing.AddConsoleExporter();
                }
            });

        return services;
    }
}
```

```csharp
// Custom activity source for domain events
public static class OrderActivitySource
{
    public static readonly ActivitySource Source = new("Orders.Application");

    public static Activity? StartPlaceOrder(Guid customerId)
    {
        var activity = Source.StartActivity("PlaceOrder");
        activity?.SetTag("order.customer_id", customerId.ToString());
        return activity;
    }

    public static Activity? StartProcessPayment(Guid orderId, decimal amount)
    {
        var activity = Source.StartActivity("ProcessPayment");
        activity?.SetTag("order.id", orderId.ToString());
        activity?.SetTag("payment.amount", amount.ToString("F2"));
        return activity;
    }
}

// Usage in handler
public async Task<PlaceOrderResult> Handle(PlaceOrderCommand cmd, CancellationToken ct)
{
    using var activity = OrderActivitySource.StartPlaceOrder(cmd.CustomerId);

    try
    {
        var order = Order.Create(cmd.CustomerId, cmd.ShippingAddress);
        // ...
        activity?.SetTag("order.id", order.Id.ToString());
        activity?.SetTag("order.item_count", cmd.Items.Count.ToString());
        return result;
    }
    catch (Exception ex)
    {
        activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
        activity?.RecordException(ex);
        throw;
    }
}
```

---

## Step 1903: Health Checks — Aggregated Status

```csharp
// Shared.Infrastructure/HealthChecks/ServiceHealthCheckExtensions.cs
using Microsoft.Extensions.Diagnostics.HealthChecks;

public static class ServiceHealthCheckExtensions
{
    public static IHealthChecksBuilder AddStandardHealthChecks(
        this IHealthChecksBuilder builder,
        IConfiguration config)
    {
        builder
            .AddNpgSql(
                config.GetConnectionString("Postgres")!,
                name: "postgres",
                tags: ["db", "critical"])
            .AddRedis(
                config.GetConnectionString("Redis")!,
                name: "redis",
                tags: ["cache"])
            .AddRabbitMQ(
                sp => sp.GetRequiredService<IConnection>(),
                name: "rabbitmq",
                tags: ["messaging", "critical"]);

        return builder;
    }
}
```

```csharp
// Program.cs
builder.Services.AddHealthChecks()
    .AddStandardHealthChecks(builder.Configuration)
    .AddCheck<OrderProcessingHealthCheck>("order-processing", tags: ["business"]);

// Expose endpoints
app.MapHealthChecks("/health", new HealthCheckOptions
{
    Predicate = _ => true, // all checks
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("critical")
});

app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false // liveness — just "is process alive"
});
```

---

## Step 1904: Rate Limiting at Gateway

```csharp
// Gateway — rate limiting per client
using Microsoft.AspNetCore.RateLimiting;
using System.Threading.RateLimiting;

builder.Services.AddRateLimiter(opts =>
{
    opts.RejectionStatusCode = StatusCodes.Status429TooManyRequests;

    // Global sliding window
    opts.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(ctx =>
    {
        var clientId = ctx.User?.FindFirst("sub")?.Value
            ?? ctx.Connection.RemoteIpAddress?.ToString()
            ?? "anonymous";

        return RateLimitPartition.GetSlidingWindowLimiter(clientId, _ =>
            new SlidingWindowRateLimiterOptions
            {
                Window = TimeSpan.FromMinutes(1),
                PermitLimit = 100,
                SegmentsPerWindow = 6,
                QueueProcessingOrder = QueueProcessingOrder.OldestFirst,
                QueueLimit = 5
            });
    });

    // Per-route fixed window
    opts.AddFixedWindowLimiter("heavy-operations", o =>
    {
        o.Window = TimeSpan.FromSeconds(10);
        o.PermitLimit = 5;
        o.QueueLimit = 0;
    });

    opts.OnRejected = async (ctx, ct) =>
    {
        ctx.HttpContext.Response.Headers.RetryAfter =
            ctx.Lease.TryGetMetadata(MetadataName.RetryAfter, out var retryAfter)
                ? ((int)retryAfter.TotalSeconds).ToString()
                : "60";

        await ctx.HttpContext.Response.WriteAsJsonAsync(new
        {
            error = "rate_limit_exceeded",
            message = "Too many requests. Please try again later."
        }, ct);
    };
});

app.UseRateLimiter();

// Apply to specific endpoints
app.MapPost("/api/orders", CreateOrder)
    .RequireRateLimiting("heavy-operations");
```

---

## Step 1905: Docker Compose for Local Development

```yaml
# docker-compose.yml
version: '3.8'

services:
  orders-api:
    build:
      context: ./src/Services/Orders
      dockerfile: Orders.Api/Dockerfile
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__Postgres=Host=postgres;Database=orders;Username=app;Password=secret
      - ConnectionStrings__Redis=redis:6379
      - ConnectionStrings__RabbitMQ=amqp://guest:guest@rabbitmq:5672
      - Services__Products__GrpcUrl=http://products-api:9090
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
    depends_on:
      postgres: { condition: service_healthy }
      rabbitmq: { condition: service_healthy }
      redis: { condition: service_started }
    ports:
      - "5001:8080"

  products-api:
    build:
      context: ./src/Services/Products
      dockerfile: Products.Api/Dockerfile
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__Postgres=Host=postgres;Database=products;Username=app;Password=secret
    depends_on:
      postgres: { condition: service_healthy }
    ports:
      - "5002:8080"
      - "9090:9090"  # gRPC

  gateway:
    build:
      context: ./src/ApiGateway
      dockerfile: Gateway.Api/Dockerfile
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
    depends_on:
      - orders-api
      - products-api
    ports:
      - "5000:8080"

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./scripts/init-databases.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      timeout: 5s
      retries: 5

  rabbitmq:
    image: rabbitmq:3.13-management-alpine
    environment:
      RABBITMQ_DEFAULT_USER: guest
      RABBITMQ_DEFAULT_PASS: guest
    volumes:
      - rabbitmq-data:/var/lib/rabbitmq
    ports:
      - "5672:5672"
      - "15672:15672"  # Management UI
    healthcheck:
      test: rabbitmq-diagnostics -q ping
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    volumes:
      - redis-data:/data

  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    volumes:
      - ./otel-config.yaml:/etc/otelcol-contrib/config.yaml
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP

  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"  # UI
      - "14250:14250"  # gRPC

volumes:
  postgres-data:
  rabbitmq-data:
  redis-data:
```

```sql
-- scripts/init-databases.sql
CREATE DATABASE orders;
CREATE DATABASE products;
CREATE DATABASE customers;
GRANT ALL PRIVILEGES ON DATABASE orders TO app;
GRANT ALL PRIVILEGES ON DATABASE products TO app;
GRANT ALL PRIVILEGES ON DATABASE customers TO app;
```

---

## Step 1906: Service Mesh Concepts (Envoy/Istio)

```yaml
# Istio VirtualService — traffic management
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: orders-api
spec:
  hosts:
    - orders-api
  http:
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: orders-api
            subset: v2
          weight: 100
    - route:
        - destination:
            host: orders-api
            subset: v1
          weight: 90
        - destination:
            host: orders-api
            subset: v2
          weight: 10
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: orders-api
spec:
  host: orders-api
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: UPGRADE
        idleTimeout: 10s
    outlierDetection:
      consecutiveGatewayErrors: 5
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
```

---

## Step 1907: Circuit Breaker Testing

```csharp
// Tests/Integration/ProductServiceClientTests.cs
public class ProductServiceClientTests : IClassFixture<WireMockServerFixture>
{
    private readonly WireMockServerFixture _wiremock;

    [Fact]
    public async Task CircuitBreaker_OpensAfterFailures()
    {
        // Simulate product service returning 500 errors
        _wiremock.Server.Given(
            Request.Create().WithPath("/products/*").UsingGet()
        ).RespondWith(
            Response.Create().WithStatusCode(500)
        );

        var client = CreateClientWithCircuitBreaker();

        // First 3 attempts should fail with HttpRequestException
        for (var i = 0; i < 3; i++)
        {
            await Assert.ThrowsAsync<HttpRequestException>(
                () => client.GetProductAsync(Guid.NewGuid()));
        }

        // Circuit should now be open — next call fails fast with BrokenCircuitException
        await Assert.ThrowsAsync<BrokenCircuitException>(
            () => client.GetProductAsync(Guid.NewGuid()));
    }

    [Fact]
    public async Task Retry_RetriesOnTransientFailure()
    {
        var callCount = 0;

        _wiremock.Server.Given(
            Request.Create().WithPath("/products/*").UsingGet()
        ).RespondWith(
            Response.Create()
                .WithCallback(_ =>
                {
                    callCount++;
                    return callCount < 3
                        ? ResponseMessage.Create().WithStatusCode(503)
                        : ResponseMessage.Create().WithStatusCode(200)
                            .WithBodyAsJson(new { id = Guid.NewGuid(), name = "Widget" });
                })
        );

        var client = CreateClientWithRetry(maxRetries: 3);
        var product = await client.GetProductAsync(Guid.NewGuid());

        product.Should().NotBeNull();
        callCount.Should().Be(3);
    }
}
```

---

## Step 1908: Complete Microservice Template

```csharp
// Program.cs — per-service template
using Shared.Infrastructure.Http;
using Shared.Observability;
using Orders.Infrastructure.Messaging;

var builder = WebApplication.CreateBuilder(args);

// Core services
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// API versioning
builder.Services.AddApiVersioning(opts =>
{
    opts.DefaultApiVersion = new ApiVersion(1, 0);
    opts.AssumeDefaultVersionWhenUnspecified = true;
    opts.ReportApiVersions = true;
}).AddApiExplorer(opts => opts.GroupNameFormat = "'v'VVV");

// Database
builder.Services.AddDbContext<OrdersDbContext>(opts =>
    opts.UseNpgsql(builder.Configuration.GetConnectionString("Postgres")));

// MassTransit / RabbitMQ
builder.Services.AddOrdersMessaging(builder.Configuration);

// HTTP clients
builder.Services.AddTransient<CorrelationIdHandler>();
builder.Services.AddTransient<ServiceAuthHandler>();
builder.Services.AddServiceClient<IProductServiceClient, ProductServiceClient>(
    builder.Configuration["Services:Products:HttpUrl"]!);

// gRPC clients
builder.Services.AddGrpcClient<InventoryService.InventoryServiceClient>(o =>
    o.Address = new Uri(builder.Configuration["Services:Products:GrpcUrl"]!))
    .AddStandardResilienceHandler();

// Observability
builder.Services.AddDistributedTracing(builder.Configuration, "orders-api",
    tracing => tracing.AddSource("Orders.Application"));
builder.Logging.AddOpenTelemetry(otel =>
    otel.AddOtlpExporter());

// Health checks
builder.Services.AddHealthChecks()
    .AddStandardHealthChecks(builder.Configuration);

// Auth
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer();
builder.Services.AddAuthorization();

// Rate limiting
builder.Services.AddRateLimiter(opts =>
    opts.AddFixedWindowLimiter("default", o =>
    {
        o.Window = TimeSpan.FromMinutes(1);
        o.PermitLimit = 1000;
    }));

// Application services
builder.Services.AddScoped<IOrderRepository, OrderRepository>();
builder.Services.AddScoped<IUnitOfWork, UnitOfWork>();
builder.Services.AddMediatR(cfg =>
    cfg.RegisterServicesFromAssembly(typeof(PlaceOrderCommand).Assembly));

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseRateLimiter();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
app.MapHealthChecks("/health");
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = c => c.Tags.Contains("critical")
});

// Apply migrations on startup (development only)
if (app.Environment.IsDevelopment())
{
    using var scope = app.Services.CreateScope();
    var db = scope.ServiceProvider.GetRequiredService<OrdersDbContext>();
    await db.Database.MigrateAsync();
}

app.Run();
```

---

## Summary

| Pattern | Tool / Abstraction | Purpose |
|---------|-------------------|---------|
| API Gateway | YARP `ReverseProxy` | Route, auth, rate limit at edge |
| Sync HTTP | `IHttpClientFactory` + Polly | Typed HTTP with resilience |
| Async events | MassTransit + RabbitMQ | Decoupled event-driven communication |
| Saga | `MassTransitStateMachine` | Distributed workflow coordination |
| Service discovery | Consul agent | Dynamic endpoint resolution |
| Internal gRPC | `Grpc.AspNetCore` | High-perf typed inter-service calls |
| API versioning | `Asp.Versioning` | URL/header/query version negotiation |
| Tracing | OpenTelemetry OTLP | Cross-service distributed traces |
| Rate limiting | `PartitionedRateLimiter` | Client-aware throttling |
| Service mesh | Istio VirtualService | Traffic splitting, mTLS, observability |

**Next**: Part 80 — Event Sourcing & CQRS with EventStoreDB
