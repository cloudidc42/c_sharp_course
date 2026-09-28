# Part 76: Background Workers & Scheduled Jobs — IHostedService, Worker Service, Hangfire & Quartz.NET

## Steps 1845–1860 | Production Background Processing

---

## Step 1845: IHostedService — The Foundation

```csharp
// The simplest background service: implement IHostedService directly
public sealed class OutboxProcessorService : IHostedService, IDisposable
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<OutboxProcessorService> _logger;
    private Timer? _timer;
    private int _executionCount;

    public OutboxProcessorService(
        IServiceScopeFactory scopeFactory,
        ILogger<OutboxProcessorService> logger)
    {
        _scopeFactory = scopeFactory;
        _logger = logger;
    }

    public Task StartAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Outbox Processor starting.");
        _timer = new Timer(DoWork, null, TimeSpan.Zero, TimeSpan.FromSeconds(10));
        return Task.CompletedTask;
    }

    private async void DoWork(object? state)
    {
        var count = Interlocked.Increment(ref _executionCount);
        _logger.LogDebug("Outbox Processor executing ({Count})", count);

        try
        {
            await using var scope = _scopeFactory.CreateAsyncScope();
            var processor = scope.ServiceProvider.GetRequiredService<IOutboxProcessor>();
            await processor.ProcessPendingMessagesAsync(CancellationToken.None);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Outbox processor error.");
        }
    }

    public Task StopAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Outbox Processor stopping.");
        _timer?.Change(Timeout.Infinite, 0);
        return Task.CompletedTask;
    }

    public void Dispose() => _timer?.Dispose();
}
```

---

## Step 1846: BackgroundService — Cleaner Long-Running Services

```csharp
// BackgroundService is the recommended base class for long-running services
public sealed class OrderExpirationService : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<OrderExpirationService> _logger;

    public OrderExpirationService(
        IServiceScopeFactory scopeFactory,
        ILogger<OrderExpirationService> logger)
    {
        _scopeFactory = scopeFactory;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Order Expiration Service started.");

        // Run immediately, then every hour
        await ExpireOldOrdersAsync(stoppingToken);

        using var timer = new PeriodicTimer(TimeSpan.FromHours(1));

        while (await timer.WaitForNextTickAsync(stoppingToken))
        {
            await ExpireOldOrdersAsync(stoppingToken);
        }
    }

    private async Task ExpireOldOrdersAsync(CancellationToken ct)
    {
        _logger.LogInformation("Checking for orders to expire...");

        try
        {
            await using var scope = _scopeFactory.CreateAsyncScope();
            var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();

            var cutoff = DateTime.UtcNow.AddDays(-7);

            var expiredCount = await db.Orders
                .Where(o => o.Status == "Submitted" && o.CreatedAt < cutoff)
                .ExecuteUpdateAsync(
                    s => s.SetProperty(o => o.Status, "Expired"),
                    ct);

            if (expiredCount > 0)
                _logger.LogInformation("Expired {Count} orders", expiredCount);
        }
        catch (Exception ex) when (!ct.IsCancellationRequested)
        {
            _logger.LogError(ex, "Failed to expire orders.");
            // Don't rethrow — service should stay running
        }
    }
}

// Registration
builder.Services.AddHostedService<OrderExpirationService>();
```

---

## Step 1847: Worker Service Project Template

```bash
# Worker Service is a console app with DI + BackgroundService
dotnet new worker -n MyApp.Worker -f net9.0
```

```xml
<!-- MyApp.Worker.csproj -->
<Project Sdk="Microsoft.NET.Sdk.Worker">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net9.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <UserSecretsId>aspnet-MyApp.Worker-8fcd2c31</UserSecretsId>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.Extensions.Hosting" Version="9.0.0" />
    <PackageReference Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="9.0.0" />
    <PackageReference Include="MassTransit.RabbitMQ" Version="8.3.0" />
    <PackageReference Include="Serilog.Extensions.Hosting" Version="8.0.0" />
    <PackageReference Include="OpenTelemetry.Exporter.OpenTelemetryProtocol" Version="1.9.0" />
  </ItemGroup>

</Project>
```

```csharp
// Program.cs — Worker Service
var builder = Host.CreateApplicationBuilder(args);

// Services
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseNpgsql(builder.Configuration.GetConnectionString("DefaultConnection")));

builder.Services.AddMassTransit(mt =>
{
    mt.AddConsumer<OrderPlacedConsumer>();
    mt.UsingRabbitMq((ctx, cfg) =>
    {
        cfg.Host(builder.Configuration["RabbitMQ:Host"], "/", h =>
        {
            h.Username(builder.Configuration["RabbitMQ:Username"]!);
            h.Password(builder.Configuration["RabbitMQ:Password"]!);
        });
        cfg.ConfigureEndpoints(ctx);
    });
});

// Background services
builder.Services.AddHostedService<OutboxProcessorService>();
builder.Services.AddHostedService<OrderExpirationService>();
builder.Services.AddHostedService<MetricsReportingService>();

// Observability
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddSource("MyApp.Worker")
        .AddOtlpExporter())
    .WithMetrics(metrics => metrics
        .AddMeter("MyApp.Worker")
        .AddOtlpExporter());

// Logging
builder.Logging.AddSerilog(new LoggerConfiguration()
    .ReadFrom.Configuration(builder.Configuration)
    .CreateLogger());

var host = builder.Build();

// Graceful shutdown timeout
host.Services.GetRequiredService<IHostApplicationLifetime>()
    .ApplicationStopping.Register(() =>
    {
        Log.Information("Worker service is shutting down...");
    });

await host.RunAsync();
```

