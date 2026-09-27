# Part 47: Advanced Database Patterns & Data Engineering

## Steps 1345-1384: Database Mastery for World-Class .NET Applications

---

## Step 1345: Entity Framework Core — Advanced Mapping

```csharp
// Advanced EF Core 9 features
// Owned entities, value objects, discriminator, complex types

// Value Object
public record Money(decimal Amount, string Currency)
{
    public static Money operator +(Money a, Money b)
    {
        if (a.Currency != b.Currency)
            throw new InvalidOperationException("Cannot add different currencies");
        return new Money(a.Amount + b.Amount, a.Currency);
    }
}

// Complex Type (EF Core 8+) — stored in same table, no identity
[ComplexType]
public class Address
{
    public string Street { get; set; } = string.Empty;
    public string City { get; set; } = string.Empty;
    public string PostalCode { get; set; } = string.Empty;
    public string Country { get; set; } = string.Empty;
}

// Entities
public class Order
{
    public Guid Id { get; private set; }
    public Guid CustomerId { get; private set; }
    public Address ShippingAddress { get; private set; } = new();
    public Money TotalAmount { get; private set; } = new(0, "THB");
    public OrderStatus Status { get; private set; }
    public List<OrderLine> Lines { get; private set; } = [];
    public DateTimeOffset CreatedAt { get; private set; }

    // Navigation property
    public Customer Customer { get; private set; } = null!;
}

public class OrderLine
{
    public Guid Id { get; private set; }
    public Guid OrderId { get; private set; }
    public Guid ProductId { get; private set; }
    public int Quantity { get; private set; }
    public Money UnitPrice { get; private set; } = new(0, "THB");

    // Backing field for computed property
    public Money LineTotal => new(UnitPrice.Amount * Quantity, UnitPrice.Currency);
}

// DbContext Configuration
public class AppDbContext(DbContextOptions<AppDbContext> options) : DbContext(options)
{
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<Customer> Customers => Set<Customer>();
    public DbSet<Product> Products => Set<Product>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
    }
}

// Fluent Configuration
public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.HasKey(o => o.Id);
        builder.Property(o => o.Id).ValueGeneratedNever();

        // Complex type mapping (EF Core 8)
        builder.ComplexProperty(o => o.ShippingAddress, address =>
        {
            address.Property(a => a.Street).HasMaxLength(200).IsRequired();
            address.Property(a => a.City).HasMaxLength(100).IsRequired();
            address.Property(a => a.PostalCode).HasMaxLength(20);
            address.Property(a => a.Country).HasMaxLength(100).IsRequired();
        });

        // Money value object → owned entity
        builder.OwnsOne(o => o.TotalAmount, money =>
        {
            money.Property(m => m.Amount).HasColumnName("total_amount")
                .HasPrecision(18, 4).IsRequired();
            money.Property(m => m.Currency).HasColumnName("total_currency")
                .HasMaxLength(3).IsRequired();
        });

        builder.Property(o => o.Status)
            .HasConversion<string>()
            .HasMaxLength(50);

        builder.HasOne(o => o.Customer)
            .WithMany(c => c.Orders)
            .HasForeignKey(o => o.CustomerId)
            .OnDelete(DeleteBehavior.Restrict);

        builder.HasMany(o => o.Lines)
            .WithOne()
            .HasForeignKey(l => l.OrderId)
            .OnDelete(DeleteBehavior.Cascade);

        // Indexes
        builder.HasIndex(o => o.CustomerId);
        builder.HasIndex(o => new { o.Status, o.CreatedAt });
        builder.HasIndex(o => o.CreatedAt);

        // Table name convention
        builder.ToTable("orders");

        // Concurrency token
        builder.Property<uint>("RowVersion").IsRowVersion();
    }
}

public class OrderLineConfiguration : IEntityTypeConfiguration<OrderLine>
{
    public void Configure(EntityTypeBuilder<OrderLine> builder)
    {
        builder.HasKey(l => l.Id);

        builder.OwnsOne(l => l.UnitPrice, money =>
        {
            money.Property(m => m.Amount).HasColumnName("unit_price").HasPrecision(18, 4);
            money.Property(m => m.Currency).HasColumnName("unit_currency").HasMaxLength(3);
        });

        // Computed column (database-level)
        builder.Property<decimal>("line_total")
            .HasComputedColumnSql("unit_price * quantity", stored: true);

        builder.ToTable("order_lines");
    }
}
```

---

## Step 1346: EF Core — Interceptors & Audit Trail

```csharp
// SaveChanges Interceptor for automatic audit
public class AuditInterceptor(IHttpContextAccessor httpContextAccessor)
    : SaveChangesInterceptor
{
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData,
        InterceptionResult<int> result,
        CancellationToken ct = default)
    {
        if (eventData.Context is null) return base.SavingChangesAsync(eventData, result, ct);

        var userId = httpContextAccessor.HttpContext?.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        var now = DateTimeOffset.UtcNow;

        foreach (var entry in eventData.Context.ChangeTracker.Entries<IAuditable>())
        {
            switch (entry.State)
            {
                case EntityState.Added:
                    entry.Entity.CreatedAt = now;
                    entry.Entity.CreatedBy = userId ?? "system";
                    entry.Entity.UpdatedAt = now;
                    entry.Entity.UpdatedBy = userId ?? "system";
                    break;

                case EntityState.Modified:
                    entry.Entity.UpdatedAt = now;
                    entry.Entity.UpdatedBy = userId ?? "system";
                    // Prevent modification of create fields
                    entry.Property(nameof(IAuditable.CreatedAt)).IsModified = false;
                    entry.Property(nameof(IAuditable.CreatedBy)).IsModified = false;
                    break;
            }
        }

        return base.SavingChangesAsync(eventData, result, ct);
    }
}

// Soft delete interceptor
public class SoftDeleteInterceptor : SaveChangesInterceptor
{
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData,
        InterceptionResult<int> result,
        CancellationToken ct = default)
    {
        if (eventData.Context is null) return base.SavingChangesAsync(eventData, result, ct);

        foreach (var entry in eventData.Context.ChangeTracker.Entries<ISoftDeletable>()
            .Where(e => e.State == EntityState.Deleted))
        {
            entry.State = EntityState.Modified;
            entry.Entity.IsDeleted = true;
            entry.Entity.DeletedAt = DateTimeOffset.UtcNow;
        }

        return base.SavingChangesAsync(eventData, result, ct);
    }
}

// Query filter for soft delete (applied globally)
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    foreach (var entityType in modelBuilder.Model.GetEntityTypes())
    {
        if (entityType.ClrType.IsAssignableTo(typeof(ISoftDeletable)))
        {
            var method = typeof(AppDbContext)
                .GetMethod(nameof(SetSoftDeleteFilter), BindingFlags.NonPublic | BindingFlags.Static)!
                .MakeGenericMethod(entityType.ClrType);
            method.Invoke(null, [modelBuilder]);
        }
    }
}

private static void SetSoftDeleteFilter<T>(ModelBuilder modelBuilder)
    where T : class, ISoftDeletable
    => modelBuilder.Entity<T>().HasQueryFilter(e => !e.IsDeleted);

// Command Interceptor — slow query logging
public class SlowQueryInterceptor(ILogger<SlowQueryInterceptor> logger)
    : DbCommandInterceptor
{
    private static readonly TimeSpan SlowThreshold = TimeSpan.FromMilliseconds(500);

    public override async ValueTask<DbDataReader> ReaderExecutedAsync(
        DbCommand command,
        CommandExecutedEventData eventData,
        DbDataReader result,
        CancellationToken ct = default)
    {
        if (eventData.Duration > SlowThreshold)
        {
            logger.LogWarning(
                "Slow query detected ({Duration}ms): {CommandText}",
                eventData.Duration.TotalMilliseconds,
                command.CommandText[..Math.Min(500, command.CommandText.Length)]);
        }
        return await base.ReaderExecutedAsync(command, eventData, result, ct);
    }
}
```

