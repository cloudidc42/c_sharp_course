# Part 14: Async/Await และ Asynchronous Programming (Steps 411-450)

## เป้าหมายการเรียนรู้
- เข้าใจ async/await และ Task-based Asynchronous Pattern (TAP)
- Task, Task<T>, ValueTask<T>
- Cancellation, Progress reporting
- Parallel operations ด้วย Task.WhenAll/WhenAny
- Channel<T> สำหรับ producer-consumer
- async streams (IAsyncEnumerable<T>)
- Error handling ใน async code

---

## Step 411: Async/Await Fundamentals

```csharp
using System;
using System.Threading.Tasks;
using System.Net.Http;

// async method คืน Task (void equivalent) หรือ Task<T>
// await หยุด method ชั่วคราวโดยไม่ block thread

class AsyncBasics
{
    // async void — ใช้สำหรับ event handlers เท่านั้น!
    // ไม่สามารถ await ได้ และ exception จะไม่ถูก catch
    static async void BadMethod()
    {
        await Task.Delay(1000);
        // Exception here = unhandled!
    }
    
    // async Task — แนะนำสำหรับ fire-and-forget methods
    static async Task GoodVoidMethod()
    {
        await Task.Delay(1000);
        Console.WriteLine("Done");
    }
    
    // async Task<T> — method ที่ return ค่า
    static async Task<string> FetchDataAsync(string url)
    {
        using var client = new HttpClient();
        string data = await client.GetStringAsync(url);
        return data;
    }
    
    // async method ธรรมดา
    static async Task<int> AddAsync(int a, int b)
    {
        await Task.Delay(100); // simulate async work
        return a + b;
    }
    
    static async Task Main()
    {
        // await รอให้ task เสร็จ
        int result = await AddAsync(3, 4);
        Console.WriteLine($"Result: {result}"); // 7
        
        // สร้าง task แต่ไม่รอ (fire and forget) — ระวัง!
        _ = GoodVoidMethod();
        
        // รอ task ที่สร้างไว้แล้ว
        Task<int> task = AddAsync(10, 20);
        // ... do other work here ...
        int result2 = await task; // wait when needed
        
        // Task.CompletedTask — pre-completed task
        Task completed = Task.CompletedTask;
        
        // Task.FromResult — completed task with value
        Task<int> immediate = Task.FromResult(42);
        int val = await immediate; // returns immediately
        
        // Task.FromException — failed task
        Task failed = Task.FromException(new InvalidOperationException("Oops"));
        
        // Task.FromCanceled — cancelled task
        Task cancelled = Task.FromCanceled(new CancellationToken(canceled: true));
        
        // Awaiting null-safe (C# 8+)
        Task? maybeTask = GetMaybeTask();
        if (maybeTask != null) await maybeTask;
        // Or:
        await (GetMaybeTask() ?? Task.CompletedTask);
    }
    
    static Task? GetMaybeTask() => null;
}
```

---

## Step 412: Task ประเภทต่างๆ

```csharp
using System.Threading.Tasks;

// Task — completed tasks
Task completed = Task.CompletedTask;

// Task<T> — task with result
Task<int> withResult = Task.FromResult(42);
int val = await withResult;

// ValueTask<T> — for hot paths (avoid heap allocation when completed synchronously)
static ValueTask<int> GetValueAsync(bool immediate)
{
    if (immediate)
        return new ValueTask<int>(42); // No allocation
    
    return new ValueTask<int>(ComputeAsync()); // Falls back to Task
    
    static async Task<int> ComputeAsync()
    {
        await Task.Delay(100);
        return 42;
    }
}

// Task.Run — run on thread pool
static async Task DoCpuWork()
{
    // Move CPU-intensive work to background thread
    int result = await Task.Run(() =>
    {
        // This runs on a thread pool thread
        return Enumerable.Range(1, 1_000_000).Sum();
    });
    Console.WriteLine($"Sum: {result}");
}

// Task.Delay — async sleep
static async Task DelayedMessage(string msg, int delayMs)
{
    await Task.Delay(delayMs);
    Console.WriteLine($"[{DateTime.Now:T}] {msg}");
}

// Task states
static async Task TaskStates()
{
    var cts = new CancellationTokenSource();
    
    Task running = Task.Run(async () => await Task.Delay(5000));
    Console.WriteLine($"IsCompleted: {running.IsCompleted}");   // False
    Console.WriteLine($"Status: {running.Status}");              // Running
    
    await Task.Delay(100);
    
    Task<int> faulted = Task.Run<int>(() => throw new Exception("Bad"));
    try { await faulted; }
    catch { }
    Console.WriteLine($"IsFaulted: {faulted.IsFaulted}");        // True
    Console.WriteLine($"Exception: {faulted.Exception?.InnerException?.Message}");
    
    cts.Cancel();
    Task cancelled = Task.Run(() => { cts.Token.ThrowIfCancellationRequested(); }, cts.Token);
    try { await cancelled; }
    catch (OperationCanceledException) { }
    Console.WriteLine($"IsCanceled: {cancelled.IsCanceled}");    // True
    
    Task done = Task.CompletedTask;
    Console.WriteLine($"IsCompleted: {done.IsCompleted}");       // True
    Console.WriteLine($"Status: {done.Status}");                 // RanToCompletion
}

// Unwrapping nested tasks
async Task<int> Nested()
{
    Task<Task<int>> nested = Task.Run(() => Task.Run(() => 42));
    Task<int> unwrapped = await nested; // Unwrap outer task
    return await unwrapped;            // Get value
    // Or: return await await nested;
}
```

