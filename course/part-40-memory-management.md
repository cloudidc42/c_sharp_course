# Part 40: Memory Management & Unsafe Code

## Steps 1151-1180 | ระดับโลก (World-Class)

---

## Step 1151: .NET Memory Model

```
.NET Memory Layout:
┌────────────────────────────────────────────┐
│                Stack                        │
│  - Value types (int, bool, struct)         │
│  - Method parameters, local variables      │
│  - Fast allocation/deallocation (LIFO)     │
│  - Default ~1MB per thread                 │
├────────────────────────────────────────────┤
│                Heap (Managed)              │
│  - Reference types (class, arrays, string) │
│  - GC manages allocation/deallocation      │
│  ┌──────────────────────────────────────┐  │
│  │  Small Object Heap (SOH)            │  │
│  │  Gen 0 → Gen 1 → Gen 2             │  │
│  │  Objects < 85,000 bytes             │  │
│  ├──────────────────────────────────────┤  │
│  │  Large Object Heap (LOH)            │  │
│  │  Objects >= 85,000 bytes            │  │
│  │  Not compacted by default           │  │
│  ├──────────────────────────────────────┤  │
│  │  Pinned Object Heap (POH) (.NET 5+) │  │
│  │  Pinned buffers, interop            │  │
│  └──────────────────────────────────────┘  │
├────────────────────────────────────────────┤
│            Unmanaged Memory                │
│  - NativeMemory, Marshal.AllocHGlobal      │
│  - P/Invoke buffers                        │
└────────────────────────────────────────────┘
```

---

## Step 1152: Garbage Collector Fundamentals

```csharp
// GC Generations
// Gen 0: newly allocated, collected frequently (fast)
// Gen 1: survived 1 Gen 0 collection (buffer)
// Gen 2: survived 1+ Gen 1 collection, long-lived (slowest)

public class GcDiagnostics
{
    public static void PrintGcInfo()
    {
        for (int gen = 0; gen <= GC.MaxGeneration; gen++)
            Console.WriteLine($"Gen {gen} collections: {GC.CollectionCount(gen)}");

        Console.WriteLine($"Total memory: {GC.GetTotalMemory(false) / 1024 / 1024} MB");
        Console.WriteLine($"GC latency mode: {GCSettings.LatencyMode}");
        Console.WriteLine($"Is server GC: {GCSettings.IsServerGC}");
    }

    // Force collection (diagnostic use only)
    public static void ForceFullGc()
    {
        GC.Collect(GC.MaxGeneration, GCCollectionMode.Forced, blocking: true, compacting: true);
        GC.WaitForPendingFinalizers();
        GC.Collect();
    }

    // GC memory info (.NET 5+)
    public static void PrintGcMemoryInfo()
    {
        var info = GC.GetGCMemoryInfo();
        Console.WriteLine($"Heap size bytes: {info.HeapSizeBytes:N0}");
        Console.WriteLine($"Memory load bytes: {info.MemoryLoadBytes:N0}");
        Console.WriteLine($"Total committed: {info.TotalCommittedBytes:N0}");
        Console.WriteLine($"Total available: {info.TotalAvailableMemoryBytes:N0}");
    }
}
```

---

## Step 1153: IDisposable and using

```csharp
// ✅ IDisposable pattern
public class DatabaseConnection : IDisposable
{
    private readonly SqlConnection _connection;
    private bool _disposed;

    public DatabaseConnection(string connectionString)
    {
        _connection = new SqlConnection(connectionString);
        _connection.Open();
    }

    public void ExecuteQuery(string sql)
    {
        ObjectDisposedException.ThrowIf(_disposed, nameof(DatabaseConnection));
        using var cmd = new SqlCommand(sql, _connection);
        cmd.ExecuteNonQuery();
    }

    protected virtual void Dispose(bool disposing)
    {
        if (_disposed) return;

        if (disposing)
        {
            // Release managed resources
            _connection.Dispose();
        }

        // Release unmanaged resources here (if any)
        _disposed = true;
    }

    public void Dispose()
    {
        Dispose(disposing: true);
        GC.SuppressFinalize(this);
    }

    // Finalizer: safety net for unmanaged resources
    ~DatabaseConnection() => Dispose(disposing: false);
}

// IAsyncDisposable
public class AsyncResource : IAsyncDisposable
{
    private readonly Stream _stream;
    private bool _disposed;

    public AsyncResource(string path)
        => _stream = File.OpenRead(path);

    public async Task<byte[]> ReadAllAsync(CancellationToken ct)
    {
        ObjectDisposedException.ThrowIf(_disposed, nameof(AsyncResource));
        var buffer = new byte[_stream.Length];
        await _stream.ReadAsync(buffer, ct);
        return buffer;
    }

    public async ValueTask DisposeAsync()
    {
        if (_disposed) return;
        _disposed = true;
        await _stream.DisposeAsync();
        GC.SuppressFinalize(this);
    }
}

// Usage
await using var resource = new AsyncResource("file.dat");
var data = await resource.ReadAllAsync(ct);
```

