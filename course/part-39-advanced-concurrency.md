# Part 39: Advanced Concurrency & Parallelism

## Steps 1121-1150 | ระดับโลก (World-Class)

---

## Step 1121: Concurrency vs Parallelism

```
Concurrency (การทำงานแบบสลับกัน):
Thread 1: ─── Task A ─── Task B ─── Task A ───
Thread 2: ─── Task C ─── Task A ─── Task C ───
(1 CPU, สลับ context)

Parallelism (การทำงานพร้อมกัน):
Thread 1: ─── Task A ─────────────────────────
Thread 2: ─── Task B ─────────────────────────
Thread 3: ─── Task C ─────────────────────────
Thread 4: ─── Task D ─────────────────────────
(หลาย CPU cores, ทำงานพร้อมกันจริง)

async/await = Concurrency (ไม่บล็อก thread)
Parallel.For = Parallelism (หลาย threads พร้อมกัน)
```

---

## Step 1122: Thread Pool และ Task Scheduler

```csharp
// ดู thread pool status
ThreadPool.GetMinThreads(out int workerMin, out int ioMin);
ThreadPool.GetMaxThreads(out int workerMax, out int ioMax);
ThreadPool.GetAvailableThreads(out int workerAvail, out int ioAvail);

Console.WriteLine($"Worker: {workerAvail}/{workerMax} available (min: {workerMin})");
Console.WriteLine($"IO: {ioAvail}/{ioMax} available (min: {ioMin})");

// ปรับ thread pool (ระวัง - ทำใน startup เท่านั้น)
ThreadPool.SetMinThreads(16, 16);

// Custom TaskScheduler
public class PriorityTaskScheduler : TaskScheduler
{
    private readonly PriorityQueue<Task, int> _queue = new();

    protected override void QueueTask(Task task)
    {
        lock (_queue)
        {
            // Low number = high priority
            _queue.Enqueue(task, 0);
            if (_queue.Count == 1) TryExecuteTaskInline(task, false);
        }
    }

    protected override bool TryExecuteTaskInline(Task task, bool taskWasPreviouslyQueued)
        => TryExecuteTask(task);

    protected override IEnumerable<Task> GetScheduledTasks()
    {
        lock (_queue) return _queue.UnorderedItems.Select(i => i.Element);
    }
}
```

---

## Step 1123: async/await Deep Dive

```csharp
// การทำงานของ async/await
public async Task<string> FetchDataAsync(HttpClient client, string url)
{
    // State machine ถูกสร้างโดย compiler
    // await → suspend method, return thread to pool
    var response = await client.GetAsync(url);           // suspend 1
    var content = await response.Content.ReadAsStringAsync(); // suspend 2
    return content;
}

// Equivalent state machine (simplified)
public class FetchDataStateMachine : IAsyncStateMachine
{
    public int _state = -1;
    public AsyncTaskMethodBuilder<string> _builder;
    private HttpClient _client;
    private string _url;
    private Task<HttpResponseMessage>? _task1;
    private Task<string>? _task2;

    public void MoveNext()
    {
        switch (_state)
        {
            case -1:  // initial
                _task1 = _client.GetAsync(_url);
                if (_task1.IsCompleted) goto case 0;
                _state = 0;
                _builder.AwaitUnsafeOnCompleted(ref _task1, ref this);
                return;

            case 0:   // after GetAsync
                var response = _task1!.Result;
                _task2 = response.Content.ReadAsStringAsync();
                if (_task2.IsCompleted) goto case 1;
                _state = 1;
                _builder.AwaitUnsafeOnCompleted(ref _task2, ref this);
                return;

            case 1:   // after ReadAsStringAsync
                var result = _task2!.Result;
                _state = -2;  // completed
                _builder.SetResult(result);
                return;
        }
    }

    public void SetStateMachine(IAsyncStateMachine stateMachine) { }
}
```

---

## Step 1124: ConfigureAwait

```csharp
// ❌ ปัญหา: deadlock ใน UI/ASP.NET classic
public string GetDataBad()
{
    return GetDataAsync().Result;  // บล็อก + รอ context
}

public async Task<string> GetDataAsync()
{
    return await httpClient.GetStringAsync(url);
    // SynchronizationContext.Current จะ resume บน original thread
    // แต่ GetDataBad() กำลังบล็อก thread นั้น → deadlock!
}

// ✅ ใน library code: ใช้ ConfigureAwait(false)
public async Task<string> GetDataLibraryAsync(HttpClient client, string url)
{
    var response = await client.GetAsync(url).ConfigureAwait(false);
    var content = await response.Content.ReadAsStringAsync().ConfigureAwait(false);
    return content;
    // ไม่ต้องการ original SynchronizationContext
}

// ✅ ใน ASP.NET Core: ไม่จำเป็นต้องใช้ ConfigureAwait(false)
// เพราะ ASP.NET Core ไม่มี SynchronizationContext
public async Task<IActionResult> GetAction()
{
    var data = await service.GetDataAsync(); // OK ใน ASP.NET Core
    return Ok(data);
}
```

---

## Step 1125: CancellationToken Best Practices

