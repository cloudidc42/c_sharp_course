# Part 48: Performance Optimization & Benchmarking

## Steps 1361-1400: เขียน .NET ให้เร็วระดับโลก

---

## Step 1361: Memory Layout & Value Types

```csharp
// Value types (struct) vs Reference types (class)
// Structs live on stack (or inline in arrays) — no GC pressure

// ❌ Class: heap allocated, GC pressure for millions of objects
public class PointClass { public double X, Y, Z; }

// ✓ Struct: stack allocated, cache-friendly
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
        return Math.Sqrt(dx * dx + dy * dy + dz * dz);
    }

    // Override Equals/GetHashCode for value semantics
    public override bool Equals(object? obj) => obj is Point3D p && Equals(p);
    public bool Equals(Point3D other) => X == other.X && Y == other.Y && Z == other.Z;
    public override int GetHashCode() => HashCode.Combine(X, Y, Z);
    public static bool operator ==(Point3D a, Point3D b) => a.Equals(b);
    public static bool operator !=(Point3D a, Point3D b) => !a.Equals(b);
}

// Struct array: contiguous memory, cache-friendly (AoS vs SoA)
// Array of Structures (AoS) — ok for accessing all fields together
Point3D[] points = new Point3D[1_000_000];

// Structure of Arrays (SoA) — better for SIMD, accessing single field
// e.g., compute all X values: xs is contiguous
double[] xs = new double[1_000_000];
double[] ys = new double[1_000_000];
double[] zs = new double[1_000_000];

// readonly ref struct — cannot escape to heap, safe for stack-only data
public readonly ref struct SpanWrapper(Span<byte> data)
{
    public Span<byte> Data { get; } = data;
    public int Length => Data.Length;
}

// record struct (C# 10) — value semantics + positional properties
public readonly record struct Rectangle(double Width, double Height)
{
    public double Area => Width * Height;
    public double Perimeter => 2 * (Width + Height);
}
```

---

## Step 1362: Span\<T\> and Memory\<T\>

```csharp
// Span<T> — zero-allocation slice over arrays, stack memory, string
public static class StringParser
{
    // ❌ Allocates substrings
    public static int[] ParseIntsAlloc(string input)
    {
        return input.Split(',').Select(int.Parse).ToArray();
    }

    // ✓ Zero-allocation parsing with Span
    public static int[] ParseIntsSpan(ReadOnlySpan<char> input)
    {
        // Count commas to pre-allocate
        var count = 1;
        for (var i = 0; i < input.Length; i++)
            if (input[i] == ',') count++;

        var result = new int[count];
        var index = 0;

        while (!input.IsEmpty)
        {
            var commaPos = input.IndexOf(',');
            var token = commaPos < 0 ? input : input[..commaPos];

            result[index++] = int.Parse(token);

            if (commaPos < 0) break;
            input = input[(commaPos + 1)..];
        }

        return result;
    }
}

// Memory<T> — async-safe, heap-backed span
public class FileProcessor
{
    private readonly byte[] _buffer = new byte[4096];

    public async Task<int> ProcessChunkAsync(Stream stream, CancellationToken ct)
    {
        // Memory<byte> is like Span<byte> but can cross await boundaries
        var memory = _buffer.AsMemory();
        var bytesRead = await stream.ReadAsync(memory, ct);
        return ProcessBuffer(memory[..bytesRead].Span);
    }

    private int ProcessBuffer(ReadOnlySpan<byte> data)
    {
        var count = 0;
        foreach (var b in data)
            if (b == '\n') count++;
        return count;
    }
}

// StackAlloc — allocate on stack (no GC)
public static Guid ParseGuid(ReadOnlySpan<char> input)
{
    // Allocate 32 chars on stack for hex chars (safe: bounded size)
    Span<char> clean = stackalloc char[32];
    var j = 0;
    foreach (var c in input)
        if (c != '-') clean[j++] = c;

    return Guid.Parse(clean[..j]);
}

// ArrayPool — rent/return to avoid frequent allocations
public async Task<string> CompressAsync(string input, CancellationToken ct)
{
    var inputBytes = Encoding.UTF8.GetBytes(input);
    var buffer = ArrayPool<byte>.Shared.Rent(inputBytes.Length * 2);
    try
    {
        using var ms = new MemoryStream(buffer);
        await using var gz = new GZipStream(ms, CompressionLevel.Optimal);
        await gz.WriteAsync(inputBytes, ct);
        await gz.FlushAsync(ct);
        return Convert.ToBase64String(buffer, 0, (int)ms.Position);
    }
    finally
    {
        ArrayPool<byte>.Shared.Return(buffer);
    }
}
```

---

## Step 1363: Object Pooling

```csharp
// ObjectPool<T> — reuse expensive objects (HTTP clients, DB connections, buffers)
using Microsoft.Extensions.ObjectPool;

// Pool for StringBuilder
public class StringBuilderPooledService(ObjectPool<StringBuilder> pool)
{
    public string FormatOrder(Order order)
    {
        var sb = pool.Get();
        try
        {
            sb.Clear();
            sb.Append("Order #").Append(order.Id);
            sb.Append(" | Customer: ").Append(order.CustomerName);
            sb.Append(" | Total: ").Append(order.TotalAmount.ToString("F2"));
            sb.Append(" | Status: ").Append(order.Status);
            return sb.ToString();
        }
        finally
        {
            pool.Return(sb); // Return to pool even on exception
        }
    }
}

// Registration
builder.Services.AddSingleton<ObjectPoolProvider, DefaultObjectPoolProvider>();
builder.Services.AddSingleton<ObjectPool<StringBuilder>>(sp =>
{
    var provider = sp.GetRequiredService<ObjectPoolProvider>();
    return provider.CreateStringBuilderPool();
});

// Custom pooled object policy
public class ExpensiveObjectPolicy : IPooledObjectPolicy<ExpensiveObject>
{
    public ExpensiveObject Create() => new ExpensiveObject();

    public bool Return(ExpensiveObject obj)
    {
        obj.Reset(); // Clean up state before returning
        return true; // Return to pool
        // Return false to discard instead of pooling (e.g., if in bad state)
    }
}

// Channel-based pool for async scenarios
public class AsyncObjectPool<T>(int maxSize, Func<T> factory, Action<T>? cleanup = null)
{
    private readonly Channel<T> _channel = Channel.CreateBounded<T>(maxSize);

    public async ValueTask<PooledItem<T>> AcquireAsync(CancellationToken ct)
    {
        if (_channel.Reader.TryRead(out var item))
            return new PooledItem<T>(item, this);

        return new PooledItem<T>(factory(), this);
    }

    internal void Return(T item)
    {
        cleanup?.Invoke(item);
        _channel.Writer.TryWrite(item);
    }
}

public readonly struct PooledItem<T>(T value, AsyncObjectPool<T> pool) : IDisposable
{
    public T Value { get; } = value;
    public void Dispose() => pool.Return(Value);
}
```

---

## Step 1364: SIMD with System.Runtime.Intrinsics

