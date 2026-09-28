# Part 81: Clean Architecture & Vertical Slice Architecture

## Steps 1925–1940

---

## Step 1925: Clean Architecture Fundamentals

The Dependency Rule: source code dependencies point inward. Outer layers depend on inner layers, never the reverse.

```
┌─────────────────────────────────────────────┐
│             Infrastructure                   │
│  ┌────────────────────────────────────────┐  │
│  │           Application                  │  │
│  │  ┌──────────────────────────────────┐  │  │
│  │  │            Domain                │  │  │
│  │  │  Entities, Value Objects         │  │  │
│  │  │  Domain Services, Events         │  │  │
│  │  │  Repository Interfaces           │  │  │
│  │  └──────────────────────────────────┘  │  │
│  │  Use Cases, DTOs, Validators           │  │  │
│  │  Application Service Interfaces        │  │  │
│  └────────────────────────────────────────┘  │
│  EF Core, HTTP Clients, File System          │  │
└─────────────────────────────────────────────┘
         ↑ Presentation (API, CLI, Workers)
```

**Layer responsibilities**:
- **Domain**: Business rules, entities, value objects — pure C#, no framework dependencies
- **Application**: Use cases, orchestration, DTOs, interface definitions
- **Infrastructure**: Framework implementations — EF Core, HTTP, file I/O
- **Presentation**: API controllers, SignalR hubs, CLI commands — thin, delegates to Application

---

## Step 1926: Solution Structure

```
CleanApp/
├── src/
│   ├── CleanApp.Domain/
│   │   ├── Entities/
│   │   ├── ValueObjects/
│   │   ├── Events/
│   │   ├── Exceptions/
│   │   ├── Repositories/         ← interfaces only
│   │   └── Services/             ← domain services
│   ├── CleanApp.Application/
│   │   ├── Common/
│   │   │   ├── Behaviors/        ← MediatR pipeline
│   │   │   ├── Exceptions/
│   │   │   ├── Interfaces/       ← ICurrentUser, IDateTime, etc.
│   │   │   └── Models/           ← PagedResult, Result<T>
│   │   ├── Orders/
│   │   │   ├── Commands/
│   │   │   │   ├── PlaceOrder/
│   │   │   │   │   ├── PlaceOrderCommand.cs
│   │   │   │   │   ├── PlaceOrderCommandHandler.cs
│   │   │   │   │   └── PlaceOrderCommandValidator.cs
│   │   │   │   └── ConfirmOrder/
│   │   │   └── Queries/
│   │   │       ├── GetOrder/
│   │   │       └── ListOrders/
│   │   └── Customers/
│   ├── CleanApp.Infrastructure/
│   │   ├── Data/
│   │   │   ├── ApplicationDbContext.cs
│   │   │   ├── Configurations/   ← IEntityTypeConfiguration<T>
│   │   │   ├── Migrations/
│   │   │   └── Repositories/     ← implementations
│   │   ├── ExternalServices/
│   │   ├── Identity/
│   │   └── DependencyInjection.cs
│   └── CleanApp.Api/
│       ├── Controllers/
│       ├── Filters/
│       ├── Middleware/
│       └── Program.cs
├── tests/
│   ├── CleanApp.Domain.Tests/
│   ├── CleanApp.Application.Tests/
│   └── CleanApp.Api.Tests/
└── CleanApp.sln
```

---

## Step 1927: Domain Layer — Pure Business Logic

```csharp
// Domain/Entities/Customer.cs
namespace CleanApp.Domain.Entities;

public sealed class Customer : Entity<CustomerId>
{
    private readonly List<Address> _addresses = [];

    public CustomerName Name { get; private set; }
    public Email Email { get; private set; }
    public CustomerTier Tier { get; private set; }
    public IReadOnlyList<Address> Addresses => _addresses.AsReadOnly();

    private Customer() { }

    public static Customer Register(CustomerName name, Email email)
    {
        var customer = new Customer
        {
            Id = new CustomerId(Guid.NewGuid()),
            Name = name,
            Email = email,
            Tier = CustomerTier.Standard
        };

        customer.RaiseDomainEvent(new CustomerRegisteredEvent(customer.Id, email));
        return customer;
    }

    public void AddAddress(Address address)
    {
        if (_addresses.Count >= 5)
            throw new DomainException("Customer cannot have more than 5 addresses");

        if (_addresses.Any(a => a == address))
            throw new DomainException("Address already exists");

        _addresses.Add(address);
    }

    public void UpgradeTier(CustomerTier newTier)
    {
        if (newTier <= Tier)
            throw new DomainException($"Cannot downgrade tier from {Tier} to {newTier}");

        var previousTier = Tier;
        Tier = newTier;
        RaiseDomainEvent(new CustomerTierUpgradedEvent(Id, previousTier, newTier));
    }

    public bool CanReceiveDiscount(decimal orderAmount) =>
        Tier switch
        {
            CustomerTier.Gold => orderAmount >= 100,
            CustomerTier.Platinum => orderAmount >= 50,
            _ => false
        };
}

public enum CustomerTier { Standard = 1, Gold = 2, Platinum = 3 }
```

