# Part 88: Advanced Caching Strategies

## Steps 2037–2052

---

## Step 2037: Caching Fundamentals & Pattern Selection

```
Cache Patterns:
┌──────────────────────────────────────────────────────────┐
│  Cache-Aside (Lazy)    │  App checks cache first.        │
│  (most common)         │  On miss, load from DB, store.  │
│                        │  Great for read-heavy data.     │
├──────────────────────────────────────────────────────────┤
│  Read-Through          │  Cache sits in front of DB.     │
│                        │  Cache loads on miss auto.      │
│                        │  App always reads from cache.   │
├──────────────────────────────────────────────────────────┤
│  Write-Through         │  Write to cache AND DB sync.    │
│                        │  No stale data risk.            │
│                        │  Slower writes.                 │
├──────────────────────────────────────────────────────────┤
│  Write-Behind          │  Write to cache; async DB sync. │
│  (Write-Back)          │  Fast writes, risk of loss.     │
│                        │  Complex; use sparingly.        │
├──────────────────────────────────────────────────────────┤
│  Refresh-Ahead         │  Proactively refresh before     │
│                        │  expiry. Avoids cold starts.    │
│                        │  Good for predictable access.   │
└──────────────────────────────────────────────────────────┘

Cache Eviction Policies:
- LRU (Least Recently Used) — Redis default-ish
- LFU (Least Frequently Used) — Redis 4.0+
- TTL-based — most common for simple use cases
- Size-based — evict when memory threshold reached
```

---

## Step 2038: .NET 9 HybridCache — The Modern Unified Cache

HybridCache (.NET 9+) combines in-process (L1) and distributed (L2) caches in one API, with stampede protection built-in.

```bash
dotnet add package Microsoft.Extensions.Caching.Hybrid
dotnet add package Microsoft.Extensions.Caching.StackExchangeRedis
```

```csharp
// Program.cs
builder.Services.AddHybridCache(options =>
{
    // Global defaults
    options.DefaultEntryOptions = new HybridCacheEntryOptions
    {
        Expiration             = TimeSpan.FromMinutes(10),  // L2 (Redis) TTL
        LocalCacheExpiration   = TimeSpan.FromMinutes(2),   // L1 (in-process) TTL
    };

    // Max in-process cache size
    options.MaximumPayloadBytes      = 1024 * 1024;        // 1 MB per entry
    options.MaximumKeyLength         = 512;
    options.ReportTaggedMetrics      = true;
});

// Wire up Redis as the distributed (L2) backend
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration   = builder.Configuration.GetConnectionString("Redis");
    options.InstanceName    = "myapp:";
});

// Optionally wire up IDistributedCache-based L2 (already done via AddStackExchangeRedisCache)
```

```csharp
// Using HybridCache in a service
public class ProductService : IProductService
{
    private readonly HybridCache _cache;
    private readonly IProductRepository _repo;

    public ProductService(HybridCache cache, IProductRepository repo)
    {
        _cache = cache;
        _repo  = repo;
    }

    public async Task<Product?> GetByIdAsync(Guid id, CancellationToken ct = default)
    {
        // GetOrCreateAsync is stampede-safe: concurrent requests for the same key
        // collapse into a single factory invocation
        return await _cache.GetOrCreateAsync(
            key:     $"product:{id}",
            factory: async innerCt => await _repo.GetByIdAsync(id, innerCt),
            options: new HybridCacheEntryOptions
            {
                Expiration           = TimeSpan.FromMinutes(30),
                LocalCacheExpiration = TimeSpan.FromMinutes(5)
            },
            tags: [$"product:{id}", "products"],
            cancellationToken: ct);
    }

    public async Task<IReadOnlyList<Product>> GetByCategoryAsync(string category, CancellationToken ct = default)
    {
        return await _cache.GetOrCreateAsync(
            key:     $"products:category:{category}",
            factory: async innerCt => (IReadOnlyList<Product>)await _repo.GetByCategoryAsync(category, innerCt),
            options: new HybridCacheEntryOptions { Expiration = TimeSpan.FromMinutes(5) },
            tags:    [$"category:{category}", "products"],
            cancellationToken: ct) ?? [];
    }

    public async Task InvalidateProductAsync(Guid id, CancellationToken ct = default)
    {
        // Remove specific key
        await _cache.RemoveAsync($"product:{id}", ct);
    }

    public async Task InvalidateCategoryAsync(string category, CancellationToken ct = default)
    {
        // Remove all entries tagged with this category
        await _cache.RemoveByTagAsync($"category:{category}", ct);
    }

    public async Task InvalidateAllProductsAsync(CancellationToken ct = default)
    {
        // Remove everything tagged "products" — affects both L1 and L2
        await _cache.RemoveByTagAsync("products", ct);
    }

    public async Task UpdateAsync(Product product, CancellationToken ct = default)
    {
        await _repo.UpdateAsync(product, ct);

        // Invalidate by tag so all related cache entries are purged
        await _cache.RemoveByTagAsync($"product:{product.Id}", ct);
    }
}
```

---

## Step 2039: IMemoryCache — In-Process Caching with Size Limits

