# Part 90: Advanced Concurrency — Channels, Dataflow & Actor Model

## Steps 2069–2084

---

## Step 2069: System.Threading.Channels — Producer-Consumer

Channels provide a high-performance, low-allocation pipeline for producer-consumer scenarios.

```csharp
// Bounded channel — backpressure built in
var channel = Channel.CreateBounded<WorkItem>(new BoundedChannelOptions(capacity: 1000)
{
    FullMode         = BoundedChannelFullMode.Wait,   // producer awaits when full
    SingleWriter     = false,  // multiple producers
    SingleReader     = false,  // multiple consumers
    AllowSynchronousContinuations = false
});

// Unbounded — careful, can grow without limit
var unbounded = Channel.CreateUnbounded<WorkItem>(new UnboundedChannelOptions
{
    SingleWriter = true,
    SingleReader = true,
    AllowSynchronousContinuations = false
});

// Producer
public class OrderImportProducer
{
    private readonly ChannelWriter<RawOrderData> _writer;

    public OrderImportProducer(ChannelWriter<RawOrderData> writer) => _writer = writer;

    public async Task ProduceAsync(Stream csvStream, CancellationToken ct = default)
    {
        using var reader = new StreamReader(csvStream);
        string? line;
        var lineNumber = 0;

        while ((line = await reader.ReadLineAsync(ct)) is not null)
        {
            lineNumber++;
            if (lineNumber == 1) continue; // skip header

            var data = ParseCsvLine(line, lineNumber);

            // Write to channel — awaits if channel is full (backpressure)
            await _writer.WriteAsync(data, ct);
        }

        // Signal no more items
        _writer.Complete();
    }

    private static RawOrderData ParseCsvLine(string line, int lineNumber)
    {
        var parts = line.Split(',');
        return new RawOrderData(
            LineNumber: lineNumber,
            CustomerId: Guid.Parse(parts[0]),
            ProductId:  Guid.Parse(parts[1]),
            Quantity:   int.Parse(parts[2]),
            UnitPrice:  decimal.Parse(parts[3]));
    }
}

// Consumer
public class OrderImportConsumer
{
    private readonly ChannelReader<RawOrderData> _reader;
    private readonly IOrderRepository _repo;
    private readonly ILogger<OrderImportConsumer> _logger;

    public OrderImportConsumer(ChannelReader<RawOrderData> reader, IOrderRepository repo, ILogger<OrderImportConsumer> logger)
    {
        _reader = reader;
        _repo   = repo;
        _logger = logger;
    }

    public async Task ConsumeAsync(CancellationToken ct = default)
    {
        var processed = 0;
        var failed    = 0;

        await foreach (var data in _reader.ReadAllAsync(ct))
        {
            try
            {
                await _repo.ImportAsync(data, ct);
                processed++;
            }
            catch (Exception ex)
            {
                failed++;
                _logger.LogWarning(ex, "Failed to import line {LineNumber}", data.LineNumber);
            }
        }

        _logger.LogInformation("Import complete: {Processed} processed, {Failed} failed", processed, failed);
    }
}
```

---

## Step 2070: Channel Pipeline — Multi-Stage

```csharp
// Multi-stage processing pipeline using channels
public class OrderProcessingPipeline
{
    public async Task RunAsync(Stream input, CancellationToken ct = default)
    {
        // Stage 1 → 2: raw CSV to parsed data
        var parseChannel = Channel.CreateBounded<RawOrderData>(500);

        // Stage 2 → 3: parsed data to validated orders
        var validateChannel = Channel.CreateBounded<ValidatedOrderData>(500);

        // Stage 3 → 4: validated orders to enriched (pricing, etc.)
        var enrichChannel = Channel.CreateBounded<EnrichedOrderData>(500);

        // Run all stages concurrently
        await Task.WhenAll(
            Stage1_ParseAsync(input, parseChannel.Writer, ct),
            Stage2_ValidateAsync(parseChannel.Reader, validateChannel.Writer, ct),
            Stage3_EnrichAsync(validateChannel.Reader, enrichChannel.Writer, ct),
            Stage4_PersistAsync(enrichChannel.Reader, ct));
    }

    private async Task Stage1_ParseAsync(Stream input, ChannelWriter<RawOrderData> writer, CancellationToken ct)
    {
        try
        {
            using var reader = new StreamReader(input);
            string? line;
            var lineNum = 0;

            while ((line = await reader.ReadLineAsync(ct)) is not null)
            {
                lineNum++;
                if (lineNum == 1) continue;
                var parsed = ParseLine(line, lineNum);
                await writer.WriteAsync(parsed, ct);
            }
        }
        finally
        {
            writer.Complete();
        }
    }

    private async Task Stage2_ValidateAsync(
        ChannelReader<RawOrderData> reader,
        ChannelWriter<ValidatedOrderData> writer,
        CancellationToken ct)
    {
        try
        {
            await foreach (var raw in reader.ReadAllAsync(ct))
            {
                if (IsValid(raw))
                    await writer.WriteAsync(new ValidatedOrderData(raw), ct);
                // Invalid items are dropped (or send to error channel)
            }
        }
        finally
        {
            writer.Complete();
        }
    }

    private async Task Stage3_EnrichAsync(
        ChannelReader<ValidatedOrderData> reader,
        ChannelWriter<EnrichedOrderData> writer,
        CancellationToken ct)
    {
        // Run 4 concurrent enrichers (fan-out within stage)
        var workers = Enumerable.Range(0, 4).Select(_ => EnrichWorkerAsync(reader, writer, ct));

        try
        {
            await Task.WhenAll(workers);
        }
        finally
        {
            writer.Complete();
        }
    }

    private async Task EnrichWorkerAsync(
        ChannelReader<ValidatedOrderData> reader,
        ChannelWriter<EnrichedOrderData> writer,
        CancellationToken ct)
    {
        await foreach (var validated in reader.ReadAllAsync(ct))
        {
            var enriched = await EnrichAsync(validated, ct);
            await writer.WriteAsync(enriched, ct);
        }
    }

    private async Task Stage4_PersistAsync(ChannelReader<EnrichedOrderData> reader, CancellationToken ct)
    {
        var batch = new List<EnrichedOrderData>(100);

        await foreach (var item in reader.ReadAllAsync(ct))
        {
            batch.Add(item);

            if (batch.Count >= 100)
            {
                await PersistBatchAsync(batch, ct);
                batch.Clear();
            }
        }

        // Flush remaining
        if (batch.Count > 0)
            await PersistBatchAsync(batch, ct);
    }

    private static bool IsValid(RawOrderData raw) => raw.Quantity > 0 && raw.UnitPrice >= 0;
    private static RawOrderData ParseLine(string line, int num) => new(num, Guid.NewGuid(), Guid.NewGuid(), 1, 10m);
    private static Task<EnrichedOrderData> EnrichAsync(ValidatedOrderData v, CancellationToken ct) => Task.FromResult(new EnrichedOrderData());
    private static Task PersistBatchAsync(List<EnrichedOrderData> batch, CancellationToken ct) => Task.CompletedTask;
}
```

