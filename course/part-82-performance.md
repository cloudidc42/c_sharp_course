# Part 82: Performance Optimization & Memory Management

## Steps 1941–1956

---

## Step 1941: Memory Fundamentals in .NET

Understanding how memory works is the foundation of performance optimization.

```
Stack:                          Heap:
├── Value types (int, bool)     ├── Reference types (class, string[])
├── Struct fields               ├── Large Object Heap (>85KB)
├── Method parameters           ├── Generation 0 (short-lived)
├── Return addresses            ├── Generation 1 (promoted from Gen0)
└── Local variables (struct)    └── Generation 2 (long-lived, e.g. static)
```

**GC pressure** — allocating frequently forces more GC collections:
- Gen0 collection: ~1ms — most objects die here
- Gen1 collection: ~5ms — promoted survivors
- Gen2 collection: ~100ms+ — full collection, stop-the-world

**Golden rules**:
1. Allocate less (object pooling, stack allocation, `Span<T>`)
2. Allocate short-lived objects (die in Gen0)
3. Avoid large object heap fragmentation (arrays > 85KB)
4. Avoid finalizers (delay collection)

---

## Step 1942: Span<T> and Memory<T>

`Span<T>` — stack-allocated slice of contiguous memory. Zero allocation.

```csharp
// Traditional — allocates substrings
public static int CountWords(string text)
{
    var words = text.Split(' ');  // allocates array + strings
    return words.Length;
}

// Span-based — zero allocation
public static int CountWordsSpan(ReadOnlySpan<char> text)
{
    if (text.IsEmpty) return 0;

    var count = 0;
    var inWord = false;

    foreach (var ch in text)
    {
        if (ch == ' ')
        {
            if (inWord)
            {
                count++;
                inWord = false;
            }
        }
        else
        {
            inWord = true;
        }
    }

    return inWord ? count + 1 : count;
}

// Usage
var text = "Hello world foo bar";
var wordCount = CountWordsSpan(text.AsSpan()); // no allocation
```

```csharp
// Parsing without allocation
public static bool TryParseOrderId(ReadOnlySpan<char> input, out Guid orderId)
{
    // "ORDER-{guid}" — extract guid part
    const string prefix = "ORDER-";
    if (input.Length < prefix.Length + 32)
    {
        orderId = Guid.Empty;
        return false;
    }

    var guidPart = input[prefix.Length..];
    return Guid.TryParse(guidPart, out orderId);
}
```

```csharp
// Memory<T> — heap-allocated, can be passed across async boundaries
public static async Task ProcessChunksAsync(Memory<byte> buffer)
{
    // Span<T> can't cross await boundaries — Memory<T> can
    var firstHalf = buffer[..(buffer.Length / 2)];
    await ProcessAsync(firstHalf); // OK — Memory<T> is heap-safe
}

// ArraySegment<T> — older, but also zero-copy
byte[] arr = new byte[1024];
var segment = new ArraySegment<byte>(arr, 100, 200); // offset=100, count=200
```

---

## Step 1943: ArrayPool and ObjectPool

Reuse buffers and objects instead of allocating new ones.

```csharp
using System.Buffers;

// ArrayPool — rent and return
public static async Task<string> ReadAndTransformAsync(Stream stream)
{
    var pool = ArrayPool<byte>.Shared;
    var buffer = pool.Rent(4096); // may get larger array — that's fine

    try
    {
        var bytesRead = await stream.ReadAsync(buffer.AsMemory(0, 4096));
        // Process buffer[0..bytesRead]
        return Encoding.UTF8.GetString(buffer, 0, bytesRead);
    }
    finally
    {
        pool.Return(buffer, clearArray: false); // return to pool
    }
}
```

