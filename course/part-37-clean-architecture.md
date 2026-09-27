# Part 37: Clean Architecture

## Steps 1051-1080 | ระดับโลก (World-Class)

---

## Step 1051: Clean Architecture คืออะไร

Clean Architecture ของ Robert C. Martin ("Uncle Bob") เป็นแนวทางออกแบบระบบที่แยก concerns ออกเป็นชั้น และให้ dependency ไหลเข้าด้านใน (Dependency Rule)

```
          ┌─────────────────────────────────┐
          │         Frameworks &            │
          │         Drivers (Web, DB)       │
          │   ┌─────────────────────────┐   │
          │   │    Interface Adapters   │   │
          │   │  (Controllers, Gateways)│   │
          │   │  ┌───────────────────┐  │   │
          │   │  │ Application Rules │  │   │
          │   │  │  (Use Cases)      │  │   │
          │   │  │ ┌─────────────┐   │  │   │
          │   │  │ │  Enterprise  │  │  │   │
          │   │  │ │   Business  │  │  │   │
          │   │  │ │   Rules     │  │  │   │
          │   │  │ │  (Domain)   │  │  │   │
          │   │  │ └─────────────┘  │  │   │
          │   │  └───────────────────┘  │   │
          │   └─────────────────────────┘   │
          └─────────────────────────────────┘

Dependency Rule: ลูกศรชี้เข้าด้านใน
→ outer layers depend on inner, NEVER the reverse
```

### Dependency Rule

- **Domain**: ไม่ขึ้นกับใครเลย — pure C# classes
- **Application**: ขึ้นกับ Domain เท่านั้น
- **Infrastructure**: ขึ้นกับ Application + Domain
- **API/Presentation**: ขึ้นกับ Application

---

## Step 1052: โครงสร้าง Solution

```
CleanApp/
├── CleanApp.Domain/          # Enterprise Business Rules
├── CleanApp.Application/     # Application Business Rules
├── CleanApp.Infrastructure/  # Interface Adapters + Frameworks
├── CleanApp.API/             # Frameworks & Drivers
└── CleanApp.Tests/
    ├── Unit/
    ├── Integration/
    └── Architecture/         # ArchUnit tests
```

```bash
# สร้าง solution
dotnet new sln -n CleanApp

dotnet new classlib -n CleanApp.Domain -f net9.0
dotnet new classlib -n CleanApp.Application -f net9.0
dotnet new classlib -n CleanApp.Infrastructure -f net9.0
dotnet new webapi -n CleanApp.API -f net9.0
dotnet new xunit -n CleanApp.Tests.Unit -f net9.0
dotnet new xunit -n CleanApp.Tests.Architecture -f net9.0

dotnet sln add CleanApp.Domain CleanApp.Application CleanApp.Infrastructure CleanApp.API
dotnet sln add CleanApp.Tests.Unit CleanApp.Tests.Architecture

# Reference rules (Dependency Rule)
# Application -> Domain
dotnet add CleanApp.Application reference CleanApp.Domain
# Infrastructure -> Application, Domain
dotnet add CleanApp.Infrastructure reference CleanApp.Application CleanApp.Domain
# API -> Application, Infrastructure
dotnet add CleanApp.API reference CleanApp.Application CleanApp.Infrastructure
```

---

## Step 1053: Domain Layer

Domain layer ต้องไม่มี NuGet dependency ใด ๆ ยกเว้น base libraries

```csharp
// Domain/Entities/BlogPost.cs
public class BlogPost : AggregateRoot<BlogPostId>
{
    private readonly List<Tag> _tags = [];
    private readonly List<Comment> _comments = [];

    public Title Title { get; private set; }
    public Content Body { get; private set; }
    public AuthorId AuthorId { get; private set; }
    public PublishStatus Status { get; private set; }
    public Slug Slug { get; private set; }
    public DateTime CreatedAt { get; private init; }
    public DateTime? PublishedAt { get; private set; }
    public IReadOnlyList<Tag> Tags => _tags.AsReadOnly();
    public IReadOnlyList<Comment> Comments => _comments.AsReadOnly();

    private BlogPost() : base(default!) { }

    private BlogPost(BlogPostId id, Title title, Content body, AuthorId authorId) : base(id)
    {
        Title = title;
        Body = body;
        AuthorId = authorId;
        Slug = Slug.FromTitle(title.Value);
        Status = PublishStatus.Draft;
        CreatedAt = DateTime.UtcNow;
    }

    public static BlogPost Create(string title, string body, Guid authorId)
        => new(BlogPostId.New(), Title.Create(title), Content.Create(body), new AuthorId(authorId));

    public void Update(string title, string body)
    {
        if (Status == PublishStatus.Published)
            throw new DomainException("Cannot edit a published post without creating revision");

        Title = Title.Create(title);
        Body = Content.Create(body);
        Slug = Slug.FromTitle(title);
        RaiseDomainEvent(new BlogPostUpdatedEvent(Id));
    }

    public void Publish()
    {
        if (Status == PublishStatus.Published)
            throw new DomainException("Post is already published");

        if (string.IsNullOrWhiteSpace(Body.Value))
            throw new DomainException("Cannot publish post with empty body");

        Status = PublishStatus.Published;
        PublishedAt = DateTime.UtcNow;
        RaiseDomainEvent(new BlogPostPublishedEvent(Id, Slug));
    }

    public void Unpublish()
    {
        if (Status != PublishStatus.Published)
            throw new DomainException("Post is not published");

        Status = PublishStatus.Draft;
        PublishedAt = null;
    }

    public void AddTag(Tag tag)
    {
        if (_tags.Count >= 10) throw new DomainException("Maximum 10 tags allowed");
        if (!_tags.Contains(tag)) _tags.Add(tag);
    }

    public Comment AddComment(Guid authorId, string text)
    {
        if (Status != PublishStatus.Published)
            throw new DomainException("Cannot comment on unpublished post");

        var comment = Comment.Create(CommentId.New(), new AuthorId(authorId), CommentText.Create(text));
        _comments.Add(comment);
        RaiseDomainEvent(new CommentAddedEvent(Id, comment.Id));
        return comment;
    }
}
```