---

## Step 1347: Raw SQL, Dapper & Hybrid Access

```csharp
// Hybrid: EF Core for writes, Dapper for complex reads

// Install: Dapper, Npgsql

public class OrderReadRepository(NpgsqlDataSource dataSource)
{
    // Complex JOIN query with Dapper
    public async Task<List<OrderSummaryDto>> GetOrderSummariesAsync(
        Guid customerId,
        int pageSize,
        int pageNumber,
        CancellationToken ct)
    {
        await using var conn = await dataSource.OpenConnectionAsync(ct);
        return (await conn.QueryAsync<OrderSummaryDto>(
            """
            SELECT
                o.id,
                o.status,
                o.total_amount,
                o.created_at,
                COUNT(ol.id) AS item_count,
                c.name AS customer_name,
                c.email AS customer_email,
                SUM(ol.quantity) AS total_items
            FROM orders o
            JOIN customers c ON c.id = o.customer_id
            JOIN order_lines ol ON ol.order_id = o.id
            WHERE o.customer_id = @customerId
              AND o.is_deleted = false
            GROUP BY o.id, o.status, o.total_amount, o.created_at, c.name, c.email
            ORDER BY o.created_at DESC
            LIMIT @pageSize OFFSET @offset
            """,
            new
            {
                customerId,
                pageSize,
                offset = (pageNumber - 1) * pageSize,
            })).ToList();
    }

    // Dapper multi-mapping (JOIN → nested objects)
    public async Task<OrderDetailDto?> GetOrderDetailAsync(Guid orderId, CancellationToken ct)
    {
        await using var conn = await dataSource.OpenConnectionAsync(ct);

        OrderDetailDto? order = null;
        await conn.QueryAsync<OrderDetailDto, OrderLineDto, OrderDetailDto>(
            """
            SELECT
                o.id, o.status, o.total_amount, o.created_at, o.shipping_address_street,
                o.shipping_address_city, o.shipping_address_country,
                ol.id AS line_id, ol.product_id, ol.quantity, ol.unit_price
            FROM orders o
            JOIN order_lines ol ON ol.order_id = o.id
            WHERE o.id = @orderId
            """,
            (o, line) =>
            {
                order ??= o;
                order.Lines.Add(line);
                return order;
            },
            new { orderId },
            splitOn: "line_id");

        return order;
    }

    // EF Core FromSql for complex queries that need tracking
    public async Task<List<Order>> GetOrdersWithRelatedAsync(
        AppDbContext db, DateTimeOffset since, CancellationToken ct)
    {
        // Parameterized — safe from SQL injection
        return await db.Orders
            .FromSql($"""
                SELECT o.*
                FROM orders o
                WHERE o.created_at >= {since}
                  AND EXISTS (
                    SELECT 1 FROM order_lines ol
                    WHERE ol.order_id = o.id AND ol.quantity > 10
                  )
                """)
            .Include(o => o.Lines)
            .Include(o => o.Customer)
            .ToListAsync(ct);
    }
}
```

---

## Step 1348: Database Migrations Strategy

```csharp
// Migration helper for zero-downtime deployments
public static class MigrationExtensions
{
    public static async Task MigrateDatabaseAsync(this IHost host, CancellationToken ct = default)
    {
        using var scope = host.Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        var logger = scope.ServiceProvider.GetRequiredService<ILogger<AppDbContext>>();

        try
        {
            logger.LogInformation("Applying database migrations...");
            var pending = await db.Database.GetPendingMigrationsAsync(ct);
            var pendingList = pending.ToList();

            if (pendingList.Count == 0)
            {
                logger.LogInformation("No pending migrations");
                return;
            }

            logger.LogInformation("Applying {Count} pending migrations: {Migrations}",
                pendingList.Count, string.Join(", ", pendingList));

            await db.Database.MigrateAsync(ct);
            logger.LogInformation("Migrations applied successfully");
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "Migration failed");
            throw;
        }
    }
}

// Program.cs
if (args.Contains("--migrate"))
{
    await app.MigrateDatabaseAsync();
    return;
}
await app.RunAsync();
```

```csharp
// Zero-downtime migration example
// Migration: add column with default, then make NOT NULL in separate migration

// Step 1: Add nullable column
public partial class AddOrderPriority : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        // Add nullable first — no lock on PostgreSQL for nullable adds
        migrationBuilder.AddColumn<string>(
            name: "priority",
            table: "orders",
            type: "varchar(20)",
            nullable: true);
    }
}

// Step 2: Backfill data (separate deployment)
public partial class BackfillOrderPriority : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.Sql("UPDATE orders SET priority = 'normal' WHERE priority IS NULL");
    }
}

// Step 3: Make NOT NULL (separate deployment after backfill completes)
public partial class MakeOrderPriorityRequired : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.AlterColumn<string>(
            name: "priority",
            table: "orders",
            type: "varchar(20)",
            nullable: false,
            defaultValue: "normal",
            oldClrType: typeof(string),
            oldNullable: true);
    }
}

// Non-blocking index creation (PostgreSQL CONCURRENTLY)
public partial class AddIndexOnOrderStatus : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        // CONCURRENTLY doesn't lock the table during index build
        migrationBuilder.Sql(
            "CREATE INDEX CONCURRENTLY IF NOT EXISTS ix_orders_status_created " +
            "ON orders(status, created_at DESC)");
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.Sql("DROP INDEX CONCURRENTLY IF EXISTS ix_orders_status_created");
    }
}
```

---

## Step 1349: PostgreSQL Advanced Features

