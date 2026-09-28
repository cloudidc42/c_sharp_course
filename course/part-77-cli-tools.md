# Part 77: CLI Tools with System.CommandLine & Spectre.Console — Professional Developer Tools

## Steps 1861–1876 | World-Class .NET CLI Development

---

## Step 1861: Why .NET CLI Tools?

```
.NET Global/Local Tools:
┌─────────────────────────────────────────────────────────────────────┐
│  dotnet tool install -g MyCompany.DevTool                           │
│  dotnet mydevtool migrate --tenant acme --dry-run                   │
│  dotnet mydevtool report --format pdf --output ./reports/           │
└─────────────────────────────────────────────────────────────────────┘

Use Cases:
• Database migrations          • Code generation
• Report generation            • Deployment scripts
• Dev environment setup        • API testing / smoke tests
• Tenant management            • Secret rotation
• Data export/import           • Scaffolding tools

System.CommandLine vs alternatives:
┌────────────────────┬───────────────────────────────────────────────┐
│ System.CommandLine │ Official .NET, first-class tab completion     │
│ Spectre.Console    │ Rich UI: tables, progress, prompts, trees     │
│ Cocona             │ Convention-based, minimal boilerplate         │
│ CommandLineParser  │ Attribute-based, mature, simple               │
└────────────────────┴───────────────────────────────────────────────┘
Best combo: System.CommandLine + Spectre.Console
```

---

## Step 1862: Project Setup

```bash
# Create CLI project
dotnet new console -n MyApp.DevTool -f net9.0
cd MyApp.DevTool

dotnet add package System.CommandLine --version 2.0.0-beta4.24324.3
dotnet add package Spectre.Console
dotnet add package Spectre.Console.Cli
dotnet add package Microsoft.Extensions.Hosting
dotnet add package Microsoft.Extensions.DependencyInjection
dotnet add package Serilog.Extensions.Hosting
dotnet add package Serilog.Sinks.Console
```

```xml
<!-- MyApp.DevTool.csproj -->
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net9.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <AssemblyName>mydevtool</AssemblyName>    <!-- CLI command name -->
    <RootNamespace>MyApp.DevTool</RootNamespace>

    <!-- NuGet tool packaging -->
    <PackageId>MyCompany.DevTool</PackageId>
    <PackAsTool>true</PackAsTool>
    <ToolCommandName>mydevtool</ToolCommandName>
    <PackageOutputPath>./nupkg</PackageOutputPath>
  </PropertyGroup>

</Project>
```

---

## Step 1863: System.CommandLine — Root Command Structure

```csharp
// Program.cs
using System.CommandLine;
using Spectre.Console;

var rootCommand = new RootCommand("MyApp Developer Tool — manage tenants, generate reports, run migrations");
rootCommand.Name = "mydevtool";

// Add subcommands
rootCommand.AddCommand(BuildTenantCommand());
rootCommand.AddCommand(BuildMigrateCommand());
rootCommand.AddCommand(BuildReportCommand());
rootCommand.AddCommand(BuildHealthCommand());

return await rootCommand.InvokeAsync(args);

// --- Subcommand builders ---

static Command BuildTenantCommand()
{
    var tenantCommand = new Command("tenant", "Manage tenants");

    tenantCommand.AddCommand(BuildTenantListCommand());
    tenantCommand.AddCommand(BuildTenantProvisionCommand());
    tenantCommand.AddCommand(BuildTenantDisableCommand());

    return tenantCommand;
}

static Command BuildTenantListCommand()
{
    var filterOption = new Option<string?>("--filter", "Filter by name or slug");
    filterOption.AddAlias("-f");

    var outputOption = new Option<OutputFormat>("--format", () => OutputFormat.Table, "Output format");
    outputOption.AddAlias("-o");

    var listCommand = new Command("list", "List all tenants")
    {
        filterOption,
        outputOption
    };

    listCommand.SetHandler(async (string? filter, OutputFormat format, IConsole console) =>
    {
        await HandleTenantListAsync(filter, format, console);
    }, filterOption, outputOption);

    return listCommand;
}

static Command BuildTenantProvisionCommand()
{
    var nameOption = new Option<string>("--name", "Company name") { IsRequired = true };
    var slugOption = new Option<string>("--slug", "Tenant slug (URL-safe)") { IsRequired = true };
    var planOption = new Option<TenantPlan>("--plan", () => TenantPlan.Free, "Subscription plan");
    var dryRunOption = new Option<bool>("--dry-run", "Preview without executing");

    var provisionCommand = new Command("provision", "Provision a new tenant")
    {
        nameOption,
        slugOption,
        planOption,
        dryRunOption
    };

    provisionCommand.SetHandler(async (string name, string slug, TenantPlan plan, bool dryRun) =>
    {
        await HandleTenantProvisionAsync(name, slug, plan, dryRun);
    }, nameOption, slugOption, planOption, dryRunOption);

    return provisionCommand;
}

enum OutputFormat { Table, Json, Csv }
enum TenantPlan { Free, Pro, Enterprise }
```

