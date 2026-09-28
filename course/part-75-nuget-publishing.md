# Part 75: NuGet Package Publishing — Reusable Libraries, Source Generators as NuGet & Package Governance

## Steps 1829–1844 | World-Class Library Development

---

## Step 1829: Why Publish NuGet Packages?

```
Internal Use Cases:
┌─────────────────────────────────────────────────────────────────────┐
│  Shared Domain Primitives  → Money, Address, ValueObject base       │
│  Shared Middleware         → Tenant resolution, auth schemes        │
│  Source Generators         → AutoMapper, AutoProperty, AutoValidate │
│  Analyzers                 → TEAM001, TEAM002 rule enforcement       │
│  Testing Utilities         → OrderBuilder, FakeEmailService         │
│  Configuration Helpers     → Strongly-typed config, vault client    │
└─────────────────────────────────────────────────────────────────────┘

Open-Source Use Cases:
┌─────────────────────────────────────────────────────────────────────┐
│  Framework extensions     → IHostedService helpers                  │
│  Utility libraries        → FluentValidation rules, EF extensions   │
│  Source generators        → Boilerplate elimination                 │
└─────────────────────────────────────────────────────────────────────┘

NuGet Registry Options:
• nuget.org              — public packages
• GitHub Packages        — private/org packages, auth via PAT
• Azure Artifacts        — enterprise, per-feed RBAC
• MyGet                  — flexible hosting
• Hosted BaGet (self)    — full control, free
```

---

## Step 1830: Library Project Structure

```
MyCompany.SharedKernel/                     ← Solution root
├── src/
│   ├── MyCompany.SharedKernel/             ← Main library
│   │   ├── MyCompany.SharedKernel.csproj
│   │   ├── Domain/
│   │   │   ├── ValueObject.cs
│   │   │   ├── Entity.cs
│   │   │   ├── AggregateRoot.cs
│   │   │   └── Money.cs
│   │   └── Extensions/
│   │       └── ServiceCollectionExtensions.cs
│   ├── MyCompany.SharedKernel.AspNetCore/  ← ASP.NET Core integrations
│   │   └── Middleware/
│   └── MyCompany.SharedKernel.Analyzers/  ← Roslyn analyzers
│       └── Rules/
├── tests/
│   ├── MyCompany.SharedKernel.Tests/
│   └── MyCompany.SharedKernel.Analyzers.Tests/
├── samples/
│   └── SampleApp/
├── Directory.Build.props                   ← Shared MSBuild properties
├── Directory.Packages.props                ← Central package management
└── README.md
```

```xml
<!-- Directory.Build.props — shared across all projects -->
<Project>
  <PropertyGroup>
    <!-- Package metadata -->
    <Authors>My Company</Authors>
    <Company>My Company</Company>
    <Copyright>Copyright © $([System.DateTime]::Now.Year) My Company</Copyright>
    <PackageLicenseExpression>MIT</PackageLicenseExpression>
    <PackageProjectUrl>https://github.com/mycompany/sharedkernel</PackageProjectUrl>
    <RepositoryUrl>https://github.com/mycompany/sharedkernel</RepositoryUrl>
    <RepositoryType>git</RepositoryType>
    <PackageIcon>icon.png</PackageIcon>
    <PackageReadmeFile>README.md</PackageReadmeFile>

    <!-- Build settings -->
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <LangVersion>latest</LangVersion>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <GenerateDocumentationFile>true</GenerateDocumentationFile>

    <!-- Source Link — maps PDB symbols to GitHub source -->
    <PublishRepositoryUrl>true</PublishRepositoryUrl>
    <EmbedUntrackedSources>true</EmbedUntrackedSources>
    <IncludeSymbols>true</IncludeSymbols>
    <SymbolPackageFormat>snupkg</SymbolPackageFormat>

    <!-- Deterministic builds (reproducible) -->
    <Deterministic>true</Deterministic>
    <ContinuousIntegrationBuild Condition="'$(CI)' == 'true'">true</ContinuousIntegrationBuild>
  </PropertyGroup>
</Project>
```