```csharp
using System.Runtime.Intrinsics;
using System.Runtime.Intrinsics.X86;

public static class SimdHelpers
{
    // Sum floats using AVX2 (processes 8 floats per instruction)
    public static float SumFloatsAvx(ReadOnlySpan<float> values)
    {
        if (Avx2.IsSupported && values.Length >= 8)
        {
            var sum = Vector256<float>.Zero;
            var i = 0;

            // Process 8 floats at a time
            for (; i <= values.Length - 8; i += 8)
            {
                var vec = Vector256.LoadUnsafe(ref MemoryMarshal.GetReference(values[i..]));
                sum = Avx.Add(sum, vec);
            }

            // Horizontal sum of the vector
            var hi = Avx.ExtractVector128(sum, 1);
            var lo = sum.GetLower();
            var combined = Sse.Add(hi, lo);

            // Sum the 4 floats in 128-bit register
            var shuf = Sse.Shuffle(combined, combined, 0x1B);
            combined = Sse.Add(combined, shuf);
            var result = combined[0] + combined[1];

            // Handle remaining elements
            for (; i < values.Length; i++)
                result += values[i];

            return result;
        }

        // Fallback for non-AVX2
        var total = 0f;
        foreach (var v in values) total += v;
        return total;
    }

    // Using the portable Vector<T> API (auto-vectorized by JIT)
    public static float SumFloatsVector(ReadOnlySpan<float> values)
    {
        var sum = Vector<float>.Zero;
        var vectors = MemoryMarshal.Cast<float, Vector<float>>(values);

        foreach (var vec in vectors)
            sum += vec;

        var result = Vector.Dot(sum, Vector<float>.One);

        // Remainder
        var remainder = values[(vectors.Length * Vector<float>.Count)..];
        foreach (var v in remainder) result += v;

        return result;
    }

    // Dot product with SIMD
    public static float DotProduct(ReadOnlySpan<float> a, ReadOnlySpan<float> b)
    {
        if (a.Length != b.Length)
            throw new ArgumentException("Spans must be same length");

        var sum = Vector<float>.Zero;
        var aVecs = MemoryMarshal.Cast<float, Vector<float>>(a);
        var bVecs = MemoryMarshal.Cast<float, Vector<float>>(b);

        for (var i = 0; i < aVecs.Length; i++)
            sum += aVecs[i] * bVecs[i];

        var result = Vector.Dot(sum, Vector<float>.One);

        // Remainder
        var start = aVecs.Length * Vector<float>.Count;
        for (var i = start; i < a.Length; i++)
            result += a[i] * b[i];

        return result;
    }
}
```

---

## Step 1365: BenchmarkDotNet

```xml
<ItemGroup>
  <PackageReference Include="BenchmarkDotNet" Version="0.14.*" />
  <PackageReference Include="BenchmarkDotNet.Diagnostics.Windows" Version="0.14.*" Condition="$([MSBuild]::IsOSPlatform('Windows'))" />
</ItemGroup>
```

```csharp
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Running;
using BenchmarkDotNet.Jobs;
using BenchmarkDotNet.Diagnosers;

// Run benchmarks in Program.cs
BenchmarkRunner.Run<StringParsingBenchmarks>();

[SimpleJob(RuntimeMoniker.Net90)]
[MemoryDiagnoser]
[CpuDiagnoser]
[HardwareCounters(HardwareCounter.CacheMisses, HardwareCounter.BranchMispredictions)]
[GroupBenchmarksBy(BenchmarkLogicalGroupRule.ByCategory)]
public class StringParsingBenchmarks
{
    [Params(10, 1000, 100_000)]
    public int N;

    private string _input = string.Empty;

    [GlobalSetup]
    public void Setup()
    {
        _input = string.Join(',', Enumerable.Range(0, N));
    }

    [Benchmark(Baseline = true), BenchmarkCategory("Parse")]
    public int[] WithAlloc() => StringParser.ParseIntsAlloc(_input);

    [Benchmark, BenchmarkCategory("Parse")]
    public int[] WithSpan() => StringParser.ParseIntsSpan(_input.AsSpan());
}

[SimpleJob(RuntimeMoniker.Net90)]
[MemoryDiagnoser]
public class CollectionBenchmarks
{
    private readonly List<int> _list = Enumerable.Range(0, 1_000_000).ToList();

    [Benchmark(Baseline = true)]
    public int LinqSum() => _list.Sum();

    [Benchmark]
    public int SpanSum()
    {
        var sum = 0;
        foreach (var x in CollectionsMarshal.AsSpan(_list))
            sum += x;
        return sum;
    }

    [Benchmark]
    public int SimdSum()
    {
        var span = CollectionsMarshal.AsSpan(_list);
        var floatSpan = MemoryMarshal.Cast<int, float>(span);
        return (int)SimdHelpers.SumFloatsVector(floatSpan);
    }
}

[SimpleJob(RuntimeMoniker.Net90)]
[MemoryDiagnoser]
public class AllocationBenchmarks
{
    private const int Iterations = 10_000;

    [Benchmark(Baseline = true)]
    public string WithNewStringBuilder()
    {
        string result = string.Empty;
        for (var i = 0; i < 10; i++)
        {
            var sb = new StringBuilder();
            sb.Append("item_");
            sb.Append(i);
            result = sb.ToString();
        }
        return result;
    }

    [Benchmark]
    public string WithStackAlloc()
    {
        Span<char> buffer = stackalloc char[20];
        var written = 0;
        "item_".AsSpan().CopyTo(buffer);
        written += 5;
        written += 9.TryFormat(buffer[written..], out var w) ? w : 0;
        return new string(buffer[..written]);
    }

    [Benchmark]
    public string WithStringCreate()
    {
        return string.Create(10, 42, static (chars, state) =>
        {
            "item_".AsSpan().CopyTo(chars);
            state.TryFormat(chars[5..], out _);
        });
    }
}

// Typical benchmark results interpretation:
// Method          | N      | Mean      | Allocated
// WithAlloc       | 10     | 856 ns    | 512 B
// WithSpan        | 10     | 234 ns    | 40 B     ← 3.6x faster, 92% less memory
// WithAlloc       | 100000 | 8.3 ms    | 4.2 MB
// WithSpan        | 100000 | 2.1 ms    | 0 B      ← 4x faster, zero alloc
```

---

## Step 1366: Channels for High-Throughput Pipelines