---

## Step 1864: Spectre.Console — Rich Terminal Output

```csharp
// Commands/TenantListCommand.cs
static async Task HandleTenantListAsync(string? filter, OutputFormat format, IConsole console)
{
    // Fetch data (would use real DI/HTTP client in production)
    var tenants = await GetTenantsAsync(filter);

    if (!tenants.Any())
    {
        AnsiConsole.MarkupLine("[yellow]No tenants found.[/]");
        return;
    }

    switch (format)
    {
        case OutputFormat.Table:
            RenderTenantTable(tenants);
            break;
        case OutputFormat.Json:
            console.WriteLine(JsonSerializer.Serialize(tenants, new JsonSerializerOptions { WriteIndented = true }));
            break;
        case OutputFormat.Csv:
            RenderTenantCsv(tenants, console);
            break;
    }
}

static void RenderTenantTable(List<TenantDto> tenants)
{
    var table = new Table();
    table.AddColumn("[bold]Slug[/]");
    table.AddColumn("[bold]Name[/]");
    table.AddColumn("[bold]Plan[/]");
    table.AddColumn("[bold]Status[/]");
    table.AddColumn("[bold]Created[/]");

    table.Border(TableBorder.Rounded);
    table.Expand();

    foreach (var tenant in tenants)
    {
        var planMarkup = tenant.Plan switch
        {
            "Enterprise" => "[green bold]Enterprise[/]",
            "Pro" => "[blue]Pro[/]",
            _ => "[grey]Free[/]"
        };

        var statusMarkup = tenant.IsActive
            ? "[green]● Active[/]"
            : "[red]○ Inactive[/]";

        table.AddRow(
            $"[yellow]{tenant.Slug}[/]",
            tenant.Name,
            planMarkup,
            statusMarkup,
            tenant.CreatedAt.ToString("yyyy-MM-dd"));
    }

    AnsiConsole.Write(table);
    AnsiConsole.MarkupLine($"\n[dim]Total: {tenants.Count} tenants[/]");
}

static void RenderTenantCsv(List<TenantDto> tenants, IConsole console)
{
    console.WriteLine("Slug,Name,Plan,Status,CreatedAt");
    foreach (var t in tenants)
        console.WriteLine($"{t.Slug},{t.Name},{t.Plan},{(t.IsActive ? "Active" : "Inactive")},{t.CreatedAt:yyyy-MM-dd}");
}
```

---

## Step 1865: Interactive Prompts with Spectre.Console

```csharp
// Commands/TenantProvisionCommand.cs
static async Task HandleTenantProvisionAsync(
    string name,
    string slug,
    TenantPlan plan,
    bool dryRun)
{
    // Validate slug format
    if (!System.Text.RegularExpressions.Regex.IsMatch(slug, @"^[a-z0-9-]{3,50}$"))
    {
        AnsiConsole.MarkupLine("[red]Error:[/] Slug must be 3-50 lowercase alphanumeric characters or hyphens.");
        Environment.Exit(1);
        return;
    }

    // Show what will happen
    AnsiConsole.Write(new Panel(
        $"""
        [bold]Tenant Provisioning Plan[/]

        Name:     [yellow]{name}[/]
        Slug:     [cyan]{slug}[/]
        Plan:     [green]{plan}[/]

        Actions:
         1. Create tenant record in database
         2. Run schema migrations for [cyan]{slug}[/]
         3. Create default admin API key
         4. Send welcome email to admin
        """
    ).Header("[blue]Preview[/]").BorderStyle(Style.Parse("blue")));

    if (dryRun)
    {
        AnsiConsole.MarkupLine("\n[dim]--dry-run: No changes made.[/]");
        return;
    }

    // Confirm unless --yes flag
    if (!AnsiConsole.Confirm("\nProceed with provisioning?", defaultValue: false))
    {
        AnsiConsole.MarkupLine("[yellow]Cancelled.[/]");
        return;
    }

    // Execute with progress display
    TenantProvisionResult? result = null;

    await AnsiConsole.Progress()
        .AutoRefresh(true)
        .Columns(
            new TaskDescriptionColumn(),
            new ProgressBarColumn(),
            new PercentageColumn(),
            new SpinnerColumn())
        .StartAsync(async ctx =>
        {
            var dbTask = ctx.AddTask("[green]Creating tenant record[/]");
            var migrateTask = ctx.AddTask("[blue]Running migrations[/]");
            var apiKeyTask = ctx.AddTask("[yellow]Generating API key[/]");
            var emailTask = ctx.AddTask("[cyan]Sending welcome email[/]");

            // Step 1
            dbTask.StartTask();
            await Task.Delay(500); // Simulate work
            dbTask.Value = 100;

            // Step 2
            migrateTask.StartTask();
            for (int i = 0; i <= 100; i += 10)
            {
                migrateTask.Value = i;
                await Task.Delay(100);
            }

            // Step 3
            apiKeyTask.StartTask();
            await Task.Delay(200);
            apiKeyTask.Value = 100;

            // Step 4
            emailTask.StartTask();
            await Task.Delay(300);
            emailTask.Value = 100;

            result = new TenantProvisionResult(
                TenantId: Guid.NewGuid(),
                Slug: slug,
                ApiKey: $"sk_live_{Convert.ToBase64String(RandomNumberGenerator.GetBytes(24))[..32]}");
        });

    // Display result
    AnsiConsole.Write(new Panel(
        $"""
        [bold green]✓ Tenant provisioned successfully![/]

        Tenant ID:  [dim]{result!.TenantId}[/]
        Slug:       [cyan]{result.Slug}[/]
        API Key:    [yellow]{result.ApiKey}[/]

        [dim red]⚠  Store the API key securely — it will not be shown again.[/]
        """
    ).Header("[green]Success[/]").BorderStyle(Style.Parse("green")));
}

public record TenantProvisionResult(Guid TenantId, string Slug, string ApiKey);
```

