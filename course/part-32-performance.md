# Part 32: Performance Optimization & Profiling

## Steps 881-915: Writing High-Performance .NET Code

---

## Step 881: Performance Mindset

การ optimize ที่ดีต้องเริ่มจากการ **measure** ก่อนเสมอ:
1. **Identify**: ระบุ bottleneck ด้วย profiler
2. **Measure**: วัด baseline
3. **Optimize**: แก้ไขจุดที่ช้า
4. **Verify**: ตรวจสอบว่าดีขึ้น
5. **Repeat**: ทำซ้ำจนพอใจ

---

## Step 882: BenchmarkDotNet

```bash
dotnet add package BenchmarkDotNet
dotnet add package BenchmarkDotNet.Diagnostics.Windows  # Windows
```

```csharp
// Benchmarks/StringBenchmarks.cs
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Running;

[MemoryDiagnoser]
[RankColumn]
[Orderer(SummaryOrderPolicy.FastestToSlowest)]
public class StringConcatenationBenchmarks
{
    private readonly string[] _parts = 
        Enumerable.Range(1, 100).Select(i => $"Part{i}").ToArray();
    
    [Benchmark(Baseline = true)]
    public string StringConcat()
    {
        var result = "";
        foreach (var part in _parts)
            result += part;
        return result;
    }
    
    [Benchmark]
    public string StringBuilder()
    {
        var sb = new System.Text.StringBuilder();
        foreach (var part in _parts)
            sb.Append(part);
        return sb.ToString();
    }
    
    [Benchmark]
    public string StringJoin()
        => string.Join("", _parts);
    
    [Benchmark]
    public string SpanBased()
    {
        var totalLength = _parts.Sum(p => p.Length);
        var buffer = new char[totalLength];
        var pos = 0;
        
        foreach (var part in _parts)
        {
            part.AsSpan().CopyTo(buffer.AsSpan(pos));
            pos += part.Length;
        }
        
        return new string(buffer);
    }
}

// Program.cs
BenchmarkRunner.Run<StringConcatenationBenchmarks>();
```

---

## Step 883: Span<T> และ Memory<T>

```csharp
// Span<T> คือ stack-allocated view ของ contiguous memory - zero allocation
// ไม่สามารถใช้ใน async methods หรือ heap allocation

public static class SpanExamples
{
    // Parse integers โดยไม่ allocate strings
    public static int ParseIntFast(ReadOnlySpan<char> input)
    {
        var result = 0;
        var isNegative = false;
        var start = 0;
        
        if (input[0] == '-')
        {
            isNegative = true;
            start = 1;
        }
        
        for (var i = start; i < input.Length; i++)
        {
            result = result * 10 + (input[i] - '0');
        }
        
        return isNegative ? -result : result;
    }
    
    // Split string โดยไม่ allocate
    public static void SplitSpan(ReadOnlySpan<char> input, char separator)
    {
        while (!input.IsEmpty)
        {
            var index = input.IndexOf(separator);
            if (index < 0)
            {
                Process(input);
                break;
            }
            
            Process(input[..index]);
            input = input[(index + 1)..];
        }
    }
    
    private static void Process(ReadOnlySpan<char> token)
    {
        // Process without allocating
        Console.WriteLine($"Token: {token.ToString()}");
    }
    
    // Stack allocation with stackalloc
    public static int SumStackAlloc(int count)
    {
        // Allocate on stack - no GC pressure
        Span<int> numbers = count <= 256 
            ? stackalloc int[count]
            : new int[count]; // heap fallback
        
        for (var i = 0; i < count; i++)
            numbers[i] = i + 1;
        
        return numbers.Sum();
    }
    
    // Efficient buffer manipulation
    public static byte[] TransformData(ReadOnlySpan<byte> input)
    {
        var output = new byte[input.Length];
        
        for (var i = 0; i < input.Length; i++)
            output[i] = (byte)(input[i] ^ 0xFF); // XOR transform
        
        return output;
    }
    
    // Memory<T> สำหรับ async
    public static async Task<string> ReadAndProcessAsync(
        Stream stream,
        CancellationToken ct)
    {
        var buffer = new byte[4096];
        Memory<byte> memory = buffer;
        
        var totalRead = 0;
        int bytesRead;
        
        while ((bytesRead = await stream.ReadAsync(memory[totalRead..], ct)) > 0)
        {
            totalRead += bytesRead;
            if (totalRead >= buffer.Length) break;
        }
        
        return System.Text.Encoding.UTF8.GetString(memory.Span[..totalRead]);
    }
}
```

---

## Step 884: ArrayPool<T>