---

## Step 413: Cancellation

```csharp
using System.Threading;

// CancellationTokenSource — เจ้าของ token
// CancellationToken — ส่งให้ operations รับรู้การยกเลิก

static async Task<string> LongOperationAsync(CancellationToken cancellationToken)
{
    for (int i = 0; i < 10; i++)
    {
        // Check for cancellation
        cancellationToken.ThrowIfCancellationRequested();
        
        Console.WriteLine($"Step {i + 1}/10...");
        await Task.Delay(500, cancellationToken); // cancellable delay
    }
    return "Completed!";
}

static async Task Main()
{
    using var cts = new CancellationTokenSource();
    
    // Cancel after 2 seconds
    cts.CancelAfter(TimeSpan.FromSeconds(2));
    
    // Or cancel manually
    _ = Task.Run(async () =>
    {
        await Task.Delay(1500);
        cts.Cancel(); // cancel from another task
    });
    
    try
    {
        string result = await LongOperationAsync(cts.Token);
        Console.WriteLine(result);
    }
    catch (OperationCanceledException)
    {
        Console.WriteLine("Operation was cancelled!");
    }
    finally
    {
        Console.WriteLine("Cleanup in finally");
    }
    
    // Linked tokens — cancel if either is cancelled
    using var cts1 = new CancellationTokenSource();
    using var cts2 = new CancellationTokenSource();
    using var linked = CancellationTokenSource.CreateLinkedTokenSource(cts1.Token, cts2.Token);
    
    cts1.CancelAfter(1000); // cancel after 1 second
    
    try
    {
        await Task.Delay(5000, linked.Token);
    }
    catch (OperationCanceledException)
    {
        Console.WriteLine($"Was cts1 cancelled: {cts1.IsCancellationRequested}"); // True
    }
    
    // Register cleanup on cancellation
    using var cts3 = new CancellationTokenSource();
    cts3.Token.Register(() => Console.WriteLine("Cleanup on cancel"));
    cts3.Cancel(); // Triggers the registered callback
    
    // Check without throwing
    if (cts3.Token.IsCancellationRequested)
        Console.WriteLine("Already cancelled");
    
    // Timeout wrapper
    using var timeout = new CancellationTokenSource(TimeSpan.FromSeconds(5));
    await DoWorkWithTimeout(timeout.Token);
}

static async Task DoWorkWithTimeout(CancellationToken token)
{
    try
    {
        await Task.Delay(Timeout.InfiniteTimeSpan, token);
    }
    catch (OperationCanceledException)
    {
        Console.WriteLine("Work timed out");
    }
}
```

---

## Step 414: Parallel Async Operations

