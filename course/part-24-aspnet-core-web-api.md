# Part 24: ASP.NET Core Web API

## ขั้นตอนที่ 650: สร้าง Web API Project

```bash
# สร้างโปรเจกต์
dotnet new webapi -n MyApi --framework net9.0
cd MyApi

# เพิ่ม packages ที่ต้องการ
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Design
dotnet add package AutoMapper.Extensions.Microsoft.DependencyInjection
dotnet add package FluentValidation.AspNetCore
dotnet add package Swashbuckle.AspNetCore
dotnet add package Serilog.AspNetCore
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer

# รัน
dotnet run
```

## ขั้นตอนที่ 651: Program.cs - Application Setup

```csharp
// Program.cs
using System.Reflection;
using Microsoft.OpenApi.Models;
using Serilog;

var builder = WebApplication.CreateBuilder(args);

// === Serilog ===
builder.Host.UseSerilog((context, services, config) =>
{
    config
        .ReadFrom.Configuration(context.Configuration)
        .ReadFrom.Services(services)
        .Enrich.FromLogContext()
        .WriteTo.Console()
        .WriteTo.File("logs/api-.txt", rollingInterval: RollingInterval.Day);
});

// === Services ===
builder.Services.AddControllers(options =>
{
    options.Filters.Add<ValidationFilter>(); // Global filter
    options.ReturnHttpNotAcceptable = true;  // 406 for unsupported formats
})
.AddJsonOptions(options =>
{
    options.JsonSerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;
    options.JsonSerializerOptions.DefaultIgnoreCondition = 
        JsonIgnoreCondition.WhenWritingNull;
});

// Swagger/OpenAPI
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "My API",
        Version = "v1",
        Description = "A comprehensive REST API",
        Contact = new OpenApiContact { Name = "Dev Team", Email = "dev@example.com" }
    });
    
    // XML documentation
    var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
    c.IncludeXmlComments(Path.Combine(AppContext.BaseDirectory, xmlFile));
    
    // JWT auth in Swagger UI
    c.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Type = SecuritySchemeType.Http,
        Scheme = "bearer",
        BearerFormat = "JWT",
        Description = "Enter your JWT token"
    });
    c.AddSecurityRequirement(new OpenApiSecurityRequirement
    {
        {
            new OpenApiSecurityScheme
            {
                Reference = new OpenApiReference { Type = ReferenceType.SecurityScheme, Id = "Bearer" }
            },
            Array.Empty<string>()
        }
    });
});

// Database
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("Default"))
        .EnableRetryOnFailure());

// Authentication & Authorization
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidAudience = builder.Configuration["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]!))
        };
    });

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("AdminOnly", policy => policy.RequireRole("Admin"));
    options.AddPolicy("ManagerOrAbove", policy => 
        policy.RequireRole("Admin", "Manager"));
});

// CORS
builder.Services.AddCors(options =>
{
    options.AddDefaultPolicy(policy =>
    {
        policy.WithOrigins(builder.Configuration.GetSection("AllowedOrigins").Get<string[]>() ?? [])
            .AllowAnyMethod()
            .AllowAnyHeader()
            .AllowCredentials();
    });
});

// Rate Limiting (.NET 7+)
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("api", opt =>
    {
        opt.PermitLimit = 100;
        opt.Window = TimeSpan.FromMinutes(1);
        opt.QueueLimit = 10;
    });
});

// Response Compression
builder.Services.AddResponseCompression(options =>
{
    options.EnableForHttps = true;
});

// Health Checks
builder.Services.AddHealthChecks()
    .AddSqlServer(builder.Configuration.GetConnectionString("Default")!)
    .AddRedis(builder.Configuration.GetConnectionString("Redis") ?? "")
    .AddCheck<ExternalApiHealthCheck>("external-api");

// Application Services (DI)
builder.Services.AddInfrastructure(builder.Configuration);
builder.Services.AddApplication();

var app = builder.Build();

// === Middleware Pipeline ===
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI(c =>
    {
        c.SwaggerEndpoint("/swagger/v1/swagger.json", "My API v1");
        c.EnableTryItOutByDefault();
    });
    
    // Seed data in development
    await SeedData.InitializeAsync(app.Services);
}

app.UseHttpsRedirection();
app.UseResponseCompression();
app.UseSerilogRequestLogging();
app.UseCors();
app.UseRateLimiter();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
app.MapHealthChecks("/health");

await app.RunAsync();
```