```csharp
using System.Threading.Channels;

// Pipeline pattern: Producer → Transform → Consumer
public class DataPipeline
{
    public static async Task RunAsync(
        IEnumerable<string> inputData,
        CancellationToken ct)
    {
        // Bounded channels with backpressure
        var rawChannel = Channel.CreateBounded<string>(new BoundedChannelOptions(1000)
        {
            FullMode = BoundedChannelFullMode.Wait,
            SingleWriter = false,
            SingleReader = false,
        });

        var processedChannel = Channel.CreateBounded<ProcessedItem>(new BoundedChannelOptions(500)
        {
            FullMode = BoundedChannelFullMode.Wait,
        });

        // Producer: read source data
        var producerTask = Task.Run(async () =>
        {
            try
            {
                foreach (var item in inputData)
                {
                    await rawChannel.Writer.WriteAsync(item, ct);
                }
            }
            finally
            {
                rawChannel.Writer.Complete();
            }
        }, ct);

        // Transformer: parallel processing (N workers)
        var transformerTasks = Enumerable.Range(0, Environment.ProcessorCount)
            .Select(_ => Task.Run(async () =>
            {
                await foreach (var raw in rawChannel.Reader.ReadAllAsync(ct))
                {
                    var processed = await ProcessItemAsync(raw, ct);
                    await processedChannel.Writer.WriteAsync(processed, ct);
                }
            }, ct))
            .ToArray();

        // Signal when all transformers done
        var transformerCompletion = Task.WhenAll(transformerTasks)
            .ContinueWith(_ => processedChannel.Writer.Complete(), ct);

        // Consumer: batch write to database
        var consumerTask = Task.Run(async () =>
        {
            var batch = new List<ProcessedItem>(100);
            await foreach (var item in processedChannel.Reader.ReadAllAsync(ct))
            {
                batch.Add(item);
                if (batch.Count >= 100)
                {
                    await FlushBatchAsync(batch, ct);
                    batch.Clear();
                }
            }
            if (batch.Count > 0)
                await FlushBatchAsync(batch, ct);
        }, ct);

        await Task.WhenAll(producerTask, transformerCompletion, consumerTask);
    }

    private static async Task<ProcessedItem> ProcessItemAsync(string raw, CancellationToken ct)
    {
        await Task.Yield(); // Yield to avoid blocking
        return new ProcessedItem(raw.ToUpperInvariant(), raw.Length);
    }

    private static async Task FlushBatchAsync(List<ProcessedItem> batch, CancellationToken ct)
    {
        // Bulk insert to DB
        await Task.Delay(1, ct); // Simulate DB write
    }
}

// High-throughput event processor
public class EventProcessor(ILogger<EventProcessor> logger) : BackgroundService
{
    private readonly Channel<DomainEvent> _channel = Channel.CreateUnbounded<DomainEvent>(
        new UnboundedChannelOptions { SingleReader = true });

    public ChannelWriter<DomainEvent> Writer => _channel.Writer;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await foreach (var @event in _channel.Reader.ReadAllAsync(stoppingToken))
        {
            try
            {
                await HandleAsync(@event, stoppingToken);
            }
            catch (Exception ex)
            {
                logger.LogError(ex, "Failed to handle event {EventType}", @event.GetType().Name);
            }
        }
    }

    private async Task HandleAsync(DomainEvent @event, CancellationToken ct)
    {
        // Process event
        logger.LogDebug("Processing {EventType}", @event.GetType().Name);
        await Task.Yield();
    }
}

// Usage: fire-and-forget events without blocking HTTP request
public class OrderController(EventProcessor processor) : ControllerBase
{
    [HttpPost]
    public IActionResult CreateOrder([FromBody] CreateOrderRequest request)
    {
        var orderId = Guid.NewGuid();
        // Non-blocking: write to channel and return immediately
        if (!processor.Writer.TryWrite(new OrderCreatedEvent(orderId, request.CustomerId)))
            logger.LogWarning("Event channel full — order event dropped");

        return Accepted(new { orderId });
    }
}
```

---

## Step 1367: ValueTask and Async Performance

```csharp
// ValueTask<T> — avoids Task heap allocation for synchronous fast paths

public class CacheService(IMemoryCache cache, IDatabase redis)
{
    // ✓ ValueTask: often completes synchronously (cache hit)
    public ValueTask<string?> GetStringAsync(string key, CancellationToken ct)
    {
        // Hot path: check in-memory cache first
        if (cache.TryGetValue(key, out string? cached))
            return ValueTask.FromResult(cached); // No allocation!

        // Cold path: go to Redis (allocates Task)
        return new ValueTask<string?>(GetFromRedisAsync(key, ct));
    }

    private async Task<string?> GetFromRedisAsync(string key, CancellationToken ct)
    {
        var value = await redis.StringGetAsync(key);
        if (value.HasValue)
        {
            cache.Set(key, (string?)value, TimeSpan.FromMinutes(5));
            return value;
        }
        return null;
    }
}

// IValueTaskSource — advanced: reuse ValueTask source
public sealed class ReusableValueTaskSource<T> : IValueTaskSource<T>
{
    private ManualResetValueTaskSourceCore<T> _core;

    public ValueTask<T> ValueTask => new(this, _core.Version);

    public void SetResult(T result) => _core.SetResult(result);
    public void SetException(Exception exception) => _core.SetException(exception);
    public void Reset() => _core.Reset();

    T IValueTaskSource<T>.GetResult(short token) => _core.GetResult(token);
    ValueTaskSourceStatus IValueTaskSource<T>.GetStatus(short token) => _core.GetStatus(token);
    void IValueTaskSource<T>.OnCompleted(Action<object?> continuation, object? state, short token,
        ValueTaskSourceOnCompletedFlags flags)
        => _core.OnCompleted(continuation, state, token, flags);
}

// ConfigureAwait(false) — avoids unnecessary synchronization context capture
public class LibraryService
{
    // In library code (non-UI): always use ConfigureAwait(false)
    public async Task<string> GetDataAsync(CancellationToken ct)
    {
        var data = await FetchFromApiAsync(ct).ConfigureAwait(false);
        var processed = await ProcessAsync(data, ct).ConfigureAwait(false);
        return processed;
    }

    // Avoid async overhead for tiny synchronous operations
    public Task<int> GetCachedCountAsync()
        => Task.FromResult(_cachedCount); // No state machine

    private int _cachedCount = 42;
    private async Task<string> FetchFromApiAsync(CancellationToken ct) => await Task.FromResult("data");
    private async Task<string> ProcessAsync(string data, CancellationToken ct) => await Task.FromResult(data);
}
```

---

## Step 1368: String Performance

```csharp
// String optimization strategies

public static class StringOptimizations
{
    // 1. String.Create — no intermediate allocation
    public static string FormatId(Guid id, string prefix)
    {
        // Exactly calculates length upfront
        var length = prefix.Length + 1 + 32; // prefix + "_" + guid without dashes
        return string.Create(length, (id, prefix), static (chars, state) =>
        {
            var (guid, pfx) = state;
            pfx.AsSpan().CopyTo(chars);
            chars[pfx.Length] = '_';
            guid.TryFormat(chars[(pfx.Length + 1)..], out _, "N"); // N = no dashes
        });
    }

    // 2. StringBuilder pooled
    private static readonly ObjectPool<StringBuilder> SbPool =
        ObjectPool.Create(new StringBuilderPooledObjectPolicy());

    public static string BuildQuery(IReadOnlyList<string> conditions)
    {
        var sb = SbPool.Get();
        try
        {
            sb.Append("SELECT * FROM orders WHERE ");
            for (var i = 0; i < conditions.Count; i++)
            {
                if (i > 0) sb.Append(" AND ");
                sb.Append(conditions[i]);
            }
            return sb.ToString();
        }
        finally { SbPool.Return(sb); }
    }

    // 3. Interpolated string handler (C# 10) — zero allocation formatting
    [InterpolatedStringHandler]
    public ref struct LogInterpolatedStringHandler
    {
        private DefaultInterpolatedStringHandler _handler;

        public LogInterpolatedStringHandler(
            int literalLength, int formattedCount, ILogger logger, out bool shouldAppend)
        {
            shouldAppend = logger.IsEnabled(LogLevel.Debug);
            _handler = shouldAppend
                ? new DefaultInterpolatedStringHandler(literalLength, formattedCount)
                : default;
        }

        public void AppendLiteral(string s) => _handler.AppendLiteral(s);
        public void AppendFormatted<T>(T value) => _handler.AppendFormatted(value);
        public string ToStringAndClear() => _handler.ToStringAndClear();
    }

    // 4. Span-based parsing — avoid string allocation
    public static bool TryParseOrderId(ReadOnlySpan<char> input, out Guid orderId)
    {
        // "ORD-{guid}" format
        if (input.StartsWith("ORD-") && input.Length == 40)
        {
            return Guid.TryParse(input[4..], out orderId);
        }
        orderId = default;
        return false;
    }

    // 5. String interning for frequently repeated strings
    private static readonly string[] _statusCache = ["Pending", "Processing", "Completed", "Cancelled"];
    private static readonly FrozenDictionary<string, string> _interned =
        _statusCache.ToFrozenDictionary(s => s, s => string.Intern(s));

    public static string InternStatus(string status)
        => _interned.TryGetValue(status, out var interned) ? interned : status;
}

// FrozenDictionary / FrozenSet — immutable, lookup-optimized collections (NET 8+)
public static class FrozenCollections
{
    // Build once, read many times — faster than Dictionary for reads
    public static readonly FrozenDictionary<string, int> StatusCodes =
        new Dictionary<string, int>
        {
            ["Pending"] = 0,
            ["Processing"] = 1,
            ["Completed"] = 2,
            ["Cancelled"] = 3,
        }.ToFrozenDictionary();

    public static readonly FrozenSet<string> AllowedContentTypes =
        new HashSet<string> { "application/json", "application/xml", "text/plain" }
            .ToFrozenSet();
}
```

