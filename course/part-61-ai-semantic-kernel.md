# Part 61: AI Integration — Semantic Kernel & MCP Servers in .NET

## Steps 1614-1635: Building Production AI Applications

---

## Step 1614: AI Application Architecture

```
Semantic Kernel Architecture
══════════════════════════════════════════════════════════════

  User / Application
        │
        ▼
  ┌──────────────────────────────────────────────────────────┐
  │                    Semantic Kernel                        │
  │                                                          │
  │  ┌──────────────┐   ┌───────────────┐   ┌────────────┐  │
  │  │   Planner    │   │    Plugins    │   │  Memory /  │  │
  │  │  (auto-step  │◄──│ (functions,  │   │  Vector DB │  │
  │  │  orchestrate)│   │  grounded)   │   │            │  │
  │  └──────────────┘   └───────────────┘   └────────────┘  │
  │         │                  │                  │          │
  │         ▼                  ▼                  ▼          │
  │  ┌──────────────────────────────────────────────────────┐ │
  │  │                  AI Connector                         │ │
  │  │  OpenAI / Azure OpenAI / Ollama / Claude             │ │
  │  └──────────────────────────────────────────────────────┘ │
  └──────────────────────────────────────────────────────────┘

  MCP (Model Context Protocol):
  AI Model ↔ MCP Client ↔ MCP Server (tools, resources, prompts)
                          ├── Database Tool
                          ├── File System Tool
                          ├── API Tool
                          └── Code Execution Tool
```

### NuGet Packages

```xml
<PackageReference Include="Microsoft.SemanticKernel"                    Version="1.*" />
<PackageReference Include="Microsoft.SemanticKernel.Connectors.OpenAI"  Version="1.*" />
<PackageReference Include="Microsoft.SemanticKernel.Plugins.Core"       Version="1.*" />
<PackageReference Include="Microsoft.SemanticKernel.Plugins.Web"        Version="1.*" />
<PackageReference Include="Microsoft.SemanticKernel.Connectors.Qdrant"  Version="1.*" />
<PackageReference Include="Microsoft.SemanticKernel.Agents.Core"        Version="1.*" />
<PackageReference Include="ModelContextProtocol"                        Version="0.*" />
<PackageReference Include="ModelContextProtocol.AspNetCore"             Version="0.*" />
```

---

## Step 1615: Semantic Kernel Setup

```csharp
// Program.cs
using Microsoft.SemanticKernel;
using Microsoft.SemanticKernel.ChatCompletion;
using Microsoft.SemanticKernel.Connectors.OpenAI;

var builder = WebApplication.CreateBuilder(args);

// ── Semantic Kernel ───────────────────────────────────────────────────────
builder.Services.AddKernel()
    // AI model
    .AddAzureOpenAIChatCompletion(
        deploymentName: builder.Configuration["AzureOpenAI:Deployment"]!,
        endpoint:       builder.Configuration["AzureOpenAI:Endpoint"]!,
        apiKey:         builder.Configuration["AzureOpenAI:ApiKey"]!)

    // Text embeddings for memory
    .AddAzureOpenAITextEmbeddingGeneration(
        deploymentName: builder.Configuration["AzureOpenAI:EmbeddingDeployment"]!,
        endpoint:       builder.Configuration["AzureOpenAI:Endpoint"]!,
        apiKey:         builder.Configuration["AzureOpenAI:ApiKey"]!)

    // Plugins
    .Plugins.AddFromType<OrderPlugin>()
    .Plugins.AddFromType<InventoryPlugin>()
    .Plugins.AddFromType<CustomerPlugin>();

// ── Vector Memory ─────────────────────────────────────────────────────────
builder.Services.AddSingleton<ITextEmbeddingGenerationService>(sp =>
{
    var kernel = sp.GetRequiredService<Kernel>();
    return kernel.GetRequiredService<ITextEmbeddingGenerationService>();
});

builder.Services.AddQdrantVectorStore(
    host: builder.Configuration["Qdrant:Host"]!,
    port: int.Parse(builder.Configuration["Qdrant:Port"]!));

builder.Services.AddScoped<IMemoryStore, QdrantMemoryStore>();
builder.Services.AddScoped<SemanticTextMemory>();

// ── Services ──────────────────────────────────────────────────────────────
builder.Services.AddScoped<OrderAiService>();
builder.Services.AddScoped<DocumentIntelligenceService>();

var app = builder.Build();

// ── Endpoints ─────────────────────────────────────────────────────────────
app.MapPost("/api/ai/chat", HandleChatAsync);
app.MapPost("/api/ai/orders/analyze", AnalyzeOrdersAsync);
app.MapPost("/api/ai/documents/extract", ExtractDocumentAsync);

app.Run();
```

---

## Step 1616: Plugins — Ground the AI in Business Logic

