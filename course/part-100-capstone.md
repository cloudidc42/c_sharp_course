# Part 100: Course Capstone - Complete Step Index และ Learning Path

## Steps 2229-2248

บทสรุปสุดท้ายของหลักสูตร: สร้าง Full Production System, ดูสรุปทุก Steps 1-2248, และ Learning Path สำหรับ World-Class .NET Developer

---

## Step 2229: Capstone Project - E-Commerce Platform

```
Full Production System
=======================
"ThaiMart" - E-Commerce Platform

สร้างขึ้นด้วยทุกสิ่งที่เรียนมาตลอดหลักสูตร:

Architecture:
├── .NET Aspire AppHost (orchestration)
├── API Gateway (YARP) 
├── Catalog Service (.NET 9 Minimal API + GraphQL)
├── Order Service (CQRS + Event Sourcing + DDD)
├── Payment Service (Saga Pattern)
├── Inventory Service (gRPC)
├── Notification Service (Background Worker)
├── BFF Mobile API
└── Web Frontend (Blazor/React)

Infrastructure:
├── PostgreSQL (orders, users)
├── Redis (caching, sessions)
├── RabbitMQ (messaging)
├── Elasticsearch (search)
├── Qdrant (vector search for AI)
└── Azure Blob Storage (files)

Operations:
├── Docker + Kubernetes (AKS)
├── Helm charts
├── GitHub Actions CI/CD
├── Prometheus + Grafana
├── Jaeger (tracing)
└── Azure App Configuration (feature flags)
```

---

## Step 2230: Capstone - Domain Model

```csharp
// Core domain entities across bounded contexts

// === Catalog Bounded Context ===
namespace ThaiMart.Catalog.Domain;

public class Product : AggregateRoot<ProductId>
{
    public string Name { get; private set; }
    public string Description { get; private set; }
    public Money Price { get; private set; }
    public ProductCategory Category { get; private set; }
    public List<ProductImage> Images { get; private set; } = [];
    public bool IsActive { get; private set; }
    public StockLevel Stock { get; private set; }

    private Product() { }

    public static Product Create(
        string name, string description, Money price, ProductCategory category)
    {
        Guard.Against.NullOrEmpty(name);
        Guard.Against.Negative(price.Amount);

        var product = new Product
        {
            Id = ProductId.New(),
            Name = name,
            Description = description,
            Price = price,
            Category = category,
            IsActive = true,
            Stock = StockLevel.Zero
        };

        product.RaiseDomainEvent(new ProductCreated(product.Id, product.Name, product.Price));
        return product;
    }

    public void UpdatePrice(Money newPrice)
    {
        if (newPrice.Currency != Price.Currency)
            throw new DomainException("Cannot change currency");

        var oldPrice = Price;
        Price = newPrice;
        RaiseDomainEvent(new ProductPriceChanged(Id, oldPrice, newPrice));
    }

    public void AddStock(int quantity)
    {
        Guard.Against.NegativeOrZero(quantity);
        Stock = Stock.Add(quantity);
        RaiseDomainEvent(new StockAdded(Id, quantity, Stock));
    }

    public void ReserveStock(int quantity)
    {
        if (!Stock.CanReserve(quantity))
            throw new InsufficientStockException(Id, quantity, Stock.Available);
        Stock = Stock.Reserve(quantity);
    }
}

// === Order Bounded Context ===
namespace ThaiMart.Orders.Domain;

public class Order : AggregateRoot<OrderId>
{
    public CustomerId CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    public Money Total { get; private set; }
    public Address ShippingAddress { get; private set; }
    public List<OrderLine> Lines { get; private set; } = [];
    public TrackingInfo? Tracking { get; private set; }
    public DateTimeOffset PlacedAt { get; private set; }

    private Order() { }

    public static Order Place(
        CustomerId customerId,
        IReadOnlyList<OrderLineRequest> lines,
        Address shippingAddress)
    {
        if (!lines.Any()) throw new DomainException("Order must have at least one item");

        var total = lines.Aggregate(
            Money.Zero("THB"),
            (sum, line) => sum + new Money(line.UnitPrice * line.Quantity, "THB"));

        var order = new Order
        {
            Id = OrderId.New(),
            CustomerId = customerId,
            Status = OrderStatus.Pending,
            ShippingAddress = shippingAddress,
            Lines = lines.Select(l => new OrderLine(
                new ProductId(l.ProductId), l.ProductName, l.Quantity,
                new Money(l.UnitPrice, "THB"))).ToList(),
            Total = total,
            PlacedAt = DateTimeOffset.UtcNow
        };

        order.RaiseDomainEvent(new OrderPlaced(
            order.Id, order.CustomerId, order.Total, order.Lines, order.PlacedAt));

        return order;
    }

    public void Confirm(PaymentId paymentId)
    {
        if (Status != OrderStatus.Pending)
            throw new DomainException($"Cannot confirm order with status {Status}");

        Status = OrderStatus.Confirmed;
        RaiseDomainEvent(new OrderConfirmed(Id, paymentId, DateTimeOffset.UtcNow));
    }

    public void Ship(TrackingInfo tracking)
    {
        if (Status != OrderStatus.Confirmed)
            throw new DomainException($"Cannot ship order with status {Status}");

        Status = OrderStatus.Shipped;
        Tracking = tracking;
        RaiseDomainEvent(new OrderShipped(Id, tracking, DateTimeOffset.UtcNow));
    }

    public void Cancel(CancellationReason reason)
    {
        if (Status is OrderStatus.Shipped or OrderStatus.Delivered)
            throw new DomainException("Cannot cancel shipped or delivered order");

        Status = OrderStatus.Cancelled;
        RaiseDomainEvent(new OrderCancelled(Id, CustomerId, Total, reason, DateTimeOffset.UtcNow));
    }
}
```

