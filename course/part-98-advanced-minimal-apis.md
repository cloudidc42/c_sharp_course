# Part 98: Advanced Minimal APIs Patterns

## Steps 2197-2212

Minimal APIs ใน .NET 9 พร้อม endpoint filters, route groups, OpenAPI/Swagger advanced configuration, typed results, validation, และ production patterns

---

## Step 2197: Minimal APIs Architecture Patterns

```
Minimal API Organization
========================

Program.cs
  └── RouteGroupBuilder extensions
        ├── /api/v1/products    → ProductEndpoints
        ├── /api/v1/orders      → OrderEndpoints
        ├── /api/v1/customers   → CustomerEndpoints
        └── /api/v1/auth        → AuthEndpoints

Each endpoint module:
  ├── MapEndpoints() extension method
  ├── Request/Response records
  ├── Endpoint filters (validation, auth)
  └── OpenAPI metadata attributes
```

---

## Step 2198: Route Groups และ Endpoint Modules

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);
// ... services ...
var app = builder.Build();

var api = app.MapGroup("/api")
    .RequireAuthorization()
    .AddEndpointFilter<RequestLoggingFilter>()
    .WithOpenApi();

var v1 = api.MapGroup("/v1")
    .WithTags("v1");

v1.MapProductEndpoints();
v1.MapOrderEndpoints();
v1.MapCustomerEndpoints();

var authGroup = app.MapGroup("/api/auth")
    .AllowAnonymous();
authGroup.MapAuthEndpoints();

app.Run();

// Endpoints/ProductEndpoints.cs
public static class ProductEndpoints
{
    public static RouteGroupBuilder MapProductEndpoints(this RouteGroupBuilder group)
    {
        var products = group.MapGroup("/products")
            .WithTags("Products")
            .WithOpenApi();

        products.MapGet("/", GetProducts)
            .WithName("GetProducts")
            .WithSummary("Get paginated product list")
            .Produces<PagedResult<ProductDto>>()
            .Produces(401);

        products.MapGet("/{id:guid}", GetProduct)
            .WithName("GetProduct")
            .WithSummary("Get product by ID")
            .Produces<ProductDto>()
            .Produces(404)
            .Produces(401);

        products.MapPost("/", CreateProduct)
            .WithName("CreateProduct")
            .WithSummary("Create a new product")
            .Accepts<CreateProductRequest>("application/json")
            .Produces<ProductDto>(201)
            .Produces<ValidationProblemDetails>(400)
            .Produces(401)
            .RequireAuthorization("admin");

        products.MapPut("/{id:guid}", UpdateProduct)
            .WithName("UpdateProduct")
            .RequireAuthorization("admin");

        products.MapDelete("/{id:guid}", DeleteProduct)
            .WithName("DeleteProduct")
            .RequireAuthorization("admin");

        return group;
    }

    private static async Task<Ok<PagedResult<ProductDto>>> GetProducts(
        [AsParameters] ProductQueryParams query,
        IProductService service,
        CancellationToken ct)
    {
        var result = await service.GetProductsAsync(query, ct);
        return TypedResults.Ok(result);
    }

    private static async Task<Results<Ok<ProductDto>, NotFound>> GetProduct(
        Guid id, IProductService service, CancellationToken ct)
    {
        var product = await service.GetByIdAsync(id, ct);
        return product is not null
            ? TypedResults.Ok(product)
            : TypedResults.NotFound();
    }

    private static async Task<Results<Created<ProductDto>, ValidationProblem>> CreateProduct(
        CreateProductRequest request,
        IProductService service,
        CancellationToken ct)
    {
        var product = await service.CreateAsync(request, ct);
        return TypedResults.Created($"/api/v1/products/{product.Id}", product);
    }

    private static async Task<Results<Ok<ProductDto>, NotFound, ValidationProblem>> UpdateProduct(
        Guid id, UpdateProductRequest request,
        IProductService service, CancellationToken ct)
    {
        var product = await service.UpdateAsync(id, request, ct);
        return product is not null
            ? TypedResults.Ok(product)
            : TypedResults.NotFound();
    }