```csharp
// Domain/ValueObjects/Title.cs
public sealed class Title : ValueObject
{
    public string Value { get; }
    private Title(string value) => Value = value;

    public static Title Create(string value)
    {
        if (string.IsNullOrWhiteSpace(value)) throw new DomainException("Title required");
        if (value.Length > 200) throw new DomainException("Title too long");
        return new Title(value.Trim());
    }

    protected override IEnumerable<object?> GetEqualityComponents() { yield return Value; }
}

// Domain/ValueObjects/Slug.cs
public sealed class Slug : ValueObject
{
    public string Value { get; }
    private Slug(string value) => Value = value;

    public static Slug FromTitle(string title)
    {
        var slug = title.ToLowerInvariant()
            .Replace(" ", "-")
            .Replace(".", "")
            .Replace(",", "")
            .Replace("'", "");
        // Remove non-alphanumeric except hyphens
        slug = Regex.Replace(slug, @"[^a-z0-9\-]", "");
        slug = Regex.Replace(slug, @"-{2,}", "-").Trim('-');
        return new Slug(slug);
    }

    protected override IEnumerable<object?> GetEqualityComponents() { yield return Value; }
}

// Domain/ValueObjects/Tag.cs
public sealed class Tag : ValueObject
{
    public string Value { get; }
    private Tag(string value) => Value = value;

    public static Tag Create(string value)
    {
        if (string.IsNullOrWhiteSpace(value)) throw new DomainException("Tag required");
        return new Tag(value.Trim().ToLowerInvariant());
    }

    protected override IEnumerable<object?> GetEqualityComponents() { yield return Value; }
    public static implicit operator string(Tag tag) => tag.Value;
}
```

---

## Step 1054: Domain Interfaces (Ports)

Domain layer กำหนด interfaces ที่ outer layers ต้อง implement

```csharp
// Domain/Repositories/IBlogPostRepository.cs
public interface IBlogPostRepository
{
    Task<BlogPost?> GetByIdAsync(BlogPostId id, CancellationToken ct = default);
    Task<BlogPost?> GetBySlugAsync(Slug slug, CancellationToken ct = default);
    Task<IReadOnlyList<BlogPost>> GetPublishedAsync(int skip, int take, CancellationToken ct = default);
    Task<int> CountPublishedAsync(CancellationToken ct = default);
    Task AddAsync(BlogPost post, CancellationToken ct = default);
    Task UpdateAsync(BlogPost post, CancellationToken ct = default);
    Task DeleteAsync(BlogPostId id, CancellationToken ct = default);
}

// Domain/Services/ISlugUniquenessChecker.cs
public interface ISlugUniquenessChecker
{
    Task<bool> IsUniqueAsync(Slug slug, BlogPostId? excludeId = null, CancellationToken ct = default);
}

// Domain/Services/IFileStorageService.cs
public interface IFileStorageService
{
    Task<Uri> UploadAsync(Stream content, string fileName, string contentType, CancellationToken ct = default);
    Task DeleteAsync(string fileKey, CancellationToken ct = default);
}
```

---

## Step 1055: Application Layer - Use Cases

```csharp
// Application/BlogPosts/Commands/CreatePost/CreateBlogPostCommand.cs
public record CreateBlogPostCommand(
    string Title,
    string Body,
    Guid AuthorId,
    IReadOnlyList<string> Tags) : IRequest<BlogPostId>;

// Application/BlogPosts/Commands/CreatePost/CreateBlogPostHandler.cs
public class CreateBlogPostHandler(
    IBlogPostRepository repository,
    ISlugUniquenessChecker slugChecker,
    ICurrentUserService currentUser) : IRequestHandler<CreateBlogPostCommand, BlogPostId>
{
    public async Task<BlogPostId> Handle(CreateBlogPostCommand cmd, CancellationToken ct)
    {
        // Authorization check
        if (currentUser.UserId != cmd.AuthorId && !currentUser.IsAdmin)
            throw new ForbiddenException("Cannot create post for another author");

        var post = BlogPost.Create(cmd.Title, cmd.Body, cmd.AuthorId);

        // Ensure slug uniqueness
        var slug = Slug.FromTitle(cmd.Title);
        if (!await slugChecker.IsUniqueAsync(slug, ct: ct))
        {
            // Append timestamp to make unique
            slug = Slug.FromTitle($"{cmd.Title}-{DateTimeOffset.UtcNow.ToUnixTimeSeconds()}");
        }

        foreach (var tagValue in cmd.Tags.Take(10))
            post.AddTag(Tag.Create(tagValue));

        await repository.AddAsync(post, ct);
        return post.Id;
    }
}

// Application/BlogPosts/Commands/CreatePost/CreateBlogPostValidator.cs
public class CreateBlogPostValidator : AbstractValidator<CreateBlogPostCommand>
{
    public CreateBlogPostValidator()
    {
        RuleFor(x => x.Title).NotEmpty().MaximumLength(200);
        RuleFor(x => x.Body).NotEmpty().MinimumLength(10);
        RuleFor(x => x.AuthorId).NotEmpty();
        RuleForEach(x => x.Tags).NotEmpty().MaximumLength(50);
    }
}
```

```csharp
// Application/BlogPosts/Commands/PublishPost/PublishBlogPostCommand.cs
public record PublishBlogPostCommand(Guid PostId) : IRequest;

// Application/BlogPosts/Commands/PublishPost/PublishBlogPostHandler.cs
public class PublishBlogPostHandler(
    IBlogPostRepository repository,
    ICurrentUserService currentUser) : IRequestHandler<PublishBlogPostCommand>
{
    public async Task Handle(PublishBlogPostCommand cmd, CancellationToken ct)
    {
        var post = await repository.GetByIdAsync(new BlogPostId(cmd.PostId), ct)
            ?? throw new NotFoundException(nameof(BlogPost), cmd.PostId);

        if (post.AuthorId.Value != currentUser.UserId && !currentUser.IsAdmin)
            throw new ForbiddenException("Cannot publish another author's post");

        post.Publish();
        await repository.UpdateAsync(post, ct);
    }
}
```

---

## Step 1056: Application Layer - Queries

```csharp
// Application/BlogPosts/Queries/GetPublishedPosts/GetPublishedPostsQuery.cs
public record GetPublishedPostsQuery(int Page = 1, int PageSize = 10)
    : IRequest<PagedResult<BlogPostSummaryDto>>;

// Application/BlogPosts/Queries/GetPublishedPosts/BlogPostSummaryDto.cs
public record BlogPostSummaryDto(
    Guid Id,
    string Title,
    string Slug,
    string AuthorName,
    DateTime PublishedAt,
    IReadOnlyList<string> Tags,
    int CommentCount);

// Application/BlogPosts/Queries/GetPublishedPosts/GetPublishedPostsHandler.cs
public class GetPublishedPostsHandler(IBlogPostQueryService queryService)
    : IRequestHandler<GetPublishedPostsQuery, PagedResult<BlogPostSummaryDto>>
{
    public async Task<PagedResult<BlogPostSummaryDto>> Handle(
        GetPublishedPostsQuery query, CancellationToken ct)
    {
        var skip = (query.Page - 1) * query.PageSize;
        return await queryService.GetPublishedPostsAsync(skip, query.PageSize, ct);
    }
}

// Application/BlogPosts/Queries/IBlogPostQueryService.cs
public interface IBlogPostQueryService
{
    Task<PagedResult<BlogPostSummaryDto>> GetPublishedPostsAsync(
        int skip, int take, CancellationToken ct = default);
    Task<BlogPostDetailDto?> GetBySlugAsync(string slug, CancellationToken ct = default);
}
```

