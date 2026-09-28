# Part 68: Database Performance — EF Core, Dapper & Query Optimization (Steps 1717-1732)

## Steps 1717-1732: High-Performance Data Access in .NET

Slow queries kill applications. This part covers every layer of data access performance: EF Core compiled queries, query splitting, projection-only reads, raw SQL when needed, Dapper for micro-ORM speed, index strategy, connection pooling, and diagnosing N+1 problems before they reach production.

---

## Step 1717: EF Core — Diagnosing Slow Queries

```csharp
// Enable query logging in Development
builder.Services.AddDbContext<AppDbContext>(options =>
{
    options.UseNpgsql(connectionString)
           .EnableSensitiveDataLogging(builder.Environment.IsDevelopment())
           .EnableDetailedErrors(builder.Environment.IsDevelopment())
           .LogTo(Console.WriteLine, LogLevel.Information);
});

// Better: use ILogger with query timing
options.LogTo(
    action: (eventId, level) => true,
    logger: (eventData) =>
    {
        if (eventData is CommandExecutedEventData cmd)
            logger.LogInformation("EF Query {Duration}ms: {Sql}",
                cmd.Duration.TotalMilliseconds, cmd.Command.CommandText);
    });
```

```csharp
// MiniProfiler for query profiling in dev
builder.Services.AddMiniProfiler(opts =>
{
    opts.RouteBasePath = "/profiler";
    opts.SqlFormatter = new StackExchange.Profiling.SqlFormatters.InlineFormatter();
    opts.TrackConnectionOpenClose = true;
})
.AddEntityFramework();

app.UseMiniProfiler();
// Visit /profiler/results-index in development
```

---

## Step 1718: The N+1 Problem — Detection and Fix

```csharp
// WRONG: N+1 — 1 query for orders, then N queries for customers
var orders = await db.Orders.ToListAsync();  // 1 query
foreach (var order in orders)
{
    var customer = await db.Customers.FindAsync(order.CustomerId);  // N queries!
    Console.WriteLine($"{customer!.Name}: {order.TotalAmount}");
}

// RIGHT: eager loading with Include
var orders = await db.Orders
    .Include(o => o.Customer)           // JOIN — single query
    .Include(o => o.OrderLines)
        .ThenInclude(ol => ol.Product)
    .ToListAsync();

// RIGHT: explicit join projection (even more efficient — no tracking)
var results = await db.Orders
    .Join(db.Customers, o => o.CustomerId, c => c.Id,
        (order, customer) => new { order, customer })
    .Select(x => new OrderSummaryDto(
        x.order.Id,
        x.customer.Name,
        x.order.TotalAmount))
    .ToListAsync();
```

```csharp
// Detect N+1 at test time with EF Core's query counter
using var db = new AppDbContext(options);
int queryCount = 0;
db.Database.GetDbConnection(); // ensure open

// Intercept commands
var interceptor = new QueryCountInterceptor(count => queryCount = count);
// → Use Respawn + EF Core interceptors or MiniProfiler in tests

// Or use the community package: EFCoreSecondLevelCacheInterceptor
// It will throw if N+1 is detected above a threshold
```

---

## Step 1719: Projection — Only Select What You Need

```csharp
// WRONG: loads entire entity including all columns
var orders = await db.Orders
    .Include(o => o.Customer)
    .ToListAsync();
// Loads: Id, CustomerId, Status, TotalAmount, ShippingAddress, Notes, CreatedAt,
//        LastModified, Version, Customer.* (all 15 columns)

// RIGHT: project to DTO — only fetches needed columns
var orderDtos = await db.Orders
    .Where(o => o.Status == OrderStatus.Pending)
    .Select(o => new OrderListDto(
        o.Id,
        o.Customer.Name,   // EF Core translates to a JOIN automatically
        o.TotalAmount,
        o.CreatedAt))
    .ToListAsync();
// SQL: SELECT o.Id, c.Name, o.TotalAmount, o.CreatedAt
//      FROM Orders o JOIN Customers c ON o.CustomerId = c.Id
//      WHERE o.Status = 0

// Projection with conditional expressions
var results = await db.Orders
    .Select(o => new OrderDto
    {
        Id = o.Id,
        Status = o.Status.ToString(),
        ItemCount = o.OrderLines.Count(),           // correlated subquery → COUNT(*)
        HasDiscount = o.Discount != null,
        LatestNote = o.Notes.OrderByDescending(n => n.CreatedAt)
                            .Select(n => n.Text)
                            .FirstOrDefault()       // subquery → LIMIT 1
    })
    .ToListAsync();
```

