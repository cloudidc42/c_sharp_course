# Part 86: Advanced EF Core — Interceptors, Query Filters & Bulk Operations

## Steps 2005–2020

---

## Step 2005: EF Core Interceptors Overview

Interceptors hook into EF Core's internal pipeline — similar to MediatR behaviors but for database operations.

```
Interceptor types:
├── IDbCommandInterceptor       — SQL command execution (query/non-query)
├── IDbConnectionInterceptor    — connection open/close
├── IDbTransactionInterceptor   — transaction begin/commit/rollback
├── ISaveChangesInterceptor     — SaveChanges/SaveChangesAsync
├── IMaterializationInterceptor — entity materialization from query results
├── IQueryExpressionInterceptor — LINQ expression tree manipulation
└── IDbContextOptionsExtensionsInterceptor — options setup
```

---

## Step 2006: Command Interceptor — Query Logging

```csharp
// Infrastructure/Interceptors/SlowQueryInterceptor.cs
using Microsoft.EntityFrameworkCore.Diagnostics;

namespace Orders.Infrastructure.Interceptors;

public class SlowQueryInterceptor : DbCommandInterceptor
{
    private readonly ILogger<SlowQueryInterceptor> _logger;
    private readonly TimeSpan _threshold;

    public SlowQueryInterceptor(
        ILogger<SlowQueryInterceptor> logger,
        TimeSpan? threshold = null)
    {
        _logger = logger;
        _threshold = threshold ?? TimeSpan.FromMilliseconds(500);
    }

    public override async ValueTask<DbDataReader> ReaderExecutedAsync(
        DbCommand command,
        CommandExecutedEventData eventData,
        DbDataReader result,
        CancellationToken ct = default)
    {
        LogIfSlow(command, eventData.Duration);
        return await base.ReaderExecutedAsync(command, eventData, result, ct);
    }

    public override DbDataReader ReaderExecuted(
        DbCommand command,
        CommandExecutedEventData eventData,
        DbDataReader result)
    {
        LogIfSlow(command, eventData.Duration);
        return base.ReaderExecuted(command, eventData, result);
    }

    private void LogIfSlow(DbCommand command, TimeSpan duration)
    {
        if (duration < _threshold) return;

        _logger.LogWarning(
            "Slow query detected ({Duration}ms):\n{Sql}\nParameters: {Parameters}",
            duration.TotalMilliseconds,
            command.CommandText,
            string.Join(", ", command.Parameters.Cast<DbParameter>()
                .Select(p => $"{p.ParameterName}={p.Value}")));
    }
}
```

---

## Step 2007: SaveChanges Interceptor — Soft Delete & Audit

```csharp
// Infrastructure/Interceptors/AuditingInterceptor.cs
public class AuditingInterceptor : SaveChangesInterceptor
{
    private readonly ICurrentUser _currentUser;
    private readonly IDateTime _dateTime;

    public AuditingInterceptor(ICurrentUser currentUser, IDateTime dateTime)
    {
        _currentUser = currentUser;
        _dateTime = dateTime;
    }

    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData,
        InterceptionResult<int> result,
        CancellationToken ct = default)
    {
        if (eventData.Context is null)
            return base.SavingChangesAsync(eventData, result, ct);

        var now = _dateTime.UtcNow;
        var userId = _currentUser.UserId;

        foreach (var entry in eventData.Context.ChangeTracker.Entries())
        {
            switch (entry)
            {
                case { Entity: IAuditable auditable, State: EntityState.Added }:
                    auditable.CreatedAt = now;
                    auditable.CreatedBy = userId;
                    auditable.UpdatedAt = now;
                    auditable.UpdatedBy = userId;
                    break;

                case { Entity: IAuditable auditable, State: EntityState.Modified }:
                    entry.Property(nameof(IAuditable.CreatedAt)).IsModified = false;
                    entry.Property(nameof(IAuditable.CreatedBy)).IsModified = false;
                    auditable.UpdatedAt = now;
                    auditable.UpdatedBy = userId;
                    break;

                case { Entity: ISoftDeletable deletable, State: EntityState.Deleted }:
                    // Intercept hard delete → soft delete
                    entry.State = EntityState.Modified;
                    deletable.DeletedAt = now;
                    deletable.DeletedBy = userId;
                    deletable.IsDeleted = true;
                    break;
            }
        }

        return base.SavingChangesAsync(eventData, result, ct);
    }
}

// Interfaces
public interface IAuditable
{
    DateTimeOffset CreatedAt { get; set; }
    string? CreatedBy { get; set; }
    DateTimeOffset UpdatedAt { get; set; }
    string? UpdatedBy { get; set; }
}

public interface ISoftDeletable
{
    bool IsDeleted { get; set; }
    DateTimeOffset? DeletedAt { get; set; }
    string? DeletedBy { get; set; }
}
```