---

## Step 2071: TPL Dataflow — Complex Pipelines

```csharp
// TPL Dataflow for complex branching pipelines
// dotnet add package System.Threading.Tasks.Dataflow (built into .NET 5+)

public class OrderProcessingDataflow
{
    public async Task<ProcessingResult> RunAsync(IEnumerable<ImportRecord> records, CancellationToken ct = default)
    {
        var result  = new ProcessingResult();
        var options = new ExecutionDataflowBlockOptions
        {
            MaxDegreeOfParallelism   = Environment.ProcessorCount,
            BoundedCapacity          = 1000,
            CancellationToken        = ct
        };

        // Stage 1: Parse
        var parseBlock = new TransformBlock<ImportRecord, ParsedOrder>(
            record => ParseRecord(record),
            new ExecutionDataflowBlockOptions { MaxDegreeOfParallelism = 4, BoundedCapacity = 500, CancellationToken = ct });

        // Stage 2: Validate (filter — drops invalid)
        var validateBlock = new TransformBlock<ParsedOrder, ValidatedOrder>(
            async order => await ValidateAsync(order, ct),
            options);

        // Stage 3: Enrich
        var enrichBlock = new TransformBlock<ValidatedOrder, EnrichedOrder>(
            async order => await EnrichAsync(order, ct),
            options);

        // Stage 4: Batch (group 100 items)
        var batchBlock = new BatchBlock<EnrichedOrder>(100,
            new GroupingDataflowBlockOptions { BoundedCapacity = 2000, CancellationToken = ct });

        // Stage 5: Persist
        var persistBlock = new ActionBlock<EnrichedOrder[]>(
            async batch =>
            {
                await PersistBatchAsync(batch, ct);
                Interlocked.Add(ref result.Saved, batch.Length);
            },
            new ExecutionDataflowBlockOptions { MaxDegreeOfParallelism = 2, BoundedCapacity = 10, CancellationToken = ct });

        // Error sink
        var errorBlock = new ActionBlock<(object Item, Exception Error)>(
            err =>
            {
                Interlocked.Increment(ref result.Failed);
                // log error
            });

        // Link the pipeline
        var propagate = new DataflowLinkOptions { PropagateCompletion = true };
        parseBlock.LinkTo(validateBlock, propagate);
        validateBlock.LinkTo(enrichBlock, propagate, order => order is not null); // filter nulls
        validateBlock.LinkTo(DataflowBlock.NullTarget<ValidatedOrder>());         // discard invalid
        enrichBlock.LinkTo(batchBlock, propagate);
        batchBlock.LinkTo(persistBlock, propagate);

        // Post all records
        foreach (var record in records)
            await parseBlock.SendAsync(record, ct);

        parseBlock.Complete();

        // Wait for pipeline to drain
        await persistBlock.Completion;

        return result;
    }

    private static ParsedOrder ParseRecord(ImportRecord r) => new(r.Id, r.CustomerId);
    private static Task<ValidatedOrder> ValidateAsync(ParsedOrder p, CancellationToken ct) => Task.FromResult(new ValidatedOrder());
    private static Task<EnrichedOrder> EnrichAsync(ValidatedOrder v, CancellationToken ct) => Task.FromResult(new EnrichedOrder());
    private static Task PersistBatchAsync(EnrichedOrder[] batch, CancellationToken ct) => Task.CompletedTask;
}

public class ProcessingResult
{
    public int Saved;
    public int Failed;
}
```

---

## Step 2072: Actor Model with Microsoft Orleans

Orleans provides virtual actors (Grains) — objects that exist virtually and are activated on-demand.

```bash
dotnet add package Microsoft.Orleans.Server
dotnet add package Microsoft.Orleans.Core.Abstractions
dotnet add package Microsoft.Orleans.Serialization.SystemTextJson
dotnet add package OrleansDashboard
```

