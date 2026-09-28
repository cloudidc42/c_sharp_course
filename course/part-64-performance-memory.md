# Part 64: Performance & Memory Optimization — Zero-Allocation, High-Throughput .NET (Steps 1653-1668)

## Steps 1653-1668: Mastering Span<T>, Memory<T>, Pools, BenchmarkDotNet, and Native AOT

Performance-critical .NET is a discipline: measure first, optimize precisely. This part covers the full stack — from zero-allocation string processing with `Span<T>`, through `ArrayPool<T>`/`MemoryPool<T>` for avoiding GC pressure, to `BenchmarkDotNet` for rigorous micro-benchmarking, and `NativeAOT`/PGO for compile-time and startup wins.

---

## Step 1653: Why Allocations Kill Performance

Every heap allocation is a GC pressure point. The GC must:
1. Allocate on the managed heap (bump a pointer — fast)
2. Discover when objects are unreachable (Gen0/1/2 scans — expensive)
3. Compact memory (stop-the-world on LOH compaction — very expensive)

**Tools to measure allocations:**

```bash
dotnet tool install -g dotnet-trace
dotnet tool install -g dotnet-counters
dotnet tool install -g dotnet-dump

# Live GC counters
dotnet counters monitor --process-id <PID> System.Runtime

# Trace GC with allocations
dotnet trace collect --process-id <PID> \
  --providers "Microsoft-DotNETRuntime:0x1:5" \
  --output trace.nettrace
```

```csharp
// Quick in-process allocation measurement
using System.Diagnostics;

long before = GC.GetAllocatedBytesForCurrentThread();
// ... code to measure ...
long allocated = GC.GetAllocatedBytesForCurrentThread() - before;
Console.WriteLine($"Allocated: {allocated:N0} bytes");
```

---

## Step 1654: Span<T> — Stack-Allocated, Slice, Zero-Copy

`Span<T>` is a ref struct — it lives on the stack, never escapes to the heap, and can point to stack memory, heap memory, or native memory, all uniformly.

```csharp
using System;

// Slicing an array — NO allocation, just a view
int[] array = { 1, 2, 3, 4, 5, 6, 7, 8 };
Span<int> full = array;            // whole array
Span<int> middle = array.AsSpan(2, 4);  // elements [2..5], no copy
middle[0] = 99;                    // mutates the original array

// Stack-allocated memory
Span<byte> stackBuffer = stackalloc byte[256]; // on the stack, zero heap
stackBuffer.Fill(0);

// String slicing without substring allocation
ReadOnlySpan<char> line = "Hello, World!";
ReadOnlySpan<char> greeting = line.Slice(0, 5);  // no String.Substring allocation
Console.WriteLine(greeting.ToString()); // "Hello"

// SequenceEqual, IndexOf, Contains — no allocation
bool hasComma = line.Contains(',');
int idx = line.IndexOf(',');
```

### Parsing CSV without allocations

```csharp
using System;
using System.Collections.Generic;

public static List<(ReadOnlyMemory<char> Field1, ReadOnlyMemory<char> Field2)>
    ParseCsvNoAlloc(string csv)
{
    var results = new List<(ReadOnlyMemory<char>, ReadOnlyMemory<char>)>();
    var memory = csv.AsMemory();
    int start = 0;

    while (start < memory.Length)
    {
        int end = memory.Span.Slice(start).IndexOf('\n');
        if (end < 0) end = memory.Length - start;

        var line = memory.Slice(start, end);
        int comma = line.Span.IndexOf(',');
        if (comma >= 0)
        {
            results.Add((line.Slice(0, comma), line.Slice(comma + 1)));
        }

        start += end + 1;
    }

    return results;
    // ReadOnlyMemory<char> slices reference the ORIGINAL string — no copies
}
```

### Zero-allocation integer parsing

```csharp
ReadOnlySpan<char> text = "12345";
if (int.TryParse(text, out int value))
{
    Console.WriteLine(value); // 12345, no intermediate string created
}

// Format into a stack buffer
Span<char> buffer = stackalloc char[20];
if (value.TryFormat(buffer, out int written))
{
    var formatted = buffer.Slice(0, written);  // ReadOnlySpan<char>
    Console.WriteLine(formatted.ToString());    // only allocates here when needed
}
```