```csharp
// In-process caching with bounded size
builder.Services.AddMemoryCache(options =>
{
    options.SizeLimit             = 1000;       // Max 1000 "units" (you define unit size)
    options.ExpirationScanFrequency = TimeSpan.FromMinutes(5);
    options.CompactionPercentage   = 0.25;      // Remove 25% when at capacity
    options.TrackStatistics        = true;
});

public class CategoryCacheService
{
    private readonly IMemoryCache _cache;
    private readonly ICategoryRepository _repo;
    private readonly ILogger<CategoryCacheService> _logger;

    public CategoryCacheService(IMemoryCache cache, ICategoryRepository repo, ILogger<CategoryCacheService> logger)
    {
        _cache  = cache;
        _repo   = repo;
        _logger = logger;
    }

    public async Task<IReadOnlyList<Category>> GetAllAsync(CancellationToken ct = default)
    {
        const string cacheKey = "categories:all";

        if (_cache.TryGetValue(cacheKey, out IReadOnlyList<Category>? cached))
            return cached!;

        var categories = await _repo.GetAllActiveAsync(ct);

        var cacheOptions = new MemoryCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30),
            SlidingExpiration               = TimeSpan.FromMinutes(5),
            Size                            = 1,    // 1 unit
            Priority                        = CacheItemPriority.Normal,
        };

        // Eviction callback for observability
        cacheOptions.RegisterPostEvictionCallback((key, value, reason, state) =>
        {
            _logger.LogDebug("Cache entry {Key} evicted: {Reason}", key, reason);
        });

        _cache.Set(cacheKey, categories, cacheOptions);
        return categories;
    }
}

// IMemoryCache statistics
public class CacheMetricsEndpoint
{
    public static IResult GetStats(IMemoryCache cache)
    {
        if (cache is MemoryCache mc)
        {
            var stats = mc.GetCurrentStatistics();
            return Results.Ok(new
            {
                CurrentEntryCount  = stats?.CurrentEntryCount,
                CurrentEstimatedSize = stats?.CurrentEstimatedSize,
                TotalMisses        = stats?.TotalMisses,
                TotalHits          = stats?.TotalHits,
                HitRate            = stats is { TotalHits: > 0, TotalMisses: > 0 }
                    ? (double)stats.TotalHits / (stats.TotalHits + stats.TotalMisses)
                    : 0,
            });
        }

        return Results.Ok(new { message = "Stats unavailable" });
    }
}
```

---

## Step 2040: IDistributedCache & Redis Caching

```csharp
// IDistributedCache — works with any backend (Redis, SQL Server, etc.)
public class UserSessionCache
{
    private readonly IDistributedCache _cache;
    private readonly JsonSerializerOptions _jsonOpts;

    public UserSessionCache(IDistributedCache cache)
    {
        _cache    = cache;
        _jsonOpts = new JsonSerializerOptions { PropertyNamingPolicy = JsonNamingPolicy.CamelCase };
    }

    public async Task SetAsync(string sessionId, UserSession session, CancellationToken ct = default)
    {
        var json  = JsonSerializer.SerializeToUtf8Bytes(session, _jsonOpts);
        var opts  = new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(8),
            SlidingExpiration               = TimeSpan.FromMinutes(30)
        };

        await _cache.SetAsync($"session:{sessionId}", json, opts, ct);
    }

    public async Task<UserSession?> GetAsync(string sessionId, CancellationToken ct = default)
    {
        var json = await _cache.GetAsync($"session:{sessionId}", ct);
        if (json is null) return null;

        return JsonSerializer.Deserialize<UserSession>(json, _jsonOpts);
    }

    public async Task RemoveAsync(string sessionId, CancellationToken ct = default)
        => await _cache.RemoveAsync($"session:{sessionId}", ct);

    public async Task RefreshAsync(string sessionId, CancellationToken ct = default)
        => await _cache.RefreshAsync($"session:{sessionId}", ct); // resets sliding expiration
}

// Direct Redis operations for advanced scenarios
public class AdvancedRedisService
{
    private readonly IConnectionMultiplexer _redis;

    public AdvancedRedisService(IConnectionMultiplexer redis) => _redis = redis;

    // Atomic increment (distributed counter)
    public async Task<long> IncrementCounterAsync(string key, TimeSpan? expiry = null)
    {
        var db    = _redis.GetDatabase();
        var count = await db.StringIncrementAsync(key);

        if (expiry.HasValue && count == 1) // set expiry only on first increment
            await db.KeyExpireAsync(key, expiry.Value);

        return count;
    }

    // Distributed lock
    public async Task<bool> TryAcquireLockAsync(string resource, string token, TimeSpan expiry)
    {
        var db = _redis.GetDatabase();
        return await db.StringSetAsync($"lock:{resource}", token, expiry, When.NotExists);
    }

    public async Task<bool> ReleaseLockAsync(string resource, string token)
    {
        var db = _redis.GetDatabase();

        // Lua script for atomic check-and-delete
        const string script = @"
            if redis.call('get', KEYS[1]) == ARGV[1] then
                return redis.call('del', KEYS[1])
            else
                return 0
            end";

        var result = await db.ScriptEvaluateAsync(script,
            keys: [(RedisKey)$"lock:{resource}"],
            values: [(RedisValue)token]);

        return (long)result == 1;
    }

    // Sorted set — leaderboard
    public async Task UpdateScoreAsync(string leaderboard, string userId, double score)
    {
        var db = _redis.GetDatabase();
        await db.SortedSetAddAsync(leaderboard, userId, score, SortedSetWhen.GreaterThan);
    }

    public async Task<IReadOnlyList<LeaderboardEntry>> GetTopNAsync(string leaderboard, int n)
    {
        var db      = _redis.GetDatabase();
        var entries = await db.SortedSetRangeByRankWithScoresAsync(leaderboard, 0, n - 1, Order.Descending);

        return entries.Select((e, i) => new LeaderboardEntry(
            Rank:   i + 1,
            UserId: e.Element.ToString(),
            Score:  e.Score)).ToList();
    }

    // Hash — shopping cart
    public async Task SetCartItemAsync(string cartId, string productId, int quantity)
    {
        var db = _redis.GetDatabase();
        await db.HashSetAsync($"cart:{cartId}", productId, quantity);
        await db.KeyExpireAsync($"cart:{cartId}", TimeSpan.FromDays(7));
    }

    public async Task<Dictionary<string, int>> GetCartAsync(string cartId)
    {
        var db      = _redis.GetDatabase();
        var entries = await db.HashGetAllAsync($"cart:{cartId}");

        return entries.ToDictionary(
            e => e.Name.ToString(),
            e => (int)e.Value);
    }
}

public record LeaderboardEntry(int Rank, string UserId, double Score);
```

