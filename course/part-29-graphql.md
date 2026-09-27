# Part 29: GraphQL with Hot Chocolate (.NET)

## Steps 781-810: GraphQL API Development

---

## Step 781: GraphQL คืออะไร?

GraphQL คือ Query Language สำหรับ API ที่พัฒนาโดย Facebook ช่วยให้ client สามารถ request ข้อมูลที่ต้องการได้อย่างแม่นยำ

### ความแตกต่างจาก REST

```
REST:
GET /api/products          → คืนค่า Product ทั้งหมดรวมถึง fields ที่ไม่ต้องการ
GET /api/products/1        → คืนค่า Product เดียว
GET /api/products/1/reviews → ต้องเรียก endpoint แยก

GraphQL:
query {
  product(id: 1) {
    id
    name
    price
    reviews {
      rating
      comment
    }
  }
}
→ คืนค่าเฉพาะ fields ที่ต้องการในครั้งเดียว
```

### ข้อดีของ GraphQL
- **No Over-fetching**: ได้ข้อมูลแค่ที่ต้องการ
- **No Under-fetching**: ไม่ต้องเรียก API หลายครั้ง
- **Strong Type System**: Schema กำหนด type ชัดเจน
- **Introspection**: Client รู้ schema ได้
- **Real-time**: Subscription รองรับ real-time

---

## Step 782: ติดตั้ง Hot Chocolate

```bash
dotnet new webapi -n GraphQLDemo --framework net9.0
cd GraphQLDemo

# Hot Chocolate packages
dotnet add package HotChocolate.AspNetCore
dotnet add package HotChocolate.Data
dotnet add package HotChocolate.Data.EntityFramework
dotnet add package HotChocolate.AspNetCore.Playground

# Entity Framework
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Design
```

---

## Step 783: Models และ DbContext

```csharp
// Models/Product.cs
namespace GraphQLDemo.Models;

public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Description { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public int StockQuantity { get; set; }
    public bool IsActive { get; set; } = true;
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    
    public int CategoryId { get; set; }
    public Category Category { get; set; } = null!;
    
    public ICollection<Review> Reviews { get; set; } = [];
    public ICollection<OrderItem> OrderItems { get; set; } = [];
}

// Models/Category.cs
public class Category
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Slug { get; set; } = string.Empty;
    
    public ICollection<Product> Products { get; set; } = [];
}

// Models/Review.cs
public class Review
{
    public int Id { get; set; }
    public int Rating { get; set; }
    public string Comment { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    
    public int ProductId { get; set; }
    public Product Product { get; set; } = null!;
    
    public string UserId { get; set; } = string.Empty;
    public User User { get; set; } = null!;
}

// Models/User.cs
public class User
{
    public string Id { get; set; } = Guid.NewGuid().ToString();
    public string Username { get; set; } = string.Empty;
    public string Email { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    
    public ICollection<Review> Reviews { get; set; } = [];
    public ICollection<Order> Orders { get; set; } = [];
}

// Models/Order.cs
public class Order
{
    public int Id { get; set; }
    public string Status { get; set; } = "Pending";
    public decimal TotalAmount { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    
    public string UserId { get; set; } = string.Empty;
    public User User { get; set; } = null!;
    
    public ICollection<OrderItem> Items { get; set; } = [];
}

// Models/OrderItem.cs
public class OrderItem
{
    public int Id { get; set; }
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
    
    public int OrderId { get; set; }
    public Order Order { get; set; } = null!;
    
    public int ProductId { get; set; }
    public Product Product { get; set; } = null!;
}
```

```csharp
// Data/AppDbContext.cs
using Microsoft.EntityFrameworkCore;
using GraphQLDemo.Models;

namespace GraphQLDemo.Data;

public class AppDbContext(DbContextOptions<AppDbContext> options) : DbContext(options)
{
    public DbSet<Product> Products => Set<Product>();
    public DbSet<Category> Categories => Set<Category>();
    public DbSet<Review> Reviews => Set<Review>();
    public DbSet<User> Users => Set<User>();
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<OrderItem> OrderItems => Set<OrderItem>();
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Product>(entity =>
        {
            entity.Property(p => p.Price).HasPrecision(18, 2);
            entity.HasIndex(p => p.Name);
        });
        
        modelBuilder.Entity<Category>(entity =>
        {
            entity.HasIndex(c => c.Slug).IsUnique();
        });
        
        // Seed data
        modelBuilder.Entity<Category>().HasData(
            new Category { Id = 1, Name = "Electronics", Slug = "electronics" },
            new Category { Id = 2, Name = "Books", Slug = "books" },
            new Category { Id = 3, Name = "Clothing", Slug = "clothing" }
        );
        
        modelBuilder.Entity<Product>().HasData(
            new Product { Id = 1, Name = "Laptop Pro", Price = 49999, CategoryId = 1, StockQuantity = 50 },
            new Product { Id = 2, Name = "Clean Code", Price = 599, CategoryId = 2, StockQuantity = 100 },
            new Product { Id = 3, Name = "T-Shirt", Price = 299, CategoryId = 3, StockQuantity = 200 }
        );
    }
}
```

---

## Step 784: Query Type พื้นฐาน

```csharp
// GraphQL/Types/ProductType.cs
using HotChocolate.Data;
using GraphQLDemo.Models;

namespace GraphQLDemo.GraphQL.Types;

public class ProductType : ObjectType<Product>
{
    protected override void Configure(IObjectTypeDescriptor<Product> descriptor)
    {
        descriptor.Description("Represents a product in the catalog");
        
        descriptor
            .Field(p => p.Id)
            .Description("The unique identifier of the product");
        
        descriptor
            .Field(p => p.Name)
            .Description("The name of the product");
        
        descriptor
            .Field(p => p.Price)
            .Description("The price of the product in THB");
        
        // Computed field
        descriptor
            .Field("priceWithVat")
            .Type<DecimalType>()
            .Description("Price including 7% VAT")
            .Resolve(ctx => ctx.Parent<Product>().Price * 1.07m);
        
        // Rename field
        descriptor
            .Field(p => p.StockQuantity)
            .Name("stockQty");
    }
}
```