```csharp
// Application/Common/PagedResult.cs
public record PagedResult<T>(
    IReadOnlyList<T> Items,
    int TotalCount,
    int Page,
    int PageSize)
{
    public int TotalPages => (int)Math.Ceiling(TotalCount / (double)PageSize);
    public bool HasNextPage => Page < TotalPages;
    public bool HasPreviousPage => Page > 1;
}
```

---

## Step 1057: Application Services - Ports

```csharp
// Application/Common/Services/ICurrentUserService.cs
public interface ICurrentUserService
{
    Guid UserId { get; }
    string UserName { get; }
    bool IsAdmin { get; }
    bool IsAuthenticated { get; }
}

// Application/Common/Services/IDateTimeService.cs
public interface IDateTimeService
{
    DateTime UtcNow { get; }
    DateTime LocalNow { get; }
}

// Application/Common/Services/IEmailService.cs
public interface IEmailService
{
    Task SendAsync(string to, string subject, string htmlBody, CancellationToken ct = default);
    Task SendTemplateAsync<T>(string to, string templateId, T data, CancellationToken ct = default);
}

// Application/Common/Services/ICacheService.cs
public interface ICacheService
{
    Task<T?> GetAsync<T>(string key, CancellationToken ct = default);
    Task SetAsync<T>(string key, T value, TimeSpan expiry, CancellationToken ct = default);
    Task RemoveAsync(string key, CancellationToken ct = default);
    Task RemoveByPatternAsync(string pattern, CancellationToken ct = default);
}
```

---

## Step 1058: Infrastructure Layer - EF Core

```csharp
// Infrastructure/Persistence/BlogDbContext.cs
public class BlogDbContext(
    DbContextOptions<BlogDbContext> options,
    ICurrentUserService currentUser,
    IDateTimeService dateTime) : DbContext(options)
{
    public DbSet<BlogPost> BlogPosts => Set<BlogPost>();
    public DbSet<Comment> Comments => Set<Comment>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(BlogDbContext).Assembly);

        // Global query filter: soft delete
        modelBuilder.Entity<BlogPost>().HasQueryFilter(p => !p.IsDeleted);
    }

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        optionsBuilder.AddInterceptors(new AuditInterceptor(currentUser, dateTime));
    }
}
```

```csharp
// Infrastructure/Persistence/Interceptors/AuditInterceptor.cs
public class AuditInterceptor(ICurrentUserService currentUser, IDateTimeService dateTime)
    : SaveChangesInterceptor
{
    public override InterceptionResult<int> SavingChanges(
        DbContextEventData eventData,
        InterceptionResult<int> result)
    {
        UpdateAuditFields(eventData.Context);
        return base.SavingChanges(eventData, result);
    }

    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData,
        InterceptionResult<int> result,
        CancellationToken ct = default)
    {
        UpdateAuditFields(eventData.Context);
        return base.SavingChangesAsync(eventData, result, ct);
    }

    private void UpdateAuditFields(DbContext? context)
    {
        if (context is null) return;
        var now = dateTime.UtcNow;
        var userId = currentUser.IsAuthenticated ? currentUser.UserId.ToString() : "system";

        foreach (var entry in context.ChangeTracker.Entries<IAuditable>())
        {
            if (entry.State == EntityState.Added)
            {
                entry.Entity.CreatedAt = now;
                entry.Entity.CreatedBy = userId;
            }
            if (entry.State is EntityState.Added or EntityState.Modified)
            {
                entry.Entity.UpdatedAt = now;
                entry.Entity.UpdatedBy = userId;
            }
        }
    }
}
```

---

## Step 1059: Infrastructure - Repository Implementation

```csharp
// Infrastructure/Repositories/BlogPostRepository.cs
public class BlogPostRepository(BlogDbContext db) : IBlogPostRepository
{
    public async Task<BlogPost?> GetByIdAsync(BlogPostId id, CancellationToken ct = default)
        => await db.BlogPosts
            .Include("_tags")   // private collection field
            .Include("_comments")
            .FirstOrDefaultAsync(p => p.Id == id, ct);

    public async Task<BlogPost?> GetBySlugAsync(Slug slug, CancellationToken ct = default)
        => await db.BlogPosts
            .Include("_tags")
            .FirstOrDefaultAsync(p => p.Slug == slug, ct);

    public async Task<IReadOnlyList<BlogPost>> GetPublishedAsync(
        int skip, int take, CancellationToken ct = default)
        => await db.BlogPosts
            .Where(p => p.Status == PublishStatus.Published)
            .OrderByDescending(p => p.PublishedAt)
            .Skip(skip)
            .Take(take)
            .ToListAsync(ct);

    public async Task<int> CountPublishedAsync(CancellationToken ct = default)
        => await db.BlogPosts
            .CountAsync(p => p.Status == PublishStatus.Published, ct);

    public async Task AddAsync(BlogPost post, CancellationToken ct = default)
    {
        await db.BlogPosts.AddAsync(post, ct);
        await db.SaveChangesAsync(ct);
    }

    public async Task UpdateAsync(BlogPost post, CancellationToken ct = default)
    {
        db.BlogPosts.Update(post);
        await db.SaveChangesAsync(ct);
    }

    public async Task DeleteAsync(BlogPostId id, CancellationToken ct = default)
    {
        var post = await db.BlogPosts.FindAsync([id], ct)
            ?? throw new NotFoundException(nameof(BlogPost), id);
        db.BlogPosts.Remove(post);
        await db.SaveChangesAsync(ct);
    }
}
```

---

## Step 1060: Infrastructure - Query Service (Read Side)

Query Service ใช้ raw SQL หรือ projection โดยตรง ไม่ผ่าน Domain model

```csharp
// Infrastructure/QueryServices/BlogPostQueryService.cs
public class BlogPostQueryService(BlogDbContext db) : IBlogPostQueryService
{
    public async Task<PagedResult<BlogPostSummaryDto>> GetPublishedPostsAsync(
        int skip, int take, CancellationToken ct = default)
    {
        var query = db.BlogPosts
            .Where(p => p.Status == PublishStatus.Published)
            .OrderByDescending(p => p.PublishedAt);

        var total = await query.CountAsync(ct);
        var items = await query
            .Skip(skip).Take(take)
            .Select(p => new BlogPostSummaryDto(
                p.Id.Value,
                p.Title.Value,
                p.Slug.Value,
                p.Author.Name.FullName,      // joined navigation
                p.PublishedAt!.Value,
                p.Tags.Select(t => t.Value).ToList(),
                p.Comments.Count))
            .ToListAsync(ct);

        return new PagedResult<BlogPostSummaryDto>(items, total, skip / take + 1, take);
    }

    public async Task<BlogPostDetailDto?> GetBySlugAsync(string slug, CancellationToken ct = default)
    {
        return await db.BlogPosts
            .Where(p => p.Slug == new Slug(slug) && p.Status == PublishStatus.Published)
            .Select(p => new BlogPostDetailDto(
                p.Id.Value,
                p.Title.Value,
                p.Body.Value,
                p.Slug.Value,
                p.Author.Name.FullName,
                p.Author.Email.Value,
                p.PublishedAt!.Value,
                p.Tags.Select(t => t.Value).ToList(),
                p.Comments.Select(c => new CommentDto(
                    c.Id.Value,
                    c.Author.Name.FullName,
                    c.Text.Value,
                    c.CreatedAt)).ToList()))
            .FirstOrDefaultAsync(ct);
    }
}
```