---

## Step 1154: Span<T> และ Memory<T>

```csharp
// Span<T>: zero-allocation slice (stack only)
public static void SpanExamples()
{
    // Slice array without allocation
    int[] array = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
    Span<int> middle = array.AsSpan(3, 4);   // [4, 5, 6, 7] - no allocation!

    // Span on stack
    Span<byte> stackBuffer = stackalloc byte[256];

    // Span from string (read-only)
    ReadOnlySpan<char> text = "Hello, World!".AsSpan();
    ReadOnlySpan<char> hello = text[..5]; // "Hello" - no allocation

    // Parse without allocation
    ReadOnlySpan<char> numberText = "12345";
    if (int.TryParse(numberText, out int num))
        Console.WriteLine(num); // 12345
}

// Memory<T>: can be stored in heap, supports async
public class MemoryExample
{
    private readonly Memory<byte> _buffer = new byte[4096];

    public async Task ReadIntoBufferAsync(Stream stream, CancellationToken ct)
    {
        // Memory<T> can be used with async
        var bytesRead = await stream.ReadAsync(_buffer, ct);
        var data = _buffer[..bytesRead];
        ProcessData(data.Span);
    }

    private static void ProcessData(ReadOnlySpan<byte> data)
    {
        for (int i = 0; i < data.Length; i++)
            _ = data[i]; // process each byte
    }
}

// SpanReader: parse binary protocol without allocation
public ref struct SpanReader(ReadOnlySpan<byte> data)
{
    private ReadOnlySpan<byte> _remaining = data;

    public bool TryReadByte(out byte value)
    {
        if (_remaining.IsEmpty) { value = 0; return false; }
        value = _remaining[0];
        _remaining = _remaining[1..];
        return true;
    }

    public bool TryReadInt32(out int value)
    {
        if (_remaining.Length < 4) { value = 0; return false; }
        value = BinaryPrimitives.ReadInt32BigEndian(_remaining);
        _remaining = _remaining[4..];
        return true;
    }

    public bool TryReadBytes(int count, out ReadOnlySpan<byte> bytes)
    {
        if (_remaining.Length < count) { bytes = default; return false; }
        bytes = _remaining[..count];
        _remaining = _remaining[count..];
        return true;
    }
}
```

---

## Step 1155: ArrayPool<T>

```csharp
// ArrayPool: reuse arrays, reduce GC pressure
public class ImageProcessor
{
    private readonly ArrayPool<byte> _pool = ArrayPool<byte>.Shared;

    public async Task ProcessImageAsync(Stream inputStream, Stream outputStream, CancellationToken ct)
    {
        // Rent buffer from pool (no allocation if pool has available)
        var buffer = _pool.Rent(64 * 1024);  // 64KB
        try
        {
            int bytesRead;
            while ((bytesRead = await inputStream.ReadAsync(buffer, ct)) > 0)
            {
                // Process the buffer (only use bytesRead bytes!)
                var validData = buffer.AsSpan(0, bytesRead);
                ApplyFilter(validData);
                await outputStream.WriteAsync(buffer, 0, bytesRead, ct);
            }
        }
        finally
        {
            _pool.Return(buffer, clearArray: true); // return to pool
        }
    }

    private static void ApplyFilter(Span<byte> data)
    {
        for (int i = 0; i < data.Length; i++)
            data[i] = (byte)(255 - data[i]); // invert
    }
}

// Custom ArrayPool for specific sizes
public class FixedSizePool<T>
{
    private readonly ConcurrentQueue<T[]> _pool = new();
    private readonly int _size;
    private int _created;
    private readonly int _maxPooled;

    public FixedSizePool(int arraySize, int maxPooled = 20)
    {
        _size = arraySize;
        _maxPooled = maxPooled;
    }

    public T[] Rent()
    {
        if (_pool.TryDequeue(out var array)) return array;
        Interlocked.Increment(ref _created);
        return new T[_size];
    }

    public void Return(T[] array)
    {
        if (array.Length != _size) return;
        if (_pool.Count >= _maxPooled) return;  // pool full, let GC handle it
        Array.Clear(array);
        _pool.Enqueue(array);
    }
}
```