```csharp
// Plugins/OrderPlugin.cs
using Microsoft.SemanticKernel;
using System.ComponentModel;

namespace AiDemo.Plugins;

public class OrderPlugin
{
    private readonly IOrderRepository _repo;
    private readonly ILogger<OrderPlugin> _log;

    public OrderPlugin(IOrderRepository repo, ILogger<OrderPlugin> log)
    {
        _repo = repo;
        _log  = log;
    }

    [KernelFunction("get_order")]
    [Description("Retrieves an order by its ID. Returns order details including status, items, and timeline.")]
    public async Task<string> GetOrderAsync(
        [Description("The unique order ID (GUID format)")]
        string orderId,
        CancellationToken cancellationToken = default)
    {
        if (!Guid.TryParse(orderId, out var id))
            return "Error: Invalid order ID format. Expected GUID.";

        var order = await _repo.GetByIdAsync(id, cancellationToken);
        if (order is null)
            return $"Order {orderId} not found.";

        return System.Text.Json.JsonSerializer.Serialize(new
        {
            id          = order.Id,
            customer_id = order.CustomerId,
            status      = order.Status.ToString(),
            total       = order.Total,
            currency    = "USD",
            items       = order.Items.Select(i => new { i.ProductId, i.Name, i.Quantity, i.UnitPrice }),
            placed_at   = order.CreatedAt,
            updated_at  = order.UpdatedAt
        });
    }

    [KernelFunction("list_orders")]
    [Description("Lists orders for a customer. Optionally filter by status. Returns up to 20 most recent orders.")]
    public async Task<string> ListOrdersAsync(
        [Description("Customer ID to look up orders for")]
        string customerId,
        [Description("Optional status filter: Pending, Confirmed, Shipped, Delivered, Cancelled")]
        string? status = null,
        CancellationToken cancellationToken = default)
    {
        var orders = await _repo.ListByCustomerAsync(customerId, status, 20, cancellationToken);

        if (!orders.Any())
            return $"No orders found for customer {customerId}" +
                   (status is not null ? $" with status {status}" : "");

        return System.Text.Json.JsonSerializer.Serialize(orders.Select(o => new
        {
            id     = o.Id,
            status = o.Status.ToString(),
            total  = o.Total,
            placed = o.CreatedAt.ToString("yyyy-MM-dd")
        }));
    }

    [KernelFunction("cancel_order")]
    [Description("Cancels an order. Returns success/failure message.")]
    public async Task<string> CancelOrderAsync(
        [Description("The order ID to cancel")]
        string orderId,
        [Description("Reason for cancellation")]
        string reason,
        CancellationToken cancellationToken = default)
    {
        if (!Guid.TryParse(orderId, out var id))
            return "Error: Invalid order ID format.";

        try
        {
            await _repo.CancelAsync(id, reason, "ai-assistant", cancellationToken);
            return $"Order {orderId} has been cancelled successfully. Reason: {reason}";
        }
        catch (DomainException ex)
        {
            return $"Cannot cancel order: {ex.Message}";
        }
    }

    [KernelFunction("get_order_tracking")]
    [Description("Gets shipping tracking information for an order")]
    public async Task<string> GetTrackingAsync(
        [Description("Order ID to get tracking for")]
        string orderId,
        CancellationToken cancellationToken = default)
    {
        if (!Guid.TryParse(orderId, out var id))
            return "Error: Invalid order ID format.";

        var tracking = await _repo.GetTrackingAsync(id, cancellationToken);
        if (tracking is null)
            return "No tracking information available for this order.";

        return System.Text.Json.JsonSerializer.Serialize(tracking);
    }
}
```

```csharp
// Plugins/InventoryPlugin.cs
public class InventoryPlugin
{
    private readonly IInventoryRepository _inventory;

    public InventoryPlugin(IInventoryRepository inventory) => _inventory = inventory;

    [KernelFunction("check_stock")]
    [Description("Checks if a product is in stock and returns availability information")]
    public async Task<string> CheckStockAsync(
        [Description("Product ID to check stock for")]
        string productId,
        [Description("Quantity to check availability for (default: 1)")]
        int quantity = 1,
        CancellationToken cancellationToken = default)
    {
        var stock = await _inventory.GetStockAsync(productId, cancellationToken);

        return System.Text.Json.JsonSerializer.Serialize(new
        {
            product_id     = productId,
            available      = stock?.Quantity >= quantity,
            current_stock  = stock?.Quantity ?? 0,
            warehouse      = stock?.Warehouse,
            restock_date   = stock?.RestockDate?.ToString("yyyy-MM-dd")
        });
    }

    [KernelFunction("get_product_details")]
    [Description("Gets detailed information about a product including price, description, and availability")]
    public async Task<string> GetProductDetailsAsync(
        [Description("Product ID or name to look up")]
        string productIdentifier,
        CancellationToken cancellationToken = default)
    {
        var product = await _inventory.FindProductAsync(productIdentifier, cancellationToken);

        if (product is null)
            return $"Product '{productIdentifier}' not found in catalog.";

        return System.Text.Json.JsonSerializer.Serialize(product);
    }
}
```