```csharp
using System.Buffers;

public class ArrayPoolExample
{
    // ไม่ดี - allocate ใหม่ทุกครั้ง
    public static byte[] ProcessDataBad(int size)
    {
        var buffer = new byte[size]; // heap allocation
        // ... process
        return buffer;
    }
    
    // ดี - rent from pool
    public static byte[] ProcessDataGood(int size)
    {
        var pool = ArrayPool<byte>.Shared;
        var buffer = pool.Rent(size);
        
        try
        {
            // ... process using buffer
            var result = new byte[size];
            Array.Copy(buffer, result, size);
            return result;
        }
        finally
        {
            pool.Return(buffer, clearArray: true);
        }
    }
    
    // ดีที่สุด - ใช้ IBufferWriter
    public static string ProcessWithWriter(ReadOnlySpan<byte> input)
    {
        var writer = new ArrayBufferWriter<byte>();
        
        // Write to writer without allocating
        writer.Write(input);
        
        return System.Text.Encoding.UTF8.GetString(writer.WrittenSpan);
    }
    
    // Custom pool สำหรับ objects
    private static readonly ObjectPool<StringBuilder> _sbPool =
        new DefaultObjectPool<StringBuilder>(
            new StringBuilderPooledObjectPolicy(),
            maximumRetained: 32);
    
    public static string BuildString(IEnumerable<string> parts)
    {
        var sb = _sbPool.Get();
        try
        {
            foreach (var part in parts)
                sb.Append(part);
            return sb.ToString();
        }
        finally
        {
            _sbPool.Return(sb);
        }
    }
}
```

---

## Step 885: Collections Performance

```csharp
// เลือก collection ที่ถูกต้อง
public class CollectionPerformance
{
    // List<T> - O(1) access, O(n) insert/remove middle
    // Dictionary<TK,TV> - O(1) lookup, unordered
    // SortedDictionary<TK,TV> - O(log n) lookup, ordered
    // HashSet<T> - O(1) contains, unique values
    // Queue<T> - O(1) enqueue/dequeue, FIFO
    // Stack<T> - O(1) push/pop, LIFO
    // LinkedList<T> - O(1) insert/remove, O(n) access
    // PriorityQueue<T,P> - O(log n) dequeue (min/max)
    
    // ระบุ capacity เพื่อหลีกเลี่ยง resize
    public static List<int> GoodList(int expectedSize)
    {
        var list = new List<int>(expectedSize); // no resize!
        for (var i = 0; i < expectedSize; i++)
            list.Add(i);
        return list;
    }
    
    // Dictionary lookups
    public static void DictionaryTips()
    {
        var dict = new Dictionary<string, int>(StringComparer.OrdinalIgnoreCase)
        {
            ["hello"] = 1
        };
        
        // TryGetValue - single lookup
        if (dict.TryGetValue("Hello", out var value))
            Console.WriteLine(value);
        
        // CollectionsMarshal for in-place update (no double lookup)
        ref var val = ref System.Runtime.InteropServices.CollectionsMarshal
            .GetValueRefOrAddDefault(dict, "world", out var exists);
        val = exists ? val + 1 : 1;
    }
    
    // Frozen collections (.NET 8) - immutable, optimized for read
    public static void FrozenCollections()
    {
        var dict = new Dictionary<string, int>
        {
            ["a"] = 1, ["b"] = 2, ["c"] = 3
        };
        
        var frozen = dict.ToFrozenDictionary(); // read-only, faster lookup
        var set = new[] { "x", "y", "z" }.ToFrozenSet();
        
        // Lookups on frozen are ~20-30% faster
        _ = frozen["a"];
        _ = set.Contains("x");
    }
}
```

---

## Step 886: LINQ Performance

```csharp
using System.Linq;

public class LinqPerformance
{
    private readonly List<int> _numbers = 
        Enumerable.Range(1, 1_000_000).ToList();
    
    // ช้า: multiple enumerations
    public int BadLinq()
    {
        var evenNumbers = _numbers.Where(n => n % 2 == 0); // deferred
        return evenNumbers.Count() + evenNumbers.Sum();     // two enumerations!
    }
    
    // ดี: materialize once
    public int GoodLinq()
    {
        var evenNumbers = _numbers.Where(n => n % 2 == 0).ToList();
        return evenNumbers.Count + evenNumbers.Sum();
    }
    
    // ดีที่สุด: single pass
    public (int Count, int Sum) BestLinq()
    {
        var count = 0;
        var sum = 0;
        
        foreach (var n in _numbers)
        {
            if (n % 2 == 0)
            {
                count++;
                sum += n;
            }
        }
        
        return (count, sum);
    }
    
    // Parallel LINQ for CPU-intensive work
    public double ParallelSum()
    {
        return _numbers
            .AsParallel()
            .WithDegreeOfParallelism(Environment.ProcessorCount)
            .Where(n => n % 2 == 0)
            .Select(n => Math.Sqrt(n))
            .Sum();
    }
    
    // Avoid capturing in closures
    public int GoodClosure(int threshold)
    {
        // Bad: captures 'threshold' variable
        // return _numbers.Where(n => n > threshold).Count();
        
        // Good (same thing but explicit - no issue here actually)
        return _numbers.Count(n => n > threshold);
    }
}
```

---

## Step 887: Async Performance