## ขั้นตอนที่ 652: Controllers - การสร้าง REST Endpoint

```csharp
// Controllers/ProductsController.cs
[ApiController]
[Route("api/v1/[controller]")]
[Produces("application/json")]
public class ProductsController : ControllerBase
{
    private readonly IProductService _service;
    private readonly IMapper _mapper;
    private readonly ILogger<ProductsController> _logger;
    
    public ProductsController(
        IProductService service,
        IMapper mapper,
        ILogger<ProductsController> logger)
    {
        _service = service;
        _mapper = mapper;
        _logger = logger;
    }
    
    /// <summary>Get all products with optional filtering</summary>
    /// <param name="category">Filter by category</param>
    /// <param name="minPrice">Minimum price filter</param>
    /// <param name="maxPrice">Maximum price filter</param>
    /// <param name="page">Page number (1-based)</param>
    /// <param name="pageSize">Items per page (max 100)</param>
    /// <response code="200">Returns paginated list of products</response>
    [HttpGet]
    [ProducesResponseType(typeof(PagedResponse<ProductDto>), 200)]
    public async Task<ActionResult<PagedResponse<ProductDto>>> GetAll(
        [FromQuery] string? category = null,
        [FromQuery] decimal? minPrice = null,
        [FromQuery] decimal? maxPrice = null,
        [FromQuery] int page = 1,
        [FromQuery] int pageSize = 20)
    {
        pageSize = Math.Min(pageSize, 100); // limit max page size
        
        var result = await _service.GetPagedAsync(
            new ProductFilter(category, minPrice, maxPrice),
            page, pageSize);
        
        return Ok(new PagedResponse<ProductDto>(
            Items: _mapper.Map<List<ProductDto>>(result.Items),
            Total: result.Total,
            Page: result.Page,
            PageSize: result.PageSize));
    }
    
    /// <summary>Get product by ID</summary>
    /// <param name="id">Product ID</param>
    /// <response code="200">Product found</response>
    /// <response code="404">Product not found</response>
    [HttpGet("{id:int}")]
    [ProducesResponseType(typeof(ProductDetailDto), 200)]
    [ProducesResponseType(typeof(ProblemDetails), 404)]
    public async Task<ActionResult<ProductDetailDto>> GetById(int id)
    {
        var product = await _service.GetByIdAsync(id);
        
        if (product is null)
            return NotFound(new ProblemDetails
            {
                Title = "Product Not Found",
                Detail = $"Product with ID {id} was not found",
                Status = 404,
                Instance = HttpContext.Request.Path
            });
        
        return Ok(_mapper.Map<ProductDetailDto>(product));
    }
    
    /// <summary>Create a new product</summary>
    /// <response code="201">Product created successfully</response>
    /// <response code="400">Invalid request data</response>
    [HttpPost]
    [Authorize(Roles = "Admin,Manager")]
    [ProducesResponseType(typeof(ProductDto), 201)]
    [ProducesResponseType(typeof(ValidationProblemDetails), 400)]
    public async Task<ActionResult<ProductDto>> Create(
        [FromBody] CreateProductRequest request)
    {
        var product = await _service.CreateAsync(_mapper.Map<Product>(request));
        var dto = _mapper.Map<ProductDto>(product);
        
        return CreatedAtAction(
            nameof(GetById),
            new { id = product.Id },
            dto);
    }
    
    /// <summary>Update an existing product</summary>
    [HttpPut("{id:int}")]
    [Authorize(Roles = "Admin,Manager")]
    [ProducesResponseType(typeof(ProductDto), 200)]
    [ProducesResponseType(404)]
    [ProducesResponseType(typeof(ValidationProblemDetails), 400)]
    public async Task<ActionResult<ProductDto>> Update(
        int id,
        [FromBody] UpdateProductRequest request)
    {
        var product = await _service.GetByIdAsync(id);
        if (product is null) return NotFound();
        
        _mapper.Map(request, product);
        await _service.UpdateAsync(product);
        
        return Ok(_mapper.Map<ProductDto>(product));
    }
    
    /// <summary>Partially update a product</summary>
    [HttpPatch("{id:int}")]
    [Authorize(Roles = "Admin,Manager")]
    public async Task<ActionResult<ProductDto>> Patch(
        int id,
        [FromBody] JsonPatchDocument<UpdateProductRequest> patchDoc)
    {
        var product = await _service.GetByIdAsync(id);
        if (product is null) return NotFound();
        
        var dto = _mapper.Map<UpdateProductRequest>(product);
        patchDoc.ApplyTo(dto, ModelState);
        
        if (!ModelState.IsValid) return BadRequest(ModelState);
        
        _mapper.Map(dto, product);
        await _service.UpdateAsync(product);
        
        return Ok(_mapper.Map<ProductDto>(product));
    }
    
    /// <summary>Delete a product</summary>
    [HttpDelete("{id:int}")]
    [Authorize(Roles = "Admin")]
    [ProducesResponseType(204)]
    [ProducesResponseType(404)]
    public async Task<IActionResult> Delete(int id)
    {
        var exists = await _service.ExistsAsync(id);
        if (!exists) return NotFound();
        
        await _service.DeleteAsync(id);
        return NoContent();
    }
    
    /// <summary>Upload product image</summary>
    [HttpPost("{id:int}/images")]
    [Consumes("multipart/form-data")]
    public async Task<ActionResult<string>> UploadImage(
        int id,
        [FromForm] IFormFile image)
    {
        if (image.Length > 5 * 1024 * 1024)
            return BadRequest("File size must be less than 5MB");
        
        var allowedTypes = new[] { "image/jpeg", "image/png", "image/webp" };
        if (!allowedTypes.Contains(image.ContentType))
            return BadRequest("Only JPEG, PNG, and WebP images are allowed");
        
        var imageUrl = await _service.UploadImageAsync(id, image);
        return Ok(new { url = imageUrl });
    }
}
```

