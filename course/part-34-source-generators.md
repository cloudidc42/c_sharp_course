# Part 34: Source Generators & Compile-Time Code Generation

## Steps 951-980: Metaprogramming in .NET

---

## Step 951: Source Generators คืออะไร?

Source Generators คือ code ที่รันใน compiler pipeline และสร้าง C# code เพิ่มเติมในขณะ compile

```
Without Source Generators:
Developer writes code → Compiler compiles

With Source Generators:
Developer writes code → Generator inspects code → Generator creates new code → Compiler compiles all
```

### ประโยชน์
- **Zero-cost abstractions**: ไม่มี runtime overhead
- **AOT-compatible**: ใช้แทน reflection
- **IDE support**: Auto-complete สำหรับ generated code
- **Debuggable**: สามารถ debug generated code ได้

---

## Step 952: สร้าง Source Generator เบื้องต้น

```xml
<!-- MyGenerator/MyGenerator.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
    <LangVersion>latest</LangVersion>
    <Nullable>enable</Nullable>
    <!-- Required for generators -->
    <EnforceExtendedAnalyzerRules>true</EnforceExtendedAnalyzerRules>
    <IsRoslynComponent>true</IsRoslynComponent>
  </PropertyGroup>
  
  <ItemGroup>
    <PackageReference Include="Microsoft.CodeAnalysis.CSharp" Version="4.9.2" />
    <PackageReference Include="Microsoft.CodeAnalysis.Analyzers" Version="3.3.4" PrivateAssets="all" />
  </ItemGroup>
</Project>
```

```csharp
// HelloWorldGenerator.cs
using Microsoft.CodeAnalysis;

namespace MyGenerator;

[Generator]
public class HelloWorldGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        // Register a syntax receiver to collect relevant nodes
        context.RegisterPostInitializationOutput(ctx =>
        {
            // Generate code that's always added
            ctx.AddSource("HelloWorld.g.cs", """
                namespace Generated;
                
                public static class HelloWorld
                {
                    public static string Greet(string name) => $"Hello, {name}!";
                }
                """);
        });
    }
}
```

---

## Step 953: Incremental Generator Pattern

```csharp
// AutoPropertyGenerator.cs
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp;
using Microsoft.CodeAnalysis.CSharp.Syntax;
using Microsoft.CodeAnalysis.Text;
using System.Collections.Immutable;
using System.Text;

namespace MyGenerator;

[Generator]
public class AutoPropertyGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        // 1. Filter syntax nodes
        var classDeclarations = context.SyntaxProvider
            .CreateSyntaxProvider(
                predicate: static (node, _) => IsTargetNode(node),
                transform: static (ctx, _) => GetTargetClass(ctx))
            .Where(static c => c != null)
            .Select(static (c, _) => c!);
        
        // 2. Combine with compilation
        var compilationAndClasses = context.CompilationProvider
            .Combine(classDeclarations.Collect());
        
        // 3. Generate source
        context.RegisterSourceOutput(compilationAndClasses, Execute);
    }
    
    private static bool IsTargetNode(SyntaxNode node)
        => node is ClassDeclarationSyntax cls &&
           cls.AttributeLists.Count > 0;
    
    private static ClassDeclarationSyntax? GetTargetClass(
        GeneratorSyntaxContext ctx)
    {
        var cls = (ClassDeclarationSyntax)ctx.Node;
        
        foreach (var attributeList in cls.AttributeLists)
        foreach (var attribute in attributeList.Attributes)
        {
            var name = ctx.SemanticModel.GetTypeInfo(attribute).Type?.Name;
            if (name == "GeneratePropertiesAttribute")
                return cls;
        }
        
        return null;
    }
    
    private static void Execute(
        SourceProductionContext context,
        (Compilation Left, ImmutableArray<ClassDeclarationSyntax> Right) source)
    {
        var (compilation, classes) = source;
        
        foreach (var cls in classes)
        {
            var model = compilation.GetSemanticModel(cls.SyntaxTree);
            var symbol = model.GetDeclaredSymbol(cls);
            if (symbol == null) continue;
            
            var code = GenerateCode(symbol);
            context.AddSource($"{symbol.Name}.g.cs", code);
        }
    }
    
    private static string GenerateCode(INamedTypeSymbol symbol)
    {
        var sb = new StringBuilder();
        var ns = symbol.ContainingNamespace.ToDisplayString();
        
        sb.AppendLine($"namespace {ns};");
        sb.AppendLine();
        sb.AppendLine($"public partial class {symbol.Name}");
        sb.AppendLine("{");
        
        // Generate backing fields and properties for each field
        foreach (var member in symbol.GetMembers().OfType<IFieldSymbol>())
        {
            if (!member.Name.StartsWith("_")) continue;
            
            var propName = char.ToUpper(member.Name[1]) + member.Name[2..];
            var typeName = member.Type.ToDisplayString();
            
            sb.AppendLine($"    public {typeName} {propName}");
            sb.AppendLine("    {");
            sb.AppendLine($"        get => {member.Name};");
            sb.AppendLine($"        set");
            sb.AppendLine("        {");
            sb.AppendLine($"            if ({member.Name} == value) return;");
            sb.AppendLine($"            {member.Name} = value;");
            sb.AppendLine($"            OnPropertyChanged(nameof({propName}));");
            sb.AppendLine("        }");
            sb.AppendLine("    }");
            sb.AppendLine();
        }
        
        sb.AppendLine("}");
        return sb.ToString();
    }
}
```