```csharp
// Custom object pool
using Microsoft.Extensions.ObjectPool;

public class ExpensiveObjectPolicy : PooledObjectPolicy<ExpensiveObject>
{
    public override ExpensiveObject Create() => new ExpensiveObject();

    public override bool Return(ExpensiveObject obj)
    {
        obj.Reset(); // clear state before returning to pool
        return true;
    }
}

// Registration
builder.Services.AddSingleton<ObjectPoolProvider, DefaultObjectPoolProvider>();
builder.Services.AddSingleton<ObjectPool<ExpensiveObject>>(sp =>
{
    var provider = sp.GetRequiredService<ObjectPoolProvider>();
    return provider.Create(new ExpensiveObjectPolicy());
});

// Usage
public class ReportService
{
    private readonly ObjectPool<ExpensiveObject> _pool;

    public async Task<byte[]> GenerateAsync()
    {
        var obj = _pool.Get();
        try
        {
            return await obj.GenerateReportAsync();
        }
        finally
        {
            _pool.Return(obj);
        }
    }
}
```

---

## Step 1944: RecyclableMemoryStream

Eliminates LOH allocations from `MemoryStream`.

```bash
dotnet add package Microsoft.IO.RecyclableMemoryStream
```

```csharp
using Microsoft.IO;

// Register as singleton
builder.Services.AddSingleton<RecyclableMemoryStreamManager>();

// Usage
public class JsonSerializer
{
    private readonly RecyclableMemoryStreamManager _streamManager;

    public JsonSerializer(RecyclableMemoryStreamManager streamManager)
    {
        _streamManager = streamManager;
    }

    public async Task<byte[]> SerializeAsync<T>(T value)
    {
        using var stream = _streamManager.GetStream("json-serialize");
        await System.Text.Json.JsonSerializer.SerializeAsync(stream, value);
        return stream.ToArray();
    }

    public async Task SerializeToStreamAsync<T>(T value, Stream target)
    {
        // Zero-copy: write directly to target
        await System.Text.Json.JsonSerializer.SerializeAsync(target, value);
    }
}
```

---

## Step 1945: ValueTask and IAsyncEnumerable Performance

```csharp
// Task allocates on heap. ValueTask is struct — often zero-allocation for sync path.

public interface ICachedLookup<TKey, TValue>
{
    ValueTask<TValue?> GetAsync(TKey key, CancellationToken ct = default);
}

public class CachedProductLookup : ICachedLookup<Guid, Product>
{
    private readonly Dictionary<Guid, Product> _cache = new();
    private readonly IProductRepository _repo;

    public ValueTask<Product?> GetAsync(Guid id, CancellationToken ct = default)
    {
        // Cache hit — synchronous, no heap allocation for Task
        if (_cache.TryGetValue(id, out var product))
            return ValueTask.FromResult<Product?>(product);

        // Cache miss — async path
        return FetchFromDbAsync(id, ct);
    }

    private async ValueTask<Product?> FetchFromDbAsync(Guid id, CancellationToken ct)
    {
        var product = await _repo.GetByIdAsync(id, ct);
        if (product is not null)
            _cache[id] = product;
        return product;
    }
}
```

```csharp
// IAsyncEnumerable — stream results without loading all into memory
public async IAsyncEnumerable<OrderRow> ExportOrdersAsync(
    DateRange range,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    var offset = 0;
    const int batchSize = 1000;

    while (true)
    {
        var batch = await _db.Orders
            .Where(o => o.CreatedAt >= range.Start && o.CreatedAt < range.End)
            .OrderBy(o => o.CreatedAt)
            .Skip(offset)
            .Take(batchSize)
            .Select(o => new OrderRow(o.Id, o.TotalAmount, o.CreatedAt))
            .ToListAsync(ct);

        foreach (var row in batch)
            yield return row;

        if (batch.Count < batchSize)
            break;

        offset += batchSize;
    }
}

// API endpoint streams JSON array
app.MapGet("/api/orders/export", async (
    IOrderExporter exporter,
    [FromQuery] DateTimeOffset from,
    [FromQuery] DateTimeOffset to,
    CancellationToken ct) =>
{
    return Results.Stream(async stream =>
    {
        await JsonSerializer.SerializeAsync(
            stream,
            exporter.ExportOrdersAsync(new DateRange(from, to), ct),
            cancellationToken: ct);
    }, "application/json");
});
```

---

## Step 1946: EF Core Performance