---

## Step 1720: Compiled Queries — Pre-Parse EF LINQ

Compiled queries parse and translate the LINQ expression tree once and cache the result.

```csharp
using Microsoft.EntityFrameworkCore;

public class OrderRepository
{
    // Static compiled queries — thread-safe, created once at startup
    private static readonly Func<AppDbContext, Guid, Task<Order?>> _getByIdQuery =
        EF.CompileAsyncQuery((AppDbContext db, Guid id) =>
            db.Orders
              .AsNoTracking()
              .Include(o => o.OrderLines)
              .FirstOrDefault(o => o.Id == id));

    private static readonly Func<AppDbContext, string, IAsyncEnumerable<OrderSummaryDto>> _getByCustomerQuery =
        EF.CompileAsyncQuery((AppDbContext db, string customerId) =>
            db.Orders
              .Where(o => o.CustomerId == customerId && o.Status != OrderStatus.Cancelled)
              .OrderByDescending(o => o.CreatedAt)
              .Select(o => new OrderSummaryDto(o.Id, o.TotalAmount, o.Status, o.CreatedAt)));

    private readonly AppDbContext _db;
    public OrderRepository(AppDbContext db) => _db = db;

    public Task<Order?> GetByIdAsync(Guid id) => _getByIdQuery(_db, id);

    public async Task<List<OrderSummaryDto>> GetByCustomerAsync(string customerId)
    {
        var results = new List<OrderSummaryDto>();
        await foreach (var item in _getByCustomerQuery(_db, customerId))
            results.Add(item);
        return results;
    }
}
```

**Performance impact of compiled queries:**

```
Without compiled query: ~2.5ms (parse + translate + execute)
With compiled query:    ~0.3ms (cached plan + execute)
Improvement:            8× faster on hot path
```

---

## Step 1721: Query Splitting — Fix Cartesian Explosion

When you Include multiple collections, EF Core generates a JOIN that creates a Cartesian product (rows × rows). Query splitting issues separate SQL statements instead.

```csharp
// PROBLEM: Cartesian explosion
// Order with 10 lines × 5 tags = 50 rows returned instead of 15
var orders = await db.Orders
    .Include(o => o.OrderLines)   // 10 rows
    .Include(o => o.Tags)         // 5 rows
    .ToListAsync();               // returns 10 × 5 = 50 rows!

// RIGHT: split into separate queries
var orders = await db.Orders
    .Include(o => o.OrderLines)
    .Include(o => o.Tags)
    .AsSplitQuery()               // 3 queries, no Cartesian product
    .ToListAsync();

// Or set globally:
options.UseNpgsql(connectionString, npg =>
    npg.UseQuerySplittingBehavior(QuerySplittingBehavior.SplitQuery));
```

---

## Step 1722: AsNoTracking — Read-Only Queries

Change tracking adds ~30% overhead for every entity loaded. Disable it for read-only queries.