---

## Step 954: Attribute Marking Pattern

```csharp
// เพิ่ม attribute ที่ใช้ mark classes
[AttributeUsage(AttributeTargets.Class)]
public class GeneratePropertiesAttribute : Attribute { }

// User code
[GenerateProperties]
public partial class ProductViewModel
{
    private string _name = string.Empty;
    private decimal _price;
    private int _quantity;
}

// Generated code (ProductViewModel.g.cs):
// public string Name { get => _name; set { if (_name == value) return; _name = value; OnPropertyChanged(nameof(Name)); } }
// public decimal Price { ... }
// public int Quantity { ... }
```

---

## Step 955: ToString Generator

```csharp
[Generator]
public class ToStringGenerator : IIncrementalGenerator
{
    private const string AttributeName = "GenerateToStringAttribute";
    private const string AttributeSource = """
        namespace Generated;
        
        [AttributeUsage(AttributeTargets.Class | AttributeTargets.Struct)]
        public sealed class GenerateToStringAttribute : Attribute { }
        """;
    
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        // Inject attribute
        context.RegisterPostInitializationOutput(ctx =>
            ctx.AddSource("GenerateToStringAttribute.g.cs", AttributeSource));
        
        var targets = context.SyntaxProvider
            .ForAttributeWithMetadataName(
                $"Generated.{AttributeName}",
                predicate: (node, _) => node is ClassDeclarationSyntax 
                                        or StructDeclarationSyntax,
                transform: (ctx, _) => ctx.TargetSymbol as INamedTypeSymbol)
            .Where(s => s != null)
            .Select((s, _) => s!);
        
        context.RegisterSourceOutput(targets, GenerateToString);
    }
    
    private static void GenerateToString(
        SourceProductionContext ctx, INamedTypeSymbol symbol)
    {
        var ns = symbol.ContainingNamespace.IsGlobalNamespace
            ? string.Empty
            : $"namespace {symbol.ContainingNamespace.ToDisplayString()};\n\n";
        
        var properties = symbol.GetMembers()
            .OfType<IPropertySymbol>()
            .Where(p => p.GetMethod != null && 
                       !p.IsStatic &&
                       p.DeclaredAccessibility == Accessibility.Public)
            .ToList();
        
        var parts = properties
            .Select(p => $"{p.Name}: {{{p.Name}}}");
        
        var typeName = symbol.IsValueType ? "readonly record struct" : "partial class";
        var format = string.Join(", ", parts);
        
        var code = $$"""
            {{ns}}public partial {{(symbol.IsValueType ? "struct" : "class")}} {{symbol.Name}}
            {
                public override string ToString()
                    => $"{{symbol.Name}} { {{format}} }";
            }
            """;
        
        ctx.AddSource($"{symbol.Name}.ToString.g.cs", code);
    }
}
```

---

## Step 956: Mapper Generator (Mapster-like)