    private static async Task<Results<NoContent, NotFound>> DeleteProduct(
        Guid id, IProductService service, CancellationToken ct)
    {
        var deleted = await service.DeleteAsync(id, ct);
        return deleted ? TypedResults.NoContent() : TypedResults.NotFound();
    }
}
```

---

## Step 2199: Endpoint Filters

```csharp
// Filters/ValidationFilter.cs
public class ValidationFilter<TRequest> : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context,
        EndpointFilterDelegate next)
    {
        var validator = context.HttpContext.RequestServices
            .GetService<IValidator<TRequest>>();

        if (validator is null)
            return await next(context);

        var request = context.Arguments.OfType<TRequest>().FirstOrDefault();
        if (request is null)
            return await next(context);

        var result = await validator.ValidateAsync(request);
        if (!result.IsValid)
        {
            return TypedResults.ValidationProblem(
                result.ToDictionary(),
                title: "Validation failed",
                detail: "One or more validation errors occurred");
        }

        return await next(context);
    }
}

// Generic filter factory extension
public static class EndpointFilterExtensions
{
    public static RouteHandlerBuilder WithValidation<TRequest>(
        this RouteHandlerBuilder builder) =>
        builder.AddEndpointFilter<ValidationFilter<TRequest>>();
}

// Usage
products.MapPost("/", CreateProduct)
    .WithValidation<CreateProductRequest>();

// Filters/RequestLoggingFilter.cs
public class RequestLoggingFilter(ILogger<RequestLoggingFilter> logger) : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context,
        EndpointFilterDelegate next)
    {
        var endpoint = context.HttpContext.GetEndpoint();
        var name = endpoint?.Metadata.GetMetadata<EndpointNameMetadata>()?.EndpointName ?? "Unknown";

        using var scope = logger.BeginScope(new Dictionary<string, object>
        {
            ["EndpointName"] = name,
            ["RequestId"] = context.HttpContext.TraceIdentifier
        });

        logger.LogInformation("Executing endpoint {EndpointName}", name);
        var sw = Stopwatch.StartNew();

        try
        {
            var result = await next(context);
            logger.LogInformation("Endpoint {EndpointName} completed in {Duration}ms",
                name, sw.ElapsedMilliseconds);
            return result;
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "Endpoint {EndpointName} failed after {Duration}ms",
                name, sw.ElapsedMilliseconds);
            throw;
        }
    }
}

// Filters/IdempotencyFilter.cs
public class IdempotencyFilter(IDistributedCache cache) : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context,
        EndpointFilterDelegate next)
    {
        var request = context.HttpContext.Request;
        if (!request.Headers.TryGetValue("Idempotency-Key", out var key))
            return await next(context);

        var cacheKey = $"idempotency:{key}";
        var cached = await cache.GetStringAsync(cacheKey);
        if (cached is not null)
        {
            context.HttpContext.Response.Headers["X-Idempotency-Replayed"] = "true";
            return Results.Json(JsonSerializer.Deserialize<JsonElement>(cached));
        }

        var result = await next(context);

        if (result is IValueHttpResult { Value: not null } valueResult)
        {
            var json = JsonSerializer.Serialize(valueResult.Value);
            await cache.SetStringAsync(cacheKey, json,
                new DistributedCacheEntryOptions
                {
                    AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(24)
                });
        }

        return result;
    }
}
```

---

## Step 2200: TypedResults และ Union Return Types

```csharp
// TypedResults provide compile-time checking and better OpenAPI metadata

// Before (.NET 6)
app.MapGet("/product/{id}", async (Guid id, IProductService svc) =>
{
    var product = await svc.GetByIdAsync(id);
    if (product is null) return Results.NotFound();
    return Results.Ok(product);  // IResult - no type info
});

// After (.NET 8+) - union return types
app.MapGet("/product/{id}", async (Guid id, IProductService svc) =>
{
    var product = await svc.GetByIdAsync(id);
    return product is not null
        ? TypedResults.Ok(product)      // Ok<ProductDto>
        : TypedResults.NotFound();      // NotFound
}) // Return type: Results<Ok<ProductDto>, NotFound>
.Produces<ProductDto>()
.Produces(404);

