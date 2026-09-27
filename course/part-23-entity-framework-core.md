# Part 23: Entity Framework Core (EF Core)

## ขั้นตอนที่ 637: EF Core คืออะไร?

Entity Framework Core คือ ORM (Object-Relational Mapper) ที่ช่วยให้คุณทำงานกับ database ผ่าน C# objects แทนที่จะเขียน SQL ตรงๆ

```bash
# ติดตั้ง packages
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Design      # Migrations
dotnet add package Microsoft.EntityFrameworkCore.Tools       # dotnet-ef

# Global tool
dotnet tool install --global dotnet-ef

# หรือ SQLite (lightweight, ไม่ต้อง server)
dotnet add package Microsoft.EntityFrameworkCore.Sqlite

# PostgreSQL
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL
```

## ขั้นตอนที่ 638: DbContext และ Entity Models

```csharp
// Entity Models
public class Customer
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public string Email { get; set; } = "";
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public bool IsActive { get; set; } = true;
    
    // Navigation properties
    public List<Order> Orders { get; set; } = [];
    public CustomerProfile? Profile { get; set; }
}

public class Order
{
    public int Id { get; set; }
    public int CustomerId { get; set; }
    public DateTime OrderDate { get; set; } = DateTime.UtcNow;
    public OrderStatus Status { get; set; } = OrderStatus.Pending;
    public decimal TotalAmount { get; set; }
    
    // Navigation properties
    public Customer Customer { get; set; } = null!;
    public List<OrderItem> Items { get; set; } = [];
    public Payment? Payment { get; set; }
}

public class OrderItem
{
    public int Id { get; set; }
    public int OrderId { get; set; }
    public int ProductId { get; set; }
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
    
    public Order Order { get; set; } = null!;
    public Product Product { get; set; } = null!;
    
    public decimal Subtotal => Quantity * UnitPrice;
}

public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public string? Description { get; set; }
    public decimal Price { get; set; }
    public int Stock { get; set; }
    public string Category { get; set; } = "";
    public bool IsAvailable { get; set; } = true;
    
    public List<OrderItem> OrderItems { get; set; } = [];
}

public class CustomerProfile
{
    public int Id { get; set; }
    public int CustomerId { get; set; }
    public string? Phone { get; set; }
    public string? Address { get; set; }
    public DateTime? BirthDate { get; set; }
    
    public Customer Customer { get; set; } = null!;
}

public enum OrderStatus { Pending, Processing, Shipped, Delivered, Cancelled }

// DbContext
public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }
    
    // DbSets - ตาราง
    public DbSet<Customer> Customers { get; set; }
    public DbSet<Order> Orders { get; set; }
    public DbSet<OrderItem> OrderItems { get; set; }
    public DbSet<Product> Products { get; set; }
    public DbSet<CustomerProfile> CustomerProfiles { get; set; }
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
        
        // Apply all configurations from assembly
        modelBuilder.ApplyConfigurationsFromAssembly(Assembly.GetExecutingAssembly());
    }
    
    // Override SaveChanges เพื่อ auto audit
    public override async Task<int> SaveChangesAsync(CancellationToken ct = default)
    {
        foreach (var entry in ChangeTracker.Entries<IAuditable>())
        {
            if (entry.State == EntityState.Added)
                entry.Entity.CreatedAt = DateTime.UtcNow;
            
            if (entry.State == EntityState.Modified)
                entry.Entity.UpdatedAt = DateTime.UtcNow;
        }
        
        return await base.SaveChangesAsync(ct);
    }
}

public interface IAuditable
{
    DateTime CreatedAt { get; set; }
    DateTime? UpdatedAt { get; set; }
}
```

## ขั้นตอนที่ 639: Fluent API Configuration