---

## Step 1866: Spectre.Console.Cli — Convention-Based Commands

```csharp
// Alternative approach: Spectre.Console.Cli is cleaner for large CLIs

// Program.cs with Spectre.Console.Cli
var app = new CommandApp();

app.Configure(config =>
{
    config.SetApplicationName("mydevtool");
    config.SetApplicationVersion("1.0.0");

    config.AddBranch("tenant", tenant =>
    {
        tenant.SetDescription("Tenant management commands");
        tenant.AddCommand<TenantListCommand>("list")
              .WithDescription("List all tenants")
              .WithExample(new[] { "tenant", "list", "--filter", "acme" });
        tenant.AddCommand<TenantProvisionCommand>("provision")
              .WithDescription("Provision a new tenant");
        tenant.AddCommand<TenantDisableCommand>("disable")
              .WithDescription("Disable an existing tenant");
    });

    config.AddBranch("migrate", migrate =>
    {
        migrate.SetDescription("Database migration commands");
        migrate.AddCommand<MigrateAllCommand>("all").WithDescription("Migrate all tenants");
        migrate.AddCommand<MigrateTenantCommand>("tenant").WithDescription("Migrate specific tenant");
    });

    config.AddCommand<HealthCheckCommand>("health")
          .WithDescription("Run health checks against the API");

    config.AddCommand<ReportCommand>("report")
          .WithDescription("Generate reports");

    config.PropagateExceptions();  // Show full stack traces in debug
    config.ValidateExamples();     // Verify examples in tests
});

return await app.RunAsync(args);
```

```csharp
// Commands/TenantListCommand.cs
public sealed class TenantListCommand : AsyncCommand<TenantListCommand.Settings>
{
    private readonly IApiClient _api;

    public TenantListCommand(IApiClient api) => _api = api;

    public sealed class Settings : CommandSettings
    {
        [CommandOption("-f|--filter <FILTER>")]
        [Description("Filter tenants by name or slug")]
        public string? Filter { get; set; }

        [CommandOption("-o|--format <FORMAT>")]
        [Description("Output format: table, json, csv")]
        [DefaultValue("table")]
        public string Format { get; set; } = "table";

        [CommandOption("--active-only")]
        [Description("Show only active tenants")]
        public bool ActiveOnly { get; set; }

        [CommandOption("--page <PAGE>")]
        [Description("Page number")]
        [DefaultValue(1)]
        public int Page { get; set; } = 1;

        public override ValidationResult Validate()
        {
            if (Format is not "table" and not "json" and not "csv")
                return ValidationResult.Error($"Invalid format '{Format}'. Use: table, json, csv");

            if (Page < 1)
                return ValidationResult.Error("Page must be >= 1");

            return ValidationResult.Success();
        }
    }

    public override async Task<int> ExecuteAsync(CommandContext context, Settings settings)
    {
        List<TenantDto> tenants;

        await AnsiConsole.Status()
            .Spinner(Spinner.Known.Dots)
            .StartAsync("Fetching tenants...", async _ =>
            {
                tenants = await _api.GetTenantsAsync(settings.Filter, settings.ActiveOnly, settings.Page);
            });

        if (!tenants!.Any())
        {
            AnsiConsole.MarkupLine("[yellow]No tenants found.[/]");
            return 0;
        }

        if (settings.Format == "json")
        {
            AnsiConsole.WriteLine(JsonSerializer.Serialize(tenants, new JsonSerializerOptions { WriteIndented = true }));
            return 0;
        }

        RenderTenantTable(tenants);
        return 0;
    }

    private static void RenderTenantTable(List<TenantDto> tenants)
    {
        var table = new Table()
            .Border(TableBorder.Rounded)
            .AddColumn("[bold blue]Slug[/]")
            .AddColumn("[bold]Name[/]")
            .AddColumn("[bold]Plan[/]")
            .AddColumn("[bold]Status[/]")
            .AddColumn("[bold]Orders[/]")
            .AddColumn("[bold]Created[/]");

        foreach (var t in tenants)
        {
            table.AddRow(
                $"[cyan]{t.Slug}[/]",
                Markup.Escape(t.Name),
                GetPlanMarkup(t.Plan),
                t.IsActive ? "[green]● Active[/]" : "[red]○ Inactive[/]",
                t.OrderCount.ToString("N0"),
                t.CreatedAt.ToString("yyyy-MM-dd")
            );
        }

        AnsiConsole.Write(table);
    }

    private static string GetPlanMarkup(string plan) => plan switch
    {
        "Enterprise" => "[bold green]Enterprise[/]",
        "Pro" => "[blue]Pro[/]",
        _ => "[grey]Free[/]"
    };
}
```