// Complex example with multiple outcomes
private static async Task<Results<
    Created<OrderDto>,
    Conflict<ProblemDetails>,
    ValidationProblem,
    UnprocessableEntity<ProblemDetails>>> CreateOrder(
    CreateOrderRequest request,
    IOrderService service,
    CancellationToken ct)
{
    try
    {
        var order = await service.CreateAsync(request, ct);
        return TypedResults.Created($"/api/v1/orders/{order.Id}", order);
    }
    catch (DuplicateOrderException ex)
    {
        return TypedResults.Conflict(new ProblemDetails
        {
            Title = "Duplicate order",
            Detail = ex.Message,
            Status = 409
        });
    }
    catch (InsufficientStockException ex)
    {
        return TypedResults.UnprocessableEntity(new ProblemDetails
        {
            Title = "Insufficient stock",
            Detail = ex.Message,
            Status = 422
        });
    }
}

// Custom TypedResult
public class AcceptedWithTrackingResult : IResult
{
    private readonly string _trackingId;
    private readonly string _statusUrl;

    public AcceptedWithTrackingResult(string trackingId, string statusUrl)
    {
        _trackingId = trackingId;
        _statusUrl = statusUrl;
    }

    public async Task ExecuteAsync(HttpContext httpContext)
    {
        httpContext.Response.StatusCode = 202;
        httpContext.Response.Headers.Location = _statusUrl;
        httpContext.Response.Headers["X-Tracking-Id"] = _trackingId;
        await httpContext.Response.WriteAsJsonAsync(new
        {
            trackingId = _trackingId,
            statusUrl = _statusUrl,
            message = "Request accepted for processing"
        });
    }
}
```

---

## Step 2201: Parameter Binding

```csharp
// All binding approaches

// Route parameters
app.MapGet("/orders/{id:guid}", (Guid id) => { });

// Query string (automatic)
app.MapGet("/products", ([FromQuery] string? category, [FromQuery] int page = 1) => { });

// [AsParameters] - bind entire object from route/query
public record ProductQueryParams(
    [property: FromQuery] string? Category,
    [property: FromQuery] string? Search,
    [property: FromQuery] int Page = 1,
    [property: FromQuery] int PageSize = 20,
    [property: FromQuery] string? SortBy = "name",
    [property: FromQuery] bool Descending = false);

app.MapGet("/products", ([AsParameters] ProductQueryParams query) => { });

// Headers
app.MapPost("/webhooks", (
    [FromHeader(Name = "X-Signature")] string signature,
    [FromBody] JsonDocument payload) => { });

// Custom binding with IParsable<T>
public record OrderId : IParsable<OrderId>
{
    public Guid Value { get; init; }

    public static OrderId Parse(string s, IFormatProvider? provider)
    {
        if (!Guid.TryParse(s, out var guid))
            throw new FormatException($"Invalid OrderId: {s}");
        return new OrderId { Value = guid };
    }

    public static bool TryParse(string? s, IFormatProvider? provider, out OrderId result)
    {
        if (Guid.TryParse(s, out var guid))
        {
            result = new OrderId { Value = guid };
            return true;
        }
        result = default!;
        return false;
    }
}

// Now works as route/query parameter automatically
app.MapGet("/orders/{orderId}", (OrderId orderId) => { });

// BindAsync for complex types
public class PaginationOptions
{
    public int Page { get; init; } = 1;
    public int PageSize { get; init; } = 20;