```csharp
// ส่ง CancellationToken ทุกที่ที่เป็นไปได้
public async Task<IReadOnlyList<Product>> GetProductsAsync(
    string category,
    CancellationToken ct = default)
{
    // Database query
    var products = await db.Products
        .Where(p => p.Category == category)
        .ToListAsync(ct);  // ใช้ ct

    // HTTP call
    var enriched = await enrichmentService.EnrichAsync(products, ct);

    return enriched;
}

// Linked CancellationToken: หยุดเมื่อ request ถูก cancel หรือ timeout
public async Task<string> FetchWithTimeoutAsync(
    string url,
    TimeSpan timeout,
    CancellationToken requestCt)
{
    using var cts = CancellationTokenSource.CreateLinkedTokenSource(requestCt);
    cts.CancelAfter(timeout);

    try
    {
        return await httpClient.GetStringAsync(url, cts.Token);
    }
    catch (OperationCanceledException) when (!requestCt.IsCancellationRequested)
    {
        throw new TimeoutException($"Request to {url} timed out after {timeout}");
    }
}

// CancellationToken.Register: cleanup เมื่อ cancel
public async Task LongRunningAsync(CancellationToken ct)
{
    using var cleanup = ct.Register(() =>
        Console.WriteLine("Cancellation requested, cleaning up..."));

    while (!ct.IsCancellationRequested)
    {
        await DoUnitOfWorkAsync(ct);
    }
}
```

---

## Step 1126: Parallel.ForEachAsync (.NET 6+)

```csharp
// Parallel.ForEachAsync สำหรับ async operations
public async Task ProcessOrdersAsync(
    IReadOnlyList<Order> orders,
    CancellationToken ct)
{
    await Parallel.ForEachAsync(
        orders,
        new ParallelOptions
        {
            MaxDegreeOfParallelism = Environment.ProcessorCount,
            CancellationToken = ct
        },
        async (order, ct) =>
        {
            await ProcessSingleOrderAsync(order, ct);
        });
}

// ควบคุม concurrency ด้วย SemaphoreSlim
public async Task ProcessWithRateLimitAsync(
    IReadOnlyList<string> urls,
    int maxConcurrency = 5)
{
    using var semaphore = new SemaphoreSlim(maxConcurrency);

    var tasks = urls.Select(async url =>
    {
        await semaphore.WaitAsync();
        try
        {
            return await httpClient.GetStringAsync(url);
        }
        finally
        {
            semaphore.Release();
        }
    });

    var results = await Task.WhenAll(tasks);
}
```

---

## Step 1127: Channel<T> - Producer/Consumer

```csharp
// System.Threading.Channels
public class OrderProcessingPipeline
{
    private readonly Channel<Order> _channel;
    private readonly CancellationToken _ct;

    public OrderProcessingPipeline(CancellationToken ct)
    {
        _ct = ct;
        _channel = Channel.CreateBounded<Order>(new BoundedChannelOptions(100)
        {
            FullMode = BoundedChannelFullMode.Wait,
            SingleReader = false,
            SingleWriter = false
        });
    }

    // Producer: ส่ง orders เข้า channel
    public async Task ProduceAsync(IAsyncEnumerable<Order> orders)
    {
        var writer = _channel.Writer;
        try
        {
            await foreach (var order in orders.WithCancellation(_ct))
            {
                await writer.WriteAsync(order, _ct);
            }
        }
        finally
        {
            writer.Complete();
        }
    }

    // Consumer: ประมวลผล orders จาก channel
    public async Task ConsumeAsync(int workerCount = 4)
    {
        var reader = _channel.Reader;
        var workers = Enumerable.Range(0, workerCount)
            .Select(_ => Task.Run(async () =>
            {
                await foreach (var order in reader.ReadAllAsync(_ct))
                {
                    await ProcessOrderAsync(order);
                }
            }));

        await Task.WhenAll(workers);
    }

    private static Task ProcessOrderAsync(Order order)
        => Task.Delay(10); // simulate work
}

// Usage
var pipeline = new OrderProcessingPipeline(ct);
var producer = pipeline.ProduceAsync(GetOrdersAsync());
var consumer = pipeline.ConsumeAsync(workerCount: 8);
await Task.WhenAll(producer, consumer);
```

---

## Step 1128: Dataflow (TPL Dataflow)

```xml
<PackageReference Include="System.Threading.Tasks.Dataflow" Version="8.0.0" />
```

```csharp
// TPL Dataflow: pipeline processing
public class ImageProcessingPipeline
{
    public static async Task ProcessImagesAsync(
        IEnumerable<string> imagePaths,
        CancellationToken ct)
    {
        // Step 1: Load image
        var loadBlock = new TransformBlock<string, (string Path, byte[] Data)>(
            async path =>
            {
                var data = await File.ReadAllBytesAsync(path, ct);
                return (path, data);
            },
            new ExecutionDataflowBlockOptions
            {
                MaxDegreeOfParallelism = 2,
                CancellationToken = ct
            });

        // Step 2: Resize
        var resizeBlock = new TransformBlock<(string Path, byte[] Data), (string Path, byte[] Resized)>(
            item =>
            {
                var resized = ResizeImage(item.Data, 800, 600);
                return (item.Path, resized);
            },
            new ExecutionDataflowBlockOptions
            {
                MaxDegreeOfParallelism = Environment.ProcessorCount,
                CancellationToken = ct
            });

        // Step 3: Save
        var saveBlock = new ActionBlock<(string Path, byte[] Resized)>(
            async item =>
            {
                var outputPath = Path.ChangeExtension(item.Path, ".resized.jpg");
                await File.WriteAllBytesAsync(outputPath, item.Resized, ct);
            },
            new ExecutionDataflowBlockOptions
            {
                MaxDegreeOfParallelism = 4,
                CancellationToken = ct
            });

        // Link blocks
        loadBlock.LinkTo(resizeBlock, new DataflowLinkOptions { PropagateCompletion = true });
        resizeBlock.LinkTo(saveBlock, new DataflowLinkOptions { PropagateCompletion = true });

        // Post all images
        foreach (var path in imagePaths)
            await loadBlock.SendAsync(path, ct);

        loadBlock.Complete();
        await saveBlock.Completion;
    }

    private static byte[] ResizeImage(byte[] data, int width, int height)
        => data; // simplified
}
```

---

## Step 1129: IAsyncEnumerable - Streaming