```csharp
using System.Threading.Tasks;

// Task.WhenAll — รอทุก task เสร็จพร้อมกัน (parallel)
static async Task WhenAllDemo()
{
    // Sequential (slow — one after another)
    var watch = System.Diagnostics.Stopwatch.StartNew();
    string a = await FetchAsync("A", 1000);
    string b = await FetchAsync("B", 1500);
    string c = await FetchAsync("C", 500);
    Console.WriteLine($"Sequential: {watch.ElapsedMilliseconds}ms"); // ~3000ms
    
    // Parallel (fast — all at once)
    watch.Restart();
    Task<string> taskA = FetchAsync("A", 1000);
    Task<string> taskB = FetchAsync("B", 1500);
    Task<string> taskC = FetchAsync("C", 500);
    string[] results = await Task.WhenAll(taskA, taskB, taskC);
    Console.WriteLine($"Parallel: {watch.ElapsedMilliseconds}ms"); // ~1500ms
    Console.WriteLine(string.Join(", ", results));
    
    // WhenAll with array of tasks
    var tasks = Enumerable.Range(1, 10)
        .Select(i => FetchAsync($"Item{i}", i * 100))
        .ToList();
    string[] allResults = await Task.WhenAll(tasks);
    
    // WhenAll error handling — AggregateException
    try
    {
        var failingTasks = new[]
        {
            Task.Run(() => "ok"),
            Task.Run<string>(() => throw new Exception("Failed 1")),
            Task.Run<string>(() => throw new Exception("Failed 2")),
        };
        await Task.WhenAll(failingTasks);
    }
    catch (Exception ex)
    {
        Console.WriteLine($"One exception: {ex.Message}");
        // To get all exceptions, check the tasks themselves:
    }
    
    // Better: collect all results including failures
    var tasks2 = new[] { SuccessTask(), FailTask(), SuccessTask() };
    var allTasks = Task.WhenAll(tasks2);
    try { await allTasks; }
    catch { }
    
    foreach (var t in tasks2)
    {
        if (t.IsFaulted)
            Console.WriteLine($"Failed: {t.Exception!.InnerException!.Message}");
        else if (t.IsCompletedSuccessfully)
            Console.WriteLine($"Success: {t.Result}");
    }
}

static async Task<string> FetchAsync(string name, int delayMs)
{
    await Task.Delay(delayMs);
    return $"{name} result";
}

static async Task<int> SuccessTask() { await Task.Delay(100); return 42; }
static async Task<int> FailTask() { await Task.Delay(50); throw new Exception("Oops"); }

// Task.WhenAny — rerturn when ANY task completes
static async Task WhenAnyDemo()
{
    // Race condition pattern
    Task<string> fast = FetchAsync("fast", 500);
    Task<string> slow = FetchAsync("slow", 2000);
    
    Task<string> winner = await Task.WhenAny(fast, slow);
    Console.WriteLine($"Winner: {await winner}"); // "fast result"
    
    // Timeout pattern using WhenAny
    static async Task<T> WithTimeout<T>(Task<T> task, TimeSpan timeout)
    {
        Task delay = Task.Delay(timeout);
        Task completed = await Task.WhenAny(task, delay);
        if (completed == delay)
            throw new TimeoutException($"Operation timed out after {timeout}");
        return await task;
    }
    
    try
    {
        string result = await WithTimeout(FetchAsync("slow", 2000), TimeSpan.FromSeconds(1));
    }
    catch (TimeoutException ex)
    {
        Console.WriteLine(ex.Message); // Operation timed out after 00:00:01
    }
    
    // Process tasks as they complete
    var tasks = Enumerable.Range(1, 5)
        .Select(i => FetchAsync($"Item{i}", (6 - i) * 200))
        .ToList();
    
    while (tasks.Count > 0)
    {
        Task<string> completed = await Task.WhenAny(tasks.Cast<Task<string>>());
        tasks.Remove(completed);
        Console.WriteLine($"Completed: {await completed}");
    }
}
```

---

## Step 415: Progress Reporting

```csharp
using System;
using System.Threading.Tasks;

// IProgress<T> — for reporting progress back to caller
static async Task<int> ProcessItemsAsync(
    IEnumerable<int> items,
    IProgress<int>? progress = null,
    CancellationToken cancellationToken = default)
{
    int processed = 0;
    int total = items.Count();
    
    foreach (int item in items)
    {
        cancellationToken.ThrowIfCancellationRequested();
        
        await Task.Delay(100, cancellationToken); // simulate work
        processed++;
        
        // Report progress (percentage)
        progress?.Report((int)((double)processed / total * 100));
    }
    
    return processed;
}

// Progress<T> — thread-safe, invokes callback on original thread
static async Task Main()
{
    // Simple progress reporting
    var progress = new Progress<int>(pct =>
    {
        Console.Write($"\rProgress: {pct}% [{new string('#', pct / 5),-20}]");
    });
    
    int count = await ProcessItemsAsync(Enumerable.Range(1, 20), progress);
    Console.WriteLine($"\nProcessed {count} items");
    
    // Detailed progress
    record ProgressInfo(int Current, int Total, string Message);
    
    static async Task LongTask(IProgress<ProgressInfo>? progress, CancellationToken ct)
    {
        string[] steps = { "Connecting", "Fetching data", "Processing", "Saving", "Cleaning up" };
        
        for (int i = 0; i < steps.Length; i++)
        {
            ct.ThrowIfCancellationRequested();
            progress?.Report(new(i + 1, steps.Length, steps[i]));
            await Task.Delay(400, ct);
        }
    }
    
    var detailedProgress = new Progress<ProgressInfo>(p =>
    {
        Console.WriteLine($"[{p.Current}/{p.Total}] {p.Message}...");
    });
    
    using var cts = new CancellationTokenSource();
    await LongTask(detailedProgress, cts.Token);
    
    // Combine progress + cancellation
    static async Task<List<string>> BatchProcessAsync(
        List<string> items,
        Func<string, CancellationToken, Task<string>> processor,
        IProgress<(int done, int total)>? progress = null,
        CancellationToken cancellationToken = default)
    {
        var results = new List<string>();
        
        for (int i = 0; i < items.Count; i++)
        {
            cancellationToken.ThrowIfCancellationRequested();
            string result = await processor(items[i], cancellationToken);
            results.Add(result);
            progress?.Report((i + 1, items.Count));
        }
        
        return results;
    }
}
```