    public static ValueTask<PaginationOptions?> BindAsync(
        HttpContext context, ParameterInfo parameter)
    {
        int.TryParse(context.Request.Query["page"], out var page);
        int.TryParse(context.Request.Query["pageSize"], out var pageSize);

        return ValueTask.FromResult<PaginationOptions?>(new PaginationOptions
        {
            Page = Math.Max(1, page),
            PageSize = Math.Clamp(pageSize == 0 ? 20 : pageSize, 1, 100)
        });
    }
}
```

---

## Step 2202: OpenAPI/Swagger Advanced

```csharp
// .NET 9 built-in OpenAPI (no Swashbuckle needed!)
builder.Services.AddOpenApi(options =>
{
    options.AddDocumentTransformer<BearerSecuritySchemeTransformer>();
    options.AddDocumentTransformer((document, context, ct) =>
    {
        document.Info = new()
        {
            Title = "E-Commerce API",
            Version = "v1",
            Description = """
                ## Overview
                The E-Commerce API provides endpoints for managing products, orders, and customers.

                ## Authentication
                Use Bearer token authentication. Obtain a token from `/api/auth/login`.
                
                ## Rate Limiting
                API calls are rate limited to 100 requests per minute per IP.
                """,
            Contact = new()
            {
                Name = "API Support",
                Email = "api@ecommerce.com"
            },
            License = new()
            {
                Name = "Proprietary"
            }
        };
        return Task.CompletedTask;
    });
    options.AddOperationTransformer<IdempotencyKeyOperationTransformer>();
});

// BearerSecuritySchemeTransformer.cs
public class BearerSecuritySchemeTransformer(
    IAuthenticationSchemeProvider schemeProvider) : IOpenApiDocumentTransformer
{
    public async Task TransformAsync(
        OpenApiDocument document, OpenApiDocumentTransformerContext context, CancellationToken ct)
    {
        var authSchemes = await schemeProvider.GetAllSchemesAsync();
        if (!authSchemes.Any(s => s.Name == JwtBearerDefaults.AuthenticationScheme))
            return;

        document.Components ??= new OpenApiComponents();
        document.Components.SecuritySchemes ??= new Dictionary<string, OpenApiSecurityScheme>();
        document.Components.SecuritySchemes["BearerAuth"] = new OpenApiSecurityScheme
        {
            Type = SecuritySchemeType.Http,
            Scheme = "bearer",
            BearerFormat = "JWT",
            Description = "Enter your JWT token"
        };

        foreach (var operation in document.Paths.Values.SelectMany(p => p.Operations.Values))
        {
            operation.Security ??= [];
            operation.Security.Add(new OpenApiSecurityRequirement
            {
                [new OpenApiSecurityScheme
                {
                    Reference = new OpenApiReference
                    {
                        Type = ReferenceType.SecurityScheme,
                        Id = "BearerAuth"
                    }
                }] = []
            });
        }
    }
}

// Endpoint metadata via attributes
[EndpointSummary("Create a new product")]
[EndpointDescription("Creates a new product with the specified details. Requires admin role.")]
[ProducesResponseType<ProductDto>(201)]
[ProducesResponseType<ValidationProblemDetails>(400)]
private static async Task<Results<Created<ProductDto>, ValidationProblem>> CreateProduct(
    CreateProductRequest request, IProductService service, CancellationToken ct)
{
    var product = await service.CreateAsync(request, ct);
    return TypedResults.Created($"/api/v1/products/{product.Id}", product);
}
```

---

## Step 2203: Versioning

```csharp
// Package: Asp.Versioning.Http
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;
    options.ApiVersionReader = ApiVersionReader.Combine(
        new UrlSegmentApiVersionReader(),
        new HeaderApiVersionReader("X-API-Version"),
        new QueryStringApiVersionReader("api-version"));
});

var versionSet = app.NewApiVersionSet()
    .HasApiVersion(new ApiVersion(1, 0))
    .HasApiVersion(new ApiVersion(2, 0))
    .ReportApiVersions()
    .Build();

// V1 routes
var v1 = app.MapGroup("/api/v{version:apiVersion}")
    .WithApiVersionSet(versionSet)
    .MapToApiVersion(1, 0);

v1.MapGet("/products", GetProductsV1);

// V2 routes (with breaking changes)
var v2 = app.MapGroup("/api/v{version:apiVersion}")
    .WithApiVersionSet(versionSet)
    .MapToApiVersion(2, 0);

v2.MapGet("/products", GetProductsV2)
    .WithName("GetProducts-V2");