```csharp
// Async streaming data
public async IAsyncEnumerable<Order> StreamOrdersAsync(
    [EnumeratorCancellation] CancellationToken ct = default)
{
    await foreach (var order in db.Orders
        .Where(o => o.Status == OrderStatus.Pending)
        .AsAsyncEnumerable()
        .WithCancellation(ct))
    {
        yield return order;
    }
}

// Consumer
await foreach (var order in StreamOrdersAsync(ct))
{
    await ProcessOrderAsync(order, ct);
}

// Batch processing with IAsyncEnumerable
public static async IAsyncEnumerable<IReadOnlyList<T>> BatchAsync<T>(
    this IAsyncEnumerable<T> source,
    int batchSize,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    var batch = new List<T>(batchSize);

    await foreach (var item in source.WithCancellation(ct))
    {
        batch.Add(item);
        if (batch.Count >= batchSize)
        {
            yield return batch.AsReadOnly();
            batch = new List<T>(batchSize);
        }
    }

    if (batch.Count > 0)
        yield return batch.AsReadOnly();
}

// Usage: process in batches of 100
await foreach (var batch in StreamOrdersAsync().BatchAsync(100, ct))
{
    await ProcessBatchAsync(batch, ct);
}
```

---

## Step 1130: ValueTask vs Task

```csharp
// Task: ใช้สำหรับ long-running async operations
public async Task<string> FetchDataAsync(string url)
{
    return await httpClient.GetStringAsync(url);
}

// ValueTask: ใช้เมื่อ operation มักจะ complete synchronously
// เช่น cache hit ที่พบบ่อย
public ValueTask<Product?> GetProductAsync(int id, CancellationToken ct = default)
{
    // Hot path: cache hit (synchronous)
    if (_cache.TryGetValue(id, out var cached))
        return ValueTask.FromResult<Product?>(cached);

    // Cold path: database (asynchronous)
    return new ValueTask<Product?>(LoadFromDatabaseAsync(id, ct));
}

private async Task<Product?> LoadFromDatabaseAsync(int id, CancellationToken ct)
{
    var product = await db.Products.FindAsync([id], ct);
    if (product is not null)
        _cache.TryAdd(id, product);
    return product;
}

// ⚠️ ข้อควรระวัง: ValueTask ไม่ควร await หลายครั้ง
var vt = GetProductAsync(1);
var p1 = await vt;   // OK
// var p2 = await vt; // ❌ ไม่ควร await ValueTask ซ้ำ

// ✅ ถ้าต้องการ await ซ้ำ ให้แปลงเป็น Task ก่อน
var task = GetProductAsync(1).AsTask();
var p1 = await task;
var p2 = await task;  // OK
```

---

## Step 1131: Interlocked Operations (Lock-Free)

```csharp
// Interlocked: atomic operations ไม่ต้องใช้ lock
public class LockFreeCounter
{
    private long _count;
    private long _errors;

    public long Count => Interlocked.Read(ref _count);
    public long Errors => Interlocked.Read(ref _errors);

    public void Increment() => Interlocked.Increment(ref _count);
    public void Decrement() => Interlocked.Decrement(ref _count);
    public void AddError() => Interlocked.Increment(ref _errors);

    // Compare-And-Swap (CAS)
    public bool TryUpdate(long expected, long newValue)
        => Interlocked.CompareExchange(ref _count, newValue, expected) == expected;
}

// Lock-free stack using CAS
public class LockFreeStack<T>
{
    private volatile StackNode<T>? _head;

    public void Push(T value)
    {
        var node = new StackNode<T>(value);
        while (true)
        {
            node.Next = _head;
            if (Interlocked.CompareExchange(ref _head, node, node.Next) == node.Next)
                return;
            // Retry on conflict
        }
    }

    public bool TryPop(out T? value)
    {
        while (true)
        {
            var head = _head;
            if (head is null) { value = default; return false; }

            if (Interlocked.CompareExchange(ref _head, head.Next, head) == head)
            {
                value = head.Value;
                return true;
            }
            // Retry on conflict
        }
    }

    private class StackNode<TItem>(TItem value)
    {
        public TItem Value { get; } = value;
        public StackNode<TItem>? Next;
    }
}
```

---

## Step 1132: ReaderWriterLockSlim

```csharp
// Multiple readers OR single writer
public class ThreadSafeCache<TKey, TValue> where TKey : notnull
{
    private readonly Dictionary<TKey, TValue> _cache = [];
    private readonly ReaderWriterLockSlim _lock = new(LockRecursionPolicy.NoRecursion);

    public bool TryGet(TKey key, out TValue? value)
    {
        _lock.EnterReadLock();
        try
        {
            return _cache.TryGetValue(key, out value);
        }
        finally
        {
            _lock.ExitReadLock();
        }
    }

    public void Set(TKey key, TValue value)
    {
        _lock.EnterWriteLock();
        try
        {
            _cache[key] = value;
        }
        finally
        {
            _lock.ExitWriteLock();
        }
    }

    public TValue GetOrAdd(TKey key, Func<TKey, TValue> factory)
    {
        // Optimistic read first
        _lock.EnterUpgradeableReadLock();
        try
        {
            if (_cache.TryGetValue(key, out var existing)) return existing;

            _lock.EnterWriteLock();
            try
            {
                // Double-check after acquiring write lock
                if (_cache.TryGetValue(key, out existing)) return existing;

                var value = factory(key);
                _cache[key] = value;
                return value;
            }
            finally
            {
                _lock.ExitWriteLock();
            }
        }
        finally
        {
            _lock.ExitUpgradeableReadLock();
        }
    }

    public void Dispose() => _lock.Dispose();
}
```

---

## Step 1133: SemaphoreSlim - Async Rate Limiting