```csharp
[Generator]
public class MapperGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        context.RegisterPostInitializationOutput(ctx =>
            ctx.AddSource("MapToAttribute.g.cs", """
                namespace Generated;
                
                [AttributeUsage(AttributeTargets.Class)]
                public sealed class MapToAttribute<TTarget> : Attribute { }
                """));
        
        var maps = context.SyntaxProvider
            .ForAttributeWithMetadataName(
                "Generated.MapToAttribute`1",
                predicate: (node, _) => node is ClassDeclarationSyntax,
                transform: GetMappingInfo)
            .Where(m => m != null)
            .Select((m, _) => m!);
        
        context.RegisterSourceOutput(maps, GenerateMapper);
    }
    
    private static MappingInfo? GetMappingInfo(
        GeneratorAttributeSyntaxContext ctx, CancellationToken ct)
    {
        if (ctx.TargetSymbol is not INamedTypeSymbol source) return null;
        
        var attribute = ctx.Attributes.FirstOrDefault(a =>
            a.AttributeClass?.Name == "MapToAttribute");
        
        if (attribute?.AttributeClass?.TypeArguments.FirstOrDefault() 
            is not INamedTypeSymbol target) return null;
        
        return new MappingInfo(source, target);
    }
    
    private static void GenerateMapper(
        SourceProductionContext ctx, MappingInfo info)
    {
        var sourceProps = info.Source.GetMembers()
            .OfType<IPropertySymbol>()
            .Where(p => p.GetMethod != null)
            .ToDictionary(p => p.Name);
        
        var targetProps = info.Target.GetMembers()
            .OfType<IPropertySymbol>()
            .Where(p => p.SetMethod != null);
        
        var assignments = targetProps
            .Where(tp => sourceProps.ContainsKey(tp.Name) &&
                        SymbolEqualityComparer.Default.Equals(
                            sourceProps[tp.Name].Type, tp.Type))
            .Select(tp => $"            {tp.Name} = source.{tp.Name},");
        
        var ns = info.Source.ContainingNamespace.ToDisplayString();
        var sourceName = info.Source.Name;
        var targetName = info.Target.ToDisplayString();
        
        var code = $$"""
            namespace {{ns}};
            
            public static partial class {{sourceName}}Mapper
            {
                public static {{targetName}} MapTo{{info.Target.Name}}(this {{sourceName}} source)
                    => new()
                    {
            {{string.Join("\n", assignments)}}
                    };
            }
            """;
        
        ctx.AddSource($"{sourceName}Mapper.g.cs", code);
    }
    
    private record MappingInfo(INamedTypeSymbol Source, INamedTypeSymbol Target);
}
```

---

## Step 957: Dependency Injection Generator

```csharp
// Auto-register services via attributes
[Generator]
public class DependencyInjectionGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        context.RegisterPostInitializationOutput(ctx =>
            ctx.AddSource("InjectableAttributes.g.cs", """
                namespace Generated;
                
                [AttributeUsage(AttributeTargets.Class)]
                public sealed class SingletonAttribute : Attribute { }
                
                [AttributeUsage(AttributeTargets.Class)]
                public sealed class ScopedAttribute : Attribute { }
                
                [AttributeUsage(AttributeTargets.Class)]
                public sealed class TransientAttribute : Attribute { }
                """));
        
        var singletons = GetServicesForLifetime(context, "Generated.SingletonAttribute");
        var scoped = GetServicesForLifetime(context, "Generated.ScopedAttribute");
        var transient = GetServicesForLifetime(context, "Generated.TransientAttribute");
        
        context.RegisterSourceOutput(
            singletons.Collect().Combine(scoped.Collect().Combine(transient.Collect())),
            GenerateRegistrations);
    }
    
    private static IncrementalValuesProvider<ServiceInfo> GetServicesForLifetime(
        IncrementalGeneratorInitializationContext context, string attributeName)
    {
        return context.SyntaxProvider
            .ForAttributeWithMetadataName(
                attributeName,
                predicate: (node, _) => node is ClassDeclarationSyntax,
                transform: (ctx, _) =>
                {
                    if (ctx.TargetSymbol is not INamedTypeSymbol impl) return null;
                    var iface = impl.Interfaces.FirstOrDefault();
                    return new ServiceInfo(impl.ToDisplayString(), 
                        iface?.ToDisplayString(), 
                        attributeName.Split('.').Last().Replace("Attribute", ""));
                })
            .Where(s => s != null)
            .Select((s, _) => s!);
    }
    
    private static void GenerateRegistrations(
        SourceProductionContext ctx,
        (ImmutableArray<ServiceInfo> Singletons, 
         (ImmutableArray<ServiceInfo> Scoped, ImmutableArray<ServiceInfo> Transient)) source)
    {
        var allServices = source.Singletons
            .Concat(source.Item2.Scoped)
            .Concat(source.Item2.Transient)
            .ToList();
        
        if (!allServices.Any()) return;
        
        var registrations = allServices.Select(s =>
        {
            var method = s.Lifetime == "Singleton" ? "AddSingleton"
                : s.Lifetime == "Scoped" ? "AddScoped" : "AddTransient";
            
            return s.Interface != null
                ? $"        services.{method}<{s.Interface}, {s.Implementation}>();"
                : $"        services.{method}<{s.Implementation}>();";
        });
        
        var code = $$"""
            using Microsoft.Extensions.DependencyInjection;
            
            namespace Generated;
            
            public static class ServiceRegistrationExtensions
            {
                public static IServiceCollection AddGeneratedServices(
                    this IServiceCollection services)
                {
            {{string.Join("\n", registrations)}}
                    return services;
                }
            }
            """;
        
        ctx.AddSource("ServiceRegistrations.g.cs", code);
    }
    
    private record ServiceInfo(string Implementation, string? Interface, string Lifetime);
}
```

---

## Step 958: Validation Generator

```csharp
// Generate validation methods from attributes
[Generator]
public class ValidationGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        var records = context.SyntaxProvider
            .CreateSyntaxProvider(
                predicate: (node, _) => node is RecordDeclarationSyntax,
                transform: (ctx, _) =>
                {
                    var record = (RecordDeclarationSyntax)ctx.Node;
                    var symbol = ctx.SemanticModel.GetDeclaredSymbol(record) 
                        as INamedTypeSymbol;
                    return symbol;
                })
            .Where(s => s?.GetMembers()
                .OfType<IPropertySymbol>()
                .Any(p => p.GetAttributes().Any(a => 
                    a.AttributeClass?.Name.EndsWith("Attribute") == true &&
                    IsValidationAttribute(a.AttributeClass.Name))) == true)
            .Select((s, _) => s!);
        
        context.RegisterSourceOutput(records, GenerateValidation);
    }
    
    private static bool IsValidationAttribute(string name)
        => name is "RequiredAttribute" or "RangeAttribute" or 
                   "MaxLengthAttribute" or "MinLengthAttribute";
    
    private static void GenerateValidation(
        SourceProductionContext ctx, INamedTypeSymbol symbol)
    {
        var validations = new List<string>();
        
        foreach (var prop in symbol.GetMembers().OfType<IPropertySymbol>())
        {
            foreach (var attr in prop.GetAttributes())
            {
                var validation = attr.AttributeClass?.Name switch
                {
                    "RequiredAttribute" => 
                        $"""
                        if (string.IsNullOrEmpty({prop.Name}))
                            errors.Add(new ValidationError("{prop.Name}", "{prop.Name} is required"));
                        """,
                    "MaxLengthAttribute" when attr.ConstructorArguments.Length > 0 =>
                        $"""
                        if ({prop.Name}?.Length > {attr.ConstructorArguments[0].Value})
                            errors.Add(new ValidationError("{prop.Name}", "{prop.Name} exceeds max length"));
                        """,
                    _ => null
                };
                
                if (validation != null)
                    validations.Add($"        {validation.Trim()}");
            }
        }
        
        if (!validations.Any()) return;
        
        var ns = symbol.ContainingNamespace.ToDisplayString();
        var code = $$"""
            using System.Collections.Generic;
            
            namespace {{ns}};
            
            public partial record {{symbol.Name}}
            {
                public IReadOnlyList<ValidationError> Validate()
                {
                    var errors = new List<ValidationError>();
            {{string.Join("\n", validations)}}
                    return errors;
                }
            }
            
            public record ValidationError(string Field, string Message);
            """;
        
        ctx.AddSource($"{symbol.Name}.Validation.g.cs", code);
    }
}
```

---

## Step 959: Enum Extensions Generator

```csharp
// Generate fast extension methods for enums
[Generator]
public class EnumExtensionsGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        var enums = context.SyntaxProvider
            .CreateSyntaxProvider(
                predicate: (node, _) => node is EnumDeclarationSyntax,
                transform: (ctx, _) => 
                    ctx.SemanticModel.GetDeclaredSymbol(ctx.Node) as INamedTypeSymbol)
            .Where(s => s != null)
            .Select((s, _) => s!);
        
        context.RegisterSourceOutput(enums, GenerateEnumExtensions);
    }
    
    private static void GenerateEnumExtensions(
        SourceProductionContext ctx, INamedTypeSymbol enumSymbol)
    {
        var members = enumSymbol.GetMembers()
            .OfType<IFieldSymbol>()
            .Where(f => f.IsStatic && f.HasConstantValue)
            .ToList();
        
        var ns = enumSymbol.ContainingNamespace.ToDisplayString();
        var typeName = enumSymbol.Name;
        var fullName = enumSymbol.ToDisplayString();
        
        // Fast ToString without reflection
        var toStringCases = members.Select(m =>
            $"            {fullName}.{m.Name} => \"{m.Name}\",");
        
        // Fast Parse without Enum.Parse
        var parseCases = members.Select(m =>
            $"            \"{m.Name}\" => {fullName}.{m.Name},");
        
        // All values array
        var values = string.Join(", ", members.Select(m => $"{fullName}.{m.Name}"));
        
        var code = $$"""
            namespace {{ns}};
            
            public static partial class {{typeName}}Extensions
            {
                private static readonly {{fullName}}[] All{{typeName}}s = 
                    [ {{values}} ];
                
                public static string ToFastString(this {{fullName}} value)
                    => value switch
                    {
            {{string.Join("\n", toStringCases)}}
                        _ => value.ToString()
                    };
                
                public static {{fullName}} Parse{{typeName}}(string value)
                    => value switch
                    {
            {{string.Join("\n", parseCases)}}
                        _ => throw new ArgumentException($"Unknown value: {value}", nameof(value))
                    };
                
                public static bool TryParse{{typeName}}(string value, out {{fullName}} result)
                {
                    try { result = Parse{{typeName}}(value); return true; }
                    catch { result = default; return false; }
                }
                
                public static IReadOnlyList<{{fullName}}> GetAll{{typeName}}s()
                    => All{{typeName}}s;
            }
            """;
        
        ctx.AddSource($"{typeName}Extensions.g.cs", code);
    }
}
```

---

## Step 960: Logging Generator (.NET 6+ style)

```csharp
// ILogger<T> source generation (similar to LoggerMessage.Define)
[Generator]
public class LoggerGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        context.RegisterPostInitializationOutput(ctx =>
            ctx.AddSource("LogMethodAttribute.g.cs", """
                namespace Generated;
                
                using Microsoft.Extensions.Logging;
                
                [AttributeUsage(AttributeTargets.Method, AllowMultiple = false)]
                public sealed class LogMethodAttribute : Attribute
                {
                    public LogLevel Level { get; }
                    public int EventId { get; }
                    public string Message { get; }
                    
                    public LogMethodAttribute(LogLevel level, int eventId, string message)
                    {
                        Level = level;
                        EventId = eventId;
                        Message = message;
                    }
                }
                """));
        
        // Similar to Microsoft.Extensions.Logging.LoggerMessage source generator
        // (which already exists - this is a simplified version for learning)
    }
}