```csharp
// EntityTypeConfiguration สำหรับ Customer
public class CustomerConfiguration : IEntityTypeConfiguration<Customer>
{
    public void Configure(EntityTypeBuilder<Customer> builder)
    {
        builder.ToTable("Customers");
        
        builder.HasKey(c => c.Id);
        
        builder.Property(c => c.Name)
            .IsRequired()
            .HasMaxLength(100);
        
        builder.Property(c => c.Email)
            .IsRequired()
            .HasMaxLength(200);
        
        // Unique index
        builder.HasIndex(c => c.Email)
            .IsUnique()
            .HasDatabaseName("IX_Customers_Email");
        
        builder.Property(c => c.CreatedAt)
            .HasDefaultValueSql("GETUTCDATE()");
        
        // One-to-many: Customer -> Orders
        builder.HasMany(c => c.Orders)
            .WithOne(o => o.Customer)
            .HasForeignKey(o => o.CustomerId)
            .OnDelete(DeleteBehavior.Restrict);
        
        // One-to-one: Customer -> Profile
        builder.HasOne(c => c.Profile)
            .WithOne(p => p.Customer)
            .HasForeignKey<CustomerProfile>(p => p.CustomerId);
        
        // Seed data
        builder.HasData(
            new Customer { Id = 1, Name = "Alice", Email = "alice@example.com" },
            new Customer { Id = 2, Name = "Bob", Email = "bob@example.com" }
        );
    }
}

// OrderConfiguration
public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.ToTable("Orders");
        
        builder.HasKey(o => o.Id);
        
        builder.Property(o => o.Status)
            .HasConversion<string>()  // Enum stored as string
            .HasMaxLength(20);
        
        builder.Property(o => o.TotalAmount)
            .HasColumnType("decimal(18,2)");
        
        // One-to-many: Order -> Items
        builder.HasMany(o => o.Items)
            .WithOne(i => i.Order)
            .HasForeignKey(i => i.OrderId)
            .OnDelete(DeleteBehavior.Cascade);
        
        // One-to-one: Order -> Payment
        builder.HasOne(o => o.Payment)
            .WithOne(p => p.Order)
            .HasForeignKey<Payment>(p => p.OrderId);
        
        // Query filter: exclude cancelled orders by default
        builder.HasQueryFilter(o => o.Status != OrderStatus.Cancelled);
        
        // Index on CustomerId for performance
        builder.HasIndex(o => o.CustomerId)
            .HasDatabaseName("IX_Orders_CustomerId");
        
        builder.HasIndex(o => o.OrderDate)
            .HasDatabaseName("IX_Orders_OrderDate");
    }
}

// Owned entity (Value Object)
public class Address
{
    public string Street { get; set; } = "";
    public string City { get; set; } = "";
    public string PostalCode { get; set; } = "";
    public string Country { get; set; } = "";
}

public class Vendor
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public Address BillingAddress { get; set; } = new();
    public Address ShippingAddress { get; set; } = new();
}

public class VendorConfiguration : IEntityTypeConfiguration<Vendor>
{
    public void Configure(EntityTypeBuilder<Vendor> builder)
    {
        // Owned entity: columns inline in same table
        builder.OwnsOne(v => v.BillingAddress, address =>
        {
            address.Property(a => a.Street).HasColumnName("BillingStreet").HasMaxLength(200);
            address.Property(a => a.City).HasColumnName("BillingCity").HasMaxLength(100);
            address.Property(a => a.PostalCode).HasColumnName("BillingPostalCode").HasMaxLength(10);
            address.Property(a => a.Country).HasColumnName("BillingCountry").HasMaxLength(50);
        });
        
        builder.OwnsOne(v => v.ShippingAddress, address =>
        {
            address.Property(a => a.Street).HasColumnName("ShippingStreet").HasMaxLength(200);
            address.Property(a => a.City).HasColumnName("ShippingCity").HasMaxLength(100);
            // ...
        });
    }
}
```

## ขั้นตอนที่ 640: CRUD Operations

