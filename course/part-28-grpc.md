# Part 28: gRPC - High-Performance RPC

## Steps 751-780: การสร้าง API ด้วย gRPC

---

## Step 751: gRPC คืออะไร

```bash
# gRPC = Google Remote Procedure Call
# ใช้ Protocol Buffers (protobuf) เป็น serialization format
# HTTP/2 - multiplexed, binary, header compression
# ดีกว่า REST สำหรับ: microservices, low latency, streaming

# Service types:
# 1. Unary - Client sends 1 request, Server sends 1 response
# 2. Server Streaming - Client sends 1 request, Server streams responses
# 3. Client Streaming - Client streams requests, Server sends 1 response
# 4. Bidirectional Streaming - Both stream simultaneously

# สร้าง gRPC server project
dotnet new grpc -n GrpcServer
cd GrpcServer

# NuGet packages (already included in template)
# Grpc.AspNetCore
# Google.Protobuf
# Grpc.Tools
```

---

## Step 752: Protocol Buffers (.proto files)

```protobuf
// Protos/product.proto
syntax = "proto3";

option csharp_namespace = "GrpcServer";

package product;

// Import common types
import "google/protobuf/timestamp.proto";
import "google/protobuf/empty.proto";
import "google/protobuf/wrappers.proto";

// Service definition
service ProductService {
    // Unary RPC
    rpc GetProduct(GetProductRequest) returns (ProductResponse);
    rpc CreateProduct(CreateProductRequest) returns (ProductResponse);
    rpc UpdateProduct(UpdateProductRequest) returns (ProductResponse);
    rpc DeleteProduct(DeleteProductRequest) returns (google.protobuf.Empty);
    
    // Server streaming - get products as stream
    rpc ListProducts(ListProductsRequest) returns (stream ProductResponse);
    
    // Client streaming - bulk create
    rpc BulkCreateProducts(stream CreateProductRequest) returns (BulkCreateResponse);
    
    // Bidirectional streaming - real-time price updates
    rpc SubscribePriceUpdates(stream PriceSubscribeRequest) returns (stream PriceUpdateResponse);
}

// Messages
message GetProductRequest {
    int32 id = 1;
}

message ListProductsRequest {
    string search = 1;        // field 1
    int32 page = 2;           // field 2
    int32 page_size = 3;      // field 3
    string category = 4;      // field 4
    bool include_inactive = 5;// field 5
}

message CreateProductRequest {
    string name = 1;
    string description = 2;
    double price = 3;
    int32 stock = 4;
    int32 category_id = 5;
    repeated string tags = 6; // repeated = array
}

message UpdateProductRequest {
    int32 id = 1;
    google.protobuf.StringValue name = 2;    // optional string
    google.protobuf.DoubleValue price = 3;   // optional double
    google.protobuf.Int32Value stock = 4;    // optional int
}

message DeleteProductRequest {
    int32 id = 1;
}

message ProductResponse {
    int32 id = 1;
    string name = 2;
    string description = 3;
    double price = 4;
    int32 stock = 5;
    string category_name = 6;
    repeated string tags = 7;
    bool is_active = 8;
    google.protobuf.Timestamp created_at = 9;
    google.protobuf.Timestamp updated_at = 10;
}

message BulkCreateResponse {
    int32 created_count = 1;
    int32 failed_count = 2;
    repeated string errors = 3;
}

message PriceSubscribeRequest {
    repeated int32 product_ids = 1;
}

message PriceUpdateResponse {
    int32 product_id = 1;
    double old_price = 2;
    double new_price = 3;
    google.protobuf.Timestamp updated_at = 4;
}

// Enums
enum ProductStatus {
    PRODUCT_STATUS_UNSPECIFIED = 0;
    PRODUCT_STATUS_ACTIVE = 1;
    PRODUCT_STATUS_INACTIVE = 2;
    PRODUCT_STATUS_DISCONTINUED = 3;
}

// Nested message
message Address {
    string street = 1;
    string city = 2;
    string country = 3;
    string postal_code = 4;
}

message Customer {
    int32 id = 1;
    string name = 2;
    string email = 3;
    Address shipping_address = 4;  // nested message
    oneof contact {                // oneof = union type
        string phone = 5;
        string line_id = 6;
    }
}
```

```xml
<!-- GrpcServer.csproj - Configure .proto files -->
<ItemGroup>
    <Protobuf Include="Protos\product.proto" GrpcServices="Server" />
    <Protobuf Include="Protos\customer.proto" GrpcServices="Both" />
</ItemGroup>

<!-- GrpcServices options:
     Server - generate server code only
     Client - generate client code only
     Both   - generate both
     None   - only C# classes, no gRPC
-->
```

---

## Step 753: gRPC Service Implementation