// Usage of Microsoft's built-in logging source generator
public partial class OrderService(ILogger<OrderService> logger)
{
    [LoggerMessage(
        EventId = 1001,
        Level = LogLevel.Information,
        Message = "Processing order {OrderId} for user {UserId}")]
    private static partial void LogOrderProcessing(
        ILogger logger, Guid orderId, string userId);
    
    [LoggerMessage(
        EventId = 1002,
        Level = LogLevel.Error,
        Message = "Failed to process order {OrderId}: {Error}")]
    private static partial void LogOrderFailed(
        ILogger logger, Guid orderId, string error);
    
    public async Task ProcessOrderAsync(Guid orderId, string userId, CancellationToken ct)
    {
        LogOrderProcessing(logger, orderId, userId);
        
        try
        {
            // ... process
        }
        catch (Exception ex)
        {
            LogOrderFailed(logger, orderId, ex.Message);
            throw;
        }
    }
}
```

---

## Step 961: Builder Pattern Generator

```csharp
// Generate fluent builder for classes
[Generator]
public class BuilderGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        context.RegisterPostInitializationOutput(ctx =>
            ctx.AddSource("GenerateBuilderAttribute.g.cs", """
                namespace Generated;
                
                [AttributeUsage(AttributeTargets.Class)]
                public sealed class GenerateBuilderAttribute : Attribute { }
                """));
        
        var targets = context.SyntaxProvider
            .ForAttributeWithMetadataName(
                "Generated.GenerateBuilderAttribute",
                predicate: (node, _) => node is ClassDeclarationSyntax,
                transform: (ctx, _) => ctx.TargetSymbol as INamedTypeSymbol)
            .Where(s => s != null)
            .Select((s, _) => s!);
        
        context.RegisterSourceOutput(targets, GenerateBuilder);
    }
    
    private static void GenerateBuilder(
        SourceProductionContext ctx, INamedTypeSymbol symbol)
    {
        var ns = symbol.ContainingNamespace.ToDisplayString();
        var typeName = symbol.Name;
        
        var props = symbol.GetMembers()
            .OfType<IPropertySymbol>()
            .Where(p => p.SetMethod != null && p.GetMethod != null)
            .ToList();
        
        var fields = props.Select(p => 
            $"    private {p.Type.ToDisplayString()} _{char.ToLower(p.Name[0])}{p.Name[1..]};");
        
        var withMethods = props.Select(p =>
        {
            var fieldName = $"_{char.ToLower(p.Name[0])}{p.Name[1..]}";
            return $$"""
                
                    public {{typeName}}Builder With{{p.Name}}({{p.Type.ToDisplayString()}} value)
                    {
                        {{fieldName}} = value;
                        return this;
                    }
                """;
        });
        
        var buildAssignments = props.Select(p =>
        {
            var fieldName = $"_{char.ToLower(p.Name[0])}{p.Name[1..]}";
            return $"            {p.Name} = {fieldName},";
        });
        
        var code = $$"""
            namespace {{ns}};
            
            public sealed class {{typeName}}Builder
            {
            {{string.Join("\n", fields)}}
            {{string.Join("\n", withMethods)}}
                
                public {{typeName}} Build()
                    => new()
                    {
            {{string.Join("\n", buildAssignments)}}
                    };
                
                public static {{typeName}}Builder Create() => new();
            }
            """;
        
        ctx.AddSource($"{typeName}Builder.g.cs", code);
    }
}