```csharp
public class CustomerRepository
{
    private readonly AppDbContext _context;
    
    public CustomerRepository(AppDbContext context) => _context = context;
    
    // === CREATE ===
    public async Task<Customer> CreateAsync(Customer customer, CancellationToken ct = default)
    {
        _context.Customers.Add(customer);
        await _context.SaveChangesAsync(ct);
        return customer;
    }
    
    public async Task<List<Customer>> BulkCreateAsync(
        List<Customer> customers, 
        CancellationToken ct = default)
    {
        await _context.Customers.AddRangeAsync(customers, ct);
        await _context.SaveChangesAsync(ct);
        return customers;
    }
    
    // === READ ===
    public async Task<Customer?> GetByIdAsync(int id, CancellationToken ct = default)
        => await _context.Customers
            .Include(c => c.Profile)
            .FirstOrDefaultAsync(c => c.Id == id, ct);
    
    public async Task<Customer?> GetWithOrdersAsync(int id, CancellationToken ct = default)
        => await _context.Customers
            .Include(c => c.Orders)
                .ThenInclude(o => o.Items)
                    .ThenInclude(i => i.Product)
            .Include(c => c.Profile)
            .FirstOrDefaultAsync(c => c.Id == id, ct);
    
    public async Task<List<Customer>> GetAllActiveAsync(CancellationToken ct = default)
        => await _context.Customers
            .Where(c => c.IsActive)
            .OrderBy(c => c.Name)
            .ToListAsync(ct);
    
    // No-tracking queries (faster, read-only)
    public async Task<List<CustomerDto>> GetAllDtosAsync(CancellationToken ct = default)
        => await _context.Customers
            .AsNoTracking()
            .Where(c => c.IsActive)
            .Select(c => new CustomerDto(c.Id, c.Name, c.Email, c.Orders.Count))
            .ToListAsync(ct);
    
    // Paging
    public async Task<PagedResult<Customer>> GetPagedAsync(
        int page, 
        int pageSize, 
        string? search = null,
        CancellationToken ct = default)
    {
        var query = _context.Customers.AsQueryable();
        
        if (!string.IsNullOrEmpty(search))
            query = query.Where(c => c.Name.Contains(search) || c.Email.Contains(search));
        
        var total = await query.CountAsync(ct);
        var items = await query
            .OrderBy(c => c.Name)
            .Skip((page - 1) * pageSize)
            .Take(pageSize)
            .AsNoTracking()
            .ToListAsync(ct);
        
        return new PagedResult<Customer>(items, total, page, pageSize);
    }
    
    // === UPDATE ===
    public async Task UpdateAsync(Customer customer, CancellationToken ct = default)
    {
        _context.Customers.Update(customer); // marks all properties as modified
        await _context.SaveChangesAsync(ct);
    }
    
    public async Task UpdateEmailAsync(int id, string newEmail, CancellationToken ct = default)
    {
        // Partial update: only update Email column
        await _context.Customers
            .Where(c => c.Id == id)
            .ExecuteUpdateAsync(s => s
                .SetProperty(c => c.Email, newEmail)
                .SetProperty(c => c.UpdatedAt, DateTime.UtcNow), ct);
    }
    
    // === DELETE ===
    public async Task DeleteAsync(int id, CancellationToken ct = default)
    {
        var customer = await _context.Customers.FindAsync([id], ct);
        if (customer is null) return;
        
        _context.Customers.Remove(customer);
        await _context.SaveChangesAsync(ct);
    }
    
    // Soft delete
    public async Task SoftDeleteAsync(int id, CancellationToken ct = default)
    {
        await _context.Customers
            .Where(c => c.Id == id)
            .ExecuteUpdateAsync(s => s.SetProperty(c => c.IsActive, false), ct);
    }
    
    // Bulk delete (EF Core 7+)
    public async Task<int> DeleteInactiveAsync(CancellationToken ct = default)
        => await _context.Customers
            .Where(c => !c.IsActive)
            .ExecuteDeleteAsync(ct);
}

public record CustomerDto(int Id, string Name, string Email, int OrderCount);
public record PagedResult<T>(List<T> Items, int Total, int Page, int PageSize)
{
    public int TotalPages => (int)Math.Ceiling((double)Total / PageSize);
    public bool HasNext => Page < TotalPages;
    public bool HasPrevious => Page > 1;
}
```

## ขั้นตอนที่ 641: Migrations

```bash
# สร้าง migration ครั้งแรก
dotnet ef migrations add InitialCreate

# สร้าง migration หลังจาก model เปลี่ยน
dotnet ef migrations add AddCustomerProfile

# Apply migration ไปยัง database
dotnet ef database update

# Apply migration เฉพาะถึง version ที่กำหนด
dotnet ef database update 20241201000000_AddCustomerProfile

# Rollback migration หนึ่ง step
dotnet ef database update 20241130000000_InitialCreate

# ดู migration ทั้งหมด
dotnet ef migrations list

# ลบ migration ล่าสุด (ยังไม่ apply)
dotnet ef migrations remove

# Generate SQL script (สำหรับ production)
dotnet ef migrations script --output migrations.sql
dotnet ef migrations script 20241130000000_InitialCreate 20241201000000_AddCustomerProfile

# Update database จาก code (สำหรับ development/CI)
# Program.cs
using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    await db.Database.MigrateAsync(); // Apply pending migrations
}
```

### Migration File ตัวอย่าง

```csharp
// 20241201000000_AddCustomerProfile.cs (Generated)
public partial class AddCustomerProfile : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.CreateTable(
            name: "CustomerProfiles",
            columns: table => new
            {
                Id = table.Column<int>(type: "int", nullable: false)
                    .Annotation("SqlServer:Identity", "1, 1"),
                CustomerId = table.Column<int>(type: "int", nullable: false),
                Phone = table.Column<string>(type: "nvarchar(20)", maxLength: 20, nullable: true),
                Address = table.Column<string>(type: "nvarchar(500)", maxLength: 500, nullable: true),
                BirthDate = table.Column<DateTime>(type: "datetime2", nullable: true)
            },
            constraints: table =>
            {
                table.PrimaryKey("PK_CustomerProfiles", x => x.Id);
                table.ForeignKey(
                    name: "FK_CustomerProfiles_Customers_CustomerId",
                    column: x => x.CustomerId,
                    principalTable: "Customers",
                    principalColumn: "Id",
                    onDelete: ReferentialAction.Cascade);
            });
        
        migrationBuilder.CreateIndex(
            name: "IX_CustomerProfiles_CustomerId",
            table: "CustomerProfiles",
            column: "CustomerId",
            unique: true);
    }
    
    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropTable(name: "CustomerProfiles");
    }
}

// Custom SQL ใน migration
public partial class AddStoredProcedure : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.Sql(@"
            CREATE PROCEDURE sp_GetCustomerSummary
                @CustomerId INT
            AS
            BEGIN
                SELECT 
                    c.Name,
                    c.Email,
                    COUNT(o.Id) AS OrderCount,
                    SUM(o.TotalAmount) AS TotalSpent
                FROM Customers c
                LEFT JOIN Orders o ON o.CustomerId = c.Id
                WHERE c.Id = @CustomerId
                GROUP BY c.Name, c.Email
            END");
    }
    
    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.Sql("DROP PROCEDURE IF EXISTS sp_GetCustomerSummary");
    }
}
```