---

## Step 1848: Channel-Based Queue (In-Process)

```csharp
// Fast in-process queue using System.Threading.Channels
// Good for: fire-and-forget within one process, email sending, metrics aggregation

// IBackgroundTaskQueue.cs
public interface IBackgroundTaskQueue
{
    ValueTask EnqueueAsync(Func<CancellationToken, ValueTask> workItem);
    ValueTask<Func<CancellationToken, ValueTask>> DequeueAsync(CancellationToken ct);
}

// BackgroundTaskQueue.cs
public sealed class BackgroundTaskQueue : IBackgroundTaskQueue
{
    private readonly Channel<Func<CancellationToken, ValueTask>> _queue;

    public BackgroundTaskQueue(int capacity = 100)
    {
        _queue = Channel.CreateBounded<Func<CancellationToken, ValueTask>>(
            new BoundedChannelOptions(capacity)
            {
                FullMode = BoundedChannelFullMode.Wait,
                SingleReader = true,   // Only one consumer
                SingleWriter = false   // Many producers
            });
    }

    public async ValueTask EnqueueAsync(Func<CancellationToken, ValueTask> workItem)
    {
        ArgumentNullException.ThrowIfNull(workItem);
        await _queue.Writer.WriteAsync(workItem);
    }

    public async ValueTask<Func<CancellationToken, ValueTask>> DequeueAsync(CancellationToken ct)
        => await _queue.Reader.ReadAsync(ct);
}

// QueuedHostedService.cs — processes items from the queue
public sealed class QueuedHostedService : BackgroundService
{
    private readonly IBackgroundTaskQueue _taskQueue;
    private readonly ILogger<QueuedHostedService> _logger;

    public QueuedHostedService(
        IBackgroundTaskQueue taskQueue,
        ILogger<QueuedHostedService> logger)
    {
        _taskQueue = taskQueue;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Queued Hosted Service running.");

        await BackgroundProcessing(stoppingToken);

        _logger.LogInformation("Queued Hosted Service stopped.");
    }

    private async Task BackgroundProcessing(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            var workItem = await _taskQueue.DequeueAsync(stoppingToken);

            try
            {
                await workItem(stoppingToken);
            }
            catch (OperationCanceledException)
            {
                // Ignore — application is shutting down
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error occurred executing background work item.");
            }
        }
    }
}

// Usage in a controller
[HttpPost("orders/{id}/notify")]
public async Task<IActionResult> NotifyOrder(Guid id, CancellationToken ct)
{
    var order = await _db.Orders.FindAsync([id], ct);
    if (order is null) return NotFound();

    // Enqueue — returns immediately, doesn't block the HTTP response
    await _taskQueue.EnqueueAsync(async stoppingToken =>
    {
        await _emailService.SendOrderNotificationAsync(order.CustomerEmail, id, stoppingToken);
    });

    return Accepted();
}
```

---

## Step 1849: Hangfire — Reliable Background Jobs

```bash
dotnet add package Hangfire.AspNetCore
dotnet add package Hangfire.PostgreSql  # or SqlServer, Redis
```

```csharp
// Program.cs — Hangfire setup
builder.Services.AddHangfire(config => config
    .SetDataCompatibilityLevel(CompatibilityLevel.Version_180)
    .UseSimpleAssemblyNameTypeSerializer()
    .UseRecommendedSerializerSettings()
    .UsePostgreSqlStorage(builder.Configuration.GetConnectionString("DefaultConnection"),
        new PostgreSqlStorageOptions
        {
            QueuePollInterval = TimeSpan.FromSeconds(5),
            InvisibilityTimeout = TimeSpan.FromMinutes(30),
            DistributedLockTimeout = TimeSpan.FromMinutes(10),
            PrepareSchemaIfNecessary = true
        }));

builder.Services.AddHangfireServer(options =>
{
    options.WorkerCount = 10;  // Concurrent job workers
    options.Queues = new[] { "critical", "default", "low" };
});

// In the pipeline:
app.UseHangfireDashboard("/jobs", new DashboardOptions
{
    Authorization = [new HangfireAuthorizationFilter()], // Protect dashboard
    ReadOnlyAuthorizationFilters = [new HangfireReadOnlyFilter()]
});
```