---

## Step 1617: Chat Completion with Tool Calling

```csharp
// Services/OrderAiService.cs
using Microsoft.SemanticKernel;
using Microsoft.SemanticKernel.ChatCompletion;
using Microsoft.SemanticKernel.Connectors.OpenAI;

public class OrderAiService
{
    private readonly Kernel _kernel;
    private readonly IChatCompletionService _chat;
    private readonly ILogger<OrderAiService> _log;

    public OrderAiService(
        Kernel kernel,
        IChatCompletionService chat,
        ILogger<OrderAiService> log)
    {
        _kernel = kernel;
        _chat   = chat;
        _log    = log;
    }

    public async Task<string> ChatAsync(
        string userId,
        string message,
        ChatHistory history,
        CancellationToken ct = default)
    {
        // System prompt grounded in business context
        history.AddSystemMessage("""
            You are a helpful customer service assistant for ShopCo.
            You have access to order information and can help customers with:
            - Checking order status and history
            - Tracking shipments
            - Cancelling orders (if not yet shipped)
            - Answering questions about products

            Always be polite, concise, and accurate.
            When an order is not found, suggest the customer check their order confirmation email.
            Never make up order information — only use the tools provided.
            Customer user ID: """ + userId);

        history.AddUserMessage(message);

        // Enable automatic function calling
        var executionSettings = new OpenAIPromptExecutionSettings
        {
            ToolCallBehavior    = ToolCallBehavior.AutoInvokeKernelFunctions,
            MaxAutoInvokeAttempts = 5,
            Temperature         = 0.3,  // lower = more factual
            MaxTokens           = 2000
        };

        var result = await _chat.GetChatMessageContentAsync(
            history,
            executionSettings,
            _kernel,
            ct);

        history.AddAssistantMessage(result.Content ?? "");
        _log.LogInformation("AI response generated for user {UserId}", userId);

        return result.Content ?? "I'm unable to process your request right now.";
    }

    public async IAsyncEnumerable<string> ChatStreamAsync(
        string userId,
        string message,
        ChatHistory history,
        [System.Runtime.CompilerServices.EnumeratorCancellation] CancellationToken ct = default)
    {
        history.AddSystemMessage("""
            You are a helpful order management assistant.
            Use available tools to answer questions accurately.
            """);
        history.AddUserMessage(message);

        var settings = new OpenAIPromptExecutionSettings
        {
            ToolCallBehavior = ToolCallBehavior.AutoInvokeKernelFunctions,
            Temperature      = 0.3
        };

        await foreach (var chunk in _chat.GetStreamingChatMessageContentsAsync(
            history, settings, _kernel, ct))
        {
            if (chunk.Content is not null)
                yield return chunk.Content;
        }
    }
}
```

---

## Step 1618: Semantic Memory & RAG (Retrieval Augmented Generation)

