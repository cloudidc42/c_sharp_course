# Part 41: Native AOT & Ahead-of-Time Compilation

## Steps 1181-1210 | ระดับโลก (World-Class)

---

## Step 1181: Native AOT คืออะไร

Native AOT (Ahead-of-Time compilation) คือการ compile โค้ด C# เป็น native binary ล่วงหน้า โดยไม่ต้องการ .NET runtime ในเครื่อง target

```
Traditional .NET:
Source (C#) → IL (Intermediate Language) → [Runtime JIT] → Native Code
                                              ↑ เกิดที่ runtime

Native AOT:
Source (C#) → IL → [AOT Compiler at build time] → Native Binary
                    ↑ เกิดที่ build time

Benefits:
✅ Fast startup time (< 50ms แทน 500ms+)
✅ Low memory footprint
✅ No runtime required on target machine
✅ Single self-contained executable
✅ Better cold start for serverless/containers

Limitations:
❌ No dynamic code generation (no Reflection.Emit)
❌ Limited reflection
❌ Larger binary size
❌ No C# features requiring dynamic IL
```

---

## Step 1182: เปิดใช้ Native AOT

```xml
<!-- MyApp.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net9.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>

    <!-- Enable Native AOT -->
    <PublishAot>true</PublishAot>

    <!-- Trimming (required for AOT) -->
    <TrimmerRootAssembly Include="MyApp" />

    <!-- Optimize for size -->
    <OptimizeSpeed>false</OptimizeSpeed>  <!-- set true for speed -->
    <IlcOptimizationPreference>Size</IlcOptimizationPreference>

    <!-- Disable invariant mode (needed for globalization in AOT) -->
    <InvariantGlobalization>false</InvariantGlobalization>
  </PropertyGroup>
</Project>
```

```bash
# Publish as Native AOT
dotnet publish -r linux-x64 -c Release

# Output: bin/Release/net9.0/linux-x64/publish/MyApp (native binary)
# Size: typically 5-20 MB self-contained

# Publish for other platforms
dotnet publish -r win-x64 -c Release       # Windows
dotnet publish -r osx-arm64 -c Release     # macOS Apple Silicon
```

---

## Step 1183: AOT-Compatible Code

```csharp
// ✅ AOT-compatible: static analysis possible
public class UserService
{
    private readonly List<User> _users = [];

    public User? GetById(int id) => _users.FirstOrDefault(u => u.Id == id);

    public void Add(User user) => _users.Add(user);
}

// ❌ NOT AOT-compatible: dynamic code generation
public class DynamicProxy<T>
{
    public T CreateProxy()
    {
        // Reflection.Emit - not supported in AOT
        var ab = AssemblyBuilder.DefineDynamicAssembly(
            new AssemblyName("DynProxy"),
            AssemblyBuilderAccess.Run);
        // ...
        return default!;
    }
}

// ❌ NOT AOT-compatible: unconstrained reflection
public static T Deserialize<T>(string json)
{
    var type = Type.GetType(json); // runtime-determined type
    return (T)JsonSerializer.Deserialize(json, type!)!; // problematic
}

// ✅ AOT-compatible: source-generated JSON
[JsonSerializable(typeof(User))]
[JsonSerializable(typeof(List<User>))]
public partial class AppJsonContext : JsonSerializerContext { }

public static User DeserializeUser(string json)
    => JsonSerializer.Deserialize(json, AppJsonContext.Default.User)!;
```

---

## Step 1184: JSON Source Generation สำหรับ AOT

```csharp
// รองรับ AOT ด้วย JsonSerializerContext
[JsonSourceGenerationOptions(
    WriteIndented = true,
    PropertyNamingPolicy = JsonKnownNamingPolicy.CamelCase,
    DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull)]
[JsonSerializable(typeof(User))]
[JsonSerializable(typeof(Product))]
[JsonSerializable(typeof(Order))]
[JsonSerializable(typeof(OrderLine))]
[JsonSerializable(typeof(List<User>))]
[JsonSerializable(typeof(List<Product>))]
[JsonSerializable(typeof(ApiResponse<User>))]
[JsonSerializable(typeof(ApiResponse<List<Product>>))]
public partial class AppJsonContext : JsonSerializerContext { }

// Usage
public static string Serialize<T>(T value, JsonTypeInfo<T> typeInfo)
    => JsonSerializer.Serialize(value, typeInfo);

var userJson = Serialize(user, AppJsonContext.Default.User);
var productJson = Serialize(product, AppJsonContext.Default.Product);

// Deserialize
var user = JsonSerializer.Deserialize("...", AppJsonContext.Default.User);
```