// Usage
[GenerateBuilder]
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public string Category { get; set; } = string.Empty;
}

// Generated: ProductBuilder with fluent API
var product = ProductBuilder.Create()
    .WithId(1)
    .WithName("Laptop")
    .WithPrice(45000)
    .WithCategory("Electronics")
    .Build();
```

---

## Step 962: Interceptors (.NET 8)

```csharp
// Interceptors - intercept method calls at compile time
// Requires: <InterceptorsPreviewNamespaces>$(InterceptorsPreviewNamespaces);MyApp</InterceptorsPreviewNamespaces>

namespace System.Runtime.CompilerServices
{
    [AttributeUsage(AttributeTargets.Method, AllowMultiple = true)]
    public sealed class InterceptsLocationAttribute : Attribute
    {
        public InterceptsLocationAttribute(string filePath, int line, int column) { }
    }
}

// Source: somewhere in the code
public class Logger
{
    public void Log(string message)
    {
        Console.WriteLine($"[LOG] {message}");
    }
}

// Generated interceptor
public static class LoggerInterceptors
{
    [System.Runtime.CompilerServices.InterceptsLocation(
        "Program.cs", 10, 16)] // intercepts specific call site
    public static void InterceptedLog(this Logger logger, string message)
    {
        Console.WriteLine($"[INTERCEPTED LOG at {DateTime.Now:HH:mm:ss}] {message}");
    }
}
```

---

## Step 963: Analyzer + CodeFix

```csharp
// Roslyn Analyzer
[DiagnosticAnalyzer(LanguageNames.CSharp)]
public class AsyncVoidAnalyzer : DiagnosticAnalyzer
{
    public static readonly DiagnosticDescriptor Rule = new(
        id: "ASYNC001",
        title: "Avoid async void",
        messageFormat: "Method '{0}' is async void. Consider returning Task instead",
        category: "AsyncUsage",
        defaultSeverity: DiagnosticSeverity.Warning,
        isEnabledByDefault: true,
        description: "Async void methods cannot be awaited and exceptions are unhandled");
    