```csharp
// Domain/ValueObjects/Email.cs
public sealed class Email : ValueObject
{
    public string Value { get; }

    private Email(string value) => Value = value;

    public static Email Create(string value)
    {
        if (string.IsNullOrWhiteSpace(value))
            throw new DomainException("Email cannot be empty");

        if (!value.Contains('@') || !value.Contains('.'))
            throw new DomainException($"'{value}' is not a valid email");

        return new Email(value.ToLowerInvariant().Trim());
    }

    protected override IEnumerable<object?> GetEqualityComponents()
    {
        yield return Value;
    }

    public override string ToString() => Value;

    public static implicit operator string(Email email) => email.Value;
}
```

```csharp
// Domain/Repositories/ICustomerRepository.cs
namespace CleanApp.Domain.Repositories;

public interface ICustomerRepository
{
    Task<Customer?> GetByIdAsync(CustomerId id, CancellationToken ct = default);
    Task<Customer?> GetByEmailAsync(Email email, CancellationToken ct = default);
    Task<bool> ExistsAsync(Email email, CancellationToken ct = default);
    Task AddAsync(Customer customer, CancellationToken ct = default);
    Task<PagedResult<Customer>> ListAsync(
        CustomerFilter filter, CancellationToken ct = default);
}
```

---

## Step 1928: Application Layer — Use Cases

```csharp
// Application/Common/Interfaces/ICurrentUser.cs
namespace CleanApp.Application.Common.Interfaces;

public interface ICurrentUser
{
    string? UserId { get; }
    string? Email { get; }
    bool IsAuthenticated { get; }
    bool IsInRole(string role);
}

public interface IDateTime
{
    DateTimeOffset UtcNow { get; }
}

public interface IEmailService
{
    Task SendWelcomeEmailAsync(string to, string name, CancellationToken ct = default);
    Task SendPasswordResetAsync(string to, string resetLink, CancellationToken ct = default);
}
```

```csharp
// Application/Common/Models/Result.cs
namespace CleanApp.Application.Common.Models;

public class Result
{
    protected Result(bool succeeded, IEnumerable<string> errors)
    {
        Succeeded = succeeded;
        Errors = errors.ToArray();
    }

    public bool Succeeded { get; }
    public string[] Errors { get; }

    public static Result Success() => new(true, []);
    public static Result Failure(params string[] errors) => new(false, errors);
}

public class Result<T> : Result
{
    private Result(bool succeeded, T? value, IEnumerable<string> errors)
        : base(succeeded, errors)
    {
        Value = value;
    }

    public T? Value { get; }

    public static Result<T> Success(T value) => new(true, value, []);
    public static new Result<T> Failure(params string[] errors) => new(false, default, errors);
}
```

```csharp
// Application/Customers/Commands/RegisterCustomer/RegisterCustomerCommand.cs
namespace CleanApp.Application.Customers.Commands.RegisterCustomer;

public record RegisterCustomerCommand : IRequest<Result<Guid>>
{
    public required string FirstName { get; init; }
    public required string LastName { get; init; }
    public required string Email { get; init; }
}
```

```csharp
// Application/Customers/Commands/RegisterCustomer/RegisterCustomerCommandValidator.cs
using FluentValidation;

public class RegisterCustomerCommandValidator : AbstractValidator<RegisterCustomerCommand>
{
    private readonly ICustomerRepository _customers;

    public RegisterCustomerCommandValidator(ICustomerRepository customers)
    {
        _customers = customers;

        RuleFor(x => x.FirstName)
            .NotEmpty().MaximumLength(100);

        RuleFor(x => x.LastName)
            .NotEmpty().MaximumLength(100);

        RuleFor(x => x.Email)
            .NotEmpty()
            .EmailAddress()
            .MaximumLength(320)
            .MustAsync(BeUniqueEmail).WithMessage("Email already registered");
    }

    private async Task<bool> BeUniqueEmail(string email, CancellationToken ct)
    {
        try
        {
            var emailVo = Email.Create(email);
            return !await _customers.ExistsAsync(emailVo, ct);
        }
        catch
        {
            return false;
        }
    }
}
```