```csharp
// PostgreSQL-specific features via EF Core + Npgsql

// 1. JSONB columns
public class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        // Store attributes as JSONB
        builder.Property(p => p.Attributes)
            .HasColumnType("jsonb")
            .HasConversion(
                v => JsonSerializer.Serialize(v, (JsonSerializerOptions?)null),
                v => JsonSerializer.Deserialize<Dictionary<string, object>>(v, (JsonSerializerOptions?)null) ?? new());

        // PostgreSQL array type
        builder.Property(p => p.Tags)
            .HasColumnType("text[]");

        // Full-text search vector
        builder.Property(p => p.SearchVector)
            .HasColumnType("tsvector")
            .ValueGeneratedOnAddOrUpdate()
            .HasComputedColumnSql(
                "to_tsvector('english', coalesce(name,'') || ' ' || coalesce(description,''))",
                stored: true);

        builder.HasIndex(p => p.SearchVector)
            .HasMethod("GIN");
    }
}

// 2. Full-text search query
public async Task<List<Product>> SearchProductsAsync(
    AppDbContext db, string searchTerm, CancellationToken ct)
{
    // EF Core 8+ — native full-text search support
    return await db.Products
        .Where(p => EF.Functions.ToTsVector("english",
            p.Name + " " + (p.Description ?? ""))
            .Matches(EF.Functions.PlainToTsQuery("english", searchTerm)))
        .OrderByDescending(p => EF.Functions.ToTsVector("english",
            p.Name + " " + (p.Description ?? ""))
            .RankCoverDensity(EF.Functions.PlainToTsQuery("english", searchTerm)))
        .Take(20)
        .ToListAsync(ct);
}

// 3. JSONB queries
public async Task<List<Product>> GetProductsByAttributeAsync(
    AppDbContext db, string attributeKey, string attributeValue, CancellationToken ct)
{
    // Queries inside JSONB column using PostgreSQL operators
    return await db.Products
        .Where(p => EF.Functions.JsonContains(
            EF.Property<string>(p, "attributes_json"),
            JsonSerializer.Serialize(new Dictionary<string, string> { [attributeKey] = attributeValue })))
        .ToListAsync(ct);
}

// 4. PostgreSQL SKIP LOCKED — job queue pattern
public async Task<List<JobRecord>> DequeueJobsAsync(
    AppDbContext db, int batchSize, CancellationToken ct)
{
    var jobs = await db.Jobs
        .FromSql($"""
            SELECT * FROM jobs
            WHERE status = 'pending'
            ORDER BY created_at
            LIMIT {batchSize}
            FOR UPDATE SKIP LOCKED
            """)
        .ToListAsync(ct);

    foreach (var job in jobs)
    {
        job.Status = "processing";
        job.StartedAt = DateTimeOffset.UtcNow;
    }
    await db.SaveChangesAsync(ct);
    return jobs;
}

// 5. Window functions via raw SQL
public async Task<List<OrderRankDto>> GetTopCustomerOrdersAsync(
    NpgsqlDataSource ds, CancellationToken ct)
{
    await using var conn = await ds.OpenConnectionAsync(ct);
    return (await conn.QueryAsync<OrderRankDto>(
        """
        SELECT
            customer_id,
            total_amount,
            ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY total_amount DESC) AS rank,
            RANK() OVER (ORDER BY total_amount DESC) AS overall_rank,
            SUM(total_amount) OVER (PARTITION BY customer_id) AS customer_total,
            AVG(total_amount) OVER () AS avg_order_value
        FROM orders
        WHERE status = 'completed'
        ORDER BY overall_rank
        LIMIT 100
        """)).ToList();
}
```

---

## Step 1350: Repository Pattern with Specifications

```csharp
// Specification Pattern — encapsulate query logic
public interface ISpecification<T>
{
    Expression<Func<T, bool>> Criteria { get; }
    List<Expression<Func<T, object>>> Includes { get; }
    List<string> IncludeStrings { get; }
    Expression<Func<T, object>>? OrderBy { get; }
    Expression<Func<T, object>>? OrderByDescending { get; }
    int Take { get; }
    int Skip { get; }
    bool IsPagingEnabled { get; }
    bool AsNoTracking { get; }
}

public abstract class BaseSpecification<T>(Expression<Func<T, bool>> criteria) : ISpecification<T>
{
    public Expression<Func<T, bool>> Criteria => criteria;
    public List<Expression<Func<T, object>>> Includes { get; } = [];
    public List<string> IncludeStrings { get; } = [];
    public Expression<Func<T, object>>? OrderBy { get; private set; }
    public Expression<Func<T, object>>? OrderByDescending { get; private set; }
    public int Take { get; private set; }
    public int Skip { get; private set; }
    public bool IsPagingEnabled { get; private set; }
    public bool AsNoTracking { get; private set; }

    protected void AddInclude(Expression<Func<T, object>> includeExpression)
        => Includes.Add(includeExpression);

    protected void AddInclude(string includeString)
        => IncludeStrings.Add(includeString);

    protected void ApplyOrderBy(Expression<Func<T, object>> orderByExpression)
        => OrderBy = orderByExpression;

    protected void ApplyOrderByDescending(Expression<Func<T, object>> orderByDescExpression)
        => OrderByDescending = orderByDescExpression;

    protected void ApplyPaging(int skip, int take)
    {
        Skip = skip;
        Take = take;
        IsPagingEnabled = true;
    }

    protected void ApplyNoTracking()
        => AsNoTracking = true;
}

// Concrete specifications
public class OrdersForCustomerSpec : BaseSpecification<Order>
{
    public OrdersForCustomerSpec(Guid customerId) : base(o => o.CustomerId == customerId)
    {
        AddInclude(o => o.Lines);
        AddInclude(o => o.Customer);
        ApplyOrderByDescending(o => o.CreatedAt);
        ApplyNoTracking();
    }
}

public class PendingOrdersSpec : BaseSpecification<Order>
{
    public PendingOrdersSpec(int pageNumber, int pageSize)
        : base(o => o.Status == OrderStatus.Pending)
    {
        ApplyOrderBy(o => o.CreatedAt);
        ApplyPaging((pageNumber - 1) * pageSize, pageSize);
        ApplyNoTracking();
    }
}

// Generic repository
public class EfRepository<T>(AppDbContext db) : IRepository<T> where T : class, IEntity
{
    public async Task<T?> GetByIdAsync(Guid id, CancellationToken ct)
        => await db.Set<T>().FindAsync([id], ct);

    public async Task<T?> GetBySpecAsync(ISpecification<T> spec, CancellationToken ct)
        => await ApplySpecification(spec).FirstOrDefaultAsync(ct);

    public async Task<List<T>> ListAsync(ISpecification<T> spec, CancellationToken ct)
        => await ApplySpecification(spec).ToListAsync(ct);

    public async Task<int> CountAsync(ISpecification<T> spec, CancellationToken ct)
        => await ApplySpecification(spec).CountAsync(ct);

    public async Task AddAsync(T entity, CancellationToken ct)
    {
        await db.Set<T>().AddAsync(entity, ct);
        await db.SaveChangesAsync(ct);
    }

    public async Task UpdateAsync(T entity, CancellationToken ct)
    {
        db.Set<T>().Update(entity);
        await db.SaveChangesAsync(ct);
    }

    public async Task DeleteAsync(T entity, CancellationToken ct)
    {
        db.Set<T>().Remove(entity);
        await db.SaveChangesAsync(ct);
    }

    private IQueryable<T> ApplySpecification(ISpecification<T> spec)
    {
        var query = db.Set<T>().AsQueryable();

        if (spec.AsNoTracking)
            query = query.AsNoTracking();

        if (spec.Criteria != null)
            query = query.Where(spec.Criteria);

        query = spec.Includes.Aggregate(query, (current, include) => current.Include(include));
        query = spec.IncludeStrings.Aggregate(query, (current, include) => current.Include(include));

        if (spec.OrderBy != null)
            query = query.OrderBy(spec.OrderBy);
        else if (spec.OrderByDescending != null)
            query = query.OrderByDescending(spec.OrderByDescending);

        if (spec.IsPagingEnabled)
            query = query.Skip(spec.Skip).Take(spec.Take);

        return query;
    }
}
```