---

## Step 2008: Domain Event Publishing Interceptor

```csharp
// Infrastructure/Interceptors/DomainEventPublishingInterceptor.cs
public class DomainEventPublishingInterceptor : SaveChangesInterceptor
{
    private readonly IPublishEndpoint _publisher;
    private readonly ILogger<DomainEventPublishingInterceptor> _logger;

    public DomainEventPublishingInterceptor(
        IPublishEndpoint publisher,
        ILogger<DomainEventPublishingInterceptor> logger)
    {
        _publisher = publisher;
        _logger = logger;
    }

    public override async ValueTask<int> SavedChangesAsync(
        SaveChangesCompletedEventData eventData,
        int result,
        CancellationToken ct = default)
    {
        if (eventData.Context is not null)
            await PublishDomainEventsAsync(eventData.Context, ct);

        return await base.SavedChangesAsync(eventData, result, ct);
    }

    private async Task PublishDomainEventsAsync(DbContext context, CancellationToken ct)
    {
        var entitiesWithEvents = context.ChangeTracker
            .Entries<Entity>()
            .Where(e => e.Entity.DomainEvents.Any())
            .Select(e => e.Entity)
            .ToList();

        foreach (var entity in entitiesWithEvents)
        {
            var events = entity.DomainEvents.ToArray();
            entity.ClearDomainEvents();

            foreach (var @event in events)
            {
                _logger.LogDebug("Publishing domain event {EventType} for entity {EntityId}",
                    @event.GetType().Name, entity.Id);

                await _publisher.Publish(@event, @event.GetType(), ct);
            }
        }
    }
}
```

---

## Step 2009: Connection Interceptor — Multi-Tenancy RLS

```csharp
// Infrastructure/Interceptors/TenantSchemaInterceptor.cs
public class TenantSchemaInterceptor : DbConnectionInterceptor
{
    private readonly ITenantContextAccessor _tenantAccessor;
    private readonly ILogger<TenantSchemaInterceptor> _logger;

    public TenantSchemaInterceptor(
        ITenantContextAccessor tenantAccessor,
        ILogger<TenantSchemaInterceptor> logger)
    {
        _tenantAccessor = tenantAccessor;
        _logger = logger;
    }

    public override async Task ConnectionOpenedAsync(
        DbConnection connection,
        ConnectionEndEventData eventData,
        CancellationToken ct = default)
    {
        var tenant = _tenantAccessor.Current;
        if (tenant is null) return;

        try
        {
            await using var cmd = connection.CreateCommand();
            // PostgreSQL row-level security — set app context variable
            cmd.CommandText = "SELECT set_config('app.current_tenant', @tenantId, false)";
            var param = cmd.CreateParameter();
            param.ParameterName = "@tenantId";
            param.Value = tenant.TenantId;
            cmd.Parameters.Add(param);
            await cmd.ExecuteNonQueryAsync(ct);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to set tenant context on connection");
            throw;
        }
    }
}
```

---

## Step 2010: Global Query Filters