---

## Step 2041: Cache-Aside Pattern with Stampede Protection

The cache stampede (thundering herd) happens when many requests simultaneously miss the cache, all hitting the database at once.

```csharp
// Manual stampede protection using SemaphoreSlim
public class StampedeProofCache<TValue>
{
    private readonly IDistributedCache _cache;
    private readonly IMemoryCache _localCache;
    private readonly ConcurrentDictionary<string, SemaphoreSlim> _locks = new();
    private readonly JsonSerializerOptions _jsonOpts = new();

    public StampedeProofCache(IDistributedCache cache, IMemoryCache localCache)
    {
        _cache      = cache;
        _localCache = localCache;
    }

    public async Task<TValue?> GetOrCreateAsync(
        string key,
        Func<CancellationToken, Task<TValue?>> factory,
        TimeSpan distributed,
        TimeSpan? local = null,
        CancellationToken ct = default)
    {
        // L1: in-process
        if (_localCache.TryGetValue(key, out TValue? localValue))
            return localValue;

        // L2: distributed cache
        var bytes = await _cache.GetAsync(key, ct);
        if (bytes is not null)
        {
            var distValue = JsonSerializer.Deserialize<TValue>(bytes, _jsonOpts);
            _localCache.Set(key, distValue, local ?? TimeSpan.FromSeconds(30));
            return distValue;
        }

        // Stampede protection: only one request per key fetches from DB
        var semaphore = _locks.GetOrAdd(key, _ => new SemaphoreSlim(1, 1));
        await semaphore.WaitAsync(ct);

        try
        {
            // Double-check after acquiring lock
            bytes = await _cache.GetAsync(key, ct);
            if (bytes is not null)
                return JsonSerializer.Deserialize<TValue>(bytes, _jsonOpts);

            // Cache miss — call factory
            var value = await factory(ct);
            if (value is not null)
            {
                var serialized = JsonSerializer.SerializeToUtf8Bytes(value, _jsonOpts);
                await _cache.SetAsync(key, serialized,
                    new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = distributed }, ct);

                _localCache.Set(key, value, local ?? TimeSpan.FromSeconds(30));
            }

            return value;
        }
        finally
        {
            semaphore.Release();
            _locks.TryRemove(key, out _);
        }
    }
}

// HybridCache handles this automatically — use it instead when on .NET 9+
```

---

## Step 2042: Cache Invalidation Strategies

```csharp
// Tag-based invalidation (works natively with HybridCache .NET 9)
// For .NET 8, implement manually:
public class TaggedCache : ITaggedCache
{
    private readonly IDistributedCache _cache;
    private readonly IConnectionMultiplexer _redis;

    public TaggedCache(IDistributedCache cache, IConnectionMultiplexer redis)
    {
        _cache = cache;
        _redis = redis;
    }

    public async Task SetAsync<T>(
        string key, T value,
        DistributedCacheEntryOptions options,
        IEnumerable<string> tags,
        CancellationToken ct = default)
    {
        var db   = _redis.GetDatabase();
        var json = JsonSerializer.SerializeToUtf8Bytes(value);

        // Store value in distributed cache
        await _cache.SetAsync(key, json, options, ct);

        // Add key to each tag's set in Redis
        foreach (var tag in tags)
        {
            await db.SetAddAsync($"tag:{tag}", key);

            // Set tag expiry (slightly longer than value expiry)
            var tagExpiry = options.AbsoluteExpirationRelativeToNow?.Add(TimeSpan.FromMinutes(1))
                            ?? TimeSpan.FromHours(1);
            await db.KeyExpireAsync($"tag:{tag}", tagExpiry);
        }
    }

    public async Task InvalidateByTagAsync(string tag, CancellationToken ct = default)
    {
        var db      = _redis.GetDatabase();
        var members = await db.SetMembersAsync($"tag:{tag}");

        // Delete all cached keys with this tag
        var tasks = members.Select(m => _cache.RemoveAsync(m.ToString(), ct));
        await Task.WhenAll(tasks);

        // Delete the tag set itself
        await db.KeyDeleteAsync($"tag:{tag}");
    }
}

// Event-driven cache invalidation via MassTransit
public class ProductUpdatedCacheInvalidator : IConsumer<ProductUpdated>
{
    private readonly HybridCache _cache;
    private readonly ILogger<ProductUpdatedCacheInvalidator> _logger;

    public ProductUpdatedCacheInvalidator(HybridCache cache, ILogger<ProductUpdatedCacheInvalidator> logger)
    {
        _cache  = cache;
        _logger = logger;
    }

    public async Task Consume(ConsumeContext<ProductUpdated> ctx)
    {
        var msg = ctx.Message;
        _logger.LogInformation("Invalidating cache for product {ProductId}", msg.ProductId);

        await Task.WhenAll(
            _cache.RemoveAsync($"product:{msg.ProductId}", ctx.CancellationToken),
            _cache.RemoveByTagAsync($"category:{msg.CategoryId}", ctx.CancellationToken),
            _cache.RemoveByTagAsync("product-list", ctx.CancellationToken));
    }
}

// Register consumer
builder.Services.AddMassTransit(cfg =>
{
    cfg.AddConsumer<ProductUpdatedCacheInvalidator>();
    // ... rest of config
});
```