```csharp
// Problem: SELECT N+1
var orders = await _db.Orders.ToListAsync(ct);
foreach (var order in orders)
{
    // Executes a query per order — N+1 queries!
    var items = await _db.OrderItems.Where(i => i.OrderId == order.Id).ToListAsync(ct);
}

// Fix 1: Eager loading
var orders = await _db.Orders
    .Include(o => o.Items)
    .ToListAsync(ct);

// Fix 2: Projection — only load what you need
var dtos = await _db.Orders
    .Select(o => new OrderDto(
        o.Id,
        o.Status.ToString(),
        o.Items.Count,
        o.TotalAmount))
    .ToListAsync(ct);

// Fix 3: Split query — separate SQL for navigation properties
var orders = await _db.Orders
    .Include(o => o.Items)
    .AsSplitQuery()  // Two SELECTs instead of a JOIN
    .ToListAsync(ct);
```

```csharp
// AsNoTracking — read-only queries
var products = await _db.Products
    .AsNoTracking()    // No change tracking overhead
    .Where(p => p.IsActive)
    .ToListAsync(ct);

// AsNoTrackingWithIdentityResolution — when you need deduplication but not tracking
var orders = await _db.Orders
    .AsNoTrackingWithIdentityResolution()
    .Include(o => o.Customer)  // Customer deduped if same customer appears in multiple orders
    .ToListAsync(ct);
```

```csharp
// Compiled queries — amortize LINQ expression compilation cost
private static readonly Func<AppDbContext, Guid, Task<Order?>> GetOrderById =
    EF.CompileAsyncQuery((AppDbContext db, Guid id) =>
        db.Orders.FirstOrDefault(o => o.Id == id));

// Usage
var order = await GetOrderById(_db, orderId);
```

```csharp
// Bulk operations with EF Core 7+ ExecuteUpdate / ExecuteDelete
// Avoids loading entities into memory

await _db.Orders
    .Where(o => o.Status == OrderStatus.Draft && o.CreatedAt < cutoff)
    .ExecuteDeleteAsync(ct); // Direct DELETE SQL — no entity tracking

await _db.Products
    .Where(p => p.CategoryId == categoryId)
    .ExecuteUpdateAsync(p => p
        .SetProperty(x => x.IsActive, false)
        .SetProperty(x => x.UpdatedAt, DateTimeOffset.UtcNow),
        ct); // Direct UPDATE SQL
```

---

## Step 1947: Caching Strategies

```csharp
// In-memory cache — single server
builder.Services.AddMemoryCache(opts =>
{
    opts.SizeLimit = 10_000; // limit item count
});

public class ProductCache
{
    private readonly IMemoryCache _cache;
    private readonly IProductRepository _repo;

    public async Task<Product?> GetAsync(Guid id, CancellationToken ct = default)
    {
        return await _cache.GetOrCreateAsync($"product:{id}", async entry =>
        {
            entry.SetAbsoluteExpiration(TimeSpan.FromMinutes(10));
            entry.SetSlidingExpiration(TimeSpan.FromMinutes(2));
            entry.SetSize(1);
            entry.RegisterPostEvictionCallback((key, value, reason, state) =>
            {
                // Log or react to eviction
            });

            return await _repo.GetByIdAsync(id, ct);
        });
    }

    public void Invalidate(Guid id) => _cache.Remove($"product:{id}");
}
```

```csharp
// Distributed cache — multi-server
builder.Services.AddStackExchangeRedisCache(opts =>
{
    opts.Configuration = builder.Configuration.GetConnectionString("Redis");
    opts.InstanceName = "myapp:";
});

public class DistributedProductCache
{
    private readonly IDistributedCache _cache;
    private readonly IProductRepository _repo;
    private static readonly JsonSerializerOptions JsonOpts = new()
    {
        PropertyNamingPolicy = JsonNamingPolicy.CamelCase
    };

    public async Task<ProductDto?> GetAsync(Guid id, CancellationToken ct = default)
    {
        var key = $"product:{id}";
        var cached = await _cache.GetAsync(key, ct);

        if (cached is not null)
            return JsonSerializer.Deserialize<ProductDto>(cached, JsonOpts);

        var product = await _repo.GetByIdAsync(id, ct);
        if (product is null) return null;

        var dto = ProductDto.From(product);
        var serialized = JsonSerializer.SerializeToUtf8Bytes(dto, JsonOpts);

        await _cache.SetAsync(key, serialized, new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10)
        }, ct);

        return dto;
    }
}
```