## ขั้นตอนที่ 653: DTOs และ Mapping

```csharp
// DTOs (Data Transfer Objects)
public record ProductDto(
    int Id,
    string Name,
    string? Description,
    decimal Price,
    int Stock,
    string Category,
    bool IsAvailable);

public record ProductDetailDto(
    int Id,
    string Name,
    string? Description,
    decimal Price,
    int Stock,
    string Category,
    bool IsAvailable,
    List<ReviewDto> Reviews,
    DateTime CreatedAt);

public record ReviewDto(int Id, string AuthorName, int Rating, string Comment, DateTime Date);

// Request models
public class CreateProductRequest
{
    [Required, MinLength(3), MaxLength(100)]
    public string Name { get; set; } = "";
    
    [MaxLength(500)]
    public string? Description { get; set; }
    
    [Range(0.01, 9999999)]
    public decimal Price { get; set; }
    
    [Range(0, 100000)]
    public int Stock { get; set; }
    
    [Required, MaxLength(50)]
    public string Category { get; set; } = "";
}

public class UpdateProductRequest
{
    [MinLength(3), MaxLength(100)]
    public string? Name { get; set; }
    
    [MaxLength(500)]
    public string? Description { get; set; }
    
    [Range(0.01, 9999999)]
    public decimal? Price { get; set; }
    
    [Range(0, 100000)]
    public int? Stock { get; set; }
}

// FluentValidation (more powerful than DataAnnotations)
public class CreateProductValidator : AbstractValidator<CreateProductRequest>
{
    private readonly IProductRepository _repo;
    
    public CreateProductValidator(IProductRepository repo)
    {
        _repo = repo;
        
        RuleFor(x => x.Name)
            .NotEmpty()
            .MinimumLength(3)
            .MaximumLength(100)
            .MustAsync(async (name, ct) => !await _repo.ExistsByNameAsync(name, ct))
            .WithMessage("Product with this name already exists");
        
        RuleFor(x => x.Price)
            .GreaterThan(0)
            .LessThanOrEqualTo(999999.99m);
        
        RuleFor(x => x.Category)
            .NotEmpty()
            .Must(c => ProductCategories.IsValid(c))
            .WithMessage("Invalid category");
    }
}

// AutoMapper Profile
public class ProductMappingProfile : Profile
{
    public ProductMappingProfile()
    {
        CreateMap<Product, ProductDto>();
        CreateMap<Product, ProductDetailDto>()
            .ForMember(d => d.Reviews, opt => opt.MapFrom(s => s.Reviews));
        
        CreateMap<CreateProductRequest, Product>()
            .ForMember(d => d.IsAvailable, opt => opt.MapFrom(_ => true));
        
        CreateMap<UpdateProductRequest, Product>()
            .ForAllMembers(opt => opt.Condition((src, dest, value) => value is not null));
        
        CreateMap<Review, ReviewDto>()
            .ForMember(d => d.AuthorName, opt => opt.MapFrom(s => s.Author.Name));
    }
}

// Response wrapper
public record PagedResponse<T>(
    List<T> Items,
    int Total,
    int Page,
    int PageSize)
{
    public int TotalPages => (int)Math.Ceiling((double)Total / PageSize);
    public bool HasNext => Page < TotalPages;
    public bool HasPrevious => Page > 1;
    public string? NextPageUrl { get; init; }
    public string? PreviousPageUrl { get; init; }
}

public record ApiResponse<T>(bool Success, T? Data, string? Message = null, List<string>? Errors = null)
{
    public static ApiResponse<T> Ok(T data) => new(true, data);
    public static ApiResponse<T> Fail(string message) => new(false, default, message);
    public static ApiResponse<T> ValidationFail(List<string> errors) => new(false, default, "Validation failed", errors);
}
```

