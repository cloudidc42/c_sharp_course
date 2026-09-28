# Part 96: AI Integration - Semantic Kernel, ML.NET, และ OpenAI

## Steps 2165-2180

การนำ AI มาใช้ใน .NET applications ด้วย Semantic Kernel สำหรับ LLM orchestration, ML.NET สำหรับ machine learning, และ Azure OpenAI/OpenAI API integration

---

## Step 2165: Semantic Kernel Overview

```
Semantic Kernel Architecture
==============================

┌─────────────────────────────────────────────┐
│                Your Application             │
├─────────────────────────────────────────────┤
│              Semantic Kernel                │
│  ┌──────────┐ ┌─────────┐ ┌─────────────┐  │
│  │  Kernel  │ │Planner  │ │  Memory     │  │
│  │          │ │(AI Plan)│ │ (Vector DB) │  │
│  └──────────┘ └─────────┘ └─────────────┘  │
│  ┌──────────────────────────────────────┐   │
│  │           Plugins (Skills)            │  │
│  │  Native Functions + Prompt Functions  │  │
│  └──────────────────────────────────────┘  │
├─────────────────────────────────────────────┤
│           AI Connectors                     │
│  OpenAI │ Azure OpenAI │ Ollama │ HuggingFace│
├─────────────────────────────────────────────┤
│           Memory Stores                     │
│  Chroma │ Qdrant │ Redis │ Azure AI Search  │
└─────────────────────────────────────────────┘
```

---

## Step 2166: Semantic Kernel Setup

```csharp
// Package installation
// dotnet add package Microsoft.SemanticKernel
// dotnet add package Microsoft.SemanticKernel.Connectors.OpenAI
// dotnet add package Microsoft.SemanticKernel.Plugins.Core

// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Add Semantic Kernel
builder.Services.AddKernel()
    .AddOpenAIChatCompletion(
        modelId: "gpt-4o",
        apiKey: builder.Configuration["OpenAI:ApiKey"]!)
    .AddOpenAITextEmbeddingGeneration(
        modelId: "text-embedding-3-small",
        apiKey: builder.Configuration["OpenAI:ApiKey"]!);

// Or Azure OpenAI
builder.Services.AddKernel()
    .AddAzureOpenAIChatCompletion(
        deploymentName: "gpt-4o",
        endpoint: builder.Configuration["AzureOpenAI:Endpoint"]!,
        apiKey: builder.Configuration["AzureOpenAI:ApiKey"]!);

// Register plugins
builder.Services.AddSingleton<OrderPlugin>();
builder.Services.AddSingleton<CustomerPlugin>();

var app = builder.Build();
```

---

## Step 2167: Kernel Plugins - Native Functions

```csharp
// Plugins/OrderPlugin.cs
using Microsoft.SemanticKernel;
using System.ComponentModel;

public class OrderPlugin(IOrderRepository orderRepo, ILogger<OrderPlugin> logger)
{
    [KernelFunction("get_order")]
    [Description("Retrieves order details by order ID")]
    [return: Description("Order details including status, items, and total")]
    public async Task<string> GetOrderAsync(
        [Description("The unique identifier of the order")] Guid orderId,
        CancellationToken ct = default)
    {
        var order = await orderRepo.GetByIdAsync(orderId, ct);
        if (order is null)
            return $"Order {orderId} not found";

        return $"""
            Order ID: {order.Id}
            Status: {order.Status}
            Customer: {order.CustomerId}
            Total: {order.Total.Amount} {order.Total.Currency}
            Items: {order.Items.Count}
            Created: {order.CreatedAt:f}
            """;
    }

    [KernelFunction("list_customer_orders")]
    [Description("Lists recent orders for a customer")]
    public async Task<string> ListCustomerOrdersAsync(
        [Description("Customer ID")] Guid customerId,
        [Description("Maximum number of orders to return (default: 5)")] int limit = 5,
        CancellationToken ct = default)
    {
        var orders = await orderRepo.GetByCustomerIdAsync(customerId, limit, ct);
        if (!orders.Any())
            return $"No orders found for customer {customerId}";

        var summary = orders.Select(o =>
            $"- Order {o.Id}: {o.Status} | {o.Total.Amount:C} | {o.CreatedAt:d}");

        return $"Recent orders for customer {customerId}:\n{string.Join("\n", summary)}";
    }

    [KernelFunction("cancel_order")]
    [Description("Cancels an order if it is still in pending or processing status")]
    public async Task<string> CancelOrderAsync(
        [Description("Order ID to cancel")] Guid orderId,
        [Description("Reason for cancellation")] string reason,
        CancellationToken ct = default)
    {
        try
        {
            await orderRepo.CancelAsync(orderId, reason, ct);
            logger.LogInformation("Order {OrderId} cancelled via AI assistant: {Reason}", orderId, reason);
            return $"Order {orderId} has been successfully cancelled. Reason: {reason}";
        }
        catch (InvalidOperationException ex)
        {
            return $"Cannot cancel order {orderId}: {ex.Message}";
        }
    }
}
```