---

## Step 2231: Capstone - Infrastructure Setup

```csharp
// AppHost/Program.cs - Complete orchestration
var builder = DistributedApplication.CreateBuilder(args);

// Infrastructure
var postgres = builder.AddPostgres("postgres")
    .WithDataVolume("thaimart-postgres");
var orderDb = postgres.AddDatabase("orders");
var userDb = postgres.AddDatabase("users");
var catalogDb = postgres.AddDatabase("catalog");

var redis = builder.AddRedis("redis")
    .WithDataVolume("thaimart-redis");

var rabbitmq = builder.AddRabbitMQ("messaging")
    .WithManagementPlugin();

var elasticsearch = builder.AddElasticsearch("search");

// Services
var catalogService = builder.AddProject<Projects.ThaiMart_Catalog>("catalog")
    .WithReference(catalogDb)
    .WithReference(redis)
    .WithReference(elasticsearch)
    .WithHttpHealthCheck("/health");

var orderService = builder.AddProject<Projects.ThaiMart_Orders>("orders")
    .WithReference(orderDb)
    .WithReference(redis)
    .WithReference(rabbitmq)
    .WithReference(catalogService)
    .WithHttpHealthCheck("/health");

var paymentService = builder.AddProject<Projects.ThaiMart_Payment>("payment")
    .WithReference(rabbitmq)
    .WithHttpHealthCheck("/health");

var inventoryService = builder.AddProject<Projects.ThaiMart_Inventory>("inventory")
    .WithReference(postgres.AddDatabase("inventory"))
    .WithReference(rabbitmq)
    .WithHttpHealthCheck("/health");

var notificationService = builder.AddProject<Projects.ThaiMart_Notifications>("notifications")
    .WithReference(rabbitmq);

var apiGateway = builder.AddProject<Projects.ThaiMart_Gateway>("gateway")
    .WithReference(catalogService)
    .WithReference(orderService)
    .WithReference(paymentService)
    .WithExternalHttpEndpoints();

var webApp = builder.AddProject<Projects.ThaiMart_Web>("web")
    .WithReference(apiGateway)
    .WithExternalHttpEndpoints();

builder.Build().Run();
```

---

## Step 2232: Capstone - Order Service Complete