## ขั้นตอนที่ 642: Querying - LINQ to Entities

```csharp
public class OrderQueryService
{
    private readonly AppDbContext _context;
    
    public OrderQueryService(AppDbContext context) => _context = context;
    
    // Basic queries
    public async Task<List<Order>> GetRecentOrdersAsync(int days = 7)
        => await _context.Orders
            .Where(o => o.OrderDate >= DateTime.UtcNow.AddDays(-days))
            .OrderByDescending(o => o.OrderDate)
            .Include(o => o.Customer)
            .AsNoTracking()
            .ToListAsync();
    
    // Projection - load only needed columns
    public async Task<List<OrderSummaryDto>> GetOrderSummariesAsync()
        => await _context.Orders
            .AsNoTracking()
            .Select(o => new OrderSummaryDto(
                o.Id,
                o.Customer.Name,
                o.OrderDate,
                o.Status.ToString(),
                o.TotalAmount,
                o.Items.Count))
            .ToListAsync();
    
    // Grouping & Aggregation
    public async Task<List<SalesReport>> GetMonthlySalesAsync(int year)
        => await _context.Orders
            .Where(o => o.OrderDate.Year == year && o.Status == OrderStatus.Delivered)
            .GroupBy(o => o.OrderDate.Month)
            .Select(g => new SalesReport(
                Month: g.Key,
                OrderCount: g.Count(),
                TotalRevenue: g.Sum(o => o.TotalAmount),
                AvgOrderValue: g.Average(o => o.TotalAmount)))
            .OrderBy(r => r.Month)
            .ToListAsync();
    
    // Join
    public async Task<List<ProductSalesDto>> GetTopProductsAsync(int topN = 10)
        => await _context.OrderItems
            .AsNoTracking()
            .GroupBy(i => new { i.ProductId, i.Product.Name })
            .Select(g => new ProductSalesDto(
                g.Key.ProductId,
                g.Key.Name,
                TotalQuantity: g.Sum(i => i.Quantity),
                TotalRevenue: g.Sum(i => i.Quantity * i.UnitPrice)))
            .OrderByDescending(p => p.TotalRevenue)
            .Take(topN)
            .ToListAsync();
    
    // Subquery
    public async Task<List<Customer>> GetHighValueCustomersAsync(decimal minSpent)
        => await _context.Customers
            .Where(c => c.Orders.Sum(o => o.TotalAmount) >= minSpent)
            .Include(c => c.Orders)
            .AsNoTracking()
            .ToListAsync();
    
    // Raw SQL (สำหรับ complex queries)
    public async Task<List<CustomerRevenue>> GetCustomerRevenueAsync()
        => await _context.Customers
            .FromSqlRaw(@"
                SELECT c.Id, c.Name, c.Email,
                    COALESCE(SUM(o.TotalAmount), 0) as Revenue
                FROM Customers c
                LEFT JOIN Orders o ON o.CustomerId = c.Id 
                    AND o.Status = 'Delivered'
                GROUP BY c.Id, c.Name, c.Email
                HAVING COALESCE(SUM(o.TotalAmount), 0) > 0
                ORDER BY Revenue DESC")
            .AsNoTracking()
            .ToListAsync();
    
    // Interpolated SQL (parameterized automatically)
    public async Task<List<Product>> SearchProductsAsync(string keyword, decimal maxPrice)
        => await _context.Products
            .FromSql($"SELECT * FROM Products WHERE Name LIKE '%{keyword}%' AND Price <= {maxPrice}")
            .AsNoTracking()
            .ToListAsync();
    
    // Compiled Query (max performance)
    private static readonly Func<AppDbContext, int, Task<Order?>> GetOrderWithItemsQuery =
        EF.CompileAsyncQuery((AppDbContext ctx, int id) =>
            ctx.Orders
                .Include(o => o.Items)
                    .ThenInclude(i => i.Product)
                .FirstOrDefault(o => o.Id == id));
    
    public Task<Order?> GetOrderWithItemsFastAsync(int id)
        => GetOrderWithItemsQuery(_context, id);
}

public record OrderSummaryDto(int Id, string CustomerName, DateTime Date, string Status, decimal Total, int ItemCount);
public record SalesReport(int Month, int OrderCount, decimal TotalRevenue, decimal AvgOrderValue);
public record ProductSalesDto(int Id, string Name, int TotalQuantity, decimal TotalRevenue);
public record CustomerRevenue
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public string Email { get; set; } = "";
    public decimal Revenue { get; set; }
}
```