---

## Step 1655: Memory<T> and IMemoryOwner<T>

`Memory<T>` is the heap-safe sibling of `Span<T>` — it can be stored in fields, captured by async lambdas, and passed across `await` boundaries.

```csharp
using System;
using System.Buffers;
using System.IO;
using System.IO.Pipelines;
using System.Threading;
using System.Threading.Tasks;

// Memory<T> can cross async boundaries; Span<T> cannot
public async Task ProcessLargeDataAsync(Memory<byte> buffer, CancellationToken ct)
{
    await Task.Delay(1, ct); // await is fine — Memory<T> is on the heap
    buffer.Span.Fill(0);     // access the underlying Span synchronously
}

// IMemoryOwner<T> — take ownership of rented memory, dispose to return it
public async Task ReadAndProcessAsync(Stream stream, CancellationToken ct)
{
    using IMemoryOwner<byte> owner = MemoryPool<byte>.Shared.Rent(4096);
    Memory<byte> buffer = owner.Memory;

    int bytesRead = await stream.ReadAsync(buffer, ct);
    if (bytesRead == 0) return;

    ProcessBuffer(buffer.Slice(0, bytesRead).Span);
    // owner.Dispose() returns the buffer to the pool
}

private static void ProcessBuffer(ReadOnlySpan<byte> data) { /* ... */ }
```

### System.IO.Pipelines — Zero-Copy I/O

`PipeReader`/`PipeWriter` are the idiomatic way to handle streaming I/O without allocating intermediate buffers.

```csharp
using System.IO.Pipelines;
using System.Text;

public static async Task ParseLinesAsync(Stream stream, CancellationToken ct)
{
    var pipe = new Pipe();

    // Fill the pipe from the stream in a background task
    var fillTask = FillPipeAsync(stream, pipe.Writer, ct);

    // Read from the pipe without allocating strings per line
    await ReadPipeAsync(pipe.Reader, ct);

    await fillTask;
}

private static async Task FillPipeAsync(Stream stream, PipeWriter writer, CancellationToken ct)
{
    const int bufferSize = 4096;
    try
    {
        while (true)
        {
            Memory<byte> buffer = writer.GetMemory(bufferSize);
            int bytesRead = await stream.ReadAsync(buffer, ct);
            if (bytesRead == 0) break;

            writer.Advance(bytesRead);
            FlushResult result = await writer.FlushAsync(ct);
            if (result.IsCompleted) break;
        }
    }
    finally
    {
        await writer.CompleteAsync();
    }
}

private static async Task ReadPipeAsync(PipeReader reader, CancellationToken ct)
{
    while (true)
    {
        ReadResult result = await reader.ReadAsync(ct);
        ReadOnlySequence<byte> buffer = result.Buffer;

        while (TryReadLine(ref buffer, out ReadOnlySequence<byte> line))
        {
            ProcessLine(line); // process WITHOUT copying to a string
        }

        reader.AdvanceTo(buffer.Start, buffer.End);
        if (result.IsCompleted) break;
    }
    await reader.CompleteAsync();
}

private static bool TryReadLine(
    ref ReadOnlySequence<byte> buffer,
    out ReadOnlySequence<byte> line)
{
    var reader = new SequenceReader<byte>(buffer);
    if (reader.TryReadTo(out line, (byte)'\n', advancePastDelimiter: true))
    {
        buffer = buffer.Slice(reader.Position);
        return true;
    }
    line = default;
    return false;
}

private static void ProcessLine(ReadOnlySequence<byte> line)
{
    // Use SequenceReader to parse fields without allocating
    var reader = new SequenceReader<byte>(line);
    Span<char> chars = stackalloc char[(int)line.Length];
    int written = Encoding.UTF8.GetChars(line.FirstSpan, chars);
    // process chars[..written]
}
```

---

## Step 1656: ArrayPool<T> — Rent, Use, Return