    public override ImmutableArray<DiagnosticDescriptor> SupportedDiagnostics => 
        ImmutableArray.Create(Rule);
    
    public override void Initialize(AnalysisContext context)
    {
        context.ConfigureGeneratedCodeAnalysis(GeneratedCodeAnalysisFlags.None);
        context.EnableConcurrentExecution();
        
        context.RegisterSyntaxNodeAction(
            AnalyzeMethod, SyntaxKind.MethodDeclaration);
    }
    
    private static void AnalyzeMethod(SyntaxNodeAnalysisContext context)
    {
        var method = (MethodDeclarationSyntax)context.Node;
        
        var isAsync = method.Modifiers.Any(m => 
            m.IsKind(SyntaxKind.AsyncKeyword));
        
        var isVoid = method.ReturnType is PredefinedTypeSyntax predefined &&
                     predefined.Keyword.IsKind(SyntaxKind.VoidKeyword);
        
        if (!isAsync || !isVoid) return;
        
        // Don't warn for event handlers
        var symbol = context.SemanticModel.GetDeclaredSymbol(method);
        if (symbol?.Parameters.Any(p => 
            p.Type.ToDisplayString() == "System.EventArgs" ||
            p.Type.AllInterfaces.Any(i => i.Name == "IEventArgs")) == true)
            return;
        
        var diagnostic = Diagnostic.Create(Rule, method.GetLocation(), 
            method.Identifier.Text);
        context.ReportDiagnostic(diagnostic);
    }
}