## ขั้นตอนที่ 643: Change Tracking

```csharp
public class ChangeTrackingDemo
{
    private readonly AppDbContext _context;
    
    public ChangeTrackingDemo(AppDbContext context) => _context = context;
    
    public async Task DemoChangeTrackingAsync()
    {
        // 1. Load entity - EF Core tracks it
        var customer = await _context.Customers.FindAsync(1);
        Console.WriteLine($"State: {_context.Entry(customer!).State}"); // Unchanged
        
        // 2. Modify - state changes to Modified
        customer!.Name = "New Name";
        Console.WriteLine($"State: {_context.Entry(customer).State}"); // Modified
        
        // 3. Check what changed
        var entry = _context.Entry(customer);
        foreach (var prop in entry.Properties)
        {
            if (prop.IsModified)
                Console.WriteLine($"  {prop.Metadata.Name}: {prop.OriginalValue} -> {prop.CurrentValue}");
        }
        
        // 4. Save
        await _context.SaveChangesAsync();
        Console.WriteLine($"State after save: {_context.Entry(customer).State}"); // Unchanged
        
        // 5. Detach to stop tracking
        _context.Entry(customer).State = EntityState.Detached;
        customer.Email = "new@email.com"; // ไม่ถูก track
        
        // 6. Attach เพื่อ update
        _context.Update(customer); // Reattach and mark as Modified
        await _context.SaveChangesAsync();
    }
    
    public async Task NoTrackingDemoAsync()
    {
        // AsNoTracking: เร็วกว่าสำหรับ read-only
        var customers = await _context.Customers
            .AsNoTracking()
            .ToListAsync();
        
        // ไม่สามารถ update ได้ตรงๆ ต้อง Attach ก่อน
        var customer = customers[0];
        customer.Name = "Changed";
        
        _context.Update(customer); // Attach ด้วย Modified state
        await _context.SaveChangesAsync();
    }
}
```

## ขั้นตอนที่ 644: Transactions

```csharp
public class TransactionDemo
{
    private readonly AppDbContext _context;
    
    public TransactionDemo(AppDbContext context) => _context = context;
    
    // Automatic transaction (SaveChanges is atomic)
    public async Task TransferStockAsync(int fromProductId, int toProductId, int qty)
    {
        var from = await _context.Products.FindAsync(fromProductId)
            ?? throw new NotFoundException("Source product not found");
        var to = await _context.Products.FindAsync(toProductId)
            ?? throw new NotFoundException("Target product not found");
        
        if (from.Stock < qty)
            throw new InsufficientStockException(fromProductId, qty);
        
        from.Stock -= qty;
        to.Stock += qty;
        
        // Both changes in one SaveChanges = one transaction
        await _context.SaveChangesAsync();
    }
    
    // Explicit transaction
    public async Task PlaceOrderWithPaymentAsync(Order order, PaymentInfo payment)
    {
        await using var transaction = await _context.Database.BeginTransactionAsync();
        
        try
        {
            // Operation 1: Save order
            _context.Orders.Add(order);
            await _context.SaveChangesAsync();
            
            // Operation 2: Update stock
            foreach (var item in order.Items)
            {
                await _context.Products
                    .Where(p => p.Id == item.ProductId)
                    .ExecuteUpdateAsync(s => 
                        s.SetProperty(p => p.Stock, p => p.Stock - item.Quantity));
            }
            
            // Operation 3: Save payment record
            var paymentRecord = new Payment
            {
                OrderId = order.Id,
                Amount = order.TotalAmount,
                TransactionId = payment.TransactionId,
                PaidAt = DateTime.UtcNow
            };
            _context.Set<Payment>().Add(paymentRecord);
            await _context.SaveChangesAsync();
            
            // All operations succeeded
            await transaction.CommitAsync();
        }
        catch
        {
            await transaction.RollbackAsync();
            throw;
        }
    }
    
    // Savepoints (.NET 6+)
    public async Task ComplexOperationAsync()
    {
        await using var transaction = await _context.Database.BeginTransactionAsync();
        
        try
        {
            // Phase 1
            await DoPhase1Async();
            await _context.SaveChangesAsync();
            
            await transaction.CreateSavepointAsync("AfterPhase1");
            
            // Phase 2 (might fail)
            try
            {
                await DoPhase2Async();
                await _context.SaveChangesAsync();
            }
            catch
            {
                await transaction.RollbackToSavepointAsync("AfterPhase1");
                // Phase 1 still committed, phase 2 rolled back
            }
            
            await transaction.CommitAsync();
        }
        catch
        {
            await transaction.RollbackAsync();
            throw;
        }
    }
}
```