---

## Step 1185: AOT-Compatible Minimal API

```csharp
// Program.cs - AOT-compatible Minimal API
using System.Text.Json.Serialization;

var builder = WebApplication.CreateSlimBuilder(args);  // AOT-optimized

// Register JSON context
builder.Services.ConfigureHttpJsonOptions(opts =>
    opts.SerializerOptions.TypeInfoResolverChain.Insert(0, AppJsonContext.Default));

var app = builder.Build();

// AOT-compatible endpoints (no MVC reflection)
app.MapGet("/users", (UserRepository repo) => repo.GetAll());

app.MapGet("/users/{id:int}", (int id, UserRepository repo) =>
{
    var user = repo.GetById(id);
    return user is null ? Results.NotFound() : Results.Ok(user);
});

app.MapPost("/users", (User user, UserRepository repo) =>
{
    repo.Add(user);
    return Results.Created($"/users/{user.Id}", user);
});

app.Run();

// Models
public record User(int Id, string Name, string Email);

// AOT JSON context
[JsonSerializable(typeof(User))]
[JsonSerializable(typeof(List<User>))]
internal partial class AppJsonContext : JsonSerializerContext { }
```

---

## Step 1186: Reflection ใน AOT Context

```csharp
// ✅ Reflection ที่รองรับ AOT: ต้องระบุ types ชัดเจน
public class AotFriendlyDi
{
    // ใช้ generic แทน reflection
    public static void Register<TService, TImplementation>(IServiceCollection services)
        where TImplementation : class, TService
    {
        services.AddScoped<TService, TImplementation>();
    }
}

// ❌ ไม่ AOT-compatible: dynamic type scanning
public static void RegisterAll(IServiceCollection services, Assembly assembly)
{
    var types = assembly.GetTypes()  // problematic in AOT
        .Where(t => t.IsClass && !t.IsAbstract);
    // ...
}

// ✅ Source Generator สำหรับ DI registration
// ใช้ [GeneratedDependencyInjection] หรือ IServiceCollection.AddXxx ชัดเจน

// DynamicDependency attribute: hint to AOT linker
[DynamicDependency(DynamicallyAccessedMemberTypes.All, typeof(User))]
public static User DeserializeWithHint(string json)
    => JsonSerializer.Deserialize<User>(json)!;

// RequiresUnreferencedCode: mark as AOT-unsafe
[RequiresUnreferencedCode("Uses reflection - not AOT compatible")]
public static object DeserializeDynamic(string json, Type type)
    => JsonSerializer.Deserialize(json, type)!;
```

---

## Step 1187: Trimming

```csharp
// Trimming: remove unused code from the binary
// รองรับ AOT - ต้องกำหนด root descriptors

// rd.xml: Rooting Descriptor
/*
<linker>
  <assembly fullname="MyApp">
    <type fullname="MyApp.Services.UserService" preserve="all"/>
    <type fullname="MyApp.Models.*" preserve="all"/>
  </assembly>
</linker>
*/

// หรือใช้ DynamicDependency attribute
[DynamicDependency(DynamicallyAccessedMemberTypes.PublicMethods, typeof(UserService))]
public static void EnsureUserServicePreserved() { }

// TrimmerRootDescriptor ใน .csproj
/*
<ItemGroup>
  <TrimmerRootDescriptor Include="TrimmerRoots.xml" />
</ItemGroup>
*/

// Suppress trimming warning
[UnconditionalSuppressMessage("Trimming", "IL2026",
    Justification = "Type is always present in this context")]
public static void SomeMethod() { }
```

---

## Step 1188: Performance Comparison