---

## Step 1351: Unit of Work Pattern

```csharp
public interface IUnitOfWork : IAsyncDisposable
{
    IRepository<Order> Orders { get; }
    IRepository<Customer> Customers { get; }
    IRepository<Product> Products { get; }
    Task<int> SaveChangesAsync(CancellationToken ct = default);
    Task BeginTransactionAsync(CancellationToken ct = default);
    Task CommitTransactionAsync(CancellationToken ct = default);
    Task RollbackTransactionAsync(CancellationToken ct = default);
}

public class UnitOfWork(AppDbContext db) : IUnitOfWork
{
    private IDbContextTransaction? _transaction;

    public IRepository<Order> Orders { get; } = new EfRepository<Order>(db);
    public IRepository<Customer> Customers { get; } = new EfRepository<Customer>(db);
    public IRepository<Product> Products { get; } = new EfRepository<Product>(db);

    public Task<int> SaveChangesAsync(CancellationToken ct = default)
        => db.SaveChangesAsync(ct);

    public async Task BeginTransactionAsync(CancellationToken ct = default)
        => _transaction = await db.Database.BeginTransactionAsync(ct);

    public async Task CommitTransactionAsync(CancellationToken ct = default)
    {
        await db.SaveChangesAsync(ct);
        if (_transaction is not null)
            await _transaction.CommitAsync(ct);
    }

    public async Task RollbackTransactionAsync(CancellationToken ct = default)
    {
        if (_transaction is not null)
            await _transaction.RollbackAsync(ct);
    }

    public async ValueTask DisposeAsync()
    {
        if (_transaction is not null)
            await _transaction.DisposeAsync();
        await db.DisposeAsync();
        GC.SuppressFinalize(this);
    }
}

// Usage
public class TransferFundsHandler(IUnitOfWork uow) : IRequestHandler<TransferFundsCommand>
{
    public async Task Handle(TransferFundsCommand request, CancellationToken ct)
    {
        await uow.BeginTransactionAsync(ct);
        try
        {
            var from = await uow.Accounts.GetByIdAsync(request.FromAccountId, ct)
                ?? throw new NotFoundException("Source account not found");
            var to = await uow.Accounts.GetByIdAsync(request.ToAccountId, ct)
                ?? throw new NotFoundException("Destination account not found");

            if (from.Balance < request.Amount)
                throw new InsufficientFundsException();

            from.Debit(request.Amount);
            to.Credit(request.Amount);

            await uow.Accounts.UpdateAsync(from, ct);
            await uow.Accounts.UpdateAsync(to, ct);
            await uow.CommitTransactionAsync(ct);
        }
        catch
        {
            await uow.RollbackTransactionAsync(ct);
            throw;
        }
    }
}
```

---

## Step 1352: Database Sharding & Multi-tenancy

```csharp
// Multi-tenancy strategies
// 1. Separate databases per tenant
// 2. Separate schemas per tenant
// 3. Shared schema with tenant column (row-level isolation)

// Strategy 3: Shared schema with global query filter
public class TenantDbContext(
    DbContextOptions<TenantDbContext> options,
    ITenantContext tenantContext) : DbContext(options)
{
    public DbSet<Order> Orders => Set<Order>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        var tenantId = tenantContext.TenantId;

        // Global query filter — all queries automatically scoped to tenant
        modelBuilder.Entity<Order>()
            .HasQueryFilter(o => o.TenantId == tenantId);

        // Composite index: tenant + primary key for performance
        modelBuilder.Entity<Order>()
            .HasIndex(o => new { o.TenantId, o.CreatedAt });
    }

    public override Task<int> SaveChangesAsync(CancellationToken ct = default)
    {
        // Auto-set tenant ID on insert
        foreach (var entry in ChangeTracker.Entries<ITenantEntity>()
            .Where(e => e.State == EntityState.Added))
        {
            entry.Entity.TenantId = tenantContext.TenantId;
        }
        return base.SaveChangesAsync(ct);
    }
}

// Tenant context from HTTP request
public class HttpTenantContext(IHttpContextAccessor httpContextAccessor) : ITenantContext
{
    public Guid TenantId =>
        Guid.TryParse(httpContextAccessor.HttpContext?.User.FindFirst("tenant_id")?.Value, out var id)
            ? id
            : throw new UnauthorizedAccessException("Missing tenant context");
}

// Strategy 1: Database per tenant with DbContext factory
public class TenantDbContextFactory(
    IConfiguration config,
    ITenantResolver resolver)
{
    public AppDbContext CreateForTenant(Guid tenantId)
    {
        var connectionString = resolver.GetConnectionString(tenantId);
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseNpgsql(connectionString,
                npgsql => npgsql.MigrationsHistoryTable("__ef_migrations", $"tenant_{tenantId:N}"))
            .Options;
        return new AppDbContext(options);
    }
}

// Horizontal sharding by hash
public class ShardRouter(List<string> connectionStrings)
{
    public string GetConnectionString(Guid entityId)
    {
        // Consistent hashing
        var hash = (int)(entityId.GetHashCode() & 0x7FFFFFFF);
        var shardIndex = hash % connectionStrings.Count;
        return connectionStrings[shardIndex];
    }
}
```

---

## Step 1353: Read Replicas & Connection Pooling

```csharp
// Read replica routing
public class ReadWriteDbContextFactory(IConfiguration config)
{
    private readonly string _writeConnectionString = config.GetConnectionString("WriteDB")!;
    private readonly List<string> _readConnectionStrings =
        config.GetSection("ReadDBs").Get<List<string>>() ?? [];

    private int _readIndex = 0;

    public AppDbContext CreateWriteContext()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseNpgsql(_writeConnectionString,
                npgsql => npgsql.CommandTimeout(30))
            .AddInterceptors(new AuditInterceptor(), new SlowQueryInterceptor())
            .Options;
        return new AppDbContext(options);
    }

    public AppDbContext CreateReadContext()
    {
        if (_readConnectionStrings.Count == 0)
            return CreateWriteContext();

        // Round-robin across replicas
        var index = Interlocked.Increment(ref _readIndex) % _readConnectionStrings.Count;
        var connStr = _readConnectionStrings[index];

        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseNpgsql(connStr)
            .UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking) // Read-only
            .Options;
        return new AppDbContext(options);
    }
}

// Pgbouncer connection pooling configuration
// pgbouncer.ini
/*
[databases]
orders = host=postgres-primary port=5432 dbname=orders
orders_replica = host=postgres-replica port=5432 dbname=orders

[pgbouncer]
pool_mode = transaction        ; transaction-level pooling
max_client_conn = 10000
default_pool_size = 100
min_pool_size = 10
reserve_pool_size = 5
server_lifetime = 3600
server_idle_timeout = 600
client_idle_timeout = 0
log_connections = 0
log_disconnections = 0
*/

// Npgsql connection pooling settings in connection string
public static class NpgsqlConnectionStringBuilder
{
    public static string Build(string host, string database, string username, string password)
    {
        var builder = new NpgsqlConnectionStringBuilder
        {
            Host = host,
            Database = database,
            Username = username,
            Password = password,
            Port = 5432,
            // Connection pool settings
            MinPoolSize = 5,
            MaxPoolSize = 100,
            ConnectionIdleLifetime = 300, // 5 minutes
            ConnectionPruningInterval = 10,
            // Performance
            NoResetOnClose = false,
            CommandTimeout = 30,
            // Resilience  
            KeepAlive = 60,
            TcpKeepAlive = true,
        };
        return builder.ConnectionString;
    }
}
```