```csharp
// Services/ProductServiceImpl.cs
using Grpc.Core;
using Google.Protobuf.WellKnownTypes;

namespace GrpcServer.Services;

public class ProductServiceImpl : ProductService.ProductServiceBase
{
    private readonly IProductRepository _repository;
    private readonly ILogger<ProductServiceImpl> _logger;

    public ProductServiceImpl(
        IProductRepository repository,
        ILogger<ProductServiceImpl> logger)
    {
        _repository = repository;
        _logger = logger;
    }

    // Unary RPC
    public override async Task<ProductResponse> GetProduct(
        GetProductRequest request,
        ServerCallContext context)
    {
        _logger.LogInformation("GetProduct called for Id: {Id}", request.Id);

        var product = await _repository.GetByIdAsync(request.Id);
        
        if (product is null)
        {
            throw new RpcException(new Status(
                StatusCode.NotFound, 
                $"Product with ID {request.Id} not found"));
        }

        return MapToResponse(product);
    }

    // Unary - Create
    public override async Task<ProductResponse> CreateProduct(
        CreateProductRequest request,
        ServerCallContext context)
    {
        // Validation
        if (string.IsNullOrWhiteSpace(request.Name))
        {
            throw new RpcException(new Status(
                StatusCode.InvalidArgument,
                "Product name is required"));
        }

        if (request.Price <= 0)
        {
            throw new RpcException(new Status(
                StatusCode.InvalidArgument,
                "Price must be greater than 0"));
        }

        // Check cancellation
        context.CancellationToken.ThrowIfCancellationRequested();

        var product = new Domain.Product
        {
            Name = request.Name,
            Description = request.Description,
            Price = (decimal)request.Price,
            Stock = request.Stock,
            CategoryId = request.CategoryId,
            Tags = request.Tags.ToList()
        };

        var created = await _repository.CreateAsync(product);
        return MapToResponse(created);
    }

    // Server streaming RPC
    public override async Task ListProducts(
        ListProductsRequest request,
        IServerStreamWriter<ProductResponse> responseStream,
        ServerCallContext context)
    {
        _logger.LogInformation("ListProducts streaming started");

        var products = _repository.GetAllAsyncEnumerable(
            request.Search,
            request.Category,
            request.IncludeInactive);

        await foreach (var product in products)
        {
            if (context.CancellationToken.IsCancellationRequested)
            {
                _logger.LogInformation("Client cancelled the stream");
                break;
            }

            await responseStream.WriteAsync(MapToResponse(product));
            await Task.Delay(10, context.CancellationToken); // Simulate processing
        }

        _logger.LogInformation("ListProducts streaming completed");
    }

    // Client streaming RPC
    public override async Task<BulkCreateResponse> BulkCreateProducts(
        IAsyncStreamReader<CreateProductRequest> requestStream,
        ServerCallContext context)
    {
        var created = 0;
        var failed = 0;
        var errors = new List<string>();

        await foreach (var request in requestStream.ReadAllAsync(context.CancellationToken))
        {
            try
            {
                var product = new Domain.Product
                {
                    Name = request.Name,
                    Price = (decimal)request.Price,
                    Stock = request.Stock
                };
                await _repository.CreateAsync(product);
                created++;
            }
            catch (Exception ex)
            {
                failed++;
                errors.Add($"Failed to create '{request.Name}': {ex.Message}");
            }
        }

        return new BulkCreateResponse
        {
            CreatedCount = created,
            FailedCount = failed,
            Errors = { errors }
        };
    }

    // Bidirectional streaming
    public override async Task SubscribePriceUpdates(
        IAsyncStreamReader<PriceSubscribeRequest> requestStream,
        IServerStreamWriter<PriceUpdateResponse> responseStream,
        ServerCallContext context)
    {
        var subscribedIds = new HashSet<int>();
        var ct = context.CancellationToken;

        // Read subscription requests in background
        var readTask = Task.Run(async () =>
        {
            await foreach (var req in requestStream.ReadAllAsync(ct))
            {
                foreach (var id in req.ProductIds)
                    subscribedIds.Add(id);
            }
        }, ct);

        // Send price updates
        while (!ct.IsCancellationRequested)
        {
            foreach (var id in subscribedIds.ToList())
            {
                var update = await _repository.GetPriceUpdateAsync(id);
                if (update is not null)
                {
                    await responseStream.WriteAsync(new PriceUpdateResponse
                    {
                        ProductId = id,
                        OldPrice = (double)update.OldPrice,
                        NewPrice = (double)update.NewPrice,
                        UpdatedAt = Timestamp.FromDateTime(update.UpdatedAt)
                    });
                }
            }
            await Task.Delay(1000, ct);
        }

        await readTask;
    }

    private static ProductResponse MapToResponse(Domain.Product product)
    {
        return new ProductResponse
        {
            Id = product.Id,
            Name = product.Name,
            Description = product.Description,
            Price = (double)product.Price,
            Stock = product.Stock,
            CategoryName = product.Category?.Name ?? "",
            IsActive = product.IsActive,
            Tags = { product.Tags },
            CreatedAt = Timestamp.FromDateTime(product.CreatedAt.ToUniversalTime()),
            UpdatedAt = Timestamp.FromDateTime(product.UpdatedAt.ToUniversalTime())
        };
    }
}
```

---

## Step 754: gRPC Server Setup