---

## Step 2168: Prompt Functions และ Templates

```csharp
// Plugins/Prompts/AnalyzeOrder/skprompt.txt
// (Semantic Kernel auto-discovers these files)

Analyze the following customer order and provide insights:

Order Information:
{{$order_info}}

Customer History:
{{$customer_history}}

Please provide:
1. Order summary in 2-3 sentences
2. Any anomalies or concerns (unusual patterns, potential fraud indicators)
3. Recommended follow-up actions
4. Customer satisfaction risk score (1-10)

Respond in JSON format with keys: summary, concerns, actions, risk_score

// OrderAnalysisPlugin.cs
public class OrderAnalysisPlugin
{
    private readonly Kernel _kernel;

    public OrderAnalysisPlugin(Kernel kernel)
    {
        _kernel = kernel;
        // Load prompt from file
        _kernel.ImportPluginFromPromptDirectory("Plugins/Prompts");
    }

    [KernelFunction]
    [Description("Analyzes an order for anomalies and recommendations")]
    public async Task<OrderAnalysis> AnalyzeOrderAsync(
        string orderInfo, string customerHistory, CancellationToken ct = default)
    {
        var result = await _kernel.InvokeAsync(
            "AnalyzeOrder", "analyze_order",
            new KernelArguments
            {
                ["order_info"] = orderInfo,
                ["customer_history"] = customerHistory
            }, ct);

        var json = result.GetValue<string>()!;
        return JsonSerializer.Deserialize<OrderAnalysis>(json)!;
    }
}

// Inline prompt function
public async Task<string> SummarizeReviewsAsync(IEnumerable<string> reviews)
{
    var prompt = """
        Summarize the following customer reviews in 3 bullet points highlighting
        key themes and overall sentiment:
        
        {{$reviews}}
        
        Bullet points:
        """;

    var function = _kernel.CreateFunctionFromPrompt(
        prompt,
        new PromptExecutionSettings
        {
            ExtensionData = new Dictionary<string, object>
            {
                ["temperature"] = 0.3,
                ["max_tokens"] = 500
            }
        });

    var result = await _kernel.InvokeAsync(function, new KernelArguments
    {
        ["reviews"] = string.Join("\n---\n", reviews)
    });

    return result.GetValue<string>()!;
}
```

---

## Step 2169: Chat Completion และ Conversation History

```csharp
// Services/AiAssistantService.cs
public class AiAssistantService(Kernel kernel)
{
    public async IAsyncEnumerable<string> ChatAsync(
        string userMessage,
        ChatHistory? history = null,
        [EnumeratorCancellation] CancellationToken ct = default)
    {
        history ??= new ChatHistory("""
            You are a helpful e-commerce assistant. You can help customers:
            - Check order status and details
            - Cancel orders (if eligible)
            - Find products and recommendations
            - Handle returns and refunds inquiries
            
            Always be polite and professional. If you cannot perform an action,
            explain why and suggest alternatives.
            """);

        history.AddUserMessage(userMessage);

        var chatService = kernel.GetRequiredService<IChatCompletionService>();
        var settings = new OpenAIPromptExecutionSettings
        {
            Temperature = 0.7,
            MaxTokens = 1000,
            ToolCallBehavior = ToolCallBehavior.AutoInvokeKernelFunctions
        };

        var fullResponse = new StringBuilder();
        await foreach (var chunk in chatService.GetStreamingChatMessageContentsAsync(
            history, settings, kernel, ct))
        {
            if (chunk.Content is { Length: > 0 })
            {
                yield return chunk.Content;
                fullResponse.Append(chunk.Content);
            }
        }

        history.AddAssistantMessage(fullResponse.ToString());
    }
}

// Controller
[ApiController]
[Route("api/ai")]
public class AiController(AiAssistantService assistant) : ControllerBase
{
    private static readonly ConcurrentDictionary<string, ChatHistory> _sessions = new();

    [HttpPost("chat")]
    public async IAsyncEnumerable<string> Chat(
        [FromBody] ChatRequest request,
        [EnumeratorCancellation] CancellationToken ct)
    {
        var history = _sessions.GetOrAdd(request.SessionId, _ => new ChatHistory());

        Response.Headers.ContentType = "text/event-stream";
        await foreach (var chunk in assistant.ChatAsync(request.Message, history, ct))
        {
            yield return $"data: {JsonSerializer.Serialize(chunk)}\n\n";
        }
        yield return "data: [DONE]\n\n";
    }
}

public record ChatRequest(string SessionId, string Message);
```