---

## Step 1061: Infrastructure - Services

```csharp
// Infrastructure/Services/CurrentUserService.cs
public class CurrentUserService(IHttpContextAccessor httpContextAccessor) : ICurrentUserService
{
    private readonly ClaimsPrincipal? _user = httpContextAccessor.HttpContext?.User;

    public Guid UserId
    {
        get
        {
            var claim = _user?.FindFirst(ClaimTypes.NameIdentifier)?.Value
                ?? throw new UnauthorizedException("User not authenticated");
            return Guid.Parse(claim);
        }
    }

    public string UserName => _user?.FindFirst(ClaimTypes.Name)?.Value ?? string.Empty;
    public bool IsAdmin => _user?.IsInRole("Admin") ?? false;
    public bool IsAuthenticated => _user?.Identity?.IsAuthenticated ?? false;
}

// Infrastructure/Services/DateTimeService.cs
public class DateTimeService : IDateTimeService
{
    public DateTime UtcNow => DateTime.UtcNow;
    public DateTime LocalNow => DateTime.Now;
}

// Infrastructure/Services/EmailService.cs
public class EmailService(
    IOptions<EmailSettings> settings,
    ILogger<EmailService> logger) : IEmailService
{
    private readonly EmailSettings _settings = settings.Value;

    public async Task SendAsync(string to, string subject, string htmlBody, CancellationToken ct)
    {
        logger.LogInformation("Sending email to {To}: {Subject}", to, subject);

        using var smtp = new SmtpClient();
        await smtp.ConnectAsync(_settings.Host, _settings.Port, SecureSocketOptions.StartTls, ct);
        await smtp.AuthenticateAsync(_settings.Username, _settings.Password, ct);

        var message = new MimeMessage();
        message.From.Add(new MailboxAddress(_settings.SenderName, _settings.SenderEmail));
        message.To.Add(new MailboxAddress(string.Empty, to));
        message.Subject = subject;
        message.Body = new TextPart(TextFormat.Html) { Text = htmlBody };

        await smtp.SendAsync(message, ct);
        await smtp.DisconnectAsync(true, ct);
    }

    public async Task SendTemplateAsync<T>(string to, string templateId, T data, CancellationToken ct)
    {
        // Integration with email service like SendGrid, Mailchimp, etc.
        logger.LogInformation("Sending template {TemplateId} to {To}", templateId, to);
        await Task.CompletedTask;
    }
}
```

---

## Step 1062: Infrastructure - Caching

```csharp
// Infrastructure/Services/RedisCacheService.cs
public class RedisCacheService(IConnectionMultiplexer redis, ILogger<RedisCacheService> logger)
    : ICacheService
{
    private readonly IDatabase _db = redis.GetDatabase();

    public async Task<T?> GetAsync<T>(string key, CancellationToken ct = default)
    {
        var value = await _db.StringGetAsync(key);
        if (!value.HasValue) return default;

        try
        {
            return JsonSerializer.Deserialize<T>(value!);
        }
        catch (JsonException ex)
        {
            logger.LogWarning(ex, "Failed to deserialize cache value for key {Key}", key);
            return default;
        }
    }

    public async Task SetAsync<T>(string key, T value, TimeSpan expiry, CancellationToken ct = default)
    {
        var json = JsonSerializer.Serialize(value);
        await _db.StringSetAsync(key, json, expiry);
    }

    public async Task RemoveAsync(string key, CancellationToken ct = default)
        => await _db.KeyDeleteAsync(key);

    public async Task RemoveByPatternAsync(string pattern, CancellationToken ct = default)
    {
        var server = redis.GetServer(redis.GetEndPoints().First());
        var keys = server.Keys(pattern: pattern).ToArray();
        if (keys.Length > 0)
            await _db.KeyDeleteAsync(keys);
    }
}
```

---

## Step 1063: Infrastructure - Dependency Injection

```csharp
// Infrastructure/DependencyInjection.cs
public static class DependencyInjection
{
    public static IServiceCollection AddInfrastructure(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        // Database
        services.AddDbContext<BlogDbContext>(options =>
            options.UseNpgsql(configuration.GetConnectionString("DefaultConnection"),
                b => b.MigrationsAssembly(typeof(BlogDbContext).Assembly.FullName)));

        // Repositories (Ports implementations)
        services.AddScoped<IBlogPostRepository, BlogPostRepository>();

        // Query Services (Read Side)
        services.AddScoped<IBlogPostQueryService, BlogPostQueryService>();

        // Application Services (Port implementations)
        services.AddScoped<ICurrentUserService, CurrentUserService>();
        services.AddSingleton<IDateTimeService, DateTimeService>();
        services.AddScoped<IEmailService, EmailService>();
        services.AddScoped<ISlugUniquenessChecker, SlugUniquenessChecker>();
        services.AddScoped<IFileStorageService, LocalFileStorageService>();

        // Cache
        services.AddStackExchangeRedisCache(options =>
            options.Configuration = configuration.GetConnectionString("Redis"));
        services.AddSingleton<IConnectionMultiplexer>(sp =>
            ConnectionMultiplexer.Connect(configuration.GetConnectionString("Redis")!));
        services.AddScoped<ICacheService, RedisCacheService>();

        // Email
        services.Configure<EmailSettings>(configuration.GetSection("Email"));
        services.AddHttpContextAccessor();

        return services;
    }
}
```

---

## Step 1064: API Layer - Minimal API

```csharp
// API/Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddApplication()
    .AddInfrastructure(builder.Configuration);

builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidAudience = builder.Configuration["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]!))
        };
    });

builder.Services.AddAuthorization();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
    await app.MigrateDbAsync();
}

app.UseAuthentication();
app.UseAuthorization();
app.UseMiddleware<ExceptionHandlingMiddleware>();

app.MapBlogPostEndpoints();
app.MapAuthEndpoints();

app.Run();
```