```csharp
// Program.cs - gRPC Server
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddGrpc(options =>
{
    options.EnableDetailedErrors = builder.Environment.IsDevelopment();
    options.MaxReceiveMessageSize = 16 * 1024 * 1024; // 16MB
    options.MaxSendMessageSize = 16 * 1024 * 1024;
    options.Interceptors.Add<LoggingInterceptor>();
    options.Interceptors.Add<ExceptionInterceptor>();
});

builder.Services.AddGrpcReflection(); // For tools like Postman, grpcurl

builder.Services.AddScoped<IProductRepository, ProductRepository>();
builder.Services.AddSingleton<LoggingInterceptor>();
builder.Services.AddSingleton<ExceptionInterceptor>();

var app = builder.Build();

app.MapGrpcService<ProductServiceImpl>();
app.MapGrpcService<CustomerServiceImpl>();

if (app.Environment.IsDevelopment())
{
    app.MapGrpcReflectionService(); // gRPC reflection for development
}

app.MapGet("/", () => "gRPC server is running. Use a gRPC client to communicate.");

app.Run();

// Kestrel configuration for HTTP/2
// In appsettings.json:
// "Kestrel": {
//     "EndpointDefaults": {
//         "Protocols": "Http2"
//     }
// }
```

---

## Step 755: gRPC Interceptors

```csharp
// Interceptors/LoggingInterceptor.cs
using Grpc.Core;
using Grpc.Core.Interceptors;

namespace GrpcServer.Interceptors;

public class LoggingInterceptor : Interceptor
{
    private readonly ILogger<LoggingInterceptor> _logger;

    public LoggingInterceptor(ILogger<LoggingInterceptor> logger)
    {
        _logger = logger;
    }

    // Unary
    public override async Task<TResponse> UnaryServerHandler<TRequest, TResponse>(
        TRequest request,
        ServerCallContext context,
        UnaryServerMethod<TRequest, TResponse> continuation)
    {
        var method = context.Method;
        var peer = context.Peer;
        
        _logger.LogInformation("gRPC call: {Method} from {Peer}", method, peer);
        var stopwatch = System.Diagnostics.Stopwatch.StartNew();

        try
        {
            var response = await continuation(request, context);
            _logger.LogInformation("gRPC {Method} completed in {Ms}ms", method, stopwatch.ElapsedMilliseconds);
            return response;
        }
        catch (RpcException)
        {
            throw;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "gRPC {Method} failed", method);
            throw;
        }
    }

    // Server streaming
    public override async Task ServerStreamingServerHandler<TRequest, TResponse>(
        TRequest request,
        IServerStreamWriter<TResponse> responseStream,
        ServerCallContext context,
        ServerStreamingServerMethod<TRequest, TResponse> continuation)
    {
        _logger.LogInformation("gRPC server stream: {Method}", context.Method);
        await continuation(request, responseStream, context);
        _logger.LogInformation("gRPC server stream completed: {Method}", context.Method);
    }
}

// Interceptors/ExceptionInterceptor.cs
public class ExceptionInterceptor : Interceptor
{
    private readonly ILogger<ExceptionInterceptor> _logger;

    public ExceptionInterceptor(ILogger<ExceptionInterceptor> logger)
    {
        _logger = logger;
    }

    public override async Task<TResponse> UnaryServerHandler<TRequest, TResponse>(
        TRequest request,
        ServerCallContext context,
        UnaryServerMethod<TRequest, TResponse> continuation)
    {
        try
        {
            return await continuation(request, context);
        }
        catch (RpcException)
        {
            throw; // Re-throw gRPC exceptions as-is
        }
        catch (NotFoundException ex)
        {
            throw new RpcException(new Status(StatusCode.NotFound, ex.Message));
        }
        catch (ValidationException ex)
        {
            throw new RpcException(new Status(StatusCode.InvalidArgument, ex.Message));
        }
        catch (UnauthorizedException ex)
        {
            throw new RpcException(new Status(StatusCode.Unauthenticated, ex.Message));
        }
        catch (ForbiddenException ex)
        {
            throw new RpcException(new Status(StatusCode.PermissionDenied, ex.Message));
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unhandled gRPC exception");
            throw new RpcException(new Status(StatusCode.Internal, "Internal server error"));
        }
    }
}

// Interceptors/AuthInterceptor.cs - Authentication
public class AuthInterceptor : Interceptor
{
    private readonly IJwtService _jwtService;

    public AuthInterceptor(IJwtService jwtService)
    {
        _jwtService = jwtService;
    }

    public override async Task<TResponse> UnaryServerHandler<TRequest, TResponse>(
        TRequest request,
        ServerCallContext context,
        UnaryServerMethod<TRequest, TResponse> continuation)
    {
        var token = context.RequestHeaders
            .FirstOrDefault(h => h.Key == "authorization")?.Value;

        if (token is not null && token.StartsWith("Bearer "))
        {
            var jwt = token[7..];
            var principal = _jwtService.ValidateToken(jwt);
            if (principal is not null)
            {
                context.UserState["user"] = principal;
            }
        }

        return await continuation(request, context);
    }
}
```