```csharp
// startup_comparison.cs
// ทดสอบ startup time

// .NET JIT app:
// - Startup: ~200-500ms
// - Memory: ~50-100 MB baseline
// - Binary: ~70 MB (self-contained) หรือ 5 MB (framework-dependent)

// Native AOT app:
// - Startup: < 50ms
// - Memory: ~15-30 MB
// - Binary: ~10-20 MB (self-contained, no framework needed)

public class StartupBenchmark
{
    // Measure from process start to first request served
    public static DateTime AppStart = DateTime.UtcNow;

    public static void ReportStartupTime()
    {
        var elapsed = DateTime.UtcNow - AppStart;
        Console.WriteLine($"Startup time: {elapsed.TotalMilliseconds:F0}ms");
    }
}
```

---

## Step 1189: AOT-Compatible ASP.NET Core Web API

```csharp
// Full AOT Web API
var builder = WebApplication.CreateSlimBuilder(args);

builder.Services.AddSingleton<IProductRepository, InMemoryProductRepository>();
builder.Services.ConfigureHttpJsonOptions(opts =>
{
    opts.SerializerOptions.TypeInfoResolverChain.Insert(0, ProductJsonContext.Default);
});

var app = builder.Build();

var products = app.MapGroup("/api/products");

products.MapGet("/", (IProductRepository repo) => repo.GetAll());

products.MapGet("/{id:int}", (int id, IProductRepository repo) =>
    repo.GetById(id) is { } product ? Results.Ok(product) : Results.NotFound());

products.MapPost("/", (CreateProductRequest req, IProductRepository repo) =>
{
    var product = new Product(repo.NextId(), req.Name, req.Price, req.Stock);
    repo.Add(product);
    return Results.Created($"/api/products/{product.Id}", product);
});

products.MapPut("/{id:int}", (int id, UpdateProductRequest req, IProductRepository repo) =>
{
    if (!repo.Update(id, req.Name, req.Price, req.Stock)) return Results.NotFound();
    return Results.NoContent();
});

products.MapDelete("/{id:int}", (int id, IProductRepository repo) =>
    repo.Delete(id) ? Results.NoContent() : Results.NotFound());

app.Run();

// Models
public record Product(int Id, string Name, decimal Price, int Stock);
public record CreateProductRequest(string Name, decimal Price, int Stock);
public record UpdateProductRequest(string Name, decimal Price, int Stock);

// Repository
public interface IProductRepository
{
    IReadOnlyList<Product> GetAll();
    Product? GetById(int id);
    int NextId();
    void Add(Product product);
    bool Update(int id, string name, decimal price, int stock);
    bool Delete(int id);
}

public class InMemoryProductRepository : IProductRepository
{
    private readonly List<Product> _products = [];
    private int _nextId = 1;

    public IReadOnlyList<Product> GetAll() => _products.AsReadOnly();
    public Product? GetById(int id) => _products.FirstOrDefault(p => p.Id == id);
    public int NextId() => _nextId++;

    public void Add(Product product) => _products.Add(product);

    public bool Update(int id, string name, decimal price, int stock)
    {
        var idx = _products.FindIndex(p => p.Id == id);
        if (idx < 0) return false;
        _products[idx] = _products[idx] with { Name = name, Price = price, Stock = stock };
        return true;
    }

    public bool Delete(int id)
    {
        var product = _products.FirstOrDefault(p => p.Id == id);
        return product is not null && _products.Remove(product);
    }
}

// AOT JSON context - must declare ALL types used in serialization
[JsonSerializable(typeof(Product))]
[JsonSerializable(typeof(List<Product>))]
[JsonSerializable(typeof(CreateProductRequest))]
[JsonSerializable(typeof(UpdateProductRequest))]
internal partial class ProductJsonContext : JsonSerializerContext { }
```

---

## Step 1190: gRPC ใน Native AOT

```csharp
// gRPC with Native AOT (.NET 9+)
// Protobuf generation is AOT-compatible

// Greet.proto
/*
syntax = "proto3";
service Greeter {
  rpc SayHello (HelloRequest) returns (HelloReply);
}
message HelloRequest { string name = 1; }
message HelloReply { string message = 1; }
*/

// Generated code is AOT-compatible
public class GreeterService : Greeter.GreeterBase
{
    public override Task<HelloReply> SayHello(
        HelloRequest request,
        ServerCallContext context)
        => Task.FromResult(new HelloReply
        {
            Message = $"Hello {request.Name} from Native AOT!"
        });
}

// Program.cs
var builder = WebApplication.CreateSlimBuilder(args);
builder.Services.AddGrpc();

var app = builder.Build();
app.MapGrpcService<GreeterService>();
app.Run();
```