---

## Step 1831: Library Project File — Best Practices

```xml
<!-- src/MyCompany.SharedKernel/MyCompany.SharedKernel.csproj -->
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFrameworks>net8.0;net9.0</TargetFrameworks>  <!-- Multi-target -->
    <PackageId>MyCompany.SharedKernel</PackageId>
    <Version>1.2.0</Version>   <!-- Managed by MinVer / Nerdbank.GitVersioning -->
    <Description>Domain primitives and shared kernel for MyCompany services. Includes ValueObject, Entity, AggregateRoot base classes, Money value object, and common extensions.</Description>
    <PackageTags>ddd;domain;value-objects;aggregates;csharp;dotnet</PackageTags>
  </PropertyGroup>

  <!-- Include README and icon in package -->
  <ItemGroup>
    <None Include="../../README.md" Pack="true" PackagePath="/" />
    <None Include="../../icon.png" Pack="true" PackagePath="/" />
  </ItemGroup>

  <!-- Source Link for debugging into library source -->
  <ItemGroup>
    <PackageReference Include="Microsoft.SourceLink.GitHub" Version="8.0.0"
                      PrivateAssets="All" />
  </ItemGroup>

  <!-- Don't include test packages in the published package -->
  <ItemGroup Condition="'$(IsTestProject)' != 'true'">
    <PackageReference Include="MinVer" Version="5.0.0" PrivateAssets="all" />
  </ItemGroup>

</Project>
```

```xml
<!-- Directory.Packages.props — Central Package Management -->
<!-- Prevents version drift across projects -->
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>

  <ItemGroup>
    <PackageVersion Include="Microsoft.SourceLink.GitHub" Version="8.0.0" />
    <PackageVersion Include="MinVer" Version="5.0.0" />
    <PackageVersion Include="Microsoft.Extensions.DependencyInjection.Abstractions" Version="9.0.0" />
    <PackageVersion Include="FluentValidation" Version="11.9.2" />
    <PackageVersion Include="xunit" Version="2.9.3" />
    <PackageVersion Include="FluentAssertions" Version="7.0.0" />
  </ItemGroup>
</Project>
```

---

## Step 1832: Semantic Versioning with MinVer

```bash
# MinVer uses git tags for semantic versioning
# Tag format: 1.2.3 or v1.2.3
# On commits between tags, adds height: 1.2.3-alpha.0.5

# Create a release tag
git tag -a v1.2.0 -m "Release 1.2.0: Add Money.Multiply support"
git push origin v1.2.0

# MinVer auto-reads this — no version property needed in csproj
# <Version> from MinVer: 1.2.0 on tagged commit
# <Version> from MinVer: 1.2.0-alpha.0.3 on 3 commits after 1.2.0

# Alpha/Beta releases
git tag -a v2.0.0-alpha.1 -m "Alpha release"
git push origin v2.0.0-alpha.1
```

```xml
<!-- MinVer configuration in csproj -->
<PropertyGroup>
  <MinVerTagPrefix>v</MinVerTagPrefix>          <!-- Tags start with 'v' -->
  <MinVerDefaultPreReleaseIdentifiers>preview</MinVerDefaultPreReleaseIdentifiers>
  <MinVerMinimumMajorMinor>1.0</MinVerMinimumMajorMinor>
  <MinVerVerbosity>Normal</MinVerVerbosity>
</PropertyGroup>
```

---

## Step 1833: Strong-Named Assemblies and Signing

```xml
<!-- Signing with a .snk key (optional but required for GAC, some enterprise scenarios) -->
<PropertyGroup>
  <SignAssembly>true</SignAssembly>
  <AssemblyOriginatorKeyFile>../../MyCompany.SharedKernel.snk</AssemblyOriginatorKeyFile>
  <DelaySign>false</DelaySign>
</PropertyGroup>
```