```csharp
using System.Buffers;

// WRONG: Allocating a new array for every request
byte[] buffer = new byte[1024]; // GC pressure if done per-request

// RIGHT: Rent from the shared pool
byte[] rented = ArrayPool<byte>.Shared.Rent(1024);
try
{
    // Note: rented.Length may be LARGER than 1024 — pools round up to power-of-2
    int actualLength = Math.Min(rented.Length, 1024);
    var usable = rented.AsSpan(0, actualLength);
    DoWork(usable);
}
finally
{
    ArrayPool<byte>.Shared.Return(rented, clearArray: false); // clearArray=true if buffer held sensitive data
}
```

### Custom ArrayPool for specific sizes

```csharp
// Create a pool tuned for your workload
var pool = ArrayPool<byte>.Create(
    maxArrayLength: 1024 * 1024, // 1 MB max single array
    maxArraysPerBucket: 50);      // 50 arrays per size bucket

// Pool is thread-safe
byte[] buf = pool.Rent(4096);
try { /* ... */ }
finally { pool.Return(buf); }
```

### RecyclableMemoryStream — pooled MemoryStream

```csharp
// Install: Microsoft.IO.RecyclableMemoryStream
using Microsoft.IO;

var manager = new RecyclableMemoryStreamManager();

// Each stream uses pooled buffers internally — no large byte[] allocations
using var stream = manager.GetStream("my-tag");
await JsonSerializer.SerializeAsync(stream, myObject);
stream.Position = 0;
var result = await JsonSerializer.DeserializeAsync<MyObject>(stream);
// Stream disposal returns all buffers to the pool
```

---

## Step 1657: ValueTask<T> — Avoiding Task Allocations on Hot Paths

```csharp
using System.Threading.Tasks;

// WRONG for hot paths: Task<T> always allocates a Task object on the heap
public async Task<int> ReadValueAsync()
{
    return await _reader.ReadInt32Async(); // always allocates a Task<int>
}

// RIGHT for hot paths: ValueTask<T> avoids allocation when result is synchronous
public ValueTask<int> ReadValueAsync()
{
    if (_buffer.TryReadInt32(out int value))
        return ValueTask.FromResult(value); // NO allocation — wraps the value directly

    return SlowPathAsync();
}

private async ValueTask<int> SlowPathAsync()
{
    await _semaphore.WaitAsync();
    try { return await _reader.ReadInt32Async(); }
    finally { _semaphore.Release(); }
}
```

```csharp
// IValueTaskSource<T> — ultimate zero-allocation async with manual state machine
using System.Threading.Tasks.Sources;

public class SocketAwaitable : IValueTaskSource<int>
{
    private ManualResetValueTaskSourceCore<int> _core;

    public ValueTask<int> WaitAsync() => new(this, _core.Version);

    public void SetResult(int result) => _core.SetResult(result);
    public void SetException(Exception ex) => _core.SetException(ex);

    int IValueTaskSource<int>.GetResult(short token) => _core.GetResult(token);
    ValueTaskSourceStatus IValueTaskSource<int>.GetStatus(short token) => _core.GetStatus(token);
    void IValueTaskSource<int>.OnCompleted(
        Action<object?> continuation, object? state, short token,
        ValueTaskSourceOnCompletedFlags flags)
        => _core.OnCompleted(continuation, state, token, flags);

    public void Reset() => _core.Reset();
}
```

---

## Step 1658: Unsafe Code and NativeMemory

For ultimate control — when you need to avoid GC entirely for large buffers:

```csharp
using System.Runtime.InteropServices;

// Allocate OUTSIDE the GC heap — no GC pressure, no collection pauses
nuint byteCount = 1024 * 1024; // 1 MB
void* ptr = NativeMemory.AllocZeroed(byteCount);
try
{
    var span = new Span<byte>(ptr, (int)byteCount);
    span.Fill(0xAB);

    // Process large buffer without any GC involvement
    ProcessLargeBuffer(span);
}
finally
{
    NativeMemory.Free(ptr); // MUST free manually
}
```

```csharp
// MemoryMarshal helpers for low-level reinterpretation
using System.Runtime.InteropServices;

[StructLayout(LayoutKind.Sequential, Pack = 1)]
struct PacketHeader
{
    public uint Magic;
    public ushort Version;
    public ushort PayloadLength;
}

ReadOnlySpan<byte> rawBytes = GetPacketBytes();
ref readonly PacketHeader header = ref MemoryMarshal.AsRef<PacketHeader>(rawBytes);
Console.WriteLine($"Magic: 0x{header.Magic:X8}, Version: {header.Version}");
// Zero-copy struct parsing from raw bytes
```