## ขั้นตอนที่ 654: Middleware

```csharp
// Custom Exception Middleware
public class ExceptionMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionMiddleware> _logger;
    
    public ExceptionMiddleware(RequestDelegate next, ILogger<ExceptionMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (NotFoundException ex)
        {
            _logger.LogWarning("Not found: {Message}", ex.Message);
            await WriteErrorAsync(context, 404, ex.Message);
        }
        catch (ValidationException ex)
        {
            _logger.LogWarning("Validation error: {Message}", ex.Message);
            await WriteValidationErrorAsync(context, ex.Errors);
        }
        catch (UnauthorizedException ex)
        {
            await WriteErrorAsync(context, 401, ex.Message);
        }
        catch (ForbiddenException ex)
        {
            await WriteErrorAsync(context, 403, ex.Message);
        }
        catch (ConflictException ex)
        {
            await WriteErrorAsync(context, 409, ex.Message);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unhandled exception");
            await WriteErrorAsync(context, 500, "An internal server error occurred");
        }
    }
    
    private static async Task WriteErrorAsync(HttpContext ctx, int status, string message)
    {
        ctx.Response.StatusCode = status;
        ctx.Response.ContentType = "application/json";
        
        var problem = new ProblemDetails
        {
            Status = status,
            Title = GetTitle(status),
            Detail = message,
            Instance = ctx.Request.Path
        };
        
        await ctx.Response.WriteAsJsonAsync(problem);
    }
    
    private static async Task WriteValidationErrorAsync(
        HttpContext ctx, 
        IEnumerable<FluentValidation.Results.ValidationFailure> errors)
    {
        ctx.Response.StatusCode = 400;
        ctx.Response.ContentType = "application/json";
        
        var problem = new ValidationProblemDetails(
            errors.GroupBy(e => e.PropertyName)
                .ToDictionary(g => g.Key, g => g.Select(e => e.ErrorMessage).ToArray()))
        {
            Status = 400,
            Title = "Validation failed",
            Instance = ctx.Request.Path
        };
        
        await ctx.Response.WriteAsJsonAsync(problem);
    }
    
    private static string GetTitle(int status) => status switch
    {
        400 => "Bad Request",
        401 => "Unauthorized",
        403 => "Forbidden",
        404 => "Not Found",
        409 => "Conflict",
        500 => "Internal Server Error",
        _ => "Error"
    };
}

// Request/Response Logging Middleware
public class RequestLoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestLoggingMiddleware> _logger;
    
    public RequestLoggingMiddleware(RequestDelegate next, ILogger<RequestLoggingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        var requestId = Guid.NewGuid().ToString("N")[..8];
        context.Items["RequestId"] = requestId;
        
        var sw = System.Diagnostics.Stopwatch.StartNew();
        
        _logger.LogInformation("[{RequestId}] {Method} {Path}",
            requestId,
            context.Request.Method,
            context.Request.Path);
        
        await _next(context);
        
        sw.Stop();
        _logger.LogInformation("[{RequestId}] {StatusCode} in {ElapsedMs}ms",
            requestId,
            context.Response.StatusCode,
            sw.ElapsedMilliseconds);
    }
}

// Registration in Program.cs
app.UseMiddleware<RequestLoggingMiddleware>();
app.UseMiddleware<ExceptionMiddleware>();
```

## ขั้นตอนที่ 655: JWT Authentication