```csharp
// GraphQL/Queries/ProductQuery.cs
using HotChocolate;
using HotChocolate.Data;
using HotChocolate.Types;
using GraphQLDemo.Data;
using GraphQLDemo.Models;

namespace GraphQLDemo.GraphQL.Queries;

[QueryType]
public class ProductQuery
{
    // ดึง product ทั้งหมดพร้อม filtering, sorting, paging
    [UseDbContext(typeof(AppDbContext))]
    [UsePaging]
    [UseProjection]
    [UseFiltering]
    [UseSorting]
    public IQueryable<Product> GetProducts([ScopedService] AppDbContext db)
        => db.Products.Where(p => p.IsActive);
    
    // ดึง product เดียวด้วย ID
    [UseDbContext(typeof(AppDbContext))]
    [UseFirstOrDefault]
    [UseProjection]
    public IQueryable<Product> GetProductById([ScopedService] AppDbContext db, int id)
        => db.Products.Where(p => p.Id == id);
    
    // ค้นหา product
    [UseDbContext(typeof(AppDbContext))]
    [UseProjection]
    [UseFiltering]
    [UseSorting]
    public IQueryable<Product> SearchProducts(
        [ScopedService] AppDbContext db,
        string searchTerm)
        => db.Products
            .Where(p => p.IsActive && 
                       (p.Name.Contains(searchTerm) || 
                        p.Description.Contains(searchTerm)));
}
```

---

## Step 785: Query Type สำหรับ Category และ User

```csharp
// GraphQL/Queries/CategoryQuery.cs
[QueryType]
public class CategoryQuery
{
    [UseDbContext(typeof(AppDbContext))]
    [UsePaging]
    [UseProjection]
    [UseFiltering]
    [UseSorting]
    public IQueryable<Category> GetCategories([ScopedService] AppDbContext db)
        => db.Categories;
    
    [UseDbContext(typeof(AppDbContext))]
    [UseFirstOrDefault]
    [UseProjection]
    public IQueryable<Category> GetCategoryBySlug(
        [ScopedService] AppDbContext db,
        string slug)
        => db.Categories.Where(c => c.Slug == slug);
}

// GraphQL/Queries/UserQuery.cs
[QueryType]
public class UserQuery
{
    [UseDbContext(typeof(AppDbContext))]
    [UseFirstOrDefault]
    [UseProjection]
    public IQueryable<User> GetMe(
        [ScopedService] AppDbContext db,
        [GlobalState("currentUserId")] string? userId)
    {
        if (userId == null) throw new UnauthorizedAccessException("Not authenticated");
        return db.Users.Where(u => u.Id == userId);
    }
    
    [UseDbContext(typeof(AppDbContext))]
    [UsePaging]
    [UseProjection]
    [UseFiltering]
    [UseSorting]
    [Authorize(Roles = new[] { "Admin" })]
    public IQueryable<User> GetUsers([ScopedService] AppDbContext db)
        => db.Users;
}
```

---

## Step 786: Mutation Type

```csharp
// GraphQL/Mutations/ProductMutation.cs
using HotChocolate;
using HotChocolate.Types;
using GraphQLDemo.Data;
using GraphQLDemo.Models;

namespace GraphQLDemo.GraphQL.Mutations;

[MutationType]
public class ProductMutation
{
    [UseDbContext(typeof(AppDbContext))]
    public async Task<Product> CreateProduct(
        CreateProductInput input,
        [ScopedService] AppDbContext db,
        CancellationToken ct)
    {
        var category = await db.Categories.FindAsync([input.CategoryId], ct)
            ?? throw new GraphQLException(
                ErrorBuilder.New()
                    .SetMessage($"Category {input.CategoryId} not found")
                    .SetCode("CATEGORY_NOT_FOUND")
                    .Build());
        
        var product = new Product
        {
            Name = input.Name,
            Description = input.Description ?? "",
            Price = input.Price,
            StockQuantity = input.StockQuantity,
            CategoryId = input.CategoryId
        };
        
        db.Products.Add(product);
        await db.SaveChangesAsync(ct);
        
        return product;
    }
    
    [UseDbContext(typeof(AppDbContext))]
    public async Task<Product> UpdateProduct(
        UpdateProductInput input,
        [ScopedService] AppDbContext db,
        CancellationToken ct)
    {
        var product = await db.Products.FindAsync([input.Id], ct)
            ?? throw new GraphQLException(
                ErrorBuilder.New()
                    .SetMessage($"Product {input.Id} not found")
                    .SetCode("PRODUCT_NOT_FOUND")
                    .Build());
        
        if (input.Name is not null) product.Name = input.Name;
        if (input.Price is not null) product.Price = input.Price.Value;
        if (input.StockQuantity is not null) product.StockQuantity = input.StockQuantity.Value;
        
        await db.SaveChangesAsync(ct);
        return product;
    }
    
    [UseDbContext(typeof(AppDbContext))]
    public async Task<bool> DeleteProduct(
        int id,
        [ScopedService] AppDbContext db,
        CancellationToken ct)
    {
        var product = await db.Products.FindAsync([id], ct)
            ?? throw new GraphQLException(
                ErrorBuilder.New()
                    .SetMessage($"Product {id} not found")
                    .SetCode("PRODUCT_NOT_FOUND")
                    .Build());
        
        product.IsActive = false; // soft delete
        await db.SaveChangesAsync(ct);
        return true;
    }
}
```

```csharp
// GraphQL/Inputs/ProductInputs.cs
namespace GraphQLDemo.GraphQL.Inputs;

public record CreateProductInput(
    string Name,
    decimal Price,
    int CategoryId,
    int StockQuantity,
    string? Description = null);

public record UpdateProductInput(
    int Id,
    string? Name = null,
    decimal? Price = null,
    int? StockQuantity = null);
```

---

## Step 787: Review Mutations

```csharp
// GraphQL/Mutations/ReviewMutation.cs
[MutationType]
public class ReviewMutation
{
    [UseDbContext(typeof(AppDbContext))]
    [Authorize]
    public async Task<Review> AddReview(
        AddReviewInput input,
        [ScopedService] AppDbContext db,
        [GlobalState("currentUserId")] string userId,
        CancellationToken ct)
    {
        if (input.Rating < 1 || input.Rating > 5)
            throw new GraphQLException(
                ErrorBuilder.New()
                    .SetMessage("Rating must be between 1 and 5")
                    .SetCode("INVALID_RATING")
                    .Build());
        
        var product = await db.Products.FindAsync([input.ProductId], ct)
            ?? throw new GraphQLException("Product not found");
        
        var existing = db.Reviews
            .FirstOrDefault(r => r.ProductId == input.ProductId && r.UserId == userId);
        
        if (existing != null)
            throw new GraphQLException("You have already reviewed this product");
        
        var review = new Review
        {
            ProductId = input.ProductId,
            Rating = input.Rating,
            Comment = input.Comment,
            UserId = userId
        };
        
        db.Reviews.Add(review);
        await db.SaveChangesAsync(ct);
        return review;
    }
}

public record AddReviewInput(int ProductId, int Rating, string Comment);
```