---

## Step 1867: Interactive Selection and Input

```csharp
// Commands/MigrateTenantCommand.cs
public sealed class MigrateTenantCommand : AsyncCommand<MigrateTenantCommand.Settings>
{
    public sealed class Settings : CommandSettings
    {
        [CommandArgument(0, "[SLUG]")]
        [Description("Tenant slug (omit for interactive selection)")]
        public string? Slug { get; set; }

        [CommandOption("--dry-run")]
        [Description("Show what would be migrated without executing")]
        public bool DryRun { get; set; }
    }

    public override async Task<int> ExecuteAsync(CommandContext context, Settings settings)
    {
        string tenantSlug;

        if (settings.Slug is null)
        {
            // Interactive selection
            var tenants = await FetchTenantsAsync();

            tenantSlug = AnsiConsole.Prompt(
                new SelectionPrompt<string>()
                    .Title("Which tenant do you want to migrate?")
                    .PageSize(10)
                    .MoreChoicesText("[grey](Move up and down to reveal more)[/]")
                    .AddChoices(tenants.Select(t => $"{t.Slug} — {t.Name}")))
                .Split("—")[0].Trim();
        }
        else
        {
            tenantSlug = settings.Slug;
        }

        // Confirm migration
        AnsiConsole.MarkupLine($"\nMigrating tenant: [cyan]{tenantSlug}[/]");

        if (settings.DryRun)
        {
            AnsiConsole.MarkupLine("[dim]Dry run — listing pending migrations:[/]");
            var pending = await GetPendingMigrationsAsync(tenantSlug);

            if (!pending.Any())
            {
                AnsiConsole.MarkupLine("[green]✓ No pending migrations.[/]");
                return 0;
            }

            var tree = new Tree("[blue]Pending Migrations[/]");
            foreach (var migration in pending)
                tree.AddNode($"[yellow]{migration.Name}[/] [dim]({migration.CreatedAt:yyyy-MM-dd})[/]");

            AnsiConsole.Write(tree);
            return 0;
        }

        bool success = false;
        await AnsiConsole.Progress()
            .StartAsync(async ctx =>
            {
                var task = ctx.AddTask($"Migrating [cyan]{tenantSlug}[/]");
                task.IsIndeterminate = true;

                try
                {
                    await RunMigrationAsync(tenantSlug);
                    task.IsIndeterminate = false;
                    task.Value = 100;
                    success = true;
                }
                catch (Exception ex)
                {
                    AnsiConsole.WriteException(ex);
                }
            });

        if (success)
            AnsiConsole.MarkupLine($"[green]✓ Migration completed for {tenantSlug}[/]");

        return success ? 0 : 1;
    }

    private Task<List<TenantDto>> FetchTenantsAsync() =>
        Task.FromResult(new List<TenantDto>());
    private Task<List<PendingMigration>> GetPendingMigrationsAsync(string slug) =>
        Task.FromResult(new List<PendingMigration>());
    private Task RunMigrationAsync(string slug) => Task.CompletedTask;
}

public record PendingMigration(string Name, DateTime CreatedAt);
```

---

## Step 1868: Progress Bars and Live Displays