```csharp
// Grain interface (contract)
public interface IOrderGrain : IGrainWithGuidKey
{
    Task<OrderState> GetStateAsync();
    Task PlaceAsync(PlaceOrderRequest request);
    Task CancelAsync(string reason);
    Task AddItemAsync(OrderItem item);
    Task ConfirmPaymentAsync(Guid paymentId);
}

// Grain implementation
[StatefulGrain(grainType: "order", storageName: "orders")]
public class OrderGrain : Grain<OrderStateData>, IOrderGrain
{
    private readonly ILogger<OrderGrain> _logger;

    public OrderGrain(ILogger<OrderGrain> logger) => _logger = logger;

    public Task<OrderState> GetStateAsync() =>
        Task.FromResult(MapToState(State));

    public async Task PlaceAsync(PlaceOrderRequest request)
    {
        if (State.Status != null)
            throw new InvalidOperationException("Order already placed");

        State.OrderId    = this.GetPrimaryKey();
        State.CustomerId = request.CustomerId;
        State.Items      = request.Items.ToList();
        State.Status     = "Pending";
        State.PlacedAt   = DateTimeOffset.UtcNow;
        State.TotalAmount = request.Items.Sum(i => i.Quantity * i.UnitPrice);

        await WriteStateAsync();

        _logger.LogInformation("Order {OrderId} placed for customer {CustomerId}",
            State.OrderId, State.CustomerId);

        // Publish domain event via Orleans streams
        var stream = this.GetStreamProvider("kafka")
            .GetStream<OrderPlacedEvent>(StreamId.Create("orders", State.OrderId));

        await stream.OnNextAsync(new OrderPlacedEvent
        {
            OrderId     = State.OrderId,
            CustomerId  = State.CustomerId,
            TotalAmount = State.TotalAmount,
            OccurredAt  = DateTimeOffset.UtcNow
        });
    }

    public async Task CancelAsync(string reason)
    {
        if (State.Status is null or "Cancelled")
            throw new InvalidOperationException($"Cannot cancel order in state: {State.Status}");

        State.Status           = "Cancelled";
        State.CancellationReason = reason;
        State.UpdatedAt        = DateTimeOffset.UtcNow;

        await WriteStateAsync();
    }

    public async Task AddItemAsync(OrderItem item)
    {
        if (State.Status != "Pending")
            throw new InvalidOperationException("Can only add items to pending orders");

        State.Items.Add(item);
        State.TotalAmount = State.Items.Sum(i => i.Quantity * i.UnitPrice);
        State.UpdatedAt   = DateTimeOffset.UtcNow;

        await WriteStateAsync();
    }

    public async Task ConfirmPaymentAsync(Guid paymentId)
    {
        State.PaymentId = paymentId;
        State.Status    = "PaymentConfirmed";
        State.UpdatedAt = DateTimeOffset.UtcNow;

        await WriteStateAsync();
    }

    private static OrderState MapToState(OrderStateData data) => new(
        OrderId:     data.OrderId,
        CustomerId:  data.CustomerId,
        TotalAmount: data.TotalAmount,
        Status:      data.Status ?? "Unknown",
        Items:       data.Items ?? [],
        PlacedAt:    data.PlacedAt,
        UpdatedAt:   data.UpdatedAt);
}

// Grain state
[GenerateSerializer, Alias("OrderStateData")]
public class OrderStateData
{
    [Id(0)] public Guid              OrderId       { get; set; }
    [Id(1)] public Guid              CustomerId    { get; set; }
    [Id(2)] public decimal           TotalAmount   { get; set; }
    [Id(3)] public string?           Status        { get; set; }
    [Id(4)] public List<OrderItem>   Items         { get; set; } = [];
    [Id(5)] public DateTimeOffset    PlacedAt      { get; set; }
    [Id(6)] public DateTimeOffset?   UpdatedAt     { get; set; }
    [Id(7)] public Guid?             PaymentId     { get; set; }
    [Id(8)] public string?           CancellationReason { get; set; }
}
```

---

## Step 2073: Orleans — Silo Configuration

```csharp
// Program.cs — Orleans silo setup
var builder = WebApplication.CreateBuilder(args);

builder.Host.UseOrleans(silo =>
{
    if (builder.Environment.IsDevelopment())
    {
        silo.UseLocalhostClustering();
        silo.AddMemoryGrainStorage("orders");
    }
    else
    {
        // Production: ADO.NET clustering with PostgreSQL
        silo.UseAdoNetClustering(options =>
        {
            options.Invariant  = "Npgsql";
            options.ConnectionString = builder.Configuration.GetConnectionString("Default");
        });

        // EF Core grain storage
        silo.AddAdoNetGrainStorage("orders", options =>
        {
            options.Invariant  = "Npgsql";
            options.ConnectionString = builder.Configuration.GetConnectionString("Default");
        });
    }

    silo.ConfigureEndpoints(
        siloPort: 11111,
        gatewayPort: 30000);

    silo.AddKafkaStreamProvider("kafka", builder =>
    {
        builder.AddTopic("orders", new TopicCreationConfig { Num_Partitions = 6 });
        builder.WithOptions(options =>
        {
            options.BrokerList = "localhost:9092";
            options.ConsumerGroupId = "orleans-orders";
        });
    });

    // Dashboard
    silo.UseDashboard(opts => opts.Port = 8082);

    // Grain activation timeout
    silo.Configure<GrainCollectionOptions>(opts =>
    {
        opts.CollectionAge       = TimeSpan.FromMinutes(10);
        opts.CollectionQuantum   = TimeSpan.FromMinutes(1);
    });
});

// Client usage
app.MapPost("/api/orders", async (
    PlaceOrderRequest req,
    IGrainFactory grains) =>
{
    var orderId    = Guid.NewGuid();
    var orderGrain = grains.GetGrain<IOrderGrain>(orderId);
    await orderGrain.PlaceAsync(req);

    return Results.Created($"/api/orders/{orderId}", new { orderId });
}).RequireAuthorization();

app.MapGet("/api/orders/{id}", async (
    Guid id,
    IGrainFactory grains) =>
{
    var grain = grains.GetGrain<IOrderGrain>(id);
    var state = await grain.GetStateAsync();
    return Results.Ok(state);
}).RequireAuthorization();
```