```csharp
// Services/DocumentIntelligenceService.cs — RAG for product catalog / documentation
using Microsoft.SemanticKernel.Memory;
using Microsoft.SemanticKernel.Text;

public class DocumentIntelligenceService
{
    private readonly SemanticTextMemory _memory;
    private readonly ITextEmbeddingGenerationService _embeddings;
    private readonly Kernel _kernel;
    private const string Collection = "product-catalog";

    public DocumentIntelligenceService(
        SemanticTextMemory memory,
        ITextEmbeddingGenerationService embeddings,
        Kernel kernel)
    {
        _memory     = memory;
        _embeddings = embeddings;
        _kernel     = kernel;
    }

    // ── Ingestion ──────────────────────────────────────────────────────────

    public async Task IndexProductsAsync(
        IEnumerable<ProductDocument> products,
        CancellationToken ct = default)
    {
        foreach (var product in products)
        {
            var text = $"""
                Product: {product.Name}
                SKU: {product.Sku}
                Category: {product.Category}
                Description: {product.Description}
                Price: ${product.Price}
                Features: {string.Join(", ", product.Features)}
                """;

            await _memory.SaveInformationAsync(
                collection:   Collection,
                id:           product.Sku,
                text:         text,
                description:  product.Name,
                additionalMetadata: System.Text.Json.JsonSerializer.Serialize(new
                {
                    product.Price,
                    product.Category,
                    product.InStock
                }),
                cancellationToken: ct);
        }
    }

    public async Task IndexDocumentAsync(
        string documentId,
        string title,
        string content,
        CancellationToken ct = default)
    {
        // Chunk large documents
        var paragraphs = TextChunker.SplitPlainTextParagraphs(
            lines: [content],
            maxTokensPerParagraph: 200,
            overlapTokens: 20);

        for (int i = 0; i < paragraphs.Count; i++)
        {
            await _memory.SaveInformationAsync(
                collection: "documentation",
                id:         $"{documentId}-chunk-{i}",
                text:       paragraphs[i],
                description: $"{title} (part {i + 1})",
                cancellationToken: ct);
        }
    }

    // ── Retrieval ──────────────────────────────────────────────────────────

    public async Task<string> AnswerProductQuestionAsync(
        string question,
        CancellationToken ct = default)
    {
        // 1. Find relevant products via semantic search
        var relevant = new List<string>();
        await foreach (var result in _memory.SearchAsync(
            Collection, question, limit: 5, minRelevanceScore: 0.7, ct))
        {
            relevant.Add(result.Metadata.Text);
        }

        if (!relevant.Any())
            return "I don't have information about that product in our catalog.";

        // 2. Generate answer grounded in retrieved context
        var context  = string.Join("\n\n---\n\n", relevant);
        var function = _kernel.CreateFunctionFromPrompt("""
            Based ONLY on the following product catalog information, answer the customer's question.
            If the answer is not in the provided context, say you don't have that information.
            Do not make up prices, availability, or features.

            Context:
            {{$context}}

            Customer Question: {{$question}}

            Answer:
            """);

        var result = await _kernel.InvokeAsync(function, new KernelArguments
        {
            ["context"]  = context,
            ["question"] = question
        }, ct);

        return result.ToString();
    }

    // ── Hybrid Search ──────────────────────────────────────────────────────

    public async Task<IReadOnlyList<ProductSearchResult>> HybridSearchAsync(
        string query,
        string? category = null,
        decimal? maxPrice = null,
        CancellationToken ct = default)
    {
        // Semantic search
        var semanticResults = new List<(string id, double score, string text)>();
        await foreach (var result in _memory.SearchAsync(
            Collection, query, limit: 20, minRelevanceScore: 0.6, ct))
        {
            var meta = System.Text.Json.JsonSerializer.Deserialize<Dictionary<string, System.Text.Json.JsonElement>>(
                result.Metadata.AdditionalMetadata ?? "{}");

            // Filter by price/category
            if (maxPrice.HasValue)
            {
                var price = meta?.GetValueOrDefault("Price").GetDecimal() ?? 0;
                if (price > maxPrice) continue;
            }

            semanticResults.Add((result.Metadata.Id, result.Relevance, result.Metadata.Text));
        }

        return semanticResults
            .OrderByDescending(r => r.score)
            .Select(r => new ProductSearchResult(r.id, r.score, r.text))
            .ToList();
    }
}

public record ProductDocument(
    string Sku, string Name, string Category,
    string Description, decimal Price,
    List<string> Features, bool InStock);

public record ProductSearchResult(string Sku, double Score, string Text);
```

---

## Step 1619: Semantic Kernel Agents

```csharp
// Services/OrderAnalysisAgent.cs
using Microsoft.SemanticKernel.Agents;
using Microsoft.SemanticKernel.Agents.Chat;

public class OrderAnalysisAgent
{
    private readonly Kernel _kernel;

    public OrderAnalysisAgent(Kernel kernel) => _kernel = kernel;

    public async Task<string> AnalyzeOrdersAsync(
        string customerId,
        CancellationToken ct = default)
    {
        // Create specialized agents
        var dataAgent = new ChatCompletionAgent
        {
            Name         = "DataAnalyst",
            Instructions = """
                You are a data analyst specializing in order analytics.
                When given order data, analyze it for:
                - Order frequency and patterns
                - Most purchased products
                - Average order value
                - Return/cancellation rate
                Provide insights in a structured format.
                """,
            Kernel = _kernel
        };

        var recommendationAgent = new ChatCompletionAgent
        {
            Name         = "RecommendationEngine",
            Instructions = """
                You are a product recommendation specialist.
                Based on customer purchase history provided by the DataAnalyst,
                suggest 3-5 relevant products they might be interested in.
                Explain why each recommendation is relevant.
                """,
            Kernel = _kernel
        };

        // Multi-agent conversation
        var chat = new AgentGroupChat(dataAgent, recommendationAgent)
        {
            ExecutionSettings = new AgentGroupChatSettings
            {
                TerminationStrategy = new ApprovalTerminationStrategy
                {
                    MaximumIterations = 10,
                    Agents = [recommendationAgent]
                }
            }
        };

        chat.AddChatMessage(new Microsoft.SemanticKernel.ChatMessageContent(
            Microsoft.SemanticKernel.ChatRole.User,
            $"Analyze orders for customer {customerId} and provide personalized recommendations."));

        var sb = new System.Text.StringBuilder();
        await foreach (var msg in chat.InvokeAsync(ct))
        {
            sb.AppendLine($"[{msg.AuthorName}]: {msg.Content}");
        }

        return sb.ToString();
    }
}

class ApprovalTerminationStrategy : TerminationStrategy
{
    protected override Task<bool> ShouldAgentTerminateAsync(
        Agent agent,
        IReadOnlyList<Microsoft.SemanticKernel.ChatMessageContent> history,
        CancellationToken cancellationToken)
    {
        // Terminate when RecommendationEngine has spoken
        return Task.FromResult(
            agent.Name == "RecommendationEngine" &&
            history.LastOrDefault()?.AuthorName == "RecommendationEngine");
    }
}
```