```csharp
// SemaphoreSlim สำหรับ async rate limiting
public class RateLimitedApiClient(HttpClient client, int maxConcurrent = 10)
{
    private readonly SemaphoreSlim _semaphore = new(maxConcurrent, maxConcurrent);

    public async Task<T> GetAsync<T>(string url, CancellationToken ct = default)
    {
        await _semaphore.WaitAsync(ct);
        try
        {
            var response = await client.GetAsync(url, ct);
            response.EnsureSuccessStatusCode();
            return await response.Content.ReadFromJsonAsync<T>(ct)
                ?? throw new InvalidOperationException("Null response");
        }
        finally
        {
            _semaphore.Release();
        }
    }

    // Batch requests with rate limiting
    public async Task<IReadOnlyList<T>> GetManyAsync<T>(
        IReadOnlyList<string> urls,
        CancellationToken ct = default)
    {
        var tasks = urls.Select(url => GetAsync<T>(url, ct));
        return await Task.WhenAll(tasks);
    }
}
```

---

## Step 1134: Parallel LINQ (PLINQ)

```csharp
// PLINQ: parallel LINQ queries
public class DataProcessor
{
    // Sequential
    public IEnumerable<ProcessedItem> ProcessSequential(IEnumerable<RawItem> items)
        => items.Select(ProcessItem);

    // Parallel (CPU-bound work)
    public IEnumerable<ProcessedItem> ProcessParallel(IEnumerable<RawItem> items)
        => items.AsParallel()
            .WithDegreeOfParallelism(Environment.ProcessorCount)
            .WithExecutionMode(ParallelExecutionMode.ForceParallelism)
            .Select(ProcessItem)
            .ToList();

    // Parallel with ordering preserved
    public IEnumerable<ProcessedItem> ProcessParallelOrdered(IEnumerable<RawItem> items)
        => items.AsParallel()
            .AsOrdered()
            .WithDegreeOfParallelism(4)
            .Select(ProcessItem);

    // Parallel with exception handling
    public IReadOnlyList<ProcessedItem> ProcessWithErrorHandling(IEnumerable<RawItem> items)
    {
        try
        {
            return items.AsParallel()
                .Select(ProcessItem)
                .ToList();
        }
        catch (AggregateException ae)
        {
            // PLINQ wraps exceptions in AggregateException
            ae.Handle(ex =>
            {
                Console.WriteLine($"Error: {ex.Message}");
                return true; // handled
            });
            return [];
        }
    }

    // Partition large datasets
    public void ProcessLargeDataset(IReadOnlyList<RawItem> items)
    {
        var partitioner = Partitioner.Create(items, true);
        Parallel.ForEach(partitioner, new ParallelOptions
        {
            MaxDegreeOfParallelism = Environment.ProcessorCount
        },
        item => ProcessItem(item));
    }

    private static ProcessedItem ProcessItem(RawItem item) => new(item.Id);
}

public record RawItem(int Id);
public record ProcessedItem(int Id);
```

---

## Step 1135: Concurrent Collections

```csharp
// ConcurrentDictionary
public class ProductCache
{
    private readonly ConcurrentDictionary<int, Product> _cache = new();

    public Product GetOrCreate(int id, Func<int, Product> factory)
        => _cache.GetOrAdd(id, factory);

    public void Update(int id, Product product)
    {
        _cache.AddOrUpdate(id, product, (key, existing) => product);
    }

    public bool Remove(int id) => _cache.TryRemove(id, out _);
}

// ConcurrentQueue / ConcurrentStack / ConcurrentBag
public class WorkQueue<T>
{
    private readonly ConcurrentQueue<T> _queue = new();
    private readonly ConcurrentBag<T> _results = [];

    public void Enqueue(T item) => _queue.Enqueue(item);

    public async Task ProcessAllAsync(Func<T, Task<T>> processor, CancellationToken ct)
    {
        var tasks = new List<Task>();

        while (_queue.TryDequeue(out var item))
        {
            var capturedItem = item;
            tasks.Add(Task.Run(async () =>
            {
                var result = await processor(capturedItem);
                _results.Add(result);
            }, ct));
        }

        await Task.WhenAll(tasks);
    }

    public IReadOnlyList<T> GetResults() => [.. _results];
}

// BlockingCollection: bounded producer/consumer
public class BoundedWorkQueue<T>
{
    private readonly BlockingCollection<T> _queue;

    public BoundedWorkQueue(int capacity = 100)
        => _queue = new BlockingCollection<T>(capacity);

    public void Produce(T item) => _queue.Add(item);
    public void CompleteAdding() => _queue.CompleteAdding();

    public IEnumerable<T> Consume() => _queue.GetConsumingEnumerable();

    public async Task ConsumeAsync(Func<T, Task> process, CancellationToken ct)
    {
        await Parallel.ForEachAsync(_queue.GetConsumingEnumerable(ct),
            new ParallelOptions { MaxDegreeOfParallelism = 4, CancellationToken = ct },
            async (item, ct) => await process(item));
    }
}
```

---

## Step 1136: async Synchronization Primitives

```csharp
// AsyncLock: SemaphoreSlim ที่ใช้ได้เหมือน lock
public class AsyncLock
{
    private readonly SemaphoreSlim _semaphore = new(1, 1);

    public async Task<IDisposable> LockAsync(CancellationToken ct = default)
    {
        await _semaphore.WaitAsync(ct);
        return new Releaser(_semaphore);
    }

    private class Releaser(SemaphoreSlim semaphore) : IDisposable
    {
        public void Dispose() => semaphore.Release();
    }
}

// Usage
public class AsyncSafeOperation
{
    private readonly AsyncLock _lock = new();

    public async Task UpdateAsync(CancellationToken ct)
    {
        using var _ = await _lock.LockAsync(ct);
        // Critical section - only one task at a time
        await DoUpdateAsync();
    }
}

// AsyncManualResetEvent
public class AsyncManualResetEvent(bool initialState = false)
{
    private volatile TaskCompletionSource<bool> _tcs =
        new(initialState ? true : default);

    public Task WaitAsync() => _tcs.Task;

    public void Set()
    {
        var tcs = _tcs;
        Task.Run(() => tcs.TrySetResult(true));
    }

    public void Reset()
        => Interlocked.Exchange(ref _tcs, new TaskCompletionSource<bool>());
}
```