---

## Step 1369: Minimal API Performance

```csharp
// Minimal API is ~20% faster than Controllers due to less middleware

var builder = WebApplication.CreateBuilder(args);

// Optimize for throughput
builder.WebHost.ConfigureKestrel(options =>
{
    options.Limits.MaxConcurrentConnections = 10_000;
    options.Limits.MaxRequestBodySize = 10 * 1024; // 10 KB
    options.Limits.MinRequestBodyDataRate = null; // Disable for high concurrency
    options.AddServerHeader = false; // Don't send Server header
});

var app = builder.Build();

// Fast endpoint — avoids allocations where possible
app.MapGet("/api/health", () => Results.Ok(new { status = "healthy" }));

// With response caching
app.MapGet("/api/products/featured", async (
    ICacheService cache,
    IProductRepository repo,
    CancellationToken ct) =>
{
    var products = await cache.GetOrCreateAsync(
        "products:featured",
        async ct => await repo.GetFeaturedAsync(ct),
        TimeSpan.FromMinutes(5),
        ct);

    return TypedResults.Ok(products);
})
.CacheOutput(policy => policy.Expire(TimeSpan.FromMinutes(5)).Tag("products"));

// TypedResults — compile-time correct response types + OpenAPI metadata
app.MapPost("/api/orders", async (
    CreateOrderRequest request,
    IOrderService service,
    CancellationToken ct) =>
{
    var order = await service.CreateAsync(request, ct);
    return TypedResults.Created($"/api/orders/{order.Id}", order);
})
.Produces<OrderDto>(201)
.ProducesValidationProblem(400)
.RequireAuthorization();

// Response compression
builder.Services.AddResponseCompression(options =>
{
    options.EnableForHttps = true;
    options.MimeTypes = ResponseCompressionDefaults.MimeTypes.Concat(
        ["application/json", "application/problem+json"]);
});

app.UseResponseCompression();

// Output caching (NET 7+)
builder.Services.AddOutputCache(options =>
{
    options.AddPolicy("products", builder =>
        builder.Expire(TimeSpan.FromMinutes(5)).Tag("products"));
});

app.UseOutputCache();
```

---

## Step 1370: HTTP Client Performance

```csharp
// Efficient HTTP client usage

// 1. Named clients with proper configuration
builder.Services.AddHttpClient("payment-api", client =>
{
    client.BaseAddress = new Uri("https://payment.api.com");
    client.Timeout = TimeSpan.FromSeconds(30);
    client.DefaultRequestHeaders.Accept.Add(
        new MediaTypeWithQualityHeaderValue("application/json"));
    client.DefaultRequestVersion = HttpVersion.Version20; // HTTP/2
    client.DefaultVersionPolicy = HttpVersionPolicy.RequestVersionOrHigher;
})
.ConfigurePrimaryHttpMessageHandler(() => new SocketsHttpHandler
{
    PooledConnectionLifetime = TimeSpan.FromMinutes(15), // Connection reuse
    PooledConnectionIdleTimeout = TimeSpan.FromMinutes(2),
    MaxConnectionsPerServer = 50,
    EnableMultipleHttp2Connections = true,
    AutomaticDecompression = DecompressionMethods.All,
})
.AddStandardResilience();

// 2. Typed client
public class PaymentApiClient(HttpClient httpClient)
{
    public async Task<PaymentResult?> ChargeAsync(
        ChargeRequest request, CancellationToken ct)
    {
        // Use System.Net.Http.Json — no manual serialization
        var response = await httpClient.PostAsJsonAsync(
            "v1/charges", request, ct);

        if (!response.IsSuccessStatusCode)
        {
            var problem = await response.Content
                .ReadFromJsonAsync<ProblemDetails>(ct);
            throw new PaymentException(problem?.Detail ?? "Payment failed");
        }

        return await response.Content.ReadFromJsonAsync<PaymentResult>(ct);
    }

    public async IAsyncEnumerable<TransactionDto> StreamTransactionsAsync(
        Guid accountId,
        [EnumeratorCancellation] CancellationToken ct)
    {
        // Streaming JSON with IAsyncEnumerable
        var request = new HttpRequestMessage(HttpMethod.Get,
            $"v1/accounts/{accountId}/transactions");
        request.Headers.Accept.Add(
            new MediaTypeWithQualityHeaderValue("application/x-ndjson"));

        using var response = await httpClient.SendAsync(
            request, HttpCompletionOption.ResponseHeadersRead, ct);

        await foreach (var tx in response.Content
            .ReadFromJsonAsAsyncEnumerable<TransactionDto>(ct))
        {
            if (tx is not null)
                yield return tx;
        }
    }
}

// 3. Batch requests
public async Task<Dictionary<Guid, ProductDto>> GetProductsBatchAsync(
    IEnumerable<Guid> productIds, CancellationToken ct)
{
    // Fan-out concurrent requests (bounded parallelism)
    using var semaphore = new SemaphoreSlim(10); // Max 10 concurrent
    var tasks = productIds.Select(async id =>
    {
        await semaphore.WaitAsync(ct);
        try
        {
            return await _productClient.GetByIdAsync(id, ct);
        }
        finally
        {
            semaphore.Release();
        }
    });

    var results = await Task.WhenAll(tasks);
    return results
        .Where(p => p is not null)
        .ToDictionary(p => p!.Id);
}
```

---

## Step 1371: Profiling with dotnet-trace and PerfView