```csharp
// API/Endpoints/BlogPostEndpoints.cs
public static class BlogPostEndpoints
{
    public static void MapBlogPostEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/posts").WithTags("BlogPosts");

        group.MapGet("/", GetPublishedPostsAsync)
            .WithName("GetPublishedPosts")
            .Produces<PagedResult<BlogPostSummaryDto>>();

        group.MapGet("/{slug}", GetPostBySlugAsync)
            .WithName("GetPostBySlug")
            .Produces<BlogPostDetailDto>()
            .ProducesProblem(404);

        group.MapPost("/", CreatePostAsync)
            .WithName("CreatePost")
            .RequireAuthorization()
            .Produces<Guid>(201)
            .ProducesValidationProblem();

        group.MapPut("/{id:guid}", UpdatePostAsync)
            .WithName("UpdatePost")
            .RequireAuthorization()
            .Produces(204);

        group.MapPost("/{id:guid}/publish", PublishPostAsync)
            .WithName("PublishPost")
            .RequireAuthorization("Admin")
            .Produces(204);

        group.MapDelete("/{id:guid}", DeletePostAsync)
            .WithName("DeletePost")
            .RequireAuthorization()
            .Produces(204);
    }

    private static async Task<IResult> GetPublishedPostsAsync(
        [AsParameters] GetPublishedPostsQuery query,
        ISender sender, CancellationToken ct)
    {
        var result = await sender.Send(query, ct);
        return Results.Ok(result);
    }

    private static async Task<IResult> GetPostBySlugAsync(
        string slug, ISender sender, CancellationToken ct)
    {
        var post = await sender.Send(new GetPostBySlugQuery(slug), ct);
        return post is null ? Results.NotFound() : Results.Ok(post);
    }

    private static async Task<IResult> CreatePostAsync(
        CreateBlogPostRequest request,
        ISender sender,
        ICurrentUserService currentUser,
        CancellationToken ct)
    {
        var command = new CreateBlogPostCommand(
            request.Title, request.Body, currentUser.UserId, request.Tags);
        var id = await sender.Send(command, ct);
        return Results.CreatedAtRoute("GetPostBySlug", new { slug = id.Value }, id.Value);
    }

    private static async Task<IResult> UpdatePostAsync(
        Guid id,
        UpdateBlogPostRequest request,
        ISender sender, CancellationToken ct)
    {
        await sender.Send(new UpdateBlogPostCommand(id, request.Title, request.Body), ct);
        return Results.NoContent();
    }

    private static async Task<IResult> PublishPostAsync(
        Guid id, ISender sender, CancellationToken ct)
    {
        await sender.Send(new PublishBlogPostCommand(id), ct);
        return Results.NoContent();
    }

    private static async Task<IResult> DeletePostAsync(
        Guid id, ISender sender, CancellationToken ct)
    {
        await sender.Send(new DeleteBlogPostCommand(id), ct);
        return Results.NoContent();
    }
}
```

---

## Step 1065: API Versioning

```csharp
// API/Program.cs - เพิ่ม versioning
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1);
    options.ReportApiVersions = true;
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ApiVersionReader = ApiVersionReader.Combine(
        new UrlSegmentApiVersionReader(),
        new HeaderApiVersionReader("X-API-Version"),
        new QueryStringApiVersionReader("api-version"));
}).AddApiExplorer(options =>
{
    options.GroupNameFormat = "'v'VVV";
    options.SubstituteApiVersionInUrl = true;
});

// API/Endpoints/BlogPostEndpoints.cs - versioned
var v1 = app.NewVersionedApi("BlogPosts").MapGroup("/api/v{version:apiVersion}/posts")
    .HasApiVersion(1);
var v2 = app.NewVersionedApi("BlogPosts").MapGroup("/api/v{version:apiVersion}/posts")
    .HasApiVersion(2);

v1.MapGet("/", GetPublishedPostsV1Async);
v2.MapGet("/", GetPublishedPostsV2Async); // enhanced response
```

---

## Step 1066: Exception Handling

```csharp
// API/Middleware/ExceptionHandlingMiddleware.cs
public class ExceptionHandlingMiddleware(
    RequestDelegate next,
    ILogger<ExceptionHandlingMiddleware> logger,
    IHostEnvironment env)
{
    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await next(context);
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "Unhandled exception: {Message}", ex.Message);
            await HandleExceptionAsync(context, ex);
        }
    }

    private async Task HandleExceptionAsync(HttpContext context, Exception exception)
    {
        var (status, title) = exception switch
        {
            ValidationException => (400, "Validation Error"),
            DomainException => (400, "Business Rule Violation"),
            NotFoundException => (404, "Not Found"),
            ForbiddenException => (403, "Forbidden"),
            UnauthorizedException => (401, "Unauthorized"),
            ConflictException => (409, "Conflict"),
            _ => (500, "Internal Server Error")
        };

        var problem = new ProblemDetails
        {
            Status = status,
            Title = title,
            Type = $"https://tools.ietf.org/html/rfc7231#section-6.{status / 100}.{status % 100}",
        };

        if (exception is ValidationException validationEx)
        {
            problem.Extensions["errors"] = validationEx.Errors
                .GroupBy(e => e.PropertyName)
                .ToDictionary(
                    g => g.Key,
                    g => g.Select(e => e.ErrorMessage).ToArray());
        }
        else
        {
            problem.Detail = exception.Message;
        }

        if (env.IsDevelopment())
            problem.Extensions["stackTrace"] = exception.StackTrace;

        context.Response.StatusCode = status;
        context.Response.ContentType = "application/problem+json";
        await context.Response.WriteAsJsonAsync(problem);
    }
}
```

---

## Step 1067: Architecture Tests

ArchUnit-style tests ตรวจสอบ dependency rule อัตโนมัติ

```xml
<!-- Tests/Architecture/CleanApp.Tests.Architecture.csproj -->
<PackageReference Include="NetArchTest.Rules" Version="1.3.2" />
```