---

## Step 1137: WhenAll, WhenAny, Task.WaitAll

```csharp
// Task.WhenAll: รอทุก tasks
public async Task<(Product Product, Customer Customer, Order[] Orders)> LoadDashboardAsync(
    Guid productId, Guid customerId, CancellationToken ct)
{
    var productTask = productRepository.GetByIdAsync(new ProductId(productId), ct);
    var customerTask = customerRepository.GetByIdAsync(new CustomerId(customerId), ct);
    var ordersTask = orderRepository.GetByCustomerAsync(new CustomerId(customerId), ct);

    // Run all 3 queries concurrently
    await Task.WhenAll(productTask, customerTask, ordersTask);

    return (
        (await productTask)!,
        (await customerTask)!,
        (IReadOnlyList<Order>)(await ordersTask)
    );
}

// Task.WhenAny: รอ task แรกที่เสร็จ (circuit breaker / fallback)
public async Task<string> GetWithFallbackAsync(string primaryUrl, string fallbackUrl, CancellationToken ct)
{
    using var cts = CancellationTokenSource.CreateLinkedTokenSource(ct);
    cts.CancelAfter(TimeSpan.FromSeconds(5)); // timeout

    var primaryTask = httpClient.GetStringAsync(primaryUrl, cts.Token);
    var fallbackTask = Task.Delay(2000, cts.Token).ContinueWith(
        _ => httpClient.GetStringAsync(fallbackUrl, ct),
        ct).Unwrap();

    var winner = await Task.WhenAny(primaryTask, fallbackTask);
    cts.Cancel(); // cancel the other

    return await winner;
}

// Collecting partial results
public async Task<IReadOnlyList<T>> WhenAllSafeAsync<T>(IEnumerable<Task<T>> tasks)
{
    var results = new List<T>();
    var exceptions = new List<Exception>();

    foreach (var task in tasks)
    {
        try { results.Add(await task); }
        catch (Exception ex) { exceptions.Add(ex); }
    }

    if (exceptions.Count > 0)
        throw new AggregateException(exceptions);

    return results;
}
```

---

## Step 1138: Timer และ Periodic Background Work

```csharp
// PeriodicTimer (.NET 6+) - best for background loops
public class MetricsCollector(IMetricsRepository repo) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromSeconds(30));

        while (await timer.WaitForNextTickAsync(ct))
        {
            try
            {
                await CollectMetricsAsync(ct);
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                Console.WriteLine($"Metrics collection failed: {ex.Message}");
            }
        }
    }

    private async Task CollectMetricsAsync(CancellationToken ct)
    {
        var metrics = GatherCurrentMetrics();
        await repo.SaveAsync(metrics, ct);
    }

    private static SystemMetrics GatherCurrentMetrics() => new(
        CpuUsage: GetCpuUsage(),
        MemoryMb: GC.GetTotalMemory(false) / 1024 / 1024,
        ThreadCount: Process.GetCurrentProcess().Threads.Count,
        Timestamp: DateTime.UtcNow);

    private static double GetCpuUsage() => 0; // simplified
}

public record SystemMetrics(double CpuUsage, long MemoryMb, int ThreadCount, DateTime Timestamp);
```

---

## Step 1139: Thread-local Storage

```csharp
// ThreadLocal<T>: per-thread values
public class RequestContext
{
    private static readonly ThreadLocal<RequestContext?> _current = new();
    private static readonly AsyncLocal<RequestContext?> _asyncCurrent = new();

    public Guid RequestId { get; init; }
    public string UserId { get; init; } = string.Empty;

    // ThreadLocal: ไม่ flow ตาม async
    public static RequestContext? CurrentThread
    {
        get => _current.Value;
        set => _current.Value = value;
    }

    // AsyncLocal: flow ตาม async context (ดีกว่าสำหรับ async code)
    public static RequestContext? Current
    {
        get => _asyncCurrent.Value;
        set => _asyncCurrent.Value = value;
    }
}

// Middleware ที่ใช้ AsyncLocal
public class RequestContextMiddleware(RequestDelegate next)
{
    public async Task InvokeAsync(HttpContext context)
    {
        RequestContext.Current = new RequestContext
        {
            RequestId = Guid.NewGuid(),
            UserId = context.User.FindFirst(ClaimTypes.NameIdentifier)?.Value ?? "anonymous"
        };

        await next(context);

        RequestContext.Current = null;
    }
}

// สามารถ access ได้ใน async methods
public async Task<Order> GetOrderAsync(Guid id)
{
    var userId = RequestContext.Current?.UserId; // flows through await!
    return await db.Orders.FindAsync([id])!;
}
```

---

## Step 1140: Parallel Matrix Operations (SIMD)