// Code Fix Provider
[ExportCodeFixProvider(LanguageNames.CSharp)]
public class AsyncVoidCodeFixProvider : CodeFixProvider
{
    public override ImmutableArray<string> FixableDiagnosticIds =>
        ImmutableArray.Create(AsyncVoidAnalyzer.Rule.Id);
    
    public override FixAllProvider? GetFixAllProvider() => 
        WellKnownFixAllProviders.BatchFixer;
    
    public override async Task RegisterCodeFixesAsync(CodeFixContext context)
    {
        var root = await context.Document
            .GetSyntaxRootAsync(context.CancellationToken);
        
        var diagnostic = context.Diagnostics.First();
        var diagnosticSpan = diagnostic.Location.SourceSpan;
        
        var declaration = root!.FindToken(diagnosticSpan.Start)
            .Parent!.AncestorsAndSelf()
            .OfType<MethodDeclarationSyntax>()
            .First();
        
        context.RegisterCodeFix(
            CodeAction.Create(
                title: "Change to async Task",
                createChangedDocument: ct => 
                    ChangeVoidToTaskAsync(context.Document, declaration, ct),
                equivalenceKey: "AsyncVoidToTask"),
            diagnostic);
    }
    
    private static async Task<Document> ChangeVoidToTaskAsync(
        Document document, MethodDeclarationSyntax method, CancellationToken ct)
    {
        var root = await document.GetSyntaxRootAsync(ct);
        
        var taskType = SyntaxFactory.ParseTypeName("System.Threading.Tasks.Task ")
            .WithLeadingTrivia(method.ReturnType.GetLeadingTrivia())
            .WithTrailingTrivia(method.ReturnType.GetTrailingTrivia());
        
        var newMethod = method.WithReturnType(taskType);
        var newRoot = root!.ReplaceNode(method, newMethod);
        
        return document.WithSyntaxRoot(newRoot);
    }
}
```

---

## Step 964: Testing Source Generators

```xml
<!-- Test project -->
<PackageReference Include="Microsoft.CodeAnalysis.CSharp" Version="4.9.2" />
<PackageReference Include="Microsoft.CodeAnalysis.CSharp.Workspaces" Version="4.9.2" />
```

```csharp
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp;
using Xunit;

public class ToStringGeneratorTests
{
    [Fact]
    public void GeneratesCorrectToString()
    {
        var source = """
            using Generated;
            
            namespace TestNamespace;
            
            [GenerateToString]
            public partial class Person
            {
                public string Name { get; set; } = string.Empty;
                public int Age { get; set; }
            }
            """;
        
        var (output, diagnostics) = RunGenerator(source);
        
        Assert.Empty(diagnostics.Where(d => d.Severity == DiagnosticSeverity.Error));
        Assert.Contains("public override string ToString()", output);
        Assert.Contains("Name", output);
        Assert.Contains("Age", output);
    }
    