---

## Step 788: Subscription Type (Real-time)

```csharp
// GraphQL/Subscriptions/ProductSubscription.cs
using HotChocolate;
using HotChocolate.Subscriptions;
using HotChocolate.Types;
using GraphQLDemo.Models;

namespace GraphQLDemo.GraphQL.Subscriptions;

[SubscriptionType]
public class ProductSubscription
{
    // Subscribe to stock changes for a specific product
    [Subscribe]
    [Topic("{productId}")]
    public Product OnStockChanged(
        int productId,
        [EventMessage] Product product)
        => product;
    
    // Subscribe to all new products
    [Subscribe]
    [Topic("new-product")]
    public Product OnProductCreated([EventMessage] Product product)
        => product;
    
    // Subscribe to order status changes
    [Subscribe]
    [Topic("order-{orderId}")]
    public OrderStatusChanged OnOrderStatusChanged(
        int orderId,
        [EventMessage] OrderStatusChanged status)
        => status;
}

public record OrderStatusChanged(int OrderId, string OldStatus, string NewStatus);
```

```csharp
// Publishing events from services
public class ProductService(
    AppDbContext db,
    ITopicEventSender eventSender)
{
    public async Task UpdateStockAsync(int productId, int quantity, CancellationToken ct)
    {
        var product = await db.Products.FindAsync([productId], ct)
            ?? throw new KeyNotFoundException($"Product {productId} not found");
        
        product.StockQuantity = quantity;
        await db.SaveChangesAsync(ct);
        
        // Publish event to subscribers
        await eventSender.SendAsync($"{productId}", product, ct);
    }
    
    public async Task<Product> CreateProductAsync(Product product, CancellationToken ct)
    {
        db.Products.Add(product);
        await db.SaveChangesAsync(ct);
        
        // Notify new product subscribers
        await eventSender.SendAsync("new-product", product, ct);
        
        return product;
    }
}
```

---

## Step 789: DataLoader Pattern

DataLoader แก้ปัญหา N+1 query โดยการ batch requests เข้าด้วยกัน

```csharp
// GraphQL/DataLoaders/CategoryBatchDataLoader.cs
using GreenDonut;
using GraphQLDemo.Data;
using GraphQLDemo.Models;
using Microsoft.EntityFrameworkCore;

namespace GraphQLDemo.GraphQL.DataLoaders;

public class CategoryBatchDataLoader(
    IServiceProvider services,
    IBatchScheduler scheduler,
    DataLoaderOptions options)
    : BatchDataLoader<int, Category>(scheduler, options)
{
    protected override async Task<IReadOnlyDictionary<int, Category>> LoadBatchAsync(
        IReadOnlyList<int> keys,
        CancellationToken ct)
    {
        using var scope = services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        
        return await db.Categories
            .Where(c => keys.Contains(c.Id))
            .ToDictionaryAsync(c => c.Id, ct);
    }
}
```

```csharp
// GraphQL/DataLoaders/ReviewsByProductDataLoader.cs
public class ReviewsByProductDataLoader(
    IServiceProvider services,
    IBatchScheduler scheduler,
    DataLoaderOptions options)
    : GroupedDataLoader<int, Review>(scheduler, options)
{
    protected override async Task<ILookup<int, Review>> LoadGroupedBatchAsync(
        IReadOnlyList<int> keys,
        CancellationToken ct)
    {
        using var scope = services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        
        var reviews = await db.Reviews
            .Where(r => keys.Contains(r.ProductId))
            .ToListAsync(ct);
        
        return reviews.ToLookup(r => r.ProductId);
    }
}
```

```csharp
// ใช้ DataLoader ใน Type
public class ProductType : ObjectType<Product>
{
    protected override void Configure(IObjectTypeDescriptor<Product> descriptor)
    {
        // ใช้ DataLoader แทนการ include directly
        descriptor
            .Field(p => p.Category)
            .ResolveWith<ProductResolvers>(r => r.GetCategoryAsync(default!, default!, default!));
        
        descriptor
            .Field(p => p.Reviews)
            .ResolveWith<ProductResolvers>(r => r.GetReviewsAsync(default!, default!, default!));
    }
}

public class ProductResolvers
{
    public async Task<Category> GetCategoryAsync(
        [Parent] Product product,
        CategoryBatchDataLoader categoryLoader,
        CancellationToken ct)
        => await categoryLoader.LoadAsync(product.CategoryId, ct);
    
    public async Task<IEnumerable<Review>> GetReviewsAsync(
        [Parent] Product product,
        ReviewsByProductDataLoader reviewsLoader,
        CancellationToken ct)
        => await reviewsLoader.LoadAsync(product.Id, ct);
}
```

---

## Step 790: Error Handling

```csharp
// GraphQL/Errors/ProductErrors.cs
namespace GraphQLDemo.GraphQL.Errors;

public static class ProductErrors
{
    public static IError ProductNotFound(int id) =>
        ErrorBuilder.New()
            .SetMessage($"Product with ID {id} was not found")
            .SetCode("PRODUCT_NOT_FOUND")
            .SetExtension("productId", id)
            .Build();
    
    public static IError InsufficientStock(int productId, int requested, int available) =>
        ErrorBuilder.New()
            .SetMessage($"Insufficient stock for product {productId}")
            .SetCode("INSUFFICIENT_STOCK")
            .SetExtension("requested", requested)
            .SetExtension("available", available)
            .Build();
}
```