```csharp
// System.Numerics.Vector<T> สำหรับ SIMD operations
using System.Numerics;

public class VectorizedMath
{
    // SIMD-accelerated sum
    public static float SumVectorized(ReadOnlySpan<float> values)
    {
        var sum = Vector<float>.Zero;
        var vectorSize = Vector<float>.Count;
        int i;

        for (i = 0; i <= values.Length - vectorSize; i += vectorSize)
        {
            var v = new Vector<float>(values.Slice(i, vectorSize));
            sum += v;
        }

        var result = Vector.Dot(sum, Vector<float>.One);

        // Handle remaining elements
        for (; i < values.Length; i++)
            result += values[i];

        return result;
    }

    // SIMD dot product
    public static float DotProductVectorized(ReadOnlySpan<float> a, ReadOnlySpan<float> b)
    {
        if (a.Length != b.Length) throw new ArgumentException("Lengths must match");

        var sum = Vector<float>.Zero;
        var vectorSize = Vector<float>.Count;
        int i;

        for (i = 0; i <= a.Length - vectorSize; i += vectorSize)
        {
            var va = new Vector<float>(a.Slice(i, vectorSize));
            var vb = new Vector<float>(b.Slice(i, vectorSize));
            sum += va * vb;
        }

        var result = Vector.Dot(sum, Vector<float>.One);

        for (; i < a.Length; i++)
            result += a[i] * b[i];

        return result;
    }

    // SIMD element-wise multiply
    public static void MultiplyVectorized(
        ReadOnlySpan<float> a,
        ReadOnlySpan<float> b,
        Span<float> result)
    {
        var vectorSize = Vector<float>.Count;
        int i;

        for (i = 0; i <= a.Length - vectorSize; i += vectorSize)
        {
            var va = new Vector<float>(a.Slice(i, vectorSize));
            var vb = new Vector<float>(b.Slice(i, vectorSize));
            (va * vb).CopyTo(result.Slice(i, vectorSize));
        }

        for (; i < a.Length; i++)
            result[i] = a[i] * b[i];
    }
}
```

---

## Step 1141: Async Streams with Backpressure

```csharp
// Controlled backpressure with channels
public class BackpressureStream<T>
{
    private readonly Channel<T> _channel;

    public BackpressureStream(int capacity = 100)
    {
        _channel = Channel.CreateBounded<T>(new BoundedChannelOptions(capacity)
        {
            FullMode = BoundedChannelFullMode.Wait  // producer waits when full
        });
    }

    public async IAsyncEnumerable<T> ReadAsync(
        [EnumeratorCancellation] CancellationToken ct = default)
    {
        await foreach (var item in _channel.Reader.ReadAllAsync(ct))
            yield return item;
    }

    public async Task WriteAsync(T item, CancellationToken ct = default)
        => await _channel.Writer.WriteAsync(item, ct);

    public void Complete() => _channel.Writer.Complete();
}

// Usage
public async Task StreamOrdersWithBackpressureAsync(CancellationToken ct)
{
    var stream = new BackpressureStream<Order>(capacity: 50);

    // Fast producer
    var producer = Task.Run(async () =>
    {
        await foreach (var order in db.Orders.AsAsyncEnumerable())
        {
            await stream.WriteAsync(order, ct);  // will wait if consumer is slow
        }
        stream.Complete();
    }, ct);

    // Slow consumer
    var consumer = Task.Run(async () =>
    {
        await foreach (var order in stream.ReadAsync(ct))
        {
            await ProcessOrderSlowlyAsync(order, ct);
        }
    }, ct);

    await Task.WhenAll(producer, consumer);
}

private static Task ProcessOrderSlowlyAsync(Order order, CancellationToken ct)
    => Task.Delay(100, ct);
```

---

## Step 1142: Task Completion Sources

```csharp
// TaskCompletionSource: สร้าง Task ที่ complete แบบ manual
public class EventWaiter<T>
{
    private readonly Dictionary<string, TaskCompletionSource<T>> _waiters = [];
    private readonly Lock _lock = new();

    public Task<T> WaitForAsync(string key, TimeSpan timeout, CancellationToken ct = default)
    {
        var tcs = new TaskCompletionSource<T>(TaskCreationOptions.RunContinuationsAsynchronously);

        lock (_lock) _waiters[key] = tcs;

        // Setup timeout
        var timeoutCts = CancellationTokenSource.CreateLinkedTokenSource(ct);
        timeoutCts.CancelAfter(timeout);
        timeoutCts.Token.Register(() =>
        {
            lock (_lock) _waiters.Remove(key);
            tcs.TrySetCanceled(ct);
        });

        return tcs.Task;
    }

    public bool TrySignal(string key, T value)
    {
        lock (_lock)
        {
            if (!_waiters.TryGetValue(key, out var tcs)) return false;
            _waiters.Remove(key);
            tcs.TrySetResult(value);
            return true;
        }
    }
}

// Usage: polling จาก external system
public class WebhookWaiter(EventWaiter<WebhookPayload> waiter)
{
    // Endpoint ที่รับ webhook
    public void ReceiveWebhook(string orderId, WebhookPayload payload)
        => waiter.TrySignal(orderId, payload);

    // Async wait for webhook
    public async Task<WebhookPayload> WaitForWebhookAsync(
        string orderId,
        TimeSpan timeout,
        CancellationToken ct)
    {
        return await waiter.WaitForAsync(orderId, timeout, ct);
    }
}
```

---

## Step 1143: Context Propagation (Activity)

```csharp
// System.Diagnostics.Activity สำหรับ distributed tracing
public class OrderService(IOrderRepository repo)
{
    private static readonly ActivitySource _activitySource = new("OrderService");

    public async Task<Order> PlaceOrderAsync(PlaceOrderCommand cmd, CancellationToken ct)
    {
        using var activity = _activitySource.StartActivity("PlaceOrder");
        activity?.SetTag("order.customer_id", cmd.CustomerId);

        try
        {
            var order = Order.Create(new CustomerId(cmd.CustomerId),
                Address.Create("...", "...", "...", "10100"));

            using var saveActivity = _activitySource.StartActivity("SaveOrder");
            await repo.AddAsync(order, ct);

            activity?.SetTag("order.id", order.Id.Value.ToString());
            activity?.SetStatus(ActivityStatusCode.Ok);

            return order;
        }
        catch (Exception ex)
        {
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            activity?.RecordException(ex);
            throw;
        }
    }
}
```