---

## Step 1191: AOT Lambda Functions (AWS/Azure)

```csharp
// AWS Lambda with Native AOT
// NuGet: Amazon.Lambda.RuntimeSupport, Amazon.Lambda.Serialization.SystemTextJson

[assembly: LambdaSerializer(typeof(SourceGeneratorLambdaJsonSerializer<AppJsonContext>))]

public class Function
{
    public static async Task<APIGatewayProxyResponse> FunctionHandler(
        APIGatewayProxyRequest request,
        ILambdaContext context)
    {
        var name = request.QueryStringParameters?.GetValueOrDefault("name") ?? "World";

        return new APIGatewayProxyResponse
        {
            StatusCode = 200,
            Headers = new Dictionary<string, string>
            {
                ["Content-Type"] = "application/json"
            },
            Body = JsonSerializer.Serialize(
                new { message = $"Hello {name}!" },
                AppJsonContext.Default.AnonymousMessage)
        };
    }
}

[JsonSerializable(typeof(APIGatewayProxyRequest))]
[JsonSerializable(typeof(APIGatewayProxyResponse))]
[JsonSerializable(typeof(Dictionary<string, string>))]
public partial class AppJsonContext : JsonSerializerContext { }
```

---

## Step 1192: ReadyToRun (R2R) - Middle Ground

```xml
<!-- ReadyToRun: pre-compile IL to native, but keep JIT capability -->
<PropertyGroup>
  <!-- R2R: faster startup, keeps JIT for uncommon paths -->
  <PublishReadyToRun>true</PublishReadyToRun>
  <!-- Full self-contained single file -->
  <PublishSingleFile>true</PublishSingleFile>
  <SelfContained>true</SelfContained>
  <!-- Profile-guided optimization -->
  <PublishReadyToRunComposite>true</PublishReadyToRunComposite>
</PropertyGroup>
```

```
AOT vs R2R vs JIT comparison:
┌────────────────┬──────────┬────────┬──────────┐
│ Feature        │ JIT      │ R2R    │ AOT      │
├────────────────┼──────────┼────────┼──────────┤
│ Startup Time   │ Slow     │ Fast   │ Fastest  │
│ Peak Perf      │ Best     │ Good   │ Good     │
│ Memory         │ Higher   │ Medium │ Lowest   │
│ Binary Size    │ Smallest │ Medium │ Larger   │
│ Dynamic Code   │ Full     │ Full   │ None     │
│ Reflection     │ Full     │ Full   │ Limited  │
│ Debug Info     │ Full     │ Full   │ Limited  │
└────────────────┴──────────┴────────┴──────────┘
```

---

## Step 1193: PGO (Profile-Guided Optimization)

```xml
<!-- Dynamic PGO: profile at runtime then optimize hot paths -->
<PropertyGroup>
  <!-- .NET 7+: Dynamic PGO enabled by default in Release -->
  <!-- Explicitly configure: -->
  <TieredPGO>true</TieredPGO>
  <TieredCompilation>true</TieredCompilation>
</PropertyGroup>
```

```csharp
// PGO hints: OptimizeSpeed / OptimizeSize
[MethodImpl(MethodImplOptions.AggressiveOptimization)]
public static int HotPath(int[] data)
{
    int sum = 0;
    foreach (var n in data) sum += n;
    return sum;
}

[MethodImpl(MethodImplOptions.NoOptimization)]
public static void ColdPath()
{
    // Rarely called - don't waste resources optimizing
    Console.WriteLine("Cold path executed");
}

// Profile-Guided AOT: collect PGO data first, then AOT compile
// Step 1: Run with PGO data collection
// dotnet run --configuration Release /p:CollectPgo=true

// Step 2: Publish with PGO data
// dotnet publish -r linux-x64 -c Release /p:PgoDataPath=./pgo-data
```

---

## Step 1194: Native AOT Console App