---

## Step 1620: MCP Server — Expose .NET APIs as AI Tools

```csharp
// McpServer/Program.cs — expose order management as MCP server
using ModelContextProtocol.Server;
using ModelContextProtocol.Protocol.Types;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddMcpServer()
    .WithHttpTransport()
    .WithTools<OrderMcpTool>()
    .WithTools<InventoryMcpTool>()
    .WithResources<OrderResourceProvider>()
    .WithPrompts<OrderPromptProvider>();

builder.Services.AddScoped<IOrderRepository, OrderRepository>();
builder.Services.AddScoped<IInventoryRepository, InventoryRepository>();

var app = builder.Build();
app.MapMcp("/mcp");  // SSE endpoint
app.Run();
```

```csharp
// McpServer/Tools/OrderMcpTool.cs
using ModelContextProtocol.Server;
using System.ComponentModel;

[McpServerToolType]
public class OrderMcpTool
{
    private readonly IOrderRepository _repo;

    public OrderMcpTool(IOrderRepository repo) => _repo = repo;

    [McpServerTool, Description("Get order details by ID")]
    public async Task<string> GetOrder(
        [Description("Order ID (GUID)")] string orderId,
        CancellationToken ct = default)
    {
        if (!Guid.TryParse(orderId, out var id))
            return "Invalid order ID format";

        var order = await _repo.GetByIdAsync(id, ct);
        return order is null
            ? $"Order {orderId} not found"
            : System.Text.Json.JsonSerializer.Serialize(order);
    }

    [McpServerTool, Description("List recent orders for a customer")]
    public async Task<string> ListOrders(
        [Description("Customer ID")] string customerId,
        [Description("Maximum number of orders to return (default 10)")] int limit = 10,
        CancellationToken ct = default)
    {
        var orders = await _repo.ListByCustomerAsync(customerId, null, limit, ct);
        return System.Text.Json.JsonSerializer.Serialize(orders);
    }

    [McpServerTool, Description("Cancel an order if it hasn't been shipped yet")]
    public async Task<string> CancelOrder(
        [Description("Order ID to cancel")] string orderId,
        [Description("Reason for cancellation")] string reason,
        CancellationToken ct = default)
    {
        if (!Guid.TryParse(orderId, out var id))
            return "Invalid order ID";

        try
        {
            await _repo.CancelAsync(id, reason, "mcp-client", ct);
            return $"Order {orderId} cancelled successfully";
        }
        catch (DomainException ex)
        {
            return $"Cannot cancel: {ex.Message}";
        }
    }

    [McpServerTool, Description("Get order statistics for a date range")]
    public async Task<string> GetOrderStats(
        [Description("Start date (YYYY-MM-DD)")] string startDate,
        [Description("End date (YYYY-MM-DD)")] string endDate,
        CancellationToken ct = default)
    {
        if (!DateOnly.TryParse(startDate, out var start) ||
            !DateOnly.TryParse(endDate, out var end))
            return "Invalid date format. Use YYYY-MM-DD";

        var stats = await _repo.GetStatsAsync(start, end, ct);
        return System.Text.Json.JsonSerializer.Serialize(stats);
    }
}
```

```csharp
// McpServer/Resources/OrderResourceProvider.cs
using ModelContextProtocol.Server;
using ModelContextProtocol.Protocol.Types;

[McpServerResourceType]
public class OrderResourceProvider
{
    private readonly IOrderRepository _repo;

    public OrderResourceProvider(IOrderRepository repo) => _repo = repo;

    [McpServerResource(UriTemplate = "orders://{orderId}")]
    [Description("Detailed order information")]
    public async Task<ResourceContents> GetOrderResource(
        string orderId,
        CancellationToken ct = default)
    {
        if (!Guid.TryParse(orderId, out var id))
            return new TextResourceContents
            {
                Uri      = $"orders://{orderId}",
                MimeType = "text/plain",
                Text     = "Invalid order ID"
            };

        var order = await _repo.GetByIdAsync(id, ct);
        if (order is null)
            return new TextResourceContents
            {
                Uri      = $"orders://{orderId}",
                MimeType = "text/plain",
                Text     = $"Order {orderId} not found"
            };

        return new TextResourceContents
        {
            Uri      = $"orders://{orderId}",
            MimeType = "application/json",
            Text     = System.Text.Json.JsonSerializer.Serialize(order,
                new System.Text.Json.JsonSerializerOptions { WriteIndented = true })
        };
    }

    [McpServerResource(UriTemplate = "customers://{customerId}/orders")]
    [Description("All orders for a customer")]
    public async Task<ResourceContents> GetCustomerOrdersResource(
        string customerId,
        CancellationToken ct = default)
    {
        var orders = await _repo.ListByCustomerAsync(customerId, null, 50, ct);
        return new TextResourceContents
        {
            Uri      = $"customers://{customerId}/orders",
            MimeType = "application/json",
            Text     = System.Text.Json.JsonSerializer.Serialize(orders,
                new System.Text.Json.JsonSerializerOptions { WriteIndented = true })
        };
    }
}
```