```csharp
// Complete Order Service showing all patterns together

// Commands
public record PlaceOrderCommand(
    Guid CustomerId,
    List<OrderLineRequest> Lines,
    AddressDto ShippingAddress) : ICommand<PlaceOrderResult>;

public class PlaceOrderCommandHandler(
    IOrderRepository orderRepo,
    ICatalogClient catalogClient,
    IUnitOfWork uow,
    ILogger<PlaceOrderCommandHandler> logger)
    : ICommandHandler<PlaceOrderCommand, PlaceOrderResult>
{
    public async Task<PlaceOrderResult> Handle(PlaceOrderCommand cmd, CancellationToken ct)
    {
        // Enrich with catalog data
        var productIds = cmd.Lines.Select(l => l.ProductId).ToList();
        var products = await catalogClient.GetProductsByIdsAsync(productIds, ct);

        var lines = cmd.Lines.Select(l =>
        {
            var product = products.FirstOrDefault(p => p.Id == l.ProductId)
                ?? throw new NotFoundException($"Product {l.ProductId} not found");
            return new OrderLineRequest(l.ProductId, product.Name, l.Quantity, product.Price);
        }).ToList();

        var address = new Address(
            cmd.ShippingAddress.Street, cmd.ShippingAddress.City,
            cmd.ShippingAddress.PostalCode, cmd.ShippingAddress.Country);

        var order = Order.Place(new CustomerId(cmd.CustomerId), lines, address);
        await orderRepo.AddAsync(order, ct);
        await uow.SaveChangesAsync(ct);

        logger.LogInformation("Order {OrderId} placed: {Total} THB", order.Id, order.Total.Amount);
        return new PlaceOrderResult(order.Id.Value, order.Total.Amount);
    }
}

// MassTransit Consumers for saga
public class PaymentProcessedConsumer(
    IOrderRepository repo, IUnitOfWork uow)
    : IConsumer<PaymentProcessed>
{
    public async Task Consume(ConsumeContext<PaymentProcessed> context)
    {
        var order = await repo.GetByIdAsync(context.Message.OrderId)
            ?? throw new NotFoundException($"Order {context.Message.OrderId} not found");

        order.Confirm(new PaymentId(context.Message.PaymentId));
        await uow.SaveChangesAsync(context.CancellationToken);
    }
}

// Endpoints
public static class OrderEndpoints
{
    public static void MapOrderEndpoints(this IEndpointRouteBuilder app)
    {
        var orders = app.MapGroup("/api/v1/orders")
            .WithTags("Orders")
            .RequireAuthorization();

        orders.MapPost("/", PlaceOrder)
            .WithName("PlaceOrder")
            .WithValidation<PlaceOrderRequest>()
            .Produces<PlaceOrderResult>(201)
            .Produces<ValidationProblemDetails>(400)
            .RequireRateLimiting("PlaceOrder");

        orders.MapGet("/{id:guid}", GetOrder)
            .WithName("GetOrder")
            .Produces<OrderDto>()
            .Produces(404)
            .CacheOutput(p => p.SetVaryByRouteValue("id").Expire(TimeSpan.FromSeconds(30)));

        orders.MapGet("/", GetMyOrders)
            .WithName("GetMyOrders")
            .Produces<PagedResult<OrderSummaryDto>>();

        orders.MapDelete("/{id:guid}", CancelOrder)
            .WithName("CancelOrder")
            .Produces<NoContent>()
            .Produces(400)
            .Produces(404);
    }

    private static async Task<Results<Created<PlaceOrderResult>, ValidationProblem>> PlaceOrder(
        PlaceOrderRequest request,
        IMediator mediator,
        ICurrentUserService user,
        CancellationToken ct)
    {
        var command = new PlaceOrderCommand(user.UserId, request.Lines, request.ShippingAddress);
        var result = await mediator.Send(command, ct);
        return TypedResults.Created($"/api/v1/orders/{result.OrderId}", result);
    }
}
```

---

## Step 2233: Complete Step Index - Parts 1-50

```
COURSE COMPLETE STEP INDEX
============================

PART 1: Introduction to C# and .NET (Steps 1-10)
  1. What is C# and .NET
  2. Setting up development environment
  3. First C# program - Hello World
  4. Variables and data types
  5. String interpolation and formatting
  6. Console I/O
  7. Comments and documentation
  8. .NET SDK and CLI basics
  9. Visual Studio Code setup
  10. Understanding compilation and IL

PART 2: Variables and Data Types (Steps 11-20)
  11. Value types vs reference types
  12. int, long, float, double, decimal
  13. bool, char, string
  14. Type inference with var
  15. Nullable types
  16. Null coalescing operators (?? and ??=)
  17. Constants and readonly
  18. Enumerations
  19. Implicit and explicit conversion
  20. Type checking with is and as

PART 3: Operators and Expressions (Steps 21-30)
  21-30. Arithmetic, comparison, logical, bitwise operators

PART 4: Control Flow (Steps 31-40)
  31-40. if/else, switch expressions, loops (for/while/foreach)

PART 5: Methods and Functions (Steps 41-50)
  41-50. Method declaration, parameters, return values, overloading

PART 6: Arrays and Collections (Steps 51-60)
  51-60. Arrays, List<T>, Dictionary, Queue, Stack, HashSet

PART 7: Object-Oriented Programming (Steps 61-70)
  61-70. Classes, objects, constructors, properties, encapsulation

PART 8: Inheritance and Polymorphism (Steps 71-80)
  71-80. Base classes, virtual/override, abstract, interfaces

PART 9: Interfaces and Abstract Classes (Steps 81-90)
  81-90. Interface design, multiple interfaces, abstract patterns

PART 10: Exception Handling (Steps 91-100)
  91-100. try/catch/finally, custom exceptions, best practices

PART 11: LINQ Fundamentals (Steps 101-110)
  101-110. Query syntax, method syntax, filtering, projection

PART 12: LINQ Advanced (Steps 121-130)
  121-130. Grouping, joining, aggregation, complex queries

PART 13-15: Generic Programming (Steps 131-150)
  131-150. Generic classes, methods, constraints, collections

PART 16-18: Async Programming (Steps 151-180)
  151-180. Task, async/await, ConfigureAwait, cancellation

PART 19-20: File I/O (Steps 181-200)
  181-200. File/Directory, StreamReader/Writer, JSON, XML

PART 21-25: Data Access (Steps 201-250)
  201-250. ADO.NET, Dapper, transactions, stored procedures

PART 26-30: Entity Framework Core (Steps 251-300)
  251-300. DbContext, entities, migrations, LINQ queries

PART 31-35: Web APIs (Steps 301-350)
  301-350. ASP.NET Core, controllers, routing, middleware

PART 36-40: Dependency Injection (Steps 351-400)
  351-400. DI container, lifetimes, service registration, patterns

PART 41-45: Design Patterns (Steps 401-450)
  401-450. Creational, structural, behavioral patterns

PART 46-50: Testing (Steps 451-500)
  451-500. xUnit, FluentAssertions, mocking, unit tests
```