```csharp
// For read-only, non-updated data
var orders = await db.Orders
    .AsNoTracking()                 // no change tracker overhead
    .Where(o => o.Status == OrderStatus.Shipped)
    .Select(o => new OrderDto(o.Id, o.TotalAmount))
    .ToListAsync();

// Faster: AsNoTrackingWithIdentityResolution
// — no tracking, but still resolves duplicate entities (avoids duplicate objects in memory)
var orders = await db.Orders
    .AsNoTrackingWithIdentityResolution()
    .Include(o => o.Customer)       // same customer won't be instantiated twice
    .ToListAsync();

// Global: configure DbContext for read replicas
builder.Services.AddDbContext<ReadOnlyDbContext>(options =>
    options.UseNpgsql(readReplicaConnectionString)
           .UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking));
```

---

## Step 1723: Pagination — Skip/Take vs Keyset

```csharp
// WRONG: offset pagination — gets slower as page number increases
// Page 1000 with 20 items must skip 20,000 rows first
var page = await db.Orders
    .OrderByDescending(o => o.CreatedAt)
    .Skip((pageNumber - 1) * pageSize)  // OFFSET — kills performance at high page numbers
    .Take(pageSize)
    .ToListAsync();

// RIGHT: keyset (cursor) pagination — O(log N) regardless of page
// Client sends the last seen values as the cursor
var page = await db.Orders
    .Where(o => o.CreatedAt < cursor.CreatedAt ||
               (o.CreatedAt == cursor.CreatedAt && o.Id < cursor.Id))
    .OrderByDescending(o => o.CreatedAt)
    .ThenByDescending(o => o.Id)
    .Take(pageSize)
    .ToListAsync();

// Generate the next cursor from the last item
var nextCursor = page.LastOrDefault() is { } last
    ? new Cursor(last.CreatedAt, last.Id)
    : null;
```

---

## Step 1724: Raw SQL with EF Core

```csharp
// FromSql — returns tracked entities
var orders = await db.Orders
    .FromSql($"SELECT * FROM Orders WHERE Status = {status} AND TotalAmount > {minAmount}")
    .Include(o => o.Customer)   // can chain LINQ after FromSql
    .OrderByDescending(o => o.CreatedAt)
    .Take(50)
    .ToListAsync();

// SqlQuery — for non-entity types (DTO projections)
var summaries = await db.Database
    .SqlQuery<OrderSummaryDto>($"""
        SELECT o.Id, c.Name AS CustomerName, o.TotalAmount, o.CreatedAt
        FROM Orders o
        JOIN Customers c ON o.CustomerId = c.Id
        WHERE o.Status = {status}
        ORDER BY o.CreatedAt DESC
        LIMIT {pageSize}
        """)
    .ToListAsync();

// ExecuteSql — for UPDATE/DELETE/INSERT (no result set)
int affected = await db.Database
    .ExecuteSqlAsync($"""
        UPDATE Orders
        SET Status = {OrderStatus.Expired}
        WHERE Status = {OrderStatus.Pending}
          AND CreatedAt < {DateTimeOffset.UtcNow.AddDays(-7)}
        """);

// Bulk update with EF Core 7+ ExecuteUpdateAsync
int updated = await db.Orders
    .Where(o => o.Status == OrderStatus.Pending &&
                o.CreatedAt < DateTimeOffset.UtcNow.AddDays(-7))
    .ExecuteUpdateAsync(s => s
        .SetProperty(o => o.Status, OrderStatus.Expired)
        .SetProperty(o => o.LastModified, DateTimeOffset.UtcNow));

// Bulk delete with EF Core 7+
int deleted = await db.OrderDrafts
    .Where(d => d.CreatedAt < DateTimeOffset.UtcNow.AddDays(-30))
    .ExecuteDeleteAsync();
```

---

## Step 1725: Dapper — When You Need Maximum SQL Control

Dapper maps raw SQL results to C# objects with minimal overhead (about 2× faster than EF Core for read operations).

```xml
<PackageReference Include="Dapper" Version="2.1.35" />
```