---

## Step 2074: Orleans — Timers, Reminders & Streams

```csharp
// Reminders (persistent, survive restarts)
public class OrderReminderGrain : Grain, IOrderReminderGrain, IRemindable
{
    private readonly IGrainFactory _grains;

    public OrderReminderGrain(IGrainFactory grains) => _grains = grains;

    public async Task ScheduleFollowUpAsync(Guid orderId, TimeSpan delay)
    {
        await this.RegisterOrUpdateReminder(
            reminderName: $"followup-{orderId}",
            dueTime:      delay,
            period:       TimeSpan.FromDays(1000)); // effectively one-shot
    }

    public async Task ReceiveReminder(string reminderName, TickStatus status)
    {
        if (reminderName.StartsWith("followup-"))
        {
            var orderId = Guid.Parse(reminderName["followup-".Length..]);
            var order   = _grains.GetGrain<IOrderGrain>(orderId);
            var state   = await order.GetStateAsync();

            if (state.Status == "Delivered")
            {
                // Send review request
            }
        }

        // Unregister one-shot reminder
        await this.UnregisterReminder(
            await this.GetReminder(reminderName));
    }
}

// Grain-to-grain streams
[ImplicitStreamSubscription("inventory-events")]
public class InventoryProjectionGrain : Grain, IInventoryProjectionGrain,
    IAsyncObserver<StockUpdatedEvent>
{
    private readonly IInventoryReadModel _readModel;

    public InventoryProjectionGrain(IInventoryReadModel readModel) => _readModel = readModel;

    public override async Task OnActivateAsync(CancellationToken ct)
    {
        var streamProvider = this.GetStreamProvider("kafka");
        var stream = streamProvider.GetStream<StockUpdatedEvent>(
            StreamId.Create("inventory-events", this.GetPrimaryKey()));

        await stream.SubscribeAsync(this);
    }

    public async Task OnNextAsync(StockUpdatedEvent evt, StreamSequenceToken? token = null)
    {
        await _readModel.UpdateStockLevelAsync(evt.ProductId, evt.NewLevel);
    }

    public Task OnErrorAsync(Exception ex) => Task.CompletedTask;
    public Task OnCompletedAsync()         => Task.CompletedTask;
}
```

---

## Step 2075: SemaphoreSlim & Async Locking

```csharp
// SemaphoreSlim for async mutual exclusion
public class DatabaseConnectionPool
{
    private readonly SemaphoreSlim _semaphore;
    private readonly Queue<DbConnection> _connections = new();

    public DatabaseConnectionPool(int maxConnections)
        => _semaphore = new SemaphoreSlim(maxConnections, maxConnections);

    public async Task<IDisposable> AcquireAsync(CancellationToken ct = default)
    {
        await _semaphore.WaitAsync(ct);
        return new ConnectionLease(this);
    }

    private void Release() => _semaphore.Release();

    private class ConnectionLease : IDisposable
    {
        private readonly DatabaseConnectionPool _pool;
        private bool _disposed;

        public ConnectionLease(DatabaseConnectionPool pool) => _pool = pool;

        public void Dispose()
        {
            if (!_disposed)
            {
                _pool.Release();
                _disposed = true;
            }
        }
    }
}

// AsyncLock — re-entrant async lock
public class AsyncLock
{
    private readonly SemaphoreSlim _semaphore = new(1, 1);

    public async Task<IDisposable> LockAsync(CancellationToken ct = default)
    {
        await _semaphore.WaitAsync(ct);
        return new Releaser(_semaphore);
    }

    private class Releaser : IDisposable
    {
        private readonly SemaphoreSlim _semaphore;

        public Releaser(SemaphoreSlim semaphore) => _semaphore = semaphore;

        public void Dispose() => _semaphore.Release();
    }
}

// Per-key locking — avoid locking all operations for a single key
public class KeyedAsyncLock
{
    private readonly ConcurrentDictionary<string, SemaphoreSlim> _locks = new();

    public async Task<IDisposable> LockAsync(string key, CancellationToken ct = default)
    {
        var semaphore = _locks.GetOrAdd(key, _ => new SemaphoreSlim(1, 1));
        await semaphore.WaitAsync(ct);

        return new KeyedReleaser(key, semaphore, _locks);
    }

    private class KeyedReleaser : IDisposable
    {
        private readonly string _key;
        private readonly SemaphoreSlim _semaphore;
        private readonly ConcurrentDictionary<string, SemaphoreSlim> _locks;

        public KeyedReleaser(string key, SemaphoreSlim semaphore, ConcurrentDictionary<string, SemaphoreSlim> locks)
        {
            _key       = key;
            _semaphore = semaphore;
            _locks     = locks;
        }

        public void Dispose()
        {
            _semaphore.Release();
            // Clean up if no waiters
            if (_semaphore.CurrentCount == 1)
                _locks.TryRemove(_key, out _);
        }
    }
}
```

---

## Step 2076: Parallel Processing — Partitioner & Batching