```csharp
// Job classes — any class with methods
public sealed class ReportGenerationJob
{
    private readonly AppDbContext _db;
    private readonly IStorageService _storage;
    private readonly IEmailService _email;

    public ReportGenerationJob(
        AppDbContext db,
        IStorageService storage,
        IEmailService email)
    {
        _db = db;
        _storage = storage;
        _email = email;
    }

    [AutomaticRetry(Attempts = 3, DelaysInSeconds = [60, 300, 900])]
    [JobDisplayName("Generate Monthly Report — {0}")]
    public async Task GenerateMonthlyReportAsync(string tenantSlug, CancellationToken ct)
    {
        var tenant = await _db.Tenants.FirstOrDefaultAsync(t => t.Slug == tenantSlug, ct)
            ?? throw new InvalidOperationException($"Tenant {tenantSlug} not found.");

        var report = await BuildReportAsync(tenant, ct);
        var url = await _storage.UploadAsync($"reports/{tenantSlug}/{DateTime.UtcNow:yyyy-MM}.pdf", report, ct);
        await _email.SendReportReadyAsync(tenant.AdminEmail, url, ct);
    }

    private async Task<byte[]> BuildReportAsync(TenantRecord tenant, CancellationToken ct)
    {
        // Build PDF report...
        await Task.Delay(100, ct); // placeholder
        return Array.Empty<byte>();
    }
}

// Enqueuing jobs
public sealed class ReportController : ControllerBase
{
    private readonly IBackgroundJobClient _backgroundJobs;
    private readonly IRecurringJobManager _recurringJobs;

    public ReportController(
        IBackgroundJobClient backgroundJobs,
        IRecurringJobManager recurringJobs)
    {
        _backgroundJobs = backgroundJobs;
        _recurringJobs = recurringJobs;
    }

    // Fire-and-forget: runs as soon as a worker is available
    [HttpPost("reports/generate/{tenantSlug}")]
    public IActionResult TriggerReport(string tenantSlug)
    {
        var jobId = _backgroundJobs.Enqueue<ReportGenerationJob>(
            j => j.GenerateMonthlyReportAsync(tenantSlug, CancellationToken.None));

        return Accepted(new { jobId });
    }

    // Delayed job: runs after a delay
    [HttpPost("orders/{id}/remind")]
    public IActionResult ScheduleReminder(Guid id)
    {
        var jobId = _backgroundJobs.Schedule<ReminderJob>(
            j => j.SendOrderReminderAsync(id, CancellationToken.None),
            TimeSpan.FromHours(24));

        return Accepted(new { jobId });
    }

    // Recurring job: CRON expression
    [HttpPost("reports/schedule-monthly")]
    public IActionResult ScheduleMonthlyReport()
    {
        _recurringJobs.AddOrUpdate<ReportGenerationJob>(
            recurringJobId: "monthly-report-all-tenants",
            job: j => j.GenerateMonthlyReportAsync("all", CancellationToken.None),
            cronExpression: Cron.Monthly(1, 3));  // 1st of month, 3am

        return Ok("Monthly report job scheduled.");
    }

    // Continuations: chain jobs
    [HttpPost("orders/{id}/process")]
    public IActionResult ProcessOrder(Guid id)
    {
        var jobId = _backgroundJobs.Enqueue<OrderValidationJob>(
            j => j.ValidateAsync(id, CancellationToken.None));

        _backgroundJobs.ContinueJobWith<PaymentJob>(
            jobId,
            j => j.ChargeAsync(id, CancellationToken.None));

        _backgroundJobs.ContinueJobWith<FulfillmentJob>(
            jobId,
            j => j.FulfillAsync(id, CancellationToken.None));

        return Accepted(new { jobId });
    }
}
```

---

## Step 1850: Quartz.NET — Enterprise Scheduling

```bash
dotnet add package Quartz.AspNetCore
dotnet add package Quartz.Serialization.Json
dotnet add package Quartz.Extensions.Hosting
```

```csharp
// Program.cs — Quartz setup
builder.Services.AddQuartz(q =>
{
    // Use DI for job instances
    q.UseMicrosoftDependencyInjectionJobFactory();

    // Job store: persistent (PostgreSQL) or in-memory
    q.UsePersistentStore(store =>
    {
        store.UsePostgres(builder.Configuration.GetConnectionString("DefaultConnection")!);
        store.UseJsonSerializer();
        store.PerformSchemaValidation = true;
    });

    // Clustering: multiple instances share the same job store
    q.SchedulerName = "MyApp Scheduler";
    q.SchedulerId = "AUTO"; // Unique per instance

    // Daily report job
    var dailyReportJob = new JobKey("DailyReportJob", "Reports");
    q.AddJob<DailyReportJob>(j => j
        .WithIdentity(dailyReportJob)
        .WithDescription("Generates daily summary reports")
        .StoreDurably()  // Keep job even without triggers
    );
    q.AddTrigger(t => t
        .ForJob(dailyReportJob)
        .WithIdentity("DailyReportTrigger", "Reports")
        .WithCronSchedule("0 0 6 * * ?",         // Every day at 6:00 AM
            x => x.InTimeZone(TimeZoneInfo.FindSystemTimeZoneById("UTC")))
        .WithDescription("Fires daily at 6am UTC")
    );

    // Cleanup job
    var cleanupJob = new JobKey("DataCleanupJob", "Maintenance");
    q.AddJob<DataCleanupJob>(j => j.WithIdentity(cleanupJob));
    q.AddTrigger(t => t
        .ForJob(cleanupJob)
        .WithIdentity("CleanupTrigger", "Maintenance")
        .WithSimpleSchedule(s => s
            .WithIntervalInHours(6)
            .RepeatForever())
        .StartNow()
    );
});

builder.Services.AddQuartzHostedService(options =>
{
    options.WaitForJobsToComplete = true; // Graceful shutdown
    options.AwaitApplicationStarted = true;
});
```