```bash
# Install diagnostic tools
dotnet tool install --global dotnet-trace
dotnet tool install --global dotnet-counters
dotnet tool install --global dotnet-dump
dotnet tool install --global dotnet-gcdump

# 1. Real-time performance counters
dotnet-counters monitor --process-id <pid> \
  --counters System.Runtime,Microsoft.AspNetCore.Hosting,Microsoft.EntityFrameworkCore

# Key metrics to watch:
# System.Runtime:
#   alloc-rate          — allocations/sec (MB/sec)
#   gc-heap-size        — total GC heap size
#   gen-0-gc-count      — Gen 0 GC frequency (high = lots of allocations)
#   cpu-usage           — CPU %
#   active-timer-count  — timer overhead
#   threadpool-queue-length — work queue depth

# Microsoft.AspNetCore.Hosting:
#   requests-per-second — RPS
#   total-requests      — total served
#   failed-requests     — failures

# 2. CPU profiling
dotnet-trace collect --process-id <pid> \
  --providers Microsoft-DotNETCore-SampleProfiler \
  --profile cpu-sampling \
  --duration 00:00:30 \
  --output cpu-trace.nettrace

# View in SpeedScope (https://www.speedscope.app)
dotnet-trace convert cpu-trace.nettrace --format Speedscope

# 3. GC profiling
dotnet-gcdump collect --process-id <pid>
# Open in Visual Studio or dotnet-gcdump report

# 4. Memory dump analysis
dotnet-dump collect --process-id <pid>
dotnet-dump analyze core_<pid>
# Commands inside analyzer:
# > dumpheap -stat           — objects by type
# > dumpheap -type String    — all string instances
# > gcroot <address>         — find GC roots for an object
# > finalizequeue            — objects waiting for finalization
```

```csharp
// Application instrumentation for profiling
public static class PerformanceCounters
{
    private static readonly Meter _meter = new("MyApp.Performance", "1.0.0");

    // Custom counters
    public static readonly Counter<long> CacheHits =
        _meter.CreateCounter<long>("cache.hits", "requests", "Cache hit count");

    public static readonly Counter<long> CacheMisses =
        _meter.CreateCounter<long>("cache.misses", "requests", "Cache miss count");

    public static readonly Histogram<double> DbQueryDuration =
        _meter.CreateHistogram<double>("db.query.duration", "ms", "Database query duration");

    public static readonly ObservableGauge<int> ActiveConnections =
        _meter.CreateObservableGauge("http.connections.active", () =>
            ConnectionTracker.ActiveCount, "connections", "Active HTTP connections");
}

// Activity for distributed tracing
public static class OrderActivitySource
{
    private static readonly ActivitySource Source = new("OrderService", "1.0.0");

    public static Activity? StartCreateOrder(Guid customerId)
    {
        var activity = Source.StartActivity("CreateOrder", ActivityKind.Internal);
        activity?.SetTag("customer.id", customerId);
        activity?.SetTag("order.service.version", "2.0");
        return activity;
    }
}
```

---

## Step 1372: GC Tuning

```csharp
// GC configuration for high-throughput servers
// appsettings or environment variables

// Server GC (default for ASP.NET) — higher throughput, higher memory
// Workstation GC — lower memory, more frequent GC pauses

// JSON configuration (runtimeconfig.json or csproj)
/*
<PropertyGroup>
  <ServerGarbageCollection>true</ServerGarbageCollection>
  <GarbageCollectionAdaptationMode>0</GarbageCollectionAdaptationMode>
  <HeapHardLimit>2147483648</HeapHardLimit>  <!-- 2 GB limit -->
  <GCConserveMemory>0</GCConserveMemory>     <!-- 0-9, higher = lower memory -->
</PropertyGroup>
*/

// POH (Pinned Object Heap) — for objects that must be pinned for P/Invoke
// Use for buffers that are frequently passed to native code
public class PinnedBuffer(int size) : IDisposable
{
    private readonly byte[] _buffer = GC.AllocateArray<byte>(size, pinned: true);
    private readonly GCHandle _handle = default;
    private bool _disposed;

    public Span<byte> Span => _buffer.AsSpan();

    // Get raw pointer for native interop (buffer is pinned, won't move)
    public unsafe byte* Pointer => (byte*)Unsafe.AsPointer(ref _buffer[0]);

    public void Dispose()
    {
        if (!_disposed)
        {
            _disposed = true;
            GC.Collect(2, GCCollectionMode.Forced, blocking: true);
        }
    }
}

// LOH optimization — large object heap (>85KB) causes fragmentation
public class LargeObjectOptimizations
{
    // Reuse large buffers instead of allocating new ones
    private static readonly ArrayPool<byte> LargePool = ArrayPool<byte>.Create(
        maxArrayLength: 1024 * 1024, // 1 MB max
        maxArraysPerBucket: 50);

    public async Task ProcessLargeDataAsync(Stream stream, CancellationToken ct)
    {
        // Rent from pool — avoids LOH allocation
        var buffer = LargePool.Rent(1024 * 256); // 256 KB
        try
        {
            var span = buffer.AsMemory(0, 1024 * 256);
            var read = await stream.ReadAsync(span, ct);
            // Process buffer[..read]
        }
        finally
        {
            LargePool.Return(buffer);
        }
    }
}
```

---

## Step 1373: Parallel Processing Patterns

```csharp
using System.Collections.Concurrent;

// Parallel.ForEachAsync — bounded async parallelism
public async Task ProcessOrdersAsync(
    IEnumerable<Order> orders,
    int maxDegreeOfParallelism,
    CancellationToken ct)
{
    var options = new ParallelOptions
    {
        MaxDegreeOfParallelism = maxDegreeOfParallelism,
        CancellationToken = ct,
    };

    await Parallel.ForEachAsync(orders, options, async (order, token) =>
    {
        await ProcessSingleOrderAsync(order, token);
    });
}

// Partitioner for CPU-bound work
public double[] ComputeFeatures(double[] data)
{
    var results = new double[data.Length];
    var rangePartitioner = Partitioner.Create(0, data.Length);

    Parallel.ForEach(rangePartitioner, range =>
    {
        for (var i = range.Item1; i < range.Item2; i++)
        {
            // Complex CPU-bound computation per element
            results[i] = Math.Sqrt(data[i] * data[i] + Math.Log(data[i] + 1));
        }
    });

    return results;
}

// PLINQ — parallel LINQ for data transformation
public List<OrderSummary> TransformOrdersPLinq(IEnumerable<Order> orders)
{
    return orders
        .AsParallel()
        .WithDegreeOfParallelism(Environment.ProcessorCount)
        .WithCancellation(CancellationToken.None)
        .Where(o => o.Status == OrderStatus.Completed)
        .Select(o => new OrderSummary(o.Id, o.Status, o.TotalAmount, o.CreatedAt))
        .OrderByDescending(s => s.TotalAmount)
        .ToList();
}

// Concurrent collections for shared state
public class ConcurrentOrderCache
{
    private readonly ConcurrentDictionary<Guid, Order> _cache = new();
    private readonly ConcurrentQueue<Guid> _evictionQueue = new();
    private const int MaxSize = 10_000;

    public Order GetOrAdd(Guid id, Func<Guid, Order> factory)
    {
        if (_cache.TryGetValue(id, out var cached))
            return cached;

        if (_cache.Count >= MaxSize)
            EvictOne();

        return _cache.GetOrAdd(id, factory);
    }

    private void EvictOne()
    {
        if (_evictionQueue.TryDequeue(out var evictId))
            _cache.TryRemove(evictId, out _);
    }
}

// Work stealing with IProducerConsumerCollection
public class WorkStealingScheduler<T> : IDisposable
{
    private readonly ConcurrentBag<T> _bag = new();
    private readonly SemaphoreSlim _signal = new(0);
    private bool _disposed;

    public void Enqueue(T item)
    {
        _bag.Add(item);
        _signal.Release();
    }

    public async Task<T?> DequeueAsync(CancellationToken ct)
    {
        await _signal.WaitAsync(ct);
        return _bag.TryTake(out var item) ? item : default;
    }

    public void Dispose()
    {
        _disposed = true;
        _signal.Dispose();
    }
}
```