```bash
# Generate a strong name key
sn -k MyCompany.SharedKernel.snk

# For CI — store as a secret, not in the repo
# .github/workflows/publish.yml:
# env:
#   SIGNING_KEY: ${{ secrets.SIGNING_KEY }}
# Then in build step:
# echo "$SIGNING_KEY" | base64 -d > MyCompany.SharedKernel.snk
```

---

## Step 1834: Source Generator as NuGet Package

```xml
<!-- src/MyCompany.SharedKernel.Generators/MyCompany.SharedKernel.Generators.csproj -->
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>  <!-- Analyzers MUST target netstandard2.0 -->
    <PackageId>MyCompany.SharedKernel.Generators</PackageId>
    <Description>Source generators for MyCompany.SharedKernel. Auto-generates INotifyPropertyChanged, ToString, and builder patterns.</Description>

    <!-- Critical: mark this as an analyzer package -->
    <IncludeBuildOutput>false</IncludeBuildOutput>  <!-- Don't add to refs -->
    <DevelopmentDependency>true</DevelopmentDependency>  <!-- Not a runtime dep -->
    <SuppressDependenciesWhenPacking>true</SuppressDependenciesWhenPacking>

    <!-- Roslyn's analyzer infrastructure requires these to be locked -->
    <EnforceExtendedAnalyzerRules>true</EnforceExtendedAnalyzerRules>
    <IsRoslynComponent>true</IsRoslynComponent>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.CodeAnalysis.CSharp" Version="4.8.0" PrivateAssets="all" />
    <PackageReference Include="Microsoft.CodeAnalysis.Analyzers" Version="3.3.4" PrivateAssets="all" />
  </ItemGroup>

  <!-- Pack the generator DLL into the analyzers/dotnet/cs path -->
  <ItemGroup>
    <None Include="$(OutputPath)\$(AssemblyName).dll"
          Pack="true"
          PackagePath="analyzers/dotnet/cs"
          Visible="false" />
  </ItemGroup>

</Project>
```

```xml
<!-- Consumer project — references the generator package -->
<ItemGroup>
  <PackageReference Include="MyCompany.SharedKernel.Generators" Version="1.2.0">
    <PrivateAssets>all</PrivateAssets>  <!-- Design-time only, not shipped with app -->
    <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
  </PackageReference>
</ItemGroup>
```

---

## Step 1835: Publishing to NuGet.org

```bash
# Pack the library
dotnet pack src/MyCompany.SharedKernel \
  --configuration Release \
  --output ./artifacts/

# Verify the package contents
dotnet tool install -g nuget-package-explorer  # or use online: https://nuget.info/
# Check: metadata, dependencies, lib/ folders, analyzers/ folder, icon, README

# Publish to NuGet.org
dotnet nuget push artifacts/MyCompany.SharedKernel.1.2.0.nupkg \
  --api-key $NUGET_API_KEY \
  --source https://api.nuget.org/v3/index.json \
  --skip-duplicate  # Don't fail if version already exists

# Publish symbol package separately
dotnet nuget push artifacts/MyCompany.SharedKernel.1.2.0.snupkg \
  --api-key $NUGET_API_KEY \
  --source https://api.nuget.org/v3/index.json
```

---

## Step 1836: GitHub Actions CI/CD for NuGet