---

## Step 756: gRPC Client

```csharp
// Client/ProductClient.cs
using Grpc.Net.Client;
using Grpc.Core;

namespace GrpcClient;

public class ProductGrpcClient : IAsyncDisposable
{
    private readonly GrpcChannel _channel;
    private readonly ProductService.ProductServiceClient _client;

    public ProductGrpcClient(string serverAddress)
    {
        _channel = GrpcChannel.ForAddress(serverAddress, new GrpcChannelOptions
        {
            MaxReceiveMessageSize = 16 * 1024 * 1024,
            MaxSendMessageSize = 16 * 1024 * 1024,
            ServiceConfig = new ServiceConfig
            {
                MethodConfigs =
                {
                    new MethodConfig
                    {
                        Names = { MethodName.Default },
                        RetryPolicy = new RetryPolicy
                        {
                            MaxAttempts = 3,
                            InitialBackoff = TimeSpan.FromSeconds(1),
                            MaxBackoff = TimeSpan.FromSeconds(5),
                            BackoffMultiplier = 1.5,
                            RetryableStatusCodes = { StatusCode.Unavailable }
                        }
                    }
                }
            }
        });

        _client = new ProductService.ProductServiceClient(_channel);
    }

    // Unary call
    public async Task<ProductResponse?> GetProductAsync(int id, CancellationToken ct = default)
    {
        try
        {
            var headers = new Metadata
            {
                { "authorization", $"Bearer {GetToken()}" },
                { "x-request-id", Guid.NewGuid().ToString() }
            };

            var response = await _client.GetProductAsync(
                new GetProductRequest { Id = id },
                headers: headers,
                deadline: DateTime.UtcNow.AddSeconds(30),
                cancellationToken: ct);

            return response;
        }
        catch (RpcException ex) when (ex.StatusCode == StatusCode.NotFound)
        {
            return null;
        }
        catch (RpcException ex)
        {
            throw new Exception($"gRPC error: {ex.Status.Detail}", ex);
        }
    }

    // Server streaming
    public async IAsyncEnumerable<ProductResponse> ListProductsAsync(
        string? search = null,
        [System.Runtime.CompilerServices.EnumeratorCancellation] CancellationToken ct = default)
    {
        var request = new ListProductsRequest { Search = search ?? "" };
        
        using var call = _client.ListProducts(request, cancellationToken: ct);
        
        await foreach (var product in call.ResponseStream.ReadAllAsync(ct))
        {
            yield return product;
        }
    }

    // Client streaming
    public async Task<BulkCreateResponse> BulkCreateAsync(
        IEnumerable<CreateProductRequest> products,
        CancellationToken ct = default)
    {
        using var call = _client.BulkCreateProducts(cancellationToken: ct);
        
        foreach (var product in products)
        {
            await call.RequestStream.WriteAsync(product, ct);
        }
        
        await call.RequestStream.CompleteAsync();
        return await call.ResponseAsync;
    }

    // Bidirectional streaming
    public async Task SubscribePriceUpdatesAsync(
        IEnumerable<int> initialProductIds,
        Func<PriceUpdateResponse, Task> onUpdate,
        CancellationToken ct = default)
    {
        using var call = _client.SubscribePriceUpdates(cancellationToken: ct);
        
        // Send initial subscription
        await call.RequestStream.WriteAsync(new PriceSubscribeRequest
        {
            ProductIds = { initialProductIds }
        });

        // Receive updates
        await foreach (var update in call.ResponseStream.ReadAllAsync(ct))
        {
            await onUpdate(update);
        }
    }

    private static string GetToken() => "your-jwt-token";

    public async ValueTask DisposeAsync()
    {
        await _channel.ShutdownAsync();
    }
}
```

---

## Step 757: gRPC with ASP.NET Core DI

```csharp
// Program.cs - Register gRPC Client
builder.Services.AddGrpcClient<ProductService.ProductServiceClient>(options =>
{
    options.Address = new Uri(builder.Configuration["GrpcServices:ProductService"]!);
})
.ConfigureChannel(channelOptions =>
{
    channelOptions.ServiceConfig = new ServiceConfig
    {
        MethodConfigs =
        {
            new MethodConfig
            {
                Names = { MethodName.Default },
                RetryPolicy = new RetryPolicy
                {
                    MaxAttempts = 3,
                    InitialBackoff = TimeSpan.FromMilliseconds(100),
                    MaxBackoff = TimeSpan.FromSeconds(5),
                    BackoffMultiplier = 2,
                    RetryableStatusCodes = { StatusCode.Unavailable, StatusCode.Internal }
                }
            }
        }
    };
})
.AddCallCredentials(async (context, metadata) =>
{
    // Add auth token to all calls
    var tokenService = context.ServiceProvider.GetRequiredService<ITokenService>();
    var token = await tokenService.GetTokenAsync();
    metadata.Add("Authorization", $"Bearer {token}");
})
.AddInterceptor<ClientLoggingInterceptor>()
.EnableCallContextPropagation(); // Propagate deadline/cancellation
```