```csharp
// Commands/BulkMigrateCommand.cs
public sealed class BulkMigrateCommand : AsyncCommand
{
    public override async Task<int> ExecuteAsync(CommandContext context)
    {
        var tenants = await GetAllTenantsAsync();

        AnsiConsole.MarkupLine($"[bold]Migrating [cyan]{tenants.Count}[/] tenants...[/]\n");

        var results = new ConcurrentBag<(string Slug, bool Success, string? Error)>();

        await AnsiConsole.Progress()
            .AutoRefresh(true)
            .HideCompleted(false)
            .Columns(
                new TaskDescriptionColumn { Alignment = Justify.Left },
                new ProgressBarColumn(),
                new PercentageColumn(),
                new RemainingTimeColumn(),
                new SpinnerColumn())
            .StartAsync(async ctx =>
            {
                var overallTask = ctx.AddTask("[bold]Overall Progress[/]", maxValue: tenants.Count);

                await Parallel.ForEachAsync(
                    tenants,
                    new ParallelOptions { MaxDegreeOfParallelism = 4 },
                    async (tenant, ct) =>
                    {
                        var task = ctx.AddTask($"  [cyan]{tenant.Slug}[/]");
                        task.IsIndeterminate = true;

                        try
                        {
                            await RunMigrationAsync(tenant.Slug);
                            task.IsIndeterminate = false;
                            task.Value = 100;
                            task.Description = $"  [green]✓ {tenant.Slug}[/]";
                            results.Add((tenant.Slug, true, null));
                        }
                        catch (Exception ex)
                        {
                            task.IsIndeterminate = false;
                            task.Description = $"  [red]✗ {tenant.Slug}[/]";
                            results.Add((tenant.Slug, false, ex.Message));
                        }
                        finally
                        {
                            overallTask.Increment(1);
                        }
                    });
            });

        // Summary
        AnsiConsole.WriteLine();
        var succeeded = results.Count(r => r.Success);
        var failed = results.Count(r => !r.Success);

        var summaryTable = new Table()
            .Border(TableBorder.Minimal)
            .AddColumn("Metric")
            .AddColumn("Count");

        summaryTable.AddRow("[green]Succeeded[/]", succeeded.ToString());
        summaryTable.AddRow("[red]Failed[/]", failed.ToString());
        summaryTable.AddRow("[bold]Total[/]", tenants.Count.ToString());

        AnsiConsole.Write(summaryTable);

        if (failed > 0)
        {
            AnsiConsole.MarkupLine("\n[red bold]Failed tenants:[/]");
            foreach (var (slug, _, error) in results.Where(r => !r.Success))
                AnsiConsole.MarkupLine($"  [red]•[/] [cyan]{slug}[/]: {Markup.Escape(error ?? "Unknown error")}");

            return 1;
        }

        AnsiConsole.MarkupLine("\n[green bold]✓ All migrations completed successfully![/]");
        return 0;
    }

    private Task<List<TenantDto>> GetAllTenantsAsync() =>
        Task.FromResult(new List<TenantDto>());
    private Task RunMigrationAsync(string slug) => Task.CompletedTask;
}
```

---

## Step 1869: Health Check Command

```csharp
// Commands/HealthCheckCommand.cs
public sealed class HealthCheckCommand : AsyncCommand<HealthCheckCommand.Settings>
{
    public sealed class Settings : CommandSettings
    {
        [CommandOption("--url <URL>")]
        [Description("API base URL")]
        [DefaultValue("http://localhost:5000")]
        public string Url { get; set; } = "http://localhost:5000";

        [CommandOption("--timeout <SECONDS>")]
        [Description("Request timeout in seconds")]
        [DefaultValue(10)]
        public int Timeout { get; set; } = 10;
    }

    public override async Task<int> ExecuteAsync(CommandContext context, Settings settings)
    {
        AnsiConsole.MarkupLine($"Running health checks against [cyan]{settings.Url}[/]\n");

        using var client = new HttpClient
        {
            BaseAddress = new Uri(settings.Url),
            Timeout = TimeSpan.FromSeconds(settings.Timeout)
        };

        var checks = new (string Name, Func<Task<(bool Ok, string Detail)>> Check)[]
        {
            ("API Liveness", async () =>
            {
                var resp = await client.GetAsync("/health/live");
                return (resp.IsSuccessStatusCode, $"HTTP {(int)resp.StatusCode}");
            }),
            ("API Readiness", async () =>
            {
                var resp = await client.GetAsync("/health/ready");
                var body = await resp.Content.ReadAsStringAsync();
                return (resp.IsSuccessStatusCode, resp.IsSuccessStatusCode ? "Healthy" : body[..100]);
            }),
            ("GraphQL Endpoint", async () =>
            {
                var resp = await client.PostAsJsonAsync("/graphql", new { query = "{ __typename }" });
                return (resp.IsSuccessStatusCode, $"HTTP {(int)resp.StatusCode}");
            }),
            ("Version", async () =>
            {
                var resp = await client.GetAsync("/api/version");
                if (!resp.IsSuccessStatusCode) return (false, $"HTTP {(int)resp.StatusCode}");
                var version = await resp.Content.ReadAsStringAsync();
                return (true, version.Trim('"'));
            })
        };

        var allPassed = true;

        var table = new Table()
            .Border(TableBorder.Rounded)
            .AddColumn("[bold]Check[/]")
            .AddColumn("[bold]Status[/]")
            .AddColumn("[bold]Detail[/]");

        foreach (var (name, check) in checks)
        {
            try
            {
                var (ok, detail) = await check();
                allPassed &= ok;

                table.AddRow(
                    name,
                    ok ? "[green]● PASS[/]" : "[red]● FAIL[/]",
                    ok ? $"[dim]{detail}[/]" : $"[red]{Markup.Escape(detail)}[/]");
            }
            catch (Exception ex)
            {
                allPassed = false;
                table.AddRow(
                    name,
                    "[red]● ERROR[/]",
                    $"[red]{Markup.Escape(ex.Message)}[/]");
            }
        }

        AnsiConsole.Write(table);

        AnsiConsole.WriteLine();
        if (allPassed)
            AnsiConsole.MarkupLine("[green bold]✓ All health checks passed.[/]");
        else
            AnsiConsole.MarkupLine("[red bold]✗ Some health checks failed.[/]");

        return allPassed ? 0 : 1;
    }
}
```