---

## Step 1156: stackalloc

```csharp
// stackalloc: stack allocation (very fast, no GC)
public static class StackAllocExamples
{
    // Parse CSV line without heap allocation
    public static int CountFields(ReadOnlySpan<char> csvLine, char delimiter = ',')
    {
        int count = 1;
        foreach (var c in csvLine)
            if (c == delimiter) count++;
        return count;
    }

    // SHA256 hash with stackalloc
    public static bool VerifyHash(ReadOnlySpan<byte> data, ReadOnlySpan<byte> expectedHash)
    {
        Span<byte> hash = stackalloc byte[32]; // SHA256 = 32 bytes
        SHA256.HashData(data, hash);
        return hash.SequenceEqual(expectedHash);
    }

    // Small buffer on stack
    public static void FormatNumbers(ReadOnlySpan<int> numbers)
    {
        const int maxNumbers = 32;
        // Use stackalloc for small arrays, heap for large
        Span<char> buffer = numbers.Length <= maxNumbers
            ? stackalloc char[numbers.Length * 10]
            : new char[numbers.Length * 10];

        int written = 0;
        foreach (var n in numbers)
        {
            n.TryFormat(buffer[written..], out int w);
            written += w;
            buffer[written++] = ',';
        }

        Console.WriteLine(buffer[..Math.Max(0, written - 1)].ToString());
    }

    // GuidToByteArray without allocation
    public static bool TryGetGuidBytes(Guid guid, Span<byte> destination)
    {
        if (destination.Length < 16) return false;
        return guid.TryWriteBytes(destination);
    }
}
```

---

## Step 1157: ref struct

```csharp
// ref struct: can only live on stack, cannot be boxed
public ref struct FastParser
{
    private ReadOnlySpan<char> _remaining;

    public FastParser(ReadOnlySpan<char> input) => _remaining = input;

    public bool TryReadInt(out int value)
    {
        SkipWhitespace();
        var start = 0;
        while (start < _remaining.Length && char.IsDigit(_remaining[start]))
            start++;

        if (start == 0) { value = 0; return false; }

        if (int.TryParse(_remaining[..start], out value))
        {
            _remaining = _remaining[start..];
            return true;
        }
        return false;
    }

    public bool TryReadToken(out ReadOnlySpan<char> token)
    {
        SkipWhitespace();
        var start = 0;
        while (start < _remaining.Length && !char.IsWhiteSpace(_remaining[start]))
            start++;

        if (start == 0) { token = default; return false; }
        token = _remaining[..start];
        _remaining = _remaining[start..];
        return true;
    }

    private void SkipWhitespace()
    {
        int i = 0;
        while (i < _remaining.Length && char.IsWhiteSpace(_remaining[i])) i++;
        _remaining = _remaining[i..];
    }

    public bool IsEmpty => _remaining.IsEmpty;
}

// Usage
var parser = new FastParser("42 hello 100".AsSpan());
if (parser.TryReadInt(out int n1))
    Console.WriteLine(n1); // 42
if (parser.TryReadToken(out var token))
    Console.WriteLine(token.ToString()); // hello
```

---

## Step 1158: unsafe Code และ Pointer

```csharp
// unsafe block: ใช้ pointers โดยตรง
public static class UnsafeOperations
{
    // Fast memory copy
    public static unsafe void FastCopy(byte* source, byte* destination, int count)
    {
        Buffer.MemoryCopy(source, destination, count, count);
    }

    // Reinterpret cast (no conversion overhead)
    public static unsafe float IntBitsToFloat(int bits)
    {
        return *(float*)&bits;
    }

    public static unsafe int FloatToIntBits(float value)
    {
        return *(int*)&value;
    }

    // Fast array access (skip bounds checking)
    public static unsafe long SumUnsafe(int[] array)
    {
        fixed (int* ptr = array)
        {
            long sum = 0;
            int* current = ptr;
            int* end = ptr + array.Length;
            while (current < end)
                sum += *current++;
            return sum;
        }
    }

    // Struct with fixed-size buffer (unsafe)
    public unsafe struct NetworkPacketHeader
    {
        public uint Magic;
        public ushort Version;
        public ushort PayloadLength;
        public fixed byte Reserved[8];  // fixed-size array in struct
    }
}
```

---

## Step 1159: Unsafe.As (Reinterpret)