```csharp
using Dapper;
using Npgsql;
using System.Data;

public class OrderReadRepository(NpgsqlDataSource dataSource)
{
    // Simple query
    public async Task<OrderDto?> GetByIdAsync(Guid id, CancellationToken ct)
    {
        await using var conn = await dataSource.OpenConnectionAsync(ct);
        return await conn.QueryFirstOrDefaultAsync<OrderDto>(
            "SELECT Id, CustomerId, TotalAmount, Status, CreatedAt FROM Orders WHERE Id = @Id",
            new { Id = id });
    }

    // Multi-mapping: map JOIN result to nested objects
    public async Task<OrderWithCustomer?> GetWithCustomerAsync(Guid id, CancellationToken ct)
    {
        await using var conn = await dataSource.OpenConnectionAsync(ct);
        const string sql = """
            SELECT o.Id, o.TotalAmount, o.Status, o.CreatedAt,
                   c.Id, c.Name, c.Email, c.Tier
            FROM Orders o
            JOIN Customers c ON o.CustomerId = c.Id
            WHERE o.Id = @Id
            """;

        var result = await conn.QueryAsync<Order, Customer, OrderWithCustomer>(
            sql,
            (order, customer) => new OrderWithCustomer(order, customer),
            new { Id = id },
            splitOn: "Id");  // second 'Id' column starts the Customer mapping

        return result.FirstOrDefault();
    }

    // Multiple result sets in one round-trip
    public async Task<(List<Order> Orders, int TotalCount)> GetPagedAsync(
        int page, int size, CancellationToken ct)
    {
        await using var conn = await dataSource.OpenConnectionAsync(ct);
        const string sql = """
            SELECT * FROM Orders ORDER BY CreatedAt DESC LIMIT @Size OFFSET @Offset;
            SELECT COUNT(*) FROM Orders;
            """;

        using var multi = await conn.QueryMultipleAsync(sql,
            new { Size = size, Offset = (page - 1) * size });

        var orders = (await multi.ReadAsync<Order>()).ToList();
        var count = await multi.ReadSingleAsync<int>();
        return (orders, count);
    }

    // Bulk insert with Dapper
    public async Task BulkInsertAsync(IEnumerable<Order> orders, CancellationToken ct)
    {
        await using var conn = await dataSource.OpenConnectionAsync(ct);
        await conn.ExecuteAsync(
            "INSERT INTO Orders (Id, CustomerId, TotalAmount, Status, CreatedAt) VALUES (@Id, @CustomerId, @TotalAmount, @Status, @CreatedAt)",
            orders);  // Dapper batches this automatically
    }
}
```

---

## Step 1726: Dapper Type Handlers

```csharp
// Custom type handler for complex types
public class DateTimeOffsetTypeHandler : SqlMapper.TypeHandler<DateTimeOffset>
{
    public override DateTimeOffset Parse(object value)
        => DateTime.SpecifyKind((DateTime)value, DateTimeKind.Utc);

    public override void SetValue(IDbDataParameter parameter, DateTimeOffset value)
    {
        parameter.Value = value.UtcDateTime;
        parameter.DbType = DbType.DateTime2;
    }
}

// Register at startup
SqlMapper.AddTypeHandler(new DateTimeOffsetTypeHandler());

// Custom handler for Value Objects
public class MoneyTypeHandler : SqlMapper.TypeHandler<Money>
{
    public override Money Parse(object value)
        => new Money((decimal)value, "USD");

    public override void SetValue(IDbDataParameter parameter, Money value)
    {
        parameter.Value = value.Amount;
        parameter.DbType = DbType.Decimal;
    }
}
```

---

## Step 1727: PostgreSQL-Specific Optimizations with Npgsql