```csharp
// Complete Native AOT console app
using System.Runtime.InteropServices;

Console.WriteLine($"Runtime: {RuntimeInformation.RuntimeIdentifier}");
Console.WriteLine($"Framework: {RuntimeInformation.FrameworkDescription}");
Console.WriteLine($"Process: {RuntimeInformation.ProcessArchitecture}");
Console.WriteLine($"OS: {RuntimeInformation.OSDescription}");

// Native AOT: can still use most .NET features
var data = Enumerable.Range(1, 1_000_000).ToArray();

// Span operations work fine
var span = data.AsSpan();
long sum = 0;
foreach (var n in span) sum += n;

Console.WriteLine($"Sum: {sum:N0}");
Console.WriteLine($"Memory: {GC.GetTotalMemory(false) / 1024:N0} KB");

// Timer works
var sw = Stopwatch.StartNew();
await Task.Delay(100);
Console.WriteLine($"Task.Delay(100): {sw.ElapsedMilliseconds}ms");

// HTTP works
using var http = new HttpClient();
var response = await http.GetStringAsync("https://httpbin.org/get");
Console.WriteLine($"HTTP response length: {response.Length}");
```

---

## Step 1195: AOT Diagnostics

```bash
# Check AOT warnings during build
dotnet publish -r linux-x64 -c Release --verbosity normal 2>&1 | grep -i "warning"

# Common AOT warnings:
# IL2026: Method uses reflection - may fail at runtime
# IL2055: typeof(T) call with unconstrained T
# IL2067: Argument passed to parameter which is not trimming compatible
# IL2075: Type discovered at runtime may be trimmed

# Suppress specific warnings in code:
# [RequiresUnreferencedCode("...")]
# [UnconditionalSuppressMessage("Trimming", "IL2026", ...)]
```

```csharp
// Handle AOT diagnostics
[RequiresDynamicCode("This method uses dynamic code generation")]
[RequiresUnreferencedCode("This method uses reflection")]
public static object CreateInstance(string typeName)
{
    var type = Type.GetType(typeName) ?? throw new InvalidOperationException($"Type {typeName} not found");
    return Activator.CreateInstance(type)!;
}

// AOT-compatible alternative: factory registry
public static class TypeRegistry
{
    private static readonly Dictionary<string, Func<object>> _factories = new()
    {
        ["User"] = () => new User(0, "", ""),
        ["Product"] = () => new Product(0, "", 0, 0),
    };

    public static object Create(string typeName)
    {
        if (_factories.TryGetValue(typeName, out var factory)) return factory();
        throw new KeyNotFoundException($"Type {typeName} not registered");
    }
}
```

---

## Step 1196: SizeOf and Alignment in AOT

```csharp
// Structure layout for interop in AOT
[StructLayout(LayoutKind.Sequential)]
public struct Point2D
{
    public float X;
    public float Y;
}

// Verify layout
Console.WriteLine($"Point2D size: {Unsafe.SizeOf<Point2D>()} bytes"); // 8
Console.WriteLine($"Point2D alignment: {RuntimeHelpers.SizeOf<Point2D>()}");

// Blittable types (directly usable in P/Invoke)
[StructLayout(LayoutKind.Sequential, Pack = 1)]
public struct PackedStruct
{
    public byte A;
    public short B;  // Pack=1: no padding, A+B+C = 7 bytes
    public int C;
}

// Verify with Span
var packet = new PackedStruct { A = 1, B = 2, C = 3 };
var bytes = MemoryMarshal.AsBytes(MemoryMarshal.CreateReadOnlySpan(ref packet, 1));
Console.WriteLine($"Packed size: {bytes.Length}"); // 7
```

---

## Step 1197: AOT Testing

```csharp
// Test AOT compatibility
[Fact]
public void Serialization_WithAotContext_ShouldWork()
{
    var user = new User(1, "John", "john@example.com");

    // Use source-generated context
    var json = JsonSerializer.Serialize(user, AppJsonContext.Default.User);
    var deserialized = JsonSerializer.Deserialize(json, AppJsonContext.Default.User);

    deserialized.Should().NotBeNull();
    deserialized!.Id.Should().Be(1);
    deserialized.Name.Should().Be("John");
}

[Fact]
public async Task HttpEndpoint_AotCompatible_Returns200()
{
    await using var app = await TestApp.CreateAsync();
    using var client = app.CreateClient();

    var response = await client.GetAsync("/api/products");
    response.StatusCode.Should().Be(HttpStatusCode.OK);

    var products = await response.Content
        .ReadFromJsonAsync(AppJsonContext.Default.ListProduct);
    products.Should().NotBeNull();
}
```