```csharp
// Mutation Error Pattern (Result pattern)
[MutationType]
public class OrderMutation
{
    [UseDbContext(typeof(AppDbContext))]
    [Authorize]
    [Error<ProductNotFoundException>]
    [Error<InsufficientStockException>]
    public async Task<CreateOrderPayload> CreateOrder(
        CreateOrderInput input,
        [ScopedService] AppDbContext db,
        [GlobalState("currentUserId")] string userId,
        CancellationToken ct)
    {
        var items = new List<OrderItem>();
        
        foreach (var item in input.Items)
        {
            var product = await db.Products.FindAsync([item.ProductId], ct)
                ?? throw new ProductNotFoundException(item.ProductId);
            
            if (product.StockQuantity < item.Quantity)
                throw new InsufficientStockException(
                    item.ProductId, item.Quantity, product.StockQuantity);
            
            items.Add(new OrderItem
            {
                ProductId = item.ProductId,
                Quantity = item.Quantity,
                UnitPrice = product.Price
            });
            
            product.StockQuantity -= item.Quantity;
        }
        
        var order = new Order
        {
            UserId = userId,
            Items = items,
            TotalAmount = items.Sum(i => i.UnitPrice * i.Quantity)
        };
        
        db.Orders.Add(order);
        await db.SaveChangesAsync(ct);
        
        return new CreateOrderPayload(order);
    }
}

public record CreateOrderPayload(Order Order);
public record OrderItemInput(int ProductId, int Quantity);
public record CreateOrderInput(IReadOnlyList<OrderItemInput> Items);

public class ProductNotFoundException(int id) 
    : Exception($"Product {id} not found");
public class InsufficientStockException(int productId, int requested, int available)
    : Exception($"Product {productId}: need {requested}, have {available}");
```

---

## Step 791: Authentication และ Authorization

```csharp
// GraphQL/Middleware/AuthMiddleware.cs
using HotChocolate.AspNetCore;
using HotChocolate.AspNetCore.Serialization;

namespace GraphQLDemo.GraphQL.Middleware;

public class AuthMiddleware(RequestDelegate next)
{
    public async Task InvokeAsync(HttpContext context, IHttpRequestInterceptor interceptor)
    {
        await next(context);
    }
}

// Custom HTTP Request Interceptor
public class CustomHttpRequestInterceptor : DefaultHttpRequestInterceptor
{
    public override async ValueTask OnCreateAsync(
        HttpContext context,
        IRequestExecutor requestExecutor,
        IQueryRequestBuilder requestBuilder,
        CancellationToken ct)
    {
        await base.OnCreateAsync(context, requestExecutor, requestBuilder, ct);
        
        // Extract user ID from JWT and add to global state
        if (context.User.Identity?.IsAuthenticated == true)
        {
            var userId = context.User.FindFirst("sub")?.Value
                ?? context.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
            
            if (userId != null)
            {
                requestBuilder.SetGlobalState("currentUserId", userId);
            }
            
            var roles = context.User.Claims
                .Where(c => c.Type == ClaimTypes.Role)
                .Select(c => c.Value)
                .ToArray();
            
            requestBuilder.SetGlobalState("currentUserRoles", roles);
        }
    }
}
```

```csharp
// Using Authorization in resolvers
[QueryType]
public class AdminQuery
{
    [Authorize(Policy = "RequireAdminRole")]
    [UseDbContext(typeof(AppDbContext))]
    [UsePaging]
    [UseProjection]
    public IQueryable<Order> GetAllOrders([ScopedService] AppDbContext db)
        => db.Orders.OrderByDescending(o => o.CreatedAt);
    
    // Field-level authorization
    [UseDbContext(typeof(AppDbContext))]
    public IQueryable<Product> GetProducts([ScopedService] AppDbContext db)
        => db.Products;
}

public class ProductType : ObjectType<Product>
{
    protected override void Configure(IObjectTypeDescriptor<Product> descriptor)
    {
        // Admin-only field
        descriptor
            .Field("costPrice")
            .Type<DecimalType>()
            .Resolve(ctx => 
            {
                var user = ctx.GetGlobalStateOrDefault<ClaimsPrincipal>("user");
                if (user?.IsInRole("Admin") != true)
                    throw new UnauthorizedAccessException();
                return ctx.Parent<Product>().Price * 0.6m; // example cost
            });
    }
}
```

---

## Step 792: Filtering, Sorting, Paging

```csharp
// Custom Filter Type
using HotChocolate.Data.Filters;

public class ProductFilterType : FilterInputType<Product>
{
    protected override void Configure(IFilterInputTypeDescriptor<Product> descriptor)
    {
        descriptor.BindFieldsExplicitly();
        
        descriptor.Field(p => p.Name).Type<StringOperationFilterInputType>();
        descriptor.Field(p => p.Price).Type<DecimalOperationFilterInputType>();
        descriptor.Field(p => p.CategoryId).Type<IntOperationFilterInputType>();
        descriptor.Field(p => p.IsActive);
        
        // Custom filter operation
        descriptor
            .Operation(DefaultFilterOperations.Contains)
            .Type<StringType>();
    }
}

// Custom Sort Type
public class ProductSortType : SortInputType<Product>
{
    protected override void Configure(ISortInputTypeDescriptor<Product> descriptor)
    {
        descriptor.BindFieldsExplicitly();
        descriptor.Field(p => p.Name);
        descriptor.Field(p => p.Price);
        descriptor.Field(p => p.CreatedAt);
    }
}

// Cursor-based paging
[QueryType]
public class ProductQuery
{
    [UseDbContext(typeof(AppDbContext))]
    [UsePaging(IncludeTotalCount = true, MaxPageSize = 50)]
    [UseProjection]
    [UseFiltering(typeof(ProductFilterType))]
    [UseSorting(typeof(ProductSortType))]
    public IQueryable<Product> GetProducts([ScopedService] AppDbContext db)
        => db.Products.Where(p => p.IsActive);
    
    // Offset-based paging
    [UseDbContext(typeof(AppDbContext))]
    [UseOffsetPaging(IncludeTotalCount = true)]
    [UseProjection]
    [UseFiltering]
    [UseSorting]
    public IQueryable<Product> GetProductsPage([ScopedService] AppDbContext db)
        => db.Products.Where(p => p.IsActive);
}
```

---

## Step 793: Schema เต็มรูปแบบ