```csharp
// System.Runtime.CompilerServices.Unsafe
public static class UnsafeHelpers
{
    // Reinterpret Span<byte> as Span<int> (no copy)
    public static Span<int> AsIntSpan(Span<byte> bytes)
    {
        return MemoryMarshal.Cast<byte, int>(bytes);
    }

    // Read struct from byte span (zero-copy deserialization)
    [StructLayout(LayoutKind.Sequential, Pack = 1)]
    public struct PacketHeader
    {
        public uint Magic;
        public ushort Version;
        public uint PayloadLength;
    }

    public static ref readonly PacketHeader ReadHeader(ReadOnlySpan<byte> data)
    {
        if (data.Length < Unsafe.SizeOf<PacketHeader>())
            throw new ArgumentException("Data too short");
        return ref MemoryMarshal.AsRef<PacketHeader>(data);
    }

    // Zero-copy struct serialization
    public static ReadOnlySpan<byte> AsBytes<T>(ref T value) where T : struct
        => MemoryMarshal.AsBytes(MemoryMarshal.CreateSpan(ref value, 1));

    // Unsafe.Add: pointer arithmetic
    public static unsafe ref T GetElement<T>(ref T arrayStart, int index) where T : struct
        => ref Unsafe.Add(ref arrayStart, index);
}
```

---

## Step 1160: NativeMemory (.NET 6+)

```csharp
// NativeMemory: allocate unmanaged memory
public sealed class NativeBuffer : IDisposable
{
    private nint _ptr;
    private readonly int _size;
    private bool _disposed;

    public NativeBuffer(int size)
    {
        _size = size;
        unsafe { _ptr = (nint)NativeMemory.Alloc((nuint)size); }
    }

    public Span<byte> AsSpan()
    {
        ObjectDisposedException.ThrowIf(_disposed, nameof(NativeBuffer));
        unsafe { return new Span<byte>((void*)_ptr, _size); }
    }

    public void Dispose()
    {
        if (_disposed) return;
        _disposed = true;
        unsafe { NativeMemory.Free((void*)_ptr); }
        _ptr = 0;
        GC.SuppressFinalize(this);
    }

    ~NativeBuffer()
    {
        if (_disposed || _ptr == 0) return;
        unsafe { NativeMemory.Free((void*)_ptr); }
    }
}

// Aligned allocation for SIMD
public static class SimdBuffers
{
    // Allocate 32-byte aligned memory for AVX2
    public static unsafe void* AllocAligned(int size, int alignment = 32)
        => NativeMemory.AlignedAlloc((nuint)size, (nuint)alignment);

    public static unsafe void FreeAligned(void* ptr)
        => NativeMemory.AlignedFree(ptr);
}
```

---

## Step 1161: Memory Pressure và GC Hints

```csharp
// Notify GC of large unmanaged allocations
public class LargeUnmanagedBuffer(int sizeInBytes) : IDisposable
{
    private readonly nint _ptr;
    private bool _disposed;

    static LargeUnmanagedBuffer()
    {
        // Tell GC this type allocates unmanaged memory
        // so it can collect managed objects sooner to make room
    }

    public LargeUnmanagedBuffer(int sizeInBytes) : this(sizeInBytes)
    {
        unsafe { _ptr = (nint)NativeMemory.Alloc((nuint)sizeInBytes); }
        // Hint to GC that we have large unmanaged allocation
        GC.AddMemoryPressure(sizeInBytes);
    }

    public void Dispose()
    {
        if (_disposed) return;
        _disposed = true;
        unsafe { NativeMemory.Free((void*)_ptr); }
        GC.RemoveMemoryPressure(sizeInBytes); // release pressure
        GC.SuppressFinalize(this);
    }
}

// GC settings for performance
public static void ConfigureGcForPerformance()
{
    // Server GC for multi-core (better throughput, more memory)
    // Set in runtimeconfig.json: "System.GC.Server": true

    // Reduce latency: use SustainedLowLatency in critical sections
    var previousMode = GCSettings.LatencyMode;
    try
    {
        GCSettings.LatencyMode = GCLatencyMode.SustainedLowLatency;
        // Critical real-time work here
    }
    finally
    {
        GCSettings.LatencyMode = previousMode;
    }
}
```

---

## Step 1162: Object Pooling