```csharp
// Job implementation
[DisallowConcurrentExecution]  // Only one instance runs at a time
[PersistJobDataAfterExecution] // Save job data between runs
public sealed class DailyReportJob : IJob
{
    private readonly AppDbContext _db;
    private readonly IEmailService _email;
    private readonly ILogger<DailyReportJob> _logger;

    public DailyReportJob(AppDbContext db, IEmailService email, ILogger<DailyReportJob> logger)
    {
        _db = db;
        _email = email;
        _logger = logger;
    }

    public async Task Execute(IJobExecutionContext context)
    {
        var ct = context.CancellationToken;
        _logger.LogInformation("Daily report job started. FireTime={FireTime}", context.FireTimeUtc);

        try
        {
            // Access job data map for dynamic config
            var reportDate = context.MergedJobDataMap.GetDateTime("ReportDate");
            if (reportDate == DateTime.MinValue)
                reportDate = DateTime.UtcNow.Date.AddDays(-1);

            var stats = await GetDailyStatsAsync(reportDate, ct);

            foreach (var adminEmail in stats.AdminEmails)
                await _email.SendDailyReportAsync(adminEmail, stats, ct);

            // Store result in job data
            context.JobDetail.JobDataMap["LastRunStats"] = JsonSerializer.Serialize(stats);

            _logger.LogInformation(
                "Daily report sent to {Count} admins", stats.AdminEmails.Count);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Daily report job failed.");
            // Wrap in JobExecutionException to control retry behavior
            throw new JobExecutionException(msg: "Daily report failed", cause: ex, refireImmediately: false)
            {
                UnscheduleAllTriggers = false // Keep the trigger, retry tomorrow
            };
        }
    }

    private async Task<DailyStats> GetDailyStatsAsync(DateTime date, CancellationToken ct)
    {
        var orders = await _db.Orders
            .Where(o => o.CreatedAt.Date == date.Date)
            .GroupBy(_ => 1)
            .Select(g => new { Count = g.Count(), Total = g.Sum(o => o.TotalAmount) })
            .FirstOrDefaultAsync(ct);

        var adminEmails = await _db.Tenants
            .Select(t => t.AdminEmail)
            .ToListAsync(ct);

        return new DailyStats(date, orders?.Count ?? 0, orders?.Total ?? 0, adminEmails);
    }
}

public record DailyStats(
    DateTime Date,
    int OrderCount,
    decimal TotalRevenue,
    List<string> AdminEmails);
```

---

## Step 1851: Outbox Pattern — Reliable Message Publishing