```csharp
// DbContext — apply query filters globally
protected override void OnModelCreating(ModelBuilder model)
{
    // Soft delete filter — all queries exclude deleted records
    foreach (var entityType in model.Model.GetEntityTypes())
    {
        if (typeof(ISoftDeletable).IsAssignableFrom(entityType.ClrType))
        {
            model.Entity(entityType.ClrType)
                .HasQueryFilter(BuildSoftDeleteFilter(entityType.ClrType));
        }

        if (typeof(ITenantEntity).IsAssignableFrom(entityType.ClrType))
        {
            model.Entity(entityType.ClrType)
                .HasQueryFilter(BuildTenantFilter(entityType.ClrType));
        }
    }
}

private static LambdaExpression BuildSoftDeleteFilter(Type entityType)
{
    var parameter = Expression.Parameter(entityType, "e");
    var property = Expression.Property(parameter, nameof(ISoftDeletable.IsDeleted));
    var notDeleted = Expression.Not(property);
    return Expression.Lambda(notDeleted, parameter);
}

private LambdaExpression BuildTenantFilter(Type entityType)
{
    var parameter = Expression.Parameter(entityType, "e");
    var tenantIdProperty = Expression.Property(parameter, nameof(ITenantEntity.TenantId));
    var currentTenantId = Expression.Property(
        Expression.Constant(this), // captures 'this' context
        typeof(AppDbContext).GetProperty(nameof(CurrentTenantId))!);

    var filter = Expression.Equal(tenantIdProperty, currentTenantId);
    return Expression.Lambda(filter, parameter);
}

// Expose current tenant for the filter lambda
private Guid? CurrentTenantId => _tenantAccessor.Current?.TenantId;
```

```csharp
// Bypass filters when needed
var includingDeleted = await _db.Orders
    .IgnoreQueryFilters()  // bypasses ALL query filters
    .Where(o => o.IsDeleted)
    .ToListAsync(ct);

// Restore deleted entity
var order = await _db.Orders.IgnoreQueryFilters()
    .FirstAsync(o => o.Id == orderId, ct);
order.IsDeleted = false;
order.DeletedAt = null;
await _db.SaveChangesAsync(ct);
```

---

## Step 2011: Owned Entities & Value Objects

```csharp
// Domain/ValueObjects/Money.cs
public sealed class Money : ValueObject
{
    public decimal Amount { get; }
    public string Currency { get; }

    private Money() { }

    public Money(decimal amount, string currency)
    {
        if (amount < 0) throw new DomainException("Amount cannot be negative");
        if (string.IsNullOrEmpty(currency) || currency.Length != 3)
            throw new DomainException("Currency must be a 3-letter code");

        Amount = amount;
        Currency = currency.ToUpperInvariant();
    }

    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new DomainException($"Cannot add {Currency} and {other.Currency}");
        return new Money(Amount + other.Amount, Currency);
    }

    public Money Multiply(decimal factor) => new(Amount * factor, Currency);

    protected override IEnumerable<object?> GetEqualityComponents()
    {
        yield return Amount;
        yield return Currency;
    }

    public override string ToString() => $"{Amount:F2} {Currency}";
}
```

```csharp
// Configuration — EF Core owned entity
builder.Entity<Order>(e =>
{
    e.OwnsOne(o => o.Total, money =>
    {
        money.Property(m => m.Amount)
            .HasColumnName("TotalAmount")
            .HasColumnType("decimal(18,4)")
            .IsRequired();
        money.Property(m => m.Currency)
            .HasColumnName("TotalCurrency")
            .HasMaxLength(3)
            .IsFixedLength()
            .IsRequired();
    });

    // Owned collection
    e.OwnsMany(o => o.LineItems, item =>
    {
        item.WithOwner().HasForeignKey("OrderId");
        item.HasKey("Id");
        item.OwnsOne(i => i.Price, price =>
        {
            price.Property(p => p.Amount).HasColumnName("UnitPrice");
            price.Property(p => p.Currency).HasColumnName("Currency");
        });
    });
});
```

---

## Step 2012: JSON Columns (EF Core 7+)

```csharp
// Store complex objects as JSON in a single column
public class Order
{
    public Guid Id { get; set; }
    public OrderStatus Status { get; set; }

    // Stored as JSON column
    public ShippingInfo Shipping { get; set; } = null!;
    public IList<OrderTag> Tags { get; set; } = [];
    public OrderMetadata? Metadata { get; set; }
}

public class ShippingInfo
{
    public string Carrier { get; set; } = null!;
    public string TrackingNumber { get; set; } = null!;
    public Address Destination { get; set; } = null!;
    public DateTimeOffset? EstimatedDelivery { get; set; }
}

public record OrderTag(string Name, string Value);

public class OrderMetadata
{
    public string? SourceChannel { get; set; }
    public string? CampaignId { get; set; }
    public Dictionary<string, string> CustomFields { get; set; } = [];
}
```