```csharp
// Efficient parallel processing with controlled concurrency
public class ParallelOrderProcessor
{
    private readonly IOrderService _service;

    public ParallelOrderProcessor(IOrderService service) => _service = service;

    // Process with bounded parallelism
    public async Task<ProcessingResult> ProcessManyAsync(
        IReadOnlyList<Guid> orderIds,
        int maxConcurrency = 10,
        CancellationToken ct = default)
    {
        var success = 0;
        var failure = 0;

        using var semaphore = new SemaphoreSlim(maxConcurrency, maxConcurrency);

        var tasks = orderIds.Select(async id =>
        {
            await semaphore.WaitAsync(ct);
            try
            {
                await _service.ProcessAsync(id, ct);
                Interlocked.Increment(ref success);
            }
            catch (Exception)
            {
                Interlocked.Increment(ref failure);
            }
            finally
            {
                semaphore.Release();
            }
        });

        await Task.WhenAll(tasks);
        return new ProcessingResult { Success = success, Failure = failure };
    }

    // Parallel.ForEachAsync (.NET 6+) — cleaner API
    public async Task<ProcessingResult> ProcessManyV2Async(
        IReadOnlyList<Guid> orderIds,
        int maxConcurrency = 10,
        CancellationToken ct = default)
    {
        var success = 0;
        var failure = 0;

        await Parallel.ForEachAsync(orderIds,
            new ParallelOptions
            {
                MaxDegreeOfParallelism = maxConcurrency,
                CancellationToken      = ct
            },
            async (id, innerCt) =>
            {
                try
                {
                    await _service.ProcessAsync(id, innerCt);
                    Interlocked.Increment(ref success);
                }
                catch (Exception)
                {
                    Interlocked.Increment(ref failure);
                }
            });

        return new ProcessingResult { Success = success, Failure = failure };
    }

    // Chunked processing — process in batches with concurrency
    public async Task ProcessInChunksAsync(
        IAsyncEnumerable<Guid> orderIds,
        int chunkSize = 100,
        int maxConcurrency = 4,
        CancellationToken ct = default)
    {
        var chunk = new List<Guid>(chunkSize);

        await foreach (var id in orderIds.WithCancellation(ct))
        {
            chunk.Add(id);

            if (chunk.Count >= chunkSize)
            {
                await ProcessChunkAsync(chunk, maxConcurrency, ct);
                chunk.Clear();
            }
        }

        if (chunk.Count > 0)
            await ProcessChunkAsync(chunk, maxConcurrency, ct);
    }

    private async Task ProcessChunkAsync(List<Guid> chunk, int maxConcurrency, CancellationToken ct)
    {
        await Parallel.ForEachAsync(chunk,
            new ParallelOptions { MaxDegreeOfParallelism = maxConcurrency, CancellationToken = ct },
            async (id, innerCt) => await _service.ProcessAsync(id, innerCt));
    }
}
```

---

## Step 2077: CancellationToken Patterns

```csharp
// Comprehensive CancellationToken usage
public class CancellationPatterns
{
    // Linked tokens — cancel when either source cancels
    public async Task LinkedCancellationAsync(CancellationToken requestCt)
    {
        using var timeoutCts = new CancellationTokenSource(TimeSpan.FromSeconds(30));
        using var linkedCts  = CancellationTokenSource.CreateLinkedTokenSource(
            requestCt, timeoutCts.Token);

        try
        {
            await DoWorkAsync(linkedCts.Token);
        }
        catch (OperationCanceledException) when (timeoutCts.IsCancellationRequested)
        {
            throw new TimeoutException("Operation exceeded 30 second limit");
        }
        catch (OperationCanceledException) when (requestCt.IsCancellationRequested)
        {
            // Request was cancelled — log and let propagate
            throw;
        }
    }

    // Cooperative cancellation with graceful shutdown
    public async IAsyncEnumerable<ProcessedItem> StreamProcessAsync(
        IAsyncEnumerable<RawItem> source,
        [EnumeratorCancellation] CancellationToken ct = default)
    {
        await foreach (var item in source.WithCancellation(ct))
        {
            ct.ThrowIfCancellationRequested();

            ProcessedItem processed;
            try
            {
                processed = await ProcessItemAsync(item, ct);
            }
            catch (OperationCanceledException)
            {
                yield break; // graceful stop
            }

            yield return processed;
        }
    }

    // CancellationToken in background tasks
    public class OrderNotificationWorker : BackgroundService
    {
        private readonly Channel<OrderNotification> _channel;

        public OrderNotificationWorker(Channel<OrderNotification> channel) => _channel = channel;

        protected override async Task ExecuteAsync(CancellationToken stoppingToken)
        {
            // stoppingToken is cancelled on app shutdown
            await foreach (var notification in _channel.Reader.ReadAllAsync(stoppingToken))
            {
                try
                {
                    await SendNotificationAsync(notification, stoppingToken);
                }
                catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested)
                {
                    // Shutting down — process remaining items after completing current
                    break;
                }
            }

            // Drain remaining items on graceful shutdown (with a short timeout)
            using var drainCts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
            while (_channel.Reader.TryRead(out var remaining))
            {
                try { await SendNotificationAsync(remaining, drainCts.Token); }
                catch { break; }
            }
        }

        private Task SendNotificationAsync(OrderNotification n, CancellationToken ct) => Task.CompletedTask;
    }

    private static Task DoWorkAsync(CancellationToken ct) => Task.CompletedTask;
    private static Task<ProcessedItem> ProcessItemAsync(RawItem item, CancellationToken ct) => Task.FromResult(new ProcessedItem());
}
```