// V2 returns richer data
private static async Task<Ok<PagedResult<ProductDetailDto>>> GetProductsV2(
    [AsParameters] ProductQueryParams query,
    IProductService service,
    CancellationToken ct)
{
    var result = await service.GetProductsDetailedAsync(query, ct);
    return TypedResults.Ok(result);
}
```

---

## Step 2204: Rate Limiting

```csharp
// .NET 7+ built-in rate limiting
builder.Services.AddRateLimiter(options =>
{
    options.RejectionStatusCode = 429;

    options.OnRejected = async (context, ct) =>
    {
        context.HttpContext.Response.Headers.RetryAfter = "60";
        await context.HttpContext.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Title = "Too Many Requests",
            Detail = "You have exceeded the rate limit. Please try again later.",
            Status = 429
        }, ct);
    };

    // Global sliding window: 100 req/min per IP
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(
        context => RateLimitPartition.GetSlidingWindowLimiter(
            partitionKey: context.Connection.RemoteIpAddress?.ToString() ?? "anonymous",
            factory: _ => new SlidingWindowRateLimiterOptions
            {
                PermitLimit = 100,
                Window = TimeSpan.FromMinutes(1),
                SegmentsPerWindow = 4,
                QueueProcessingOrder = QueueProcessingOrder.OldestFirst,
                QueueLimit = 10
            }));

    // Named policy for specific endpoints
    options.AddPolicy("CreateOrder", httpContext =>
        RateLimitPartition.GetTokenBucketLimiter(
            partitionKey: httpContext.User.FindFirstValue(ClaimTypes.NameIdentifier) ?? "anon",
            factory: _ => new TokenBucketRateLimiterOptions
            {
                TokenLimit = 10,
                QueueProcessingOrder = QueueProcessingOrder.OldestFirst,
                QueueLimit = 2,
                ReplenishmentPeriod = TimeSpan.FromMinutes(1),
                TokensPerPeriod = 10,
                AutoReplenishment = true
            }));

    // Concurrency limiter for expensive operations
    options.AddConcurrencyLimiter("ExpensiveOperation",
        options =>
        {
            options.PermitLimit = 5;
            options.QueueProcessingOrder = QueueProcessingOrder.NewestFirst;
            options.QueueLimit = 10;
        });
});

app.UseRateLimiter();

// Apply per endpoint
orders.MapPost("/", CreateOrder)
    .RequireRateLimiting("CreateOrder");

reports.MapGet("/export", ExportReport)
    .RequireRateLimiting("ExpensiveOperation");
```

---

## Step 2205: Output Caching

```csharp
builder.Services.AddOutputCache(options =>
{
    options.AddBasePolicy(policy => policy.Expire(TimeSpan.FromSeconds(10)));

    options.AddPolicy("Products", policy =>
        policy.Expire(TimeSpan.FromMinutes(5))
              .SetVaryByQuery("category", "search", "page", "pageSize")
              .Tag("products"));

    options.AddPolicy("ProductDetail", policy =>
        policy.Expire(TimeSpan.FromMinutes(10))
              .SetVaryByRouteValue("id")
              .Tag("products", "product-detail"));
});

// Package: Aspire.StackExchange.Redis.OutputCaching (for distributed)
builder.AddRedisOutputCache("redis");

app.UseOutputCache();

// Apply per endpoint
products.MapGet("/", GetProducts)
    .CacheOutput("Products");

products.MapGet("/{id:guid}", GetProduct)
    .CacheOutput("ProductDetail");

// Invalidate cache when product changes
public class ProductService(IOutputCacheStore cacheStore)
{
    public async Task UpdateProductAsync(Guid id, UpdateProductRequest request, CancellationToken ct)
    {
        // ... update product ...
        
        // Invalidate by tag
        await cacheStore.EvictByTagAsync("products", ct);
    }
}
```

---

## Step 2206: Problem Details

```csharp
// .NET 7+ built-in Problem Details
builder.Services.AddProblemDetails(options =>
{
    options.CustomizeProblemDetails = context =>
    {
        context.ProblemDetails.Extensions["traceId"] =
            context.HttpContext.TraceIdentifier;
        context.ProblemDetails.Extensions["instance"] =
            context.HttpContext.Request.Path;

        if (context.Exception is BusinessException businessEx)
        {
            context.ProblemDetails.Type = $"https://errors.ecommerce.com/{businessEx.ErrorCode}";
            context.ProblemDetails.Extensions["errorCode"] = businessEx.ErrorCode;
        }
    };
});