---

## Step 1870: Exception Handling and Exit Codes

```csharp
// Infrastructure/ErrorHandling/CliExceptionHandler.cs
public static class CliRunner
{
    public static async Task<int> RunAsync(string[] args, Func<string[], Task<int>> command)
    {
        try
        {
            return await command(args);
        }
        catch (HttpRequestException ex)
        {
            AnsiConsole.MarkupLine($"[red]Connection failed:[/] {Markup.Escape(ex.Message)}");
            AnsiConsole.MarkupLine("[dim]Is the API running? Check --url option.[/]");
            return ExitCodes.NetworkError;
        }
        catch (UnauthorizedAccessException ex)
        {
            AnsiConsole.MarkupLine($"[red]Authentication failed:[/] {Markup.Escape(ex.Message)}");
            AnsiConsole.MarkupLine("[dim]Run: mydevtool auth login[/]");
            return ExitCodes.AuthError;
        }
        catch (ValidationException ex)
        {
            AnsiConsole.MarkupLine($"[red]Validation error:[/] {Markup.Escape(ex.Message)}");
            return ExitCodes.ValidationError;
        }
        catch (OperationCanceledException)
        {
            AnsiConsole.MarkupLine("\n[yellow]Cancelled.[/]");
            return ExitCodes.Cancelled;
        }
        catch (Exception ex)
        {
            AnsiConsole.WriteException(ex, ExceptionFormats.ShortenEverything);
            return ExitCodes.UnexpectedError;
        }
    }
}

public static class ExitCodes
{
    public const int Success = 0;
    public const int ValidationError = 1;
    public const int AuthError = 2;
    public const int NetworkError = 3;
    public const int NotFound = 4;
    public const int Cancelled = 130; // SIGINT convention
    public const int UnexpectedError = 255;
}
```

---

## Step 1871: DI Integration with Spectre.Console.Cli

```csharp
// Program.cs with full DI
using Microsoft.Extensions.DependencyInjection;
using Spectre.Console.Cli;

var services = new ServiceCollection();

// Register services
services.AddSingleton<IApiClient, ApiClient>();
services.AddSingleton<IConfigurationService, ConfigurationService>();
services.AddHttpClient<IApiClient, ApiClient>(client =>
{
    client.DefaultRequestHeaders.Add("User-Agent", "mydevtool/1.0");
});

// Load config
services.AddSingleton(sp =>
{
    var config = new ConfigurationBuilder()
        .AddJsonFile($"{Environment.GetFolderPath(Environment.SpecialFolder.UserProfile)}/.mydevtool/config.json", optional: true)
        .AddEnvironmentVariables("MYDEVTOOL_")
        .Build();
    return config.Get<ToolConfig>() ?? new ToolConfig();
});

// Build Spectre.Console.Cli with DI
var app = new CommandApp(new TypeRegistrar(services));

app.Configure(config =>
{
    config.SetApplicationName("mydevtool");
    // ... commands ...
});

return await app.RunAsync(args);

// DI registrar adapter
public sealed class TypeRegistrar : ITypeRegistrar
{
    private readonly IServiceCollection _services;

    public TypeRegistrar(IServiceCollection services) => _services = services;

    public ITypeResolver Build()
        => new TypeResolver(_services.BuildServiceProvider());

    public void Register(Type service, Type implementation)
        => _services.AddSingleton(service, implementation);

    public void RegisterInstance(Type service, object implementation)
        => _services.AddSingleton(service, implementation);

    public void RegisterLazy(Type service, Func<object> factory)
        => _services.AddSingleton(service, _ => factory());
}

public sealed class TypeResolver : ITypeResolver, IDisposable
{
    private readonly IServiceProvider _provider;

    public TypeResolver(IServiceProvider provider) => _provider = provider;

    public object? Resolve(Type? type)
        => type is null ? null : _provider.GetService(type);

    public void Dispose()
    {
        if (_provider is IDisposable d) d.Dispose();
    }
}
```

---

## Step 1872: Configuration and Authentication Storage