```csharp
// JWT Token Service
public interface IJwtTokenService
{
    string GenerateToken(UserClaims claims);
    string GenerateRefreshToken();
    ClaimsPrincipal? ValidateToken(string token);
}

public record UserClaims(
    string UserId,
    string Email,
    string Name,
    IEnumerable<string> Roles);

public class JwtTokenService : IJwtTokenService
{
    private readonly JwtOptions _options;
    
    public JwtTokenService(IOptions<JwtOptions> options)
    {
        _options = options.Value;
    }
    
    public string GenerateToken(UserClaims claims)
    {
        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_options.Key));
        var credentials = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);
        
        var claimsList = new List<Claim>
        {
            new(ClaimTypes.NameIdentifier, claims.UserId),
            new(ClaimTypes.Email, claims.Email),
            new(ClaimTypes.Name, claims.Name),
            new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
            new(JwtRegisteredClaimNames.Iat, 
                DateTimeOffset.UtcNow.ToUnixTimeSeconds().ToString(), 
                ClaimValueTypes.Integer64)
        };
        
        claimsList.AddRange(claims.Roles.Select(r => new Claim(ClaimTypes.Role, r)));
        
        var token = new JwtSecurityToken(
            issuer: _options.Issuer,
            audience: _options.Audience,
            claims: claimsList,
            expires: DateTime.UtcNow.AddMinutes(_options.ExpiryMinutes),
            signingCredentials: credentials);
        
        return new JwtSecurityTokenHandler().WriteToken(token);
    }
    
    public string GenerateRefreshToken()
        => Convert.ToBase64String(System.Security.Cryptography.RandomNumberGenerator.GetBytes(64));
    
    public ClaimsPrincipal? ValidateToken(string token)
    {
        var handler = new JwtSecurityTokenHandler();
        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_options.Key));
        
        try
        {
            var principal = handler.ValidateToken(token, new TokenValidationParameters
            {
                ValidateIssuer = true,
                ValidateAudience = true,
                ValidateLifetime = false, // ไม่ตรวจ expiry สำหรับ refresh
                ValidateIssuerSigningKey = true,
                ValidIssuer = _options.Issuer,
                ValidAudience = _options.Audience,
                IssuerSigningKey = key
            }, out _);
            
            return principal;
        }
        catch
        {
            return null;
        }
    }
}

public class JwtOptions
{
    public string Key { get; set; } = "";
    public string Issuer { get; set; } = "";
    public string Audience { get; set; } = "";
    public int ExpiryMinutes { get; set; } = 60;
    public int RefreshExpiryDays { get; set; } = 7;
}

// Auth Controller
[ApiController]
[Route("api/v1/[controller]")]
public class AuthController : ControllerBase
{
    private readonly IUserService _userService;
    private readonly IJwtTokenService _jwtService;
    private readonly ILogger<AuthController> _logger;
    
    public AuthController(
        IUserService userService,
        IJwtTokenService jwtService,
        ILogger<AuthController> logger)
    {
        _userService = userService;
        _jwtService = jwtService;
        _logger = logger;
    }
    
    [HttpPost("login")]
    [AllowAnonymous]
    public async Task<ActionResult<LoginResponse>> Login([FromBody] LoginRequest request)
    {
        var user = await _userService.AuthenticateAsync(request.Email, request.Password);
        
        if (user is null)
            return Unauthorized(new ProblemDetails
            {
                Title = "Authentication Failed",
                Detail = "Invalid email or password"
            });
        
        var claims = new UserClaims(
            user.Id.ToString(),
            user.Email,
            user.Name,
            user.Roles.Select(r => r.Name));
        
        var accessToken = _jwtService.GenerateToken(claims);
        var refreshToken = _jwtService.GenerateRefreshToken();
        
        await _userService.SaveRefreshTokenAsync(user.Id, refreshToken);
        
        _logger.LogInformation("User {Email} logged in", request.Email);
        
        return Ok(new LoginResponse(
            AccessToken: accessToken,
            RefreshToken: refreshToken,
            ExpiresIn: 3600,
            User: new UserDto(user.Id, user.Name, user.Email, user.Roles.Select(r => r.Name).ToList())));
    }
    
    [HttpPost("refresh")]
    [AllowAnonymous]
    public async Task<ActionResult<LoginResponse>> Refresh([FromBody] RefreshRequest request)
    {
        var principal = _jwtService.ValidateToken(request.AccessToken);
        if (principal is null)
            return Unauthorized("Invalid access token");
        
        var userId = int.Parse(principal.FindFirstValue(ClaimTypes.NameIdentifier)!);
        var user = await _userService.GetByIdAsync(userId);
        
        if (user is null || !await _userService.ValidateRefreshTokenAsync(userId, request.RefreshToken))
            return Unauthorized("Invalid refresh token");
        
        // Revoke old refresh token
        await _userService.RevokeRefreshTokenAsync(userId, request.RefreshToken);
        
        var claims = new UserClaims(
            user.Id.ToString(),
            user.Email,
            user.Name,
            user.Roles.Select(r => r.Name));
        
        return Ok(new LoginResponse(
            AccessToken: _jwtService.GenerateToken(claims),
            RefreshToken: _jwtService.GenerateRefreshToken(),
            ExpiresIn: 3600,
            User: new UserDto(user.Id, user.Name, user.Email, [])));
    }
    
    [HttpPost("logout")]
    [Authorize]
    public async Task<IActionResult> Logout([FromBody] LogoutRequest request)
    {
        var userId = int.Parse(User.FindFirstValue(ClaimTypes.NameIdentifier)!);
        await _userService.RevokeRefreshTokenAsync(userId, request.RefreshToken);
        return NoContent();
    }
    
    [HttpGet("me")]
    [Authorize]
    public async Task<ActionResult<UserDto>> GetMe()
    {
        var userId = int.Parse(User.FindFirstValue(ClaimTypes.NameIdentifier)!);
        var user = await _userService.GetByIdAsync(userId);
        
        if (user is null) return NotFound();
        
        return Ok(new UserDto(user.Id, user.Name, user.Email, 
            user.Roles.Select(r => r.Name).ToList()));
    }
}

public record LoginRequest([Required, EmailAddress] string Email, [Required] string Password);
public record LoginResponse(string AccessToken, string RefreshToken, int ExpiresIn, UserDto User);
public record RefreshRequest(string AccessToken, string RefreshToken);
public record LogoutRequest(string RefreshToken);
public record UserDto(int Id, string Name, string Email, List<string> Roles);
```