---

## Step 2043: Output Caching (HTTP Response Caching)

```csharp
// Output caching caches entire HTTP responses
builder.Services.AddOutputCache(options =>
{
    // Default policy
    options.AddBasePolicy(policy =>
        policy.Expire(TimeSpan.FromMinutes(5)));

    // Named policies
    options.AddPolicy("ProductDetail", policy =>
        policy
            .Expire(TimeSpan.FromMinutes(30))
            .SetVaryByHeader("Accept-Language")
            .Tag("products")
            .Cache());

    options.AddPolicy("ShortLived", policy =>
        policy.Expire(TimeSpan.FromSeconds(30)));

    options.AddPolicy("UserSpecific", policy =>
        policy
            .Expire(TimeSpan.FromMinutes(1))
            .SetVaryByRouteValue("userId")
            .SetVaryByHeader("Authorization"));

    options.AddPolicy("NoCache", policy =>
        policy.NoCache());
});

// Wire up Redis store for distributed output caching
builder.Services.AddStackExchangeRedisOutputCache(options =>
    options.Configuration = builder.Configuration.GetConnectionString("Redis"));

var app = builder.Build();
app.UseOutputCache();

// Apply to endpoints
app.MapGet("/api/products/{id}", async (Guid id, IProductService service) =>
{
    var product = await service.GetByIdAsync(id);
    return product is null ? Results.NotFound() : Results.Ok(product);
})
.CacheOutput("ProductDetail")
.WithName("GetProductById");

app.MapGet("/api/categories", async (ICategoryService service) =>
    Results.Ok(await service.GetAllAsync()))
.CacheOutput(policy => policy
    .Expire(TimeSpan.FromHours(1))
    .Tag("categories"));

// Invalidate output cache by tag from a command handler
app.MapPost("/api/products/{id}", async (
    Guid id, [FromBody] UpdateProductRequest req,
    IProductService service,
    IOutputCacheStore outputCacheStore,
    CancellationToken ct) =>
{
    await service.UpdateAsync(id, req, ct);

    // Invalidate cached responses for affected endpoints
    await outputCacheStore.EvictByTagAsync("products", ct);

    return Results.NoContent();
}).RequireAuthorization("products:write");
```

---

## Step 2044: Response Caching Middleware (HTTP/CDN Friendly)

```csharp
// HTTP-level response caching (leverages ETags, Cache-Control headers)
builder.Services.AddResponseCaching(options =>
{
    options.MaximumBodySize          = 64 * 1024; // 64KB max cacheable body
    options.UseCaseSensitivePaths    = false;
    options.SizeLimit                = 100 * 1024 * 1024; // 100MB store
});

var app = builder.Build();
app.UseResponseCaching();

// Endpoint with cache headers
app.MapGet("/api/public/catalog", async (ICatalogService service) =>
{
    var catalog = await service.GetPublicCatalogAsync();
    return Results.Ok(catalog);
})
.WithMetadata(new ResponseCacheAttribute
{
    Duration             = 300,                       // Cache-Control: max-age=300
    Location             = ResponseCacheLocation.Any, // public
    VaryByHeader         = "Accept-Language",
    VaryByQueryKeys      = ["page", "pageSize", "sort"]
});

// Custom cache control via middleware
app.Use(async (ctx, next) =>
{
    await next();

    // Prevent caching of auth endpoints
    if (ctx.Request.Path.StartsWithSegments("/api/auth"))
    {
        ctx.Response.Headers.CacheControl = "no-store, no-cache";
        ctx.Response.Headers.Pragma       = "no-cache";
    }
});
```

---

## Step 2045: Cache-Aside with Write-Through Pattern