```csharp
// The Outbox pattern: write message to DB in same transaction as domain change
// A background service polls the outbox and publishes to the message bus
// Guarantees exactly-once publishing even if the app crashes

// Domain/Outbox/OutboxMessage.cs
public sealed class OutboxMessage
{
    public Guid Id { get; init; } = Guid.NewGuid();
    public string Type { get; init; } = string.Empty;
    public string Payload { get; init; } = string.Empty;
    public DateTime CreatedAt { get; init; } = DateTime.UtcNow;
    public DateTime? ProcessedAt { get; set; }
    public OutboxMessageStatus Status { get; set; } = OutboxMessageStatus.Pending;
    public string? Error { get; set; }
    public int RetryCount { get; set; }
}

public enum OutboxMessageStatus { Pending, Processed, Failed }

// Infrastructure/Outbox/OutboxProcessor.cs
public sealed class OutboxProcessor : IOutboxProcessor
{
    private static readonly JsonSerializerOptions _jsonOptions = new(JsonSerializerDefaults.Web);
    private readonly AppDbContext _db;
    private readonly IPublishEndpoint _bus;
    private readonly ILogger<OutboxProcessor> _logger;

    public OutboxProcessor(AppDbContext db, IPublishEndpoint bus, ILogger<OutboxProcessor> logger)
    {
        _db = db;
        _bus = bus;
        _logger = logger;
    }

    public async Task ProcessPendingMessagesAsync(CancellationToken ct)
    {
        // Process in batches of 50
        var messages = await _db.OutboxMessages
            .Where(m => m.Status == OutboxMessageStatus.Pending && m.RetryCount < 5)
            .OrderBy(m => m.CreatedAt)
            .Take(50)
            .ToListAsync(ct);

        if (!messages.Any()) return;

        _logger.LogDebug("Processing {Count} outbox messages", messages.Count);

        foreach (var message in messages)
        {
            await ProcessMessageAsync(message, ct);
        }

        await _db.SaveChangesAsync(ct);
    }

    private async Task ProcessMessageAsync(OutboxMessage message, CancellationToken ct)
    {
        try
        {
            var eventType = Type.GetType(message.Type)
                ?? throw new InvalidOperationException($"Unknown type: {message.Type}");

            var @event = JsonSerializer.Deserialize(message.Payload, eventType, _jsonOptions)
                ?? throw new InvalidOperationException($"Could not deserialize: {message.Payload}");

            await _bus.Publish(@event, eventType, ct);

            message.Status = OutboxMessageStatus.Processed;
            message.ProcessedAt = DateTime.UtcNow;

            _logger.LogDebug("Outbox message {Id} published.", message.Id);
        }
        catch (Exception ex)
        {
            message.RetryCount++;
            message.Error = ex.Message;

            if (message.RetryCount >= 5)
            {
                message.Status = OutboxMessageStatus.Failed;
                _logger.LogError(ex, "Outbox message {Id} failed permanently.", message.Id);
            }
            else
            {
                _logger.LogWarning(ex, "Outbox message {Id} failed (retry {Count}).", message.Id, message.RetryCount);
            }
        }
    }
}

// Background service that polls the outbox
public sealed class OutboxProcessorBackgroundService : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<OutboxProcessorBackgroundService> _logger;

    public OutboxProcessorBackgroundService(
        IServiceScopeFactory scopeFactory,
        ILogger<OutboxProcessorBackgroundService> logger)
    {
        _scopeFactory = scopeFactory;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromSeconds(5));

        while (await timer.WaitForNextTickAsync(stoppingToken))
        {
            try
            {
                await using var scope = _scopeFactory.CreateAsyncScope();
                var processor = scope.ServiceProvider.GetRequiredService<IOutboxProcessor>();
                await processor.ProcessPendingMessagesAsync(stoppingToken);
            }
            catch (Exception ex) when (!stoppingToken.IsCancellationRequested)
            {
                _logger.LogError(ex, "Outbox processor background service error.");
            }
        }
    }
}
```

---

## Step 1852: Inbox Pattern — Idempotent Message Consumption

```csharp
// The Inbox pattern: record received messages to prevent duplicate processing
// If a message is delivered twice (at-least-once delivery), process it only once

// Domain/Inbox/InboxMessage.cs
public sealed class InboxMessage
{
    public Guid Id { get; init; }           // Message ID from the broker
    public string Type { get; init; } = string.Empty;
    public string Payload { get; init; } = string.Empty;
    public DateTime ReceivedAt { get; init; } = DateTime.UtcNow;
    public DateTime? ProcessedAt { get; set; }
    public bool IsProcessed => ProcessedAt.HasValue;
}

// Middleware for MassTransit consumers
public sealed class IdempotentConsumerMiddleware<T> : IFilter<ConsumeContext<T>>
    where T : class
{
    private readonly AppDbContext _db;

    public IdempotentConsumerMiddleware(AppDbContext db) => _db = db;

    public async Task Send(ConsumeContext<T> context, IPipe<ConsumeContext<T>> next)
    {
        var messageId = context.MessageId?.ToString()
            ?? throw new InvalidOperationException("MessageId is required for idempotency.");

        // Check if already processed
        var alreadyProcessed = await _db.InboxMessages
            .AnyAsync(m => m.Id == Guid.Parse(messageId) && m.ProcessedAt != null);

        if (alreadyProcessed)
        {
            // Silently skip — already handled
            return;
        }

        // Record inbox entry
        var inbox = new InboxMessage
        {
            Id = Guid.Parse(messageId),
            Type = typeof(T).AssemblyQualifiedName!,
            Payload = JsonSerializer.Serialize(context.Message)
        };
        _db.InboxMessages.Add(inbox);
        await _db.SaveChangesAsync();

        // Process the message
        await next.Send(context);

        // Mark as processed
        inbox.ProcessedAt = DateTime.UtcNow;
        await _db.SaveChangesAsync();
    }

    public void Probe(ProbeContext context) =>
        context.CreateFilterScope("IdempotentConsumerMiddleware");
}
```

---

## Step 1853: Rate-Limited Background Processing