---

## Step 758: gRPC-Web (Browser Support)

```csharp
// Program.cs - gRPC-Web for browser clients
builder.Services.AddGrpc();
builder.Services.AddGrpcWeb(options => options.GrpcWebEnabled = true);

app.UseGrpcWeb(new GrpcWebOptions { DefaultEnabled = true });

app.MapGrpcService<ProductServiceImpl>()
    .EnableGrpcWeb()
    .RequireCors("allow-all");
```

```javascript
// Browser JavaScript client with gRPC-Web
import { GrpcWebFetchTransport } from "@protobuf-ts/grpcweb-transport";
import { ProductServiceClient } from "./generated/product.client";
import { ListProductsRequest } from "./generated/product";

const transport = new GrpcWebFetchTransport({
    baseUrl: "https://localhost:7001"
});

const client = new ProductServiceClient(transport);

// Unary call
const { response } = await client.getProduct({ id: 1 });
console.log(response.name);

// Server streaming
const stream = client.listProducts(
    ListProductsRequest.create({ search: "laptop" })
);

for await (const product of stream.responses) {
    console.log(product.name, product.price);
}
```

---

## Step 759: Error Handling และ Metadata

```csharp
// Error details with rich metadata
public override async Task<ProductResponse> GetProduct(
    GetProductRequest request,
    ServerCallContext context)
{
    var product = await _repository.GetByIdAsync(request.Id);
    
    if (product is null)
    {
        var metadata = new Metadata
        {
            { "product-id", request.Id.ToString() },
            { "timestamp", DateTime.UtcNow.ToString("O") }
        };
        
        throw new RpcException(
            new Status(StatusCode.NotFound, $"Product {request.Id} not found"),
            metadata);
    }

    // Add response metadata
    await context.WriteResponseHeadersAsync(new Metadata
    {
        { "x-server-id", Environment.MachineName },
        { "x-version", "1.0" }
    });

    context.ResponseTrailers.Add("x-operation-time", "5ms");

    return MapToResponse(product);
}

// Reading metadata on client
public async Task<ProductResponse?> GetProductWithMetadataAsync(int id)
{
    try
    {
        var call = _client.GetProductAsync(new GetProductRequest { Id = id });
        
        // Read response headers before getting the response
        var headers = await call.ResponseHeadersAsync;
        var serverId = headers.GetValue("x-server-id");
        Console.WriteLine($"Served by: {serverId}");
        
        var response = await call.ResponseAsync;
        
        // Read trailers after the response
        var trailers = call.GetTrailers();
        Console.WriteLine($"Operation time: {trailers.GetValue("x-operation-time")}");
        
        return response;
    }
    catch (RpcException ex)
    {
        // Access error metadata
        var productId = ex.Trailers.GetValue("product-id");
        Console.WriteLine($"Not found: product ID {productId}");
        return null;
    }
}
```

---

## Step 760: gRPC Health Checks

```csharp
// Program.cs
using Grpc.HealthCheck;

builder.Services.AddGrpcHealthChecks()
    .AddCheck("product-db", () =>
    {
        // Check database connection
        return HealthCheckResult.Healthy();
    });

app.MapGrpcHealthChecksService();

// Client health check
var healthClient = new Health.HealthClient(channel);
var response = await healthClient.CheckAsync(new HealthCheckRequest 
{ 
    Service = "GrpcServer.ProductService" 
});

Console.WriteLine($"Status: {response.Status}");
// SERVING | NOT_SERVING | UNKNOWN
```

---

## Step 761: gRPC Performance Optimization

```csharp
// Use MessagePack instead of JSON for internal services
// dotnet add package Grpc.Net.Client
// dotnet add package MessagePack

// Connection pooling and reuse
public class GrpcClientFactory
{
    private static GrpcChannel? _channel;
    private static readonly object _lock = new();

    public static ProductService.ProductServiceClient CreateClient(string address)
    {
        if (_channel is null)
        {
            lock (_lock)
            {
                _channel ??= GrpcChannel.ForAddress(address, new GrpcChannelOptions
                {
                    // Reuse HttpClient for connection pooling
                    HttpClient = new HttpClient(new SocketsHttpHandler
                    {
                        EnableMultipleHttp2Connections = true,
                        PooledConnectionIdleTimeout = TimeSpan.FromMinutes(5),
                        KeepAlivePingDelay = TimeSpan.FromSeconds(60),
                        KeepAlivePingTimeout = TimeSpan.FromSeconds(30)
                    })
                });
            }
        }
        return new ProductService.ProductServiceClient(_channel);
    }
}

// Deadlines - always set deadlines!
var deadline = DateTime.UtcNow.AddSeconds(10);
var response = await _client.GetProductAsync(
    request,
    deadline: deadline);

// Cancellation
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(10));
var response = await _client.GetProductAsync(
    request,
    cancellationToken: cts.Token);
```

---

## Step 762: Transcoding - REST to gRPC