---

## Step 2234: Complete Step Index - Parts 51-100

```
PART 51-55: Authentication & Authorization (Steps 501-550)
  501-550. JWT, Identity, roles, policies, claims

PART 56-60: Caching (Steps 551-600)
  551-600. IMemoryCache, IDistributedCache, Redis, patterns

PART 61-65: Message Queuing (Steps 601-650)
  601-650. RabbitMQ, MassTransit, consumers, exchanges

PART 66-70: SignalR Real-Time (Steps 651-700)
  651-700. Hubs, clients, groups, streams, authentication

PART 71-75: Blazor (Steps 701-750)
  701-750. Components, data binding, routing, forms, JS interop

PART 76-80: Background Services (Steps 751-800)
  751-800. IHostedService, BackgroundService, Hangfire, Quartz.NET

PART 81-85: Performance Optimization (Steps 801-900)
  801-900. Span<T>, Memory<T>, ArrayPool, profiling, benchmarking

PART 86: Advanced EF Core (Steps 2005-2020)
  2005. Slow query interceptor
  2006. Auditing interceptor
  2007. Domain event publishing interceptor
  2008. Tenant schema interceptor
  2009. Global query filters
  2010. JSON columns
  2011. EFCore.BulkExtensions
  2012. Dapper for reporting
  2013. Optimistic concurrency
  2014. Compiled queries
  2015. Connection resiliency
  2016. Table splitting
  2017. Owned entities
  2018. Shadow properties
  2019. Raw SQL interpolation
  2020. Migration bundles for CI/CD

PART 87: Advanced Authentication (Steps 2021-2036)
  2021. PKCE flow
  2022. OpenIddict setup
  2023. Authorization code flow
  2024. Client credentials (M2M)
  2025. Refresh token rotation
  2026. TOTP MFA
  2027. Resource-based authorization
  2028. JWT hardening
  2029. API key authentication
  2030. Social login federation
  2031. Token exchange
  2032. Device flow
  2033. Authorization policies
  2034. Claim transformation
  2035. Audit logging
  2036. Security event monitoring

PART 88: Advanced Caching (Steps 2037-2052)
  2037. HybridCache (.NET 9 L1+L2)
  2038. IMemoryCache with size limits
  2039. IDistributedCache patterns
  2040. Cache-aside pattern
  2041. Write-through pattern
  2042. Refresh-ahead pattern
  2043. Output Caching
  2044. ETags with SHA256
  2045. Stampede protection
  2046. Redis pub/sub invalidation
  2047. Polly circuit breaker for Redis
  2048. Cache warming
  2049. Cache compression
  2050. Distributed locking
  2051. Cache tags
  2052. Cache monitoring

PART 89: Message-Driven Architecture (Steps 2053-2068)
  2053. Transactional Outbox pattern
  2054. Inbox idempotency pattern
  2055. SaveChanges interceptor
  2056. MassTransit Saga
  2057. Compensation transactions
  2058. Dead letter queues
  2059. Request-reply pattern
  2060. Message scheduling
  2061. Azure Service Bus
  2062. Message versioning
  2063. Event-driven CQRS projections
  2064. Saga timeout handling
  2065. Message retry policies
  2066. Batch processing
  2067. Message routing
  2068. Monitoring message flows

PART 90: Advanced Concurrency (Steps 2069-2084)
  2069. System.Threading.Channels
  2070. Bounded channels with backpressure
  2071. Multi-stage pipelines
  2072. Fan-out patterns
  2073. TPL Dataflow
  2074. Microsoft Orleans actors
  2075. Grain state and reminders
  2076. Orleans streams
  2077. SemaphoreSlim per-key locking
  2078. Parallel.ForEachAsync
  2079. AsyncLocal<T>
  2080. CancellationToken patterns
  2081. Rx.NET
  2082. Lock-free Interlocked
  2083. ValueTask optimization
  2084. Concurrent collections

PART 91: gRPC Advanced (Steps 2085-2100)
  2085. All 4 service types
  2086. Unary calls
  2087. Server streaming
  2088. Client streaming
  2089. Bidirectional streaming
  2090. Server interceptors
  2091. Client interceptors
  2092. Code-first with protobuf-net.Grpc
  2093. gRPC-Web
  2094. HTTP/JSON Transcoding
  2095. Deadlines and cancellation
  2096. Retry policy
  2097. mTLS
  2098. Compression
  2099. Service discovery
  2100. Performance optimization

PART 92: GraphQL (Steps 2101-2116)
  2101. Hot Chocolate setup
  2102. Code-first schema
  2103. EF Core integration
  2104. Cursor pagination
  2105. Filtering and sorting
  2106. Projections (N+1 prevention)
  2107. BatchDataLoader
  2108. GroupedDataLoader
  2109. Subscriptions with Redis
  2110. Union result types
  2111. Persisted queries
  2112. Query complexity limits
  2113. Authentication/Authorization
  2114. Error handling
  2115. Schema introspection
  2116. Performance tuning

PART 93: Testing Strategies (Steps 2117-2132)
  2117. Testing pyramid
  2118. xUnit, FluentAssertions, NSubstitute
  2119. AutoFixture
  2120. Bogus domain fakes
  2121. WebApplicationFactory
  2122. Testcontainers.PostgreSql
  2123. Respawner DB reset
  2124. Property-based testing (FsCheck)
  2125. Playwright E2E
  2126. Snapshot testing (Verify.Xunit)
  2127. Contract testing (PactNet)
  2128. BenchmarkDotNet
  2129. NBomber load testing
  2130. Stryker.NET mutation testing
  2131. Test data builders
  2132. Flaky test elimination

PART 94: Domain-Driven Design (Steps 2133-2148)
  2133. DDD overview
  2134. Value Objects (Money, Address, Email)
  2135. Strongly-typed IDs
  2136. Entity<TId> base class
  2137. AggregateRoot<TId> base class
  2138. Order aggregate with factory method
  2139. Domain Services
  2140. Repository pattern
  2141. Unit of Work
  2142. Domain events vs Integration events
  2143. Bounded Contexts
  2144. Anti-Corruption Layer
  2145. Specification pattern
  2146. Domain exceptions + Guard
  2147. EF Core configuration for DDD
  2148. DomainEventDispatchingInterceptor

PART 95: .NET Aspire (Steps 2149-2164)
  2149. Aspire overview
  2150. AppHost project setup
  2151. Service orchestration
  2152. Service Defaults
  2153. Service discovery
  2154. Database/Redis/RabbitMQ components
  2155. Custom metrics and tracing
  2156. Aspire Dashboard
  2157. Testing Aspire applications
  2158. Custom resources
  2159. Manifest and deployment
  2160. Azure integrations
  2161. Health checks
  2162. Resilience pipelines
  2163. Configuration management
  2164. Full solution structure

PART 96: AI Integration (Steps 2165-2180)
  2165. Semantic Kernel overview
  2166. SK setup and configuration
  2167. Native function plugins
  2168. Prompt functions
  2169. Chat completion + history
  2170. Semantic memory + RAG
  2171. Auto function calling
  2172. ML.NET sentiment analysis
  2173. ML.NET recommendations
  2174. Structured output
  2175. Embedding-based search
  2176. AI-powered order processing
  2177. Content moderation
  2178. Multi-step planning
  2179. Streaming responses
  2180. AI observability + cost management

PART 97: CI/CD & DevOps (Steps 2181-2196)
  2181. CI/CD strategy
  2182. GitHub Actions CI pipeline
  2183. Security scanning
  2184. Optimized Dockerfile
  2185. CD pipeline
  2186. Kubernetes manifests
  2187. Helm charts
  2188. Terraform IaC
  2189. Semantic versioning
  2190. Prometheus alerting
  2191. Database migrations in CI/CD
  2192. Feature flags
  2193. Blue/Green deployment
  2194. Observability stack
  2195. Environment configuration
  2196. Full CI/CD summary

PART 98: Advanced Minimal APIs (Steps 2197-2212)
  2197. Architecture patterns
  2198. Route groups + endpoint modules
  2199. Endpoint filters
  2200. TypedResults + union types
  2201. Parameter binding
  2202. OpenAPI advanced
  2203. API versioning
  2204. Rate limiting
  2205. Output caching
  2206. Problem Details
  2207. File upload/download
  2208. Streaming endpoints
  2209. Middleware as filters
  2210. Complete request pipeline
  2211. Testing minimal APIs
  2212. Best practices

PART 99: Architecture Patterns (Steps 2213-2228)
  2213. Production architecture overview
  2214. Clean Architecture
  2215. CQRS with MediatR
  2216. Event Sourcing
  2217. Read model projections
  2218. Saga patterns
  2219. API Gateway / BFF
  2220. Strangler Fig migration
  2221. Resilience patterns
  2222. Performance optimization
  2223. Security best practices
  2224. Multi-tenancy
  2225. SRE practices
  2226. Load testing
  2227. Architecture Decision Records
  2228. Production readiness checklist
```