## ขั้นตอนที่ 656: Minimal APIs (.NET 6+)

```csharp
// Program.cs - Minimal API (ไม่ใช้ Controllers)
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
builder.Services.AddApplication();

var app = builder.Build();

app.UseSwagger();
app.UseSwaggerUI();

// Group endpoints
var api = app.MapGroup("/api/v1")
    .RequireAuthorization();

// Products
var products = api.MapGroup("/products")
    .WithTags("Products")
    .WithOpenApi();

products.MapGet("/", async (
    IProductService service,
    IMapper mapper,
    [FromQuery] string? category,
    [FromQuery] int page = 1,
    [FromQuery] int pageSize = 20) =>
{
    var result = await service.GetPagedAsync(category, page, pageSize);
    return Results.Ok(mapper.Map<PagedResponse<ProductDto>>(result));
})
.WithName("GetProducts")
.WithSummary("Get all products")
.Produces<PagedResponse<ProductDto>>();

products.MapGet("/{id:int}", async (int id, IProductService service, IMapper mapper) =>
{
    var product = await service.GetByIdAsync(id);
    return product is null
        ? Results.NotFound()
        : Results.Ok(mapper.Map<ProductDetailDto>(product));
})
.WithName("GetProductById")
.Produces<ProductDetailDto>()
.Produces(404);

products.MapPost("/", async (
    CreateProductRequest request,
    IProductService service,
    IMapper mapper,
    IValidator<CreateProductRequest> validator) =>
{
    var validation = await validator.ValidateAsync(request);
    if (!validation.IsValid)
        return Results.ValidationProblem(validation.ToDictionary());
    
    var product = await service.CreateAsync(mapper.Map<Product>(request));
    return Results.CreatedAtRoute("GetProductById", new { id = product.Id }, mapper.Map<ProductDto>(product));
})
.RequireAuthorization("AdminOnly")
.WithName("CreateProduct")
.Produces<ProductDto>(201)
.ProducesValidationProblem();

products.MapPut("/{id:int}", async (
    int id,
    UpdateProductRequest request,
    IProductService service) =>
{
    var product = await service.GetByIdAsync(id);
    if (product is null) return Results.NotFound();
    
    product.Name = request.Name ?? product.Name;
    product.Price = request.Price ?? product.Price;
    await service.UpdateAsync(product);
    
    return Results.Ok(product);
})
.RequireAuthorization("ManagerOrAbove");

products.MapDelete("/{id:int}", async (int id, IProductService service) =>
{
    if (!await service.ExistsAsync(id)) return Results.NotFound();
    await service.DeleteAsync(id);
    return Results.NoContent();
})
.RequireAuthorization("AdminOnly");

// Typed results (better type safety)
products.MapGet("/{id:int}/stock", async Task<Results<Ok<StockDto>, NotFound>> (
    int id, IProductService service) =>
{
    var product = await service.GetByIdAsync(id);
    return product is null
        ? TypedResults.NotFound()
        : TypedResults.Ok(new StockDto(product.Id, product.Stock));
});

await app.RunAsync();
```