```csharp
// Configuration
builder.Entity<Order>(e =>
{
    e.OwnsOne(o => o.Shipping, ship =>
    {
        ship.ToJson("shipping_info"); // serialize to JSON column

        ship.OwnsOne(s => s.Destination, addr =>
        {
            // Nested JSON — automatic
        });
    });

    e.OwnsMany(o => o.Tags, tag =>
    {
        tag.ToJson("tags"); // list stored as JSON array
    });

    e.OwnsOne(o => o.Metadata, meta =>
    {
        meta.ToJson("metadata");
    });
});
```

```csharp
// Query against JSON columns
var expressOrders = await _db.Orders
    .Where(o => o.Shipping.Carrier == "FedEx")
    .ToListAsync(ct);

var campaignOrders = await _db.Orders
    .Where(o => o.Metadata != null && o.Metadata.CampaignId == "SUMMER24")
    .ToListAsync(ct);

// JSON_CONTAINS-style via Any()
var taggedOrders = await _db.Orders
    .Where(o => o.Tags.Any(t => t.Name == "priority" && t.Value == "high"))
    .ToListAsync(ct);
```

---

## Step 2013: Compiled Models for Startup Performance

```bash
# Generate compiled model — eliminates model building overhead at startup
dotnet ef dbcontext optimize --output-dir CompiledModels --namespace Orders.Infrastructure.CompiledModels
```

```csharp
// Program.cs — use compiled model
builder.Services.AddDbContext<OrdersDbContext>(opts =>
{
    opts.UseNpgsql(connectionString);
    opts.UseModel(OrdersDbContextModel.Instance); // pre-built model
});
```

---

## Step 2014: EF Core Bulk Extensions

```bash
dotnet add package EFCore.BulkExtensions
```

```csharp
// EFCore.BulkExtensions — bypass EF for large operations
using EFCore.BulkExtensions;

public class ProductImportService
{
    private readonly AppDbContext _db;

    public async Task BulkImportAsync(
        IReadOnlyList<Product> products, CancellationToken ct = default)
    {
        var bulkConfig = new BulkConfig
        {
            BatchSize = 500,
            UseTempDB = false,
            SetOutputIdentity = false,
            CalculateStats = false,
            UpdateByProperties = [nameof(Product.Sku)] // upsert by SKU
        };

        // Insert or update — single SQL MERGE / INSERT ON CONFLICT
        await _db.BulkInsertOrUpdateAsync(products.ToList(), bulkConfig, cancellationToken: ct);
    }

    public async Task BulkDeleteExpiredAsync(DateTimeOffset cutoff, CancellationToken ct)
    {
        await _db.BulkDeleteAsync(
            await _db.Products
                .Where(p => p.ExpiresAt < cutoff)
                .ToListAsync(ct),
            cancellationToken: ct);
    }

    public async Task<List<Product>> BulkReadAsync(
        IReadOnlyList<Guid> ids, CancellationToken ct = default)
    {
        var tempTable = new List<TempTable>(ids.Select(id => new TempTable { Id = id }));

        await _db.BulkInsertAsync(tempTable, cancellationToken: ct);

        return await _db.Products
            .Join(_db.Set<TempTable>(), p => p.Id, t => t.Id, (p, _) => p)
            .ToListAsync(ct);
    }
}
```

---

## Step 2015: Raw SQL with Dapper for Complex Queries

```bash
dotnet add package Dapper
```