```yaml
# .github/workflows/publish-nuget.yml
name: Publish NuGet Package

on:
  push:
    tags:
      - 'v*.*.*'   # Trigger on version tags: v1.2.0, v2.0.0-alpha.1

env:
  DOTNET_SKIP_FIRST_TIME_EXPERIENCE: 1
  DOTNET_CLI_TELEMETRY_OPTOUT: 1

jobs:
  validate:
    name: Build & Test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full history for MinVer

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: |
            8.0.x
            9.0.x

      - name: Restore
        run: dotnet restore

      - name: Build
        run: dotnet build --no-restore --configuration Release

      - name: Test
        run: dotnet test --no-build --configuration Release --logger "trx" --collect:"XPlat Code Coverage"

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: "**/*.trx"

  pack-and-publish:
    name: Pack & Publish
    runs-on: ubuntu-latest
    needs: validate
    permissions:
      contents: write    # For creating GitHub releases
      packages: write    # For GitHub Packages

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '9.0.x'

      - name: Pack
        run: |
          dotnet pack src/MyCompany.SharedKernel \
            --configuration Release \
            --output ./artifacts/ \
            -p:ContinuousIntegrationBuild=true

          dotnet pack src/MyCompany.SharedKernel.AspNetCore \
            --configuration Release \
            --output ./artifacts/ \
            -p:ContinuousIntegrationBuild=true

          dotnet pack src/MyCompany.SharedKernel.Generators \
            --configuration Release \
            --output ./artifacts/ \
            -p:ContinuousIntegrationBuild=true

      - name: Verify packages
        run: |
          dotnet tool install -g dotnet-validate
          for f in ./artifacts/*.nupkg; do
            dotnet validate package local "$f"
          done

      - name: Publish to NuGet.org
        run: |
          dotnet nuget push ./artifacts/*.nupkg \
            --api-key ${{ secrets.NUGET_API_KEY }} \
            --source https://api.nuget.org/v3/index.json \
            --skip-duplicate

          dotnet nuget push ./artifacts/*.snupkg \
            --api-key ${{ secrets.NUGET_API_KEY }} \
            --source https://api.nuget.org/v3/index.json \
            --skip-duplicate

      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          files: ./artifacts/*
          generate_release_notes: true
          draft: false
          prerelease: ${{ contains(github.ref, '-alpha') || contains(github.ref, '-beta') || contains(github.ref, '-preview') }}
```

---

## Step 1837: Private NuGet Feed (GitHub Packages)

```bash
# Add GitHub Packages as NuGet source
dotnet nuget add source \
  --username YOUR_GITHUB_USERNAME \
  --password $GITHUB_TOKEN \
  --store-password-in-clear-text \
  --name github \
  "https://nuget.pkg.github.com/OWNER/index.json"

# Publish to GitHub Packages
dotnet nuget push ./artifacts/MyCompany.SharedKernel.1.2.0.nupkg \
  --api-key $GITHUB_TOKEN \
  --source "github"
```

```xml
<!-- NuGet.config — check into repo -->
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" protocolVersion="3" />
    <add key="github" value="https://nuget.pkg.github.com/OWNER/index.json" />
  </packageSources>
  <packageSourceCredentials>
    <github>
      <add key="Username" value="OWNER" />
      <add key="ClearTextPassword" value="%GITHUB_TOKEN%" />
    </github>
  </packageSourceCredentials>
  <!-- Enable audit for security vulnerabilities -->
  <config>
    <add key="auditLevel" value="moderate" />
  </config>
</configuration>
```

---

## Step 1838: Package Governance — SBOM and Vulnerability Scanning

```bash
# Generate Software Bill of Materials (SBOM)
dotnet tool install --global Microsoft.Sbom.DotNetTool
sbom-tool generate \
  -b ./artifacts \
  -bc ./src \
  -pn MyCompany.SharedKernel \
  -pv 1.2.0 \
  -ps MyCompany \
  -nsb https://mycompany.com/sbom

# NuGet audit (built into dotnet restore since .NET 8)
dotnet restore --audit
# Output: Package 'SomePackage' 2.0.0 has a known vulnerability severity High
# CVE-2024-XXXXX: ...

# Lock files — pin exact transitive versions
dotnet restore --use-lock-file
# Generates packages.lock.json — check into version control
# CI validation:
dotnet restore --locked-mode  # Fails if lock file is out of date
```