---

## Step 1659: BenchmarkDotNet — Rigorous Micro-Benchmarking

```xml
<PackageReference Include="BenchmarkDotNet" Version="0.14.0" />
```

```csharp
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Running;
using System;
using System.Text;

// Run with: dotnet run -c Release
BenchmarkRunner.Run<StringBenchmarks>();

[MemoryDiagnoser]          // shows allocations per operation
[ThreadingDiagnoser]       // shows thread contention
[DisassemblyDiagnoser]     // shows JIT assembly output
[RankColumn]               // ranks results fastest to slowest
[CsvExporter]              // exports results as CSV
public class StringBenchmarks
{
    private const int Iterations = 1000;
    private readonly string[] _parts = Enumerable.Range(0, Iterations)
        .Select(i => $"part{i}").ToArray();

    [Benchmark(Baseline = true)]
    public string StringConcat()
    {
        string result = "";
        foreach (var part in _parts) result += part;
        return result;
    }

    [Benchmark]
    public string StringBuilder()
    {
        var sb = new StringBuilder();
        foreach (var part in _parts) sb.Append(part);
        return sb.ToString();
    }

    [Benchmark]
    public string StringCreate()
    {
        int totalLen = _parts.Sum(p => p.Length);
        return string.Create(totalLen, _parts, static (span, parts) =>
        {
            int pos = 0;
            foreach (var part in parts)
            {
                part.CopyTo(span.Slice(pos));
                pos += part.Length;
            }
        });
    }

    [Benchmark]
    public string StringJoin() => string.Join("", _parts);
}
```

```csharp
// Benchmark with parameters
[MemoryDiagnoser]
public class ArrayPoolBenchmarks
{
    [Params(256, 4096, 65536, 1_048_576)]
    public int BufferSize { get; set; }

    [Benchmark(Baseline = true)]
    public byte[] NewArray() => new byte[BufferSize];

    [Benchmark]
    public void RentAndReturn()
    {
        byte[] buffer = ArrayPool<byte>.Shared.Rent(BufferSize);
        ArrayPool<byte>.Shared.Return(buffer);
    }

    [Benchmark]
    public void NativeAlloc()
    {
        unsafe
        {
            void* ptr = NativeMemory.Alloc((nuint)BufferSize);
            NativeMemory.Free(ptr);
        }
    }
}
```

### Reading BenchmarkDotNet results

```
| Method       | BufferSize |       Mean |    Error |   StdDev | Ratio |   Gen0 | Allocated |
|------------- |----------- |-----------:|---------:|---------:|------:|-------:|----------:|
| NewArray     |        256 |   4.623 ns | 0.019 ns | 0.015 ns |  1.00 | 0.0057 |      32 B |
| RentAndReturn|        256 |   8.211 ns | 0.031 ns | 0.027 ns |  1.78 |      - |         - |  ← no alloc!
| NativeAlloc  |        256 |  13.401 ns | 0.043 ns | 0.040 ns |  2.90 |      - |         - |
| NewArray     |    1048576 | 127.451 ns | 0.421 ns | 0.373 ns |  1.00 |      - | 1048592 B |
| RentAndReturn|    1048576 |  30.211 ns | 0.044 ns | 0.037 ns |  0.24 |      - |         - |  ← 4× faster
```

---

## Step 1660: Struct Layout Optimization

Cache lines are 64 bytes. Hot structs should fit in as few as possible.

```csharp
using System.Runtime.InteropServices;

// WRONG: padding wastes cache lines
struct BadLayout
{
    public byte Flag;     // 1 byte + 7 bytes padding
    public double Value;  // 8 bytes
    public byte Status;   // 1 byte + 7 bytes padding
    public double Extra;  // 8 bytes
    // Total: 32 bytes (2 cache-line slots wasted)
}

// RIGHT: pack smaller fields together
[StructLayout(LayoutKind.Sequential, Pack = 1)]
struct GoodLayout
{
    public byte Flag;   // 1 byte
    public byte Status; // 1 byte
    // 6 bytes padding for alignment of double
    public double Value; // 8 bytes
    public double Extra; // 8 bytes
    // Total: 18 bytes (fits in 1 cache line)
}

// Validate your layout at compile time
[StructLayout(LayoutKind.Explicit)]
struct ExplicitLayout
{
    [FieldOffset(0)] public long Timestamp;   // 8 bytes at offset 0
    [FieldOffset(8)] public int  OrderId;     // 4 bytes at offset 8
    [FieldOffset(12)] public short Quantity;  // 2 bytes at offset 12
    [FieldOffset(14)] public byte Status;     // 1 byte at offset 14
    // Total: 15 bytes (no padding, fully controlled)
}
```