## ขั้นตอนที่ 657: Action Filters

```csharp
// Action Filters: Logic that runs before/after action methods

// Validation filter
public class ValidationFilter : IActionFilter
{
    public void OnActionExecuting(ActionExecutingContext context)
    {
        if (!context.ModelState.IsValid)
        {
            var errors = context.ModelState
                .Where(kv => kv.Value?.Errors.Any() == true)
                .ToDictionary(
                    kv => kv.Key,
                    kv => kv.Value!.Errors.Select(e => e.ErrorMessage).ToArray());
            
            context.Result = new BadRequestObjectResult(
                new ValidationProblemDetails(errors));
        }
    }
    
    public void OnActionExecuted(ActionExecutedContext context) { }
}

// Cache Filter
[AttributeUsage(AttributeTargets.Method | AttributeTargets.Class)]
public class CacheResponseAttribute : ActionFilterAttribute
{
    private readonly int _durationSeconds;
    
    public CacheResponseAttribute(int durationSeconds = 60)
    {
        _durationSeconds = durationSeconds;
    }
    
    public override async Task OnActionExecutionAsync(
        ActionExecutingContext context, 
        ActionExecutionDelegate next)
    {
        var cache = context.HttpContext.RequestServices
            .GetRequiredService<IDistributedCache>();
        
        var cacheKey = GenerateCacheKey(context.HttpContext.Request);
        var cached = await cache.GetStringAsync(cacheKey);
        
        if (cached is not null)
        {
            context.HttpContext.Response.Headers["X-Cache"] = "HIT";
            context.Result = new ContentResult
            {
                Content = cached,
                ContentType = "application/json",
                StatusCode = 200
            };
            return;
        }
        
        var executed = await next();
        
        if (executed.Result is ObjectResult { StatusCode: 200 or null } okResult)
        {
            var json = JsonSerializer.Serialize(okResult.Value);
            await cache.SetStringAsync(cacheKey, json, new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromSeconds(_durationSeconds)
            });
            context.HttpContext.Response.Headers["X-Cache"] = "MISS";
        }
    }
    
    private static string GenerateCacheKey(HttpRequest request)
        => $"api:{request.Path}:{string.Join("&", request.Query.Select(q => $"{q.Key}={q.Value}"))}";
}

// Authorization filter
public class RequireAdminFilter : IAuthorizationFilter
{
    public void OnAuthorization(AuthorizationFilterContext context)
    {
        if (!context.HttpContext.User.IsInRole("Admin"))
        {
            context.Result = new ForbidResult();
        }
    }
}

// การใช้งาน
[HttpGet]
[CacheResponse(300)] // Cache for 5 minutes
public async Task<ActionResult<List<ProductDto>>> GetAll() { /* ... */ }
```

## ขั้นตอนที่ 658: Versioning

```csharp
// dotnet add package Asp.Versioning.Mvc
// dotnet add package Asp.Versioning.Mvc.ApiExplorer

// Registration
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;
    options.ApiVersionReader = ApiVersionReader.Combine(
        new UrlSegmentApiVersionReader(),           // /api/v1/products
        new HeaderApiVersionReader("X-Api-Version"), // Header
        new QueryStringApiVersionReader("api-version")); // ?api-version=1.0
})
.AddApiExplorer(options =>
{
    options.GroupNameFormat = "'v'VVV";
    options.SubstituteApiVersionInUrl = true;
});

// Controllers
[ApiController]
[ApiVersion("1.0")]
[ApiVersion("1.1")]
[Route("api/v{version:apiVersion}/[controller]")]
public class ProductsV1Controller : ControllerBase
{
    [HttpGet]
    [MapToApiVersion("1.0")]
    public async Task<ActionResult<List<ProductDtoV1>>> GetV1() { /* v1 */ }
    
    [HttpGet]
    [MapToApiVersion("1.1")]
    public async Task<ActionResult<List<ProductDtoV11>>> GetV11() { /* v1.1 with extra fields */ }
}

[ApiController]
[ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/products")]
public class ProductsV2Controller : ControllerBase
{
    // Completely new implementation
    [HttpGet]
    public async Task<ActionResult<PagedResponse<ProductDtoV2>>> Get(
        [FromQuery] ProductFilterV2 filter) { /* v2 */ }
}
```