```csharp
public class AsyncPerformance
{
    // ValueTask<T> สำหรับ hot paths (often synchronous)
    public ValueTask<int> GetCachedValueAsync(string key)
    {
        if (_cache.TryGetValue(key, out var cached))
            return ValueTask.FromResult(cached); // no heap allocation!
        
        return new ValueTask<int>(LoadFromDatabaseAsync(key));
    }
    
    private async Task<int> LoadFromDatabaseAsync(string key) => 0; // stub
    private readonly Dictionary<string, int> _cache = [];
    
    // ไม่ต้อง async/await เมื่อไม่มี logic หลัง await
    // Bad:
    public async Task<string> BadWrap()
    {
        return await GetDataAsync(); // unnecessary state machine
    }
    
    // Good:
    public Task<string> GoodWrap()
    {
        return GetDataAsync(); // no overhead
    }
    
    private Task<string> GetDataAsync() => Task.FromResult("data");
    
    // ConfigureAwait(false) ใน library code
    public async Task<int> LibraryMethod(CancellationToken ct)
    {
        var data = await GetDataAsync().ConfigureAwait(false);
        var processed = await ProcessAsync(data, ct).ConfigureAwait(false);
        return processed.Length;
    }
    
    private async Task<string> ProcessAsync(string s, CancellationToken ct)
    {
        await Task.Delay(1, ct).ConfigureAwait(false);
        return s;
    }
    
    // Concurrent async operations
    public async Task<(string A, string B, string C)> ConcurrentOpsAsync(
        CancellationToken ct)
    {
        // รัน parallel
        var taskA = GetDataAsync();
        var taskB = GetDataAsync();
        var taskC = GetDataAsync();
        
        // Wait for all
        var results = await Task.WhenAll(taskA, taskB, taskC).ConfigureAwait(false);
        return (results[0], results[1], results[2]);
    }
    
    // Cancellation propagation
    public async Task<string> CancelableOperation(CancellationToken ct)
    {
        using var cts = CancellationTokenSource.CreateLinkedTokenSource(ct);
        cts.CancelAfter(TimeSpan.FromSeconds(5));
        
        return await GetDataAsync().WaitAsync(cts.Token);
    }
    
    // Channel for producer-consumer
    public async Task ProducerConsumerAsync(CancellationToken ct)
    {
        var channel = System.Threading.Channels.Channel.CreateBounded<int>(
            new System.Threading.Channels.BoundedChannelOptions(100)
            {
                FullMode = System.Threading.Channels.BoundedChannelFullMode.Wait
            });
        
        var producer = Task.Run(async () =>
        {
            for (var i = 0; i < 1000; i++)
            {
                await channel.Writer.WriteAsync(i, ct);
            }
            channel.Writer.Complete();
        }, ct);
        
        var consumer = Task.Run(async () =>
        {
            await foreach (var item in channel.Reader.ReadAllAsync(ct))
            {
                // Process item
                Console.WriteLine(item);
            }
        }, ct);
        
        await Task.WhenAll(producer, consumer);
    }
}
```

---

## Step 888: Memory Management

```csharp
// IDisposable pattern
public sealed class ResourceHolder : IDisposable
{
    private Stream? _stream;
    private bool _disposed;
    
    public ResourceHolder()
    {
        _stream = new MemoryStream();
    }
    
    public void Write(byte[] data)
    {
        ObjectDisposedException.ThrowIf(_disposed, this);
        _stream!.Write(data);
    }
    
    public void Dispose()
    {
        if (!_disposed)
        {
            _stream?.Dispose();
            _stream = null;
            _disposed = true;
        }
    }
}

// IAsyncDisposable
public sealed class AsyncResourceHolder : IAsyncDisposable
{
    private HttpClient? _client;
    
    public AsyncResourceHolder()
    {
        _client = new HttpClient();
    }
    
    public async ValueTask DisposeAsync()
    {
        if (_client != null)
        {
            await _client.GetAsync("http://example.com/cleanup")
                .ConfigureAwait(false);
            _client.Dispose();
            _client = null;
        }
    }
}

// WeakReference สำหรับ large objects
public class LargeObjectCache
{
    private readonly Dictionary<string, WeakReference<byte[]>> _cache = [];
    
    public byte[] GetOrCreate(string key, Func<byte[]> factory)
    {
        if (_cache.TryGetValue(key, out var weakRef))
        {
            if (weakRef.TryGetTarget(out var cached))
                return cached;
        }
        
        var data = factory();
        _cache[key] = new WeakReference<byte[]>(data);
        return data;
    }
}

// struct vs class
// struct: stack-allocated, no heap pressure, value semantics
public readonly struct Point3D(double x, double y, double z)
{
    public double X { get; } = x;
    public double Y { get; } = y;
    public double Z { get; } = z;
    
    public double DistanceTo(Point3D other)
    {
        var dx = X - other.X;
        var dy = Y - other.Y;
        var dz = Z - other.Z;
        return Math.Sqrt(dx*dx + dy*dy + dz*dz);
    }
}

// record struct (.NET 6+)
public readonly record struct Temperature(double Celsius)
{
    public double Fahrenheit => Celsius * 9.0 / 5.0 + 32;
    public double Kelvin => Celsius + 273.15;
}
```