```csharp
// Batch multiple commands in one round-trip
await using var batch = dataSource.CreateBatch();
batch.BatchCommands.Add(new NpgsqlBatchCommand(
    "UPDATE Orders SET Status = $1 WHERE Id = $2") { Parameters = { status, orderId } });
batch.BatchCommands.Add(new NpgsqlBatchCommand(
    "INSERT INTO AuditLog (OrderId, Action, At) VALUES ($1, $2, $3)")
    { Parameters = { orderId, "StatusChanged", DateTimeOffset.UtcNow } });
await batch.ExecuteNonQueryAsync();
// Both commands in ONE network round-trip

// COPY for ultra-fast bulk insert (faster than INSERT ... VALUES)
await using var writer = await conn.BeginBinaryImportAsync(
    "COPY Orders (Id, CustomerId, TotalAmount, Status, CreatedAt) FROM STDIN (FORMAT BINARY)");

foreach (var order in orders)
{
    await writer.StartRowAsync();
    await writer.WriteAsync(order.Id, NpgsqlDbType.Uuid);
    await writer.WriteAsync(order.CustomerId, NpgsqlDbType.Varchar);
    await writer.WriteAsync(order.TotalAmount, NpgsqlDbType.Numeric);
    await writer.WriteAsync((int)order.Status, NpgsqlDbType.Integer);
    await writer.WriteAsync(order.CreatedAt, NpgsqlDbType.TimestampTz);
}
await writer.CompleteAsync();
// 10-50× faster than individual INSERTs for large datasets
```

---

## Step 1728: Index Strategy

```csharp
// EF Core index configuration
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Order>(entity =>
    {
        // B-tree index for equality + range queries
        entity.HasIndex(o => o.CustomerId)
              .HasDatabaseName("IX_Orders_CustomerId");

        // Composite index — order matters: most selective first
        entity.HasIndex(o => new { o.Status, o.CreatedAt })
              .HasDatabaseName("IX_Orders_Status_CreatedAt");

        // Partial index — only index active orders (smaller, faster)
        entity.HasIndex(o => o.CreatedAt)
              .HasFilter("\"Status\" NOT IN (3, 4)")  // NOT Shipped, NOT Cancelled
              .HasDatabaseName("IX_Orders_Active_CreatedAt");

        // Unique index
        entity.HasIndex(o => o.ReferenceNumber)
              .IsUnique()
              .HasDatabaseName("UX_Orders_ReferenceNumber");

        // Index for text search with pg_trgm
        entity.HasIndex(o => o.CustomerName)
              .HasMethod("gin")
              .HasOperators("gin_trgm_ops")
              .HasDatabaseName("IX_Orders_CustomerName_Trgm");
    });
}
```

```sql
-- Check index usage in PostgreSQL
SELECT schemaname, tablename, indexname, idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
WHERE tablename = 'Orders'
ORDER BY idx_scan DESC;

-- Find missing indexes (sequential scans on large tables)
SELECT schemaname, tablename, seq_scan, seq_tup_read, idx_scan,
       round(seq_scan::numeric / (seq_scan + idx_scan + 1) * 100, 1) AS seq_scan_pct
FROM pg_stat_user_tables
WHERE seq_scan > 100
ORDER BY seq_tup_read DESC;

-- Find slow queries
SELECT query, calls, total_exec_time, mean_exec_time, rows
FROM pg_stat_statements
WHERE mean_exec_time > 100
ORDER BY mean_exec_time DESC
LIMIT 20;
```

---

## Step 1729: Connection Pooling

```csharp
// Npgsql connection pool configuration
var dataSourceBuilder = new NpgsqlDataSourceBuilder(connectionString);
dataSourceBuilder.ConnectionStringBuilder.MaxPoolSize = 100;     // max connections
dataSourceBuilder.ConnectionStringBuilder.MinPoolSize = 5;       // keep alive
dataSourceBuilder.ConnectionStringBuilder.ConnectionIdleLifetime = 300; // seconds
dataSourceBuilder.ConnectionStringBuilder.ConnectionPruningInterval = 60;
dataSourceBuilder.ConnectionStringBuilder.CommandTimeout = 30;

var dataSource = dataSourceBuilder.Build();
builder.Services.AddSingleton(dataSource);

// Health check for connection pool
builder.Services.AddHealthChecks()
    .AddNpgsql(connectionString, name: "postgres", timeout: TimeSpan.FromSeconds(5));
```