```csharp
// Microsoft.Extensions.ObjectPool
public class ExpensiveObject : IResettable
{
    public StringBuilder Buffer { get; } = new();
    public List<string> Results { get; } = [];

    public bool TryReset()
    {
        Buffer.Clear();
        Results.Clear();
        return true;
    }
}

// Custom ObjectPool
public class ObjectPool<T>(Func<T> objectGenerator, int maxSize = 100) where T : class
{
    private readonly ConcurrentQueue<T> _objects = new();

    public T Rent()
    {
        if (_objects.TryDequeue(out var item)) return item;
        return objectGenerator();
    }

    public void Return(T item)
    {
        if (_objects.Count < maxSize)
            _objects.Enqueue(item);
        // else: let GC collect it
    }
}

// PooledObject: RAII wrapper
public sealed class PooledObject<T>(ObjectPool<T> pool, T value) : IDisposable where T : class
{
    public T Value { get; } = value;
    private bool _returned;

    public void Dispose()
    {
        if (_returned) return;
        _returned = true;
        pool.Return(Value);
    }
}

// Usage
public class StringBuilderPool
{
    private static readonly ObjectPool<StringBuilder> _pool = new(
        () => new StringBuilder(256), maxSize: 50);

    public static PooledObject<StringBuilder> Rent()
    {
        var sb = _pool.Rent();
        sb.Clear();
        return new PooledObject<StringBuilder>(_pool, sb);
    }
}

// Usage
using var pooled = StringBuilderPool.Rent();
pooled.Value.Append("Hello, ").Append("World!");
var result = pooled.Value.ToString();
// Returns to pool on dispose
```

---

## Step 1163: Weak References

```csharp
// WeakReference: GC can collect even if reference exists
public class WeakCache<TKey, TValue> where TKey : notnull where TValue : class
{
    private readonly Dictionary<TKey, WeakReference<TValue>> _cache = [];
    private readonly ReaderWriterLockSlim _lock = new();

    public bool TryGet(TKey key, out TValue? value)
    {
        _lock.EnterReadLock();
        try
        {
            if (_cache.TryGetValue(key, out var wr))
                return wr.TryGetTarget(out value);

            value = null;
            return false;
        }
        finally { _lock.ExitReadLock(); }
    }

    public void Set(TKey key, TValue value)
    {
        _lock.EnterWriteLock();
        try { _cache[key] = new WeakReference<TValue>(value); }
        finally { _lock.ExitWriteLock(); }
    }

    // Clean up dead references
    public void Purge()
    {
        _lock.EnterWriteLock();
        try
        {
            var deadKeys = _cache
                .Where(kv => !kv.Value.TryGetTarget(out _))
                .Select(kv => kv.Key)
                .ToList();

            foreach (var key in deadKeys)
                _cache.Remove(key);
        }
        finally { _lock.ExitWriteLock(); }
    }
}
```

---

## Step 1164: P/Invoke (Platform Invocation)

```csharp
// Call native Windows/Linux APIs
public static class NativeApi
{
    // Windows: GetSystemInfo
    [StructLayout(LayoutKind.Sequential)]
    public struct SYSTEM_INFO
    {
        public uint ProcessorArchitecture;
        public uint PageSize;
        public nint MinimumApplicationAddress;
        public nint MaximumApplicationAddress;
        public nint ActiveProcessorMask;
        public uint NumberOfProcessors;
        public uint ProcessorType;
        public uint AllocationGranularity;
        public ushort ProcessorLevel;
        public ushort ProcessorRevision;
    }

    [DllImport("kernel32.dll")]
    private static extern void GetSystemInfo(out SYSTEM_INFO info);

    // Linux: getpagesize
    [DllImport("libc", EntryPoint = "getpagesize")]
    private static extern int GetPageSize();

    public static int GetSystemPageSize()
    {
        if (OperatingSystem.IsWindows())
        {
            GetSystemInfo(out var info);
            return (int)info.PageSize;
        }
        if (OperatingSystem.IsLinux() || OperatingSystem.IsMacOS())
            return GetPageSize();

        return Environment.SystemPageSize;
    }
}

// LibraryImport (.NET 7+, source-generated, better performance)
public static partial class ModernNativeApi
{
    [LibraryImport("kernel32.dll", SetLastError = true)]
    [return: MarshalAs(UnmanagedType.Bool)]
    public static partial bool GetSystemInfo(out NativeApi.SYSTEM_INFO info);
}
```

---

## Step 1165: Custom Memory Manager

```csharp
// Custom MemoryManager<T>
public class UnmanagedMemoryManager<T> : MemoryManager<T> where T : unmanaged
{
    private unsafe readonly T* _ptr;
    private readonly int _length;
    private bool _disposed;

    public unsafe UnmanagedMemoryManager(T* ptr, int length)
    {
        _ptr = ptr;
        _length = length;
    }

    public override unsafe Span<T> GetSpan() => new(_ptr, _length);

    public override unsafe MemoryHandle Pin(int elementIndex = 0)
    {
        // Already pinned (unmanaged memory)
        return new MemoryHandle(_ptr + elementIndex);
    }

    public override void Unpin() { }

    protected override void Dispose(bool disposing)
    {
        _disposed = true;
    }
}

// Usage
unsafe
{
    var nativePtr = (int*)NativeMemory.Alloc((nuint)(100 * sizeof(int)));
    try
    {
        using var manager = new UnmanagedMemoryManager<int>(nativePtr, 100);
        Memory<int> memory = manager.Memory;

        memory.Span.Fill(42);
        Console.WriteLine(memory.Span[0]); // 42
    }
    finally
    {
        NativeMemory.Free(nativePtr);
    }
}
```