---

## Step 2170: Semantic Memory และ RAG Pattern

```csharp
// Retrieval-Augmented Generation (RAG)
// Package: Microsoft.SemanticKernel.Connectors.Qdrant

// Program.cs - Memory configuration
builder.Services.AddSingleton<IMemoryStore>(sp =>
{
    return new QdrantMemoryStore(
        host: "localhost",
        port: 6333,
        vectorSize: 1536);  // text-embedding-3-small dimension
});

builder.Services.AddSingleton<SemanticTextMemory>(sp =>
{
    var memoryStore = sp.GetRequiredService<IMemoryStore>();
    var embeddings = sp.GetRequiredService<ITextEmbeddingGenerationService>();
    return new SemanticTextMemory(memoryStore, embeddings);
});

// DocumentIndexingService.cs
public class DocumentIndexingService(SemanticTextMemory memory)
{
    private const string CollectionName = "product-docs";

    public async Task IndexProductAsync(Product product, CancellationToken ct = default)
    {
        var text = $"""
            Product: {product.Name}
            Category: {product.Category}
            Description: {product.Description}
            Features: {string.Join(", ", product.Features)}
            Price: {product.Price}
            SKU: {product.Sku}
            """;

        await memory.SaveInformationAsync(
            collection: CollectionName,
            text: text,
            id: product.Id.ToString(),
            description: product.Name,
            additionalMetadata: JsonSerializer.Serialize(new
            {
                product.Id,
                product.Name,
                product.Price,
                product.Category
            }),
            cancellationToken: ct);
    }

    public async Task<IEnumerable<ProductSearchResult>> SearchProductsAsync(
        string query, int limit = 5, CancellationToken ct = default)
    {
        var results = await memory.SearchAsync(
            collection: CollectionName,
            query: query,
            limit: limit,
            minRelevanceScore: 0.7,
            cancellationToken: ct);

        return results.Select(r => new ProductSearchResult
        {
            Id = Guid.Parse(r.Metadata.Id),
            Name = r.Metadata.Description,
            RelevanceScore = r.Relevance,
            Metadata = r.Metadata.AdditionalMetadata
        });
    }
}

// RAG-powered product search endpoint
[KernelFunction("search_products")]
[Description("Searches for products based on natural language query")]
public async Task<string> SearchProductsAsync(
    [Description("Natural language search query")] string query,
    CancellationToken ct = default)
{
    var results = await _documentIndexing.SearchProductsAsync(query, 5, ct);

    if (!results.Any())
        return "No matching products found.";

    var productList = results.Select((r, i) =>
        $"{i + 1}. {r.Name} (Relevance: {r.RelevanceScore:P0})\n   {r.Metadata}");

    return $"Found {results.Count()} products matching '{query}':\n{string.Join("\n", productList)}";
}
```

---

## Step 2171: Function Calling และ Auto-Invocation