---

## Step 889: String Performance

```csharp
using System.Text;

public class StringPerformance
{
    // Avoid string allocations in hot paths
    
    // InterpolatedStringHandler (.NET 6) - avoid alloc for logging
    [System.Runtime.CompilerServices.InterpolatedStringHandler]
    public ref struct LogInterpolatedStringHandler
    {
        private readonly StringBuilder _builder;
        private bool _enabled;
        
        public LogInterpolatedStringHandler(int literalLength, int formattedCount, 
            bool enabled, out bool shouldBuild)
        {
            _enabled = enabled;
            shouldBuild = enabled;
            _builder = enabled ? new StringBuilder(literalLength) : null!;
        }
        
        public void AppendLiteral(string s)
        {
            if (_enabled) _builder.Append(s);
        }
        
        public void AppendFormatted<T>(T value)
        {
            if (_enabled) _builder.Append(value);
        }
        
        public string GetFormattedText() => _builder.ToString();
    }
    
    // String.Create for zero-allocation string creation
    public static string FormatId(int id, string prefix)
    {
        return string.Create(prefix.Length + 10, (id, prefix), (span, state) =>
        {
            state.prefix.AsSpan().CopyTo(span);
            state.id.TryFormat(span[state.prefix.Length..], out _);
        });
    }
    
    // Avoid unnecessary ToString()
    public static void NoUnnecessaryToString(int value)
    {
        // Bad
        var s1 = value.ToString();
        Console.WriteLine(s1);
        
        // Good - Console.WriteLine handles int directly
        Console.WriteLine(value);
    }
    
    // String.Contains vs IndexOf for char
    public static bool ContainsChar(string input, char c)
    {
        // Good: overload for char is optimized
        return input.Contains(c, StringComparison.Ordinal);
    }
    
    // Ordinal comparison for non-linguistic strings
    public static bool CompareUrls(string a, string b)
        => string.Equals(a, b, StringComparison.OrdinalIgnoreCase);
}
```

---

## Step 890: Entity Framework Core Performance

```csharp
public class EFCorePerformance
{
    private readonly AppDbContext _db;
    
    public EFCorePerformance(AppDbContext db) => _db = db;
    
    // AsNoTracking สำหรับ read-only queries
    public async Task<List<ProductDto>> GetProductsReadOnly(CancellationToken ct)
    {
        return await _db.Products
            .AsNoTracking()
            .Where(p => p.IsActive)
            .Select(p => new ProductDto(p.Id, p.Name, p.Price, p.Category))
            .ToListAsync(ct);
    }
    
    // Select เฉพาะ fields ที่ต้องการ (projection)
    public async Task<IEnumerable<string>> GetProductNames(CancellationToken ct)
    {
        return await _db.Products
            .AsNoTracking()
            .Where(p => p.IsActive)
            .Select(p => p.Name)  // only NAME column is fetched
            .ToListAsync(ct);
    }
    
    // Batch operations
    public async Task UpdatePricesAsync(decimal multiplier, CancellationToken ct)
    {
        // EF Core 7+ ExecuteUpdate - no load into memory
        await _db.Products
            .Where(p => p.Category == "Electronics")
            .ExecuteUpdateAsync(setters =>
                setters.SetProperty(p => p.Price, p => p.Price * multiplier),
                ct);
    }
    
    // Batch delete
    public async Task DeleteOldOrdersAsync(DateTime before, CancellationToken ct)
    {
        await _db.Orders
            .Where(o => o.CreatedAt < before && o.Status == "Completed")
            .ExecuteDeleteAsync(ct);
    }
    
    // Compiled queries
    private static readonly Func<AppDbContext, int, Task<Product?>> _getProductById =
        EF.CompileAsyncQuery((AppDbContext db, int id) =>
            db.Products
                .AsNoTracking()
                .FirstOrDefault(p => p.Id == id));
    
    public Task<Product?> GetProductFast(int id)
        => _getProductById(_db, id);
    
    // Split queries for complex includes (prevent cartesian explosion)
    public async Task<List<Product>> GetProductsWithReviews(CancellationToken ct)
    {
        return await _db.Products
            .AsNoTracking()
            .Include(p => p.Reviews)
            .AsSplitQuery()  // Separate SQL queries
            .ToListAsync(ct);
    }
    
    // Raw SQL when EF is too slow
    public async Task<List<ProductStats>> GetProductStats(CancellationToken ct)
    {
        return await _db.Database
            .SqlQuery<ProductStats>(
                $"""
                SELECT 
                    p.Id,
                    p.Name,
                    COUNT(r.Id) as ReviewCount,
                    AVG(r.Rating) as AvgRating
                FROM Products p
                LEFT JOIN Reviews r ON r.ProductId = p.Id
                GROUP BY p.Id, p.Name
                """)
            .ToListAsync(ct);
    }
}
```