// Exception handler middleware
app.UseExceptionHandler();

// Custom exception handler for specific types
app.UseExceptionHandler(exceptionHandlerApp =>
{
    exceptionHandlerApp.Run(async context =>
    {
        var exceptionHandlerFeature = context.Features.Get<IExceptionHandlerFeature>();
        var exception = exceptionHandlerFeature?.Error;

        var problemDetails = exception switch
        {
            NotFoundException ex => new ProblemDetails
            {
                Status = 404,
                Title = "Resource not found",
                Detail = ex.Message
            },
            ConflictException ex => new ProblemDetails
            {
                Status = 409,
                Title = "Conflict",
                Detail = ex.Message
            },
            ValidationException ex => new ValidationProblemDetails(
                ex.Errors.GroupBy(e => e.PropertyName)
                    .ToDictionary(g => g.Key, g => g.Select(e => e.ErrorMessage).ToArray()))
            {
                Status = 400
            },
            _ => new ProblemDetails
            {
                Status = 500,
                Title = "An unexpected error occurred",
                Detail = context.RequestServices
                    .GetRequiredService<IHostEnvironment>().IsDevelopment()
                    ? exception?.ToString()
                    : null
            }
        };

        problemDetails.Extensions["traceId"] = context.TraceIdentifier;
        context.Response.StatusCode = problemDetails.Status ?? 500;
        await context.Response.WriteAsJsonAsync(problemDetails);
    });
});
```

---

## Step 2207: File Upload và Download

```csharp
// File upload
app.MapPost("/api/products/{id:guid}/images", async (
    Guid id,
    IFormFile file,
    IProductImageService imageService,
    CancellationToken ct) =>
{
    if (file.ContentType is not ("image/jpeg" or "image/png" or "image/webp"))
        return Results.BadRequest("Only JPEG, PNG, and WebP images are allowed");

    if (file.Length > 5 * 1024 * 1024)  // 5MB
        return Results.BadRequest("File size must be less than 5MB");

    await using var stream = file.OpenReadStream();
    var imageUrl = await imageService.UploadAsync(id, stream, file.FileName, ct);
    return Results.Ok(new { imageUrl });
})
.DisableAntiforgery()
.Accepts<IFormFile>("multipart/form-data")
.RequireAuthorization("admin");

// Multiple files
app.MapPost("/api/products/{id:guid}/images/bulk", async (
    Guid id,
    IFormFileCollection files,
    IProductImageService imageService,
    CancellationToken ct) =>
{
    var results = new List<object>();
    foreach (var file in files)
    {
        await using var stream = file.OpenReadStream();
        var url = await imageService.UploadAsync(id, stream, file.FileName, ct);
        results.Add(new { fileName = file.FileName, url });
    }
    return Results.Ok(results);
})
.DisableAntiforgery();

// File download
app.MapGet("/api/orders/{id:guid}/invoice", async (
    Guid id,
    IInvoiceService invoiceService,
    CancellationToken ct) =>
{
    var invoice = await invoiceService.GeneratePdfAsync(id, ct);
    if (invoice is null) return Results.NotFound();

    return Results.File(
        invoice.Content,
        contentType: "application/pdf",
        fileDownloadName: $"invoice-{id:N}.pdf",
        lastModified: invoice.GeneratedAt,
        entityTag: new EntityTagHeaderValue($"\"{invoice.ETag}\""));
});
```

---

## Step 2208: Streaming Endpoints

```csharp
// Server-Sent Events (SSE)
app.MapGet("/api/orders/{id:guid}/status", async (
    Guid id,
    IOrderStatusService statusService,
    HttpContext context,
    CancellationToken ct) =>
{
    context.Response.Headers.ContentType = "text/event-stream";
    context.Response.Headers.CacheControl = "no-cache";
    context.Response.Headers["X-Accel-Buffering"] = "no";

    await foreach (var status in statusService.WatchOrderAsync(id, ct))
    {
        var json = JsonSerializer.Serialize(new { status, timestamp = DateTimeOffset.UtcNow });
        await context.Response.WriteAsync($"data: {json}\n\n", ct);
        await context.Response.Body.FlushAsync(ct);
    }
});