```csharp
// Register multiple plugins on kernel
var kernel = app.Services.GetRequiredService<Kernel>();
kernel.ImportPluginFromObject(app.Services.GetRequiredService<OrderPlugin>(), "Orders");
kernel.ImportPluginFromObject(app.Services.GetRequiredService<CustomerPlugin>(), "Customers");
kernel.ImportPluginFromObject(app.Services.GetRequiredService<ProductPlugin>(), "Products");

// Auto function calling - AI decides which functions to call
var settings = new OpenAIPromptExecutionSettings
{
    ToolCallBehavior = ToolCallBehavior.AutoInvokeKernelFunctions,
    Temperature = 0,
    MaxTokens = 2000
};

var history = new ChatHistory("You are a customer service agent with access to order, customer, and product data.");
history.AddUserMessage("Can you tell me about order abc123 and whether the customer has ordered before?");

var chatService = kernel.GetRequiredService<IChatCompletionService>();
var response = await chatService.GetChatMessageContentAsync(history, settings, kernel);

// AI automatically called:
// 1. Orders.get_order("abc123")
// 2. Customers.get_customer_history(customerId)
// And composed the answer from the results

Console.WriteLine(response.Content);

// Manual function invocation
var result = await kernel.InvokeAsync(
    pluginName: "Orders",
    functionName: "get_order",
    arguments: new KernelArguments { ["orderId"] = orderId });

var orderInfo = result.GetValue<string>();
```

---

## Step 2172: ML.NET - Sentiment Analysis

```csharp
// Package: Microsoft.ML
// Package: Microsoft.ML.FastTree

// Models/SentimentData.cs
public class ReviewData
{
    [LoadColumn(0)]
    public string ReviewText { get; set; } = null!;

    [LoadColumn(1), ColumnName("Label")]
    public bool IsPositive { get; set; }
}

public class ReviewPrediction
{
    [ColumnName("PredictedLabel")]
    public bool IsPositive { get; set; }

    public float Probability { get; set; }
    public float Score { get; set; }
}

// ML/SentimentModel.cs
public class SentimentAnalysisModel
{
    private PredictionEngine<ReviewData, ReviewPrediction>? _engine;
    private readonly string _modelPath;

    public SentimentAnalysisModel(string modelPath)
    {
        _modelPath = modelPath;
    }

    public void Train(IEnumerable<ReviewData> trainingData)
    {
        var mlContext = new MLContext(seed: 42);

        var data = mlContext.Data.LoadFromEnumerable(trainingData);
        var trainTestSplit = mlContext.Data.TrainTestSplit(data, testFraction: 0.2);

        var pipeline = mlContext.Transforms.Text
            .FeaturizeText("Features", nameof(ReviewData.ReviewText))
            .Append(mlContext.BinaryClassification.Trainers.FastTree(
                labelColumnName: "Label",
                featureColumnName: "Features",
                numberOfTrees: 100,
                numberOfLeaves: 20,
                minimumExampleCountPerLeaf: 10));

        var model = pipeline.Fit(trainTestSplit.TrainSet);

        // Evaluate
        var predictions = model.Transform(trainTestSplit.TestSet);
        var metrics = mlContext.BinaryClassification.Evaluate(predictions);

        Console.WriteLine($"Accuracy: {metrics.Accuracy:P2}");
        Console.WriteLine($"AUC: {metrics.AreaUnderRocCurve:P2}");
        Console.WriteLine($"F1: {metrics.F1Score:P2}");

        // Save model
        mlContext.Model.Save(model, data.Schema, _modelPath);
    }

    public void Load()
    {
        var mlContext = new MLContext();
        var model = mlContext.Model.Load(_modelPath, out _);
        _engine = mlContext.Model.CreatePredictionEngine<ReviewData, ReviewPrediction>(model);
    }

    public ReviewPrediction Predict(string reviewText)
    {
        if (_engine is null)
            throw new InvalidOperationException("Model not loaded");

        return _engine.Predict(new ReviewData { ReviewText = reviewText });
    }
}
```

---

## Step 2173: ML.NET - Product Recommendation