---

## Step 1144: Benchmarking Concurrency

```csharp
// BenchmarkDotNet สำหรับ benchmark concurrency
[MemoryDiagnoser]
[ThreadingDiagnoser]
public class ConcurrencyBenchmarks
{
    private readonly int[] _data = Enumerable.Range(0, 10_000).ToArray();

    [Benchmark(Baseline = true)]
    public long Sequential()
        => _data.Sum(x => (long)x * x);

    [Benchmark]
    public long PLINQ()
        => _data.AsParallel().Sum(x => (long)x * x);

    [Benchmark]
    public long ParallelFor()
    {
        long total = 0;
        Parallel.For(0, _data.Length, () => 0L,
            (i, state, local) => local + (long)_data[i] * _data[i],
            local => Interlocked.Add(ref total, local));
        return total;
    }

    [Benchmark]
    public long Vectorized()
    {
        ReadOnlySpan<float> floats = Array.ConvertAll(_data, x => (float)x);
        return (long)VectorizedMath.DotProductVectorized(floats, floats);
    }
}
```

---

## Step 1145: Avoiding Common Concurrency Pitfalls

```csharp
// ❌ ANTI-PATTERN: async void
public async void HandleEvent(object sender, EventArgs e)
{
    await DoSomethingAsync(); // exception unhandled!
}

// ✅ async Task หรือ fire-and-forget ที่ถูกต้อง
public async Task HandleEventAsync()
{
    await DoSomethingAsync();
}

// ❌ ANTI-PATTERN: .Result / .Wait() ใน async context
public string GetDataBad()
{
    return GetDataAsync().Result; // deadlock risk
}

// ✅ async all the way
public async Task<string> GetDataGood()
{
    return await GetDataAsync();
}

// ❌ ANTI-PATTERN: Task.Run ใน async method ที่เป็น I/O bound
public async Task<string> FetchDataBad(string url)
{
    return await Task.Run(async () => await httpClient.GetStringAsync(url));
    // ไม่ได้ประโยชน์ - เสีย thread pool thread โดยไม่จำเป็น
}

// ✅ I/O-bound: ไม่ต้องใช้ Task.Run
public async Task<string> FetchDataGood(string url)
{
    return await httpClient.GetStringAsync(url);
}

// ✅ CPU-bound: ใช้ Task.Run เพื่อ offload
public async Task<string> ProcessDataGood(byte[] data)
{
    return await Task.Run(() => HeavyCpuProcessing(data));
}

// ❌ ANTI-PATTERN: lock ใน async method
public async Task UpdateAsyncBad()
{
    lock (_lockObj)
    {
        await DoAsyncWork(); // compile error! cannot await inside lock
    }
}

// ✅ SemaphoreSlim แทน lock
public async Task UpdateAsyncGood()
{
    await _semaphore.WaitAsync();
    try
    {
        await DoAsyncWork();
    }
    finally
    {
        _semaphore.Release();
    }
}
```

---

## Step 1146: Rx.NET (Reactive Extensions)

```xml
<PackageReference Include="System.Reactive" Version="6.0.1" />
```

```csharp
// IObservable: push-based data streams
public class StockPriceMonitor
{
    // สร้าง observable stream จาก events
    public IObservable<StockPrice> CreatePriceStream(IStockPriceService service)
    {
        return Observable.Create<StockPrice>(async (observer, ct) =>
        {
            await foreach (var price in service.GetPriceStreamAsync(ct))
            {
                observer.OnNext(price);
            }
            observer.OnCompleted();
        });
    }

    public IDisposable MonitorPrices(IObservable<StockPrice> priceStream)
    {
        return priceStream
            .Where(p => p.Symbol == "AAPL")
            .Throttle(TimeSpan.FromSeconds(1))   // debounce
            .Buffer(TimeSpan.FromMinutes(1))      // batch per minute
            .Select(prices => new PriceSummary(
                prices.First().Symbol,
                prices.Min(p => p.Value),
                prices.Max(p => p.Value),
                prices.Average(p => p.Value)))
            .DistinctUntilChanged(s => s.Average) // skip duplicates
            .Subscribe(
                summary => Console.WriteLine($"AAPL 1m: low={summary.Min}, high={summary.Max}, avg={summary.Average:F2}"),
                ex => Console.WriteLine($"Error: {ex.Message}"),
                () => Console.WriteLine("Stream completed"));
    }
}

public record StockPrice(string Symbol, decimal Value, DateTime Timestamp);
public record PriceSummary(string Symbol, decimal Min, decimal Max, decimal Average);
```

---

## Step 1147: Cancellation Token Source Patterns

```csharp
// Composite cancellation
public static CancellationToken CreateCompositeToken(
    CancellationToken requestToken,
    TimeSpan timeout,
    CancellationToken shutdownToken)
{
    var cts = CancellationTokenSource.CreateLinkedTokenSource(
        requestToken, shutdownToken);
    cts.CancelAfter(timeout);
    return cts.Token;
}

// Graceful shutdown
public class OrderProcessor : BackgroundService
{
    private const int _gracePeriodSeconds = 30;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await ProcessNextOrderAsync(stoppingToken);
        }
    }

    public override async Task StopAsync(CancellationToken ct)
    {
        Console.WriteLine("Shutdown requested, finishing current work...");

        // Give 30 seconds to complete current work
        using var graceCts = new CancellationTokenSource(
            TimeSpan.FromSeconds(_gracePeriodSeconds));

        await base.StopAsync(graceCts.Token);
        Console.WriteLine("Shutdown complete");
    }

    private static Task ProcessNextOrderAsync(CancellationToken ct)
        => Task.Delay(100, ct);
}
```