---

## Step 2235: Learning Path Recommendations

```
World-Class .NET Developer Learning Path
==========================================

BEGINNER (Parts 1-30)
Duration: 3-6 months
Focus:
  ✓ C# syntax and fundamentals
  ✓ OOP principles
  ✓ LINQ and collections
  ✓ Async/await basics
  ✓ Basic EF Core
  ✓ Simple REST APIs

Goal: Build simple CRUD applications with .NET

INTERMEDIATE (Parts 31-60)
Duration: 6-12 months
Focus:
  ✓ Design patterns
  ✓ Unit testing (xUnit + NSubstitute)
  ✓ Dependency injection mastery
  ✓ Authentication (JWT)
  ✓ Redis caching
  ✓ RabbitMQ basics
  ✓ Docker containerization

Goal: Build production-ready APIs with testing

ADVANCED (Parts 61-80)
Duration: 6-12 months
Focus:
  ✓ CQRS + MediatR
  ✓ Domain-Driven Design
  ✓ Event-driven architecture
  ✓ Performance optimization
  ✓ Integration testing
  ✓ CI/CD pipelines
  ✓ Kubernetes basics

Goal: Build distributed systems with microservices

WORLD-CLASS (Parts 81-100)
Duration: 12+ months
Focus:
  ✓ Advanced concurrency patterns
  ✓ gRPC + GraphQL
  ✓ Event Sourcing
  ✓ .NET Aspire
  ✓ AI integration (Semantic Kernel)
  ✓ Advanced security (OAuth2/PKCE)
  ✓ SRE + production operations
  ✓ Architecture decisions

Goal: Lead architecture of large-scale systems
```