```csharp
// Application/Customers/Commands/RegisterCustomer/RegisterCustomerCommandHandler.cs
public class RegisterCustomerCommandHandler
    : IRequestHandler<RegisterCustomerCommand, Result<Guid>>
{
    private readonly ICustomerRepository _customers;
    private readonly IUnitOfWork _uow;
    private readonly IEmailService _emailService;

    public RegisterCustomerCommandHandler(
        ICustomerRepository customers,
        IUnitOfWork uow,
        IEmailService emailService)
    {
        _customers = customers;
        _uow = uow;
        _emailService = emailService;
    }

    public async Task<Result<Guid>> Handle(
        RegisterCustomerCommand cmd, CancellationToken ct)
    {
        var name = CustomerName.Create(cmd.FirstName, cmd.LastName);
        var email = Email.Create(cmd.Email);

        var customer = Customer.Register(name, email);
        await _customers.AddAsync(customer, ct);
        await _uow.CommitAsync(ct);

        // Send welcome email (fire-and-forget — don't fail registration if email fails)
        _ = _emailService.SendWelcomeEmailAsync(
            customer.Email.Value, customer.Name.FullName, CancellationToken.None);

        return Result<Guid>.Success(customer.Id.Value);
    }
}
```

---

## Step 1929: MediatR Pipeline Behaviors

```csharp
// Application/Common/Behaviors/ValidationBehavior.cs
using FluentValidation;
using MediatR;

namespace CleanApp.Application.Common.Behaviors;

public class ValidationBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;

    public ValidationBehavior(IEnumerable<IValidator<TRequest>> validators)
    {
        _validators = validators;
    }

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken ct)
    {
        if (!_validators.Any()) return await next();

        var context = new ValidationContext<TRequest>(request);
        var results = await Task.WhenAll(
            _validators.Select(v => v.ValidateAsync(context, ct)));

        var failures = results
            .SelectMany(r => r.Errors)
            .Where(f => f is not null)
            .GroupBy(f => f.PropertyName)
            .ToDictionary(
                g => g.Key,
                g => g.Select(f => f.ErrorMessage).ToArray());

        if (failures.Any())
            throw new ValidationException(failures);

        return await next();
    }
}
```

```csharp
// Application/Common/Behaviors/LoggingBehavior.cs
public class LoggingBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    private readonly ILogger<LoggingBehavior<TRequest, TResponse>> _logger;
    private readonly ICurrentUser _currentUser;

    public LoggingBehavior(
        ILogger<LoggingBehavior<TRequest, TResponse>> logger,
        ICurrentUser currentUser)
    {
        _logger = logger;
        _currentUser = currentUser;
    }

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken ct)
    {
        var requestName = typeof(TRequest).Name;
        var userId = _currentUser.UserId ?? "anonymous";

        _logger.LogInformation(
            "Handling {RequestName} for user {UserId}", requestName, userId);

        var sw = Stopwatch.StartNew();
        try
        {
            var response = await next();
            sw.Stop();

            if (sw.ElapsedMilliseconds > 500)
            {
                _logger.LogWarning(
                    "Slow request: {RequestName} took {Elapsed}ms",
                    requestName, sw.ElapsedMilliseconds);
            }

            return response;
        }
        catch (Exception ex)
        {
            sw.Stop();
            _logger.LogError(ex,
                "Request {RequestName} failed after {Elapsed}ms",
                requestName, sw.ElapsedMilliseconds);
            throw;
        }
    }
}
```

```csharp
// Application/Common/Behaviors/PerformanceBehavior.cs
public class PerformanceBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    private readonly ILogger<PerformanceBehavior<TRequest, TResponse>> _logger;

    public PerformanceBehavior(
        ILogger<PerformanceBehavior<TRequest, TResponse>> logger)
    {
        _logger = logger;
    }

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken ct)
    {
        using var activity = Activity.Current;
        var sw = Stopwatch.StartNew();

        var response = await next();

        sw.Stop();
        activity?.SetTag("request.duration_ms", sw.ElapsedMilliseconds);

        return response;
    }
}
```

---

## Step 1930: Infrastructure Layer — EF Core