```csharp
// Models/ProductInteraction.cs
public class ProductInteraction
{
    public float UserId { get; set; }
    public float ProductId { get; set; }
    public float Rating { get; set; }
}

public class ProductPrediction
{
    public float Score { get; set; }
}

// ML/RecommendationModel.cs
public class ProductRecommendationModel
{
    private readonly MLContext _mlContext = new(seed: 0);
    private ITransformer? _model;

    public void Train(IEnumerable<ProductInteraction> interactions)
    {
        var data = _mlContext.Data.LoadFromEnumerable(interactions);
        var trainTest = _mlContext.Data.TrainTestSplit(data, 0.2);

        var options = new MatrixFactorizationTrainer.Options
        {
            MatrixColumnIndexColumnName = nameof(ProductInteraction.UserId),
            MatrixRowIndexColumnName = nameof(ProductInteraction.ProductId),
            LabelColumnName = nameof(ProductInteraction.Rating),
            NumberOfIterations = 20,
            ApproximationRank = 100
        };

        var pipeline = _mlContext.Transforms
            .Conversion.MapValueToKey(
                nameof(ProductInteraction.UserId), nameof(ProductInteraction.UserId))
            .Append(_mlContext.Transforms.Conversion.MapValueToKey(
                nameof(ProductInteraction.ProductId), nameof(ProductInteraction.ProductId)))
            .Append(_mlContext.Recommendation().Trainers.MatrixFactorization(options));

        _model = pipeline.Fit(trainTest.TrainSet);

        var predictions = _model.Transform(trainTest.TestSet);
        var metrics = _mlContext.Regression.Evaluate(predictions,
            labelColumnName: nameof(ProductInteraction.Rating));

        Console.WriteLine($"R-Squared: {metrics.RSquared:0.##}");
        Console.WriteLine($"RMSE: {metrics.RootMeanSquaredError:0.##}");
    }

    public IEnumerable<(int ProductId, float Score)> GetRecommendations(
        int userId, IEnumerable<int> candidateProductIds, int top = 10)
    {
        var predictionEngine = _mlContext.Model
            .CreatePredictionEngine<ProductInteraction, ProductPrediction>(_model!);

        return candidateProductIds
            .Select(productId => (
                ProductId: productId,
                Score: predictionEngine.Predict(new ProductInteraction
                {
                    UserId = userId,
                    ProductId = productId
                }).Score))
            .OrderByDescending(x => x.Score)
            .Take(top);
    }
}
```

---

## Step 2174: Structured Output with OpenAI

```csharp
// Using OpenAI SDK directly for structured output
// Package: OpenAI

public class ProductExtractor(OpenAIClient client)
{
    // JSON Schema-based structured output
    public async Task<ExtractedProduct> ExtractFromDescriptionAsync(
        string description, CancellationToken ct = default)
    {
        var chatClient = client.GetChatClient("gpt-4o");

        var response = await chatClient.CompleteChatAsync(
            [
                ChatMessage.CreateSystemMessage(
                    "Extract product information from the description. Be precise and concise."),
                ChatMessage.CreateUserMessage(description)
            ],
            new ChatCompletionOptions
            {
                ResponseFormat = ChatResponseFormat.CreateJsonSchemaFormat(
                    name: "extracted_product",
                    jsonSchema: BinaryData.FromString("""
                    {
                        "type": "object",
                        "properties": {
                            "name": { "type": "string" },
                            "category": { "type": "string" },
                            "price": { "type": "number" },
                            "features": {
                                "type": "array",
                                "items": { "type": "string" }
                            },
                            "target_audience": { "type": "string" },
                            "sentiment": {
                                "type": "string",
                                "enum": ["positive", "neutral", "negative"]
                            }
                        },
                        "required": ["name", "category", "features", "sentiment"],
                        "additionalProperties": false
                    }
                    """),
                    strictSchemaEnabled: true)
            }, ct);

        var json = response.Value.Content[0].Text;
        return JsonSerializer.Deserialize<ExtractedProduct>(json)!;
    }
}

public record ExtractedProduct(
    string Name,
    string Category,
    decimal? Price,
    List<string> Features,
    string? TargetAudience,
    string Sentiment);
```

---

## Step 2175: Embedding-Based Search