    private static (string Output, IReadOnlyList<Diagnostic> Diagnostics) RunGenerator(
        string source)
    {
        var syntaxTree = CSharpSyntaxTree.ParseText(source);
        
        var references = AppDomain.CurrentDomain.GetAssemblies()
            .Where(a => !a.IsDynamic && !string.IsNullOrEmpty(a.Location))
            .Select(a => MetadataReference.CreateFromFile(a.Location))
            .Cast<MetadataReference>();
        
        var compilation = CSharpCompilation.Create(
            assemblyName: "TestAssembly",
            syntaxTrees: new[] { syntaxTree },
            references: references,
            options: new CSharpCompilationOptions(OutputKind.DynamicallyLinkedLibrary));
        
        var generator = new ToStringGenerator();
        var driver = CSharpGeneratorDriver.Create(generator);
        driver = (CSharpGeneratorDriver)driver.RunGenerators(compilation);
        
        var result = driver.GetRunResult();
        var generatedTrees = result.GeneratedTrees;
        
        var output = generatedTrees.Length > 0
            ? generatedTrees[0].GetText().ToString()
            : string.Empty;
        
        return (output, result.Diagnostics);
    }
}
```

---

## Step 965: Real-World Generator: AutoInterface

```csharp
// Generate interface from implementation class
[Generator]
public class AutoInterfaceGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        context.RegisterPostInitializationOutput(ctx =>
            ctx.AddSource("AutoInterfaceAttribute.g.cs", """
                namespace Generated;
                
                [AttributeUsage(AttributeTargets.Class)]
                public sealed class AutoInterfaceAttribute : Attribute { }
                """));
        
        var classes = context.SyntaxProvider
            .ForAttributeWithMetadataName(
                "Generated.AutoInterfaceAttribute",
                predicate: (node, _) => node is ClassDeclarationSyntax,
                transform: (ctx, _) => ctx.TargetSymbol as INamedTypeSymbol)
            .Where(s => s != null)
            .Select((s, _) => s!);
        
        context.RegisterSourceOutput(classes, GenerateInterface);
    }
    
    private static void GenerateInterface(
        SourceProductionContext ctx, INamedTypeSymbol symbol)
    {
        var ns = symbol.ContainingNamespace.ToDisplayString();
        var typeName = symbol.Name;
        var interfaceName = $"I{typeName}";
        
        var publicMethods = symbol.GetMembers()
            .OfType<IMethodSymbol>()
            .Where(m => m.DeclaredAccessibility == Accessibility.Public &&
                       !m.IsStatic &&
                       m.MethodKind == MethodKind.Ordinary)
            .ToList();
        
        var publicProps = symbol.GetMembers()
            .OfType<IPropertySymbol>()
            .Where(p => p.DeclaredAccessibility == Accessibility.Public &&
                       !p.IsStatic)
            .ToList();
        
        var methodSignatures = publicMethods.Select(m =>
        {
            var returnType = m.ReturnType.ToDisplayString();
            var methodName = m.Name;
            var parameters = string.Join(", ", m.Parameters.Select(p =>
                $"{p.Type.ToDisplayString()} {p.Name}"));
            return $"    {returnType} {methodName}({parameters});";
        });
        
        var propSignatures = publicProps.Select(p =>
        {
            var type = p.Type.ToDisplayString();
            var getSet = (p.GetMethod != null, p.SetMethod != null) switch
            {
                (true, true) => "{ get; set; }",
                (true, false) => "{ get; }",
                (false, true) => "{ set; }",
                _ => ";"
            };
            return $"    {type} {p.Name} {getSet}";
        });
        
        var allMembers = propSignatures.Concat(methodSignatures);
        
        var code = $$"""
            namespace {{ns}};
            
            public interface {{interfaceName}}
            {
            {{string.Join("\n", allMembers)}}
            }
            
            public partial class {{typeName}} : {{interfaceName}} { }
            """;
        
        ctx.AddSource($"{interfaceName}.g.cs", code);
    }
}
```

---

## Step 966-980: สรุป Source Generators

### ประเภทของ Source Generators

| Type | Use Case | Example |
|------|----------|---------|
| IIncrementalGenerator | Modern, recommended | All generators above |
| ISourceGenerator | Legacy, less efficient | Old code |

### Patterns ที่ใช้บ่อย
1. **Attribute marking** - `[GenerateX]` บน class
2. **Syntax filtering** - หา nodes ที่ต้องการ
3. **Semantic model** - ใช้ symbols แทน syntax
4. **Incremental** - cache ผลลัพธ์เพื่อประสิทธิภาพ
5. **PostInitialization** - inject attributes เอง

### Best Practices
1. ใช้ `IIncrementalGenerator` ไม่ใช่ `ISourceGenerator`
2. Cache computations ด้วย `Transform` + `Where`  
3. ใช้ `.ForAttributeWithMetadataName()` แทน manual filtering
4. Test generators ด้วย `CSharpGeneratorDriver`
5. Generate `partial class/method` เสมอ - ไม่ต้อง override existing code

### Existing Source Generators ใน .NET

| Generator | Package | Purpose |
|-----------|---------|---------|
| System.Text.Json | Built-in | JSON serialization |
| LoggerMessage | Microsoft.Extensions.Logging | Structured logging |
| Regex | System.Text.RegularExpressions | Compiled regex |
| LibraryImport | Runtime | P/Invoke |
| JsonSerializerContext | Built-in | AOT JSON |

---

*จบ Part 34: Source Generators & Compile-Time Code Generation*
*ต่อไป Part 35: Testing Strategies - Unit, Integration, E2E*