```protobuf
// Protos/product_api.proto - HTTP/REST to gRPC transcoding
syntax = "proto3";

import "google/api/annotations.proto";

service ProductApi {
    rpc GetProduct(GetProductRequest) returns (ProductResponse) {
        option (google.api.http) = {
            get: "/v1/products/{id}"
        };
    }
    
    rpc CreateProduct(CreateProductRequest) returns (ProductResponse) {
        option (google.api.http) = {
            post: "/v1/products"
            body: "*"
        };
    }
    
    rpc ListProducts(ListProductsRequest) returns (ListProductsResponse) {
        option (google.api.http) = {
            get: "/v1/products"
        };
    }
}
```

```csharp
// Program.cs - Enable transcoding
builder.Services.AddGrpc().AddJsonTranscoding();
builder.Services.AddGrpcSwagger();

// Now REST clients can call:
// GET /v1/products/1
// POST /v1/products
// GET /v1/products?search=laptop
// These are automatically converted to gRPC calls
```

---

## Step 763: gRPC vs REST Comparison

```csharp
// Performance comparison
public class BenchmarkComparison
{
    // REST equivalent
    [HttpGet("products/{id}")]
    public async Task<IActionResult> GetProduct(int id)
    {
        var product = await _service.GetAsync(id);
        return Ok(product); // JSON serialization: ~500 bytes
    }

    // gRPC equivalent (protobuf): ~150 bytes (70% smaller)
    // + binary format = faster parsing
    // + HTTP/2 = multiplexed, header compression
    // + strong typing = no JSON schema negotiation
}

// When to use gRPC:
// ✅ Microservices internal communication
// ✅ Performance-critical systems
// ✅ Streaming data (logs, metrics, events)
// ✅ Polyglot environments (generated clients for many languages)
// ✅ Strongly typed contracts

// When to use REST:
// ✅ Public APIs (browser-friendly)
// ✅ Simple CRUD operations
// ✅ Human-readable debugging
// ✅ Simple client requirements
// ✅ Caching (REST has better HTTP cache support)
```

---

## Step 764: Microservices Communication Pattern

```csharp
// Gateway API receives REST, calls gRPC services internally
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly ProductService.ProductServiceClient _productGrpc;
    private readonly InventoryService.InventoryServiceClient _inventoryGrpc;
    private readonly PricingService.PricingServiceClient _pricingGrpc;

    public ProductsController(
        ProductService.ProductServiceClient productGrpc,
        InventoryService.InventoryServiceClient inventoryGrpc,
        PricingService.PricingServiceClient pricingGrpc)
    {
        _productGrpc = productGrpc;
        _inventoryGrpc = inventoryGrpc;
        _pricingGrpc = pricingGrpc;
    }

    [HttpGet("{id}")]
    public async Task<IActionResult> GetProduct(int id, CancellationToken ct)
    {
        // Fan-out calls to multiple gRPC services
        var productTask = _productGrpc.GetProductAsync(
            new GetProductRequest { Id = id }, cancellationToken: ct).ResponseAsync;
        var inventoryTask = _inventoryGrpc.GetStockAsync(
            new GetStockRequest { ProductId = id }, cancellationToken: ct).ResponseAsync;
        var pricingTask = _pricingGrpc.GetPriceAsync(
            new GetPriceRequest { ProductId = id }, cancellationToken: ct).ResponseAsync;

        await Task.WhenAll(productTask, inventoryTask, pricingTask);

        return Ok(new ProductDetailDto
        {
            Id = id,
            Name = productTask.Result.Name,
            Price = pricingTask.Result.CurrentPrice,
            Stock = inventoryTask.Result.Available,
            Description = productTask.Result.Description
        });
    }
}
```

---

## Step 765: gRPC with Polly (Resilience)

```csharp
// Using Polly for retry and circuit breaker with gRPC
builder.Services.AddGrpcClient<ProductService.ProductServiceClient>(options =>
{
    options.Address = new Uri("https://product-service:5001");
})
.AddTransientHttpErrorPolicy(policy =>
    policy.WaitAndRetryAsync(3, retryAttempt =>
        TimeSpan.FromSeconds(Math.Pow(2, retryAttempt))))
.AddTransientHttpErrorPolicy(policy =>
    policy.CircuitBreakerAsync(5, TimeSpan.FromSeconds(30)));

// Custom gRPC resilience policy
public class GrpcResiliencePolicy
{
    private readonly AsyncPolicy _policy;

    public GrpcResiliencePolicy()
    {
        var retry = Policy<object>
            .Handle<RpcException>(ex => 
                ex.StatusCode == StatusCode.Unavailable ||
                ex.StatusCode == StatusCode.Internal)
            .WaitAndRetryAsync(3, 
                retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)),
                (ex, delay, attempt, _) =>
                {
                    Console.WriteLine($"Retry {attempt} after {delay}: {ex.Exception.Message}");
                });

        var circuitBreaker = Policy<object>
            .Handle<RpcException>(ex => ex.StatusCode == StatusCode.Unavailable)
            .CircuitBreakerAsync(5, TimeSpan.FromMinutes(1));

        _policy = Policy.WrapAsync(retry, circuitBreaker);
    }

    public async Task<T> ExecuteAsync<T>(Func<Task<T>> action)
        => (T)await _policy.ExecuteAsync(async () => (object)(await action())!);
}
```