```csharp
// Program.cs
using HotChocolate.AspNetCore;
using GraphQLDemo.Data;
using GraphQLDemo.GraphQL.Queries;
using GraphQLDemo.GraphQL.Mutations;
using GraphQLDemo.GraphQL.Subscriptions;
using GraphQLDemo.GraphQL.DataLoaders;
using GraphQLDemo.GraphQL.Middleware;
using Microsoft.EntityFrameworkCore;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;
using System.Text;

var builder = WebApplication.CreateBuilder(args);

// Database
builder.Services.AddDbContext<AppDbContext>(opts =>
    opts.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

// Authentication
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(opts =>
    {
        opts.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Secret"]!)),
            ValidateIssuer = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidateAudience = false
        };
        
        // Support WebSocket authentication for subscriptions
        opts.Events = new JwtBearerEvents
        {
            OnMessageReceived = ctx =>
            {
                var token = ctx.Request.Query["access_token"];
                if (!string.IsNullOrEmpty(token))
                    ctx.Token = token;
                return Task.CompletedTask;
            }
        };
    });

builder.Services.AddAuthorization(opts =>
{
    opts.AddPolicy("RequireAdminRole", 
        policy => policy.RequireRole("Admin"));
});

// Hot Chocolate GraphQL
builder.Services
    .AddGraphQLServer()
    .AddQueryType()
    .AddTypeExtension<ProductQuery>()
    .AddTypeExtension<CategoryQuery>()
    .AddTypeExtension<UserQuery>()
    .AddMutationType()
    .AddTypeExtension<ProductMutation>()
    .AddTypeExtension<ReviewMutation>()
    .AddTypeExtension<OrderMutation>()
    .AddSubscriptionType()
    .AddTypeExtension<ProductSubscription>()
    .AddType<ProductType>()
    .AddDataLoader<CategoryBatchDataLoader>()
    .AddDataLoader<ReviewsByProductDataLoader>()
    .AddFiltering()
    .AddSorting()
    .AddProjections()
    .AddAuthorization()
    .AddInMemorySubscriptions()
    .ModifyRequestOptions(opts =>
    {
        opts.IncludeExceptionDetails = builder.Environment.IsDevelopment();
    })
    .AddHttpRequestInterceptor<CustomHttpRequestInterceptor>();

// Scoped DbContext support for HotChocolate
builder.Services.AddDbContextFactory<AppDbContext>(
    opts => opts.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();

app.MapGraphQL("/graphql");

// GraphQL Playground in development
if (app.Environment.IsDevelopment())
{
    app.UseRouting();
    app.MapBananaCakePop("/ui");
}

// Apply migrations
using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    await db.Database.MigrateAsync();
}

app.Run();
```

---

## Step 794: Testing GraphQL Queries

```graphql
# ดึง products พร้อม pagination
query GetProducts($first: Int, $after: String) {
  products(first: $first, after: $after) {
    totalCount
    pageInfo {
      hasNextPage
      endCursor
    }
    edges {
      cursor
      node {
        id
        name
        price
        priceWithVat
        category {
          name
          slug
        }
        reviews {
          rating
          comment
        }
      }
    }
  }
}

# Filter และ Sort
query GetFilteredProducts {
  products(
    where: { price: { gte: 100 }, isActive: { eq: true } }
    order: [{ price: ASC }]
    first: 10
  ) {
    edges {
      node {
        id
        name
        price
      }
    }
  }
}

# Mutation
mutation CreateProduct {
  createProduct(input: {
    name: "New Laptop"
    price: 45000
    categoryId: 1
    stockQuantity: 30
    description: "High performance laptop"
  }) {
    id
    name
    price
    category {
      name
    }
  }
}

# Subscription
subscription OnStockChanged {
  onStockChanged(productId: 1) {
    id
    name
    stockQty
  }
}
```

---

## Step 795: Persisted Queries

```csharp
// Program.cs - เพิ่ม persisted queries
builder.Services
    .AddGraphQLServer()
    // ... 
    .UsePersistedQueryPipeline()
    .AddInMemoryQueryStorage();
    
// หรือใช้ Redis สำหรับ production
builder.Services.AddStackExchangeRedisCache(opts =>
    opts.Configuration = builder.Configuration["Redis:ConnectionString"]);

builder.Services
    .AddGraphQLServer()
    .UsePersistedQueryPipeline()
    .AddRedisQueryStorage(sp => 
        sp.GetRequiredService<IConnectionMultiplexer>().GetDatabase());
```

```javascript
// Client ส่ง persisted query hash แทน query string
// ลด network payload

// First request - register query
POST /graphql
{
  "id": "GetProducts-1",
  "query": "query GetProducts { products { edges { node { id name } } } }"
}

// Subsequent requests - use hash only
POST /graphql  
{
  "id": "GetProducts-1"
}
```

---

## Step 796: Introspection และ Schema Documentation

```csharp
// Schema documentation
public class ProductType : ObjectType<Product>
{
    protected override void Configure(IObjectTypeDescriptor<Product> descriptor)
    {
        descriptor.Description("A product available in the store catalog");
        
        descriptor
            .Field(p => p.Id)
            .Description("Unique identifier for the product");
        
        descriptor
            .Field(p => p.Price)
            .Description("Product price in Thai Baht (THB)");
        
        // Deprecate old field
        descriptor
            .Field("oldField")
            .Type<StringType>()
            .Deprecated("Use 'name' instead")
            .Resolve(_ => "");
    }
}
```

```graphql
# Introspection query
query IntrospectSchema {
  __schema {
    types {
      name
      description
      fields {
        name
        description
        type {
          name
          kind
        }
        isDeprecated
        deprecationReason
      }
    }
  }
}

# Check specific type
query CheckProductType {
  __type(name: "Product") {
    name
    fields {
      name
      type {
        name
        kind
        ofType {
          name
          kind
        }
      }
    }
  }
}
```

---

## Step 797: Federation (Microservices)

```csharp
// Product Service - federated schema
builder.Services
    .AddGraphQLServer()
    .AddApolloFederation()
    .AddQueryType<ProductQuery>()
    .AddType<ProductType>();

// Product type with federation key
[ObjectType]
[Key("id")]
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
}

// Reference resolver for federation
[ReferenceResolver]
public static Product? ResolveProductReference(
    int id,
    [ScopedService] AppDbContext db)
    => db.Products.Find(id);
```