```csharp
// Tests/Architecture/ArchitectureTests.cs
public class ArchitectureTests
{
    private const string DomainNamespace = "CleanApp.Domain";
    private const string ApplicationNamespace = "CleanApp.Application";
    private const string InfrastructureNamespace = "CleanApp.Infrastructure";
    private const string ApiNamespace = "CleanApp.API";

    [Fact]
    public void Domain_ShouldNotDependOnApplication()
    {
        var result = Types.InAssembly(DomainAssembly)
            .ShouldNot().HaveDependencyOn(ApplicationNamespace)
            .GetResult();

        result.IsSuccessful.Should().BeTrue(
            because: result.FailingTypeNames != null
                ? string.Join(", ", result.FailingTypeNames)
                : "Domain should be independent");
    }

    [Fact]
    public void Domain_ShouldNotDependOnInfrastructure()
    {
        var result = Types.InAssembly(DomainAssembly)
            .ShouldNot().HaveDependencyOn(InfrastructureNamespace)
            .GetResult();

        result.IsSuccessful.Should().BeTrue();
    }

    [Fact]
    public void Application_ShouldNotDependOnInfrastructure()
    {
        var result = Types.InAssembly(ApplicationAssembly)
            .ShouldNot().HaveDependencyOn(InfrastructureNamespace)
            .GetResult();

        result.IsSuccessful.Should().BeTrue(
            because: "Application should not depend on Infrastructure (Dependency Inversion)");
    }

    [Fact]
    public void Application_ShouldNotDependOnApi()
    {
        var result = Types.InAssembly(ApplicationAssembly)
            .ShouldNot().HaveDependencyOn(ApiNamespace)
            .GetResult();

        result.IsSuccessful.Should().BeTrue();
    }

    [Fact]
    public void Domain_Entities_ShouldNotBePubliclyMutable()
    {
        var result = Types.InAssembly(DomainAssembly)
            .That().Inherit(typeof(Entity<>))
            .Should().HaveOnlyPrivateConstructors()
            .GetResult();

        // Properties should use private setters
        // This checks constructors are not public (factory method pattern)
        result.IsSuccessful.Should().BeTrue();
    }

    [Fact]
    public void Handlers_ShouldBeInApplicationLayer()
    {
        var result = Types.InAssembly(ApplicationAssembly)
            .That().ImplementInterface(typeof(IRequestHandler<,>))
            .Should().ResideInNamespaceContaining(ApplicationNamespace)
            .GetResult();

        result.IsSuccessful.Should().BeTrue();
    }

    [Fact]
    public void Repositories_ShouldOnlyBeInInfrastructure()
    {
        var result = Types.InAssemblies([DomainAssembly, ApplicationAssembly])
            .That().HaveNameEndingWith("Repository")
            .And().AreNotInterfaces()
            .Should().NotExist()
            .GetResult();

        result.IsSuccessful.Should().BeTrue(
            because: "Concrete repositories should only be in Infrastructure layer");
    }

    [Fact]
    public void ValueObjects_ShouldBeSealed()
    {
        var result = Types.InAssembly(DomainAssembly)
            .That().Inherit(typeof(ValueObject))
            .Should().BeSealed()
            .GetResult();

        result.IsSuccessful.Should().BeTrue();
    }

    private static Assembly DomainAssembly => typeof(BlogPost).Assembly;
    private static Assembly ApplicationAssembly => typeof(CreateBlogPostCommand).Assembly;
    private static Assembly InfrastructureAssembly => typeof(BlogDbContext).Assembly;
}
```

---

## Step 1068: Testing Use Cases

```csharp
// Tests/Unit/Application/CreateBlogPostHandlerTests.cs
public class CreateBlogPostHandlerTests
{
    private readonly Mock<IBlogPostRepository> _mockRepo = new();
    private readonly Mock<ISlugUniquenessChecker> _mockSlugChecker = new();
    private readonly Mock<ICurrentUserService> _mockCurrentUser = new();

    [Fact]
    public async Task Handle_ValidCommand_ShouldCreateAndReturnId()
    {
        // Arrange
        var authorId = Guid.NewGuid();
        _mockCurrentUser.Setup(s => s.UserId).Returns(authorId);
        _mockCurrentUser.Setup(s => s.IsAdmin).Returns(false);
        _mockSlugChecker.Setup(s => s.IsUniqueAsync(It.IsAny<Slug>(), null, default))
            .ReturnsAsync(true);
        _mockRepo.Setup(r => r.AddAsync(It.IsAny<BlogPost>(), default))
            .Returns(Task.CompletedTask);

        var handler = new CreateBlogPostHandler(
            _mockRepo.Object, _mockSlugChecker.Object, _mockCurrentUser.Object);

        var command = new CreateBlogPostCommand(
            "My First Post",
            "This is the body of the post with enough content",
            authorId,
            ["dotnet", "csharp"]);

        // Act
        var id = await handler.Handle(command, default);

        // Assert
        id.Should().NotBeNull();
        _mockRepo.Verify(r => r.AddAsync(
            It.Is<BlogPost>(p =>
                p.Title.Value == "My First Post" &&
                p.Tags.Count == 2),
            default), Times.Once);
    }

    [Fact]
    public async Task Handle_DifferentAuthor_ShouldThrowForbidden()
    {
        _mockCurrentUser.Setup(s => s.UserId).Returns(Guid.NewGuid());
        _mockCurrentUser.Setup(s => s.IsAdmin).Returns(false);

        var handler = new CreateBlogPostHandler(
            _mockRepo.Object, _mockSlugChecker.Object, _mockCurrentUser.Object);

        var command = new CreateBlogPostCommand(
            "Post", "Body content here minimum",
            Guid.NewGuid(), // different author
            []);

        await handler.Invoking(h => h.Handle(command, default))
            .Should().ThrowAsync<ForbiddenException>();
    }
}
```

---

## Step 1069: Database Migration Strategy

```csharp
// Infrastructure/Migrations/DbMigrator.cs
public static class DbMigrator
{
    public static async Task MigrateDbAsync(this WebApplication app)
    {
        using var scope = app.Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<BlogDbContext>();
        var logger = scope.ServiceProvider.GetRequiredService<ILogger<BlogDbContext>>();

        try
        {
            await db.Database.MigrateAsync();
            logger.LogInformation("Database migration completed");

            // Seed initial data
            await SeedDataAsync(db, logger);
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "Database migration failed");
            throw;
        }
    }

    private static async Task SeedDataAsync(BlogDbContext db, ILogger logger)
    {
        if (!await db.BlogPosts.AnyAsync())
        {
            logger.LogInformation("Seeding initial blog posts...");
            // Add seed data
        }
    }
}
```

---

## Step 1070: Health Checks

```csharp
// API/HealthChecks/BlogHealthCheck.cs
public class BlogHealthCheck(BlogDbContext db) : IHealthCheck
{
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken ct = default)
    {
        try
        {
            await db.Database.CanConnectAsync(ct);
            var postCount = await db.BlogPosts.CountAsync(ct);
            return HealthCheckResult.Healthy($"Database OK, {postCount} posts");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("Database connection failed", ex);
        }
    }
}

// In Program.cs
builder.Services.AddHealthChecks()
    .AddCheck<BlogHealthCheck>("blog-db")
    .AddNpgSql(builder.Configuration.GetConnectionString("DefaultConnection")!)
    .AddRedis(builder.Configuration.GetConnectionString("Redis")!);

app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false // liveness: just return 200
});
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});
```

---

## Step 1071: Output Caching + Cache Invalidation