---

## Step 766: Reflection và API Discovery

```csharp
// gRPC Reflection for development tools
// Allows grpcurl, Postman, and gRPC UI to discover services

// Enable in Development
if (app.Environment.IsDevelopment())
{
    app.MapGrpcReflectionService();
}

// Usage with grpcurl CLI:
// grpcurl -plaintext localhost:5001 list
// grpcurl -plaintext localhost:5001 describe product.ProductService
// grpcurl -plaintext -d '{"id": 1}' localhost:5001 product.ProductService/GetProduct

// Usage with gRPC UI (web-based Postman for gRPC)
// docker run -p 8080:8080 -e GRPCUI_SERVER=localhost:5001 fullstorydev/grpcui
```

---

## Step 767: Unit Testing gRPC

```csharp
// Tests/ProductServiceTests.cs
using Grpc.Core;
using Grpc.Core.Testing;
using Moq;
using Xunit;

public class ProductServiceTests
{
    private readonly Mock<IProductRepository> _mockRepo;
    private readonly ProductServiceImpl _service;

    public ProductServiceTests()
    {
        _mockRepo = new Mock<IProductRepository>();
        _service = new ProductServiceImpl(
            _mockRepo.Object,
            Mock.Of<ILogger<ProductServiceImpl>>());
    }

    [Fact]
    public async Task GetProduct_ReturnsProduct_WhenExists()
    {
        // Arrange
        var product = new Domain.Product { Id = 1, Name = "Laptop", Price = 999.99m };
        _mockRepo.Setup(r => r.GetByIdAsync(1)).ReturnsAsync(product);

        var context = TestServerCallContext.Create(
            method: "GetProduct",
            host: "localhost",
            deadline: DateTime.UtcNow.AddHours(1),
            requestHeaders: new Metadata(),
            cancellationToken: CancellationToken.None,
            peer: "127.0.0.1",
            authContext: null,
            contextPropagationToken: null,
            writeHeadersFunc: _ => Task.CompletedTask,
            writeOptionsGetter: () => WriteOptions.Default,
            writeOptionsSetter: _ => { });

        // Act
        var result = await _service.GetProduct(new GetProductRequest { Id = 1 }, context);

        // Assert
        Assert.Equal(1, result.Id);
        Assert.Equal("Laptop", result.Name);
        Assert.Equal(999.99, result.Price, precision: 2);
    }

    [Fact]
    public async Task GetProduct_ThrowsNotFound_WhenMissing()
    {
        _mockRepo.Setup(r => r.GetByIdAsync(999)).ReturnsAsync((Domain.Product?)null);

        var context = TestServerCallContext.Create(
            "GetProduct", "localhost", DateTime.UtcNow.AddHours(1),
            new Metadata(), CancellationToken.None,
            "127.0.0.1", null, null,
            _ => Task.CompletedTask, () => WriteOptions.Default, _ => { });

        var ex = await Assert.ThrowsAsync<RpcException>(
            () => _service.GetProduct(new GetProductRequest { Id = 999 }, context));

        Assert.Equal(StatusCode.NotFound, ex.StatusCode);
    }

    [Fact]
    public async Task ListProducts_StreamsAllProducts()
    {
        // Arrange
        var products = new[]
        {
            new Domain.Product { Id = 1, Name = "A" },
            new Domain.Product { Id = 2, Name = "B" },
            new Domain.Product { Id = 3, Name = "C" }
        };
        
        _mockRepo.Setup(r => r.GetAllAsyncEnumerable(It.IsAny<string>(), It.IsAny<string>(), false))
            .Returns(products.ToAsyncEnumerable());

        var streamWriter = new Mock<IServerStreamWriter<ProductResponse>>();
        streamWriter.Setup(w => w.WriteAsync(It.IsAny<ProductResponse>(), default))
            .Returns(Task.CompletedTask);

        var context = TestServerCallContext.Create(
            "ListProducts", "localhost", DateTime.UtcNow.AddHours(1),
            new Metadata(), CancellationToken.None,
            "127.0.0.1", null, null,
            _ => Task.CompletedTask, () => WriteOptions.Default, _ => { });

        // Act
        await _service.ListProducts(new ListProductsRequest(), streamWriter.Object, context);

        // Assert
        streamWriter.Verify(w => w.WriteAsync(It.IsAny<ProductResponse>(), default), Times.Exactly(3));
    }
}
```

---

## Step 768: gRPC Integration Testing