```csharp
// Review Service - extending Product from another service
[ExtendObjectType("Product")]
public class ProductExtensions
{
    public async Task<IEnumerable<Review>> GetReviews(
        [Parent] Product product,
        [ScopedService] ReviewDbContext db)
        => await db.Reviews
            .Where(r => r.ProductId == product.Id)
            .ToListAsync();
}

// Register in Review service
builder.Services
    .AddGraphQLServer()
    .AddApolloFederation()
    .AddQueryType<ReviewQuery>()
    .AddTypeExtension<ProductExtensions>();
```

---

## Step 798: Performance Optimization

```csharp
// Query complexity analysis
builder.Services
    .AddGraphQLServer()
    .AddMaxExecutionDepthRule(10)
    .AddQueryComplexityRule(opts =>
    {
        opts.MaximumAllowed = 1000;
        opts.DefaultComplexity = 1;
        opts.DefaultResolverComplexity = 5;
    });

// Query cost rules
public class ProductType : ObjectType<Product>
{
    protected override void Configure(IObjectTypeDescriptor<Product> descriptor)
    {
        // Mark expensive field
        descriptor
            .Field(p => p.Reviews)
            .Cost(10); // counts as 10 in complexity
    }
}

// Response caching
builder.Services
    .AddGraphQLServer()
    .UseQueryCachePipeline() // Cache parsed queries
    .AddHttpCacheControlHeader(); // Add cache-control headers

// Field-level caching
[QueryType]
public class ProductQuery
{
    [UseDbContext(typeof(AppDbContext))]
    [CacheControl(MaxAge = 300)] // Cache for 5 minutes
    public IQueryable<Category> GetCategories([ScopedService] AppDbContext db)
        => db.Categories;
}
```

---

## Step 799: Integration Testing

```csharp
// Tests/GraphQLIntegrationTests.cs
using Microsoft.AspNetCore.Mvc.Testing;
using System.Net.Http.Json;
using Xunit;

public class ProductQueryTests(WebApplicationFactory<Program> factory) 
    : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client = factory.CreateClient();
    
    [Fact]
    public async Task GetProducts_ReturnsProductList()
    {
        var query = new
        {
            query = """
                query {
                  products(first: 5) {
                    totalCount
                    edges {
                      node {
                        id
                        name
                        price
                      }
                    }
                  }
                }
                """
        };
        
        var response = await _client.PostAsJsonAsync("/graphql", query);
        response.EnsureSuccessStatusCode();
        
        var result = await response.Content.ReadFromJsonAsync<JsonDocument>();
        var products = result!.RootElement
            .GetProperty("data")
            .GetProperty("products")
            .GetProperty("edges");
        
        Assert.True(products.GetArrayLength() > 0);
    }
    
    [Fact]
    public async Task CreateProduct_RequiresAuthentication()
    {
        var mutation = new
        {
            query = """
                mutation {
                  createProduct(input: {
                    name: "Test"
                    price: 100
                    categoryId: 1
                    stockQuantity: 10
                  }) { id }
                }
                """
        };
        
        var response = await _client.PostAsJsonAsync("/graphql", mutation);
        var result = await response.Content.ReadFromJsonAsync<JsonDocument>();
        
        var errors = result!.RootElement.GetProperty("errors");
        Assert.True(errors.GetArrayLength() > 0);
    }
}
```

---

## Step 800: Hot Chocolate Client (StrawberryShake)

```bash
# ติดตั้ง StrawberryShake
dotnet add package StrawberryShake.Transport.Http
dotnet add package StrawberryShake.CodeGeneration.CSharp.Analyzers

# เพิ่ม .graphqlrc.json
```

```json
{
  "schema": "http://localhost:5000/graphql?sdl",
  "documents": "**/*.graphql"
}
```

```graphql
# Queries/GetProducts.graphql
query GetProducts($first: Int = 10) {
  products(first: $first) {
    edges {
      node {
        id
        name
        price
        category {
          name
        }
      }
    }
  }
}

mutation CreateProduct($input: CreateProductInput!) {
  createProduct(input: $input) {
    id
    name
  }
}
```

```csharp
// Auto-generated strongly typed client usage
public class ProductService(IProductsClient client)
{
    public async Task<IReadOnlyList<IGetProducts_Products_Edges_Node>> GetProductsAsync()
    {
        var result = await client.GetProducts.ExecuteAsync();
        result.EnsureNoErrors();
        
        return result.Data!.Products!.Edges!
            .Select(e => e!.Node!)
            .ToList();
    }
    
    public async Task<int> CreateProductAsync(string name, decimal price, int categoryId)
    {
        var result = await client.CreateProduct.ExecuteAsync(new CreateProductInput
        {
            Name = name,
            Price = price,
            CategoryId = categoryId,
            StockQuantity = 0
        });
        
        result.EnsureNoErrors();
        return result.Data!.CreateProduct.Id;
    }
}
```

---

## Step 801: Banana Cake Pop (BCP) IDE

```csharp
// Program.cs - เพิ่ม BCP
app.MapBananaCakePop("/graphql/ui")
    .WithOptions(new GraphQLToolOptions
    {
        Document = "query { products { edges { node { id name } } } }",
        UseBrowserUrlAsGraphQLEndpoint = true,
        ServeMode = GraphQLToolServeMode.Embedded
    });
```

---

## Step 802: Custom Scalar Types