```csharp
// Infrastructure/Queries/OrderQueryService.cs
using Dapper;

public class OrderQueryService
{
    private readonly IDbConnectionFactory _connectionFactory;

    public OrderQueryService(IDbConnectionFactory connectionFactory)
    {
        _connectionFactory = connectionFactory;
    }

    public async Task<PagedResult<OrderReportRow>> GetOrderReportAsync(
        OrderReportFilter filter, CancellationToken ct = default)
    {
        await using var conn = await _connectionFactory.OpenAsync(ct);

        const string sql = """
            SELECT
                o.id AS Id,
                c.full_name AS CustomerName,
                o.status AS Status,
                o.total_amount AS TotalAmount,
                COUNT(oi.id) AS ItemCount,
                o.created_at AS CreatedAt
            FROM orders o
            INNER JOIN customers c ON c.id = o.customer_id
            LEFT JOIN order_items oi ON oi.order_id = o.id
            WHERE (@CustomerId::uuid IS NULL OR o.customer_id = @CustomerId)
              AND (@Status IS NULL OR o.status = @Status)
              AND o.created_at >= @FromDate
              AND o.created_at < @ToDate
              AND o.is_deleted = FALSE
            GROUP BY o.id, c.full_name
            ORDER BY o.created_at DESC
            OFFSET @Offset ROWS FETCH NEXT @PageSize ROWS ONLY
            """;

        const string countSql = """
            SELECT COUNT(*) FROM orders o
            WHERE (@CustomerId::uuid IS NULL OR o.customer_id = @CustomerId)
              AND (@Status IS NULL OR o.status = @Status)
              AND o.created_at >= @FromDate
              AND o.created_at < @ToDate
              AND o.is_deleted = FALSE
            """;

        var parameters = new
        {
            filter.CustomerId,
            filter.Status,
            filter.FromDate,
            filter.ToDate,
            Offset = (filter.Page - 1) * filter.PageSize,
            filter.PageSize
        };

        var cmd = new CommandDefinition(sql, parameters, cancellationToken: ct);
        var countCmd = new CommandDefinition(countSql, parameters, cancellationToken: ct);

        var [rows, total] = await Task.WhenAll(
            conn.QueryAsync<OrderReportRow>(cmd),
            conn.ExecuteScalarAsync<int>(countCmd));

        return new PagedResult<OrderReportRow>(
            [.. rows], total, filter.Page, filter.PageSize);
    }

    public async Task<OrderDetailReport?> GetDetailAsync(Guid orderId, CancellationToken ct)
    {
        await using var conn = await _connectionFactory.OpenAsync(ct);

        const string sql = """
            SELECT
                o.id, o.status, o.total_amount, o.created_at,
                oi.id AS ItemId, oi.sku, oi.quantity, oi.unit_price,
                p.name AS ProductName
            FROM orders o
            LEFT JOIN order_items oi ON oi.order_id = o.id
            LEFT JOIN products p ON p.id = oi.product_id
            WHERE o.id = @OrderId AND o.is_deleted = FALSE
            """;

        var rows = await conn.QueryAsync<OrderDetailRow, OrderItemRow, OrderDetailRow>(
            new CommandDefinition(sql, new { OrderId = orderId }, cancellationToken: ct),
            (order, item) => { order.Items.Add(item); return order; },
            splitOn: "ItemId");

        return rows.GroupBy(r => r.Id)
            .Select(g =>
            {
                var first = g.First();
                first.Items = g.SelectMany(r => r.Items).ToList();
                return first;
            })
            .FirstOrDefault();
    }
}
```

---

## Step 2016: Database Connection Factory

```csharp
// Infrastructure/Data/IDbConnectionFactory.cs
public interface IDbConnectionFactory
{
    Task<IDbConnection> OpenAsync(CancellationToken ct = default);
    IDbConnection Create();
}

public class NpgsqlConnectionFactory : IDbConnectionFactory
{
    private readonly string _connectionString;

    public NpgsqlConnectionFactory(string connectionString)
    {
        _connectionString = connectionString;
    }

    public async Task<IDbConnection> OpenAsync(CancellationToken ct = default)
    {
        var connection = new NpgsqlConnection(_connectionString);
        await connection.OpenAsync(ct);
        return connection;
    }

    public IDbConnection Create() => new NpgsqlConnection(_connectionString);
}

// Registration
builder.Services.AddSingleton<IDbConnectionFactory>(
    new NpgsqlConnectionFactory(
        builder.Configuration.GetConnectionString("Postgres")!));
```

---

## Step 2017: Database Migrations Strategy

```csharp
// Migrations/MigrationRunner.cs
public class MigrationRunner
{
    private readonly AppDbContext _db;
    private readonly ILogger<MigrationRunner> _logger;

    public MigrationRunner(AppDbContext db, ILogger<MigrationRunner> logger)
    {
        _db = db;
        _logger = logger;
    }

    public async Task RunAsync(CancellationToken ct = default)
    {
        _logger.LogInformation("Running database migrations...");

        var pendingMigrations = await _db.Database
            .GetPendingMigrationsAsync(ct);

        var migrations = pendingMigrations.ToList();
        if (!migrations.Any())
        {
            _logger.LogInformation("No pending migrations");
            return;
        }

        _logger.LogInformation("Applying {Count} migrations: {Names}",
            migrations.Count, string.Join(", ", migrations));

        await _db.Database.MigrateAsync(ct);

        _logger.LogInformation("Migrations applied successfully");
    }
}
```