---

## Step 1198: Interop ใน AOT

```csharp
// P/Invoke ใน AOT (fully supported)
public static partial class SystemInterop
{
    // Windows
    [LibraryImport("ntdll.dll")]
    private static partial int NtQuerySystemInformation(
        int systemInformationClass,
        nint systemInformation,
        int systemInformationLength,
        out int returnLength);

    // Linux/macOS
    [LibraryImport("libc")]
    private static partial int getpid();

    public static int GetProcessId()
    {
        if (OperatingSystem.IsLinux() || OperatingSystem.IsMacOS())
            return getpid();
        return Environment.ProcessId;
    }
}

// Custom marshaling
[StructLayout(LayoutKind.Sequential, CharSet = CharSet.Unicode)]
public struct WIN32_FIND_DATA
{
    public uint FileAttributes;
    public System.Runtime.InteropServices.ComTypes.FILETIME CreationTime;
    public System.Runtime.InteropServices.ComTypes.FILETIME LastAccessTime;
    public System.Runtime.InteropServices.ComTypes.FILETIME LastWriteTime;
    public uint FileSizeHigh;
    public uint FileSizeLow;
    public uint Reserved0;
    public uint Reserved1;
    [MarshalAs(UnmanagedType.ByValTStr, SizeConst = 260)]
    public string cFileName;
    [MarshalAs(UnmanagedType.ByValTStr, SizeConst = 14)]
    public string cAlternateFileName;
}
```

---

## Step 1199: Docker สำหรับ Native AOT

```dockerfile
# Multi-stage build for Native AOT
# Stage 1: Build
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /src

# Install AOT prerequisites
RUN apt-get update && apt-get install -y clang zlib1g-dev

COPY ["MyApp.csproj", "."]
RUN dotnet restore -r linux-x64

COPY . .
RUN dotnet publish -c Release -r linux-x64 --no-restore \
    -o /app/publish

# Stage 2: Runtime (scratch or distroless - no .NET runtime needed!)
FROM mcr.microsoft.com/dotnet/runtime-deps:9.0-jammy-chiseled AS runtime
WORKDIR /app
COPY --from=build /app/publish .

# Non-root user
USER $APP_UID

EXPOSE 8080
ENTRYPOINT ["./MyApp"]
```

```yaml
# docker-compose.yml
services:
  myapp-aot:
    build: .
    ports: ["8080:8080"]
    environment:
      ASPNETCORE_URLS: "http://+:8080"
    # No .NET runtime in container - just native binary!
    # Container size: ~50MB instead of ~200MB
```

---

## Step 1200: รวม Native AOT Best Practices

```csharp
/*
 * Native AOT Best Practices:
 *
 * ✅ DO:
 * - ใช้ source-generated JSON (JsonSerializerContext)
 * - ใช้ Minimal API สำหรับ Web
 * - ใช้ generic แทน reflection
 * - ระบุ DynamicDependency สำหรับ dynamic types
 * - ทดสอบ AOT build ใน CI pipeline
 * - ใช้ LibraryImport แทน DllImport
 * - ใช้ [RequiresUnreferencedCode] เพื่อ document unsafe methods
 *
 * ❌ DON'T:
 * - ใช้ Reflection.Emit
 * - ใช้ dynamic type loading (Assembly.LoadFile)
 * - ใช้ Expression.Compile ใน hot paths
 * - assume reflection works for private members
 * - ใช้ third-party libraries ที่ไม่รองรับ AOT
 *
 * Libraries ที่รองรับ AOT (.NET 9+):
 * ✅ System.Text.Json (with source gen)
 * ✅ gRPC (protobuf-net)
 * ✅ EF Core (limited - use Dapper/raw SQL for better compat)
 * ✅ Microsoft.Extensions.DI (explicit registrations)
 * ✅ Serilog (with AOT-compatible sinks)
 * ✅ HttpClient
 * ✅ System.IO.Pipelines
 *
 * Use Cases for Native AOT:
 * ✅ AWS Lambda / Azure Functions
 * ✅ CLI tools (fast startup)
 * ✅ IoT / Edge devices
 * ✅ Microservices (low memory)
 * ✅ High-frequency APIs (minimal latency)
 */
```