---

## Step 891: HTTP Client Performance

```csharp
// IHttpClientFactory - ถูกวิธีในการใช้ HttpClient
builder.Services.AddHttpClient("ProductApi", client =>
{
    client.BaseAddress = new Uri("https://api.products.com");
    client.DefaultRequestHeaders.Add("Accept", "application/json");
    client.Timeout = TimeSpan.FromSeconds(30);
})
.AddStandardResilienceHandler()
.ConfigurePrimaryHttpMessageHandler(() => new SocketsHttpHandler
{
    PooledConnectionLifetime = TimeSpan.FromMinutes(2),
    PooledConnectionIdleTimeout = TimeSpan.FromMinutes(1),
    MaxConnectionsPerServer = 20,
    EnableMultipleHttp2Connections = true,
    AllowAutoRedirect = false
});

// Typed client
public class ProductApiClient(HttpClient client)
{
    public async Task<ProductDto?> GetProductAsync(int id, CancellationToken ct)
    {
        var response = await client
            .GetAsync($"/products/{id}", ct)
            .ConfigureAwait(false);
        
        if (response.StatusCode == HttpStatusCode.NotFound)
            return null;
        
        response.EnsureSuccessStatusCode();
        
        // Use JsonSerializer directly from stream
        return await response.Content
            .ReadFromJsonAsync<ProductDto>(ct)
            .ConfigureAwait(false);
    }
    
    public async Task<List<ProductDto>> GetProductsAsync(
        string category, CancellationToken ct)
    {
        // Build URL without string interpolation for performance
        var url = QueryHelpers.AddQueryString("/products", "category", category);
        
        return await client
            .GetFromJsonAsync<List<ProductDto>>(url, ct)
            .ConfigureAwait(false) ?? [];
    }
}
```

---

## Step 892: Caching Strategies

```csharp
// Multi-level cache
public class MultiLevelCache<T>(
    IMemoryCache l1,
    IDistributedCache l2,
    ILogger<MultiLevelCache<T>> logger)
{
    private readonly string _typeName = typeof(T).Name;
    
    public async Task<T?> GetAsync(string key, CancellationToken ct)
    {
        var fullKey = $"{_typeName}:{key}";
        
        // L1: In-memory (fastest)
        if (l1.TryGetValue(fullKey, out T? l1Value))
        {
            logger.LogDebug("L1 cache hit: {Key}", fullKey);
            return l1Value;
        }
        
        // L2: Distributed cache
        var bytes = await l2.GetAsync(fullKey, ct).ConfigureAwait(false);
        if (bytes != null)
        {
            var l2Value = JsonSerializer.Deserialize<T>(bytes);
            
            // Populate L1
            l1.Set(fullKey, l2Value, TimeSpan.FromMinutes(5));
            logger.LogDebug("L2 cache hit: {Key}", fullKey);
            return l2Value;
        }
        
        return default;
    }
    
    public async Task SetAsync(string key, T value, TimeSpan ttl, CancellationToken ct)
    {
        var fullKey = $"{_typeName}:{key}";
        
        // Set L1 (shorter TTL)
        l1.Set(fullKey, value, ttl < TimeSpan.FromMinutes(5) ? ttl : TimeSpan.FromMinutes(5));
        
        // Set L2
        await l2.SetAsync(
            fullKey,
            JsonSerializer.SerializeToUtf8Bytes(value),
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = ttl
            }, ct).ConfigureAwait(false);
    }
}

// Output caching (.NET 7+)
builder.Services.AddOutputCache(opts =>
{
    opts.DefaultExpirationTimeSpan = TimeSpan.FromMinutes(5);
    
    opts.AddPolicy("Products", policy =>
        policy
            .Expire(TimeSpan.FromMinutes(10))
            .VaryByQuery("category", "page")
            .Tag("products"));
    
    opts.AddPolicy("ProductDetail", policy =>
        policy
            .Expire(TimeSpan.FromMinutes(30))
            .VaryByRouteValue("id")
            .Tag("product"));
});

app.UseOutputCache();

app.MapGet("/api/products", GetProducts)
    .CacheOutput("Products");

app.MapGet("/api/products/{id:int}", GetProduct)
    .CacheOutput("ProductDetail");

// Invalidate cache
app.MapPost("/api/products", async (
    CreateProductRequest request,
    IOutputCacheStore store,
    CancellationToken ct) =>
{
    // ... create product
    await store.EvictByTagAsync("products", ct);
    return Results.Created("/api/products/1", null);
});
```

---

## Step 893: JSON Performance