---

## Step 416: Async Streams (IAsyncEnumerable<T>)

```csharp
using System.Collections.Generic;
using System.Runtime.CompilerServices;

// IAsyncEnumerable<T> — C# 8+
// สำหรับ sequence ที่ generate แบบ async

// สร้าง async stream ด้วย yield return
static async IAsyncEnumerable<int> GenerateNumbersAsync(
    int count,
    [EnumeratorCancellation] CancellationToken cancellationToken = default)
{
    for (int i = 0; i < count; i++)
    {
        cancellationToken.ThrowIfCancellationRequested();
        await Task.Delay(100, cancellationToken); // async work
        yield return i;
    }
}

// อ่าน database แบบ streaming
static async IAsyncEnumerable<string> ReadRecordsAsync(
    string connectionString,
    [EnumeratorCancellation] CancellationToken cancellationToken = default)
{
    // Simulated database reader
    for (int i = 1; i <= 100; i++)
    {
        cancellationToken.ThrowIfCancellationRequested();
        await Task.Delay(10, cancellationToken);
        yield return $"Record {i}: Data from DB";
    }
}

// Consume async stream
static async Task ConsumeAsyncStream()
{
    // await foreach
    await foreach (int number in GenerateNumbersAsync(5))
    {
        Console.WriteLine(number);
    }
    
    // With cancellation
    using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(2));
    try
    {
        await foreach (int num in GenerateNumbersAsync(100, cts.Token).WithCancellation(cts.Token))
        {
            Console.Write(num + " ");
        }
    }
    catch (OperationCanceledException)
    {
        Console.WriteLine("\nCancelled!");
    }
    
    // LINQ with async streams
    var filtered = GenerateNumbersAsync(20)
        .Where(n => n % 2 == 0)
        .Select(n => n * n);
    
    await foreach (int sq in filtered)
        Console.Write(sq + " "); // 0, 4, 16, 36, ...
    
    // Collect to list
    List<int> list = await GenerateNumbersAsync(10).ToListAsync();
    
    // First/Any/etc.
    int? first = await GenerateNumbersAsync(10).FirstOrDefaultAsync(n => n > 5);
    Console.WriteLine($"First > 5: {first}"); // 6
}

// Channel-based async stream
using System.Threading.Channels;

static async Task ChannelDemo()
{
    // Channel = high-performance producer-consumer
    var channel = Channel.CreateBounded<int>(capacity: 10);
    
    // Producer
    async Task Producer()
    {
        for (int i = 0; i < 20; i++)
        {
            await channel.Writer.WriteAsync(i);
            Console.WriteLine($"Produced: {i}");
            await Task.Delay(100);
        }
        channel.Writer.Complete();
    }
    
    // Consumer
    async Task Consumer()
    {
        await foreach (int item in channel.Reader.ReadAllAsync())
        {
            Console.WriteLine($"Consumed: {item}");
            await Task.Delay(200); // slower than producer
        }
    }
    
    await Task.WhenAll(Producer(), Consumer());
}

// Extension methods for IAsyncEnumerable
static class AsyncEnumerableExtensions
{
    public static async Task<List<T>> ToListAsync<T>(this IAsyncEnumerable<T> source)
    {
        var list = new List<T>();
        await foreach (var item in source)
            list.Add(item);
        return list;
    }
    
    public static async IAsyncEnumerable<TResult> Select<T, TResult>(
        this IAsyncEnumerable<T> source,
        Func<T, TResult> selector,
        [EnumeratorCancellation] CancellationToken cancellationToken = default)
    {
        await foreach (T item in source.WithCancellation(cancellationToken))
            yield return selector(item);
    }
    
    public static async IAsyncEnumerable<T> Where<T>(
        this IAsyncEnumerable<T> source,
        Func<T, bool> predicate,
        [EnumeratorCancellation] CancellationToken cancellationToken = default)
    {
        await foreach (T item in source.WithCancellation(cancellationToken))
            if (predicate(item))
                yield return item;
    }
    
    public static async Task<T?> FirstOrDefaultAsync<T>(
        this IAsyncEnumerable<T> source,
        Func<T, bool> predicate)
    {
        await foreach (T item in source)
            if (predicate(item))
                return item;
        return default;
    }
}
```