// IAsyncEnumerable streaming
app.MapGet("/api/reports/sales-stream", (
    ISalesService salesService,
    CancellationToken ct) =>
{
    return salesService.GetSalesDataStreamAsync(ct);
    // ASP.NET Core automatically serializes as JSON array streamed
})
.Produces<IAsyncEnumerable<SalesRecord>>();

// WebSockets
app.MapGet("/ws/orders", async (
    HttpContext context,
    IOrderWebSocketService wsService) =>
{
    if (!context.WebSockets.IsWebSocketRequest)
    {
        context.Response.StatusCode = 400;
        return;
    }

    using var webSocket = await context.WebSockets.AcceptWebSocketAsync();
    await wsService.HandleAsync(webSocket, context.RequestAborted);
});
```

---

## Step 2209: Middleware as Endpoint Filters

```csharp
// Combine middleware-style logic with endpoint filters

// ApiKeyFilter - validates API key for service-to-service calls
public class ApiKeyFilter(IConfiguration config) : IEndpointFilter
{
    private const string ApiKeyHeader = "X-API-Key";

    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context,
        EndpointFilterDelegate next)
    {
        if (!context.HttpContext.Request.Headers.TryGetValue(ApiKeyHeader, out var apiKey))
            return TypedResults.Unauthorized();

        var validKey = config["ServiceApiKey"];
        if (!CryptographicOperations.FixedTimeEquals(
            Encoding.UTF8.GetBytes(apiKey.ToString()),
            Encoding.UTF8.GetBytes(validKey!)))
        {
            return TypedResults.Unauthorized();
        }

        return await next(context);
    }
}

// TenantResolutionFilter
public class TenantResolutionFilter(ITenantResolver tenantResolver) : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context,
        EndpointFilterDelegate next)
    {
        var httpContext = context.HttpContext;
        var tenant = await tenantResolver.ResolveAsync(httpContext);

        if (tenant is null)
            return TypedResults.BadRequest("Invalid or missing tenant");

        httpContext.Items["Tenant"] = tenant;
        httpContext.Items["TenantId"] = tenant.Id;

        return await next(context);
    }
}

// Combine filters with extension
public static RouteGroupBuilder RequireApiKey(this RouteGroupBuilder group) =>
    group.AddEndpointFilter<ApiKeyFilter>();

public static RouteGroupBuilder WithTenantResolution(this RouteGroupBuilder group) =>
    group.AddEndpointFilter<TenantResolutionFilter>();

// Usage
var internalApi = app.MapGroup("/internal")
    .RequireApiKey()
    .WithTenantResolution();
```

---

## Step 2210: Complete Request/Response Pipeline

```csharp
// Full pipeline for a minimal API endpoint
// Showing all the layers

/*
Request
  → Rate Limiter
  → Authentication middleware
  → Authorization middleware
  → OutputCache middleware
  → Route matching
  → RequestLoggingFilter
  → TenantResolutionFilter
  → ValidationFilter<TRequest>
  → IdempotencyFilter
  → Handler function
  → Response
*/

// Full example: Create Order endpoint
orders.MapPost("/", CreateOrder)
    .WithName("CreateOrder")
    .WithSummary("Place a new order")
    .WithDescription("Creates a new order for the authenticated customer.")
    .Accepts<CreateOrderRequest>("application/json")
    .Produces<OrderDto>(201)
    .Produces<ValidationProblemDetails>(400)
    .Produces<ProblemDetails>(409)
    .Produces(401)
    .Produces(429)
    .WithOpenApi(operation =>
    {
        operation.Parameters.Add(new OpenApiParameter
        {
            Name = "Idempotency-Key",
            In = ParameterLocation.Header,
            Required = false,
            Schema = new JsonSchemaBuilder().Type(SchemaValueType.String).Build(),
            Description = "Optional idempotency key to prevent duplicate orders"
        });
        return operation;
    })
    .WithValidation<CreateOrderRequest>()
    .AddEndpointFilter<IdempotencyFilter>()
    .RequireRateLimiting("CreateOrder")
    .RequireAuthorization();