---

## Step 1354: Elasticsearch Integration

```xml
<ItemGroup>
  <PackageReference Include="Elastic.Clients.Elasticsearch" Version="8.*" />
</ItemGroup>
```

```csharp
// Elasticsearch service for full-text search and analytics
public class ProductSearchService(ElasticsearchClient client)
{
    private const string IndexName = "products";

    public async Task IndexProductAsync(ProductDocument product, CancellationToken ct)
    {
        var response = await client.IndexAsync(product, i => i
            .Index(IndexName)
            .Id(product.Id.ToString())
            .Routing(product.CategoryId.ToString()), ct);

        if (!response.IsSuccess())
            throw new ElasticsearchException($"Index failed: {response.ElasticsearchServerError}");
    }

    public async Task BulkIndexProductsAsync(IEnumerable<ProductDocument> products, CancellationToken ct)
    {
        var response = await client.BulkAsync(b => b
            .Index(IndexName)
            .IndexMany(products), ct);

        if (response.Errors)
        {
            var errors = response.ItemsWithErrors.Select(i => i.Error?.Reason).ToList();
            throw new ElasticsearchException($"Bulk index errors: {string.Join("; ", errors)}");
        }
    }

    public async Task<SearchResult<ProductDocument>> SearchAsync(
        ProductSearchRequest request, CancellationToken ct)
    {
        var response = await client.SearchAsync<ProductDocument>(s => s
            .Index(IndexName)
            .From((request.Page - 1) * request.PageSize)
            .Size(request.PageSize)
            .Query(q => q
                .Bool(b =>
                {
                    // Full-text search across multiple fields
                    if (!string.IsNullOrWhiteSpace(request.Query))
                    {
                        b.Must(m => m
                            .MultiMatch(mm => mm
                                .Query(request.Query)
                                .Fields(new[] { "name^3", "description^1", "tags^2" })
                                .Type(TextQueryType.BestFields)
                                .Fuzziness(new Fuzziness("AUTO"))
                                .MinimumShouldMatch("75%")));
                    }

                    // Filters (cached, don't affect scoring)
                    var filters = new List<Action<QueryDescriptor<ProductDocument>>>();
                    if (request.CategoryId.HasValue)
                        filters.Add(f => f.Term(t => t.Field(p => p.CategoryId).Value(request.CategoryId.Value)));
                    if (request.MinPrice.HasValue)
                        filters.Add(f => f.Range(r => r.NumberRange(nr => nr.Field(p => p.Price).Gte(request.MinPrice))));
                    if (request.MaxPrice.HasValue)
                        filters.Add(f => f.Range(r => r.NumberRange(nr => nr.Field(p => p.Price).Lte(request.MaxPrice))));
                    if (request.InStockOnly)
                        filters.Add(f => f.Term(t => t.Field(p => p.InStock).Value(true)));

                    if (filters.Any())
                        b.Filter(filters.ToArray());
                }))
            .Sort(ss =>
            {
                if (!string.IsNullOrWhiteSpace(request.Query))
                    ss.Score(sc => sc.Order(SortOrder.Desc)); // Relevance first when searching
                else
                    ss.Field(f => f.Field(p => p.CreatedAt).Order(SortOrder.Desc));
                return ss;
            })
            .Aggregations(a => a
                .Terms("categories", t => t.Field(p => p.CategoryId).Size(20))
                .Range("price_ranges", r => r
                    .Field(p => p.Price)
                    .Ranges(
                        new NumberRangeExpression { To = 100 },
                        new NumberRangeExpression { From = 100, To = 500 },
                        new NumberRangeExpression { From = 500, To = 1000 },
                        new NumberRangeExpression { From = 1000 }))
                .Stats("price_stats", st => st.Field(p => p.Price))),
            ct);

        if (!response.IsSuccess())
            throw new ElasticsearchException($"Search failed: {response.ElasticsearchServerError}");

        return new SearchResult<ProductDocument>
        {
            Items = response.Documents.ToList(),
            TotalCount = (int)(response.Total ?? 0),
            Aggregations = ParseAggregations(response.Aggregations),
        };
    }

    public async Task DeleteProductAsync(Guid productId, CancellationToken ct)
    {
        await client.DeleteAsync<ProductDocument>(
            productId.ToString(), d => d.Index(IndexName), ct);
    }
}

// Registration
builder.Services.AddSingleton(sp =>
{
    var uri = new Uri(builder.Configuration["Elasticsearch:Uri"]!);
    var settings = new ElasticsearchClientSettings(uri)
        .DefaultIndex("products")
        .EnableDebugMode()
        .PrettyJson();

    return new ElasticsearchClient(settings);
});
```

---

## Step 1355: Redis Advanced Patterns