```csharp
// SoA (Structure of Arrays) vs AoS (Array of Structures)
// AoS — bad for SIMD/vectorized operations
struct Point3D_AoS { public float X, Y, Z; }
Point3D_AoS[] points = new Point3D_AoS[10000]; // [XYZXYZXYZXYZ...]

// SoA — optimal for SIMD operations on one axis
struct Points3D_SoA
{
    public float[] X;
    public float[] Y;
    public float[] Z;
}
// process all X values contiguously — excellent cache performance
```

---

## Step 1661: SIMD Intrinsics with System.Runtime.Intrinsics

```csharp
using System.Runtime.Intrinsics;
using System.Runtime.Intrinsics.X86;
using System.Runtime.CompilerServices;

public static class VectorizedOps
{
    // Sum array of floats using AVX2 (8 floats per operation)
    public static float SumFloats(ReadOnlySpan<float> data)
    {
        if (!Avx.IsSupported || data.Length < 8)
            return SumFloatsScalar(data);

        return SumFloatsAvx(data);
    }

    private static unsafe float SumFloatsAvx(ReadOnlySpan<float> data)
    {
        var accumulator = Vector256<float>.Zero;
        int i = 0;
        int vectorLength = Vector256<float>.Count; // 8

        fixed (float* ptr = data)
        {
            for (; i <= data.Length - vectorLength; i += vectorLength)
            {
                var vec = Avx.LoadVector256(ptr + i);
                accumulator = Avx.Add(accumulator, vec);
            }
        }

        // Horizontal sum of the 8-element accumulator
        var sum128 = Sse.Add(
            accumulator.GetLower(),
            accumulator.GetUpper()); // reduce 256 to 128
        sum128 = Sse.Add(sum128, Sse.MoveHighToLow(sum128, sum128));
        sum128 = Sse.AddScalar(sum128, Sse.Shuffle(sum128, sum128, 1));

        float total = sum128.ToScalar();

        // Handle remaining elements
        for (; i < data.Length; i++) total += data[i];
        return total;
    }

    private static float SumFloatsScalar(ReadOnlySpan<float> data)
    {
        float sum = 0;
        foreach (float f in data) sum += f;
        return sum;
    }
}
```

### Using `Vector<T>` for portable SIMD

```csharp
using System.Numerics;

public static void MultiplyVectors(
    Span<float> result,
    ReadOnlySpan<float> a,
    ReadOnlySpan<float> b)
{
    int vectorSize = Vector<float>.Count; // hardware-determined (4, 8, or 16)
    int i = 0;

    for (; i <= result.Length - vectorSize; i += vectorSize)
    {
        var va = new Vector<float>(a.Slice(i));
        var vb = new Vector<float>(b.Slice(i));
        (va * vb).CopyTo(result.Slice(i));
    }

    for (; i < result.Length; i++) result[i] = a[i] * b[i];
}
```

---

## Step 1662: GC Tuning and Configuration

```json
// runtimeconfig.json — configure GC behaviour
{
  "configProperties": {
    "System.GC.Server": true,         // Server GC: one heap per CPU core
    "System.GC.Concurrent": true,     // Background GC (default in .NET 6+)
    "System.GC.HeapHardLimit": 536870912,  // 512 MB hard limit
    "System.GC.HeapHardLimitPercent": 50,  // OR 50% of container memory
    "System.GC.NoAffinitize": false,  // Keep GC threads on dedicated CPUs
    "System.GC.GCConserveMemory": 5,  // 0-9 scale; higher = more aggressive collection
    "System.GC.HighMemoryPercent": 90 // Trigger aggressive GC at 90% of heap limit
  }
}
```