```csharp
using System.Text.Json;
using System.Text.Json.Serialization;

// Source generation for AOT and performance
[JsonSerializable(typeof(ProductDto))]
[JsonSerializable(typeof(List<ProductDto>))]
[JsonSerializable(typeof(OrderDto))]
[JsonSourceGenerationOptions(
    PropertyNamingPolicy = JsonKnownNamingPolicy.CamelCase,
    DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull,
    WriteIndented = false)]
public partial class AppJsonContext : JsonSerializerContext { }

// Usage
public class JsonPerformance
{
    // Source-generated serialization (faster, AOT-compatible)
    public static byte[] SerializeFast(ProductDto product)
        => JsonSerializer.SerializeToUtf8Bytes(product, AppJsonContext.Default.ProductDto);
    
    public static ProductDto? DeserializeFast(ReadOnlySpan<byte> json)
        => JsonSerializer.Deserialize(json, AppJsonContext.Default.ProductDto);
    
    // Streaming JSON for large payloads
    public static async IAsyncEnumerable<ProductDto> StreamLargeJson(
        Stream jsonStream,
        [System.Runtime.CompilerServices.EnumeratorCancellation] CancellationToken ct)
    {
        await foreach (var product in JsonSerializer
            .DeserializeAsyncEnumerable<ProductDto>(
                jsonStream,
                AppJsonContext.Default.ProductDto,
                ct))
        {
            if (product != null)
                yield return product;
        }
    }
    
    // Utf8JsonWriter for high-performance writing
    public static byte[] WriteCustomJson(IEnumerable<ProductDto> products)
    {
        var buffer = new ArrayBufferWriter<byte>();
        using var writer = new Utf8JsonWriter(buffer);
        
        writer.WriteStartArray();
        foreach (var product in products)
        {
            writer.WriteStartObject();
            writer.WriteNumber("id"u8, product.Id);
            writer.WriteString("name"u8, product.Name);
            writer.WriteNumber("price"u8, product.Price);
            writer.WriteEndObject();
        }
        writer.WriteEndArray();
        
        writer.Flush();
        return buffer.WrittenSpan.ToArray();
    }
}
```

---

## Step 894: Thread Pool และ Task Management

```csharp
public class ThreadManagement
{
    // ไม่บล็อก thread pool threads
    
    // Bad: blocking async code
    public void BadBlocking()
    {
        var result = GetDataAsync().Result; // blocks thread
        var result2 = GetDataAsync().GetAwaiter().GetResult(); // also blocks
    }
    
    // Good: async all the way
    public async Task GoodAsync()
    {
        var result = await GetDataAsync();
    }
    
    private Task<string> GetDataAsync() => Task.FromResult("data");
    
    // Semaphore for resource limiting
    private readonly SemaphoreSlim _semaphore = new(10, 10);
    
    public async Task<string> LimitedConcurrency(CancellationToken ct)
    {
        await _semaphore.WaitAsync(ct);
        try
        {
            return await GetDataAsync();
        }
        finally
        {
            _semaphore.Release();
        }
    }
    
    // Parallel processing with bounded concurrency
    public async Task ProcessInParallel<T>(
        IEnumerable<T> items,
        Func<T, CancellationToken, Task> processor,
        int maxConcurrency,
        CancellationToken ct)
    {
        var options = new ParallelOptions
        {
            MaxDegreeOfParallelism = maxConcurrency,
            CancellationToken = ct
        };
        
        await Parallel.ForEachAsync(items, options, 
            async (item, token) => await processor(item, token));
    }
    
    // Long running CPU work on dedicated thread
    public async Task<int> CpuBoundWork()
    {
        return await Task.Run(() =>
        {
            // CPU-intensive: matrix multiplication, encryption, etc.
            var result = 0;
            for (var i = 0; i < 1_000_000_000; i++)
                result += i;
            return result;
        });
    }
    
    // Lazy initialization (thread-safe)
    private readonly Lazy<ExpensiveObject> _lazyObj = 
        new(() => new ExpensiveObject(), LazyThreadSafetyMode.ExecutionAndPublication);
    
    public ExpensiveObject GetExpensiveObject() => _lazyObj.Value;
}

public class ExpensiveObject
{
    public ExpensiveObject()
    {
        Thread.Sleep(100); // simulate expensive init
    }
}
```

---

## Step 895: Profiling Tools