```csharp
// Advanced Redis patterns beyond basic caching
public class RedisAdvancedService(IConnectionMultiplexer redis, ILogger<RedisAdvancedService> logger)
{
    private readonly IDatabase _db = redis.GetDatabase();
    private readonly ISubscriber _sub = redis.GetSubscriber();

    // 1. Distributed rate limiting with sliding window
    public async Task<bool> IsRateLimitedAsync(
        string identifier, int maxRequests, TimeSpan window, CancellationToken ct)
    {
        var now = DateTimeOffset.UtcNow;
        var windowStart = now - window;
        var key = $"ratelimit:{identifier}";

        var result = await _db.SortedSetRangeByScoreWithScoresAsync(key,
            start: windowStart.ToUnixTimeMilliseconds(),
            stop: now.ToUnixTimeMilliseconds());

        var requestCount = result.Length;
        if (requestCount >= maxRequests)
        {
            logger.LogWarning("Rate limit exceeded for {Identifier}: {Count}/{Max}",
                identifier, requestCount, maxRequests);
            return true; // Rate limited
        }

        var tx = _db.CreateTransaction();
        _ = tx.SortedSetAddAsync(key, Guid.NewGuid().ToString(), now.ToUnixTimeMilliseconds());
        _ = tx.SortedSetRemoveRangeByScoreAsync(key, 0, windowStart.ToUnixTimeMilliseconds());
        _ = tx.KeyExpireAsync(key, window);
        await tx.ExecuteAsync();
        return false;
    }

    // 2. Session storage
    public async Task SetSessionAsync(string sessionId, UserSession session, TimeSpan ttl, CancellationToken ct)
    {
        var key = $"session:{sessionId}";
        var value = JsonSerializer.Serialize(session);
        await _db.StringSetAsync(key, value, ttl);
    }

    public async Task<UserSession?> GetSessionAsync(string sessionId, CancellationToken ct)
    {
        var key = $"session:{sessionId}";
        var value = await _db.StringGetAsync(key);
        return value.HasValue ? JsonSerializer.Deserialize<UserSession>(value!) : null;
    }

    // 3. Pub/Sub for real-time notifications
    public async Task PublishOrderUpdateAsync(Guid orderId, string status)
    {
        var channel = new RedisChannel($"order-updates:{orderId}", RedisChannel.PatternMode.Literal);
        var message = JsonSerializer.Serialize(new { OrderId = orderId, Status = status, Timestamp = DateTimeOffset.UtcNow });
        await _sub.PublishAsync(channel, message);
    }

    public async Task SubscribeToOrderUpdatesAsync(
        Guid orderId, Func<string, Task> handler, CancellationToken ct)
    {
        var channel = new RedisChannel($"order-updates:{orderId}", RedisChannel.PatternMode.Literal);
        await _sub.SubscribeAsync(channel, async (_, message) =>
        {
            if (message.HasValue)
                await handler(message!);
        });
    }

    // 4. Bloom filter (approximate membership test)
    public async Task<bool> MightContainAsync(string filterKey, string item)
    {
        // Uses RedisBloom module
        var script = """
            local exists = redis.call('BF.EXISTS', KEYS[1], ARGV[1])
            return exists
            """;
        var result = (int)await _db.ScriptEvaluateAsync(script, [$"{filterKey}"], [$"{item}"]);
        return result == 1;
    }

    // 5. Leaderboard with sorted sets
    public async Task UpdateLeaderboardAsync(string leaderboardKey, Guid userId, double score)
    {
        await _db.SortedSetAddAsync(leaderboardKey, userId.ToString(), score);
    }

    public async Task<List<LeaderboardEntry>> GetLeaderboardAsync(
        string leaderboardKey, int top, CancellationToken ct)
    {
        var entries = await _db.SortedSetRangeByRankWithScoresAsync(
            leaderboardKey, 0, top - 1, Order.Descending);

        return entries.Select((entry, index) => new LeaderboardEntry(
            Rank: index + 1,
            UserId: Guid.Parse(entry.Element.ToString()),
            Score: entry.Score
        )).ToList();
    }

    // 6. HyperLogLog for unique visitor counting
    public async Task RecordVisitAsync(string pageKey, string visitorId)
    {
        await _db.HyperLogLogAddAsync(pageKey, visitorId);
    }

    public async Task<long> GetUniqueVisitorCountAsync(string pageKey)
    {
        return await _db.HyperLogLogLengthAsync(pageKey);
    }
}

public record LeaderboardEntry(int Rank, Guid UserId, double Score);
public record UserSession(Guid UserId, string Role, Dictionary<string, string> Claims, DateTimeOffset ExpiresAt);
```

---

## Step 1356: Time Series Data with TimescaleDB

```csharp
// TimescaleDB (PostgreSQL extension) for time series
// Usage: metrics, IoT data, financial data, logs

public class MetricsRepository(NpgsqlDataSource dataSource)
{
    public async Task InsertMetricAsync(MetricPoint metric, CancellationToken ct)
    {
        await using var conn = await dataSource.OpenConnectionAsync(ct);
        await conn.ExecuteAsync(
            """
            INSERT INTO metrics (time, service, metric_name, value, tags)
            VALUES (@Time, @Service, @MetricName, @Value, @Tags::jsonb)
            """,
            new
            {
                Time = metric.Timestamp,
                metric.Service,
                metric.MetricName,
                metric.Value,
                Tags = JsonSerializer.Serialize(metric.Tags),
            });
    }

    public async Task BulkInsertMetricsAsync(IEnumerable<MetricPoint> metrics, CancellationToken ct)
    {
        await using var conn = await dataSource.OpenConnectionAsync(ct);
        await using var writer = await conn.BeginBinaryImportAsync(
            "COPY metrics (time, service, metric_name, value, tags) FROM STDIN (FORMAT BINARY)", ct);

        foreach (var metric in metrics)
        {
            await writer.StartRowAsync(ct);
            await writer.WriteAsync(metric.Timestamp, NpgsqlDbType.TimestampTz, ct);
            await writer.WriteAsync(metric.Service, NpgsqlDbType.Text, ct);
            await writer.WriteAsync(metric.MetricName, NpgsqlDbType.Text, ct);
            await writer.WriteAsync(metric.Value, NpgsqlDbType.Double, ct);
            await writer.WriteAsync(JsonSerializer.Serialize(metric.Tags), NpgsqlDbType.Jsonb, ct);
        }

        await writer.CompleteAsync(ct);
    }

    // TimescaleDB time_bucket function for aggregation
    public async Task<List<MetricAggregation>> GetHourlyAveragesAsync(
        string serviceName, string metricName,
        DateTimeOffset from, DateTimeOffset to,
        CancellationToken ct)
    {
        await using var conn = await dataSource.OpenConnectionAsync(ct);
        return (await conn.QueryAsync<MetricAggregation>(
            """
            SELECT
                time_bucket('1 hour', time) AS bucket,
                avg(value) AS avg_value,
                min(value) AS min_value,
                max(value) AS max_value,
                count(*) AS sample_count,
                percentile_cont(0.95) WITHIN GROUP (ORDER BY value) AS p95
            FROM metrics
            WHERE service = @service
              AND metric_name = @metricName
              AND time BETWEEN @from AND @to
            GROUP BY bucket
            ORDER BY bucket
            """,
            new { service = serviceName, metricName, from, to })).ToList();
    }
}

// CREATE TABLE metrics (
//   time        TIMESTAMPTZ NOT NULL,
//   service     TEXT NOT NULL,
//   metric_name TEXT NOT NULL,
//   value       DOUBLE PRECISION NOT NULL,
//   tags        JSONB
// );
// SELECT create_hypertable('metrics', 'time', chunk_time_interval => INTERVAL '1 day');
// CREATE INDEX ON metrics (service, metric_name, time DESC);

public record MetricPoint(
    DateTimeOffset Timestamp,
    string Service,
    string MetricName,
    double Value,
    Dictionary<string, string> Tags
);

public record MetricAggregation(
    DateTimeOffset Bucket,
    double AvgValue,
    double MinValue,
    double MaxValue,
    long SampleCount,
    double P95
);
```

---

## Step 1357: MongoDB Integration

```xml
<ItemGroup>
  <PackageReference Include="MongoDB.Driver" Version="3.*" />
</ItemGroup>
```