```csharp
// Services/SemanticSearchService.cs
public class SemanticSearchService(
    ITextEmbeddingGenerationService embeddingService,
    IVectorStore vectorStore)
{
    public async Task IndexProductsAsync(
        IEnumerable<Product> products, CancellationToken ct = default)
    {
        var collection = vectorStore.GetCollection<Guid, ProductDocument>("products");
        await collection.CreateCollectionIfNotExistsAsync(ct);

        var batch = products.Select(p => new ProductDocument
        {
            Id = p.Id,
            Name = p.Name,
            Description = p.Description,
            Category = p.Category,
            Price = (float)p.Price.Amount
        }).ToList();

        // Generate embeddings in batch
        var texts = batch.Select(p => $"{p.Name} {p.Description}").ToList();
        var embeddings = await embeddingService.GenerateEmbeddingsAsync(texts, cancellationToken: ct);

        for (int i = 0; i < batch.Count; i++)
            batch[i].Embedding = embeddings[i];

        await collection.UpsertBatchAsync(batch, ct).ToListAsync(ct);
    }

    public async Task<IEnumerable<ProductDocument>> SearchAsync(
        string query, int top = 5, CancellationToken ct = default)
    {
        var collection = vectorStore.GetCollection<Guid, ProductDocument>("products");

        var queryEmbedding = await embeddingService.GenerateEmbeddingAsync(query, cancellationToken: ct);

        var searchResults = await collection.VectorizedSearchAsync(
            queryEmbedding,
            new VectorSearchOptions { Top = top, IncludeVectors = false },
            ct);

        var results = new List<ProductDocument>();
        await foreach (var result in searchResults.Results.WithCancellation(ct))
            results.Add(result.Record);

        return results;
    }
}

// Vector document model
public class ProductDocument
{
    [VectorStoreRecordKey]
    public Guid Id { get; set; }

    [VectorStoreRecordData(IsFilterable = true)]
    public string Name { get; set; } = null!;

    [VectorStoreRecordData]
    public string Description { get; set; } = null!;

    [VectorStoreRecordData(IsFilterable = true)]
    public string Category { get; set; } = null!;

    [VectorStoreRecordData]
    public float Price { get; set; }

    [VectorStoreRecordVector(Dimensions: 1536, DistanceFunction.CosineSimilarity)]
    public ReadOnlyMemory<float> Embedding { get; set; }
}
```

---

## Step 2176: AI-Powered Order Processing

```csharp
// Services/IntelligentOrderService.cs
public class IntelligentOrderService(
    Kernel kernel,
    IOrderRepository orderRepo,
    IFraudDetectionService fraudDetection,
    ILogger<IntelligentOrderService> logger)
{
    public async Task<OrderProcessingResult> ProcessOrderIntelligentlyAsync(
        CreateOrderCommand command, CancellationToken ct = default)
    {
        // Step 1: Fraud detection with ML
        var fraudScore = await fraudDetection.GetFraudScoreAsync(command, ct);
        if (fraudScore > 0.8)
        {
            logger.LogWarning("High fraud risk {Score} for order from {CustomerId}",
                fraudScore, command.CustomerId);
            return new OrderProcessingResult
            {
                Success = false,
                Reason = "Order flagged for manual review due to risk assessment"
            };
        }

        // Step 2: AI-powered address validation and correction
        var validatedAddress = await ValidateAddressWithAiAsync(command.ShippingAddress, ct);

        // Step 3: Create order
        var order = await orderRepo.CreateAsync(
            command with { ShippingAddress = validatedAddress }, ct);

        // Step 4: AI-powered customer communication
        var confirmationMessage = await GenerateConfirmationMessageAsync(order, ct);
        await SendConfirmationAsync(command.CustomerEmail, confirmationMessage, ct);

        return new OrderProcessingResult
        {
            Success = true,
            OrderId = order.Id,
            FraudScore = fraudScore
        };
    }

    private async Task<string> ValidateAddressWithAiAsync(
        AddressDto address, CancellationToken ct)
    {
        var function = kernel.CreateFunctionFromPrompt("""
            Validate and correct the following Thai address if needed.
            Return only the corrected address in the same format.
            If address is valid, return it unchanged.
            
            Address: {{$address}}
            """);

        var result = await kernel.InvokeAsync(function,
            new KernelArguments { ["address"] = address.ToString() }, ct);

        return result.GetValue<string>()!;
    }

    private async Task<string> GenerateConfirmationMessageAsync(
        Order order, CancellationToken ct)
    {
        var function = kernel.CreateFunctionFromPrompt("""
            Generate a friendly order confirmation message in Thai for:
            Order ID: {{$orderId}}
            Items: {{$items}}
            Total: {{$total}}
            Estimated delivery: {{$delivery}}
            
            Keep it concise, warm, and include the order number prominently.
            """);

        var result = await kernel.InvokeAsync(function, new KernelArguments
        {
            ["orderId"] = order.Id.ToString()[..8].ToUpper(),
            ["items"] = string.Join(", ", order.Items.Select(i => $"{i.ProductName} x{i.Quantity}")),
            ["total"] = $"{order.Total.Amount:N0} บาท",
            ["delivery"] = DateTime.Now.AddDays(3).ToString("d MMMM yyyy", new System.Globalization.CultureInfo("th-TH"))
        }, ct);

        return result.GetValue<string>()!;
    }
}
```

---

## Step 2177: AI Content Moderation