## ขั้นตอนที่ 645: Performance Optimization

```csharp
public class EFPerformanceTips
{
    private readonly AppDbContext _context;
    
    public EFPerformanceTips(AppDbContext context) => _context = context;
    
    // 1. Use AsNoTracking() for read-only queries
    public async Task<List<Product>> GetReadOnlyAsync()
        => await _context.Products.AsNoTracking().ToListAsync();
    
    // 2. Select only needed columns (projection)
    public async Task<List<(int Id, string Name)>> GetNamesAsync()
        => await _context.Products
            .AsNoTracking()
            .Select(p => new ValueTuple<int, string>(p.Id, p.Name))
            .ToListAsync();
    
    // 3. Load related data efficiently
    
    // ❌ N+1 problem
    public async Task NPlus1ProblemAsync()
    {
        var customers = await _context.Customers.ToListAsync();
        foreach (var customer in customers)
        {
            // Each iteration = 1 more SQL query!
            var orders = await _context.Orders
                .Where(o => o.CustomerId == customer.Id)
                .ToListAsync();
        }
    }
    
    // ✅ Eager loading (one query with JOIN)
    public async Task EagerLoadAsync()
    {
        var customers = await _context.Customers
            .Include(c => c.Orders)
            .ToListAsync();
    }
    
    // ✅ Split query (for multiple collections - avoid cartesian explosion)
    public async Task SplitQueryAsync()
    {
        var customers = await _context.Customers
            .AsSplitQuery()  // Uses multiple queries instead of one big JOIN
            .Include(c => c.Orders)
                .ThenInclude(o => o.Items)
            .Include(c => c.Profile)
            .ToListAsync();
    }
    
    // 4. Bulk operations (EF Core 7+)
    public async Task BulkUpdateAsync()
    {
        // ❌ Slow: load all, modify, save
        // var products = await _context.Products.ToListAsync();
        // foreach (var p in products) p.Price *= 1.1m;
        // await _context.SaveChangesAsync();
        
        // ✅ Fast: single UPDATE query
        await _context.Products
            .Where(p => p.Category == "Electronics")
            .ExecuteUpdateAsync(s => s
                .SetProperty(p => p.Price, p => p.Price * 1.1m));
    }
    
    // 5. Use indexes (defined in configuration)
    
    // 6. Pagination - always use Skip/Take
    public async Task<List<Product>> GetPagedAsync(int page, int size)
        => await _context.Products
            .AsNoTracking()
            .OrderBy(p => p.Id)  // Must have ORDER BY with pagination
            .Skip((page - 1) * size)
            .Take(size)
            .ToListAsync();
    
    // 7. Batch size (for insertions)
    public async Task BulkInsertAsync(List<Product> products)
    {
        // EF Core inserts in batches automatically
        await _context.Products.AddRangeAsync(products);
        await _context.SaveChangesAsync();
        
        // For very large sets, use EFCore.BulkExtensions
        // await _context.BulkInsertAsync(products);
    }
    
    // 8. Connection pooling (configured at DbContext level)
    // builder.Services.AddDbContext<AppDbContext>(opts => 
    //     opts.UseSqlServer(connString, sqlOpts => 
    //         sqlOpts.MinBatchSize(1).MaxBatchSize(100)));
    
    // 9. Query logging
    public async Task LoggedQueryAsync()
    {
        var products = await _context.Products
            .TagWith("GetAllActiveProducts") // Tag in SQL comment
            .Where(p => p.IsAvailable)
            .ToListAsync();
    }
}
```

## ขั้นตอนที่ 646: Interceptors