```csharp
// Infrastructure/Data/ApplicationDbContext.cs
namespace CleanApp.Infrastructure.Data;

public class ApplicationDbContext : DbContext, IUnitOfWork
{
    private readonly ICurrentUser _currentUser;
    private readonly IDomainEventPublisher _eventPublisher;

    public ApplicationDbContext(
        DbContextOptions<ApplicationDbContext> options,
        ICurrentUser currentUser,
        IDomainEventPublisher eventPublisher)
        : base(options)
    {
        _currentUser = currentUser;
        _eventPublisher = eventPublisher;
    }

    public DbSet<Customer> Customers => Set<Customer>();
    public DbSet<Order> Orders => Set<Order>();

    protected override void OnModelCreating(ModelBuilder builder)
    {
        builder.ApplyConfigurationsFromAssembly(typeof(ApplicationDbContext).Assembly);
        base.OnModelCreating(builder);
    }

    public override async Task<int> SaveChangesAsync(CancellationToken ct = default)
    {
        // Audit fields
        foreach (var entry in ChangeTracker.Entries<AuditableEntity>())
        {
            switch (entry.State)
            {
                case EntityState.Added:
                    entry.Entity.CreatedAt = DateTimeOffset.UtcNow;
                    entry.Entity.CreatedBy = _currentUser.UserId;
                    break;
                case EntityState.Modified:
                    entry.Entity.UpdatedAt = DateTimeOffset.UtcNow;
                    entry.Entity.UpdatedBy = _currentUser.UserId;
                    break;
            }
        }

        var result = await base.SaveChangesAsync(ct);

        // Dispatch domain events after successful save
        await DispatchDomainEventsAsync(ct);

        return result;
    }

    private async Task DispatchDomainEventsAsync(CancellationToken ct)
    {
        var entitiesWithEvents = ChangeTracker.Entries<Entity>()
            .Where(e => e.Entity.DomainEvents.Any())
            .Select(e => e.Entity)
            .ToList();

        foreach (var entity in entitiesWithEvents)
        {
            var events = entity.DomainEvents.ToList();
            entity.ClearDomainEvents();

            foreach (var @event in events)
                await _eventPublisher.PublishAsync(@event, ct);
        }
    }

    async Task IUnitOfWork.CommitAsync(CancellationToken ct)
    {
        await SaveChangesAsync(ct);
    }
}
```

```csharp
// Infrastructure/Data/Configurations/CustomerConfiguration.cs
using Microsoft.EntityFrameworkCore.Metadata.Builders;

public class CustomerConfiguration : IEntityTypeConfiguration<Customer>
{
    public void Configure(EntityTypeBuilder<Customer> builder)
    {
        builder.HasKey(c => c.Id);
        builder.Property(c => c.Id)
            .HasConversion(id => id.Value, value => new CustomerId(value));

        builder.OwnsOne(c => c.Name, name =>
        {
            name.Property(n => n.FirstName).HasMaxLength(100).IsRequired();
            name.Property(n => n.LastName).HasMaxLength(100).IsRequired();
        });

        builder.OwnsOne(c => c.Email, email =>
        {
            email.Property(e => e.Value)
                .HasColumnName("Email")
                .HasMaxLength(320)
                .IsRequired();
            email.HasIndex(e => e.Value).IsUnique();
        });

        builder.Property(c => c.Tier)
            .HasConversion<string>()
            .HasMaxLength(20);

        builder.OwnsMany(c => c.Addresses, addr =>
        {
            addr.WithOwner().HasForeignKey("CustomerId");
            addr.HasKey("Id");
            addr.Property<Guid>("Id").ValueGeneratedOnAdd();
            addr.Property(a => a.Street).HasMaxLength(200).IsRequired();
            addr.Property(a => a.City).HasMaxLength(100).IsRequired();
            addr.Property(a => a.Country).HasMaxLength(2).IsRequired().IsFixedLength();
        });
    }
}
```

---

## Step 1931: Infrastructure — Repository Implementation

```csharp
// Infrastructure/Data/Repositories/CustomerRepository.cs
namespace CleanApp.Infrastructure.Data.Repositories;

public class CustomerRepository : ICustomerRepository
{
    private readonly ApplicationDbContext _db;

    public CustomerRepository(ApplicationDbContext db)
    {
        _db = db;
    }

    public async Task<Customer?> GetByIdAsync(CustomerId id, CancellationToken ct = default)
    {
        return await _db.Customers
            .Include(c => c.Addresses) // eager-load owned entities
            .FirstOrDefaultAsync(c => c.Id == id, ct);
    }

    public async Task<Customer?> GetByEmailAsync(Email email, CancellationToken ct = default)
    {
        return await _db.Customers
            .FirstOrDefaultAsync(c => c.Email.Value == email.Value, ct);
    }

    public async Task<bool> ExistsAsync(Email email, CancellationToken ct = default)
    {
        return await _db.Customers
            .AnyAsync(c => c.Email.Value == email.Value, ct);
    }

    public async Task AddAsync(Customer customer, CancellationToken ct = default)
    {
        await _db.Customers.AddAsync(customer, ct);
    }

    public async Task<PagedResult<Customer>> ListAsync(
        CustomerFilter filter, CancellationToken ct = default)
    {
        var query = _db.Customers.AsQueryable();

        if (!string.IsNullOrEmpty(filter.Search))
        {
            var search = filter.Search.ToLower();
            query = query.Where(c =>
                c.Email.Value.Contains(search) ||
                c.Name.FirstName.ToLower().Contains(search) ||
                c.Name.LastName.ToLower().Contains(search));
        }

        if (filter.Tier.HasValue)
            query = query.Where(c => c.Tier == filter.Tier.Value);

        var total = await query.CountAsync(ct);

        var items = await query
            .OrderBy(c => c.Name.LastName)
            .ThenBy(c => c.Name.FirstName)
            .Skip((filter.Page - 1) * filter.PageSize)
            .Take(filter.PageSize)
            .ToListAsync(ct);

        return new PagedResult<Customer>(items, total, filter.Page, filter.PageSize);
    }
}
```