```csharp
// Write-through: update cache and DB simultaneously
public class WriteThroughOrderRepository : IOrderRepository
{
    private readonly IOrderRepository _inner;
    private readonly HybridCache _cache;
    private readonly ILogger<WriteThroughOrderRepository> _logger;

    public WriteThroughOrderRepository(
        IOrderRepository inner,
        HybridCache cache,
        ILogger<WriteThroughOrderRepository> logger)
    {
        _inner  = inner;
        _cache  = cache;
        _logger = logger;
    }

    public async Task<Order?> GetByIdAsync(Guid id, CancellationToken ct = default)
    {
        return await _cache.GetOrCreateAsync(
            $"order:{id}",
            async innerCt => await _inner.GetByIdAsync(id, innerCt),
            cancellationToken: ct);
    }

    public async Task AddAsync(Order order, CancellationToken ct = default)
    {
        await _inner.AddAsync(order, ct);

        // Populate cache immediately after write
        await _cache.SetAsync(
            $"order:{order.Id}",
            order,
            new HybridCacheEntryOptions
            {
                Expiration           = TimeSpan.FromMinutes(30),
                LocalCacheExpiration = TimeSpan.FromMinutes(5)
            },
            tags: [$"order:{order.Id}", $"customer:{order.CustomerId}"],
            cancellationToken: ct);

        _logger.LogDebug("Order {OrderId} written to cache after insert", order.Id);
    }

    public async Task UpdateAsync(Order order, CancellationToken ct = default)
    {
        await _inner.UpdateAsync(order, ct);

        // Update cache with new version
        await _cache.SetAsync(
            $"order:{order.Id}",
            order,
            new HybridCacheEntryOptions { Expiration = TimeSpan.FromMinutes(30) },
            tags: [$"order:{order.Id}", $"customer:{order.CustomerId}"],
            cancellationToken: ct);
    }

    public async Task DeleteAsync(Guid id, CancellationToken ct = default)
    {
        await _inner.DeleteAsync(id, ct);
        await _cache.RemoveAsync($"order:{id}", ct);
    }
}

// Register with decorator pattern
builder.Services.AddScoped<IOrderRepository, EfCoreOrderRepository>();
builder.Services.Decorate<IOrderRepository, WriteThroughOrderRepository>();

// Scrutor for decorator registration
// dotnet add package Scrutor
builder.Services.AddScoped<EfCoreOrderRepository>();
builder.Services.AddScoped<IOrderRepository>(sp =>
    new WriteThroughOrderRepository(
        sp.GetRequiredService<EfCoreOrderRepository>(),
        sp.GetRequiredService<HybridCache>(),
        sp.GetRequiredService<ILogger<WriteThroughOrderRepository>>()));
```

---

## Step 2046: Refresh-Ahead Caching

```csharp
// Refresh-ahead: proactively refresh cache before expiry
public class RefreshAheadCache<TValue>
{
    private readonly IDistributedCache _cache;
    private readonly IMemoryCache _localCache;
    private readonly JsonSerializerOptions _json = new();

    public RefreshAheadCache(IDistributedCache cache, IMemoryCache localCache)
    {
        _cache      = cache;
        _localCache = localCache;
    }

    public async Task<TValue?> GetOrCreateAsync(
        string key,
        Func<CancellationToken, Task<TValue?>> factory,
        TimeSpan ttl,
        double refreshThreshold = 0.75, // refresh when 75% of TTL elapsed
        CancellationToken ct = default)
    {
        var metaKey = $"{key}:meta";

        // Check local cache first
        if (_localCache.TryGetValue(key, out TValue? local))
            return local;

        var bytes = await _cache.GetAsync(key, ct);
        if (bytes is not null)
        {
            var metaBytes = await _cache.GetAsync(metaKey, ct);
            if (metaBytes is not null)
            {
                var meta = JsonSerializer.Deserialize<CacheMeta>(metaBytes, _json);
                var elapsed = DateTimeOffset.UtcNow - meta!.CreatedAt;

                // Trigger background refresh if approaching expiry
                if (elapsed > ttl * refreshThreshold)
                {
                    _ = Task.Run(async () =>
                    {
                        try { await RefreshAsync(key, factory, ttl, ct); }
                        catch { /* swallow — serve stale */ }
                    }, CancellationToken.None);
                }
            }

            var value = JsonSerializer.Deserialize<TValue>(bytes, _json);
            _localCache.Set(key, value, TimeSpan.FromSeconds(30));
            return value;
        }

        // Cold miss — fetch and store
        return await RefreshAsync(key, factory, ttl, ct);
    }

    private async Task<TValue?> RefreshAsync(
        string key,
        Func<CancellationToken, Task<TValue?>> factory,
        TimeSpan ttl,
        CancellationToken ct)
    {
        var value = await factory(ct);
        if (value is null) return default;

        var bytes    = JsonSerializer.SerializeToUtf8Bytes(value, _json);
        var metaBytes = JsonSerializer.SerializeToUtf8Bytes(new CacheMeta(DateTimeOffset.UtcNow), _json);
        var options  = new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = ttl };

        await _cache.SetAsync(key, bytes, options, ct);
        await _cache.SetAsync($"{key}:meta", metaBytes, options, ct);
        _localCache.Set(key, value, TimeSpan.FromSeconds(30));

        return value;
    }

    private record CacheMeta(DateTimeOffset CreatedAt);
}
```

---

## Step 2047: Conditional Requests & ETags