```csharp
// Interceptor: เพิ่ม behavior ก่อน/หลัง database operations
public class AuditInterceptor : SaveChangesInterceptor
{
    private readonly ICurrentUserService _currentUser;
    
    public AuditInterceptor(ICurrentUserService currentUser)
    {
        _currentUser = currentUser;
    }
    
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData,
        InterceptionResult<int> result,
        CancellationToken ct = default)
    {
        var context = eventData.Context;
        if (context is null) return base.SavingChangesAsync(eventData, result, ct);
        
        var userId = _currentUser.UserId;
        var now = DateTime.UtcNow;
        
        foreach (var entry in context.ChangeTracker.Entries())
        {
            if (entry.Entity is IAuditable auditable)
            {
                switch (entry.State)
                {
                    case EntityState.Added:
                        auditable.CreatedAt = now;
                        auditable.CreatedBy = userId;
                        break;
                    case EntityState.Modified:
                        auditable.UpdatedAt = now;
                        auditable.UpdatedBy = userId;
                        break;
                }
            }
        }
        
        return base.SavingChangesAsync(eventData, result, ct);
    }
}

// Query Interceptor: slow query logging
public class SlowQueryInterceptor : DbCommandInterceptor
{
    private readonly ILogger<SlowQueryInterceptor> _logger;
    private static readonly TimeSpan Threshold = TimeSpan.FromSeconds(1);
    
    public SlowQueryInterceptor(ILogger<SlowQueryInterceptor> logger)
    {
        _logger = logger;
    }
    
    public override async ValueTask<DbDataReader> ReaderExecutedAsync(
        DbCommand command,
        CommandExecutedEventData eventData,
        DbDataReader result,
        CancellationToken ct = default)
    {
        if (eventData.Duration >= Threshold)
        {
            _logger.LogWarning(
                "Slow query ({Duration}ms): {Sql}",
                eventData.Duration.TotalMilliseconds,
                command.CommandText);
        }
        
        return await base.ReaderExecutedAsync(command, eventData, result, ct);
    }
}

// Registration
builder.Services.AddDbContext<AppDbContext>((sp, options) =>
{
    options.UseSqlServer(builder.Configuration.GetConnectionString("Default"))
        .AddInterceptors(
            sp.GetRequiredService<AuditInterceptor>(),
            sp.GetRequiredService<SlowQueryInterceptor>());
});
```

## ขั้นตอนที่ 647: DbContext Registration (Best Practices)

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Basic registration
builder.Services.AddDbContext<AppDbContext>(options =>
{
    options.UseSqlServer(
        builder.Configuration.GetConnectionString("Default"),
        sqlOptions =>
        {
            sqlOptions.EnableRetryOnFailure(
                maxRetryCount: 5,
                maxRetryDelay: TimeSpan.FromSeconds(30),
                errorNumbersToAdd: null);
            
            sqlOptions.CommandTimeout(60); // 60 second timeout
            
            sqlOptions.MigrationsAssembly("MyApp.Infrastructure");
        });
    
    // Development: show SQL queries
    if (builder.Environment.IsDevelopment())
    {
        options.EnableSensitiveDataLogging();
        options.EnableDetailedErrors();
        options.LogTo(Console.WriteLine, LogLevel.Information);
    }
});

// DbContextFactory สำหรับ background services
builder.Services.AddDbContextFactory<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

// Background service ใช้ factory
public class OrderCleanupService : BackgroundService
{
    private readonly IDbContextFactory<AppDbContext> _factory;
    
    public OrderCleanupService(IDbContextFactory<AppDbContext> factory) => _factory = factory;
    
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            await using var context = await _factory.CreateDbContextAsync(ct);
            
            await context.Orders
                .Where(o => o.Status == OrderStatus.Pending && 
                           o.OrderDate < DateTime.UtcNow.AddDays(-7))
                .ExecuteUpdateAsync(s => s.SetProperty(o => o.Status, OrderStatus.Cancelled), ct);
            
            await Task.Delay(TimeSpan.FromHours(1), ct);
        }
    }
}
```

## ขั้นตอนที่ 648: Unit of Work Pattern

```csharp
// Unit of Work: ทำ operations หลายๆ อย่างใน transaction เดียวกัน

public interface IUnitOfWork : IDisposable
{
    ICustomerRepository Customers { get; }
    IOrderRepository Orders { get; }
    IProductRepository Products { get; }
    
    Task<int> SaveChangesAsync(CancellationToken ct = default);
    Task BeginTransactionAsync(CancellationToken ct = default);
    Task CommitTransactionAsync(CancellationToken ct = default);
    Task RollbackTransactionAsync(CancellationToken ct = default);
}

public class UnitOfWork : IUnitOfWork
{
    private readonly AppDbContext _context;
    private IDbContextTransaction? _transaction;
    
    private ICustomerRepository? _customers;
    private IOrderRepository? _orders;
    private IProductRepository? _products;
    
    public UnitOfWork(AppDbContext context) => _context = context;
    
    public ICustomerRepository Customers
        => _customers ??= new CustomerRepository(_context);
    
    public IOrderRepository Orders
        => _orders ??= new OrderRepository(_context);
    
    public IProductRepository Products
        => _products ??= new ProductRepository(_context);
    
    public async Task<int> SaveChangesAsync(CancellationToken ct = default)
        => await _context.SaveChangesAsync(ct);
    
    public async Task BeginTransactionAsync(CancellationToken ct = default)
        => _transaction = await _context.Database.BeginTransactionAsync(ct);
    