```csharp
// MongoDB for document storage (e.g., product catalog, CMS content)
public class MongoProductRepository(IMongoDatabase db)
{
    private readonly IMongoCollection<ProductDocument> _collection =
        db.GetCollection<ProductDocument>("products");

    public async Task<ProductDocument?> GetByIdAsync(Guid id, CancellationToken ct)
        => await _collection
            .Find(p => p.Id == id)
            .FirstOrDefaultAsync(ct);

    public async Task<List<ProductDocument>> SearchAsync(
        string? query, Guid? categoryId, int page, int pageSize, CancellationToken ct)
    {
        var builder = Builders<ProductDocument>.Filter;
        var filter = builder.Empty;

        if (!string.IsNullOrWhiteSpace(query))
            filter &= builder.Text(query);

        if (categoryId.HasValue)
            filter &= builder.Eq(p => p.CategoryId, categoryId.Value);

        return await _collection
            .Find(filter)
            .Sort(Builders<ProductDocument>.Sort.Descending(p => p.CreatedAt))
            .Skip((page - 1) * pageSize)
            .Limit(pageSize)
            .ToListAsync(ct);
    }

    public async Task UpsertAsync(ProductDocument product, CancellationToken ct)
    {
        var filter = Builders<ProductDocument>.Filter.Eq(p => p.Id, product.Id);
        var options = new ReplaceOptions { IsUpsert = true };
        await _collection.ReplaceOneAsync(filter, product, options, ct);
    }

    public async Task UpdateFieldsAsync(
        Guid id,
        Dictionary<string, object> updates,
        CancellationToken ct)
    {
        var filter = Builders<ProductDocument>.Filter.Eq(p => p.Id, id);
        var updateDefs = updates.Select(kvp =>
            Builders<ProductDocument>.Update.Set(kvp.Key, kvp.Value));
        var update = Builders<ProductDocument>.Update
            .Combine(updateDefs)
            .Set(p => p.UpdatedAt, DateTimeOffset.UtcNow);

        await _collection.UpdateOneAsync(filter, update, null, ct);
    }

    // Aggregation pipeline
    public async Task<List<CategoryStats>> GetCategoryStatsAsync(CancellationToken ct)
    {
        var pipeline = new[]
        {
            new BsonDocument("$group", new BsonDocument
            {
                { "_id", "$categoryId" },
                { "count", new BsonDocument("$sum", 1) },
                { "avgPrice", new BsonDocument("$avg", "$price") },
                { "minPrice", new BsonDocument("$min", "$price") },
                { "maxPrice", new BsonDocument("$max", "$price") },
            }),
            new BsonDocument("$sort", new BsonDocument("count", -1)),
            new BsonDocument("$limit", 50),
        };

        return await _collection
            .Aggregate<CategoryStats>(pipeline)
            .ToListAsync(ct);
    }
}

[BsonCollection("products")]
public class ProductDocument
{
    [BsonId]
    [BsonRepresentation(BsonType.String)]
    public Guid Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string? Description { get; set; }
    public decimal Price { get; set; }
    public Guid CategoryId { get; set; }
    public List<string> Tags { get; set; } = [];
    public Dictionary<string, object> Attributes { get; set; } = new();
    public bool InStock { get; set; }
    public DateTimeOffset CreatedAt { get; set; }
    public DateTimeOffset? UpdatedAt { get; set; }
}

// Registration
builder.Services.AddSingleton<IMongoClient>(sp =>
{
    var settings = MongoClientSettings.FromConnectionString(
        builder.Configuration.GetConnectionString("MongoDB"));
    settings.ServerApi = new ServerApi(ServerApiVersion.V1);
    return new MongoClient(settings);
});
builder.Services.AddScoped(sp =>
    sp.GetRequiredService<IMongoClient>().GetDatabase("ecommerce"));
```

---

## Step 1358: Database Performance & Query Optimization

```csharp
// 1. Compiled queries — precompile LINQ to SQL once
public static class CompiledQueries
{
    public static readonly Func<AppDbContext, Guid, Task<Order?>> GetOrderById =
        EF.CompileAsyncQuery((AppDbContext db, Guid id) =>
            db.Orders
                .Include(o => o.Lines)
                .FirstOrDefault(o => o.Id == id));

    public static readonly Func<AppDbContext, Guid, IAsyncEnumerable<OrderSummary>>
        GetOrderSummariesForCustomer =
        EF.CompileAsyncQuery((AppDbContext db, Guid customerId) =>
            db.Orders
                .Where(o => o.CustomerId == customerId)
                .Select(o => new OrderSummary(o.Id, o.Status, o.TotalAmount, o.CreatedAt))
                .OrderByDescending(o => o.CreatedAt)
                .AsNoTracking());
}

// Usage
var order = await CompiledQueries.GetOrderById(db, orderId);

// 2. Split queries for large includes (avoids cartesian explosion)
var orders = await db.Orders
    .Include(o => o.Lines)
    .Include(o => o.Comments)
    .AsSplitQuery() // Runs multiple SELECT instead of one JOIN
    .Where(o => o.CustomerId == customerId)
    .ToListAsync(ct);

// 3. Projection to avoid loading unused columns
var summaries = await db.Orders
    .Where(o => o.Status == OrderStatus.Pending)
    .Select(o => new OrderSummary(
        o.Id,
        o.Status,
        o.Lines.Sum(l => l.Quantity * l.UnitPrice.Amount),
        o.CreatedAt))
    .ToListAsync(ct);

// 4. Batch operations with EF Core 9 ExecuteUpdate / ExecuteDelete
// Zero entity loading — direct SQL UPDATE
var affected = await db.Orders
    .Where(o => o.CreatedAt < DateTimeOffset.UtcNow.AddDays(-90)
        && o.Status == OrderStatus.Cancelled)
    .ExecuteDeleteAsync(ct);

await db.Orders
    .Where(o => o.Status == OrderStatus.Pending
        && o.CreatedAt < DateTimeOffset.UtcNow.AddHours(-24))
    .ExecuteUpdateAsync(s => s
        .SetProperty(o => o.Status, OrderStatus.Expired)
        .SetProperty(o => o.UpdatedAt, DateTimeOffset.UtcNow),
        ct);

// 5. Keyset pagination (faster than OFFSET for large datasets)
public async Task<KeysetPage<Order>> GetOrdersPageAsync(
    AppDbContext db,
    Guid? afterId,
    DateTimeOffset? afterCreatedAt,
    int pageSize,
    CancellationToken ct)
{
    var query = db.Orders.AsNoTracking();

    if (afterId.HasValue && afterCreatedAt.HasValue)
    {
        // Keyset: get rows after the cursor
        query = query.Where(o =>
            o.CreatedAt < afterCreatedAt.Value
            || (o.CreatedAt == afterCreatedAt.Value && o.Id < afterId.Value));
    }

    var items = await query
        .OrderByDescending(o => o.CreatedAt)
        .ThenByDescending(o => o.Id)
        .Take(pageSize + 1) // Fetch one extra to determine hasMore
        .ToListAsync(ct);

    var hasMore = items.Count > pageSize;
    if (hasMore) items.RemoveAt(items.Count - 1);

    return new KeysetPage<Order>(items, hasMore,
        hasMore ? items.Last().Id : null,
        hasMore ? items.Last().CreatedAt : null);
}

public record KeysetPage<T>(List<T> Items, bool HasMore, Guid? NextId, DateTimeOffset? NextCreatedAt);

// 6. Database seeding with bulk insert
public static class DataSeeder
{
    public static async Task SeedAsync(AppDbContext db, CancellationToken ct)
    {
        if (await db.Products.AnyAsync(ct)) return;

        var products = Enumerable.Range(1, 10000).Select(i => new Product
        {
            Id = Guid.NewGuid(),
            Name = $"Product {i}",
            Price = Random.Shared.Next(100, 10000) / 100m,
            InStock = Random.Shared.NextDouble() > 0.2,
            CreatedAt = DateTimeOffset.UtcNow.AddDays(-Random.Shared.Next(0, 365)),
        }).ToList();

        // Bulk insert via EF Core — better than AddRangeAsync for large datasets
        await db.BulkInsertAsync(products, new BulkConfig { BatchSize = 1000 }, ct);
    }
}
```