```csharp
// ETags enable conditional requests — client sends If-None-Match header
// to check if resource changed; server returns 304 Not Modified if not

public class ETagMiddleware
{
    private readonly RequestDelegate _next;

    public ETagMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(HttpContext ctx)
    {
        if (!HttpMethods.IsGet(ctx.Request.Method) && !HttpMethods.IsHead(ctx.Request.Method))
        {
            await _next(ctx);
            return;
        }

        var originalBody = ctx.Response.Body;
        using var buffer = new MemoryStream();
        ctx.Response.Body = buffer;

        await _next(ctx);

        if (ctx.Response.StatusCode != 200)
        {
            buffer.Seek(0, SeekOrigin.Begin);
            await buffer.CopyToAsync(originalBody);
            ctx.Response.Body = originalBody;
            return;
        }

        buffer.Seek(0, SeekOrigin.Begin);
        var body  = buffer.ToArray();
        var hash  = SHA256.HashData(body);
        var etag  = $"\"{Convert.ToHexString(hash)[..16]}\"";

        ctx.Response.Headers.ETag = etag;

        // Check If-None-Match
        var inm = ctx.Request.Headers.IfNoneMatch.ToString();
        if (!string.IsNullOrEmpty(inm) && EntityTagHeaderValue.TryParse(etag, out var serverEtag))
        {
            var clientEtags = inm.Split(',').Select(s => s.Trim());
            if (clientEtags.Contains(etag) || inm == "*")
            {
                ctx.Response.StatusCode    = 304;
                ctx.Response.ContentLength = 0;
                ctx.Response.Body          = originalBody;
                return;
            }
        }

        ctx.Response.Body = originalBody;
        await originalBody.WriteAsync(body, ctx.RequestAborted);
    }
}

// ETag from entity version (more efficient — no buffering)
public class OrderController : ControllerBase
{
    [HttpGet("{id}")]
    public async Task<IActionResult> GetOrder(Guid id, IOrderRepository repo)
    {
        var order = await repo.GetByIdAsync(id);
        if (order is null) return NotFound();

        var etag = $"\"{order.Version}-{order.UpdatedAt.Ticks}\"";

        // Check conditional request
        if (Request.Headers.TryGetValue("If-None-Match", out var ifNoneMatch) &&
            ifNoneMatch.ToString() == etag)
        {
            return StatusCode(304);
        }

        Response.Headers.ETag         = etag;
        Response.Headers.LastModified = order.UpdatedAt.ToString("R");

        return Ok(order);
    }
}
```

---

## Step 2048: Caching with Dependency Injection & Named Caches

```csharp
// Multiple named Redis connections (different databases for different cache types)
builder.Services.AddSingleton<IConnectionMultiplexer>(sp =>
    ConnectionMultiplexer.Connect(builder.Configuration.GetConnectionString("Redis")!));

// Named distributed caches
builder.Services.AddKeyedSingleton<IDistributedCache>("sessions", (sp, _) =>
{
    var redis = sp.GetRequiredService<IConnectionMultiplexer>();
    return new RedisCache(new RedisCacheOptions
    {
        ConnectionMultiplexerFactory = () => Task.FromResult(redis),
        InstanceName = "session:",
    });
});

builder.Services.AddKeyedSingleton<IDistributedCache>("products", (sp, _) =>
{
    var redis = sp.GetRequiredService<IConnectionMultiplexer>();
    return new RedisCache(new RedisCacheOptions
    {
        ConnectionMultiplexerFactory = () => Task.FromResult(redis),
        InstanceName = "product:",
    });
});

// Usage with keyed injection
public class ProductCache
{
    private readonly IDistributedCache _cache;

    public ProductCache([FromKeyedServices("products")] IDistributedCache cache)
        => _cache = cache;
}

// Cache abstraction with metrics
public class MeteredCache : IDistributedCache
{
    private readonly IDistributedCache _inner;
    private readonly Counter<long> _hits;
    private readonly Counter<long> _misses;

    public MeteredCache(IDistributedCache inner, IMeterFactory meterFactory)
    {
        _inner  = inner;
        var meter = meterFactory.Create("MyApp.Cache");
        _hits   = meter.CreateCounter<long>("cache.hits");
        _misses = meter.CreateCounter<long>("cache.misses");
    }

    public async Task<byte[]?> GetAsync(string key, CancellationToken token = default)
    {
        var result = await _inner.GetAsync(key, token);
        if (result is not null)
            _hits.Add(1, new TagList { { "key_prefix", ExtractPrefix(key) } });
        else
            _misses.Add(1, new TagList { { "key_prefix", ExtractPrefix(key) } });

        return result;
    }

    public Task SetAsync(string key, byte[] value, DistributedCacheEntryOptions options, CancellationToken token = default)
        => _inner.SetAsync(key, value, options, token);

    public Task RemoveAsync(string key, CancellationToken token = default)
        => _inner.RemoveAsync(key, token);

    public Task RefreshAsync(string key, CancellationToken token = default)
        => _inner.RefreshAsync(key, token);

    public byte[]? Get(string key) => _inner.Get(key);
    public void Set(string key, byte[] value, DistributedCacheEntryOptions options) => _inner.Set(key, value, options);
    public void Remove(string key) => _inner.Remove(key);
    public void Refresh(string key) => _inner.Refresh(key);

    private static string ExtractPrefix(string key)
    {
        var idx = key.IndexOf(':');
        return idx >= 0 ? key[..idx] : "unknown";
    }
}
```

---

## Step 2049: Redis Pub/Sub for Cache Invalidation Across Instances