---

## Step 417: Async Error Handling

```csharp
// Exception handling ใน async code

// 1. Basic try-catch
static async Task BasicErrorHandling()
{
    try
    {
        await ThrowingMethodAsync();
    }
    catch (InvalidOperationException ex)
    {
        Console.WriteLine($"Caught: {ex.Message}");
    }
    catch (Exception ex)
    {
        Console.WriteLine($"General: {ex.Message}");
    }
    finally
    {
        Console.WriteLine("Finally block runs");
    }
}

// 2. AggregateException from WhenAll
static async Task WhenAllErrors()
{
    Task[] tasks =
    {
        Task.Run(() => throw new Exception("Error 1")),
        Task.Run(() => throw new Exception("Error 2")),
        Task.Run(() => throw new Exception("Error 3")),
    };
    
    try
    {
        await Task.WhenAll(tasks);
    }
    catch (Exception ex)
    {
        // Only first exception in catch
        Console.WriteLine($"First exception: {ex.Message}");
        
        // Get ALL exceptions from the Task itself
        var aggregate = tasks
            .Where(t => t.IsFaulted)
            .SelectMany(t => t.Exception!.InnerExceptions);
        
        foreach (var e in aggregate)
            Console.WriteLine($"  - {e.Message}");
    }
}

// 3. Exception filter
static async Task ExceptionFilters()
{
    try
    {
        await RiskyOperationAsync();
    }
    catch (HttpRequestException ex) when (ex.StatusCode == System.Net.HttpStatusCode.NotFound)
    {
        Console.WriteLine("404 Not Found");
    }
    catch (HttpRequestException ex) when (ex.StatusCode == System.Net.HttpStatusCode.ServiceUnavailable)
    {
        Console.WriteLine("503 Service Unavailable");
    }
    catch (HttpRequestException ex)
    {
        Console.WriteLine($"HTTP Error: {ex.Message}");
    }
}

// 4. Retry with exponential backoff
static async Task<T> RetryAsync<T>(
    Func<Task<T>> operation,
    int maxAttempts = 3,
    Func<int, TimeSpan>? delay = null,
    Func<Exception, bool>? shouldRetry = null)
{
    delay ??= attempt => TimeSpan.FromSeconds(Math.Pow(2, attempt));
    shouldRetry ??= _ => true;
    
    for (int attempt = 0; attempt < maxAttempts; attempt++)
    {
        try
        {
            return await operation();
        }
        catch (Exception ex) when (attempt < maxAttempts - 1 && shouldRetry(ex))
        {
            TimeSpan waitTime = delay(attempt);
            Console.WriteLine($"Attempt {attempt + 1} failed: {ex.Message}. Retrying in {waitTime}...");
            await Task.Delay(waitTime);
        }
    }
    
    // Last attempt (let exception propagate)
    return await operation();
}

// 5. Result type for async operations
record AsyncResult<T>
{
    public T? Value { get; init; }
    public Exception? Error { get; init; }
    public bool IsSuccess => Error == null;
    
    public static AsyncResult<T> Success(T value) => new() { Value = value };
    public static AsyncResult<T> Failure(Exception error) => new() { Error = error };
}

static async Task<AsyncResult<string>> SafeFetchAsync(string url)
{
    try
    {
        using var client = new HttpClient();
        string result = await client.GetStringAsync(url);
        return AsyncResult<string>.Success(result);
    }
    catch (Exception ex)
    {
        return AsyncResult<string>.Failure(ex);
    }
}

static async Task ThrowingMethodAsync()
{
    await Task.Delay(100);
    throw new InvalidOperationException("Something went wrong");
}

static async Task RiskyOperationAsync()
{
    await Task.Delay(100);
    throw new HttpRequestException("Not found", null, System.Net.HttpStatusCode.NotFound);
}
```

---

## Step 418: ConfigureAwait และ Synchronization Context