```csharp
// Hybrid cache (.NET 9) — L1 in-process + L2 distributed
builder.Services.AddHybridCache(opts =>
{
    opts.DefaultEntryOptions = new HybridCacheEntryOptions
    {
        Expiration = TimeSpan.FromMinutes(10),
        LocalCacheExpiration = TimeSpan.FromMinutes(1)
    };
});

public class HybridProductCache
{
    private readonly HybridCache _cache;

    public async Task<ProductDto?> GetAsync(Guid id, CancellationToken ct = default)
    {
        return await _cache.GetOrCreateAsync(
            $"product:{id}",
            async token => await _repo.GetByIdAsync(id, token),
            cancellationToken: ct);
    }
}
```

---

## Step 1948: Response Caching & Output Caching

```csharp
// Output caching — cache entire HTTP response
builder.Services.AddOutputCache(opts =>
{
    opts.AddBasePolicy(policy => policy.Expire(TimeSpan.FromSeconds(60)));
    opts.AddPolicy("products", policy => policy
        .Expire(TimeSpan.FromMinutes(5))
        .Tag("products")
        .SetVaryByQuery("page", "pageSize", "search"));
});

app.UseOutputCache();

// Apply to endpoints
app.MapGet("/api/products", GetProducts)
    .CacheOutput("products");

app.MapPost("/api/products", CreateProduct)
    .CacheOutput(p => p.NoCache()); // Never cache mutations

// Invalidate cache tag when product changes
public class ProductCreatedHandler : INotificationHandler<ProductCreatedEvent>
{
    private readonly IOutputCacheStore _cacheStore;

    public async Task Handle(ProductCreatedEvent notification, CancellationToken ct)
    {
        await _cacheStore.EvictByTagAsync("products", ct);
    }
}
```

---

## Step 1949: Parallel Processing

```csharp
// Parallel.ForEachAsync — bounded concurrency
public async Task ProcessOrdersAsync(IReadOnlyList<Guid> orderIds, CancellationToken ct)
{
    var options = new ParallelOptions
    {
        MaxDegreeOfParallelism = Environment.ProcessorCount,
        CancellationToken = ct
    };

    await Parallel.ForEachAsync(orderIds, options, async (orderId, innerCt) =>
    {
        await ProcessSingleOrderAsync(orderId, innerCt);
    });
}
```

```csharp
// Channel-based producer/consumer pipeline
public class OrderProcessingPipeline
{
    public async Task RunAsync(
        IAsyncEnumerable<RawOrderMessage> source,
        CancellationToken ct)
    {
        var validateChannel = Channel.CreateBounded<RawOrderMessage>(100);
        var enrichChannel = Channel.CreateBounded<ValidatedOrder>(100);
        var persistChannel = Channel.CreateBounded<EnrichedOrder>(100);

        // Stage 1: Validate (2 consumers)
        var validateTasks = Enumerable.Range(0, 2)
            .Select(_ => ValidateAsync(validateChannel.Reader, enrichChannel.Writer, ct));

        // Stage 2: Enrich (4 consumers — most CPU-intensive)
        var enrichTasks = Enumerable.Range(0, 4)
            .Select(_ => EnrichAsync(enrichChannel.Reader, persistChannel.Writer, ct));

        // Stage 3: Persist (1 consumer — DB batching)
        var persistTask = PersistBatchAsync(persistChannel.Reader, ct);

        // Producer
        var producerTask = Task.Run(async () =>
        {
            await foreach (var msg in source.WithCancellation(ct))
                await validateChannel.Writer.WriteAsync(msg, ct);
            validateChannel.Writer.Complete();
        }, ct);

        await Task.WhenAll(validateTasks
            .Concat(enrichTasks)
            .Append(persistTask)
            .Append(producerTask));
    }

    private async Task ValidateAsync(
        ChannelReader<RawOrderMessage> reader,
        ChannelWriter<ValidatedOrder> writer,
        CancellationToken ct)
    {
        await foreach (var msg in reader.ReadAllAsync(ct))
        {
            var validated = Validate(msg);
            if (validated is not null)
                await writer.WriteAsync(validated, ct);
        }
    }
}
```