## ขั้นตอนที่ 659: Health Checks

```csharp
// dotnet add package AspNetCore.HealthChecks.SqlServer
// dotnet add package AspNetCore.HealthChecks.Redis
// dotnet add package AspNetCore.HealthChecks.Uris

builder.Services.AddHealthChecks()
    .AddSqlServer(
        connectionString: builder.Configuration.GetConnectionString("Default")!,
        name: "database",
        failureStatus: HealthStatus.Unhealthy,
        tags: ["db", "sql"])
    .AddRedis(
        builder.Configuration.GetConnectionString("Redis")!,
        name: "cache",
        tags: ["cache"])
    .AddUrlGroup(
        new Uri("https://api.stripe.com/v1"),
        name: "stripe",
        tags: ["payment"])
    .AddCheck<DiskSpaceHealthCheck>("disk-space", tags: ["storage"]);

// Custom health check
public class DiskSpaceHealthCheck : IHealthCheck
{
    private readonly long _minimumBytes = 500 * 1024 * 1024; // 500MB
    
    public Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken ct = default)
    {
        var drive = new DriveInfo(Directory.GetCurrentDirectory());
        
        if (drive.AvailableFreeSpace < _minimumBytes)
        {
            return Task.FromResult(HealthCheckResult.Unhealthy(
                $"Low disk space: {drive.AvailableFreeSpace / 1024 / 1024}MB available"));
        }
        
        return Task.FromResult(HealthCheckResult.Healthy(
            $"Disk space OK: {drive.AvailableFreeSpace / 1024 / 1024}MB available"));
    }
}

// Endpoints
app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("db")
});

app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false // Always returns 200
});
```

## ขั้นตอนที่ 660: สรุป ASP.NET Core Web API

```
ASP.NET Core Web API Architecture
═══════════════════════════════════════════════════════════════

Client Request
  ↓
Middleware Pipeline:
  1. HTTPS Redirection
  2. Response Compression
  3. Request Logging
  4. CORS
  5. Rate Limiting
  6. Authentication (JWT)
  7. Authorization
  8. Exception Handling
  ↓
Routing → Controller/Endpoint
  ↓
Action Filters (Validation, Caching)
  ↓
Controller Action / Minimal API Handler
  ↓
Application Service
  ↓
Repository / EF Core
  ↓
Database

HTTP Status Codes Best Practices:
  200 OK          → GET success, PUT/PATCH success
  201 Created     → POST success (with Location header)
  204 No Content  → DELETE success, PUT/PATCH with no body
  400 Bad Request → Validation errors, malformed request
  401 Unauthorized → Not authenticated
  403 Forbidden   → Authenticated but not authorized
  404 Not Found   → Resource doesn't exist
  409 Conflict    → Duplicate resource, optimistic concurrency
  422 Unprocessable → Business rule violation
  429 Too Many Req → Rate limited
  500 Internal    → Unexpected server error

REST API Design:
  GET    /api/products         → list
  GET    /api/products/{id}    → single
  POST   /api/products         → create
  PUT    /api/products/{id}    → full update
  PATCH  /api/products/{id}    → partial update
  DELETE /api/products/{id}    → delete
═══════════════════════════════════════════════════════════════
```

## สรุป Part 24

ใน Part นี้คุณได้เรียนรู้:
- **Program.cs**: Application setup, middleware pipeline
- **Controllers**: REST endpoints, request/response, action results
- **DTOs & Validation**: FluentValidation, AutoMapper
- **Middleware**: Exception handling, request logging
- **JWT Authentication**: token generation, refresh tokens
- **Minimal APIs**: endpoint-based approach
- **Action Filters**: validation, caching
- **API Versioning**: URL segment, header
- **Health Checks**: built-in and custom checks

## แบบฝึกหัด

1. สร้าง complete REST API สำหรับ Todo List (CRUD)
2. เพิ่ม JWT authentication กับ login/register endpoints
3. Implement pagination, filtering, sorting สำหรับ GET /todos
4. สร้าง custom exception middleware ที่ return RFC 7807 ProblemDetails
5. เพิ่ม health check endpoint ที่ตรวจ database connection