```xml
<!-- Enable NuGet audit in all projects -->
<PropertyGroup>
  <NuGetAudit>true</NuGetAudit>
  <NuGetAuditLevel>moderate</NuGetAuditLevel>
  <NuGetAuditMode>all</NuGetAuditMode>  <!-- audit transitive deps too -->
</PropertyGroup>

<!-- Restore lock files to prevent supply chain attacks -->
<PropertyGroup>
  <RestorePackagesWithLockFile>true</RestorePackagesWithLockFile>
  <!-- In CI: dotnet restore --locked-mode -->
</PropertyGroup>
```

---

## Step 1839: Package Compatibility and Multi-Targeting

```xml
<!-- Multi-target for broad compatibility -->
<PropertyGroup>
  <TargetFrameworks>net8.0;net9.0;netstandard2.1</TargetFrameworks>
</PropertyGroup>

<!-- Conditional compilation per target -->
<ItemGroup Condition="'$(TargetFramework)' == 'net9.0'">
  <PackageReference Include="System.Threading.Channels" Version="9.0.0" />
</ItemGroup>
<ItemGroup Condition="'$(TargetFramework)' != 'net9.0'">
  <PackageReference Include="System.Threading.Channels" Version="8.0.0" />
</ItemGroup>
```

```csharp
// Use #if for target-specific code
public sealed class HighPerformanceDictionary<TKey, TValue>
    where TKey : notnull
{
#if NET9_0_OR_GREATER
    private readonly System.Collections.Frozen.FrozenDictionary<TKey, TValue> _inner;

    public HighPerformanceDictionary(IDictionary<TKey, TValue> source)
        => _inner = source.ToFrozenDictionary();

    public bool TryGetValue(TKey key, out TValue? value)
        => _inner.TryGetValue(key, out value);
#else
    private readonly Dictionary<TKey, TValue> _inner;

    public HighPerformanceDictionary(IDictionary<TKey, TValue> source)
        => _inner = new Dictionary<TKey, TValue>(source);

    public bool TryGetValue(TKey key, out TValue? value)
        => _inner.TryGetValue(key, out value);
#endif
}
```

---

## Step 1840: Documentation and README for NuGet

```markdown
<!-- README.md — shown on NuGet.org package page -->
# MyCompany.SharedKernel

Domain primitives and shared kernel for .NET 8/9 services.

## Installation

```bash
dotnet add package MyCompany.SharedKernel
```

## Quick Start

```csharp
// Value Objects with structural equality
var price = Money.Of(29.99m, "USD");
var total = price.Multiply(3); // Money.Of(89.97, "USD")

// Address value object
var address = Address.Create("123 Main St", "Springfield", "62701", "US");

// Aggregate with domain events
public class Order : AggregateRoot<OrderId>
{
    public void Submit()
    {
        // Validates invariants, then raises event
        Status = OrderStatus.Submitted;
        RaiseDomainEvent(new OrderSubmittedEvent(Id));
    }
}
```

## Packages

| Package | Description |
|---|---|
| `MyCompany.SharedKernel` | Core domain primitives |
| `MyCompany.SharedKernel.AspNetCore` | Middleware, filters, extensions |
| `MyCompany.SharedKernel.Generators` | Source generators |

## Versioning

Follows [Semantic Versioning](https://semver.org/).

## License

MIT
```

---

## Step 1841: Package Deprecation and Migration

```bash
# Deprecate an old package on NuGet.org (use the web UI or API)
# NuGet.org → manage package → deprecate

# Or list a replacement package in the deprecated package's metadata
```

```xml
<!-- Alert consumers that this package is deprecated -->
<!-- In the new version of the .csproj or a .targets file -->
<PropertyGroup>
  <Deprecated>true</Deprecated>
  <DeprecationMessage>Use MyCompany.NewSharedKernel instead. Migration guide: https://...</DeprecationMessage>
  <AlternativePackageId>MyCompany.NewSharedKernel</AlternativePackageId>
  <AlternativePackageVersion>[2.0.0, )</AlternativePackageVersion>
</PropertyGroup>
```