```csharp
// ConfigureAwait(false) — ไม่จำเป็นต้อง resume บน original context
// ใช้ใน library code เพื่อป้องกัน deadlock

// UI Thread context (เช่น WPF/WinForms)
static async Task UIMethod()
{
    // Control operations must be on UI thread
    label.Text = "Starting...";
    
    // ConfigureAwait(true) — default, resume on UI thread
    await Task.Delay(1000).ConfigureAwait(true);
    label.Text = "Done!"; // Safe — back on UI thread
}

static async Task LibraryMethod()
{
    // Library code: don't need UI context, improves performance
    await Task.Delay(1000).ConfigureAwait(false);
    // Can't access UI here, but that's OK for library code
}

// Deadlock scenario (classic ASP.NET problem — not in modern ASP.NET Core)
static async Task<string> GetDataAsync()
{
    await Task.Delay(100);
    return "data";
}

// Deadlock: calling .Result on async method that needs to resume on same thread
// string result = GetDataAsync().Result; // DEADLOCK!

// Fix 1: await properly
// string result = await GetDataAsync(); // OK

// Fix 2: ConfigureAwait(false) in library
static async Task<string> GetDataAsyncSafe()
{
    await Task.Delay(100).ConfigureAwait(false);
    return "data";
}

// Fix 3: Task.Run to break out of synchronization context
// string result = await Task.Run(() => GetDataAsync());

// SynchronizationContext and TaskScheduler
static async Task ContextDemo()
{
    // Get current context
    SynchronizationContext? context = SynchronizationContext.Current;
    Console.WriteLine($"Context: {context?.GetType().Name ?? "null"}");
    
    // Execute on ThreadPool
    await Task.Run(() =>
    {
        Console.WriteLine($"On thread pool. Thread: {Thread.CurrentThread.IsThreadPoolThread}");
    });
    
    // Execute on specific scheduler
    TaskScheduler scheduler = TaskScheduler.Default; // ThreadPool scheduler
    await Task.Factory.StartNew(
        () => Console.WriteLine("On specific scheduler"),
        CancellationToken.None,
        TaskCreationOptions.None,
        scheduler
    );
}
```

---

## Step 419: Semaphore, Mutex, Lock ใน Async

```csharp
using System.Threading;

// SemaphoreSlim — async-compatible semaphore
static async Task SemaphoreDemo()
{
    // ควบคุมจำนวน concurrent operations
    var semaphore = new SemaphoreSlim(initialCount: 3, maxCount: 3);
    
    async Task DoWorkAsync(int id)
    {
        await semaphore.WaitAsync();
        try
        {
            Console.WriteLine($"[{id}] Working... (Active: {3 - semaphore.CurrentCount})");
            await Task.Delay(1000); // simulate work
        }
        finally
        {
            semaphore.Release();
        }
    }
    
    // Run 10 tasks with max 3 concurrent
    var tasks = Enumerable.Range(1, 10).Select(i => DoWorkAsync(i));
    await Task.WhenAll(tasks);
}

// Rate limiter with SemaphoreSlim
class RateLimiter
{
    private readonly SemaphoreSlim _semaphore;
    private readonly Queue<DateTime> _requestTimes = new();
    private readonly int _maxRequests;
    private readonly TimeSpan _window;
    private readonly SemaphoreSlim _lock = new(1, 1);
    
    public RateLimiter(int maxRequests, TimeSpan window)
    {
        _maxRequests = maxRequests;
        _window = window;
        _semaphore = new SemaphoreSlim(maxRequests, maxRequests);
    }
    
    public async Task<T> ExecuteAsync<T>(Func<Task<T>> operation)
    {
        await _semaphore.WaitAsync();
        try
        {
            return await operation();
        }
        finally
        {
            // Release after window expires
            _ = Task.Delay(_window).ContinueWith(_ => _semaphore.Release());
        }
    }
}

// Async lock pattern (Monitor doesn't work with async!)
class AsyncLock
{
    private readonly SemaphoreSlim _semaphore = new(1, 1);
    
    public async Task<IDisposable> LockAsync()
    {
        await _semaphore.WaitAsync();
        return new LockReleaser(_semaphore);
    }
    
    class LockReleaser : IDisposable
    {
        private readonly SemaphoreSlim _semaphore;
        public LockReleaser(SemaphoreSlim s) => _semaphore = s;
        public void Dispose() => _semaphore.Release();
    }
}

// Usage
static async Task AsyncLockDemo()
{
    var asyncLock = new AsyncLock();
    var counter = 0;
    
    async Task IncrementAsync()
    {
        using var _ = await asyncLock.LockAsync();
        int current = counter;
        await Task.Delay(1); // simulate async work between read and write
        counter = current + 1;
    }
    
    await Task.WhenAll(Enumerable.Range(0, 100).Select(_ => IncrementAsync()));
    Console.WriteLine($"Counter: {counter}"); // Should be 100
}
```

---

## Step 420: โปรแกรมตัวอย่าง — Async Download Manager