---

## Step 2236: Key Libraries Reference

```
Essential .NET Libraries
=========================

DATA ACCESS
├── Microsoft.EntityFrameworkCore          # ORM
├── EFCore.BulkExtensions                  # Bulk operations
├── Dapper                                  # Micro ORM for queries
├── Npgsql.EntityFrameworkCore.PostgreSQL   # PostgreSQL driver
└── Microsoft.Data.SqlClient               # SQL Server

MESSAGING
├── MassTransit                            # Message bus abstraction
├── MassTransit.RabbitMQ                   # RabbitMQ transport
├── MassTransit.Azure.ServiceBus.Core      # Azure Service Bus
└── MassTransit.Redis                      # Redis transport

VALIDATION & MAPPING
├── FluentValidation                       # Validation rules
├── Mapster                                # Fast object mapping
└── AutoMapper                             # Object mapping

RESILIENCE
├── Polly                                  # Resilience pipelines
└── Microsoft.Extensions.Http.Resilience   # HTTP resilience

OBSERVABILITY
├── OpenTelemetry.Sdk                      # Core OTel
├── OpenTelemetry.Exporter.Otlp            # OTLP exporter
├── OpenTelemetry.Instrumentation.AspNetCore
└── OpenTelemetry.Instrumentation.Http

AUTHENTICATION
├── Microsoft.AspNetCore.Authentication.JwtBearer  # JWT
├── OpenIddict.AspNetCore                   # Identity server
└── Microsoft.Identity.Web                  # Azure AD / B2C

CACHING
├── Microsoft.Extensions.Caching.Memory    # In-memory cache
├── Microsoft.Extensions.Caching.StackExchangeRedis  # Redis
└── Microsoft.AspNetCore.OutputCaching     # Response caching

API
├── Asp.Versioning.Http                    # API versioning
├── Microsoft.AspNetCore.OpenApi           # OpenAPI (.NET 9)
└── Swashbuckle.AspNetCore                # Swagger (.NET 8-)

GRAPHQL
└── HotChocolate.AspNetCore               # GraphQL server

GRPC
├── Grpc.AspNetCore                        # gRPC server
└── protobuf-net.Grpc                      # Code-first gRPC

TESTING
├── xunit                                  # Test framework
├── FluentAssertions                       # Assertion DSL
├── NSubstitute                            # Mocking
├── Bogus                                  # Fake data
├── Testcontainers                         # Docker containers
├── Verify.Xunit                           # Snapshot testing
├── PactNet                                # Contract testing
├── BenchmarkDotNet                        # Benchmarking
├── NBomber                                # Load testing
└── Stryker.NET                            # Mutation testing

AI/ML
├── Microsoft.SemanticKernel              # LLM orchestration
├── Microsoft.ML                           # ML.NET
└── OpenAI                                # OpenAI SDK

ORCHESTRATION
└── Aspire.Hosting.AppHost                # .NET Aspire

BACKGROUND JOBS
├── Hangfire                               # Background jobs
└── Quartz.NET                             # Job scheduling
```