```csharp
// McpServer/Prompts/OrderPromptProvider.cs
using ModelContextProtocol.Server;
using ModelContextProtocol.Protocol.Types;

[McpServerPromptType]
public class OrderPromptProvider
{
    [McpServerPrompt, Description("Generate a customer service response for an order issue")]
    public static GetPromptResult OrderIssueResponse(
        [Description("Order ID")] string orderId,
        [Description("Issue type: late, damaged, wrong, missing")] string issueType)
    {
        return new GetPromptResult
        {
            Description = "Customer service response template",
            Messages =
            [
                new PromptMessage
                {
                    Role = Role.User,
                    Content = new TextContent
                    {
                        Text = $"""
                            Generate a professional and empathetic customer service response for:
                            - Order ID: {orderId}
                            - Issue Type: {issueType}

                            The response should:
                            1. Acknowledge the issue and apologize
                            2. Explain what steps will be taken
                            3. Provide a realistic resolution timeline
                            4. Include a goodwill gesture (discount code or priority shipping)

                            Keep the response under 150 words and maintain a friendly, professional tone.
                            """
                    }
                }
            ]
        };
    }
}
```

---

## Step 1621: MCP Client — Consume MCP Server from .NET

```csharp
// McpClient/McpOrderClient.cs
using ModelContextProtocol.Client;
using ModelContextProtocol.Protocol.Transport;

public class McpOrderClient : IDisposable
{
    private readonly IMcpClient _client;

    public McpOrderClient(string mcpServerUrl)
    {
        _client = McpClientFactory.Create(
            new SseClientTransport(new SseClientTransportOptions
            {
                Endpoint = new Uri(mcpServerUrl)
            }));
    }

    public async Task<string> GetOrderAsync(string orderId, CancellationToken ct = default)
    {
        var result = await _client.CallToolAsync(
            "GetOrder",
            new Dictionary<string, object?> { ["orderId"] = orderId },
            ct);

        return result.Content.First().Text ?? "No result";
    }

    public async Task<IReadOnlyList<McpTool>> ListAvailableToolsAsync(CancellationToken ct = default)
    {
        var tools = await _client.ListToolsAsync(ct);
        return tools;
    }

    public async Task<IReadOnlyList<McpResource>> ListResourcesAsync(CancellationToken ct = default)
    {
        var resources = await _client.ListResourcesAsync(ct);
        return resources;
    }

    // Use MCP server tools from Semantic Kernel
    public async Task<Kernel> BuildKernelWithMcpToolsAsync(
        string aiEndpoint,
        string apiKey,
        CancellationToken ct = default)
    {
        var tools = await _client.ListToolsAsync(ct);

        var kernelBuilder = Kernel.CreateBuilder()
            .AddAzureOpenAIChatCompletion("gpt-4o", aiEndpoint, apiKey);

        // Add MCP tools as kernel functions
        foreach (var tool in tools)
        {
            var function = KernelFunctionFactory.CreateFromMethod(
                async (KernelArguments args, CancellationToken innerCt) =>
                {
                    var arguments = args.ToDictionary(
                        kv => kv.Key,
                        kv => kv.Value);

                    var result = await _client.CallToolAsync(tool.Name, arguments, innerCt);
                    return result.Content.First().Text;
                },
                functionName: tool.Name,
                description: tool.Description);

            kernelBuilder.Plugins.AddFromFunctions(tool.Name, [function]);
        }

        return kernelBuilder.Build();
    }

    public void Dispose() => _client.DisposeAsync().AsTask().Wait();
}
```

---

## Step 1622: Structured Output & JSON Mode