```csharp
// Custom scalar for Thai currency
public class ThaiCurrencyType : ScalarType<decimal, FloatValueNode>
{
    public ThaiCurrencyType() : base("THB") { }
    
    public override IValueNode ParseResult(object? resultValue)
    {
        if (resultValue is decimal d)
            return new FloatValueNode(d);
        throw new SerializationException("Cannot parse result to THB", this);
    }
    
    protected override decimal ParseLiteral(FloatValueNode valueSyntax)
        => (decimal)valueSyntax.ToDouble();
    
    protected override FloatValueNode ParseValue(decimal runtimeValue)
        => new(runtimeValue);
    
    public override bool TryDeserialize(object? resultValue, out object? runtimeValue)
    {
        if (resultValue is double d)
        {
            runtimeValue = (decimal)d;
            return true;
        }
        runtimeValue = null;
        return false;
    }
    
    public override bool TrySerialize(object? runtimeValue, out object? resultValue)
    {
        if (runtimeValue is decimal d)
        {
            resultValue = (double)d;
            return true;
        }
        resultValue = null;
        return false;
    }
}

// JSON scalar
public class JsonType : ScalarType<JsonDocument, StringValueNode>
{
    public JsonType() : base("JSON") { }
    
    public override IValueNode ParseResult(object? resultValue)
    {
        if (resultValue is JsonDocument doc)
            return new StringValueNode(doc.RootElement.GetRawText());
        throw new SerializationException("Cannot parse to JSON", this);
    }
    
    protected override JsonDocument ParseLiteral(StringValueNode valueSyntax)
        => JsonDocument.Parse(valueSyntax.Value);
    
    protected override StringValueNode ParseValue(JsonDocument runtimeValue)
        => new(runtimeValue.RootElement.GetRawText());
    
    public override bool TryDeserialize(object? resultValue, out object? runtimeValue)
    {
        if (resultValue is string s)
        {
            runtimeValue = JsonDocument.Parse(s);
            return true;
        }
        runtimeValue = null;
        return false;
    }
    
    public override bool TrySerialize(object? runtimeValue, out object? resultValue)
    {
        if (runtimeValue is JsonDocument doc)
        {
            resultValue = doc.RootElement.GetRawText();
            return true;
        }
        resultValue = null;
        return false;
    }
}

// Register custom scalars
builder.Services
    .AddGraphQLServer()
    .AddType<ThaiCurrencyType>()
    .AddType<JsonType>();
```

---

## Step 803: Upload Files via GraphQL

```csharp
// Add upload support
builder.Services
    .AddGraphQLServer()
    .AddType<UploadType>();

// Mutation with file upload
[MutationType]
public class FileMutation
{
    public async Task<string> UploadProductImage(
        int productId,
        IFile file,
        [ScopedService] AppDbContext db,
        IWebHostEnvironment env,
        CancellationToken ct)
    {
        var product = await db.Products.FindAsync([productId], ct)
            ?? throw new GraphQLException("Product not found");
        
        var fileName = $"{productId}_{Guid.NewGuid()}{Path.GetExtension(file.Name)}";
        var path = Path.Combine(env.WebRootPath, "images", "products", fileName);
        
        Directory.CreateDirectory(Path.GetDirectoryName(path)!);
        
        await using var stream = file.OpenReadStream();
        await using var fileStream = File.Create(path);
        await stream.CopyToAsync(fileStream, ct);
        
        return $"/images/products/{fileName}";
    }
    
    // Multiple file upload
    public async Task<IEnumerable<string>> UploadMultipleFiles(
        IReadOnlyList<IFile> files,
        IWebHostEnvironment env,
        CancellationToken ct)
    {
        var urls = new List<string>();
        
        foreach (var file in files)
        {
            var fileName = $"{Guid.NewGuid()}{Path.GetExtension(file.Name)}";
            var path = Path.Combine(env.WebRootPath, "uploads", fileName);
            
            await using var stream = file.OpenReadStream();
            await using var fileStream = File.Create(path);
            await stream.CopyToAsync(fileStream, ct);
            
            urls.Add($"/uploads/{fileName}");
        }
        
        return urls;
    }
}
```

---

## Step 804: Real-time Dashboard with Subscriptions

```csharp
// Background service publishing events
public class DashboardBackgroundService(
    ITopicEventSender eventSender,
    IServiceProvider services,
    ILogger<DashboardBackgroundService> logger) 
    : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            try
            {
                using var scope = services.CreateScope();
                var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
                
                var stats = new DashboardStats
                {
                    TotalOrders = await db.Orders.CountAsync(ct),
                    TotalRevenue = await db.Orders.SumAsync(o => o.TotalAmount, ct),
                    ActiveUsers = await db.Users.CountAsync(ct),
                    Timestamp = DateTime.UtcNow
                };
                
                await eventSender.SendAsync("dashboard-stats", stats, ct);
            }
            catch (Exception ex)
            {
                logger.LogError(ex, "Error publishing dashboard stats");
            }
            
            await Task.Delay(TimeSpan.FromSeconds(30), ct);
        }
    }
}

public record DashboardStats(
    int TotalOrders, 
    decimal TotalRevenue,
    int ActiveUsers,
    DateTime Timestamp);

// Subscription
[SubscriptionType]
public class DashboardSubscription
{
    [Subscribe]
    [Topic("dashboard-stats")]
    [Authorize(Roles = new[] { "Admin" })]
    public DashboardStats OnDashboardUpdated([EventMessage] DashboardStats stats)
        => stats;
}
```

```graphql
# Client subscription
subscription WatchDashboard {
  onDashboardUpdated {
    totalOrders
    totalRevenue
    activeUsers
    timestamp
  }
}
```

---

## Step 805: Hot Chocolate Middleware Pipeline

```csharp
// Custom middleware
public class RequestLoggingMiddleware(
    RequestDelegate next,
    ILogger<RequestLoggingMiddleware> logger)
{
    public async Task InvokeAsync(IRequestContext context)
    {
        var start = Stopwatch.GetTimestamp();
        
        logger.LogInformation(
            "GraphQL request started: {OperationName}",
            context.Request.OperationName);
        
        await next(context);
        
        var elapsed = Stopwatch.GetElapsedTime(start);
        
        logger.LogInformation(
            "GraphQL request completed: {OperationName} in {Elapsed}ms, errors: {ErrorCount}",
            context.Request.OperationName,
            elapsed.TotalMilliseconds,
            context.Result?.Errors?.Count ?? 0);
    }
}

// Field middleware
public class ValidationMiddleware
{
    private readonly FieldDelegate _next;
    
    public ValidationMiddleware(FieldDelegate next) => _next = next;
    
    public async ValueTask InvokeAsync(IMiddlewareContext context)
    {
        // Pre-field execution
        var field = context.Selection.Field;
        
        if (field.ContextData.ContainsKey("RequireAuthentication"))
        {
            var userId = context.GetGlobalStateOrDefault<string>("currentUserId");
            if (userId == null)
            {
                context.ReportError(
                    ErrorBuilder.New()
                        .SetMessage("Authentication required")
                        .SetCode("UNAUTHENTICATED")
                        .Build());
                return;
            }
        }
        
        await _next(context);
        
        // Post-field execution
    }
}

// Register middleware
builder.Services
    .AddGraphQLServer()
    .Use<RequestLoggingMiddleware>()
    .UseField<ValidationMiddleware>();
```

---

## Step 806: GraphQL Schema-First Approach