```csharp
// Programmatic GC control
using System.Runtime;
using System;

// Force a full blocking collection (avoid in production hot paths)
GC.Collect(2, GCCollectionMode.Forced, blocking: true);
GC.WaitForPendingFinalizers();

// Suppress finalizer when you manually cleaned up
public class ManagedResource : IDisposable
{
    private bool _disposed;
    private readonly IntPtr _handle;

    public ManagedResource() => _handle = AllocateHandle();

    public void Dispose()
    {
        if (_disposed) return;
        FreeHandle(_handle);
        _disposed = true;
        GC.SuppressFinalize(this); // prevents finalizer from running
    }

    ~ManagedResource() => FreeHandle(_handle); // safety net
}

// Pin memory to prevent GC movement (use sparingly)
GCHandle pin = GCHandle.Alloc(myArray, GCHandleType.Pinned);
try
{
    IntPtr ptr = pin.AddrOfPinnedObject();
    // Pass ptr to native code
}
finally
{
    pin.Free();
}
```

---

## Step 1663: Object Pooling with ObjectPool<T>

```csharp
using Microsoft.Extensions.ObjectPool;

// Custom pool policy
public class HttpClientMessagePolicy : IPooledObjectPolicy<HttpRequestMessage>
{
    public HttpRequestMessage Create() => new HttpRequestMessage();

    public bool Return(HttpRequestMessage obj)
    {
        obj.Headers.Clear();
        obj.Content?.Dispose();
        obj.Content = null;
        obj.Method = HttpMethod.Get;
        obj.RequestUri = null;
        return true; // true = return to pool; false = discard
    }
}

// Register in DI
builder.Services.AddSingleton<ObjectPoolProvider, DefaultObjectPoolProvider>();
builder.Services.AddSingleton(sp =>
{
    var provider = sp.GetRequiredService<ObjectPoolProvider>();
    return provider.Create(new HttpClientMessagePolicy());
});

// Use in a service
public class OrderService(ObjectPool<HttpRequestMessage> pool)
{
    public async Task<string> FetchAsync(string url, CancellationToken ct)
    {
        var request = pool.Get();
        try
        {
            request.Method = HttpMethod.Get;
            request.RequestUri = new Uri(url);
            using var response = await _client.SendAsync(request, ct);
            return await response.Content.ReadAsStringAsync(ct);
        }
        finally
        {
            pool.Return(request);
        }
    }
}
```

---

## Step 1664: Frozen Collections — Read-Only, JIT-Optimized Lookups

.NET 8 introduced `FrozenDictionary<K,V>` and `FrozenSet<T>`: immutable, highly optimized for read-heavy scenarios (startup configuration, lookup tables).

```csharp
using System.Collections.Frozen;
using System.Collections.Generic;

// Build once at startup
var countryData = new Dictionary<string, string>
{
    ["US"] = "United States",
    ["GB"] = "United Kingdom",
    ["DE"] = "Germany",
    ["JP"] = "Japan",
    // ... hundreds of entries
};

// Freeze — creates optimized internal structures, perfect hashing possible
FrozenDictionary<string, string> frozen = countryData.ToFrozenDictionary();

// Lookups are 2-4× faster than regular Dictionary for read-only usage
bool found = frozen.TryGetValue("US", out string? name);

// FrozenSet for membership tests
FrozenSet<string> validCurrencies = new[] { "USD", "EUR", "GBP", "JPY" }
    .ToFrozenSet(StringComparer.OrdinalIgnoreCase);
bool isValid = validCurrencies.Contains("usd"); // true
```

---

## Step 1665: Profile-Guided Optimization (PGO)

PGO allows the JIT to collect profiling data from a real run and use it to make better compilation decisions on subsequent runs.

```json
// runtimeconfig.json — enable Dynamic PGO (default in .NET 8+)
{
  "configProperties": {
    "System.Runtime.TieredCompilation": true,
    "System.Runtime.TieredPGO": true,
    "System.Runtime.ReadyToRun": true
  }
}
```