```bash
# Common migration commands
dotnet ef migrations add AddOrderStatusIndex \
  --project src/Orders.Infrastructure \
  --startup-project src/Orders.Api \
  --output-dir Migrations

# Generate SQL script for review before applying
dotnet ef migrations script --idempotent \
  --project src/Orders.Infrastructure \
  --startup-project src/Orders.Api \
  --output migrations.sql

# Apply specific migration
dotnet ef database update AddOrderStatusIndex \
  --project src/Orders.Infrastructure \
  --startup-project src/Orders.Api
```

---

## Step 2018: Database Seeding

```csharp
// Infrastructure/Data/Seeders/DatabaseSeeder.cs
public class DatabaseSeeder
{
    private readonly AppDbContext _db;
    private readonly ILogger<DatabaseSeeder> _logger;
    private readonly IWebHostEnvironment _env;

    public DatabaseSeeder(
        AppDbContext db,
        ILogger<DatabaseSeeder> logger,
        IWebHostEnvironment env)
    {
        _db = db;
        _logger = logger;
        _env = env;
    }

    public async Task SeedAsync(CancellationToken ct = default)
    {
        await SeedRolesAsync(ct);
        await SeedAdminUserAsync(ct);

        if (_env.IsDevelopment())
            await SeedTestDataAsync(ct);
    }

    private async Task SeedRolesAsync(CancellationToken ct)
    {
        var roles = new[] { "Admin", "Manager", "Customer" };
        foreach (var roleName in roles)
        {
            if (!await _db.Roles.AnyAsync(r => r.Name == roleName, ct))
            {
                _db.Roles.Add(new Role { Id = Guid.NewGuid(), Name = roleName });
                _logger.LogInformation("Seeded role: {Role}", roleName);
            }
        }
        await _db.SaveChangesAsync(ct);
    }

    private async Task SeedTestDataAsync(CancellationToken ct)
    {
        if (await _db.Products.AnyAsync(ct)) return; // already seeded

        var products = Enumerable.Range(1, 50).Select(i => new Product
        {
            Id = Guid.NewGuid(),
            Name = $"Product {i:D3}",
            Sku = $"SKU-{i:D4}",
            Price = new Money(Random.Shared.Next(10, 500), "USD"),
            Stock = Random.Shared.Next(0, 100),
            IsActive = true,
            CreatedAt = DateTimeOffset.UtcNow
        }).ToList();

        await _db.Products.AddRangeAsync(products, ct);
        await _db.SaveChangesAsync(ct);

        _logger.LogInformation("Seeded {Count} test products", products.Count);
    }
}
```

---

## Step 2019: EF Core Concurrency Tokens

```csharp
// Optimistic concurrency — detect concurrent modifications
public class Order : AuditableEntity
{
    public Guid Id { get; set; }
    public OrderStatus Status { get; set; }
    public decimal TotalAmount { get; set; }

    // Concurrency token — automatically updated on each save
    [Timestamp]
    public byte[] RowVersion { get; set; } = null!;

    // Or use xmin for PostgreSQL
    // public uint XMin { get; set; }
}
```

```csharp
// Configuration
builder.Entity<Order>(e =>
{
    e.Property(o => o.RowVersion)
        .IsRowVersion()
        .IsConcurrencyToken();

    // PostgreSQL — use xmin system column
    // e.Property<uint>("xmin").HasColumnName("xmin").ValueGeneratedOnAddOrUpdate().IsConcurrencyToken();
});
```