---

## Step 1932: API Layer — Thin Controllers

```csharp
// Api/Controllers/CustomersController.cs
using MediatR;

[ApiController]
[Route("api/customers")]
[Authorize]
public class CustomersController : ControllerBase
{
    private readonly IMediator _mediator;

    public CustomersController(IMediator mediator) => _mediator = mediator;

    [HttpPost]
    [AllowAnonymous]
    [ProducesResponseType(typeof(Guid), 201)]
    [ProducesResponseType(typeof(ValidationProblemDetails), 400)]
    public async Task<ActionResult<Guid>> Register(
        RegisterCustomerRequest request, CancellationToken ct)
    {
        var command = new RegisterCustomerCommand
        {
            FirstName = request.FirstName,
            LastName = request.LastName,
            Email = request.Email
        };

        var result = await _mediator.Send(command, ct);

        if (!result.Succeeded)
            return BadRequest(new ProblemDetails { Title = string.Join("; ", result.Errors) });

        return CreatedAtAction(nameof(GetCustomer), new { id = result.Value }, result.Value);
    }

    [HttpGet("{id:guid}")]
    [ProducesResponseType(typeof(CustomerDto), 200)]
    [ProducesResponseType(404)]
    public async Task<ActionResult<CustomerDto>> GetCustomer(Guid id, CancellationToken ct)
    {
        var result = await _mediator.Send(new GetCustomerQuery(id), ct);
        return result is null ? NotFound() : Ok(result);
    }

    [HttpGet]
    public async Task<ActionResult<PagedResult<CustomerSummaryDto>>> ListCustomers(
        [FromQuery] ListCustomersQuery query, CancellationToken ct)
    {
        return Ok(await _mediator.Send(query, ct));
    }
}
```

---

## Step 1933: Architecture Enforcement with ArchUnitNET

```bash
dotnet add package ArchUnitNET
dotnet add package ArchUnitNET.xUnit
```

```csharp
// Tests/Architecture/ArchitectureTests.cs
using ArchUnitNET.Domain;
using ArchUnitNET.Loader;
using ArchUnitNET.xUnit;
using static ArchUnitNET.Fluent.ArchRuleDefinition;

namespace CleanApp.Tests.Architecture;

public class ArchitectureTests
{
    private static readonly Architecture App = new ArchLoader()
        .LoadAssemblies(
            typeof(Customer).Assembly,         // Domain
            typeof(RegisterCustomerCommand).Assembly,  // Application
            typeof(ApplicationDbContext).Assembly,     // Infrastructure
            typeof(CustomersController).Assembly)      // Api
        .Build();

    private readonly IObjectProvider<IType> DomainLayer =
        Types().That().ResideInAssembly(typeof(Customer).Assembly).As("Domain");

    private readonly IObjectProvider<IType> ApplicationLayer =
        Types().That().ResideInAssembly(typeof(RegisterCustomerCommand).Assembly).As("Application");

    private readonly IObjectProvider<IType> InfrastructureLayer =
        Types().That().ResideInAssembly(typeof(ApplicationDbContext).Assembly).As("Infrastructure");

    [Fact]
    public void Domain_ShouldNotDependOnApplication()
    {
        Classes().That().Are(DomainLayer)
            .Should().NotDependOnAny(ApplicationLayer)
            .Check(App);
    }

    [Fact]
    public void Domain_ShouldNotDependOnInfrastructure()
    {
        Classes().That().Are(DomainLayer)
            .Should().NotDependOnAny(InfrastructureLayer)
            .Check(App);
    }

    [Fact]
    public void Application_ShouldNotDependOnInfrastructure()
    {
        Classes().That().Are(ApplicationLayer)
            .Should().NotDependOnAny(InfrastructureLayer)
            .Check(App);
    }

    [Fact]
    public void Controllers_ShouldResideInApiNamespace()
    {
        Classes().That().HaveNameEndingWith("Controller")
            .Should().ResideInNamespace("CleanApp.Api")
            .Check(App);
    }

    [Fact]
    public void CommandHandlers_ShouldResideInApplicationLayer()
    {
        Classes().That().HaveNameEndingWith("CommandHandler")
            .Should().ResideInAssembly(typeof(RegisterCustomerCommand).Assembly)
            .Check(App);
    }

    [Fact]
    public void Repositories_ShouldNotBeUsedDirectlyInControllers()
    {
        Classes().That().HaveNameEndingWith("Controller")
            .Should().NotDependOnAny(
                Types().That().HaveNameEndingWith("Repository")
                    .And().ResideInAssembly(typeof(ApplicationDbContext).Assembly))
            .Check(App);
    }
}
```