```csharp
// When running multiple app instances, use Redis pub/sub to invalidate
// in-process caches across all instances simultaneously

public class RedisCacheInvalidationService : IHostedService, ICacheInvalidator
{
    private readonly IConnectionMultiplexer _redis;
    private readonly IMemoryCache _localCache;
    private readonly ILogger<RedisCacheInvalidationService> _logger;
    private ISubscriber? _subscriber;

    private const string Channel = "cache:invalidate";

    public RedisCacheInvalidationService(
        IConnectionMultiplexer redis,
        IMemoryCache localCache,
        ILogger<RedisCacheInvalidationService> logger)
    {
        _redis      = redis;
        _localCache = localCache;
        _logger     = logger;
    }

    public async Task StartAsync(CancellationToken ct)
    {
        _subscriber = _redis.GetSubscriber();

        await _subscriber.SubscribeAsync(
            RedisChannel.Literal(Channel),
            (_, message) =>
            {
                var key = message.ToString();
                _localCache.Remove(key);
                _logger.LogDebug("Invalidated local cache entry: {Key}", key);
            });

        _logger.LogInformation("Cache invalidation subscriber started");
    }

    public async Task StopAsync(CancellationToken ct)
    {
        if (_subscriber is not null)
            await _subscriber.UnsubscribeAllAsync();
    }

    public async Task InvalidateAsync(string key, CancellationToken ct = default)
    {
        // Remove locally
        _localCache.Remove(key);

        // Broadcast to all instances
        var sub = _redis.GetSubscriber();
        await sub.PublishAsync(RedisChannel.Literal(Channel), key);
    }

    public async Task InvalidateByPatternAsync(string pattern, CancellationToken ct = default)
    {
        var db      = _redis.GetDatabase();
        var server  = _redis.GetServer(_redis.GetEndPoints().First());
        var keys    = server.KeysAsync(pattern: pattern);

        await foreach (var key in keys)
        {
            _localCache.Remove(key.ToString());
            await db.KeyDeleteAsync(key);
        }
    }
}

// Register
builder.Services.AddSingleton<RedisCacheInvalidationService>();
builder.Services.AddSingleton<IHostedService>(sp => sp.GetRequiredService<RedisCacheInvalidationService>());
builder.Services.AddSingleton<ICacheInvalidator>(sp => sp.GetRequiredService<RedisCacheInvalidationService>());
```

---

## Step 2050: Caching with Polly — Resilient Cache Access

```csharp
// Polly v8 + Redis: circuit breaker + fallback for resilient caching
// dotnet add package Polly
// dotnet add package Microsoft.Extensions.Http.Polly

builder.Services.AddResiliencePipeline("redis-cache", builder =>
{
    builder
        // Retry transient Redis failures
        .AddRetry(new RetryStrategyOptions
        {
            MaxRetryAttempts = 3,
            Delay            = TimeSpan.FromMilliseconds(100),
            BackoffType      = DelayBackoffType.Exponential,
            ShouldHandle     = args => args.Outcome.Exception is RedisException
                                       ? PredicateResult.True()
                                       : PredicateResult.False()
        })
        // Open circuit if Redis is consistently failing
        .AddCircuitBreaker(new CircuitBreakerStrategyOptions
        {
            SamplingDuration     = TimeSpan.FromSeconds(30),
            MinimumThroughput    = 5,
            FailureRatio         = 0.5,
            BreakDuration        = TimeSpan.FromSeconds(15),
            OnOpened             = args =>
            {
                // Log circuit breaker opened
                return ValueTask.CompletedTask;
            }
        })
        // Timeout per Redis call
        .AddTimeout(TimeSpan.FromMilliseconds(500));
});

public class ResilientCacheService : ICacheService
{
    private readonly IDistributedCache _cache;
    private readonly ResiliencePipeline _pipeline;
    private readonly ILogger<ResilientCacheService> _logger;

    public ResilientCacheService(
        IDistributedCache cache,
        ResiliencePipelineProvider<string> pipelineProvider,
        ILogger<ResilientCacheService> logger)
    {
        _cache    = cache;
        _pipeline = pipelineProvider.GetPipeline("redis-cache");
        _logger   = logger;
    }

    public async Task<T?> GetOrCreateAsync<T>(
        string key,
        Func<CancellationToken, Task<T?>> factory,
        TimeSpan ttl,
        CancellationToken ct = default)
    {
        byte[]? bytes = null;

        try
        {
            bytes = await _pipeline.ExecuteAsync(
                async innerCt => await _cache.GetAsync(key, innerCt),
                ct);
        }
        catch (BrokenCircuitException)
        {
            _logger.LogWarning("Cache circuit open — bypassing for key: {Key}", key);
            return await factory(ct); // fall through to DB
        }

        if (bytes is not null)
            return JsonSerializer.Deserialize<T>(bytes);

        var value = await factory(ct);
        if (value is not null)
        {
            try
            {
                var serialized = JsonSerializer.SerializeToUtf8Bytes(value);
                await _pipeline.ExecuteAsync(
                    async innerCt => await _cache.SetAsync(
                        key, serialized,
                        new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = ttl },
                        innerCt),
                    ct);
            }
            catch (BrokenCircuitException)
            {
                _logger.LogWarning("Cache circuit open — skipping write for key: {Key}", key);
            }
        }

        return value;
    }
}
```