```csharp
// Services/OrderClassifierService.cs — structured AI output
using Microsoft.SemanticKernel;
using System.Text.Json;

public class OrderClassifierService
{
    private readonly Kernel _kernel;

    public OrderClassifierService(Kernel kernel) => _kernel = kernel;

    public async Task<OrderIntent> ClassifyIntentAsync(
        string userMessage,
        CancellationToken ct = default)
    {
        var function = _kernel.CreateFunctionFromPrompt(
            promptTemplate: """
                Classify the user's intent regarding their order.
                Return a JSON object with the following fields:
                - intent: one of [track_order, cancel_order, modify_order, check_status, return_item, general_question]
                - order_id: extracted order ID if present (null otherwise)
                - urgency: one of [low, medium, high]
                - confidence: number between 0 and 1

                Respond with valid JSON only, no markdown.

                User message: {{$message}}
                """,
            executionSettings: new OpenAIPromptExecutionSettings
            {
                ResponseFormat = typeof(OrderIntent),  // structured output
                Temperature    = 0
            });

        var result  = await _kernel.InvokeAsync(function,
            new KernelArguments { ["message"] = userMessage }, ct);

        return JsonSerializer.Deserialize<OrderIntent>(result.ToString())
            ?? new OrderIntent("general_question", null, "low", 0.5);
    }

    public async Task<ExtractedOrderData> ExtractOrderDataAsync(
        string emailText,
        CancellationToken ct = default)
    {
        var function = _kernel.CreateFunctionFromPrompt("""
            Extract order information from the following email text.
            Return JSON with: order_id, customer_name, items (array of {product, quantity}), total, date.
            If a field is not found, use null.
            Return valid JSON only.

            Email: {{$email}}
            """);

        var result = await _kernel.InvokeAsync(function,
            new KernelArguments { ["email"] = emailText }, ct);

        return JsonSerializer.Deserialize<ExtractedOrderData>(result.ToString())
            ?? new ExtractedOrderData();
    }
}

public record OrderIntent(
    string Intent,
    string? OrderId,
    string Urgency,
    double Confidence);

public class ExtractedOrderData
{
    public string? OrderId       { get; set; }
    public string? CustomerName  { get; set; }
    public List<ExtractedItem> Items { get; set; } = [];
    public decimal? Total        { get; set; }
    public string? Date          { get; set; }
}

public record ExtractedItem(string Product, int Quantity);
```

---

## Step 1623: Streaming Chat Endpoint

```csharp
// Api/AiChatEndpoints.cs
public static class AiChatEndpoints
{
    public static IEndpointRouteBuilder MapAiChatEndpoints(this IEndpointRouteBuilder app)
    {
        app.MapPost("/api/ai/chat", HandleChatAsync)
            .RequireAuthorization();

        app.MapPost("/api/ai/chat/stream", HandleStreamingChatAsync)
            .RequireAuthorization();

        app.MapPost("/api/ai/classify", ClassifyIntentAsync)
            .RequireAuthorization();

        return app;
    }

    private static async Task<IResult> HandleChatAsync(
        ChatRequest req,
        OrderAiService service,
        HttpContext context,
        CancellationToken ct)
    {
        var userId  = context.User.FindFirst("sub")?.Value ?? "anonymous";
        var history = new ChatHistory();

        // In production: load history from session store
        var response = await service.ChatAsync(userId, req.Message, history, ct);

        return Results.Ok(new ChatResponse(response, history.Count));
    }

    private static async Task HandleStreamingChatAsync(
        ChatRequest req,
        OrderAiService service,
        HttpContext context,
        CancellationToken ct)
    {
        context.Response.ContentType = "text/event-stream";
        context.Response.Headers.CacheControl = "no-cache";

        var userId  = context.User.FindFirst("sub")?.Value ?? "anonymous";
        var history = new ChatHistory();

        await foreach (var chunk in service.ChatStreamAsync(userId, req.Message, history, ct))
        {
            var data  = System.Text.Json.JsonSerializer.Serialize(new { content = chunk });
            await context.Response.WriteAsync($"data: {data}\n\n", ct);
            await context.Response.Body.FlushAsync(ct);
        }

        await context.Response.WriteAsync("data: [DONE]\n\n", ct);
    }

    private static async Task<IResult> ClassifyIntentAsync(
        ClassifyRequest req,
        OrderClassifierService service,
        CancellationToken ct)
    {
        var intent = await service.ClassifyIntentAsync(req.Message, ct);
        return Results.Ok(intent);
    }
}

public record ChatRequest(string Message, string? SessionId = null);
public record ChatResponse(string Content, int HistoryLength);
public record ClassifyRequest(string Message);
```

---

## Step 1624: Testing AI Services