```csharp
// หรือใช้ schema-first แทน code-first
builder.Services
    .AddGraphQLServer()
    .AddDocumentFromFile("schema.graphql")
    .BindRuntimeType<Product, ProductType>()
    .AddResolver<ProductQuery>();
```

```graphql
# schema.graphql
type Query {
  products(first: Int, after: String): ProductConnection!
  product(id: ID!): Product
  categories: [Category!]!
}

type Mutation {
  createProduct(input: CreateProductInput!): Product!
  updateProduct(input: UpdateProductInput!): Product!
  deleteProduct(id: ID!): Boolean!
}

type Subscription {
  onStockChanged(productId: ID!): Product!
  onProductCreated: Product!
}

type Product {
  id: ID!
  name: String!
  description: String
  price: Float!
  priceWithVat: Float!
  stockQty: Int!
  category: Category!
  reviews: [Review!]!
  createdAt: DateTime!
}

type Category {
  id: ID!
  name: String!
  slug: String!
  products: [Product!]!
}

type Review {
  id: ID!
  rating: Int!
  comment: String!
  user: User!
  createdAt: DateTime!
}

type ProductConnection {
  totalCount: Int!
  pageInfo: PageInfo!
  edges: [ProductEdge!]!
}

type ProductEdge {
  cursor: String!
  node: Product!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}

input CreateProductInput {
  name: String!
  description: String
  price: Float!
  categoryId: Int!
  stockQuantity: Int!
}

input UpdateProductInput {
  id: ID!
  name: String
  price: Float
  stockQuantity: Int
}
```

---

## Step 807: Hot Reload Schema

```csharp
// Dynamic schema changes
public class SchemaVersioner : ISchemaVersioner
{
    private int _version = 1;
    
    public void BumpVersion() => Interlocked.Increment(ref _version);
    
    public int GetVersion() => _version;
}

// Feature flags in schema
[QueryType]
public class FeatureQuery
{
    [UseDbContext(typeof(AppDbContext))]
    public IQueryable<Product> GetProducts(
        [ScopedService] AppDbContext db,
        [Service] IFeatureFlagService featureFlags)
    {
        var query = db.Products.AsQueryable();
        
        if (featureFlags.IsEnabled("ShowDraftProducts"))
            return query;
        
        return query.Where(p => p.IsActive);
    }
}
```

---

## Step 808: API Gateway Pattern

```csharp
// Combine multiple GraphQL APIs
// gateway/Program.cs
builder.Services
    .AddHttpClient("ProductService", c => 
        c.BaseAddress = new Uri("http://product-service/graphql"))
    .AddHttpClient("ReviewService", c =>
        c.BaseAddress = new Uri("http://review-service/graphql"))
    .AddHttpClient("UserService", c =>
        c.BaseAddress = new Uri("http://user-service/graphql"));

builder.Services
    .AddGraphQLServer()
    .AddRemoteSchema("ProductService")
    .AddRemoteSchema("ReviewService")
    .AddRemoteSchema("UserService")
    .AddTypeExtensionsFromFile("./stitching.graphql");
```

```graphql
# stitching.graphql - เชื่อมต่อ types จาก services ต่างๆ
extend type Product {
  reviews: [Review!]! @delegate(schema: "ReviewService", path: "reviewsByProduct(productId: $fields:id)")
  reviewStats: ReviewStats! @delegate(schema: "ReviewService")
}

extend type Review {
  user: User! @delegate(schema: "UserService", path: "user(id: $fields:userId)")
}
```

---

## Step 809: Production Configuration

```json
// appsettings.json
{
  "ConnectionStrings": {
    "Default": "Server=...;Database=GraphQLDemo;..."
  },
  "Jwt": {
    "Secret": "your-256-bit-secret-key",
    "Issuer": "GraphQLDemo",
    "ExpiryMinutes": 60
  },
  "GraphQL": {
    "MaxExecutionDepth": 10,
    "MaxComplexity": 1000,
    "IncludeExceptionDetails": false,
    "EnableIntrospection": true
  },
  "Redis": {
    "ConnectionString": "localhost:6379"
  }
}
```

```csharp
// Production security settings
builder.Services
    .AddGraphQLServer()
    .ModifyRequestOptions(opts =>
    {
        opts.IncludeExceptionDetails = false;
    })
    .ModifyParserOptions(opts =>
    {
        opts.MaxAllowedNodes = 1000;
    })
    .AddMaxExecutionDepthRule(
        int.Parse(builder.Configuration["GraphQL:MaxExecutionDepth"]!))
    .AddQueryComplexityRule(opts =>
    {
        opts.MaximumAllowed = 
            int.Parse(builder.Configuration["GraphQL:MaxComplexity"]!);
    });

// Disable introspection in production
if (!builder.Environment.IsDevelopment())
{
    builder.Services
        .AddGraphQLServer()
        .ModifyOptions(opts => opts.EnableSchemaIntrospection = false);
}
```

---

## Step 810: สรุป Hot Chocolate GraphQL

### ความสามารถหลัก

| Feature | Hot Chocolate |
|---------|--------------|
| Code-First | ✅ ObjectType<T>, [QueryType] |
| Schema-First | ✅ .graphql files |
| Filtering | ✅ [UseFiltering] |
| Sorting | ✅ [UseSorting] |
| Paging | ✅ Cursor & Offset |
| Projection | ✅ [UseProjection] |
| DataLoader | ✅ BatchDataLoader<K,V> |
| Subscriptions | ✅ In-memory, Redis |
| Federation | ✅ Apollo Federation |
| StrawberryShake | ✅ Typed .NET client |
| Authentication | ✅ ASP.NET Core Auth |
| Middleware | ✅ Request & Field |

### Best Practices
1. ใช้ **DataLoader** เสมอเพื่อหลีกเลี่ยง N+1
2. กำหนด **Query Complexity** ป้องกัน abuse
3. ใช้ **Cursor Paging** แทน offset สำหรับ large datasets
4. ทำ **Persisted Queries** สำหรับ production
5. ใช้ **Projections** เพื่อ select เฉพาะ columns ที่ต้องการ
6. แยก **Error types** ให้ชัดเจน (domain errors vs system errors)

---

*จบ Part 29: GraphQL with Hot Chocolate*
*ต่อไป Part 30: Microservices Architecture with .NET Aspire*