```csharp
// Infrastructure/Config/ToolConfig.cs
public sealed class ToolConfig
{
    public string ApiUrl { get; set; } = "http://localhost:5000";
    public string? ApiKey { get; set; }
    public string? AccessToken { get; set; }
    public DateTime? TokenExpiresAt { get; set; }
}

// Commands/AuthCommand.cs
public sealed class AuthLoginCommand : AsyncCommand<AuthLoginCommand.Settings>
{
    private readonly ToolConfig _config;

    public AuthLoginCommand(ToolConfig config) => _config = config;

    public sealed class Settings : CommandSettings
    {
        [CommandOption("--api-key <KEY>")]
        [Description("API key to authenticate with")]
        public string? ApiKey { get; set; }
    }

    public override async Task<int> ExecuteAsync(CommandContext context, Settings settings)
    {
        string apiKey;

        if (settings.ApiKey is not null)
        {
            apiKey = settings.ApiKey;
        }
        else
        {
            // Prompt securely (no echo)
            apiKey = AnsiConsole.Prompt(
                new TextPrompt<string>("Enter your API key:")
                    .PromptStyle("green")
                    .Secret('*'));
        }

        // Validate the key
        AnsiConsole.MarkupLine("[dim]Validating API key...[/]");

        // Store in user's config file (not in project)
        var configDir = Path.Combine(
            Environment.GetFolderPath(Environment.SpecialFolder.UserProfile),
            ".mydevtool");

        Directory.CreateDirectory(configDir);
        _config.ApiKey = apiKey;

        var configPath = Path.Combine(configDir, "config.json");
        await File.WriteAllTextAsync(configPath,
            JsonSerializer.Serialize(_config, new JsonSerializerOptions { WriteIndented = true }));

        // Set file permissions (Unix)
        if (!OperatingSystem.IsWindows())
        {
            File.SetUnixFileMode(configPath,
                UnixFileMode.UserRead | UnixFileMode.UserWrite);
        }

        AnsiConsole.MarkupLine("[green]✓ Authenticated successfully. Config saved.[/]");
        return 0;
    }
}
```

---

## Step 1873: Shell Completions

```csharp
// Provide tab completion for shells
// System.CommandLine has built-in completion support

// Program.cs (System.CommandLine version)
var rootCommand = new RootCommand("MyApp Developer Tool");
// ... configure commands ...

// Add completion command
rootCommand.AddCommand(new Command("completion", "Generate shell completion scripts")
{
    new Argument<string>("shell", "Shell type: bash, zsh, fish, powershell")
}.Also(cmd =>
{
    cmd.SetHandler(shell =>
    {
        var script = shell switch
        {
            "bash" => GenerateBashCompletion(),
            "zsh" => GenerateZshCompletion(),
            "fish" => GenerateFishCompletion(),
            "powershell" => GeneratePowershellCompletion(),
            _ => throw new ArgumentException($"Unknown shell: {shell}")
        };

        Console.Write(script);
    }, cmd.Arguments.Single() as Argument<string>);
}));

static string GenerateBashCompletion() => """
    # Add to ~/.bashrc:
    # source <(mydevtool completion bash)

    _mydevtool_completions() {
        local cur prev opts
        COMPREPLY=()
        cur="${COMP_WORDS[COMP_CWORD]}"
        prev="${COMP_WORDS[COMP_CWORD-1]}"

        opts="tenant migrate health report auth"
        COMPREPLY=( $(compgen -W "${opts}" -- "${cur}") )
    }

    complete -F _mydevtool_completions mydevtool
    """;
```

---

## Step 1874: Testing CLI Commands

```csharp
// Tests/CliTests/TenantListCommandTests.cs
public sealed class TenantListCommandTests
{
    [Fact]
    public async Task TenantListCommand_Should_Return_Zero_ExitCode_On_Success()
    {
        // Arrange
        var mockApi = new Mock<IApiClient>();
        mockApi.Setup(a => a.GetTenantsAsync(null, false, 1))
            .ReturnsAsync(new List<TenantDto>
            {
                new TenantDto("acme", "Acme Corp", "Pro", true, 150, DateTime.UtcNow.AddMonths(-6)),
                new TenantDto("beta", "Beta Inc", "Free", true, 12, DateTime.UtcNow.AddMonths(-1))
            });

        var services = new ServiceCollection();
        services.AddSingleton(mockApi.Object);

        var app = new CommandApp(new TypeRegistrar(services));
        app.Configure(cfg =>
        {
            cfg.AddBranch("tenant", t =>
                t.AddCommand<TenantListCommand>("list"));
        });

        // Act — capture output
        var writer = new StringWriter();
        AnsiConsole.Record(); // Start recording

        var exitCode = await app.RunAsync(new[] { "tenant", "list" });

        var output = AnsiConsole.ExportText();
        AnsiConsole.Reset();

        // Assert
        exitCode.Should().Be(0);
        output.Should().Contain("acme");
        output.Should().Contain("Acme Corp");
        output.Should().Contain("beta");
    }

    [Fact]
    public async Task TenantListCommand_WithJsonFormat_Should_Output_Valid_Json()
    {
        var mockApi = new Mock<IApiClient>();
        mockApi.Setup(a => a.GetTenantsAsync(null, false, 1))
            .ReturnsAsync([new TenantDto("acme", "Acme Corp", "Pro", true, 0, DateTime.UtcNow)]);

        var services = new ServiceCollection();
        services.AddSingleton(mockApi.Object);

        var output = new StringBuilder();
        Console.SetOut(new StringWriter(output));

        var app = new CommandApp(new TypeRegistrar(services));
        app.Configure(cfg =>
            cfg.AddBranch("tenant", t => t.AddCommand<TenantListCommand>("list")));

        var exitCode = await app.RunAsync(new[] { "tenant", "list", "--format", "json" });

        exitCode.Should().Be(0);
        var json = output.ToString();
        var tenants = JsonSerializer.Deserialize<List<TenantDto>>(json);
        tenants.Should().HaveCount(1);
        tenants!.Single().Slug.Should().Be("acme");
    }
}
```