---

## Step 1148: Testing Concurrent Code

```csharp
// Tests สำหรับ concurrent code
public class ConcurrencyTests
{
    [Fact]
    public async Task LockFreeCounter_ConcurrentIncrements_ShouldBeAccurate()
    {
        var counter = new LockFreeCounter();
        const int iterations = 10_000;
        const int threads = 10;

        await Parallel.ForAsync(0, threads, async (_, ct) =>
        {
            await Task.Yield(); // force actual parallelism
            for (int i = 0; i < iterations; i++)
                counter.Increment();
        });

        counter.Count.Should().Be(threads * iterations);
    }

    [Fact]
    public async Task ThreadSafeCache_ConcurrentAccess_ShouldNotCorrupt()
    {
        var cache = new ThreadSafeCache<int, string>();
        var tasks = Enumerable.Range(0, 1000).Select(i =>
            Task.Run(() =>
            {
                if (i % 2 == 0)
                    cache.Set(i / 2, $"value-{i}");
                else
                    cache.TryGet(i / 2, out _);
            }));

        // Should complete without exception or corruption
        var act = async () => await Task.WhenAll(tasks);
        await act.Should().NotThrowAsync();
    }

    [Fact]
    public async Task Channel_ProducerConsumer_ShouldProcessAllItems()
    {
        var channel = Channel.CreateUnbounded<int>();
        var results = new ConcurrentBag<int>();

        var producer = Task.Run(async () =>
        {
            for (int i = 0; i < 100; i++)
                await channel.Writer.WriteAsync(i);
            channel.Writer.Complete();
        });

        var consumer = Task.Run(async () =>
        {
            await foreach (var item in channel.Reader.ReadAllAsync())
                results.Add(item * 2);
        });

        await Task.WhenAll(producer, consumer);
        results.Should().HaveCount(100);
        results.Sum().Should().Be(Enumerable.Range(0, 100).Sum() * 2);
    }
}
```

---

## Step 1149: Actor Model Pattern (Orleans)

```xml
<PackageReference Include="Microsoft.Orleans.Core" Version="8.2.0" />
<PackageReference Include="Microsoft.Orleans.Server" Version="8.2.0" />
```

```csharp
// Grain Interface
public interface IOrderGrain : IGrainWithGuidKey
{
    Task<OrderState> GetStateAsync();
    Task PlaceOrderAsync(PlaceOrderRequest request);
    Task CancelAsync(string reason);
}

// Grain Implementation
public class OrderGrain(ILogger<OrderGrain> logger) : Grain, IOrderGrain
{
    private OrderState _state = new();

    public Task<OrderState> GetStateAsync() => Task.FromResult(_state);

    public Task PlaceOrderAsync(PlaceOrderRequest request)
    {
        if (_state.Status != "None")
            throw new InvalidOperationException("Order already placed");

        _state = _state with
        {
            OrderId = this.GetPrimaryKey(),
            CustomerId = request.CustomerId,
            Status = "Placed",
            PlacedAt = DateTime.UtcNow
        };

        logger.LogInformation("Order {OrderId} placed", _state.OrderId);
        return Task.CompletedTask;
    }

    public Task CancelAsync(string reason)
    {
        if (_state.Status == "Cancelled")
            throw new InvalidOperationException("Already cancelled");

        _state = _state with { Status = "Cancelled" };
        return Task.CompletedTask;
    }
}

public record OrderState(
    Guid OrderId = default,
    Guid CustomerId = default,
    string Status = "None",
    DateTime? PlacedAt = null);
```

---

## Step 1150: สรุป Advanced Concurrency

```
Concurrency Toolbox:
┌────────────────────────────────────────────────────────┐
│  Foundation                                             │
│  - async/await (I/O bound)                             │
│  - Task.Run (CPU bound)                                │
│  - CancellationToken (cooperative cancellation)        │
│  - ConfigureAwait(false) (library code)                │
├────────────────────────────────────────────────────────┤
│  Synchronization                                        │
│  - SemaphoreSlim (async semaphore)                     │
│  - ReaderWriterLockSlim (multiple readers)             │
│  - Interlocked (atomic operations)                     │
│  - AsyncLock (async-friendly lock)                     │
├────────────────────────────────────────────────────────┤
│  Data Structures                                        │
│  - Channel<T> (producer/consumer with backpressure)    │
│  - ConcurrentDictionary, Queue, Stack                  │
│  - BlockingCollection (bounded queue)                  │
├────────────────────────────────────────────────────────┤
│  Parallel Processing                                    │
│  - Parallel.ForEachAsync (parallel I/O)                │
│  - PLINQ (parallel LINQ for CPU)                       │
│  - TPL Dataflow (pipeline processing)                  │
│  - SIMD/Vectorized (math operations)                   │
├────────────────────────────────────────────────────────┤
│  Streaming                                              │
│  - IAsyncEnumerable<T> (async streams)                 │
│  - Rx.NET / IObservable (reactive streams)             │
│  - Channel<T> with backpressure                        │
├────────────────────────────────────────────────────────┤
│  Patterns                                               │
│  - Actor Model (Orleans)                               │
│  - ValueTask (hot path optimization)                   │
│  - TaskCompletionSource (manual completion)            │
│  - AsyncLocal (context propagation)                    │
└────────────────────────────────────────────────────────┘
```

---

*Part 39 ครอบคลุม Advanced Concurrency & Parallelism ทั้งหมด 30 Steps (1121-1150)*  
*ต่อไป Part 40: Memory Management & Unsafe Code*