```csharp
// dotnet-trace: CLI profiling
// dotnet tool install -g dotnet-trace
// dotnet-trace collect --process-id <pid> --duration 00:00:30

// dotnet-counters: monitor metrics
// dotnet tool install -g dotnet-counters
// dotnet-counters monitor --process-id <pid>

// dotnet-dump: memory dump analysis
// dotnet tool install -g dotnet-dump
// dotnet-dump collect --process-id <pid>
// dotnet-dump analyze dump.dmp

// Activity tracking for custom profiling
using System.Diagnostics;

public class ProfilingExample
{
    private static readonly ActivitySource ActivitySource = 
        new("MyApp.Performance");
    
    public async Task<string> TrackedOperation(string input)
    {
        using var activity = ActivitySource.StartActivity("ProcessInput");
        activity?.SetTag("input.length", input.Length);
        
        var sw = Stopwatch.StartNew();
        
        try
        {
            var result = await ProcessAsync(input);
            
            activity?.SetTag("result.length", result.Length);
            activity?.SetStatus(ActivityStatusCode.Ok);
            
            return result;
        }
        catch (Exception ex)
        {
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            activity?.RecordException(ex);
            throw;
        }
        finally
        {
            sw.Stop();
            activity?.SetTag("duration.ms", sw.ElapsedMilliseconds);
        }
    }
    
    private Task<string> ProcessAsync(string s) => Task.FromResult(s.ToUpper());
}

// EventCounters for custom metrics
public class AppEventSource : EventSource
{
    public static readonly AppEventSource Log = new();
    
    private EventCounter? _requestDuration;
    private PollingCounter? _activeRequests;
    private long _activeRequestCount;
    
    protected override void OnEventSourceCreated()
    {
        _requestDuration = new EventCounter("request-duration", this);
        _activeRequests = new PollingCounter("active-requests", this, 
            () => Interlocked.Read(ref _activeRequestCount));
    }
    
    public void RequestStart()
        => Interlocked.Increment(ref _activeRequestCount);
    
    public void RequestEnd(double durationMs)
    {
        Interlocked.Decrement(ref _activeRequestCount);
        _requestDuration?.WriteMetric(durationMs);
    }
}
```

---

## Step 896: Native AOT

```xml
<!-- MyApp.csproj -->
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <PublishAot>true</PublishAot>
    <InvariantGlobalization>true</InvariantGlobalization>
  </PropertyGroup>
</Project>
```

```csharp
// AOT-compatible code
// ใช้ source generators แทน reflection

// Bad for AOT: reflection
var props = typeof(MyClass).GetProperties();

// Good for AOT: source generation
[JsonSerializable(typeof(MyClass))]
partial class MyJsonContext : JsonSerializerContext { }

// Request delegate generator for AOT-compatible Minimal APIs
// ใช้ [FromBody], [FromQuery] etc. attributes
app.MapPost("/products", (
    [FromBody] CreateProductRequest request,
    [FromServices] IProductService service,
    CancellationToken ct) => service.CreateProductAsync(request, ct));

// Publish AOT binary
// dotnet publish -r linux-x64 -c Release
// Binary size: ~20MB vs ~100MB for self-contained
// Startup time: ~50ms vs ~300ms
```

---

## Step 897: Unsafe Code และ Interop

```csharp
// unsafe สำหรับ pointer operations
public static class UnsafePerformance
{
    // Reinterpret bytes directly
    public static unsafe float BytesToFloat(byte[] bytes)
    {
        fixed (byte* ptr = bytes)
            return *(float*)ptr;
    }
    
    // MemoryMarshal - safe wrapper for unsafe operations
    public static float BytesToFloatSafe(ReadOnlySpan<byte> bytes)
        => System.Runtime.InteropServices.MemoryMarshal.Read<float>(bytes);
    
    // Inline array (.NET 8) - fixed-size array on stack
    [System.Runtime.CompilerServices.InlineArray(16)]
    public struct FixedBuffer
    {
        private byte _element;
    }
    
    public static void UseInlineArray()
    {
        FixedBuffer buffer = default;
        buffer[0] = 0xFF;
        buffer[15] = 0x00;
        
        // Span view
        Span<byte> span = buffer;
        span.Fill(0xAA);
    }
    
    // P/Invoke for native code
    [System.Runtime.InteropServices.LibraryImport("libc", StringMarshalling = StringMarshalling.Utf8)]
    private static partial int strlen(string s);
    
    public static int GetStringLength(string s) => strlen(s);
}
```

---

## Step 898: Response Compression และ Caching HTTP

```csharp
// Response compression
builder.Services.AddResponseCompression(opts =>
{
    opts.EnableForHttps = true;
    opts.Providers.Add<BrotliCompressionProvider>();
    opts.Providers.Add<GzipCompressionProvider>();
    opts.MimeTypes = ResponseCompressionDefaults.MimeTypes.Concat([
        "application/json",
        "application/xml",
        "text/plain"
    ]);
});

builder.Services.Configure<BrotliCompressionProviderOptions>(opts =>
{
    opts.Level = System.IO.Compression.CompressionLevel.Fastest;
});

app.UseResponseCompression();

// HTTP cache headers
app.MapGet("/api/products", async (IProductService service, HttpContext ctx, CancellationToken ct) =>
{
    var products = await service.GetProductsAsync(ct);
    
    // ETag
    var etag = $"\"{products.GetHashCode()}\"";
    
    if (ctx.Request.Headers.IfNoneMatch == etag)
        return Results.StatusCode(304); // Not Modified
    
    ctx.Response.Headers.ETag = etag;
    ctx.Response.Headers.CacheControl = "public, max-age=300"; // 5 minutes
    ctx.Response.Headers.Vary = "Accept-Encoding";
    
    return Results.Ok(products);
});
```

---

## Step 899: Load Testing