---

## Step 2078: Interlocked & Lock-Free Data Structures

```csharp
// Lock-free counter using Interlocked
public class LockFreeCounter
{
    private long _value;

    public long Increment() => Interlocked.Increment(ref _value);
    public long Decrement() => Interlocked.Decrement(ref _value);
    public long Add(long amount) => Interlocked.Add(ref _value, amount);
    public long Value => Interlocked.Read(ref _value);

    // Compare-and-swap — only update if current value matches expected
    public bool TrySetIfEquals(long expected, long newValue) =>
        Interlocked.CompareExchange(ref _value, newValue, expected) == expected;
}

// Lock-free lazy initialization
public class LockFreeLazy<T>
{
    private T? _value;
    private int _initialized;

    public T Value
    {
        get
        {
            if (Interlocked.CompareExchange(ref _initialized, 1, 0) == 0)
                _value = Initialize();

            // Ensure visibility after initialization
            Thread.MemoryBarrier();
            return _value!;
        }
    }

    protected virtual T Initialize() => default!;
}

// ConcurrentQueue for FIFO operations
public class AsyncWorkQueue<T>
{
    private readonly ConcurrentQueue<T> _queue = new();
    private readonly SemaphoreSlim _available = new(0);

    public void Enqueue(T item)
    {
        _queue.Enqueue(item);
        _available.Release();
    }

    public async Task<T> DequeueAsync(CancellationToken ct = default)
    {
        await _available.WaitAsync(ct);

        if (_queue.TryDequeue(out var item))
            return item;

        throw new InvalidOperationException("Queue in inconsistent state");
    }
}

// ImmutableList for thread-safe collections
public class EventStore<T>
{
    private volatile ImmutableList<T> _events = ImmutableList<T>.Empty;

    public void Append(T @event)
    {
        ImmutableList<T> current, updated;
        do
        {
            current = _events;
            updated = current.Add(@event);
        }
        while (Interlocked.CompareExchange(ref _events, updated, current) != current);
    }

    public IReadOnlyList<T> GetAll() => _events;
}
```

---

## Step 2079: ValueTask Optimization

```csharp
// ValueTask for hot paths that often complete synchronously
public class OptimizedCacheService
{
    private readonly Dictionary<Guid, CachedItem> _hotCache = new();
    private readonly IDistributedCache _coldCache;

    public OptimizedCacheService(IDistributedCache coldCache) => _coldCache = coldCache;

    // ValueTask — returns synchronously when cache hit, async on miss
    public ValueTask<Product?> GetProductAsync(Guid id, CancellationToken ct = default)
    {
        // Hot path: synchronous in-process cache
        if (_hotCache.TryGetValue(id, out var cached) && !cached.IsExpired)
            return ValueTask.FromResult<Product?>(cached.Product);

        // Cold path: async distributed cache lookup
        return GetFromColdCacheAsync(id, ct);
    }

    private async ValueTask<Product?> GetFromColdCacheAsync(Guid id, CancellationToken ct)
    {
        var bytes = await _coldCache.GetAsync($"product:{id}", ct);
        if (bytes is null) return null;

        var product = JsonSerializer.Deserialize<Product>(bytes);

        // Populate hot cache
        if (product is not null)
            _hotCache[id] = new CachedItem(product, DateTimeOffset.UtcNow.AddSeconds(30));

        return product;
    }

    private record CachedItem(Product Product, DateTimeOffset ExpiresAt)
    {
        public bool IsExpired => DateTimeOffset.UtcNow > ExpiresAt;
    }
}

// DO: use ValueTask for IAsyncEnumerable
public async IAsyncEnumerable<Order> StreamOrdersAsync(
    Guid customerId,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    await foreach (var batch in FetchBatchesAsync(customerId, ct))
        foreach (var order in batch)
            yield return order;
}

// AVOID: ValueTask pitfalls
// ❌ Don't await ValueTask multiple times (undefined behavior)
// ❌ Don't store ValueTask in a variable and await later from multiple threads
// ✓ If you need to await multiple times, convert: await task.AsTask()
```

---

## Step 2080: Thread-Local Storage & AsyncLocal

```csharp
// ThreadLocal — per-thread state (use sparingly)
public class ThreadLocalRandom
{
    private static readonly ThreadLocal<Random> _random =
        new(() => new Random(Environment.CurrentManagedThreadId));

    public static int Next(int minValue, int maxValue) =>
        _random.Value!.Next(minValue, maxValue);

    public static double NextDouble() =>
        _random.Value!.NextDouble();
}

// AsyncLocal — flows through async continuations
public class CorrelationContext
{
    private static readonly AsyncLocal<string?> _correlationId = new();

    public static string? Current
    {
        get => _correlationId.Value;
        set => _correlationId.Value = value;
    }

    public static IDisposable Set(string correlationId)
    {
        var previous = _correlationId.Value;
        _correlationId.Value = correlationId;
        return new Disposable(() => _correlationId.Value = previous);
    }
}

// Middleware to propagate correlation ID through async context
public class CorrelationIdMiddleware
{
    private readonly RequestDelegate _next;

    public CorrelationIdMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(HttpContext ctx)
    {
        var correlationId = ctx.Request.Headers["X-Correlation-ID"].FirstOrDefault()
                           ?? Guid.NewGuid().ToString("N");

        ctx.Response.Headers["X-Correlation-ID"] = correlationId;

        using (CorrelationContext.Set(correlationId))
        {
            await _next(ctx);
        }
    }
}

// Usage — accessible anywhere in the async call chain
public class OrderService
{
    private readonly ILogger<OrderService> _logger;

    public OrderService(ILogger<OrderService> logger) => _logger = logger;

    public async Task ProcessAsync(Guid orderId, CancellationToken ct)
    {
        // CorrelationContext.Current flows through all awaits
        _logger.LogInformation("[{CorrelationId}] Processing order {OrderId}",
            CorrelationContext.Current, orderId);

        await Task.Delay(100, ct); // even after await, context is preserved

        _logger.LogInformation("[{CorrelationId}] Order {OrderId} processed",
            CorrelationContext.Current, orderId);
    }
}
```