```bash
# Static PGO with crossgen2 (best for containerized production)
# Step 1: instrument build
dotnet publish -c Release -r linux-x64 --self-contained \
  /p:PublishReadyToRun=true \
  /p:PublishReadyToRunShowWarnings=true

# Step 2: collect profile data with PGO instrumentation
dotnet publish -c Release -r linux-x64 --self-contained \
  /p:PublishReadyToRun=true \
  /p:RunCrossGen2=true \
  /p:TieredPGO=true
```

---

## Step 1666: Native AOT — Zero JIT, Ultra-Fast Startup

NativeAOT compiles your entire .NET app to a standalone native binary ahead of time.

```xml
<!-- MyApp.csproj -->
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <PublishAot>true</PublishAot>
    <InvariantGlobalization>true</InvariantGlobalization>   <!-- smaller binary -->
    <StripSymbols>true</StripSymbols>
    <TrimmerRootDescriptor>TrimmerRoots.xml</TrimmerRootDescriptor>
  </PropertyGroup>
</Project>
```

```bash
# Publish a native binary (requires the native toolchain installed)
dotnet publish -c Release -r linux-x64

# Result: ~10 MB self-contained native binary
# Startup time: < 5ms (vs ~50ms for JIT cold start)
# Memory: ~30 MB RSS (vs ~100 MB for JIT)
```

**AOT-compatible minimal API:**

```csharp
using Microsoft.AspNetCore.Builder;
using System.Text.Json.Serialization;

var builder = WebApplication.CreateSlimBuilder(args); // slim = AOT-compatible

// AOT requires source-generated JSON serialization (no reflection)
builder.Services.ConfigureHttpJsonOptions(options =>
{
    options.SerializerOptions.TypeInfoResolverChain.Insert(0, AppJsonContext.Default);
});

var app = builder.Build();

app.MapGet("/health", () => new HealthResponse("ok", DateTime.UtcNow));
app.MapGet("/orders/{id:guid}", (Guid id) => new OrderResponse(id, "pending"));

app.Run();

// Source-generated JSON context — required for AOT, avoids reflection
[JsonSerializable(typeof(HealthResponse))]
[JsonSerializable(typeof(OrderResponse))]
[JsonSourceGenerationOptions(PropertyNamingPolicy = JsonKnownNamingPolicy.CamelCase)]
internal partial class AppJsonContext : JsonSerializerContext { }

record HealthResponse(string Status, DateTime Timestamp);
record OrderResponse(Guid Id, string Status);
```

**AOT-incompatible patterns to avoid:**

```csharp
// WRONG for AOT: reflection-based code
var obj = Activator.CreateInstance(Type.GetType("MyApp.MyClass")!); // ← reflection
var props = typeof(MyClass).GetProperties(BindingFlags.Public | BindingFlags.Instance); // ← reflection

// RIGHT for AOT: use source generators
// JsonSerializer.Serialize with AppJsonContext.Default.MyClass (source-generated)
// IIncrementalGenerator for code generation
// Expression trees are NOT supported in AOT
```

---

## Step 1667: Trimming — Reducing Binary Size

```xml
<PublishTrimmed>true</PublishTrimmed>
<TrimMode>full</TrimMode>  <!-- trim all unreachable code -->
```

```csharp
// Annotate code to help the trimmer
using System.Diagnostics.CodeAnalysis;

// Tell the trimmer that this method uses reflection on T
[RequiresDynamicCode("Requires dynamic code generation")]
[RequiresUnreferencedCode("Requires all members of T to be preserved")]
public static T DeserializeXml<T>(string xml)
{
    // ...
}

// Preserve specific members that trimmer would remove
[DynamicallyAccessedMembers(DynamicallyAccessedMemberTypes.PublicProperties)]
public static void Serialize(Type type, object obj) { /* uses reflection */ }
```

```xml
<!-- TrimmerRoots.xml — preserve types the trimmer can't detect -->
<linker>
  <assembly fullname="MyApp">
    <type fullname="MyApp.Plugins.*" preserve="all" />
  </assembly>
</linker>
```

---

## Step 1668: Complete Benchmark Suite — Before vs. After