```csharp
// Services/ContentModerationService.cs
public class ContentModerationService(Kernel kernel)
{
    public async Task<ModerationResult> ModerateReviewAsync(
        string reviewText, CancellationToken ct = default)
    {
        var function = kernel.CreateFunctionFromPrompt(
            """
            Analyze this product review for policy violations. Check for:
            - Spam or promotional content
            - Hate speech or offensive language
            - Personal information (phone numbers, addresses, emails)
            - Fake or incentivized reviews
            
            Review: {{$review}}
            
            Respond in JSON:
            {
              "approved": boolean,
              "violations": string[],
              "cleaned_text": "string (review with PII removed, or null if rejected)",
              "confidence": number (0-1)
            }
            """,
            new PromptExecutionSettings { ExtensionData = { ["temperature"] = 0 } });

        var result = await kernel.InvokeAsync(function,
            new KernelArguments { ["review"] = reviewText }, ct);

        return JsonSerializer.Deserialize<ModerationResult>(result.GetValue<string>()!)!;
    }
}

public record ModerationResult(
    bool Approved,
    List<string> Violations,
    string? CleanedText,
    double Confidence);
```

---

## Step 2178: Planner - Multi-Step AI Reasoning

```csharp
// Using Handlebars planner for complex tasks
// Package: Microsoft.SemanticKernel.Planners.Handlebars

public class OrderFulfillmentPlanner(Kernel kernel)
{
    public async Task<string> FulfillOrderWithPlanAsync(
        string fulfillmentRequest, CancellationToken ct = default)
    {
        var planner = new HandlebarsPlanner(new HandlebarsPlannerOptions
        {
            MaxTokens = 4000,
            AllowLoops = true
        });

        // AI creates a plan combining multiple plugin functions
        var plan = await planner.CreatePlanAsync(kernel, fulfillmentRequest, ct);

        Console.WriteLine("Generated Plan:");
        Console.WriteLine(plan);

        // Execute the plan
        var result = await plan.InvokeAsync(kernel, new KernelArguments(), ct);
        return result;
    }
}

// Example request:
// "Check if order ORD-123 exists, verify the customer hasn't exceeded their credit limit,
//  check inventory for all items, and if everything is OK, confirm the order and 
//  send a confirmation email."
//
// AI will automatically plan:
// 1. Call Orders.get_order("ORD-123")
// 2. Call Customers.check_credit_limit(customerId, orderTotal)
// 3. For each item, call Inventory.check_stock(productId, quantity)
// 4. If all checks pass, call Orders.confirm_order("ORD-123")
// 5. Call Communications.send_email(customerEmail, confirmationTemplate)
```

---

## Step 2179: Streaming Responses

```csharp
// API endpoint with SSE streaming
app.MapPost("/api/ai/analyze-order", async (
    AnalyzeOrderRequest request,
    AiAssistantService assistant,
    CancellationToken ct) =>
{
    return Results.Stream(async stream =>
    {
        var writer = new StreamWriter(stream) { AutoFlush = true };

        await foreach (var chunk in assistant.AnalyzeOrderStreamAsync(request.OrderId, ct))
        {
            await writer.WriteAsync($"data: {JsonSerializer.Serialize(new { text = chunk })}\n\n");
        }

        await writer.WriteAsync("data: [DONE]\n\n");
    }, contentType: "text/event-stream");
});

// Blazor component consuming SSE
@code {
    private string _analysis = "";
    private bool _isLoading;

    private async Task AnalyzeOrder(string orderId)
    {
        _isLoading = true;
        _analysis = "";

        using var http = new HttpClient();
        using var response = await http.PostAsJsonAsync(
            "/api/ai/analyze-order", new { OrderId = orderId });

        using var stream = await response.Content.ReadAsStreamAsync();
        using var reader = new StreamReader(stream);

        while (!reader.EndOfStream)
        {
            var line = await reader.ReadLineAsync();
            if (line?.StartsWith("data: ") == true && !line.Contains("[DONE]"))
            {
                var data = JsonSerializer.Deserialize<JsonElement>(line[6..]);
                _analysis += data.GetProperty("text").GetString();
                StateHasChanged();
            }
        }
        _isLoading = false;
    }
}
```

---

## Step 2180: AI Integration Best Practices