```csharp
// PgBouncer pooling configuration (external connection pooler)
// Transaction mode: best for microservices
// Connection string points to PgBouncer, not Postgres directly
var pgBouncerConnectionString =
    "Host=pgbouncer;Port=5432;Database=orders;Username=api;Password=...;" +
    "MaxPoolSize=1;MinPoolSize=0;"; // PgBouncer handles the pooling, not Npgsql

// Important: in transaction mode, avoid session-level features:
// - LISTEN/NOTIFY
// - SET session variables
// - Prepared statements (disable with No Reset On Close=true)
```

---

## Step 1730: Query Caching with Redis

```csharp
using Microsoft.Extensions.Caching.Distributed;
using System.Text.Json;

public class CachedOrderRepository(
    IOrderRepository inner,
    IDistributedCache cache,
    ILogger<CachedOrderRepository> logger) : IOrderRepository
{
    private static string CacheKey(Guid id) => $"order:{id}";
    private static readonly TimeSpan Ttl = TimeSpan.FromMinutes(5);

    public async Task<Order?> GetByIdAsync(Guid id, CancellationToken ct)
    {
        var key = CacheKey(id);

        // Try cache first
        var cached = await cache.GetStringAsync(key, ct);
        if (cached is not null)
        {
            logger.LogDebug("Cache HIT for order {OrderId}", id);
            return JsonSerializer.Deserialize<Order>(cached);
        }

        // Cache miss — fetch from DB
        logger.LogDebug("Cache MISS for order {OrderId}", id);
        var order = await inner.GetByIdAsync(id, ct);

        if (order is not null)
        {
            await cache.SetStringAsync(key,
                JsonSerializer.Serialize(order),
                new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = Ttl },
                ct);
        }

        return order;
    }

    public async Task SaveAsync(Order order, CancellationToken ct)
    {
        await inner.SaveAsync(order, ct);
        // Invalidate cache on write
        await cache.RemoveAsync(CacheKey(order.Id), ct);
    }
}
```

---

## Step 1731: EF Core Second Level Cache

```csharp
// Install: EFCoreSecondLevelCacheInterceptor
builder.Services.AddEFSecondLevelCache(options =>
    options
        .UseRedis(redisConnectionString, "orders-cache:")
        .CacheAllQueries(CacheExpirationMode.Absolute, TimeSpan.FromMinutes(5))
        .SkipCachingCommands(cmd =>
            cmd.StartsWith("INSERT") ||
            cmd.StartsWith("UPDATE") ||
            cmd.StartsWith("DELETE"))
        .UseDbCallsIfCachingProviderIsDown(TimeSpan.FromMinutes(1)));

// Add interceptor to DbContext
options.UseNpgsql(connectionString)
       .AddInterceptors(
           serviceProvider.GetRequiredService<SecondLevelCacheInterceptor>());

// Per-query cache control
var order = await db.Orders
    .Where(o => o.Id == id)
    .Cacheable(CacheExpirationMode.Absolute, TimeSpan.FromMinutes(10))
    .FirstOrDefaultAsync();

// Skip cache for this query
var freshOrder = await db.Orders
    .Where(o => o.Id == id)
    .NotCacheable()
    .FirstOrDefaultAsync();
```

---

## Step 1732: Complete Performance Benchmark — EF Core vs Dapper vs Raw ADO.NET