---

## Step 1934: Vertical Slice Architecture

Instead of horizontal layers (Domain/Application/Infrastructure), organize by **feature** — everything for one feature stays together.

```
VerticalSliceApp/
├── Features/
│   ├── Orders/
│   │   ├── PlaceOrder.cs          ← Command + Handler + Validator + Response
│   │   ├── GetOrder.cs            ← Query + Handler + Response
│   │   ├── ListOrders.cs          ← Query + Handler + Response
│   │   ├── ConfirmOrder.cs
│   │   └── ShipOrder.cs
│   ├── Customers/
│   │   ├── RegisterCustomer.cs
│   │   ├── GetCustomer.cs
│   │   └── UpdateProfile.cs
│   └── Products/
│       ├── CreateProduct.cs
│       └── GetProduct.cs
├── Common/
│   ├── Behaviors/
│   ├── Exceptions/
│   └── Models/
├── Infrastructure/
│   ├── Data/
│   └── DependencyInjection.cs
└── Program.cs
```

Each slice is self-contained — a developer working on `PlaceOrder` only touches `PlaceOrder.cs`.

---

## Step 1935: Vertical Slice — Feature File

```csharp
// Features/Orders/PlaceOrder.cs
using Carter;
using FluentValidation;
using MediatR;

namespace VerticalSliceApp.Features.Orders;

// Everything for "Place Order" in one file

public static class PlaceOrder
{
    // Request DTO
    public record Request(
        Guid CustomerId,
        string Street,
        string City,
        string Country,
        IReadOnlyList<RequestItem> Items);

    public record RequestItem(Guid ProductId, string Sku, int Quantity, decimal UnitPrice);

    // Response DTO
    public record Response(Guid OrderId, string Status, decimal Total);

    // Command
    public record Command : IRequest<Response>
    {
        public required Guid CustomerId { get; init; }
        public required Address ShippingAddress { get; init; }
        public required IReadOnlyList<OrderItemInput> Items { get; init; }
    }

    // Validator
    public class Validator : AbstractValidator<Command>
    {
        public Validator()
        {
            RuleFor(x => x.CustomerId).NotEmpty();
            RuleFor(x => x.ShippingAddress).NotNull();
            RuleFor(x => x.Items).NotEmpty().WithMessage("Order must have at least one item");
            RuleForEach(x => x.Items).ChildRules(item =>
            {
                item.RuleFor(i => i.Quantity).GreaterThan(0);
                item.RuleFor(i => i.UnitPrice).GreaterThan(0);
            });
        }
    }

    // Handler — inline use case logic
    public class Handler : IRequestHandler<Command, Response>
    {
        private readonly AppDbContext _db;

        public Handler(AppDbContext db)
        {
            _db = db;
        }

        public async Task<Response> Handle(Command cmd, CancellationToken ct)
        {
            var order = Order.Create(cmd.CustomerId, cmd.ShippingAddress);

            foreach (var item in cmd.Items)
                order.AddItem(item.ProductId, item.Sku, item.Quantity, item.UnitPrice);

            order.Confirm();

            _db.Orders.Add(order);
            await _db.SaveChangesAsync(ct);

            return new Response(order.Id, order.Status.ToString(), order.TotalAmount);
        }
    }

    // Endpoint (Carter module)
    public class Endpoint : ICarterModule
    {
        public void AddRoutes(IEndpointRouteBuilder app)
        {
            app.MapPost("/api/orders", async (
                Request request,
                IMediator mediator,
                ICurrentUser currentUser,
                CancellationToken ct) =>
            {
                var cmd = new Command
                {
                    CustomerId = currentUser.GetUserId(),
                    ShippingAddress = new Address(request.Street, request.City, request.Country),
                    Items = request.Items.Select(i => new OrderItemInput(
                        i.ProductId, i.Sku, i.Quantity, i.UnitPrice)).ToList()
                };

                var result = await mediator.Send(cmd, ct);
                return Results.Created($"/api/orders/{result.OrderId}", result);
            })
            .RequireAuthorization()
            .WithName("PlaceOrder")
            .WithOpenApi()
            .Produces<Response>(201)
            .ProducesProblem(400)
            .ProducesProblem(401);
        }
    }
}
```