```csharp
using System;
using System.Collections.Concurrent;
using System.Collections.Generic;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;
using System.Threading.Channels;

// ============================================================
// Async Download Manager — ระบบดาวน์โหลดแบบ async
// ============================================================

record DownloadTask(int Id, string Url, string Filename);
record DownloadResult(int TaskId, string Filename, bool Success, string? Error, TimeSpan Duration);

class DownloadProgress
{
    public int TaskId { get; init; }
    public string Filename { get; init; } = "";
    public double Percentage { get; set; }
    public string Status { get; set; } = "";
    public long BytesDownloaded { get; set; }
}

class DownloadManager
{
    private readonly int _maxConcurrent;
    private readonly SemaphoreSlim _semaphore;
    private readonly CancellationTokenSource _globalCts;
    private readonly ConcurrentDictionary<int, DownloadProgress> _progress;
    private readonly Channel<DownloadResult> _resultChannel;
    
    public event EventHandler<DownloadProgress>? ProgressChanged;
    public event EventHandler<DownloadResult>? DownloadCompleted;
    
    public DownloadManager(int maxConcurrent = 3)
    {
        _maxConcurrent = maxConcurrent;
        _semaphore = new SemaphoreSlim(maxConcurrent, maxConcurrent);
        _globalCts = new CancellationTokenSource();
        _progress = new ConcurrentDictionary<int, DownloadProgress>();
        _resultChannel = Channel.CreateUnbounded<DownloadResult>();
    }
    
    public async Task<List<DownloadResult>> DownloadAllAsync(
        List<DownloadTask> tasks,
        IProgress<(int completed, int total)>? overallProgress = null,
        CancellationToken cancellationToken = default)
    {
        using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(
            _globalCts.Token, cancellationToken);
        
        int completed = 0;
        int total = tasks.Count;
        
        var downloadTasks = tasks.Select(task => DownloadWithSemaphoreAsync(task, linkedCts.Token));
        var results = await Task.WhenAll(downloadTasks);
        
        return results.ToList();
    }
    
    private async Task<DownloadResult> DownloadWithSemaphoreAsync(
        DownloadTask task,
        CancellationToken cancellationToken)
    {
        await _semaphore.WaitAsync(cancellationToken);
        try
        {
            return await DownloadAsync(task, cancellationToken);
        }
        finally
        {
            _semaphore.Release();
        }
    }
    
    private async Task<DownloadResult> DownloadAsync(
        DownloadTask task,
        CancellationToken cancellationToken)
    {
        var progress = new DownloadProgress
        {
            TaskId = task.Id,
            Filename = task.Filename,
            Status = "Starting"
        };
        _progress[task.Id] = progress;
        
        var stopwatch = System.Diagnostics.Stopwatch.StartNew();
        
        try
        {
            // Simulate download with progress
            progress.Status = "Downloading";
            int totalBytes = new Random().Next(1000, 10000);
            
            for (int i = 0; i <= 10; i++)
            {
                cancellationToken.ThrowIfCancellationRequested();
                
                await Task.Delay(200 + new Random().Next(100), cancellationToken);
                
                progress.BytesDownloaded = (long)(totalBytes * i / 10.0);
                progress.Percentage = i * 10;
                ProgressChanged?.Invoke(this, progress);
            }
            
            progress.Status = "Completed";
            progress.Percentage = 100;
            ProgressChanged?.Invoke(this, progress);
            
            var result = new DownloadResult(task.Id, task.Filename, true, null, stopwatch.Elapsed);
            DownloadCompleted?.Invoke(this, result);
            return result;
        }
        catch (OperationCanceledException)
        {
            progress.Status = "Cancelled";
            ProgressChanged?.Invoke(this, progress);
            return new DownloadResult(task.Id, task.Filename, false, "Cancelled", stopwatch.Elapsed);
        }
        catch (Exception ex)
        {
            progress.Status = "Failed";
            ProgressChanged?.Invoke(this, progress);
            return new DownloadResult(task.Id, task.Filename, false, ex.Message, stopwatch.Elapsed);
        }
        finally
        {
            stopwatch.Stop();
        }
    }
    
    public void CancelAll() => _globalCts.Cancel();
    
    public IReadOnlyDictionary<int, DownloadProgress> GetProgress()
        => _progress;
}

// Progress display
class ProgressDisplay
{
    private readonly int _totalTasks;
    private readonly object _lock = new();
    private int _completedTasks;
    
    public ProgressDisplay(int totalTasks)
    {
        _totalTasks = totalTasks;
        Console.Clear();
    }
    
    public void Update(DownloadProgress progress)
    {
        lock (_lock)
        {
            // Simple console output (in real app, could use more sophisticated UI)
            string bar = new string('#', (int)(progress.Percentage / 5));
            Console.WriteLine($"[{progress.TaskId:D2}] {progress.Filename,-20} [{bar,-20}] {progress.Percentage:F0}% {progress.Status}");
        }
    }
    
    public void OnCompleted(DownloadResult result)
    {
        lock (_lock)
        {
            _completedTasks++;
            string status = result.Success ? "✓" : "✗";
            Console.WriteLine($"{status} Task {result.TaskId}: {result.Filename} ({result.Duration.TotalSeconds:F1}s) {(result.Error != null ? $"- {result.Error}" : "")}");
        }
    }
}

// Main demo
static async Task Main()
{
    var downloads = new List<DownloadTask>
    {
        new(1, "https://example.com/file1.zip",  "file1.zip"),
        new(2, "https://example.com/file2.zip",  "file2.zip"),
        new(3, "https://example.com/data.csv",   "data.csv"),
        new(4, "https://example.com/image.png",  "image.png"),
        new(5, "https://example.com/video.mp4",  "video.mp4"),
        new(6, "https://example.com/report.pdf", "report.pdf"),
        new(7, "https://example.com/music.mp3",  "music.mp3"),
        new(8, "https://example.com/docs.zip",   "docs.zip"),
    };
    
    var manager = new DownloadManager(maxConcurrent: 3);
    var display = new ProgressDisplay(downloads.Count);
    
    manager.ProgressChanged += (_, p) => display.Update(p);
    manager.DownloadCompleted += (_, r) => display.OnCompleted(r);
    
    Console.WriteLine($"Starting {downloads.Count} downloads (max 3 concurrent)...\n");
    
    using var cts = new CancellationTokenSource();
    
    // Optional: cancel after 5 seconds for demo
    // cts.CancelAfter(5000);
    
    var stopwatch = System.Diagnostics.Stopwatch.StartNew();
    var results = await manager.DownloadAllAsync(downloads, cancellationToken: cts.Token);
    stopwatch.Stop();
    
    Console.WriteLine($"\n=== Summary ===");
    Console.WriteLine($"Total time: {stopwatch.Elapsed.TotalSeconds:F1}s");
    Console.WriteLine($"Successful: {results.Count(r => r.Success)}/{results.Count}");
    Console.WriteLine($"Failed: {results.Count(r => !r.Success)}");
    
    // Async stream version
    Console.WriteLine("\n=== Processing results as stream ===");
    await foreach (var result in ProcessResultsAsync(results))
    {
        Console.WriteLine($"Processed: {result}");
    }
    
    static async IAsyncEnumerable<string> ProcessResultsAsync(IEnumerable<DownloadResult> results)
    {
        foreach (var result in results.Where(r => r.Success))
        {
            await Task.Delay(100); // async processing
            yield return $"Processed {result.Filename} in {result.Duration.TotalSeconds:F1}s";
        }
    }
}
```