```csharp
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Running;

BenchmarkRunner.Run<DataAccessBenchmarks>();

[MemoryDiagnoser]
[SimpleJob(iterationCount: 50, warmupCount: 5)]
public class DataAccessBenchmarks
{
    private AppDbContext _db = null!;
    private NpgsqlDataSource _dataSource = null!;
    private Guid _orderId;

    [GlobalSetup]
    public async Task Setup()
    {
        _db = new AppDbContext(/* options */);
        _dataSource = NpgsqlDataSource.Create(connectionString);
        _orderId = (await _db.Orders.Select(o => o.Id).FirstAsync());
    }

    [Benchmark(Baseline = true)]
    public async Task<Order?> EfCore_Tracked()
        => await _db.Orders.FindAsync(_orderId);

    [Benchmark]
    public async Task<Order?> EfCore_AsNoTracking()
        => await _db.Orders.AsNoTracking().FirstOrDefaultAsync(o => o.Id == _orderId);

    [Benchmark]
    public async Task<Order?> EfCore_Compiled()
        => await _getByIdQuery(_db, _orderId);

    [Benchmark]
    public async Task<OrderDto?> EfCore_Projection()
        => await _db.Orders
            .Where(o => o.Id == _orderId)
            .Select(o => new OrderDto(o.Id, o.TotalAmount, o.Status))
            .FirstOrDefaultAsync();

    [Benchmark]
    public async Task<OrderDto?> Dapper()
    {
        await using var conn = await _dataSource.OpenConnectionAsync();
        return await conn.QueryFirstOrDefaultAsync<OrderDto>(
            "SELECT Id, TotalAmount, Status FROM Orders WHERE Id = @Id",
            new { Id = _orderId });
    }

    [Benchmark]
    public async Task<OrderDto?> RawAdoNet()
    {
        await using var conn = await _dataSource.OpenConnectionAsync();
        await using var cmd = conn.CreateCommand();
        cmd.CommandText = "SELECT Id, TotalAmount, Status FROM Orders WHERE Id = $1";
        cmd.Parameters.AddWithValue(_orderId);
        await using var reader = await cmd.ExecuteReaderAsync();
        if (!await reader.ReadAsync()) return null;
        return new OrderDto(reader.GetGuid(0), reader.GetDecimal(1), (OrderStatus)reader.GetInt32(2));
    }

    private static readonly Func<AppDbContext, Guid, Task<Order?>> _getByIdQuery =
        EF.CompileAsyncQuery((AppDbContext db, Guid id) =>
            db.Orders.AsNoTracking().FirstOrDefault(o => o.Id == id));

    [GlobalCleanup]
    public async Task Cleanup()
    {
        await _db.DisposeAsync();
        await _dataSource.DisposeAsync();
    }
}

// Typical results (PostgreSQL, local):
// EfCore_Tracked:     0.92ms  72KB alloc
// EfCore_AsNoTracking: 0.61ms  48KB alloc
// EfCore_Compiled:    0.38ms  31KB alloc  ← compiled = big win
// EfCore_Projection:  0.29ms  18KB alloc  ← projection = best EF approach
// Dapper:             0.21ms  12KB alloc
// RawAdoNet:          0.18ms   8KB alloc
```

---

## Data Access Decision Matrix

| Scenario | Recommended Approach |
|---|---|
| Domain operations (write) | EF Core with change tracking |
| Read-only queries (simple) | EF Core `.AsNoTracking()` + projection |
| Read-only queries (hot path) | EF Core compiled query + projection |
| Complex reporting / analytics | Raw SQL via `SqlQuery<T>` or Dapper |
| Bulk insert (>1000 rows) | `ExecuteUpdateAsync` / Npgsql COPY |
| Bulk update/delete | `ExecuteUpdateAsync` / `ExecuteDeleteAsync` |
| Stored procedures | `FromSqlRaw` or Dapper |
| Read replicas | Separate `ReadOnlyDbContext` with `AsNoTracking` |
| High read throughput | Dapper with compiled SQL |
| Time-series / analytics | Raw Npgsql with binary protocol |

### What Next?

- **Part 69**: Observability Deep Dive — OpenTelemetry traces, Grafana Tempo, Jaeger, structured logging
- **Part 70**: Resilience Patterns — Polly v8, Circuit Breaker, Bulkhead, Hedging, Fallback
- **Part 71**: Multi-Tenancy Architecture — Row-level security, schema-per-tenant, tenant isolation