---

## Step 2237: Common Patterns Quick Reference

```csharp
// =========================================
// PATTERN QUICK REFERENCE
// =========================================

// 1. Repository Pattern
public interface IRepository<T, TId> where T : AggregateRoot<TId>
{
    Task<T?> GetByIdAsync(TId id, CancellationToken ct = default);
    Task AddAsync(T entity, CancellationToken ct = default);
    Task<bool> ExistsAsync(TId id, CancellationToken ct = default);
}

// 2. CQRS Command
public record CreateItemCommand(string Name, decimal Price) : ICommand<Guid>;
public class CreateItemHandler(IRepository<Item, Guid> repo, IUnitOfWork uow)
    : ICommandHandler<CreateItemCommand, Guid>
{
    public async Task<Guid> Handle(CreateItemCommand cmd, CancellationToken ct)
    {
        var item = Item.Create(cmd.Name, cmd.Price);
        await repo.AddAsync(item, ct);
        await uow.SaveChangesAsync(ct);
        return item.Id;
    }
}

// 3. Specification
public class ActiveProductsSpec : Specification<Product>
{
    public override Expression<Func<Product, bool>> ToExpression()
        => p => p.IsActive && p.Stock.Available > 0;
}

// 4. Value Object
public sealed record Money(decimal Amount, string Currency)
{
    public Money Add(Money other)
    {
        if (Currency != other.Currency) throw new InvalidOperationException("Currency mismatch");
        return this with { Amount = Amount + other.Amount };
    }
    public static Money Zero(string currency) => new(0, currency);
    public static Money operator +(Money a, Money b) => a.Add(b);
}

// 5. Result Pattern
public class Result<T>
{
    public bool IsSuccess { get; }
    public T? Value { get; }
    public string? Error { get; }

    private Result(T value) { IsSuccess = true; Value = value; }
    private Result(string error) { IsSuccess = false; Error = error; }

    public static Result<T> Success(T value) => new(value);
    public static Result<T> Failure(string error) => new(error);
}

// 6. Guard Clauses
public static class Guard
{
    public static T Against<T>(this T value, string? message = null)
        where T : struct
    {
        if (EqualityComparer<T>.Default.Equals(value, default))
            throw new ArgumentException(message ?? $"Value cannot be default");
        return value;
    }

    public static string AgainstNullOrEmpty(string value, string paramName = "")
    {
        if (string.IsNullOrEmpty(value))
            throw new ArgumentException($"{paramName} cannot be null or empty", paramName);
        return value;
    }
}

// 7. Decorator Pattern for cross-cutting
public class CachedProductRepository(
    IProductRepository inner, IDistributedCache cache)
    : IProductRepository
{
    public async Task<Product?> GetByIdAsync(Guid id, CancellationToken ct)
    {
        var cacheKey = $"product:{id}";
        var cached = await cache.GetStringAsync(cacheKey, ct);
        if (cached is not null) return JsonSerializer.Deserialize<Product>(cached);

        var product = await inner.GetByIdAsync(id, ct);
        if (product is not null)
            await cache.SetStringAsync(cacheKey, JsonSerializer.Serialize(product),
                new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10) }, ct);

        return product;
    }
}
```

---

## Step 2238-2248: Final Words and Summary