---

## Step 1166: String Interning และ String Memory

```csharp
// String interning: reuse identical strings
public class StringMemoryOptimizer
{
    // String.Intern: ทำให้ identical strings ใช้ memory เดียวกัน
    public static string InternString(string value) => string.Intern(value);

    // StringPool (custom): สำหรับ short-lived strings
    private readonly ConcurrentDictionary<string, string> _pool = new();

    public string Pool(string value)
        => _pool.GetOrAdd(value, v => v);

    // Avoid string allocations with StringBuilder.AppendInterpolatedStringHandler
    // .NET 10 / C# 13 ใช้ InterpolatedStringHandler เพื่อ zero-allocation formatting
    public static string FormatOrder(int id, string status, decimal amount)
    {
        // ✅ Efficient: ใช้ string.Create
        return string.Create(CultureInfo.InvariantCulture,
            $"Order #{id}: {status} - {amount:C}");
    }

    // Span-based string parsing ไม่ allocate
    public static bool TryParseKeyValue(
        ReadOnlySpan<char> line,
        out ReadOnlySpan<char> key,
        out ReadOnlySpan<char> value)
    {
        var idx = line.IndexOf('=');
        if (idx < 0)
        {
            key = value = default;
            return false;
        }
        key = line[..idx].Trim();
        value = line[(idx + 1)..].Trim();
        return true;
    }
}
```

---

## Step 1167: Finalizers และ SafeHandle

```csharp
// SafeHandle: safer way to manage native handles
public class SafeFileHandle : SafeHandle
{
    private SafeFileHandle() : base(IntPtr.Zero, ownsHandle: true) { }

    public override bool IsInvalid => handle == IntPtr.Zero || handle == new IntPtr(-1);

    protected override bool ReleaseHandle()
    {
        // Native close
        return CloseHandle(handle);
    }

    [DllImport("kernel32.dll", SetLastError = true)]
    [return: MarshalAs(UnmanagedType.Bool)]
    private static extern bool CloseHandle(IntPtr hObject);
}

// CriticalFinalizerObject: finalizer runs even during AppDomain unload
public abstract class CriticalResource : CriticalFinalizerObject, IDisposable
{
    private bool _disposed;

    ~CriticalResource()
    {
        // Called even during catastrophic failures
        if (!_disposed)
            ReleaseNativeResource();
    }

    protected abstract void ReleaseNativeResource();

    public void Dispose()
    {
        if (_disposed) return;
        _disposed = true;
        ReleaseNativeResource();
        GC.SuppressFinalize(this);
    }
}
```

---

## Step 1168: Memory Diagnostics

```csharp
// Memory diagnostic tools
public class MemoryDiagnostics
{
    public static MemoryReport GetReport()
    {
        var process = Process.GetCurrentProcess();
        var gcInfo = GC.GetGCMemoryInfo();

        return new MemoryReport(
            WorkingSetMb: process.WorkingSet64 / 1024 / 1024,
            PrivateMemoryMb: process.PrivateMemorySize64 / 1024 / 1024,
            ManagedHeapMb: GC.GetTotalMemory(false) / 1024 / 1024,
            HeapSizeMb: gcInfo.HeapSizeBytes / 1024 / 1024,
            Gen0Collections: GC.CollectionCount(0),
            Gen1Collections: GC.CollectionCount(1),
            Gen2Collections: GC.CollectionCount(2),
            FragmentedMb: gcInfo.FragmentedBytes / 1024 / 1024);
    }

    // Track allocation before/after operation
    public static async Task<AllocationReport> MeasureAllocationsAsync(Func<Task> action)
    {
        GC.Collect(GC.MaxGeneration, GCCollectionMode.Forced, true);
        var before = GC.GetTotalAllocatedBytes(precise: false);
        var gen0Before = GC.CollectionCount(0);

        await action();

        GC.Collect(0, GCCollectionMode.Optimized);
        var after = GC.GetTotalAllocatedBytes(precise: false);
        var gen0After = GC.CollectionCount(0);

        return new AllocationReport(
            AllocatedBytes: after - before,
            Gen0Collections: gen0After - gen0Before);
    }
}

public record MemoryReport(
    long WorkingSetMb, long PrivateMemoryMb, long ManagedHeapMb,
    long HeapSizeMb, int Gen0Collections, int Gen1Collections, int Gen2Collections,
    long FragmentedMb);

public record AllocationReport(long AllocatedBytes, int Gen0Collections);
```