```csharp
// Tests/OrderAiServiceTests.cs
using Microsoft.SemanticKernel;
using Microsoft.SemanticKernel.ChatCompletion;
using Moq;

namespace AiDemo.Tests;

public class OrderAiServiceTests
{
    [Fact]
    public async Task ChatAsync_WithOrderQuery_ReturnsResponse()
    {
        // Arrange: mock chat completion
        var mockChat = new Mock<IChatCompletionService>();
        mockChat.Setup(c => c.GetChatMessageContentAsync(
            It.IsAny<ChatHistory>(),
            It.IsAny<PromptExecutionSettings>(),
            It.IsAny<Kernel>(),
            It.IsAny<CancellationToken>()))
        .ReturnsAsync(new ChatMessageContent(
            AuthorRole.Assistant,
            "Your order ORD-123 is currently in transit."));

        var kernel = Kernel.CreateBuilder()
            .Build();

        // Manually add the mock chat service
        kernel.Services.GetRequiredService<IChatCompletionService>();

        // ... (Kernel with mock)
        // In practice, use IKernelBuilder overrides

        var service = new OrderAiService(kernel, mockChat.Object,
            Mock.Of<ILogger<OrderAiService>>());

        // Act
        var history = new ChatHistory();
        var result  = await service.ChatAsync("user-1", "Where is my order?", history);

        // Assert
        Assert.NotEmpty(result);
        Assert.Equal(3, history.Count);  // system + user + assistant
    }

    [Fact]
    public async Task ClassifyIntent_OrderTracking_ReturnsCorrectIntent()
    {
        // Test with actual pattern matching (no AI call needed)
        var service = new OrderIntentClassifier();  // rule-based fallback

        var intent = service.Classify("Can you tell me where my order #1234 is?");

        Assert.Equal("track_order", intent.Intent);
        Assert.Equal("1234", intent.OrderId);
    }
}

// Deterministic rule-based classifier for testing
public class OrderIntentClassifier
{
    public OrderIntent Classify(string message)
    {
        var lower = message.ToLower();
        var orderId = ExtractOrderId(message);

        if (lower.Contains("track") || lower.Contains("where") || lower.Contains("status"))
            return new OrderIntent("track_order", orderId, "medium", 0.9);

        if (lower.Contains("cancel"))
            return new OrderIntent("cancel_order", orderId, "high", 0.9);

        if (lower.Contains("return") || lower.Contains("refund"))
            return new OrderIntent("return_item", orderId, "medium", 0.85);

        return new OrderIntent("general_question", null, "low", 0.5);
    }

    private static string? ExtractOrderId(string message)
    {
        var match = System.Text.RegularExpressions.Regex.Match(message, @"#?(\d{4,})");
        return match.Success ? match.Groups[1].Value : null;
    }
}
```

---

## Step 1625: Production AI Checklist

```
AI Integration Production Checklist
═══════════════════════════════════════════════════════════

Semantic Kernel
✅ Kernel.CreateBuilder() per request (not singleton)
✅ Kernel is registered as transient/scoped
✅ Plugin functions have clear [Description] annotations
✅ MaxAutoInvokeAttempts bounded (prevent infinite loops)
✅ Temperature tuned: 0.0-0.3 for factual, 0.7-1.0 for creative
✅ MaxTokens set to prevent runaway responses
✅ Tool execution timeout configured

Safety & Reliability
✅ Never return raw AI output for financial operations
✅ Validate AI-generated data before acting on it
✅ Human-in-the-loop for irreversible actions (cancellation)
✅ Content filtering / safety classifier on output
✅ Rate limit AI endpoints (expensive per token)
✅ Circuit breaker on AI provider calls

MCP Server
✅ All tool descriptions are accurate and unambiguous
✅ Tools return structured JSON (not prose)
✅ Error messages are actionable (not "error occurred")
✅ Authentication on MCP endpoint
✅ Input validation before calling business logic
✅ Resources return appropriate MIME types

RAG / Memory
✅ Chunking strategy matches content type
✅ Overlap between chunks (avoid splitting context)
✅ Relevance threshold to filter noise (>= 0.7)
✅ Re-ranking for large result sets
✅ Index freshness: re-index when source data changes
✅ Attribution: cite source when answering from memory

Observability
✅ Log every AI call: model, tokens, latency
✅ Track tool invocations and their results
✅ Monitor rejection/safety filter rate
✅ Cost tracking per user/feature
✅ Response quality metrics (user feedback)
```

---

**Part 61 ครอบคลุม:**
- **Semantic Kernel**: kernel setup, plugins ด้วย `[KernelFunction]`, auto tool calling
- **RAG**: semantic memory indexing, chunking ด้วย `TextChunker`, retrieval + generation
- **Agents**: multi-agent conversations ด้วย `AgentGroupChat`
- **MCP Server**: expose .NET APIs เป็น AI tools ด้วย `[McpServerTool]`, resources, prompts
- **MCP Client**: consume MCP server จาก .NET, wire tools เข้า Semantic Kernel
- **Structured Output**: JSON mode, intent classification
- **Streaming**: SSE streaming ด้วย `GetStreamingChatMessageContentsAsync`