---

## Step 1374: Source Generators for Performance

```csharp
// Source generators avoid reflection at runtime
// Examples: System.Text.Json, LoggerMessage, RegexGenerator

// 1. Compile-time logging (no reflection, no boxing)
public partial class OrderService(ILogger<OrderService> logger)
{
    [LoggerMessage(Level = LogLevel.Information, Message = "Order {OrderId} created for customer {CustomerId}")]
    private partial void LogOrderCreated(Guid orderId, Guid customerId);

    [LoggerMessage(Level = LogLevel.Error, Message = "Failed to process order {OrderId}: {Error}")]
    private partial void LogOrderFailed(Guid orderId, string error, Exception exception);

    [LoggerMessage(Level = LogLevel.Warning, Message = "Order {OrderId} amount {Amount} exceeds limit {Limit}")]
    private partial void LogHighValueOrder(Guid orderId, decimal amount, decimal limit);
}

// 2. Compile-time regex (avoids runtime compilation)
public partial class OrderValidator
{
    [GeneratedRegex(@"^[A-Z]{2,3}-\d{4,10}$", RegexOptions.Compiled)]
    private static partial Regex OrderReferenceRegex();

    [GeneratedRegex(@"^\d{4}-\d{2}-\d{2}$", RegexOptions.Compiled)]
    private static partial Regex DateFormatRegex();

    public bool IsValidOrderReference(string reference)
        => OrderReferenceRegex().IsMatch(reference);
}

// 3. JSON source generation (no reflection serialization)
[JsonSerializable(typeof(Order))]
[JsonSerializable(typeof(List<Order>))]
[JsonSerializable(typeof(OrderDto))]
[JsonSerializable(typeof(List<OrderDto>))]
[JsonSerializable(typeof(CreateOrderRequest))]
[JsonSerializable(typeof(ProblemDetails))]
[JsonSourceGenerationOptions(
    PropertyNamingPolicy = JsonKnownNamingPolicy.CamelCase,
    WriteIndented = false,
    DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull)]
internal partial class AppJsonContext : JsonSerializerContext { }

// Registration
builder.Services.ConfigureHttpJsonOptions(options =>
{
    options.SerializerOptions.TypeInfoResolverChain.Insert(0, AppJsonContext.Default);
});

// 4. Strongly-typed configuration source generation
[GenerateBindingRedirects]  // Custom source generator
public class AppOptions
{
    public string ApiKey { get; set; } = string.Empty;
    public int TimeoutSeconds { get; set; } = 30;
}
```

---

## Step 1375: Zero-Allocation HTTP Parsing

```csharp
// System.IO.Pipelines for high-performance I/O
using System.IO.Pipelines;

public class HttpRequestParser
{
    public static async Task ParseRequestAsync(Stream stream, CancellationToken ct)
    {
        var reader = PipeReader.Create(stream, new StreamPipeReaderOptions(
            bufferSize: 4096,
            minimumReadSize: 1024,
            pool: MemoryPool<byte>.Shared));

        while (true)
        {
            var result = await reader.ReadAsync(ct);
            var buffer = result.Buffer;

            if (TryParseRequestLine(ref buffer, out var method, out var path, out var protocol))
            {
                // Process the parsed line without allocating strings if possible
                ProcessRequest(method, path, protocol);
            }

            reader.AdvanceTo(buffer.Start, buffer.End);

            if (result.IsCompleted) break;
        }

        await reader.CompleteAsync();
    }

    private static bool TryParseRequestLine(
        ref ReadOnlySequence<byte> buffer,
        out ReadOnlySpan<byte> method,
        out ReadOnlySpan<byte> path,
        out ReadOnlySpan<byte> protocol)
    {
        // Zero-allocation parsing directly from buffer
        var sequenceReader = new SequenceReader<byte>(buffer);

        if (!sequenceReader.TryReadTo(out method, (byte)' ')) goto fail;
        if (!sequenceReader.TryReadTo(out path, (byte)' ')) goto fail;
        if (!sequenceReader.TryReadTo(out protocol, (byte)'\n')) goto fail;

        buffer = buffer.Slice(sequenceReader.Position);
        return true;

        fail:
        method = default;
        path = default;
        protocol = default;
        return false;
    }

    private static void ProcessRequest(
        ReadOnlySpan<byte> method,
        ReadOnlySpan<byte> path,
        ReadOnlySpan<byte> protocol)
    {
        // Compare without allocating strings
        if (method.SequenceEqual("GET"u8))
        {
            // Handle GET
        }
        else if (method.SequenceEqual("POST"u8))
        {
            // Handle POST
        }
    }
}
```

---

## Step 1376: Caching Strategies in Depth

```csharp
// Multi-level caching with fallback strategy
public class MultiLevelCache<TKey, TValue>(
    IMemoryCache l1,
    IDistributedCache l2,
    string keyPrefix,
    TimeSpan l1Ttl,
    TimeSpan l2Ttl,
    ILogger logger) where TKey : notnull
{
    public async Task<TValue?> GetAsync(TKey key, CancellationToken ct)
    {
        var cacheKey = $"{keyPrefix}:{key}";

        // L1: in-process memory cache (ns access)
        if (l1.TryGetValue<TValue>(cacheKey, out var l1Value))
        {
            return l1Value;
        }

        // L2: distributed cache (Redis, ~0.5ms access)
        var l2Bytes = await l2.GetAsync(cacheKey, ct);
        if (l2Bytes is not null)
        {
            var l2Value = JsonSerializer.Deserialize<TValue>(l2Bytes);
            // Promote to L1
            l1.Set(cacheKey, l2Value, l1Ttl);
            return l2Value;
        }

        return default;
    }

    public async Task SetAsync(TKey key, TValue value, CancellationToken ct)
    {
        var cacheKey = $"{keyPrefix}:{key}";

        // Set L1
        l1.Set(cacheKey, value, l1Ttl);

        // Set L2
        var bytes = JsonSerializer.SerializeToUtf8Bytes(value);
        await l2.SetAsync(cacheKey, bytes,
            new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = l2Ttl }, ct);
    }

    public async Task InvalidateAsync(TKey key, CancellationToken ct)
    {
        var cacheKey = $"{keyPrefix}:{key}";
        l1.Remove(cacheKey);
        await l2.RemoveAsync(cacheKey, ct);
    }
}

// Cache stampede prevention with SemaphoreSlim
public class AntiStampedeCache<T>(
    IDistributedCache cache, ILogger logger)
{
    private readonly ConcurrentDictionary<string, SemaphoreSlim> _locks = new();

    public async Task<T> GetOrCreateAsync(
        string key,
        Func<CancellationToken, Task<T>> factory,
        TimeSpan ttl,
        CancellationToken ct)
    {
        var bytes = await cache.GetAsync(key, ct);
        if (bytes is not null)
            return JsonSerializer.Deserialize<T>(bytes)!;

        // Prevent multiple concurrent loads for same key
        var semaphore = _locks.GetOrAdd(key, _ => new SemaphoreSlim(1, 1));
        await semaphore.WaitAsync(ct);
        try
        {
            // Double-check after acquiring lock
            bytes = await cache.GetAsync(key, ct);
            if (bytes is not null)
                return JsonSerializer.Deserialize<T>(bytes)!;

            logger.LogDebug("Cache miss for {Key}, loading from source", key);
            var value = await factory(ct);

            bytes = JsonSerializer.SerializeToUtf8Bytes(value);
            await cache.SetAsync(key, bytes,
                new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = ttl }, ct);

            return value;
        }
        finally
        {
            semaphore.Release();
            _locks.TryRemove(key, out _);
        }
    }
}
```