```csharp
// Process jobs with rate limiting to avoid overwhelming downstream services
public sealed class EmailCampaignProcessor : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<EmailCampaignProcessor> _logger;

    // Rate limiter: max 100 emails per second
    private readonly RateLimiter _rateLimiter = new TokenBucketRateLimiter(
        new TokenBucketRateLimiterOptions
        {
            TokenLimit = 100,
            QueueProcessingOrder = QueueProcessingOrder.OldestFirst,
            QueueLimit = 1000,
            ReplenishmentPeriod = TimeSpan.FromSeconds(1),
            TokensPerPeriod = 100,
            AutoReplenishment = true
        });

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromMinutes(1));

        while (await timer.WaitForNextTickAsync(stoppingToken))
        {
            await ProcessPendingCampaignsAsync(stoppingToken);
        }
    }

    private async Task ProcessPendingCampaignsAsync(CancellationToken ct)
    {
        await using var scope = _scopeFactory.CreateAsyncScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        var emailService = scope.ServiceProvider.GetRequiredService<IEmailService>();

        var pendingEmails = await db.CampaignEmails
            .Where(e => e.Status == EmailStatus.Pending)
            .Take(1000)
            .ToListAsync(ct);

        foreach (var email in pendingEmails)
        {
            // Acquire rate limit token
            using var lease = await _rateLimiter.AcquireAsync(
                permitCount: 1, cancellationToken: ct);

            if (!lease.IsAcquired)
            {
                _logger.LogWarning("Rate limit exceeded, pausing...");
                await Task.Delay(TimeSpan.FromSeconds(1), ct);
                continue;
            }

            try
            {
                await emailService.SendAsync(email.To, email.Subject, email.Body, ct);
                email.Status = EmailStatus.Sent;
                email.SentAt = DateTime.UtcNow;
            }
            catch (Exception ex)
            {
                email.Status = EmailStatus.Failed;
                email.Error = ex.Message;
                _logger.LogError(ex, "Failed to send email {Id}", email.Id);
            }
        }

        await db.SaveChangesAsync(ct);
    }

    public override void Dispose()
    {
        _rateLimiter.Dispose();
        base.Dispose();
    }
}
```

---

## Step 1854: Health Checks for Background Services

```csharp
// BackgroundServiceHealthCheck.cs
public sealed class OutboxProcessorHealthCheck : IHealthCheck
{
    private readonly AppDbContext _db;

    public OutboxProcessorHealthCheck(AppDbContext db) => _db = db;

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken ct)
    {
        // Check if outbox is backed up (indicates processor is stuck)
        var pendingCount = await _db.OutboxMessages
            .Where(m => m.Status == OutboxMessageStatus.Pending &&
                        m.CreatedAt < DateTime.UtcNow.AddMinutes(-5))
            .CountAsync(ct);

        if (pendingCount > 100)
        {
            return HealthCheckResult.Unhealthy(
                $"Outbox backed up: {pendingCount} messages pending > 5 minutes");
        }

        if (pendingCount > 20)
        {
            return HealthCheckResult.Degraded(
                $"Outbox warning: {pendingCount} messages pending > 5 minutes");
        }

        var failedCount = await _db.OutboxMessages
            .Where(m => m.Status == OutboxMessageStatus.Failed)
            .CountAsync(ct);

        if (failedCount > 0)
        {
            return HealthCheckResult.Degraded(
                $"{failedCount} outbox messages permanently failed. Manual intervention required.");
        }

        return HealthCheckResult.Healthy($"Outbox healthy. {pendingCount} pending.");
    }
}

// Registration
builder.Services.AddHealthChecks()
    .AddCheck<OutboxProcessorHealthCheck>(
        "outbox",
        failureStatus: HealthStatus.Degraded,
        tags: ["background", "outbox"])
    .AddCheck<QuartzHealthCheck>("quartz", tags: ["background", "scheduler"])
    .AddNpgSql(connectionString, tags: ["database"])
    .AddRabbitMQ(rabbitConnectionString, tags: ["messaging"]);

// Kubernetes liveness / readiness
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false // Just checks if the process is alive
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("database") || check.Tags.Contains("messaging"),
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});
```

---

## Step 1855: Distributed Locking for Workers

```csharp
// Prevent multiple instances from running the same job simultaneously
// Install: dotnet add package RedLock.net or DistributedLock.Redis

public sealed class ReportGenerationService : BackgroundService
{
    private readonly IDistributedLockFactory _lockFactory;
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<ReportGenerationService> _logger;

    private const string LockKey = "report-generation-lock";
    private static readonly TimeSpan LockExpiry = TimeSpan.FromMinutes(30);

    public ReportGenerationService(
        IDistributedLockFactory lockFactory,
        IServiceScopeFactory scopeFactory,
        ILogger<ReportGenerationService> logger)
    {
        _lockFactory = lockFactory;
        _scopeFactory = scopeFactory;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromHours(1));

        while (await timer.WaitForNextTickAsync(stoppingToken))
        {
            await TryRunWithLockAsync(stoppingToken);
        }
    }

    private async Task TryRunWithLockAsync(CancellationToken ct)
    {
        // Try to acquire distributed lock (across all instances)
        await using var handle = await _lockFactory.TryAcquireAsync(LockKey, LockExpiry, ct);

        if (handle is null)
        {
            _logger.LogDebug("Could not acquire lock — another instance is running reports.");
            return;
        }

        _logger.LogInformation("Acquired report generation lock. Starting reports...");

        try
        {
            await using var scope = _scopeFactory.CreateAsyncScope();
            var service = scope.ServiceProvider.GetRequiredService<ReportGenerationJob>();
            await service.GenerateMonthlyReportAsync("all", ct);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Report generation failed.");
        }
    }
}
```