---

## Step 2081: Reactive Extensions (Rx.NET)

```csharp
// Rx.NET for event streams
// dotnet add package System.Reactive

public class OrderStreamService
{
    private readonly Subject<OrderEvent> _events = new();

    public IObservable<OrderEvent> OrderEvents => _events.AsObservable();

    public void PublishEvent(OrderEvent @event) => _events.OnNext(@event);

    // Stream processing
    public IObservable<OrderAlert> GetHighValueAlerts(decimal threshold)
    {
        return OrderEvents
            .Where(e => e is OrderPlacedEvent)
            .Cast<OrderPlacedEvent>()
            .Where(e => e.TotalAmount >= threshold)
            .Select(e => new OrderAlert(e.OrderId, e.TotalAmount, "High-value order"));
    }

    // Rate limiting — max 10 events per second
    public IObservable<OrderEvent> GetThrottledStream() =>
        OrderEvents.Sample(TimeSpan.FromSeconds(1));

    // Sliding window analytics
    public IObservable<OrderMetrics> GetSlidingWindowMetrics() =>
        OrderEvents
            .Where(e => e is OrderPlacedEvent)
            .Cast<OrderPlacedEvent>()
            .Buffer(TimeSpan.FromMinutes(1))
            .Select(window => new OrderMetrics(
                Count:   window.Count,
                Revenue: window.Sum(e => e.TotalAmount)));

    // Combine streams
    public IObservable<EnrichedOrder> GetEnrichedOrders(IObservable<CustomerData> customers)
    {
        return Observable.Zip(
            OrderEvents.Where(e => e is OrderPlacedEvent).Cast<OrderPlacedEvent>(),
            customers,
            (order, customer) => new EnrichedOrder(order.OrderId, customer.Name, order.TotalAmount));
    }
}

// Subscribe and bridge to channel
public class RxToBridgeService : IHostedService
{
    private readonly OrderStreamService _stream;
    private readonly ChannelWriter<OrderAlert> _alertWriter;
    private IDisposable? _subscription;

    public RxToBridgeService(OrderStreamService stream, ChannelWriter<OrderAlert> alertWriter)
    {
        _stream      = stream;
        _alertWriter = alertWriter;
    }

    public Task StartAsync(CancellationToken ct)
    {
        _subscription = _stream
            .GetHighValueAlerts(threshold: 10_000m)
            .Subscribe(alert => _alertWriter.TryWrite(alert));

        return Task.CompletedTask;
    }

    public Task StopAsync(CancellationToken ct)
    {
        _subscription?.Dispose();
        return Task.CompletedTask;
    }
}
```

---

## Step 2082: WaitGroup / Task Coordination

```csharp
// Coordinating multiple concurrent tasks
public class TaskCoordinator
{
    // Fan-out + fan-in: run many tasks, collect all results
    public async Task<IReadOnlyList<TResult>> FanOutAsync<TInput, TResult>(
        IEnumerable<TInput> items,
        Func<TInput, CancellationToken, Task<TResult>> worker,
        int maxConcurrency = Environment.ProcessorCount,
        CancellationToken ct = default)
    {
        var semaphore = new SemaphoreSlim(maxConcurrency, maxConcurrency);
        var results   = new ConcurrentBag<TResult>();

        var tasks = items.Select(async item =>
        {
            await semaphore.WaitAsync(ct);
            try
            {
                var result = await worker(item, ct);
                results.Add(result);
            }
            finally
            {
                semaphore.Release();
            }
        });

        await Task.WhenAll(tasks);
        return results.ToList();
    }

    // WhenAny with fallback
    public async Task<T> WithFallbackAsync<T>(
        Task<T> primary,
        Task<T> fallback,
        TimeSpan primaryTimeout)
    {
        using var timeoutCts = new CancellationTokenSource(primaryTimeout);

        var timeoutTask = Task.Delay(primaryTimeout, timeoutCts.Token)
                              .ContinueWith(_ => default(T)!, CancellationToken.None);

        var winner = await Task.WhenAny(primary, timeoutTask);

        if (winner == primary && !primary.IsFaulted)
        {
            timeoutCts.Cancel();
            return await primary;
        }

        return await fallback;
    }

    // Conditional completion with TaskCompletionSource
    public async Task<bool> WaitForConditionAsync(
        Func<bool> condition,
        TimeSpan timeout,
        CancellationToken ct = default)
    {
        var tcs = new TaskCompletionSource<bool>(TaskCreationOptions.RunContinuationsAsynchronously);

        using var timer = new System.Timers.Timer(interval: 500) { AutoReset = true };
        timer.Elapsed += (_, _) =>
        {
            if (condition())
                tcs.TrySetResult(true);
        };

        ct.Register(() => tcs.TrySetCanceled(ct));

        using var timeoutCts = new CancellationTokenSource(timeout);
        timeoutCts.Token.Register(() => tcs.TrySetResult(false));

        timer.Start();
        return await tcs.Task;
    }
}
```