---

## Step 1842: BaGet — Self-Hosted NuGet Server

```yaml
# docker-compose.yml — self-hosted NuGet with BaGet
services:
  baget:
    image: loicsharma/baget:latest
    environment:
      - ApiKey=your-secure-api-key
      - Storage__Type=FileSystem
      - Storage__Path=/var/baget/packages
      - Database__Type=Sqlite
      - Database__ConnectionString=Data Source=/var/baget/baget.db
      - Mirror__Enabled=true
      - Mirror__PackageSource=https://api.nuget.org/v3/index.json
    ports:
      - "5555:80"
    volumes:
      - baget-data:/var/baget

volumes:
  baget-data:
```

```bash
# Point NuGet to your BaGet instance
dotnet nuget add source http://localhost:5555/v3/index.json \
  --name baget

# Push to BaGet
dotnet nuget push ./artifacts/MyCompany.SharedKernel.1.2.0.nupkg \
  --api-key your-secure-api-key \
  --source baget
```

---

## Step 1843: Analyzer Package with Code Fixes

```csharp
// src/MyCompany.SharedKernel.Analyzers/Rules/UseValueObjectRule.cs
[DiagnosticAnalyzer(LanguageNames.CSharp)]
public sealed class UseValueObjectAnalyzer : DiagnosticAnalyzer
{
    public static readonly DiagnosticDescriptor Rule = new(
        id: "SHARED001",
        title: "Use Money value object instead of decimal for monetary values",
        messageFormat: "Property '{0}' appears to hold a monetary amount. Use Money instead of decimal.",
        category: "Design",
        defaultSeverity: DiagnosticSeverity.Warning,
        isEnabledByDefault: true,
        description: "Monetary amounts should be encapsulated in the Money value object to ensure currency safety.",
        helpLinkUri: "https://docs.mycompany.com/SHARED001");

    public override ImmutableArray<DiagnosticDescriptor> SupportedDiagnostics =>
        ImmutableArray.Create(Rule);

    public override void Initialize(AnalysisContext context)
    {
        context.ConfigureGeneratedCodeAnalysis(GeneratedCodeAnalysisFlags.None);
        context.EnableConcurrentExecution();
        context.RegisterSymbolAction(AnalyzeProperty, SymbolKind.Property);
    }

    private static void AnalyzeProperty(SymbolAnalysisContext context)
    {
        var property = (IPropertySymbol)context.Symbol;

        if (property.Type.SpecialType != SpecialType.System_Decimal) return;

        var name = property.Name.ToLowerInvariant();
        if (!name.Contains("price") && !name.Contains("amount") &&
            !name.Contains("cost") && !name.Contains("fee") &&
            !name.Contains("total")) return;

        // Check if the containing type is already a Money class
        if (property.ContainingType.Name == "Money") return;

        var diagnostic = Diagnostic.Create(
            Rule,
            property.Locations[0],
            property.Name);

        context.ReportDiagnostic(diagnostic);
    }
}

// Code fix for the analyzer
[ExportCodeFixProvider(LanguageNames.CSharp, Name = nameof(UseValueObjectCodeFix))]
[Shared]
public sealed class UseValueObjectCodeFix : CodeFixProvider
{
    public override ImmutableArray<string> FixableDiagnosticIds =>
        ImmutableArray.Create(UseValueObjectAnalyzer.Rule.Id);

    public override FixAllProvider GetFixAllProvider() =>
        WellKnownFixAllProviders.BatchFixer;

    public override async Task RegisterCodeFixesAsync(CodeFixContext context)
    {
        var root = await context.Document.GetSyntaxRootAsync(context.CancellationToken);
        var diagnostic = context.Diagnostics.First();
        var node = root!.FindNode(diagnostic.Location.SourceSpan);

        context.RegisterCodeFix(
            CodeAction.Create(
                title: "Change to Money type",
                createChangedDocument: ct =>
                    ChangeToMoneyTypeAsync(context.Document, node, ct),
                equivalenceKey: "ChangeToMoneyType"),
            diagnostic);
    }

    private static async Task<Document> ChangeToMoneyTypeAsync(
        Document document,
        SyntaxNode node,
        CancellationToken ct)
    {
        var root = await document.GetSyntaxRootAsync(ct);
        // Find the property declaration and replace `decimal` with `Money`
        var oldProperty = node.AncestorsAndSelf().OfType<PropertyDeclarationSyntax>().FirstOrDefault();
        if (oldProperty is null) return document;

        var newType = SyntaxFactory.ParseTypeName("Money")
            .WithTriviaFrom(oldProperty.Type);

        var newProperty = oldProperty.WithType(newType);
        var newRoot = root!.ReplaceNode(oldProperty, newProperty);
        return document.WithSyntaxRoot(newRoot);
    }
}
```