---

## Step 1856: Monitoring Background Jobs

```csharp
// Metrics for background job health
public sealed class BackgroundJobMetrics
{
    private readonly Counter<long> _jobsStarted;
    private readonly Counter<long> _jobsCompleted;
    private readonly Counter<long> _jobsFailed;
    private readonly Histogram<double> _jobDuration;

    public BackgroundJobMetrics(IMeterFactory meterFactory)
    {
        var meter = meterFactory.Create("MyApp.BackgroundJobs");

        _jobsStarted = meter.CreateCounter<long>(
            "background.jobs.started",
            description: "Total background jobs started");

        _jobsCompleted = meter.CreateCounter<long>(
            "background.jobs.completed",
            description: "Total background jobs completed successfully");

        _jobsFailed = meter.CreateCounter<long>(
            "background.jobs.failed",
            description: "Total background jobs that failed");

        _jobDuration = meter.CreateHistogram<double>(
            "background.jobs.duration",
            unit: "ms",
            description: "Background job execution time");
    }

    public void RecordStart(string jobName) =>
        _jobsStarted.Add(1, new KeyValuePair<string, object?>("job.name", jobName));

    public void RecordSuccess(string jobName, TimeSpan duration)
    {
        _jobsCompleted.Add(1, new KeyValuePair<string, object?>("job.name", jobName));
        _jobDuration.Record(duration.TotalMilliseconds, new KeyValuePair<string, object?>("job.name", jobName));
    }

    public void RecordFailure(string jobName) =>
        _jobsFailed.Add(1, new KeyValuePair<string, object?>("job.name", jobName));
}

// Usage wrapper
public sealed class InstrumentedJob
{
    private readonly BackgroundJobMetrics _metrics;
    private readonly ILogger _logger;

    public async Task RunAsync(string jobName, Func<CancellationToken, Task> job, CancellationToken ct)
    {
        _metrics.RecordStart(jobName);
        var sw = Stopwatch.StartNew();

        try
        {
            await job(ct);
            sw.Stop();
            _metrics.RecordSuccess(jobName, sw.Elapsed);
            _logger.LogInformation("{JobName} completed in {Elapsed}ms", jobName, sw.ElapsedMilliseconds);
        }
        catch (Exception ex)
        {
            _metrics.RecordFailure(jobName);
            _logger.LogError(ex, "{JobName} failed after {Elapsed}ms", jobName, sw.ElapsedMilliseconds);
            throw;
        }
    }
}
```

---

## Step 1857: Saga with Step-by-Step Progress

```csharp
// Long-running process that spans multiple steps
// Each step is a separate job that can be retried independently
public sealed class OnboardingWorkflow
{
    private readonly IBackgroundJobClient _jobs;

    public OnboardingWorkflow(IBackgroundJobClient jobs) => _jobs = jobs;

    public void Start(Guid tenantId)
    {
        // Step 1: Provision database schema
        var step1 = _jobs.Enqueue<SchemaProvisioningJob>(
            j => j.RunAsync(tenantId, CancellationToken.None));

        // Step 2: Seed initial data (after step 1 completes)
        var step2 = _jobs.ContinueJobWith<SeedDataJob>(
            step1,
            j => j.RunAsync(tenantId, CancellationToken.None));

        // Step 3: Send welcome email (after step 2 completes)
        var step3 = _jobs.ContinueJobWith<WelcomeEmailJob>(
            step2,
            j => j.RunAsync(tenantId, CancellationToken.None));

        // Step 4: Enable features (after email sent)
        _jobs.ContinueJobWith<EnableFeaturesJob>(
            step3,
            j => j.RunAsync(tenantId, CancellationToken.None));
    }
}
```

---

## Step 1858: Graceful Shutdown