---

## Step 1950: SIMD with System.Numerics

```csharp
using System.Numerics;
using System.Runtime.Intrinsics;
using System.Runtime.Intrinsics.X86;

// Vector<T> — uses CPU SIMD registers automatically
public static decimal SumPrices(ReadOnlySpan<float> prices)
{
    var sum = Vector<float>.Zero;
    var i = 0;
    var vectorSize = Vector<float>.Count;

    // Process SIMD-width chunks
    for (; i <= prices.Length - vectorSize; i += vectorSize)
    {
        sum += new Vector<float>(prices[i..]);
    }

    var result = 0f;
    for (var j = 0; j < vectorSize; j++)
        result += sum[j];

    // Handle remainder
    for (; i < prices.Length; i++)
        result += prices[i];

    return (decimal)result;
}

// AVX2 intrinsics — explicit SIMD (x86 only, check Avx2.IsSupported)
public static void MultiplyPrices(Span<float> prices, float multiplier)
{
    if (Avx2.IsSupported)
    {
        var multiplierVec = Vector256.Create(multiplier);
        var i = 0;
        for (; i <= prices.Length - 8; i += 8)
        {
            var vec = Vector256.LoadUnsafe(ref prices[i]);
            var result = Avx.Multiply(vec, multiplierVec);
            result.StoreUnsafe(ref prices[i]);
        }
        // Handle remainder
        for (; i < prices.Length; i++)
            prices[i] *= multiplier;
    }
    else
    {
        for (var i = 0; i < prices.Length; i++)
            prices[i] *= multiplier;
    }
}
```

---

## Step 1951: Unsafe Code for Hot Paths

```csharp
using System.Runtime.CompilerServices;
using System.Runtime.InteropServices;

// MemoryMarshal — zero-copy reinterpretation
public static ReadOnlySpan<byte> GetBytes(ReadOnlySpan<int> ints)
{
    return MemoryMarshal.AsBytes(ints);
}

// Unsafe.Add — pointer arithmetic in managed code
public static int SumArray(int[] arr)
{
    if (arr.Length == 0) return 0;

    ref var current = ref MemoryMarshal.GetArrayDataReference(arr);
    var sum = 0;

    for (var i = 0; i < arr.Length; i++)
    {
        sum += Unsafe.Add(ref current, i);
    }

    return sum;
}

// [MethodImpl] — inline hot methods
[MethodImpl(MethodImplOptions.AggressiveInlining)]
public static bool IsValidStatus(int status) => status is >= 0 and <= 5;

[MethodImpl(MethodImplOptions.AggressiveOptimization)]
public static void HotLoop(Span<int> data)
{
    for (var i = 0; i < data.Length; i++)
        data[i] = data[i] * 2 + 1;
}
```

---

## Step 1952: BenchmarkDotNet

```bash
dotnet add package BenchmarkDotNet
```

```csharp
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Running;

[MemoryDiagnoser]
[HtmlExporter]
[RankColumn]
public class StringParsingBenchmarks
{
    private const string Input = "ORDER-550e8400-e29b-41d4-a716-446655440000";
    private static readonly byte[] InputBytes = Encoding.UTF8.GetBytes(Input);

    [Benchmark(Baseline = true)]
    public Guid ParseWithSubstring()
    {
        var guidStr = Input.Substring(6); // allocates
        return Guid.Parse(guidStr);
    }

    [Benchmark]
    public Guid ParseWithSpan()
    {
        var span = Input.AsSpan(6); // no allocation
        return Guid.Parse(span);
    }

    [Benchmark]
    public bool TryParseWithSpan()
    {
        var span = Input.AsSpan(6);
        return Guid.TryParse(span, out _);
    }
}

// Run in Program.cs / separate console project
BenchmarkRunner.Run<StringParsingBenchmarks>();
```