---

## Step 1936: Vertical Slice — Query Feature

```csharp
// Features/Orders/GetOrder.cs
namespace VerticalSliceApp.Features.Orders;

public static class GetOrder
{
    public record Query(Guid OrderId) : IRequest<Response?>;

    public record Response(
        Guid Id,
        string Status,
        decimal TotalAmount,
        IReadOnlyList<ItemDto> Items,
        DateTimeOffset CreatedAt);

    public record ItemDto(Guid ProductId, string Sku, int Quantity, decimal UnitPrice);

    public class Handler : IRequestHandler<Query, Response?>
    {
        private readonly AppDbContext _db;

        public Handler(AppDbContext db) => _db = db;

        public async Task<Response?> Handle(Query query, CancellationToken ct)
        {
            // Direct projection — no domain object loaded
            var order = await _db.Orders
                .Where(o => o.Id == query.OrderId)
                .Select(o => new Response(
                    o.Id,
                    o.Status.ToString(),
                    o.TotalAmount,
                    o.Items.Select(i => new ItemDto(
                        i.ProductId, i.Sku, i.Quantity, i.UnitPrice)).ToList(),
                    o.CreatedAt))
                .FirstOrDefaultAsync(ct);

            return order;
        }
    }

    public class Endpoint : ICarterModule
    {
        public void AddRoutes(IEndpointRouteBuilder app)
        {
            app.MapGet("/api/orders/{id:guid}", async (
                Guid id,
                IMediator mediator,
                CancellationToken ct) =>
            {
                var result = await mediator.Send(new Query(id), ct);
                return result is null ? Results.NotFound() : Results.Ok(result);
            })
            .RequireAuthorization()
            .WithName("GetOrder")
            .WithOpenApi()
            .Produces<Response>()
            .ProducesProblem(404);
        }
    }
}
```

---

## Step 1937: Minimal API with Carter

```bash
dotnet add package Carter
```

```csharp
// Program.cs — Vertical Slice + Carter
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddMediatR(cfg =>
    cfg.RegisterServicesFromAssembly(typeof(PlaceOrder.Command).Assembly)
       .AddBehavior(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>))
       .AddBehavior(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>)));

builder.Services.AddValidatorsFromAssembly(typeof(PlaceOrder.Validator).Assembly);

builder.Services.AddCarter();

builder.Services.AddDbContext<AppDbContext>(opts =>
    opts.UseNpgsql(builder.Configuration.GetConnectionString("Postgres")));

builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer();
builder.Services.AddAuthorization();
builder.Services.AddHttpContextAccessor();
builder.Services.AddScoped<ICurrentUser, HttpContextCurrentUser>();

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();
app.MapCarter(); // discovers all ICarterModule implementations

app.Run();
```

---

## Step 1938: Exception Handling Middleware

```csharp
// Common/Exceptions/ApplicationException.cs
namespace CleanApp.Application.Common.Exceptions;

public class NotFoundException : Exception
{
    public NotFoundException(string name, object key)
        : base($"Entity '{name}' ({key}) was not found.") { }
}

public class ValidationException : Exception
{
    public ValidationException(IDictionary<string, string[]> errors)
        : base("One or more validation failures occurred.")
    {
        Errors = errors;
    }

    public IDictionary<string, string[]> Errors { get; }
}

public class ForbiddenException : Exception
{
    public ForbiddenException() : base("You do not have permission to perform this action.") { }
}
```

```csharp
// Api/Middleware/ExceptionHandlerMiddleware.cs
public class ExceptionHandlerMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionHandlerMiddleware> _logger;

    public ExceptionHandlerMiddleware(RequestDelegate next,
        ILogger<ExceptionHandlerMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            await HandleExceptionAsync(context, ex);
        }
    }

    private async Task HandleExceptionAsync(HttpContext context, Exception exception)
    {
        var (statusCode, problem) = exception switch
        {
            ValidationException ve => (
                StatusCodes.Status400BadRequest,
                new ValidationProblemDetails(ve.Errors)
                {
                    Title = "Validation failed",
                    Status = StatusCodes.Status400BadRequest
                }),

            NotFoundException nfe => (
                StatusCodes.Status404NotFound,
                new ProblemDetails
                {
                    Title = "Resource not found",
                    Detail = nfe.Message,
                    Status = StatusCodes.Status404NotFound
                }),

            ForbiddenException => (
                StatusCodes.Status403Forbidden,
                new ProblemDetails
                {
                    Title = "Forbidden",
                    Status = StatusCodes.Status403Forbidden
                }),

            _ => (
                StatusCodes.Status500InternalServerError,
                new ProblemDetails
                {
                    Title = "An error occurred",
                    Status = StatusCodes.Status500InternalServerError
                })
        };

        if (statusCode == StatusCodes.Status500InternalServerError)
            _logger.LogError(exception, "Unhandled exception");

        context.Response.StatusCode = statusCode;
        context.Response.ContentType = "application/problem+json";
        await context.Response.WriteAsJsonAsync(problem);
    }
}
```