```csharp
// 1. Token usage tracking
public class TokenTrackingFilter(ILogger<TokenTrackingFilter> logger, IMetricsService metrics)
    : IPromptRenderFilter
{
    public async Task OnPromptRenderAsync(PromptRenderContext context, Func<PromptRenderContext, Task> next)
    {
        await next(context);

        // Log rendered prompt for debugging (not in production!)
        if (logger.IsEnabled(LogLevel.Debug))
            logger.LogDebug("Rendered prompt: {Prompt}", context.RenderedPrompt);
    }
}

public class FunctionInvocationFilter(ILogger<FunctionInvocationFilter> logger, IMetricsService metrics)
    : IFunctionInvocationFilter
{
    public async Task OnFunctionInvocationAsync(
        FunctionInvocationContext context, Func<FunctionInvocationContext, Task> next)
    {
        var sw = Stopwatch.StartNew();
        try
        {
            await next(context);
            
            if (context.Result.Metadata?.TryGetValue("Usage", out var usage) == true
                && usage is CompletionsUsage completionsUsage)
            {
                metrics.RecordTokenUsage(
                    context.Function.Name,
                    completionsUsage.InputTokenCount ?? 0,
                    completionsUsage.OutputTokenCount ?? 0);

                logger.LogInformation(
                    "AI function {Function}: {InputTokens} input, {OutputTokens} output tokens in {Duration}ms",
                    context.Function.Name,
                    completionsUsage.InputTokenCount,
                    completionsUsage.OutputTokenCount,
                    sw.ElapsedMilliseconds);
            }
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "AI function {Function} failed after {Duration}ms",
                context.Function.Name, sw.ElapsedMilliseconds);
            throw;
        }
    }
}

// Register filters
builder.Services.AddSingleton<IFunctionInvocationFilter, FunctionInvocationFilter>();
builder.Services.AddSingleton<IPromptRenderFilter, TokenTrackingFilter>();

// 2. Cost management - cache embeddings
public class CachedEmbeddingService(
    ITextEmbeddingGenerationService inner, IDistributedCache cache)
    : ITextEmbeddingGenerationService
{
    public IReadOnlyDictionary<string, object?> Attributes => inner.Attributes;

    public async Task<IList<ReadOnlyMemory<float>>> GenerateEmbeddingsAsync(
        IList<string> data, Kernel? kernel = null, CancellationToken ct = default)
    {
        var results = new ReadOnlyMemory<float>[data.Count];
        var uncachedIndices = new List<int>();

        for (int i = 0; i < data.Count; i++)
        {
            var cacheKey = $"emb:{Convert.ToHexString(SHA256.HashData(Encoding.UTF8.GetBytes(data[i])))[..16]}";
            var cached = await cache.GetAsync(cacheKey, ct);
            if (cached is not null)
                results[i] = MemoryMarshal.Cast<byte, float>(cached).ToArray();
            else
                uncachedIndices.Add(i);
        }

        if (uncachedIndices.Count > 0)
        {
            var uncachedTexts = uncachedIndices.Select(i => data[i]).ToList();
            var embeddings = await inner.GenerateEmbeddingsAsync(uncachedTexts, kernel, ct);

            for (int i = 0; i < uncachedIndices.Count; i++)
            {
                var cacheKey = $"emb:{Convert.ToHexString(SHA256.HashData(Encoding.UTF8.GetBytes(uncachedTexts[i])))[..16]}";
                var bytes = MemoryMarshal.Cast<float, byte>(embeddings[i].Span).ToArray();
                await cache.SetAsync(cacheKey, bytes,
                    new DistributedCacheEntryOptions { SlidingExpiration = TimeSpan.FromDays(7) }, ct);
                results[uncachedIndices[i]] = embeddings[i];
            }
        }

        return results;
    }
}
```

---

## สรุป Part 96

AI Integration ใน .NET ด้วย Semantic Kernel:
- **Semantic Kernel** เป็น orchestration layer สำหรับ LLMs
- **Plugins** ให้ AI เรียกใช้ business functions ได้
- **RAG** (Retrieval-Augmented Generation) เพิ่ม context จาก vector database
- **Structured Output** รับผลลัพธ์ที่ typed และ validated
- **ML.NET** สำหรับ on-premise ML (sentiment, recommendation)
- **Streaming** ให้ UX ที่ responsive สำหรับ long-running AI calls
- **Filters** สำหรับ observability, cost tracking, และ safety
- **Cached embeddings** ลด API costs อย่างมีนัยสำคัญ