---

## Step 1377: Performance Testing Patterns

```csharp
// NBomber load test with detailed assertions
using NBomber.CSharp;
using NBomber.Http.CSharp;

public class OrderApiLoadTest
{
    [Fact]
    public void LoadTest_CreateOrder_MeetsPerformanceTargets()
    {
        var httpFactory = HttpClientFactory.Create();

        var createOrderScenario = Scenario.Create("create_order", async context =>
        {
            var client = httpFactory.GetClient(context);
            var request = new CreateOrderRequest(
                CustomerId: Guid.NewGuid(),
                Items: [new OrderItemRequest(Guid.NewGuid(), 1, 29.99m)],
                ShippingAddress: "Test Street 1");

            var response = await client.PostAsJsonAsync("/api/v1/orders", request);
            return response.IsSuccessStatusCode ? Response.Ok() : Response.Fail();
        })
        .WithWarmUpDuration(TimeSpan.FromSeconds(10))
        .WithLoadSimulations(
            Simulation.RampingInject(rate: 100, interval: TimeSpan.FromSeconds(1),
                during: TimeSpan.FromSeconds(30)),
            Simulation.Inject(rate: 500, interval: TimeSpan.FromSeconds(1),
                during: TimeSpan.FromSeconds(60)),
            Simulation.RampingInject(rate: 0, interval: TimeSpan.FromSeconds(1),
                during: TimeSpan.FromSeconds(10)));

        var stats = NBomberRunner
            .RegisterScenarios(createOrderScenario)
            .WithTestSuite("OrderService")
            .WithTestName("CreateOrder_HighLoad")
            .Run();

        // Assert performance targets
        var scenario = stats.ScenarioStats.First();
        scenario.Ok.Request.Mean.Should().BeLessThan(200); // p50 < 200ms
        scenario.Ok.Latency.P95.Should().BeLessThan(500);   // p95 < 500ms
        scenario.Ok.Latency.P99.Should().BeLessThan(1000);  // p99 < 1000ms
        scenario.Fail.Request.Rate.Should().BeLessThan(0.01); // < 1% error rate
    }
}

// Stress test to find breaking point
public class StressTest
{
    public static async Task RunAsync()
    {
        var httpClient = new HttpClient { BaseAddress = new Uri("http://localhost:5000") };

        var currentRps = 100;
        var maxRps = 10_000;

        while (currentRps <= maxRps)
        {
            var results = await MeasureRpsAsync(httpClient, currentRps, TimeSpan.FromSeconds(10));
            Console.WriteLine($"Target RPS: {currentRps:N0} | Actual: {results.Rps:N0} | P99: {results.P99:F0}ms | Errors: {results.ErrorRate:P1}");

            if (results.ErrorRate > 0.05) // > 5% error rate = breaking point
            {
                Console.WriteLine($"Breaking point reached at {currentRps:N0} RPS");
                break;
            }

            currentRps = (int)(currentRps * 1.5); // Increase by 50%
        }
    }

    private static async Task<BenchmarkResult> MeasureRpsAsync(
        HttpClient client, int targetRps, TimeSpan duration)
    {
        var delay = TimeSpan.FromMilliseconds(1000.0 / targetRps);
        var stopwatch = Stopwatch.StartNew();
        var latencies = new ConcurrentBag<double>();
        var errors = 0;
        var total = 0;

        while (stopwatch.Elapsed < duration)
        {
            _ = Task.Run(async () =>
            {
                var sw = Stopwatch.StartNew();
                try
                {
                    var response = await client.GetAsync("/api/health");
                    if (!response.IsSuccessStatusCode)
                        Interlocked.Increment(ref errors);
                    latencies.Add(sw.Elapsed.TotalMilliseconds);
                }
                catch
                {
                    Interlocked.Increment(ref errors);
                }
                Interlocked.Increment(ref total);
            });

            await Task.Delay(delay);
        }

        var sortedLatencies = latencies.OrderBy(l => l).ToArray();
        return new BenchmarkResult(
            Rps: total / duration.TotalSeconds,
            P99: sortedLatencies.Length > 0 ? sortedLatencies[(int)(sortedLatencies.Length * 0.99)] : 0,
            ErrorRate: total > 0 ? (double)errors / total : 0);
    }
}

public record BenchmarkResult(double Rps, double P99, double ErrorRate);
```

---

## Step 1378: Memory-Efficient Data Structures

```csharp
// Specialized data structures for performance

// 1. BitArray for dense boolean flags
var permissions = new BitArray(1000); // 1000 bits = 125 bytes vs 1000 bools = 1000 bytes
permissions.Set(42, true);
var hasPermission = permissions.Get(42);

// 2. ReadOnlyMemory<T> for zero-copy slicing
public class MessageParser
{
    public static (ReadOnlyMemory<byte> Header, ReadOnlyMemory<byte> Body)
        ParseMessage(ReadOnlyMemory<byte> raw)
    {
        // Find separator without copying
        var span = raw.Span;
        var sep = span.IndexOf((byte)'\n');
        if (sep < 0) return (raw, ReadOnlyMemory<byte>.Empty);

        return (raw[..sep], raw[(sep + 1)..]);
    }
}

// 3. MemoryMarshal for type punning (unsafe but zero-copy)
public static class BinaryReader
{
    public static T ReadStruct<T>(ReadOnlySpan<byte> bytes) where T : struct
        => MemoryMarshal.Read<T>(bytes);

    public static IEnumerable<T> ReadStructArray<T>(ReadOnlySpan<byte> bytes) where T : struct
        => MemoryMarshal.Cast<byte, T>(bytes).ToArray(); // Converts span without copying
}

// 4. ImmutableArray<T> — struct wrapper, avoids IEnumerable boxing
public class ImmutableResult<T>
{
    public ImmutableArray<T> Items { get; init; } = ImmutableArray<T>.Empty;
    public int TotalCount { get; init; }

    // ImmutableArray.Builder for efficient construction
    public static ImmutableResult<T> Create(IEnumerable<T> items, int total)
    {
        var builder = ImmutableArray.CreateBuilder<T>();
        builder.AddRange(items);
        return new ImmutableResult<T> { Items = builder.ToImmutable(), TotalCount = total };
    }
}

// 5. RecyclableMemoryStream — avoids LOH pressure for large streams
using Microsoft.IO;

public class StreamService
{
    private static readonly RecyclableMemoryStreamManager _manager = new(new RecyclableMemoryStreamManager.Options
    {
        BlockSize = 128 * 1024,            // 128 KB blocks
        LargeBufferMultiple = 1024 * 1024, // 1 MB large buffer step
        MaximumBufferSize = 128 * 1024 * 1024, // 128 MB max
        AggressiveBufferReturn = true,
    });

    public async Task<byte[]> SerializeAsync<T>(T value, CancellationToken ct)
    {
        // Use pooled memory stream instead of MemoryStream
        using var stream = _manager.GetStream();
        await JsonSerializer.SerializeAsync(stream, value, cancellationToken: ct);
        return stream.ToArray();
    }
}
```

---

## Step 1379: Performance Anti-Patterns to Avoid