---

## Step 1201: Native AOT Microservice Example

```csharp
// Complete production-ready AOT microservice
var builder = WebApplication.CreateSlimBuilder(args);

// Logging
builder.Logging.AddConsole(opts =>
    opts.FormatterName = ConsoleFormatterNames.Json);

// Health checks
builder.Services.AddHealthChecks();

// Services
builder.Services.AddSingleton<IOrderService, OrderService>();
builder.Services.AddSingleton<OrderEventPublisher>();

// Configure JSON serialization
builder.Services.ConfigureHttpJsonOptions(options =>
    options.SerializerOptions.TypeInfoResolverChain.Insert(0, OrderApiContext.Default));

var app = builder.Build();

// Middleware
app.UseExceptionHandler(ex =>
    ex.Run(async context =>
    {
        context.Response.StatusCode = 500;
        await context.Response.WriteAsJsonAsync(
            new { error = "Internal server error" },
            OrderApiContext.Default.AnonymousError);
    }));

// Endpoints
app.MapHealthChecks("/health");
app.MapOrderEndpoints();

app.Run();

// Models
public record CreateOrderRequest(Guid CustomerId, IReadOnlyList<OrderItem> Items);
public record OrderItem(Guid ProductId, int Quantity);
public record OrderResponse(Guid Id, string Status, decimal Total);

// JSON context
[JsonSerializable(typeof(CreateOrderRequest))]
[JsonSerializable(typeof(OrderResponse))]
[JsonSerializable(typeof(List<OrderResponse>))]
[JsonSerializable(typeof(object))]
internal partial class OrderApiContext : JsonSerializerContext { }

// Endpoints extension
public static class OrderApiExtensions
{
    public static void MapOrderEndpoints(this WebApplication app)
    {
        var group = app.MapGroup("/api/orders").WithTags("Orders");

        group.MapPost("/", async (
            CreateOrderRequest req,
            IOrderService service,
            CancellationToken ct) =>
        {
            var order = await service.CreateAsync(req, ct);
            return Results.Created($"/api/orders/{order.Id}", order);
        });

        group.MapGet("/{id:guid}", async (
            Guid id,
            IOrderService service,
            CancellationToken ct) =>
        {
            var order = await service.GetByIdAsync(id, ct);
            return order is null ? Results.NotFound() : Results.Ok(order);
        });
    }
}
```

---

## Step 1202-1210: AOT Ecosystem

```
AOT-Compatible Libraries (2024-2025):
┌────────────────────────────────────────────────────────┐
│  Web Frameworks                                         │
│  ✅ ASP.NET Core Minimal API                           │
│  ✅ Carter (Minimal API extension)                     │
│  ⚠️  ASP.NET MVC (limited)                            │
│                                                         │
│  Databases                                              │
│  ✅ Dapper (SQL micro-ORM)                             │
│  ✅ Npgsql (PostgreSQL)                               │
│  ⚠️  EF Core (partial support in .NET 9+)             │
│  ✅ MongoDB.Driver (2.x+)                             │
│  ✅ StackExchange.Redis                               │
│                                                         │
│  Messaging                                              │
│  ✅ gRPC (protobuf-net)                               │
│  ⚠️  MassTransit (partial)                            │
│  ✅ Azure Service Bus SDK                             │
│                                                         │
│  Serialization                                          │
│  ✅ System.Text.Json (with source gen)                │
│  ❌ Newtonsoft.Json                                   │
│  ✅ MessagePack (source gen mode)                     │
│                                                         │
│  DI/IoC                                                 │
│  ✅ Microsoft.Extensions.DI                           │
│  ⚠️  Autofac (partial)                               │
│  ✅ Pure DI (manual composition)                      │
│                                                         │
│  Testing                                                │
│  ✅ xUnit (AOT tests in .NET 9+)                      │
│  ✅ FluentAssertions                                  │
└────────────────────────────────────────────────────────┘
```

---

*Part 41 ครอบคลุม Native AOT & Ahead-of-Time Compilation ทั้งหมด 30 Steps (1181-1210)*  
*ต่อไป Part 42: AI/ML Integration with .NET*