private static async Task<Results<Created<OrderDto>, ValidationProblem, Conflict<ProblemDetails>>> CreateOrder(
    [FromBody] CreateOrderRequest request,
    [FromServices] IOrderService orderService,
    [FromServices] ICurrentUserService currentUser,
    CancellationToken ct)
{
    var command = request with { CustomerId = currentUser.UserId };
    try
    {
        var order = await orderService.CreateAsync(command, ct);
        return TypedResults.Created($"/api/v1/orders/{order.Id}", order);
    }
    catch (DuplicateOrderException ex)
    {
        return TypedResults.Conflict(new ProblemDetails
        {
            Title = "Duplicate order",
            Detail = ex.Message,
            Status = 409
        });
    }
}
```

---

## Step 2211: Testing Minimal APIs

```csharp
// Integration tests for minimal API endpoints
public class ProductEndpointTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public ProductEndpointTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureTestServices(services =>
            {
                services.AddSingleton<IProductService, FakeProductService>();
            });
        }).CreateClient();
    }

    [Fact]
    public async Task GetProducts_ReturnsPagedResult()
    {
        var response = await _client.GetAsync("/api/v1/products?page=1&pageSize=10");

        response.StatusCode.Should().Be(HttpStatusCode.OK);
        var result = await response.Content.ReadFromJsonAsync<PagedResult<ProductDto>>();
        result.Should().NotBeNull();
        result!.Items.Should().NotBeEmpty();
    }

    [Fact]
    public async Task CreateProduct_WithoutAuth_Returns401()
    {
        var response = await _client.PostAsJsonAsync("/api/v1/products",
            new CreateProductRequest("Test", "Desc", 100m, "Electronics"));

        response.StatusCode.Should().Be(HttpStatusCode.Unauthorized);
    }

    [Fact]
    public async Task CreateProduct_WithInvalidData_Returns400()
    {
        _client.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", GetAdminToken());

        var response = await _client.PostAsJsonAsync("/api/v1/products",
            new CreateProductRequest("", "", -1, ""));

        response.StatusCode.Should().Be(HttpStatusCode.BadRequest);
        var problem = await response.Content.ReadFromJsonAsync<ValidationProblemDetails>();
        problem!.Errors.Should().ContainKey("Name");
        problem.Errors.Should().ContainKey("Price");
    }
}
```

---

## Step 2212: Minimal API Best Practices Summary

```csharp
// 1. Use route groups for organization
// 2. Use TypedResults for type safety and OpenAPI metadata
// 3. Use endpoint filters for cross-cutting concerns
// 4. Use [AsParameters] for complex parameter binding
// 5. Register endpoints in extension methods (not Program.cs)
// 6. Use built-in rate limiting and output caching
// 7. Use built-in problem details

// Anti-patterns to avoid:
// ❌ Putting business logic in endpoint handlers
// ❌ Using Results.Ok() instead of TypedResults.Ok()
// ❌ Not using cancellation tokens
// ❌ Not adding OpenAPI metadata
// ❌ Mixing V1 and V2 routes in the same group

// Complete Minimal API structure:
/*
src/
├── Program.cs                  (setup only: builder, middlewares, app.Run)
├── Endpoints/
│   ├── ProductEndpoints.cs
│   ├── OrderEndpoints.cs
│   └── AuthEndpoints.cs
├── Filters/
│   ├── ValidationFilter.cs
│   ├── RequestLoggingFilter.cs
│   └── IdempotencyFilter.cs
├── Models/
│   ├── Requests/
│   └── Responses/
└── Extensions/
    ├── ServiceExtensions.cs
    └── EndpointExtensions.cs
*/

// Performance tips:
// - Use record types for DTOs (faster equality checks)
// - Use IAsyncEnumerable for large datasets
// - Use Output Caching with Redis for frequently read data
// - Use Rate Limiting to protect against abuse
// - Use structured logging (ILogger, not Console.WriteLine)
// - Compile route constraints once on startup
// - Use minimal middleware: only add what you need
```