```
COURSE COMPLETION SUMMARY
==========================

ยินดีด้วย! คุณได้เรียนรู้ทักษะ C# และ .NET ครบ 100% แล้ว!

จาก Steps 1-2248 คุณได้เรียนรู้:

FOUNDATION (Steps 1-500)
  ✓ C# syntax, OOP, LINQ, async/await
  ✓ Entity Framework Core
  ✓ REST APIs with ASP.NET Core
  ✓ Design patterns
  ✓ Unit testing

INTERMEDIATE (Steps 501-1500)
  ✓ Authentication & Authorization (JWT, OAuth2)
  ✓ Caching strategies (Redis, HybridCache)
  ✓ Message queuing (RabbitMQ, MassTransit)
  ✓ Real-time with SignalR
  ✓ Blazor frontend
  ✓ Background services
  ✓ Performance optimization

ADVANCED (Steps 1501-2004)
  ✓ Advanced EF Core patterns
  ✓ Microservices patterns
  ✓ Event-driven architecture
  ✓ Security hardening

WORLD-CLASS (Steps 2005-2248)
  ✓ gRPC and GraphQL
  ✓ Domain-Driven Design
  ✓ CQRS + Event Sourcing
  ✓ .NET Aspire
  ✓ AI Integration (Semantic Kernel)
  ✓ Production CI/CD
  ✓ Kubernetes deployment
  ✓ Observability + SRE

THE WORLD-CLASS DEVELOPER MINDSET:
=====================================

1. CODE QUALITY
   - Clean, readable, maintainable code
   - SOLID principles in practice
   - Test-driven when it adds value
   - Document the WHY, not the WHAT

2. SYSTEM THINKING
   - Design for failure (resilience)
   - Design for scale (horizontal scaling)
   - Design for observability (metrics, traces, logs)
   - Design for security (defense in depth)

3. PRAGMATISM
   - Use the right tool for the job
   - Simple solutions over complex ones
   - Measure before optimizing
   - Ship working software, then improve

4. CONTINUOUS LEARNING
   - .NET evolves with every release
   - Follow .NET Blog and GitHub
   - Contribute to open source
   - Share knowledge with community

NEXT STEPS AFTER THIS COURSE:
================================
1. Build your own production project
2. Contribute to OSS (.NET ecosystem)
3. Get Microsoft certifications (AZ-204, AZ-400)
4. Study distributed systems fundamentals
5. Learn cloud architecture (Azure/AWS/GCP)
6. Join .NET community (dotnet.microsoft.com/community)

IMPORTANT RESOURCES:
====================
- docs.microsoft.com/dotnet     # Official docs
- github.com/dotnet             # .NET source
- github.com/jasontaylordev     # Clean Architecture template
- github.com/ardalis             # Specifications, Guard Clauses
- github.com/ChilliCream         # Hot Chocolate GraphQL
- github.com/MassTransit         # MassTransit messaging
- github.com/microsoft/semantic-kernel  # AI integration

ขอให้โชคดีในการเป็น World-Class .NET Developer!
The journey continues - keep building, keep learning! 🚀
```

---

## Final Architecture Diagram

```
COMPLETE WORLD-CLASS .NET SYSTEM
==================================

                    ┌─────────────┐
                    │  CDN/WAF    │
                    │ (CloudFlare)│
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │ Load Balancer│
                    │   (nginx)    │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
         ┌────▼────┐  ┌────▼────┐  ┌───▼─────┐
         │  BFF    │  │  BFF    │  │ GraphQL │
         │  Web    │  │ Mobile  │  │  API    │
         └────┬────┘  └────┬────┘  └───┬─────┘
              └────────────┼────────────┘
                           │
              ┌────────────▼────────────┐
              │     API Gateway         │
              │   (YARP + Auth)         │
              └────────────┬────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
   ┌────▼─────┐      ┌─────▼────┐      ┌─────▼────┐
   │ Catalog  │      │  Order   │      │  User    │
   │ Service  │      │ Service  │      │ Service  │
   │ (Aspire) │      │ (CQRS+ES)│      │(Identity)│
   └────┬─────┘      └─────┬────┘      └─────┬────┘
        │                  │                  │
   ┌────▼─────┐      ┌─────▼────┐      ┌─────▼────┐
   │PostgreSQL│      │EventStore│      │PostgreSQL│
   │  Redis   │      │ RabbitMQ │      │  Redis   │
   │  Search  │      │  Outbox  │      └──────────┘
   └──────────┘      └──────────┘
                           │
              ┌────────────▼────────────┐
              │      Message Bus        │
              │     (RabbitMQ/ASB)      │
              └────────────┬────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
   ┌────▼─────┐      ┌─────▼────┐      ┌─────▼────┐
   │Inventory │      │ Payment  │      │  Notif.  │
   │ Service  │      │ Service  │      │ Service  │
   │  (gRPC)  │      │  (Saga)  │      │(Worker)  │
   └──────────┘      └──────────┘      └──────────┘

OBSERVABILITY:
   All services → OpenTelemetry Collector → Jaeger + Prometheus + Loki
   Grafana dashboards + Alertmanager → PagerDuty/Slack

DEPLOYMENT:
   GitHub Actions → Docker Registry → Kubernetes (AKS)
   Helm Charts + Terraform IaC + .NET Aspire manifest

SECURITY:
   Azure Key Vault → Secrets
   HTTPS everywhere + mTLS for service-to-service
   JWT with RS256 + PKCE for frontend auth
   Rate limiting + WAF at edge
```