```csharp
// k6 load test script (JavaScript)
// import http from 'k6/http';
// import { check, sleep } from 'k6';

var k6Script = """
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '30s', target: 20 },   // Ramp up
    { duration: '1m', target: 100 },   // Stay at 100 users
    { duration: '30s', target: 200 },  // Ramp up more
    { duration: '2m', target: 200 },   // Stay at peak
    { duration: '30s', target: 0 },    // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],   // 95% < 500ms
    http_req_failed: ['rate<0.01'],     // Error rate < 1%
  },
};

export default function () {
  const res = http.get('http://localhost:5000/api/products');
  
  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 200ms': (r) => r.timings.duration < 200,
  });
  
  sleep(1);
}
""";
```

```bash
# Install k6
# brew install k6
# k6 run load-test.js

# NBomber (.NET load testing)
dotnet add package NBomber
dotnet add package NBomber.Http
```

```csharp
// NBomber load test in C#
using NBomber.CSharp;
using NBomber.Http.CSharp;

var httpClient = new HttpClient();
httpClient.BaseAddress = new Uri("http://localhost:5000");

var scenario = Scenario.Create("get_products", async context =>
{
    var response = await httpClient.GetAsync("/api/products");
    
    return response.IsSuccessStatusCode
        ? Response.Ok(statusCode: (int)response.StatusCode)
        : Response.Fail(statusCode: (int)response.StatusCode);
})
.WithLoadSimulations(
    Simulation.Inject(rate: 10, interval: TimeSpan.FromSeconds(1), 
        during: TimeSpan.FromMinutes(1)),
    Simulation.KeepConstant(copies: 50, during: TimeSpan.FromMinutes(5))
);

NBomberRunner
    .RegisterScenarios(scenario)
    .WithReportFormats(ReportFormat.Html, ReportFormat.Csv)
    .Run();
```

---

## Step 900: Milestone - สรุป Performance Tips

### Top 10 Performance Rules

1. **Measure first** - ใช้ BenchmarkDotNet, profiler
2. **Avoid allocations** - Span<T>, ArrayPool, stackalloc
3. **AsNoTracking** - สำหรับ read-only EF queries
4. **Compiled queries** - EF.CompileAsyncQuery
5. **Caching** - Memory cache → Distributed cache → CDN
6. **Async/await correctly** - ไม่ block, ConfigureAwait(false)
7. **Choose right collection** - FrozenDictionary, capacity hints
8. **Parallel work** - Task.WhenAll, Parallel.ForEachAsync
9. **JSON source gen** - [JsonSerializable], Utf8JsonWriter
10. **Database indexes** - ตรวจ query plans

### Performance Benchmarks ที่ควรรู้

| Operation | Approximate Time |
|-----------|-----------------|
| L1 cache access | 0.5 ns |
| L2 cache access | 3-4 ns |
| RAM access | 60-100 ns |
| Redis GET | ~100 μs |
| Database query | ~1-10 ms |
| HTTP call (local) | ~1-5 ms |
| HTTP call (internet) | ~50-300 ms |
| Disk read (SSD) | ~10-50 μs |

---

## Step 901-915: Advanced Performance Topics

### Step 901: SIMD and Vectorization

```csharp
using System.Numerics;
using System.Runtime.Intrinsics;
using System.Runtime.Intrinsics.X86;

public static class VectorizedMath
{
    // Sum array using SIMD
    public static float SumVectorized(float[] array)
    {
        if (!Vector.IsHardwareAccelerated)
            return array.Sum();
        
        var vectorSize = Vector<float>.Count;
        var sum = Vector<float>.Zero;
        var i = 0;
        
        for (; i <= array.Length - vectorSize; i += vectorSize)
        {
            var v = new Vector<float>(array, i);
            sum += v;
        }
        
        var total = Vector.Dot(sum, Vector<float>.One);
        
        // Handle remaining elements
        for (; i < array.Length; i++)
            total += array[i];
        
        return total;
    }
    
    // Find max using AVX2 (x86 specific)
    public static unsafe float MaxAvx2(float[] array)
    {
        if (!Avx2.IsSupported || array.Length < 8)
            return array.Max();
        
        fixed (float* ptr = array)
        {
            var maxVec = Vector256<float>.Zero;
            var i = 0;
            
            for (; i <= array.Length - 8; i += 8)
            {
                var v = Avx.LoadVector256(ptr + i);
                maxVec = Avx.Max(maxVec, v);
            }
            
            // Horizontal max
            var lo = Vector256.GetLower(maxVec);
            var hi = Vector256.GetUpper(maxVec);
            var max4 = Sse.Max(lo, hi);
            var max2 = Sse.Max(max4, Sse.MoveHighToLow(max4, max4));
            var max1 = Sse.MaxScalar(max2, Sse.Shuffle(max2, max2, 1));
            
            var result = max1.ToScalar();
            
            for (; i < array.Length; i++)
                result = Math.Max(result, array[i]);
            
            return result;
        }
    }
}
```

---

*จบ Part 32: Performance Optimization & Profiling*
*ต่อไป Part 33: Security in .NET*