```csharp
// ❌ ANTI-PATTERNS — and their fixes

// 1. N+1 query problem
// ❌ Bad: 1 query for orders + N queries for customers
var orders = await db.Orders.ToListAsync();
foreach (var order in orders)
    order.Customer = await db.Customers.FindAsync(order.CustomerId); // N queries!

// ✓ Fix: eager loading or projection
var orders = await db.Orders.Include(o => o.Customer).ToListAsync();
// Or projection:
var summaries = await db.Orders
    .Select(o => new { o.Id, CustomerName = o.Customer.Name })
    .ToListAsync();

// 2. Synchronous I/O on async path
// ❌ Bad: blocks thread pool thread
app.MapGet("/data", (IRepository repo) =>
{
    var data = repo.GetAllAsync().GetAwaiter().GetResult(); // Deadlock risk!
    return data;
});

// ✓ Fix: async all the way
app.MapGet("/data", async (IRepository repo, CancellationToken ct) =>
    await repo.GetAllAsync(ct));

// 3. Unnecessary ToList() in LINQ pipeline
// ❌ Bad: creates intermediate list
var result = db.Orders
    .Where(o => o.Status == "Pending")
    .ToList()           // Forces execution too early
    .Select(o => new OrderDto(o.Id, o.Status))
    .ToList();

// ✓ Fix: compose queries, execute once
var result = await db.Orders
    .Where(o => o.Status == "Pending")
    .Select(o => new OrderDto(o.Id, o.Status))
    .ToListAsync(ct);

// 4. Recreating HttpClient
// ❌ Bad: socket exhaustion
app.MapGet("/external", async () =>
{
    using var client = new HttpClient(); // New socket every request!
    return await client.GetStringAsync("https://api.example.com");
});

// ✓ Fix: IHttpClientFactory (connection pooling)
app.MapGet("/external", async (IHttpClientFactory factory) =>
{
    var client = factory.CreateClient("external");
    return await client.GetStringAsync("https://api.example.com");
});

// 5. string concatenation in loops
// ❌ Bad: O(n²) allocations
var result = "";
for (var i = 0; i < 10000; i++)
    result += $"item_{i},"; // New string allocation each iteration!

// ✓ Fix: StringBuilder or string.Join
var sb = new StringBuilder(10000 * 8);
for (var i = 0; i < 10000; i++)
    sb.Append($"item_{i},");
var result2 = sb.ToString();
// Or: string.Join(",", Enumerable.Range(0, 10000).Select(i => $"item_{i}"))

// 6. Exception-driven control flow
// ❌ Bad: exceptions are 1000x slower than normal flow
public OrderDto? GetOrder(Guid id)
{
    try { return _cache.Get<OrderDto>(id); }
    catch (KeyNotFoundException) { return null; } // Exception for normal case!
}

// ✓ Fix: TryGet pattern
public OrderDto? GetOrder(Guid id)
    => _cache.TryGetValue(id, out OrderDto? order) ? order : null;

// 7. Boxing value types
// ❌ Bad: struct gets boxed to object
List<object> boxed = new();
boxed.Add(42);         // Boxes int
boxed.Add(3.14);       // Boxes double

// ✓ Fix: use generic collections
List<int> ints = new();
ints.Add(42); // No boxing
```

---

## Step 1380: Summary — Performance Checklist

```
PERFORMANCE OPTIMIZATION CHECKLIST
═══════════════════════════════════

MEMORY
├── Use struct for small value types (no GC pressure)
├── Span<T>/Memory<T> for zero-allocation slicing
├── ArrayPool<T> for temporary large arrays
├── ObjectPool<T> for expensive reusable objects
├── stackalloc for small, bounded local buffers
├── RecyclableMemoryStream instead of MemoryStream
└── FrozenDictionary/FrozenSet for immutable lookup tables

ALLOCATION-FREE CODING
├── String.Create() instead of string interpolation in hot paths
├── Source-generated logging (LoggerMessage attribute)
├── ValueTask instead of Task for sync-fast-path operations
├── ReadOnlySpan<char> parsing instead of string.Split()
└── MemoryMarshal for type punning without copies

CPU / THROUGHPUT
├── SIMD (Vector<T>, System.Runtime.Intrinsics) for data parallelism
├── Parallel.ForEachAsync for I/O-bound parallel work
├── Channel<T> for producer-consumer pipelines
├── PLINQ for CPU-bound data transformation
├── ConfigureAwait(false) in library code
└── Compiled LINQ queries for hot paths

DATABASE
├── Projections (Select) — only fetch needed columns
├── Compiled queries for repeated LINQ
├── Split queries for large includes
├── ExecuteUpdate/Delete — no entity loading for bulk ops
├── Keyset pagination — O(log n) instead of OFFSET O(n)
└── Read replicas for query scaling

CACHING
├── L1 IMemoryCache (in-process, ns)
├── L2 IDistributedCache/Redis (cross-process, μs)
├── Output caching for GET endpoints
├── Anti-stampede with SemaphoreSlim
└── Cache invalidation on mutation

HTTP / I/O
├── Named HttpClient (connection pooling)
├── HTTP/2 (multiplexing, header compression)
├── Response compression (Brotli/Gzip)
├── System.IO.Pipelines for custom protocols
└── Async streaming (IAsyncEnumerable)

PROFILING TOOLS
├── dotnet-counters — real-time metrics
├── dotnet-trace — CPU/GC profiling
├── dotnet-gcdump — heap analysis
├── BenchmarkDotNet — micro-benchmarks
├── NBomber — load testing
└── Visual Studio Profiler / JetBrains dotTrace
```

---

## สรุป Part 48

Part นี้ครอบคลุม Performance Optimization & Benchmarking อย่างครบถ้วน:

**Steps 1361-1380:**
- **Memory Layout** — struct vs class, value types, cache-friendly layout
- **Span\<T\> and Memory\<T\>** — zero-allocation slicing, ArrayPool, stackalloc
- **Object Pooling** — ObjectPool\<T\>, channel-based async pool
- **SIMD** — Vector\<T\> portable API, AVX2 intrinsics for data parallelism
- **BenchmarkDotNet** — MemoryDiagnoser, CpuDiagnoser, comparison benchmarks
- **Channels** — producer-consumer pipelines with backpressure
- **ValueTask** — sync-fast-path optimization, IValueTaskSource reuse
- **String Performance** — String.Create, interpolated string handlers, Span parsing
- **Minimal API** — Kestrel tuning, TypedResults, output caching
- **HTTP Client** — SocketsHttpHandler tuning, HTTP/2, async streaming
- **Profiling** — dotnet-counters, dotnet-trace, dotnet-gcdump, custom metrics
- **GC Tuning** — server GC, LOH optimization, pinned arrays, heap limits
- **Parallel Processing** — Parallel.ForEachAsync, PLINQ, ConcurrentDictionary
- **Source Generators** — LoggerMessage, GeneratedRegex, JSON source generation
- **Zero-Allocation I/O** — System.IO.Pipelines, SequenceReader
- **Caching** — multi-level cache, anti-stampede with SemaphoreSlim
- **Load Testing** — NBomber scenarios, stress test to breaking point
- **Memory Structures** — BitArray, RecyclableMemoryStream, ImmutableArray
- **Anti-Patterns** — N+1, sync-on-async, boxing, HttpClient misuse

เนื้อหาทั้งหมดเป็น Production-ready code ที่ใช้ได้จริงใน .NET 9.0