```csharp
// Ensure in-flight jobs complete before the app stops
public sealed class GracefulShutdownService : IHostedService
{
    private readonly IHostApplicationLifetime _lifetime;
    private readonly ILogger<GracefulShutdownService> _logger;
    private readonly ActiveJobTracker _tracker;

    public GracefulShutdownService(
        IHostApplicationLifetime lifetime,
        ILogger<GracefulShutdownService> logger,
        ActiveJobTracker tracker)
    {
        _lifetime = lifetime;
        _logger = logger;
        _tracker = tracker;
    }

    public Task StartAsync(CancellationToken ct)
    {
        _lifetime.ApplicationStopping.Register(OnStopping);
        return Task.CompletedTask;
    }

    private void OnStopping()
    {
        _logger.LogInformation("Application stopping. Waiting for {Count} active jobs...",
            _tracker.ActiveCount);

        var timeout = TimeSpan.FromSeconds(30);
        var deadline = DateTime.UtcNow.Add(timeout);

        while (_tracker.ActiveCount > 0 && DateTime.UtcNow < deadline)
        {
            Thread.Sleep(100);
        }

        if (_tracker.ActiveCount > 0)
            _logger.LogWarning("{Count} jobs still running at shutdown.", _tracker.ActiveCount);
        else
            _logger.LogInformation("All jobs completed cleanly.");
    }

    public Task StopAsync(CancellationToken ct) => Task.CompletedTask;
}

// appsettings.json
// "WorkerOptions": {
//   "ShutdownTimeout": "00:00:30",
//   "MaxConcurrency": 10
// }
```

---

## Step 1859: Comparison — When to Use What

```
┌─────────────────────────────────────────────────────────────────────┐
│  Background Processing Decision Tree                                │
│                                                                     │
│  Simple delay/timer?                                                │
│  └─▶ BackgroundService + PeriodicTimer                              │
│                                                                     │
│  Reliable: survives crashes, at-least-once?                         │
│  └─▶ Hangfire or Quartz.NET (persisted job store)                   │
│                                                                     │
│  Event-driven, message bus?                                         │
│  └─▶ MassTransit + Outbox pattern                                   │
│                                                                     │
│  In-process, low latency (<100ms)?                                  │
│  └─▶ System.Threading.Channels + QueuedHostedService               │
│                                                                     │
│  Enterprise scheduling with CRON, clustering?                       │
│  └─▶ Quartz.NET with persistent store                               │
│                                                                     │
│  Simple dashboard, retries, continuations?                          │
│  └─▶ Hangfire                                                       │
│                                                                     │
│  Prevent duplicate processing?                                      │
│  └─▶ Inbox pattern (idempotency keys)                               │
│                                                                     │
│  Reliable integration event publishing?                             │
│  └─▶ Outbox pattern                                                 │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Step 1860: Full Configuration Summary

```csharp
// Complete Program.cs for a Worker Service with everything wired up
var builder = Host.CreateApplicationBuilder(args);

// Database
builder.Services.AddDbContext<AppDbContext>(opt =>
    opt.UseNpgsql(builder.Configuration.GetConnectionString("DefaultConnection")));

// Hangfire
builder.Services.AddHangfire(cfg => cfg
    .UsePostgreSqlStorage(builder.Configuration.GetConnectionString("DefaultConnection")));
builder.Services.AddHangfireServer(opt => opt.WorkerCount = 5);

// Quartz
builder.Services.AddQuartz(q =>
{
    q.UsePersistentStore(store =>
        store.UsePostgres(builder.Configuration.GetConnectionString("DefaultConnection")!));
    q.UseMicrosoftDependencyInjectionJobFactory();
    // Add jobs and triggers here
});
builder.Services.AddQuartzHostedService(opt => opt.WaitForJobsToComplete = true);

// Channel queue
builder.Services.AddSingleton<IBackgroundTaskQueue>(
    new BackgroundTaskQueue(capacity: 200));
builder.Services.AddHostedService<QueuedHostedService>();

// Custom background services
builder.Services.AddHostedService<OutboxProcessorBackgroundService>();
builder.Services.AddHostedService<OrderExpirationService>();

// Distributed locking (Redis)
builder.Services.AddSingleton<IDistributedLockFactory>(sp =>
    RedLockFactory.Create(
        new[] { sp.GetRequiredService<ConnectionMultiplexer>() }));

// Health checks
builder.Services.AddHealthChecks()
    .AddCheck<OutboxProcessorHealthCheck>("outbox")
    .AddNpgSql(builder.Configuration.GetConnectionString("DefaultConnection")!);

// Metrics
builder.Services.AddSingleton<BackgroundJobMetrics>();

var host = builder.Build();
await host.RunAsync();
```

---

## Summary: Background Processing Patterns

| Pattern | Tool | Persistence | Distributed | Use When |
|---|---|---|---|---|
| Simple timer | `PeriodicTimer` + `BackgroundService` | No | No | Lightweight polling |
| In-process queue | `Channel<T>` | No | No | Low-latency fire-and-forget |
| Reliable jobs | Hangfire | DB | Yes (polling) | Dashboard, retries, chains |
| Enterprise CRON | Quartz.NET | DB | Yes (clustering) | Complex schedules |
| Event-driven | MassTransit | Message broker | Yes | Microservices |
| Outbox | Custom + BackgroundService | DB | — | Reliable event publishing |
| Inbox | Custom middleware | DB | — | Idempotent consumption |

---

*Next: Part 77 — CLI Tools with System.CommandLine & Spectre.Console: Build Professional Developer Tools*