```csharp
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Running;
using System;
using System.Buffers;
using System.Text;
using System.Text.Json;

BenchmarkRunner.Run<HttpRequestParseBenchmarks>();

[MemoryDiagnoser]
[RankColumn]
public class HttpRequestParseBenchmarks
{
    private const string RequestLine = "GET /api/orders/123?page=1&size=20 HTTP/1.1\r\n";
    private readonly byte[] _requestBytes = Encoding.ASCII.GetBytes(RequestLine);

    // Naive: many allocations
    [Benchmark(Baseline = true)]
    public (string Method, string Path, string Version) NaiveParse()
    {
        var line = Encoding.ASCII.GetString(_requestBytes);        // alloc string
        var parts = line.Split(' ');                                // alloc string[]
        return (parts[0], parts[1], parts[2].TrimEnd('\r', '\n')); // alloc trimmed
    }

    // Span-based: zero allocations
    [Benchmark]
    public (ReadOnlySpan<byte> Method, ReadOnlySpan<byte> Path, ReadOnlySpan<byte> Version)
        SpanParse()
    {
        ReadOnlySpan<byte> span = _requestBytes;

        int sp1 = span.IndexOf((byte)' ');
        var method = span.Slice(0, sp1);
        span = span.Slice(sp1 + 1);

        int sp2 = span.IndexOf((byte)' ');
        var path = span.Slice(0, sp2);
        span = span.Slice(sp2 + 1);

        // trim \r\n
        var version = span.TrimEnd(new byte[] { (byte)'\r', (byte)'\n' });
        return (method, path, version);
    }
}

[MemoryDiagnoser]
[RankColumn]
public class JsonSerializationBenchmarks
{
    private readonly Order _order = new(Guid.NewGuid(), "CUST-001",
        new[] { new OrderLine("SKU-1", 2, 9.99m) }, DateTime.UtcNow);

    private readonly byte[] _jsonBytes;
    private readonly JsonSerializerOptions _options = new()
        { PropertyNamingPolicy = JsonNamingPolicy.CamelCase };

    public JsonSerializationBenchmarks()
        => _jsonBytes = JsonSerializer.SerializeToUtf8Bytes(_order, _options);

    [Benchmark(Baseline = true)]
    public string SerializeToString()
        => JsonSerializer.Serialize(_order, _options);

    [Benchmark]
    public byte[] SerializeToUtf8Bytes()
        => JsonSerializer.SerializeToUtf8Bytes(_order, _options);

    [Benchmark]
    public Order? Deserialize()
        => JsonSerializer.Deserialize<Order>(_jsonBytes, _options);

    [Benchmark]
    public async Task SerializeToStreamAsync()
    {
        using var ms = new System.IO.MemoryStream();
        await JsonSerializer.SerializeAsync(ms, _order, _options);
    }
}

record Order(Guid Id, string CustomerId, OrderLine[] Lines, DateTime CreatedAt);
record OrderLine(string Sku, int Quantity, decimal UnitPrice);
```

---

## Performance Checklist

| Area | What to do |
|---|---|
| Strings | `string.Create`, `StringBuilder` for concatenation, `Span<char>` for parsing |
| Arrays / buffers | `ArrayPool<T>.Shared.Rent/Return`, never `new byte[N]` per request |
| Async return | `ValueTask<T>` on hot paths; `Task<T>` only when always async |
| JSON | `SerializeToUtf8Bytes` over `Serialize` (string), source-generated for AOT |
| Collections | `FrozenDictionary` for read-only lookups, `ImmutableArray` for small lists |
| GC | `Server GC` in server apps, `HeapHardLimit` in containers |
| Startup | `NativeAOT` for CLIs / lambdas, `ReadyToRun` for web servers |
| Benchmarking | Always `dotnet run -c Release`, always `[MemoryDiagnoser]`, warmup=auto |
| Profiling | `dotnet-trace`, `dotnet-counters`, PerfView, JetBrains dotMemory |
| Struct layout | Pack hot structs ≤ 64 bytes, prefer SoA over AoS for SIMD |

### What Next?

- **Part 65**: Security Deep Dive — OWASP Top 10 mitigations, Data Protection API, CSP/HSTS headers, SAST integration
- **Part 66**: Kubernetes & Container Orchestration — Helm charts, KEDA autoscaling, readiness/liveness probes, Dapr
- **Part 67**: Messaging Patterns — MassTransit, Wolverine, Outbox pattern, Saga orchestration vs choreography