---

## Step 1359: Database Testing

```csharp
// EF Core In-Memory (fast, no real SQL)
public class InMemoryOrderTests
{
    private AppDbContext CreateContext()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseInMemoryDatabase(databaseName: Guid.NewGuid().ToString())
            .Options;
        return new AppDbContext(options);
    }

    [Fact]
    public async Task CreateOrder_SavesCorrectly()
    {
        await using var db = CreateContext();
        var order = new Order { Id = Guid.NewGuid(), CustomerId = Guid.NewGuid(), Status = OrderStatus.Pending };
        db.Orders.Add(order);
        await db.SaveChangesAsync();

        var saved = await db.Orders.FindAsync(order.Id);
        saved.Should().NotBeNull();
        saved!.Status.Should().Be(OrderStatus.Pending);
    }
}

// Testcontainers — real PostgreSQL (recommended for integration tests)
public class PostgresOrderTests : IAsyncLifetime
{
    private PostgreSqlContainer _container = null!;
    private AppDbContext _db = null!;

    public async Task InitializeAsync()
    {
        _container = new PostgreSqlBuilder()
            .WithImage("postgres:16-alpine")
            .WithDatabase("orders_test")
            .Build();
        await _container.StartAsync();

        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseNpgsql(_container.GetConnectionString())
            .Options;
        _db = new AppDbContext(options);
        await _db.Database.MigrateAsync();
    }

    [Fact]
    public async Task GetOrdersByCustomer_ReturnsOnlyThatCustomer()
    {
        var customerId = Guid.NewGuid();
        var otherId = Guid.NewGuid();

        _db.Orders.AddRange(
            new Order { Id = Guid.NewGuid(), CustomerId = customerId, Status = OrderStatus.Pending, CreatedAt = DateTimeOffset.UtcNow },
            new Order { Id = Guid.NewGuid(), CustomerId = customerId, Status = OrderStatus.Completed, CreatedAt = DateTimeOffset.UtcNow },
            new Order { Id = Guid.NewGuid(), CustomerId = otherId, Status = OrderStatus.Pending, CreatedAt = DateTimeOffset.UtcNow }
        );
        await _db.SaveChangesAsync();

        var orders = await _db.Orders
            .Where(o => o.CustomerId == customerId)
            .ToListAsync();

        orders.Should().HaveCount(2);
        orders.Should().AllSatisfy(o => o.CustomerId.Should().Be(customerId));
    }

    public async Task DisposeAsync()
    {
        await _db.DisposeAsync();
        await _container.DisposeAsync();
    }
}

// Respawn — fast database reset between tests
public class DatabaseFixture : IAsyncLifetime
{
    private Respawner _respawner = null!;
    public AppDbContext Db { get; private set; } = null!;
    public NpgsqlConnection Connection { get; private set; } = null!;

    public async Task InitializeAsync()
    {
        Connection = new NpgsqlConnection(TestConnectionString);
        await Connection.OpenAsync();

        Db = CreateContext();
        await Db.Database.MigrateAsync();

        _respawner = await Respawner.CreateAsync(Connection, new RespawnerOptions
        {
            DbAdapter = DbAdapter.Postgres,
            TablesToIgnore = [new Table("__EFMigrationsHistory")],
        });
    }

    public async Task ResetAsync()
        => await _respawner.ResetAsync(Connection);

    public async Task DisposeAsync()
    {
        await Db.DisposeAsync();
        await Connection.DisposeAsync();
    }
}
```

---

## Step 1360: Summary — Database Patterns Checklist

```
DATABASE PATTERNS REFERENCE
════════════════════════════

EF CORE ADVANCED
├── Complex Types (EF Core 8) — same table, no PK
├── Owned Entities — Value Objects mapping
├── Global Query Filters — soft delete, multi-tenancy
├── Interceptors — audit, slow query logging
├── Compiled Queries — precompile LINQ for hot paths
├── Split Queries — avoid cartesian explosion
├── ExecuteUpdate/Delete — bulk ops without loading
└── Keyset Pagination — O(log n) vs OFFSET O(n)

DATA ACCESS PATTERNS
├── Repository Pattern with Specifications
├── Unit of Work for transactions
├── CQRS Read/Write separation
├── Read Replicas for query scaling
└── Cache-Aside (L1 in-memory + L2 Redis)

DATABASE ENGINES
├── PostgreSQL — primary relational, JSONB, full-text
├── Redis — cache, sessions, rate limiting, pub/sub
├── Elasticsearch — full-text search, analytics
├── MongoDB — document storage, flexible schema
├── TimescaleDB — time series (metrics, IoT)
└── SQL Server — enterprise .NET default

PERFORMANCE
├── Connection Pooling (Npgsql, PgBouncer)
├── Index strategy (B-tree, GIN, partial indexes)
├── Non-blocking index creation (CONCURRENTLY)
├── Bulk inserts (COPY, ExecuteInsertAsync)
└── Materialized views for heavy aggregations

MULTI-TENANCY
├── Separate DB per tenant (isolation, cost)
├── Schema per tenant (moderate isolation)
├── Row-level isolation + query filters (simplest)
└── Sharding (horizontal scale)

TESTING
├── In-Memory — fast unit tests
├── Testcontainers + real PostgreSQL — integration
├── Respawn — fast DB reset between tests
└── Pact — contract tests for DB consumers
```

---

## สรุป Part 47

Part นี้ครอบคลุม Advanced Database Patterns อย่างครบถ้วน:

**Steps 1345-1360:**
- **EF Core Advanced Mapping** — complex types, owned entities, fluent configuration
- **Interceptors** — audit trail, soft delete, slow query logging
- **Raw SQL & Dapper** — hybrid access, multi-mapping, window functions
- **Zero-Downtime Migrations** — add nullable → backfill → make required, CONCURRENTLY index
- **PostgreSQL Features** — JSONB, full-text search, SKIP LOCKED, arrays
- **Repository + Specification Pattern** — generic repository with predicate/include/sort/paging
- **Unit of Work** — transaction management across repositories
- **Multi-tenancy** — row-level isolation, database-per-tenant, sharding
- **Read Replicas** — round-robin routing, PgBouncer configuration
- **Elasticsearch** — full-text search, aggregations, bulk indexing
- **Redis Advanced** — sliding window rate limiting, sessions, pub/sub, HyperLogLog, leaderboard
- **TimescaleDB** — time series data, hypertables, time_bucket aggregation
- **MongoDB** — document storage, aggregation pipeline, upsert
- **Query Optimization** — compiled queries, split queries, batch ExecuteUpdate/Delete, keyset pagination
- **Database Testing** — in-memory, Testcontainers, Respawn for fast reset

เนื้อหาทั้งหมดเป็น Production-ready code ที่ใช้ได้จริงใน .NET 9.0