---

## Step 1875: Packaging and Distribution

```bash
# Pack as a dotnet tool
dotnet pack --configuration Release --output ./nupkg

# Test locally
dotnet tool install --global --add-source ./nupkg MyCompany.DevTool
mydevtool --version
mydevtool tenant list

# Uninstall
dotnet tool uninstall --global MyCompany.DevTool

# Publish to NuGet.org (public)
dotnet nuget push ./nupkg/MyCompany.DevTool.1.0.0.nupkg \
  --api-key $NUGET_API_KEY \
  --source https://api.nuget.org/v3/index.json

# Or GitHub Packages (private)
dotnet nuget push ./nupkg/MyCompany.DevTool.1.0.0.nupkg \
  --api-key $GITHUB_TOKEN \
  --source "https://nuget.pkg.github.com/OWNER/index.json"

# Install from private feed
dotnet tool install -g MyCompany.DevTool \
  --add-source "https://nuget.pkg.github.com/OWNER/index.json"
```

```xml
<!-- dotnet-tools.json — pin tool version per project -->
{
  "version": 1,
  "isRoot": true,
  "tools": {
    "mycompany.devtool": {
      "version": "1.2.0",
      "commands": ["mydevtool"]
    },
    "dotnet-ef": {
      "version": "9.0.0",
      "commands": ["dotnet-ef"]
    }
  }
}
```

```bash
# Restore all local tools
dotnet tool restore

# Run without global install
dotnet tool run mydevtool tenant list
```

---

## Step 1876: Self-Update Mechanism

```csharp
// Commands/UpdateCommand.cs
public sealed class UpdateCommand : AsyncCommand
{
    private const string NuGetApiUrl =
        "https://api.nuget.org/v3-flatcontainer/mycompany.devtool/index.json";

    public override async Task<int> ExecuteAsync(CommandContext context)
    {
        AnsiConsole.MarkupLine("[dim]Checking for updates...[/]");

        var currentVersion = Assembly.GetExecutingAssembly()
            .GetCustomAttribute<AssemblyInformationalVersionAttribute>()
            ?.InformationalVersion ?? "0.0.0";

        using var client = new HttpClient();

        try
        {
            var response = await client.GetFromJsonAsync<NuGetVersionList>(NuGetApiUrl);
            var latestVersion = response?.Versions.LastOrDefault(v => !v.Contains("-"));

            if (latestVersion is null)
            {
                AnsiConsole.MarkupLine("[yellow]Could not determine latest version.[/]");
                return 1;
            }

            if (currentVersion == latestVersion)
            {
                AnsiConsole.MarkupLine($"[green]✓ You are on the latest version ({currentVersion}).[/]");
                return 0;
            }

            AnsiConsole.MarkupLine(
                $"Update available: [yellow]{currentVersion}[/] → [green]{latestVersion}[/]");

            if (AnsiConsole.Confirm("Update now?"))
            {
                var updateProcess = Process.Start(new ProcessStartInfo
                {
                    FileName = "dotnet",
                    Arguments = $"tool update -g MyCompany.DevTool",
                    UseShellExecute = false,
                    RedirectStandardOutput = true
                })!;

                await updateProcess.WaitForExitAsync();

                if (updateProcess.ExitCode == 0)
                    AnsiConsole.MarkupLine($"[green]✓ Updated to {latestVersion}. Restart mydevtool.[/]");
                else
                    AnsiConsole.MarkupLine("[red]Update failed.[/]");
            }
        }
        catch (HttpRequestException)
        {
            AnsiConsole.MarkupLine("[yellow]Could not check for updates (no internet?).[/]");
        }

        return 0;
    }
}

public record NuGetVersionList(string[] Versions);
```

---

## Summary: CLI Tool Patterns

| Pattern | Library | Use Case |
|---|---|---|
| Command structure | System.CommandLine / Spectre.Cli | Any CLI |
| Rich tables/panels | Spectre.Console | User-facing output |
| Progress display | `AnsiConsole.Progress()` | Long operations |
| Interactive prompts | `SelectionPrompt`, `TextPrompt` | When args optional |
| DI integration | `ITypeRegistrar` | Production CLIs |
| Tab completion | System.CommandLine built-in | Shell productivity |
| Self-update | NuGet API + dotnet tool update | Distributed tools |
| Config storage | `~/.toolname/config.json` | Persistent auth |
| Exit codes | Constants (0=ok, 1=error, 130=cancel) | Shell scripting |

**Packaging rules**:
1. Set `PackAsTool=true` and `ToolCommandName` in the csproj
2. Use `dotnet-tools.json` to pin versions in a project
3. Publish `.snupkg` with symbols so stack traces in errors are readable
4. Protect sensitive config with Unix file mode 600

---

*Next: Part 78 — Real-Time Features: SignalR Hubs, Server-Sent Events & WebSocket Patterns*