---

## สรุป Part 14

### Async/Await Basics
- ✅ `async Task`, `async Task<T>`, `async ValueTask<T>`
- ✅ Deferred execution — จะไม่รันจนกว่า await
- ✅ `Task.CompletedTask`, `Task.FromResult`, `Task.FromException`

### Parallel Async
- ✅ `Task.WhenAll` — รอทุก task (parallel execution)
- ✅ `Task.WhenAny` — รอ task แรกที่เสร็จ
- ✅ Timeout pattern ด้วย `WhenAny`

### Cancellation
- ✅ `CancellationTokenSource`, `CancellationToken`
- ✅ `ThrowIfCancellationRequested()`, `token.Register()`
- ✅ Linked tokens, `CancelAfter`

### Progress Reporting
- ✅ `IProgress<T>`, `Progress<T>` (thread-safe)
- ✅ ส่ง progress กลับไปยัง caller

### Async Streams
- ✅ `IAsyncEnumerable<T>` (C# 8+)
- ✅ `yield return` ใน async method
- ✅ `await foreach`, `[EnumeratorCancellation]`
- ✅ Channel<T> for producer-consumer

### Error Handling
- ✅ `try-catch-finally` ใน async methods
- ✅ `AggregateException` จาก `WhenAll`
- ✅ Exception filters `when`
- ✅ Retry with exponential backoff

### Advanced
- ✅ `ConfigureAwait(false)` — library best practice
- ✅ `SemaphoreSlim` สำหรับ rate limiting
- ✅ Async lock pattern

---

## แบบฝึกหัด

1. สร้าง `AsyncCache<TKey, TValue>` ที่ prevent thundering herd (multiple concurrent requests for same key)
2. เขียน `AsyncPipeline<T>` ที่รองรับ async steps แต่ละ step ใน parallel
3. สร้าง `BulkheadPolicy` ที่จำกัด concurrent calls และ queue overflow ด้วย `Channel<T>`
4. Implement `AsyncDisposable` pattern ด้วย `IAsyncDisposable`
5. สร้าง `PeriodicTimer`-based scheduler ที่รัน tasks ตาม schedule โดยรองรับ cancellation