---

## Step 1844: Package Authoring Checklist

```
Pre-Publishing Checklist:
┌─────────────────────────────────────────────────────────────────────┐
│  ✅ PackageId is unique and follows convention (Company.Feature)    │
│  ✅ Version follows SemVer (MinVer from git tags)                   │
│  ✅ Description is clear and accurate (shown on NuGet.org)          │
│  ✅ PackageTags help discoverability                                 │
│  ✅ PackageProjectUrl links to GitHub repo                          │
│  ✅ PackageLicenseExpression set (MIT, Apache-2.0, etc.)            │
│  ✅ README.md included in package                                   │
│  ✅ Icon.png included (128x128 png)                                 │
│  ✅ RepositoryUrl and RepositoryType set                            │
│  ✅ Source Link configured (debug into library source)              │
│  ✅ Symbol package (.snupkg) published                              │
│  ✅ Multi-targeted for all supported frameworks                     │
│  ✅ No private implementation details in public API (InternalsVisibleTo for tests)
│  ✅ XML documentation generated (GenerateDocumentationFile=true)    │
│  ✅ Breaking changes documented (CHANGELOG.md)                      │
│  ✅ NuGet audit passes (no high/critical vulnerabilities)           │
│  ✅ Deterministic build (ContinuousIntegrationBuild=true in CI)     │
│  ✅ Package validated locally before push                           │
└─────────────────────────────────────────────────────────────────────┘

API Design Rules for Libraries:
• Seal classes that shouldn't be subclassed
• Make constructors internal, expose factory methods
• Use interfaces for DI registration
• Never change public API signatures in minor/patch versions
• Add [Obsolete] before removing — warn for 2+ minor versions
• Document all public members with XML
• Test all public APIs — tests ARE the contract
```

---

## Summary: NuGet Package Publishing Workflow

```
Development → Local Test → CI Build → Pack → Publish

1. git tag -a v1.2.0 -m "Release 1.2.0"
2. GitHub Actions triggers on the tag
3. dotnet test → dotnet pack → dotnet validate package
4. dotnet nuget push → nuget.org (public) or GitHub Packages (private)
5. GitHub Release created automatically with changelog

Package types:
• Runtime package     → lib/net8.0/*.dll + dependencies
• Analyzer package    → analyzers/dotnet/cs/*.dll (PrivateAssets=all)
• Source generator    → analyzers/dotnet/cs/*.dll (PrivateAssets=all)
• Multi-targeting     → lib/net8.0/ + lib/net9.0/ + lib/netstandard2.1/
• Symbol package      → .snupkg (PDB symbols for debugging)
```

---

*Next: Part 76 — Background Workers & Scheduled Jobs: IHostedService, Worker Service, Hangfire & Quartz.NET*