---

## Step 1169: Zero-Copy Buffer Management

```csharp
// RecyclableMemoryStream: reuse MemoryStream buffers
// NuGet: Microsoft.IO.RecyclableMemoryStream
public class StreamProcessor(RecyclableMemoryStreamManager memoryManager)
{
    public async Task<byte[]> CompressAsync(byte[] data, CancellationToken ct)
    {
        // Use recyclable stream instead of new MemoryStream()
        using var compressed = memoryManager.GetStream();
        using (var gzip = new GZipStream(compressed, CompressionLevel.Fastest, leaveOpen: true))
        {
            await gzip.WriteAsync(data, ct);
        }
        return compressed.ToArray();
    }

    public async Task<string> SerializeAsync<T>(T value, CancellationToken ct)
    {
        using var stream = memoryManager.GetStream();
        await JsonSerializer.SerializeAsync(stream, value, cancellationToken: ct);

        stream.Position = 0;
        using var reader = new StreamReader(stream);
        return await reader.ReadToEndAsync(ct);
    }
}

// PipeWriter/PipeReader: zero-copy I/O
public class PipelineExample
{
    public static async Task ProcessStreamAsync(Stream input, Stream output, CancellationToken ct)
    {
        var pipe = new Pipe();

        var reading = FillPipeAsync(input, pipe.Writer, ct);
        var writing = ReadFromPipeAsync(pipe.Reader, output, ct);

        await Task.WhenAll(reading, writing);
    }

    private static async Task FillPipeAsync(Stream input, PipeWriter writer, CancellationToken ct)
    {
        const int minimumBufferSize = 512;
        while (true)
        {
            var memory = writer.GetMemory(minimumBufferSize);
            int bytesRead = await input.ReadAsync(memory, ct);
            if (bytesRead == 0) break;

            writer.Advance(bytesRead);
            var result = await writer.FlushAsync(ct);
            if (result.IsCompleted) break;
        }
        await writer.CompleteAsync();
    }

    private static async Task ReadFromPipeAsync(PipeReader reader, Stream output, CancellationToken ct)
    {
        while (true)
        {
            var result = await reader.ReadAsync(ct);
            var buffer = result.Buffer;

            foreach (var segment in buffer)
                await output.WriteAsync(segment, ct);

            reader.AdvanceTo(buffer.End);
            if (result.IsCompleted) break;
        }
        await reader.CompleteAsync();
    }
}
```

---

## Step 1170: Benchmark Memory Patterns

```csharp
[MemoryDiagnoser]
[HideColumns("Job", "RatioSD")]
public class MemoryBenchmarks
{
    private readonly byte[] _data = new byte[4096];
    private static readonly RecyclableMemoryStreamManager _memManager = new();

    [Benchmark(Baseline = true)]
    public async Task<byte[]> WithNewMemoryStream()
    {
        using var ms = new MemoryStream();
        await ms.WriteAsync(_data);
        return ms.ToArray();
    }

    [Benchmark]
    public async Task<byte[]> WithRecyclableMemoryStream()
    {
        using var ms = _memManager.GetStream();
        await ms.WriteAsync(_data);
        return ms.ToArray();
    }

    [Benchmark]
    public byte[] WithArrayPool()
    {
        var buffer = ArrayPool<byte>.Shared.Rent(_data.Length);
        try
        {
            _data.CopyTo(buffer, 0);
            return buffer[.._data.Length];
        }
        finally
        {
            ArrayPool<byte>.Shared.Return(buffer);
        }
    }

    [Benchmark]
    public unsafe long SumWithPointer()
    {
        fixed (byte* ptr = _data)
        {
            long sum = 0;
            byte* end = ptr + _data.Length;
            for (byte* p = ptr; p < end; p++)
                sum += *p;
            return sum;
        }
    }

    [Benchmark]
    public long SumWithSpan()
    {
        long sum = 0;
        foreach (var b in _data.AsSpan())
            sum += b;
        return sum;
    }
}
```

---

## Step 1171: Memory Leak Detection