```csharp
// Handle concurrency conflicts
public async Task ConfirmOrderAsync(Guid orderId, byte[] expectedVersion, CancellationToken ct)
{
    try
    {
        var order = await _db.Orders.FirstAsync(o => o.Id == orderId, ct);

        if (!order.RowVersion.SequenceEqual(expectedVersion))
            throw new ConcurrencyException("Order was modified by another process");

        order.Confirm();
        await _db.SaveChangesAsync(ct);
    }
    catch (DbUpdateConcurrencyException ex)
    {
        // EF Core detected stale data via RowVersion
        var entry = ex.Entries.Single();
        var databaseValues = await entry.GetDatabaseValuesAsync(ct);

        if (databaseValues is null)
            throw new NotFoundException("Order was deleted");

        // Refresh and retry
        entry.OriginalValues.SetValues(databaseValues);
        throw new ConcurrencyException("Order was modified. Please retry.", ex);
    }
}
```

---

## Step 2020: Complete DbContext with All Interceptors

```csharp
// Infrastructure/Data/AppDbContext.cs
public class AppDbContext : DbContext, IUnitOfWork
{
    private readonly ICurrentUser _currentUser;
    private readonly IDateTime _dateTime;

    public AppDbContext(
        DbContextOptions<AppDbContext> options,
        ICurrentUser currentUser,
        IDateTime dateTime)
        : base(options)
    {
        _currentUser = currentUser;
        _dateTime = dateTime;
    }

    public DbSet<Order> Orders => Set<Order>();
    public DbSet<Product> Products => Set<Product>();
    public DbSet<Customer> Customers => Set<Customer>();

    protected override void OnModelCreating(ModelBuilder builder)
    {
        builder.ApplyConfigurationsFromAssembly(GetType().Assembly);

        // Global soft-delete filter
        builder.Model.GetEntityTypes()
            .Where(t => typeof(ISoftDeletable).IsAssignableFrom(t.ClrType))
            .ToList()
            .ForEach(t => builder
                .Entity(t.ClrType)
                .HasQueryFilter(BuildSoftDeleteFilter(t.ClrType)));

        base.OnModelCreating(builder);
    }

    async Task IUnitOfWork.CommitAsync(CancellationToken ct)
    {
        await SaveChangesAsync(ct);
    }

    private static LambdaExpression BuildSoftDeleteFilter(Type type)
    {
        var param = Expression.Parameter(type, "x");
        var prop = Expression.Property(param, nameof(ISoftDeletable.IsDeleted));
        var notDeleted = Expression.Not(prop);
        return Expression.Lambda(notDeleted, param);
    }
}
```

```csharp
// DependencyInjection.cs — register everything
services.AddDbContext<AppDbContext>((sp, opts) =>
{
    opts.UseNpgsql(config.GetConnectionString("Postgres"),
        npgsql =>
        {
            npgsql.CommandTimeout(30);
            npgsql.EnableRetryOnFailure(3, TimeSpan.FromSeconds(5), null);
        })
        .AddInterceptors(
            sp.GetRequiredService<SlowQueryInterceptor>(),
            sp.GetRequiredService<AuditingInterceptor>(),
            sp.GetRequiredService<DomainEventPublishingInterceptor>(),
            sp.GetRequiredService<TenantSchemaInterceptor>())
        .EnableSensitiveDataLogging(isDevelopment)
        .EnableDetailedErrors(isDevelopment);
});

// Register interceptors (scoped to match DbContext lifecycle)
services.AddScoped<SlowQueryInterceptor>();
services.AddScoped<AuditingInterceptor>();
services.AddScoped<DomainEventPublishingInterceptor>();
services.AddScoped<TenantSchemaInterceptor>();
```

---

## Summary

| Feature | Implementation | Use Case |
|---------|----------------|---------|
| Slow query logging | `DbCommandInterceptor` | Performance investigation |
| Audit fields | `SaveChangesInterceptor` | CreatedAt/By, UpdatedAt/By |
| Soft delete | `SaveChangesInterceptor` + `HasQueryFilter` | Logical deletion |
| Domain events | `SavedChangesInterceptor` | Post-save event dispatch |
| Multi-tenancy | `DbConnectionInterceptor` | RLS via session variable |
| JSON columns | `OwnsOne.ToJson()` | Flexible schema |
| Concurrency | `[Timestamp]` + `DbUpdateConcurrencyException` | Optimistic locking |
| Compiled models | `dotnet ef dbcontext optimize` | Startup performance |
| Bulk ops | `EFCore.BulkExtensions` | Import/export large data |
| Complex queries | Dapper | Reporting, aggregates |

**Next**: Part 87 — Advanced Authentication: OAuth 2.0, PKCE & Identity Server