```csharp
// Tests/Integration/ProductServiceIntegrationTests.cs
using Microsoft.AspNetCore.Mvc.Testing;
using Grpc.Net.Client;

public class ProductServiceIntegrationTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;

    public ProductServiceIntegrationTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory;
    }

    private GrpcChannel CreateChannel()
    {
        var httpClient = _factory.CreateClient();
        return GrpcChannel.ForAddress(
            _factory.Server.BaseAddress, 
            new GrpcChannelOptions { HttpClient = httpClient });
    }

    [Fact]
    public async Task ProductService_GetProduct_ReturnsCorrectProduct()
    {
        using var channel = CreateChannel();
        var client = new ProductService.ProductServiceClient(channel);

        var response = await client.GetProductAsync(new GetProductRequest { Id = 1 });

        Assert.NotNull(response);
        Assert.Equal(1, response.Id);
    }

    [Fact]
    public async Task ProductService_ListProducts_ReturnsStream()
    {
        using var channel = CreateChannel();
        var client = new ProductService.ProductServiceClient(channel);

        var responses = new List<ProductResponse>();
        using var call = client.ListProducts(new ListProductsRequest());
        
        await foreach (var product in call.ResponseStream.ReadAllAsync())
        {
            responses.Add(product);
        }

        Assert.NotEmpty(responses);
    }
}
```

---

## Step 769: gRPC-JSON Gateway Pattern

```csharp
// API Gateway: translates REST -> gRPC

[ApiController]
[Route("gateway")]
public class GatewayController : ControllerBase
{
    private readonly GrpcJsonMapper _mapper;
    private readonly ProductService.ProductServiceClient _grpcClient;

    // Forward REST requests to gRPC services
    [HttpGet("products/{id}")]
    public async Task<IActionResult> GetProduct(int id)
    {
        try
        {
            var grpcResponse = await _grpcClient.GetProductAsync(
                new GetProductRequest { Id = id });
            
            return Ok(_mapper.MapToDto(grpcResponse));
        }
        catch (RpcException ex) when (ex.StatusCode == StatusCode.NotFound)
        {
            return NotFound(new { message = ex.Status.Detail });
        }
        catch (RpcException ex)
        {
            return StatusCode(
                MapGrpcStatusToHttp(ex.StatusCode),
                new { message = ex.Status.Detail });
        }
    }

    private static int MapGrpcStatusToHttp(StatusCode grpcStatus) => grpcStatus switch
    {
        StatusCode.OK => 200,
        StatusCode.InvalidArgument => 400,
        StatusCode.Unauthenticated => 401,
        StatusCode.PermissionDenied => 403,
        StatusCode.NotFound => 404,
        StatusCode.AlreadyExists => 409,
        StatusCode.ResourceExhausted => 429,
        _ => 500
    };
}
```

---

## Step 770: Deployment gRPC

```dockerfile
# Dockerfile for gRPC service
FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS base
WORKDIR /app
EXPOSE 5001
EXPOSE 5002

FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /src
COPY ["GrpcServer.csproj", "."]
RUN dotnet restore
COPY . .
RUN dotnet build -c Release -o /app/build

FROM build AS publish
RUN dotnet publish -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "GrpcServer.dll"]
```

```yaml
# kubernetes/grpc-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-grpc-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: product-grpc
  template:
    metadata:
      labels:
        app: product-grpc
    spec:
      containers:
      - name: product-grpc
        image: myregistry/product-grpc:latest
        ports:
        - containerPort: 5001
          name: grpc
        livenessProbe:
          grpc:
            port: 5001
          initialDelaySeconds: 10
          periodSeconds: 30
        readinessProbe:
          grpc:
            port: 5001
          initialDelaySeconds: 5
          periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: product-grpc-service
spec:
  selector:
    app: product-grpc
  ports:
  - port: 5001
    targetPort: 5001
    name: grpc
  type: ClusterIP
```

```csharp
// appsettings.json - Production gRPC config
{
    "Kestrel": {
        "Endpoints": {
            "Grpc": {
                "Url": "https://+:5001",
                "Protocols": "Http2"
            },
            "GrpcInsecure": {
                "Url": "http://+:5002",
                "Protocols": "Http2"
            }
        }
    }
}
```

---

## สรุป Part 28: gRPC

ในส่วนนี้เราได้เรียนรู้:

| หัวข้อ | ขั้นตอน |
|--------|---------|
| gRPC Overview และ Protocol Buffers | 751 |
| .proto file syntax และ Service types | 752 |
| Service Implementation (Unary, Streaming) | 753 |
| Server Setup และ Reflection | 754 |
| Interceptors (Logging, Exception, Auth) | 755 |
| gRPC .NET Client | 756 |
| Client Factory (DI Integration) | 757 |
| gRPC-Web (Browser Support) | 758 |
| Error Handling และ Metadata | 759 |
| Health Checks | 760 |
| Performance Optimization | 761 |
| REST to gRPC Transcoding | 762 |
| gRPC vs REST | 763 |
| Microservices Communication | 764 |
| Polly Resilience | 765 |
| Reflection & API Discovery | 766 |
| Unit Testing | 767 |
| Integration Testing | 768 |
| gRPC-JSON Gateway | 769 |
| Docker/Kubernetes Deployment | 770 |

**ขั้นตอนต่อไป**: Part 29 จะเรียนรู้ **GraphQL** ด้วย Hot Chocolate