---

## Step 1939: Comparing the Two Approaches

```
Clean Architecture:
  Pros:
  + Clear separation of concerns (Domain, Application, Infrastructure)
  + Enforced dependency rules via project references
  + Domain model is pure — easy to unit test
  + Well understood, familiar to most teams

  Cons:
  - Adding a feature touches 3–4 projects/folders
  - Ceremony for simple CRUD use cases
  - Anemia risk if domain logic migrates to Application

Vertical Slice:
  Pros:
  + Feature is self-contained — one file per use case
  + Low cognitive load per slice
  + Easy to delete or refactor a feature
  + Scales well on large teams (minimal merge conflicts)

  Cons:
  - Code duplication across slices (acceptable — cohesion over DRY)
  - Harder to see global consistency
  - Domain model can get thin if shared logic isn't extracted

Hybrid approach (recommended for most projects):
  - Keep Domain and Infrastructure as layers (Clean Architecture)
  - Organize Application / Presentation by feature (Vertical Slice)
  - Each feature slice can access Domain objects directly
```

---

## Step 1940: Complete Registration

```csharp
// Infrastructure/DependencyInjection.cs
using CleanApp.Application.Common.Interfaces;

namespace CleanApp.Infrastructure;

public static class DependencyInjection
{
    public static IServiceCollection AddInfrastructure(
        this IServiceCollection services,
        IConfiguration config)
    {
        services.AddDbContext<ApplicationDbContext>(opts =>
            opts.UseNpgsql(config.GetConnectionString("Postgres"),
                b => b.MigrationsAssembly(typeof(ApplicationDbContext).Assembly.FullName)));

        // Repositories
        services.AddScoped<ICustomerRepository, CustomerRepository>();
        services.AddScoped<IOrderRepository, OrderRepository>();

        // Unit of Work
        services.AddScoped<IUnitOfWork>(sp =>
            sp.GetRequiredService<ApplicationDbContext>());

        // Domain Event Publisher
        services.AddScoped<IDomainEventPublisher, MediatRDomainEventPublisher>();

        // External services
        services.AddTransient<IEmailService, SendGridEmailService>();

        // Date/time
        services.AddSingleton<IDateTime, SystemDateTime>();

        return services;
    }
}
```

```csharp
// Application/DependencyInjection.cs
public static class DependencyInjection
{
    public static IServiceCollection AddApplication(this IServiceCollection services)
    {
        var assembly = typeof(RegisterCustomerCommand).Assembly;

        services.AddMediatR(cfg =>
        {
            cfg.RegisterServicesFromAssembly(assembly);
            cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
            cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
            cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(PerformanceBehavior<,>));
        });

        services.AddValidatorsFromAssembly(assembly);
        services.AddAutoMapper(assembly);

        return services;
    }
}
```

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddApplication()
    .AddInfrastructure(builder.Configuration);

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
builder.Services.AddHttpContextAccessor();
builder.Services.AddScoped<ICurrentUser, HttpContextCurrentUser>();

builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer();
builder.Services.AddAuthorization();
builder.Services.AddHealthChecks()
    .AddDbContextCheck<ApplicationDbContext>();

var app = builder.Build();

app.UseMiddleware<ExceptionHandlerMiddleware>();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
app.MapHealthChecks("/health");

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
    using var scope = app.Services.CreateScope();
    var db = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
    await db.Database.MigrateAsync();
}

app.Run();
```

---

## Summary

| Concept | Clean Architecture | Vertical Slice |
|---------|-------------------|----------------|
| Organization | By layer | By feature |
| Dependency flow | Inward (domain is center) | Shared abstractions + slices |
| Test target | Domain layer in isolation | Each slice independently |
| Ceremony | More (abstractions per layer) | Less (flat, one file) |
| Duplication | Avoided via shared services | Accepted within slices |
| Cross-cutting | MediatR pipeline behaviors | Same MediatR behaviors |
| Best for | DDD-heavy complex domains | CRUD-heavy, large teams |

**Next**: Part 82 — Performance Optimization & Memory Management