```
// Typical output:
| Method            | Mean     | Error   | StdDev  | Rank | Allocated |
|------------------ |---------:|--------:|--------:|-----:|----------:|
| ParseWithSubstring| 185.3 ns | 1.23 ns | 1.09 ns |    3 |     120 B |
| ParseWithSpan     |  82.1 ns | 0.41 ns | 0.36 ns |    2 |       - B |
| TryParseWithSpan  |  78.6 ns | 0.28 ns | 0.24 ns |    1 |       - B |
```

---

## Step 1953: dotnet-counters and dotnet-trace

```bash
# Live metrics
dotnet-counters monitor --process-id <pid> --counters \
  System.Runtime,Microsoft.AspNetCore.Hosting

# Key counters to watch:
# gc-heap-size          — total managed heap
# gen-0-gc-count        — Gen0 collections per minute
# gen-1-gc-count        — Gen1 collections per minute
# gen-2-gc-count        — Gen2 (expensive)
# threadpool-queue-length — work items waiting for thread
# alloc-rate            — MB/sec allocation rate

# Capture trace
dotnet-trace collect --process-id <pid> --profile cpu-sampling

# Analyze in PerfView or SpeedScope
```

```csharp
// EventCounters — expose custom application metrics
public class OrderCounters : IDisposable
{
    private readonly EventCounter _ordersThroughput;
    private readonly IncrementingEventCounter _totalOrders;
    private readonly PollingCounter _activeOrders;
    private int _activeOrderCount;

    public OrderCounters()
    {
        var source = new EventSource("MyApp.Orders");
        _ordersThroughput = new EventCounter("orders-per-second", source);
        _totalOrders = new IncrementingEventCounter("total-orders", source);
        _activeOrders = new PollingCounter("active-orders", source,
            () => Volatile.Read(ref _activeOrderCount));
    }

    public void RecordOrderPlaced(double durationMs)
    {
        _ordersThroughput.WriteMetric(durationMs);
        _totalOrders.Increment();
        Interlocked.Increment(ref _activeOrderCount);
    }

    public void RecordOrderCompleted()
    {
        Interlocked.Decrement(ref _activeOrderCount);
    }

    public void Dispose()
    {
        _ordersThroughput.Dispose();
        _totalOrders.Dispose();
        _activeOrders.Dispose();
    }
}
```

---

## Step 1954: Connection Pooling & Async I/O

```csharp
// Npgsql — tune connection pool
var dataSource = new NpgsqlDataSourceBuilder(connectionString)
    .ConfigureConnectionPool(pool =>
    {
        pool.MinPoolSize = 5;
        pool.MaxPoolSize = 100;
        pool.ConnectionIdleLifetime = TimeSpan.FromMinutes(5);
        pool.ConnectionLifetime = TimeSpan.FromMinutes(30);
    })
    .Build();

builder.Services.AddDbContext<AppDbContext>(opts =>
    opts.UseNpgsql(dataSource));
```

```csharp
// Avoid sync over async (deadlocks)
// WRONG
public string GetData()
{
    return GetDataAsync().Result; // deadlock on ASP.NET
}

// RIGHT
public async Task<string> GetDataAsync()
{
    return await _service.GetDataAsync();
}

// ConfigureAwait(false) in library code
public async Task<byte[]> LibraryMethodAsync()
{
    var data = await FetchDataAsync().ConfigureAwait(false);
    return Process(data);
}
```

---

## Step 1955: StringBuilder and StringPool