```csharp
// Application/BlogPosts/Queries/GetPublishedPosts/GetPublishedPostsHandler.cs - with cache
public class GetPublishedPostsHandler(
    IBlogPostQueryService queryService,
    ICacheService cache) : IRequestHandler<GetPublishedPostsQuery, PagedResult<BlogPostSummaryDto>>
{
    public async Task<PagedResult<BlogPostSummaryDto>> Handle(
        GetPublishedPostsQuery query, CancellationToken ct)
    {
        var cacheKey = $"posts:published:p{query.Page}:s{query.PageSize}";
        var cached = await cache.GetAsync<PagedResult<BlogPostSummaryDto>>(cacheKey, ct);
        if (cached is not null) return cached;

        var skip = (query.Page - 1) * query.PageSize;
        var result = await queryService.GetPublishedPostsAsync(skip, query.PageSize, ct);

        await cache.SetAsync(cacheKey, result, TimeSpan.FromMinutes(10), ct);
        return result;
    }
}

// Application/BlogPosts/EventHandlers/BlogPostPublishedEventHandler.cs
public class BlogPostPublishedEventHandler(ICacheService cache)
    : INotificationHandler<BlogPostPublishedEvent>
{
    public async Task Handle(BlogPostPublishedEvent notification, CancellationToken ct)
    {
        // Invalidate list cache when new post is published
        await cache.RemoveByPatternAsync("posts:published:*", ct);
    }
}
```

---

## Step 1072: Request/Response Logging

```csharp
// API/Middleware/RequestLoggingMiddleware.cs
public class RequestLoggingMiddleware(RequestDelegate next, ILogger<RequestLoggingMiddleware> logger)
{
    public async Task InvokeAsync(HttpContext context)
    {
        var sw = Stopwatch.StartNew();
        var requestId = context.TraceIdentifier;
        var method = context.Request.Method;
        var path = context.Request.Path;

        logger.LogInformation("Request [{RequestId}] {Method} {Path}", requestId, method, path);

        await next(context);

        sw.Stop();
        var statusCode = context.Response.StatusCode;
        var level = statusCode >= 500 ? LogLevel.Error
            : statusCode >= 400 ? LogLevel.Warning
            : LogLevel.Information;

        logger.Log(level, "Response [{RequestId}] {StatusCode} {Method} {Path} in {ElapsedMs}ms",
            requestId, statusCode, method, path, sw.ElapsedMilliseconds);
    }
}
```

---

## Step 1073: Rate Limiting

```csharp
// API/RateLimiting/RateLimitingConfiguration.cs
public static class RateLimitingConfiguration
{
    public static IServiceCollection AddRateLimiting(this IServiceCollection services)
    {
        services.AddRateLimiter(options =>
        {
            options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;

            // Global: 100 req/min
            options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(
                context => RateLimitPartition.GetFixedWindowLimiter(
                    context.Connection.RemoteIpAddress?.ToString() ?? "unknown",
                    _ => new FixedWindowRateLimiterOptions
                    {
                        Window = TimeSpan.FromMinutes(1),
                        PermitLimit = 100,
                        QueueProcessingOrder = QueueProcessingOrder.OldestFirst,
                        QueueLimit = 0
                    }));

            // Authenticated: 1000 req/min
            options.AddPolicy("authenticated", context =>
            {
                if (context.User.Identity?.IsAuthenticated == true)
                {
                    var userId = context.User.FindFirst(ClaimTypes.NameIdentifier)?.Value ?? "unknown";
                    return RateLimitPartition.GetSlidingWindowLimiter(
                        userId,
                        _ => new SlidingWindowRateLimiterOptions
                        {
                            Window = TimeSpan.FromMinutes(1),
                            SegmentsPerWindow = 6,
                            PermitLimit = 1000
                        });
                }

                return RateLimitPartition.GetNoLimiter(string.Empty);
            });

            // Write operations: 10 req/min per user
            options.AddPolicy("write", context =>
            {
                var userId = context.User.FindFirst(ClaimTypes.NameIdentifier)?.Value ?? "anon";
                return RateLimitPartition.GetTokenBucketLimiter(
                    $"write:{userId}",
                    _ => new TokenBucketRateLimiterOptions
                    {
                        TokenLimit = 10,
                        TokensPerPeriod = 5,
                        ReplenishmentPeriod = TimeSpan.FromMinutes(1)
                    });
            });
        });

        return services;
    }
}
```

---

## Step 1074: CORS Configuration

```csharp
// API/CorsConfiguration.cs
public static class CorsConfiguration
{
    public static IServiceCollection AddCorsPolicy(
        this IServiceCollection services, IConfiguration config)
    {
        var origins = config.GetSection("Cors:AllowedOrigins").Get<string[]>() ?? [];

        services.AddCors(options =>
        {
            options.AddPolicy("DefaultPolicy", builder =>
            {
                if (origins.Length > 0)
                    builder.WithOrigins(origins);
                else
                    builder.AllowAnyOrigin();

                builder
                    .WithMethods("GET", "POST", "PUT", "DELETE", "PATCH")
                    .WithHeaders("Authorization", "Content-Type", "X-API-Version")
                    .AllowCredentials()
                    .SetPreflightMaxAge(TimeSpan.FromMinutes(10));
            });

            options.AddPolicy("PublicReadPolicy", builder =>
                builder.AllowAnyOrigin()
                    .WithMethods("GET")
                    .WithHeaders("Content-Type"));
        });

        return services;
    }
}
```

---

## Step 1075: OpenTelemetry Integration

```csharp
// Infrastructure/Observability/OpenTelemetryConfiguration.cs
public static class OpenTelemetryConfiguration
{
    public static IServiceCollection AddObservability(
        this IServiceCollection services, IConfiguration config)
    {
        services.AddOpenTelemetry()
            .ConfigureResource(resource => resource
                .AddService("CleanApp.API",
                    serviceVersion: "1.0.0",
                    serviceInstanceId: Environment.MachineName))
            .WithTracing(tracing =>
            {
                tracing
                    .AddAspNetCoreInstrumentation(options =>
                    {
                        options.RecordException = true;
                        options.Filter = ctx => !ctx.Request.Path.StartsWithSegments("/health");
                    })
                    .AddEntityFrameworkCoreInstrumentation()
                    .AddRedisInstrumentation()
                    .AddOtlpExporter(options =>
                        options.Endpoint = new Uri(config["Otlp:Endpoint"]!));
            })
            .WithMetrics(metrics =>
            {
                metrics
                    .AddAspNetCoreInstrumentation()
                    .AddRuntimeInstrumentation()
                    .AddPrometheusExporter();
            });

        return services;
    }
}
```

---

## Step 1076: Background Services