---

## Step 2051: Cache Warming & Preloading

```csharp
// Background service to warm critical caches on startup
public class CacheWarmingService : BackgroundService
{
    private readonly IServiceProvider _sp;
    private readonly ILogger<CacheWarmingService> _logger;

    public CacheWarmingService(IServiceProvider sp, ILogger<CacheWarmingService> logger)
    {
        _sp     = sp;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        _logger.LogInformation("Starting cache warm-up...");
        var sw = Stopwatch.StartNew();

        await using var scope = _sp.CreateAsyncScope();
        var productService  = scope.ServiceProvider.GetRequiredService<IProductService>();
        var categoryService = scope.ServiceProvider.GetRequiredService<ICategoryService>();

        await Task.WhenAll(
            WarmCategoriesAsync(categoryService, ct),
            WarmFeaturedProductsAsync(productService, ct));

        sw.Stop();
        _logger.LogInformation("Cache warm-up completed in {ElapsedMs}ms", sw.ElapsedMilliseconds);
    }

    private async Task WarmCategoriesAsync(ICategoryService svc, CancellationToken ct)
    {
        try
        {
            var categories = await svc.GetAllAsync(ct);
            _logger.LogDebug("Warmed {Count} categories", categories.Count);
        }
        catch (Exception ex)
        {
            _logger.LogWarning(ex, "Failed to warm categories cache");
        }
    }

    private async Task WarmFeaturedProductsAsync(IProductService svc, CancellationToken ct)
    {
        try
        {
            var featured = await svc.GetFeaturedAsync(ct);
            _logger.LogDebug("Warmed {Count} featured products", featured.Count);
        }
        catch (Exception ex)
        {
            _logger.LogWarning(ex, "Failed to warm featured products cache");
        }
    }
}

// Register
builder.Services.AddHostedService<CacheWarmingService>();
```

---

## Step 2052: Complete Caching Configuration & Monitoring

```csharp
// appsettings.json
{
  "ConnectionStrings": {
    "Redis": "localhost:6379,abortConnect=false,connectTimeout=5000,syncTimeout=5000,asyncTimeout=5000"
  },
  "Cache": {
    "DefaultExpiryMinutes": 10,
    "LocalExpirySeconds": 30,
    "ProductExpiryMinutes": 30,
    "CategoryExpiryHours": 1,
    "SessionExpiryHours": 8
  }
}

// Complete cache setup in Program.cs
var builder = WebApplication.CreateBuilder(args);

// In-process cache
builder.Services.AddMemoryCache(opts =>
{
    opts.SizeLimit            = 10000;
    opts.CompactionPercentage = 0.25;
    opts.TrackStatistics      = true;
});

// Redis distributed cache
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration   = builder.Configuration.GetConnectionString("Redis");
    options.InstanceName    = $"{builder.Environment.EnvironmentName}:";
});

// HybridCache (.NET 9)
builder.Services.AddHybridCache(opts =>
{
    opts.DefaultEntryOptions = new HybridCacheEntryOptions
    {
        Expiration           = TimeSpan.FromMinutes(10),
        LocalCacheExpiration = TimeSpan.FromSeconds(30)
    };
});

// Output cache (Redis-backed)
builder.Services.AddOutputCache(opts =>
{
    opts.AddBasePolicy(p => p.Expire(TimeSpan.FromMinutes(5)));
    opts.AddPolicy("LongLived", p => p.Expire(TimeSpan.FromHours(1)).Tag("static"));
});
builder.Services.AddStackExchangeRedisOutputCache(opts =>
    opts.Configuration = builder.Configuration.GetConnectionString("Redis"));

// Cache invalidation
builder.Services.AddSingleton<RedisCacheInvalidationService>();
builder.Services.AddHostedService(sp => sp.GetRequiredService<RedisCacheInvalidationService>());
builder.Services.AddSingleton<ICacheInvalidator>(sp => sp.GetRequiredService<RedisCacheInvalidationService>());

// Cache warming
builder.Services.AddHostedService<CacheWarmingService>();

var app = builder.Build();
app.UseOutputCache();

// Cache health check
builder.Services.AddHealthChecks()
    .AddRedis(
        builder.Configuration.GetConnectionString("Redis")!,
        name:        "redis",
        tags:        ["cache", "infrastructure"],
        failureStatus: HealthStatus.Degraded);

// Cache metrics endpoint
app.MapGet("/api/diagnostics/cache", (IMemoryCache cache) =>
{
    var stats = (cache as MemoryCache)?.GetCurrentStatistics();
    return Results.Ok(new
    {
        InProcess = new
        {
            stats?.CurrentEntryCount,
            stats?.CurrentEstimatedSize,
            HitRate = stats is { TotalHits: > 0 }
                ? (double)stats.TotalHits / (stats.TotalHits + stats.TotalMisses)
                : 0d
        }
    });
}).RequireAuthorization("admin");
```

**Summary**: Part 88 covers all major caching patterns — cache-aside, write-through, refresh-ahead — using HybridCache (.NET 9), IMemoryCache, IDistributedCache, Redis, Output Caching, conditional requests/ETags, Redis pub/sub for cross-instance invalidation, Polly-resilient cache access, and cache warming on startup. Steps 2037–2052 complete.