```csharp
// StringBuilder — avoid string concatenation in loops
public static string BuildCsv(IEnumerable<OrderRow> rows)
{
    var sb = new StringBuilder(capacity: 4096);
    sb.AppendLine("Id,Status,Total,CreatedAt");

    foreach (var row in rows)
    {
        sb.Append(row.Id).Append(',')
          .Append(row.Status).Append(',')
          .Append(row.Total.ToString("F2")).Append(',')
          .AppendLine(row.CreatedAt.ToString("O"));
    }

    return sb.ToString();
}

// String.Create — zero intermediate allocations for formatted strings
public static string FormatOrderId(long orderId)
{
    return string.Create(15, orderId, (span, id) =>
    {
        span[0] = 'O';
        span[1] = 'R';
        span[2] = 'D';
        span[3] = '-';
        id.TryFormat(span[4..], out _, "D10");
    });
}

// String interning — reuse common strings
var statusCache = new ConcurrentDictionary<string, string>();
var interned = statusCache.GetOrAdd(rawStatus, string.Intern);
```

---

## Step 1956: Profiling Checklist and Common Anti-Patterns

```csharp
// Anti-pattern 1: LINQ inside tight loops
// WRONG
foreach (var order in orders)
{
    var total = order.Items.Sum(i => i.Price); // enumerates every iteration
}

// RIGHT — compute once
var totals = orders.ToDictionary(o => o.Id, o => o.Items.Sum(i => i.Price));

// Anti-pattern 2: async void (fire-and-forget without error handling)
// WRONG
public async void ProcessOrderAsync() { ... } // exceptions are unobserved!

// RIGHT
public async Task ProcessOrderAsync() { ... }
// For fire-and-forget:
_ = ProcessOrderAsync().ContinueWith(t =>
    _logger.LogError(t.Exception, "ProcessOrder failed"),
    TaskContinuationOptions.OnlyOnFaulted);

// Anti-pattern 3: Not using CancellationToken
// WRONG
public async Task<Order> GetOrderAsync(Guid id)
{
    return await _db.Orders.FindAsync(id); // ignores cancellation
}

// RIGHT
public async Task<Order?> GetOrderAsync(Guid id, CancellationToken ct = default)
{
    return await _db.Orders.FindAsync([id], ct);
}

// Anti-pattern 4: Premature string conversion
// WRONG
var key = orderId.ToString() + "_" + userId.ToString(); // 3 string allocations

// RIGHT — use interpolation (compiler optimizes to composite)
var key = $"{orderId}_{userId}";

// Anti-pattern 5: Exceptions for control flow
// WRONG
try { return int.Parse(input); } catch { return 0; }

// RIGHT
return int.TryParse(input, out var result) ? result : 0;

// Anti-pattern 6: Task.Run in ASP.NET (wastes threads)
// WRONG
var result = await Task.Run(() => _service.ComputeSync());

// RIGHT — make it truly async or move CPU work to background service
```

```
Performance Checklist:
□ Profiled before optimizing (measure, don't guess)
□ No blocking calls (Result/.Wait) in async code
□ CancellationToken propagated everywhere
□ EF Core queries project only needed columns
□ No N+1 queries (check EF Core logging)
□ AsNoTracking() on read-only queries
□ Cached frequently-read, rarely-changed data
□ ArrayPool/ObjectPool for large or frequent allocations
□ Span<T> for string parsing in hot paths
□ BenchmarkDotNet before/after for hot methods
□ dotnet-counters GC metrics acceptable
□ Gen2 GC rate is low (< 1/min under load)
```

---

## Summary

| Technique | Use Case | Allocation Impact |
|-----------|----------|-------------------|
| `Span<T>` | String parsing, buffer slicing | Zero allocation |
| `ArrayPool<T>` | Byte buffers, large temp arrays | Reuse — near-zero |
| `Memory<T>` | Async buffers | Heap, but reused |
| `ValueTask<T>` | Cached/sync-path async | Near-zero (sync path) |
| `IAsyncEnumerable<T>` | Streaming large result sets | Low (one item at a time) |
| `AsNoTracking()` | Read-only EF queries | Reduced per-entity allocation |
| `ExecuteUpdate/Delete` | Bulk updates/deletes | No entity loading |
| `HybridCache` | L1+L2 cache | Redis round-trip saved |
| `Output Cache` | HTTP response cache | Zero per-request work |
| `Parallel.ForEachAsync` | CPU-bound fan-out | Thread-pool bound |
| `BenchmarkDotNet` | Measure actual change | — |

**Next**: Part 83 — Security Hardening & Secure Coding Practices