```csharp
// Track allocations to detect leaks
public class AllocationTracker : IDisposable
{
    private static volatile int _activeCount;
    public static int ActiveCount => _activeCount;

    public AllocationTracker()
    {
        Interlocked.Increment(ref _activeCount);
        Console.WriteLine($"Allocated #{_activeCount}");
    }

    public void Dispose()
    {
        Interlocked.Decrement(ref _activeCount);
        GC.SuppressFinalize(this);
    }

    ~AllocationTracker()
    {
        // Finalizer called = Dispose was NOT called = potential leak
        Console.WriteLine($"WARNING: {GetType().Name} was not disposed! Active: {_activeCount}");
        Interlocked.Decrement(ref _activeCount);
    }
}

// EventSource for ETW events (can track with PerfView/dotnet-trace)
[EventSource(Name = "MyApp-Memory")]
public class MemoryEventSource : EventSource
{
    public static readonly MemoryEventSource Log = new();

    [Event(1, Level = EventLevel.Informational)]
    public void LargeAllocation(int bytes, string source)
        => WriteEvent(1, bytes, source);

    [Event(2, Level = EventLevel.Warning)]
    public void PoolExhausted(string poolName, int size)
        => WriteEvent(2, poolName, size);
}
```

---

## Step 1172-1175: Advanced Unsafe Patterns

```csharp
// Unsafe fixed-size struct for protocol parsing
[StructLayout(LayoutKind.Explicit, Size = 20)]
public struct TcpHeader
{
    [FieldOffset(0)] public ushort SourcePort;
    [FieldOffset(2)] public ushort DestinationPort;
    [FieldOffset(4)] public uint SequenceNumber;
    [FieldOffset(8)] public uint AcknowledgmentNumber;
    [FieldOffset(12)] public byte DataOffset;
    [FieldOffset(13)] public byte Flags;
    [FieldOffset(14)] public ushort WindowSize;
    [FieldOffset(16)] public ushort Checksum;
    [FieldOffset(18)] public ushort UrgentPointer;
}

// Parse from raw bytes
public static ref readonly TcpHeader ParseTcpHeader(ReadOnlySpan<byte> packet)
{
    if (packet.Length < 20) throw new ArgumentException("Packet too short");
    return ref MemoryMarshal.AsRef<TcpHeader>(packet[..20]);
}

// Union type using LayoutKind.Explicit
[StructLayout(LayoutKind.Explicit)]
public struct FloatIntUnion
{
    [FieldOffset(0)] public float FloatValue;
    [FieldOffset(0)] public int IntValue;
    [FieldOffset(0)] public uint UintValue;
}

// Fast type checking
public static bool IsNaN_Fast(float f)
{
    var u = new FloatIntUnion { FloatValue = f };
    // NaN: exponent all 1s and mantissa != 0
    return (u.UintValue & 0x7FFFFFFF) > 0x7F800000;
}
```

---

## Step 1176-1180: สรุป Memory Management

```csharp
// Memory optimization checklist
public static class MemoryOptimizationChecklist
{
    /*
     * ✅ Stack Allocation (fastest):
     * - Value types (struct) on stack
     * - stackalloc for small arrays (<= ~256 bytes)
     * - ref struct (Span<T>, FastParser)
     *
     * ✅ Pool-based Allocation:
     * - ArrayPool<byte>.Shared for buffers
     * - ObjectPool<T> for expensive objects
     * - RecyclableMemoryStream for streams
     * - StringBuilder pooling
     *
     * ✅ Zero-Copy Operations:
     * - Span<T>/Memory<T> for slicing
     * - MemoryMarshal.Cast for reinterpret
     * - PipeWriter/PipeReader for streaming I/O
     * - System.IO.Pipelines
     *
     * ✅ Reduce GC Pressure:
     * - Avoid LINQ in hot paths (allocates)
     * - Use struct instead of class for small objects
     * - string.Create() instead of interpolation in hot paths
     * - Minimize closures and lambda captures
     *
     * ✅ Native Memory:
     * - NativeMemory.Alloc for large unmanaged buffers
     * - P/Invoke with SafeHandle
     * - IMemoryOwner<T> for ownership transfer
     *
     * ✅ GC Tuning:
     * - Server GC for server apps
     * - GCSettings.LatencyMode for real-time
     * - GC.AddMemoryPressure for native allocations
     *
     * Measurement Tools:
     * - BenchmarkDotNet with MemoryDiagnoser
     * - dotnet-counters (System.Runtime metrics)
     * - dotnet-trace (ETW events)
     * - PerfView (detailed GC/memory analysis)
     * - JetBrains dotMemory
     */
}
```

---

*Part 40 ครอบคลุม Memory Management & Unsafe Code ทั้งหมด 30 Steps (1151-1180)*  
*ต่อไป Part 41: Native AOT & Ahead-of-Time Compilation*