    public async Task CommitTransactionAsync(CancellationToken ct = default)
    {
        try
        {
            await _context.SaveChangesAsync(ct);
            if (_transaction is not null)
                await _transaction.CommitAsync(ct);
        }
        catch
        {
            await RollbackTransactionAsync(ct);
            throw;
        }
    }
    
    public async Task RollbackTransactionAsync(CancellationToken ct = default)
    {
        if (_transaction is not null)
        {
            await _transaction.RollbackAsync(ct);
            await _transaction.DisposeAsync();
            _transaction = null;
        }
    }
    
    public void Dispose()
    {
        _transaction?.Dispose();
        _context.Dispose();
    }
}

// การใช้งาน
public class OrderApplicationService
{
    private readonly IUnitOfWork _uow;
    
    public OrderApplicationService(IUnitOfWork uow) => _uow = uow;
    
    public async Task<Order> PlaceOrderAsync(PlaceOrderCommand cmd)
    {
        await _uow.BeginTransactionAsync();
        
        // All operations use the same DbContext (same transaction)
        var customer = await _uow.Customers.GetByIdAsync(cmd.CustomerId)
            ?? throw new NotFoundException($"Customer {cmd.CustomerId} not found");
        
        var order = new Order
        {
            CustomerId = customer.Id,
            Lines = []
        };
        
        foreach (var item in cmd.Items)
        {
            var product = await _uow.Products.GetByIdAsync(item.ProductId)
                ?? throw new NotFoundException($"Product {item.ProductId} not found");
            
            if (product.Stock < item.Quantity)
                throw new InsufficientStockException(product.Name, item.Quantity);
            
            product.Stock -= item.Quantity;
            order.Lines.Add(new OrderLine(product.Id, product.Name, product.Price, item.Quantity));
        }
        
        order.TotalAmount = order.Lines.Sum(l => l.UnitPrice * l.Quantity);
        
        var savedOrder = await _uow.Orders.CreateAsync(order);
        
        await _uow.CommitTransactionAsync();
        return savedOrder;
    }
}
```

## ขั้นตอนที่ 649: สรุป Entity Framework Core

```
EF Core Architecture Overview
═══════════════════════════════════════════════════════════════

Your Code
  ↓
DbContext (DbSet<T> + ChangeTracker + Configuration)
  ↓
LINQ Queries → IQueryable<T>
  ↓
EF Core Query Pipeline:
  1. Expression Tree Analysis
  2. Query Translation (LINQ → SQL)
  3. Connection Management
  4. Parameter Binding
  ↓
SQL Query (logged, parameterized)
  ↓
Database (SQL Server, SQLite, PostgreSQL, MySQL, ...)
  ↓
Result Mapping → Entity Objects
  ↓
Change Tracking → SaveChanges() → UPDATE/INSERT/DELETE

═══════════════════════════════════════════════════════════════
Key Commands:
  dotnet ef migrations add <Name>   - สร้าง migration
  dotnet ef database update          - apply migrations
  dotnet ef database drop            - drop database
  dotnet ef migrations list          - list migrations
  dotnet ef migrations script        - generate SQL

Performance Tips:
  ✓ AsNoTracking() for read queries
  ✓ Select() specific columns
  ✓ AsSplitQuery() for multiple collections
  ✓ ExecuteUpdate/DeleteAsync() for bulk ops
  ✓ Compiled queries for hot paths
  ✓ EnableRetryOnFailure() in production
  ✗ Avoid N+1 queries (use Include)
  ✗ Avoid loading entire table into memory
═══════════════════════════════════════════════════════════════
```

## สรุป Part 23

ใน Part นี้คุณได้เรียนรู้:
- **DbContext**: กำหนดค่า entities, relationships, seed data
- **Fluent API**: Configuration ด้วย IEntityTypeConfiguration<T>
- **CRUD**: Create, Read, Update, Delete operations
- **Migrations**: สร้าง/apply migrations, generate SQL scripts
- **LINQ to Entities**: querying, grouping, projection, raw SQL
- **Transactions**: automatic, explicit, savepoints
- **Performance**: AsNoTracking, Split queries, bulk operations
- **Unit of Work**: pattern สำหรับ consistent transactions

## แบบฝึกหัด

1. สร้าง DbContext สำหรับ Blog system (Post, Comment, Tag, User)
2. เขียน migration เพื่อสร้างตารางทั้งหมด
3. Implement Repository pattern สำหรับ Post ด้วย GetAll, GetById, Create, Update, Delete
4. เขียน LINQ query หา top 5 posts ที่มี comments มากที่สุด
5. ใช้ ExecuteUpdateAsync เพื่อ archive posts ที่เก่ากว่า 1 ปี