```csharp
// Application/BackgroundJobs/PublishScheduledPostsJob.cs
public class PublishScheduledPostsJob(
    IServiceScopeFactory scopeFactory,
    ILogger<PublishScheduledPostsJob> logger) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await DoWorkAsync(stoppingToken);
            await Task.Delay(TimeSpan.FromMinutes(1), stoppingToken);
        }
    }

    private async Task DoWorkAsync(CancellationToken ct)
    {
        using var scope = scopeFactory.CreateScope();
        var sender = scope.ServiceProvider.GetRequiredService<ISender>();

        try
        {
            await sender.Send(new PublishScheduledPostsCommand(), ct);
        }
        catch (Exception ex) when (ex is not OperationCanceledException)
        {
            logger.LogError(ex, "Error publishing scheduled posts");
        }
    }
}

// Register in Program.cs
builder.Services.AddHostedService<PublishScheduledPostsJob>();
```

---

## Step 1077: Integration Testing with WebApplicationFactory

```csharp
// Tests/Integration/BlogPostApiTests.cs
public class BlogPostApiTests : IClassFixture<BlogApiFactory>
{
    private readonly HttpClient _client;

    public BlogPostApiTests(BlogApiFactory factory)
    {
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task GetPublishedPosts_ReturnsPagedResult()
    {
        var response = await _client.GetAsync("/api/v1/posts");
        response.EnsureSuccessStatusCode();

        var result = await response.Content.ReadFromJsonAsync<PagedResult<BlogPostSummaryDto>>();
        result.Should().NotBeNull();
        result!.Items.Should().NotBeNull();
    }

    [Fact]
    public async Task CreatePost_Unauthorized_Returns401()
    {
        var request = new { title = "Test", body = "Content", tags = new[] { "test" } };
        var response = await _client.PostAsJsonAsync("/api/v1/posts", request);
        response.StatusCode.Should().Be(HttpStatusCode.Unauthorized);
    }
}

// Tests/Integration/BlogApiFactory.cs
public class BlogApiFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureTestServices(services =>
        {
            // Replace real DB with in-memory
            var descriptor = services.Single(d => d.ServiceType == typeof(DbContextOptions<BlogDbContext>));
            services.Remove(descriptor);

            services.AddDbContext<BlogDbContext>(options =>
                options.UseInMemoryDatabase("TestBlog"));

            // Replace email service with mock
            services.AddScoped<IEmailService, NoOpEmailService>();
        });
    }
}

public class NoOpEmailService : IEmailService
{
    public Task SendAsync(string to, string subject, string htmlBody, CancellationToken ct) => Task.CompletedTask;
    public Task SendTemplateAsync<T>(string to, string templateId, T data, CancellationToken ct) => Task.CompletedTask;
}
```

---

## Step 1078: Decorator Pattern in Application Layer

```csharp
// Application/Common/Decorators/CachingDecorator.cs
public class CachingBlogPostQueryService(
    IBlogPostQueryService inner,
    ICacheService cache) : IBlogPostQueryService
{
    public async Task<PagedResult<BlogPostSummaryDto>> GetPublishedPostsAsync(
        int skip, int take, CancellationToken ct)
    {
        var key = $"blog:posts:{skip}:{take}";
        return await cache.GetOrCreateAsync(key, TimeSpan.FromMinutes(5),
            () => inner.GetPublishedPostsAsync(skip, take, ct), ct);
    }

    public async Task<BlogPostDetailDto?> GetBySlugAsync(string slug, CancellationToken ct)
    {
        var key = $"blog:post:{slug}";
        return await cache.GetOrCreateAsync(key, TimeSpan.FromMinutes(30),
            () => inner.GetBySlugAsync(slug, ct), ct);
    }
}

// Extension on ICacheService
public static class CacheServiceExtensions
{
    public static async Task<T> GetOrCreateAsync<T>(
        this ICacheService cache,
        string key,
        TimeSpan expiry,
        Func<Task<T>> factory,
        CancellationToken ct = default)
    {
        var cached = await cache.GetAsync<T>(key, ct);
        if (cached is not null) return cached;

        var value = await factory();
        await cache.SetAsync(key, value, expiry, ct);
        return value;
    }
}
```

---

## Step 1079: Feature Flags

```csharp
// Application/Common/Features/IFeatureFlagService.cs
public interface IFeatureFlagService
{
    Task<bool> IsEnabledAsync(string featureName, CancellationToken ct = default);
}

// Infrastructure/Services/FeatureFlagService.cs
public class FeatureFlagService(IConfiguration config) : IFeatureFlagService
{
    public Task<bool> IsEnabledAsync(string featureName, CancellationToken ct = default)
    {
        var isEnabled = config.GetValue($"Features:{featureName}", false);
        return Task.FromResult(isEnabled);
    }
}

// Feature flag usage in handler
public class GetPublishedPostsHandler(
    IBlogPostQueryService queryService,
    IFeatureFlagService features) : IRequestHandler<GetPublishedPostsQuery, PagedResult<BlogPostSummaryDto>>
{
    public async Task<PagedResult<BlogPostSummaryDto>> Handle(
        GetPublishedPostsQuery query, CancellationToken ct)
    {
        var useEnhancedSearch = await features.IsEnabledAsync("EnhancedSearch", ct);
        if (useEnhancedSearch)
        {
            // Use new search implementation
        }

        var skip = (query.Page - 1) * query.PageSize;
        return await queryService.GetPublishedPostsAsync(skip, query.PageSize, ct);
    }
}
```

---

## Step 1080: สรุป Clean Architecture

```
Clean Architecture Layers:
┌──────────────────────────────────────────────────────┐
│                   API Layer                           │
│  - Minimal API / Controllers                         │
│  - Middleware (Auth, Logging, Rate Limiting)         │
│  - DI Composition Root                               │
├──────────────────────────────────────────────────────┤
│              Infrastructure Layer                     │
│  - EF Core (Repository implementations)              │
│  - Redis (Cache service)                             │
│  - Email, File Storage implementations               │
│  - External API clients                              │
├──────────────────────────────────────────────────────┤
│               Application Layer                       │
│  - Use Cases (Commands / Queries)                    │
│  - MediatR Handlers                                  │
│  - Pipeline Behaviors (Validation, Logging, Cache)   │
│  - Application Service Interfaces (Ports)            │
├──────────────────────────────────────────────────────┤
│                  Domain Layer                         │
│  - Entities, Value Objects, Aggregates               │
│  - Domain Events                                     │
│  - Repository Interfaces                             │
│  - Domain Services                                   │
│  - Business Rules & Invariants                       │
└──────────────────────────────────────────────────────┘

Key Principles:
✅ Dependency Rule: dependencies point inward
✅ Ports & Adapters: interfaces defined in Domain/Application
✅ SOLID: each class has single responsibility
✅ Testable: all layers independently testable
✅ Framework-agnostic: Domain has zero external dependencies
```

---

*Part 37 ครอบคลุม Clean Architecture ทั้งหมด 30 Steps (1051-1080)*  
*ต่อไป Part 38: Event-Driven Architecture*