---

## Step 2083: Concurrent Collections Deep Dive

```csharp
// Thread-safe collection comparison
public class ConcurrentCollectionExamples
{
    // ConcurrentDictionary — most versatile
    private readonly ConcurrentDictionary<Guid, Order> _orders = new();

    public void AddOrUpdateOrder(Order order)
    {
        _orders.AddOrUpdate(
            key:            order.Id,
            addValueFactory:    id => order,
            updateValueFactory: (id, existing) => order.Version > existing.Version ? order : existing);
    }

    public Order GetOrAddOrder(Guid id, Func<Guid, Order> factory)
        => _orders.GetOrAdd(id, factory);

    // ConcurrentBag — unordered, good for pools
    private readonly ConcurrentBag<byte[]> _bufferPool = new();

    public byte[] RentBuffer()
    {
        if (_bufferPool.TryTake(out var buffer))
            return buffer;

        return new byte[4096]; // allocate new if pool empty
    }

    public void ReturnBuffer(byte[] buffer)
    {
        if (_bufferPool.Count < 100) // cap pool size
            _bufferPool.Add(buffer);
    }

    // ConcurrentStack — LIFO
    private readonly ConcurrentStack<WorkItem> _workStack = new();

    // ConcurrentQueue — FIFO
    private readonly ConcurrentQueue<WorkItem> _workQueue = new();

    // BlockingCollection — bounded with blocking
    private readonly BlockingCollection<WorkItem> _bounded = new(boundedCapacity: 1000);

    public void ProduceBlocking(WorkItem item) => _bounded.Add(item); // blocks if full

    public IEnumerable<WorkItem> ConsumeAllBlocking()
    {
        foreach (var item in _bounded.GetConsumingEnumerable()) // blocks until items available
            yield return item;
    }
}
```

---

## Step 2084: Complete Concurrency Architecture

```csharp
// Production-ready concurrent order processing service
public class ConcurrentOrderProcessor : IHostedService
{
    private readonly Channel<OrderProcessingRequest> _channel;
    private readonly IServiceProvider _sp;
    private readonly ILogger<ConcurrentOrderProcessor> _logger;
    private readonly CancellationTokenSource _cts = new();
    private readonly List<Task> _workers = [];

    private const int WorkerCount  = 4;
    private const int ChannelBound = 5000;

    public ConcurrentOrderProcessor(IServiceProvider sp, ILogger<ConcurrentOrderProcessor> logger)
    {
        _sp      = sp;
        _logger  = logger;
        _channel = Channel.CreateBounded<OrderProcessingRequest>(new BoundedChannelOptions(ChannelBound)
        {
            FullMode     = BoundedChannelFullMode.Wait,
            SingleWriter = false,
            SingleReader = false
        });
    }

    public async Task EnqueueAsync(OrderProcessingRequest request, CancellationToken ct = default)
        => await _channel.Writer.WriteAsync(request, ct);

    public Task StartAsync(CancellationToken ct)
    {
        for (var i = 0; i < WorkerCount; i++)
        {
            var workerId = i;
            _workers.Add(Task.Run(() => WorkerLoopAsync(workerId, _cts.Token), _cts.Token));
        }

        _logger.LogInformation("Started {WorkerCount} order processing workers", WorkerCount);
        return Task.CompletedTask;
    }

    public async Task StopAsync(CancellationToken ct)
    {
        _channel.Writer.Complete();
        _cts.Cancel();

        try
        {
            await Task.WhenAll(_workers).WaitAsync(TimeSpan.FromSeconds(30), ct);
        }
        catch (TimeoutException)
        {
            _logger.LogWarning("Workers did not stop within 30 seconds");
        }
    }

    private async Task WorkerLoopAsync(int workerId, CancellationToken ct)
    {
        _logger.LogDebug("Worker {WorkerId} started", workerId);

        await foreach (var request in _channel.Reader.ReadAllAsync(ct))
        {
            await using var scope = _sp.CreateAsyncScope();
            var service = scope.ServiceProvider.GetRequiredService<IOrderService>();

            try
            {
                await service.ProcessAsync(request.OrderId, ct);
            }
            catch (Exception ex) when (!ct.IsCancellationRequested)
            {
                _logger.LogError(ex, "Worker {WorkerId} failed on order {OrderId}", workerId, request.OrderId);
            }
        }

        _logger.LogDebug("Worker {WorkerId} stopped", workerId);
    }
}

// Register
builder.Services.AddSingleton<ConcurrentOrderProcessor>();
builder.Services.AddHostedService(sp => sp.GetRequiredService<ConcurrentOrderProcessor>());

// Inject and use
app.MapPost("/api/orders/{id}/process", async (
    Guid id,
    ConcurrentOrderProcessor processor) =>
{
    await processor.EnqueueAsync(new OrderProcessingRequest(id));
    return Results.Accepted();
}).RequireAuthorization();
```

**Summary**: Part 90 covers System.Threading.Channels for high-throughput producer-consumer pipelines, multi-stage channel pipelines with fan-out, TPL Dataflow for complex branching pipelines, Microsoft Orleans virtual actor model (grains, state, timers, reminders, streams), SemaphoreSlim and per-key async locking, Parallel.ForEachAsync, comprehensive CancellationToken patterns, lock-free data structures with Interlocked, ValueTask optimization, AsyncLocal for context propagation, Rx.NET observable streams, task coordination patterns (fan-out/fan-in, WhenAny with fallback), concurrent collections deep-dive, and a complete concurrent order processing architecture. Steps 2069–2084 complete.
