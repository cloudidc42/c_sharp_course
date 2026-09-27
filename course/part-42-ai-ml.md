# Part 42: AI/ML Integration with .NET (Steps 1211-1250)

## Step 1211: ภาพรวม AI/ML Ecosystem ใน .NET

### ทำไม .NET ถึงเหมาะกับ AI/ML?

```
┌─────────────────────────────────────────────────────────────────┐
│                   .NET AI/ML Ecosystem                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐ │
│  │   ML.NET     │  │ ONNX Runtime │  │  Semantic Kernel     │ │
│  │  (Training + │  │  (Inference) │  │  (LLM Orchestration) │ │
│  │  Inference)  │  │              │  │                      │ │
│  └──────────────┘  └──────────────┘  └──────────────────────┘ │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐ │
│  │  Microsoft   │  │  Azure AI    │  │    OpenAI SDK        │ │
│  │ .Extensions  │  │  Services    │  │    for .NET          │ │
│  │    .AI       │  │              │  │                      │ │
│  └──────────────┘  └──────────────┘  └──────────────────────┘ │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              Vector Databases                            │  │
│  │  (Qdrant, Pinecone, Azure AI Search, PostgreSQL pgvector)│  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Use Cases หลัก

| Use Case | เครื่องมือแนะนำ | Package |
|----------|----------------|---------|
| Classification/Regression | ML.NET | `Microsoft.ML` |
| Image Recognition | ML.NET + TensorFlow | `Microsoft.ML.TensorFlow` |
| NLP Text Analysis | ML.NET | `Microsoft.ML` |
| Pre-trained Model Inference | ONNX Runtime | `Microsoft.ML.OnnxRuntime` |
| LLM Integration | Semantic Kernel | `Microsoft.SemanticKernel` |
| Chat Completions | OpenAI SDK | `OpenAI` |
| Embeddings + Search | Semantic Kernel | `Microsoft.SemanticKernel.Connectors.*` |
| Structured AI | Microsoft.Extensions.AI | `Microsoft.Extensions.AI` |

---

## Step 1212: ML.NET - Machine Learning Framework

### ติดตั้ง Package

```xml
<PackageReference Include="Microsoft.ML" Version="3.0.0" />
<PackageReference Include="Microsoft.ML.AutoML" Version="0.21.0" />
<PackageReference Include="Microsoft.ML.FastTree" Version="3.0.0" />
<PackageReference Include="Microsoft.ML.LightGbm" Version="3.0.0" />
```

### ML Pipeline Architecture

```
┌─────────┐    ┌──────────────┐    ┌───────────┐    ┌──────────┐
│  Data   │───▶│  Transform   │───▶│  Trainer  │───▶│  Model   │
│ Loading │    │  Pipeline    │    │ Algorithm │    │ Evaluate │
└─────────┘    └──────────────┘    └───────────┘    └──────────┘
     │                │                  │                │
     ▼                ▼                  ▼                ▼
  LoadFrom       Normalize,          FastTree,        R², RMSE,
  CSV/DB         Encode, Hash        LightGbm,        AUC, F1
                                     SDCA, etc.
```

### Binary Classification - Sentiment Analysis

```csharp
using Microsoft.ML;
using Microsoft.ML.Data;

// Input data model
public class SentimentData
{
    [LoadColumn(0)]
    public string SentimentText { get; set; } = string.Empty;

    [LoadColumn(1), ColumnName("Label")]
    public bool Sentiment { get; set; }
}

// Prediction output
public class SentimentPrediction : SentimentData
{
    [ColumnName("PredictedLabel")]
    public bool PredictedLabel { get; set; }

    public float Probability { get; set; }
    public float Score { get; set; }
}

public class SentimentAnalyzer
{
    private readonly MLContext _mlContext;
    private ITransformer? _model;

    public SentimentAnalyzer()
    {
        _mlContext = new MLContext(seed: 42);
    }

    public void Train(string dataPath)
    {
        // 1. Load data
        var dataView = _mlContext.Data.LoadFromTextFile<SentimentData>(
            dataPath,
            hasHeader: true,
            separatorChar: '\t');

        // 2. Split train/test
        var splitData = _mlContext.Data.TrainTestSplit(dataView, testFraction: 0.2);

        // 3. Build pipeline
        var pipeline = _mlContext.Transforms.Text
            .FeaturizeText(
                outputColumnName: "Features",
                inputColumnName: nameof(SentimentData.SentimentText))
            .Append(_mlContext.BinaryClassification.Trainers
                .SdcaLogisticRegression(
                    labelColumnName: "Label",
                    featureColumnName: "Features"));

        // 4. Train
        Console.WriteLine("Training model...");
        _model = pipeline.Fit(splitData.TrainSet);

        // 5. Evaluate
        var predictions = _model.Transform(splitData.TestSet);
        var metrics = _mlContext.BinaryClassification
            .Evaluate(predictions, labelColumnName: "Label");

        Console.WriteLine($"Accuracy: {metrics.Accuracy:P2}");
        Console.WriteLine($"AUC: {metrics.AreaUnderRocCurve:P2}");
        Console.WriteLine($"F1 Score: {metrics.F1Score:P2}");
    }

    public SentimentPrediction Predict(string text)
    {
        if (_model is null) throw new InvalidOperationException("Model not trained");

        var engine = _mlContext.Model
            .CreatePredictionEngine<SentimentData, SentimentPrediction>(_model);

        return engine.Predict(new SentimentData { SentimentText = text });
    }

    public void SaveModel(string modelPath)
    {
        _mlContext.Model.Save(_model!, null, modelPath);
    }

    public void LoadModel(string modelPath)
    {
        _model = _mlContext.Model.Load(modelPath, out _);
    }
}
```

---

## Step 1213: ML.NET - Regression and Multi-class Classification

### House Price Prediction (Regression)

```csharp
public class HouseData
{
    public float Size { get; set; }       // sq meters
    public float Bedrooms { get; set; }
    public float Bathrooms { get; set; }
    public float Age { get; set; }
    public float Distance { get; set; }  // km from city center
    public float Price { get; set; }     // label
}

public class HousePricePrediction
{
    [ColumnName("Score")]
    public float Price { get; set; }
}

public class HousePriceModel
{
    private readonly MLContext _mlContext = new(seed: 42);

    public void TrainAndEvaluate(IEnumerable<HouseData> data)
    {
        var dataView = _mlContext.Data.LoadFromEnumerable(data);
        var split = _mlContext.Data.TrainTestSplit(dataView, testFraction: 0.2);

        var pipeline = _mlContext.Transforms
            .Concatenate("Features",
                nameof(HouseData.Size),
                nameof(HouseData.Bedrooms),
                nameof(HouseData.Bathrooms),
                nameof(HouseData.Age),
                nameof(HouseData.Distance))
            .Append(_mlContext.Transforms.NormalizeMinMax("Features"))
            .Append(_mlContext.Regression.Trainers.FastTree(
                labelColumnName: nameof(HouseData.Price),
                numberOfLeaves: 20,
                numberOfTrees: 100,
                minimumExampleCountPerLeaf: 10,
                learningRate: 0.2));

        var model = pipeline.Fit(split.TrainSet);
        var predictions = model.Transform(split.TestSet);
        var metrics = _mlContext.Regression.Evaluate(predictions,
            labelColumnName: nameof(HouseData.Price));

        Console.WriteLine($"R²: {metrics.RSquared:F4}");
        Console.WriteLine($"RMSE: {metrics.RootMeanSquaredError:F2}");
        Console.WriteLine($"MAE: {metrics.MeanAbsoluteError:F2}");
    }
}
```

### Multi-class Classification - Product Category

```csharp
public class ProductData
{
    public string Description { get; set; } = string.Empty;
    public string Category { get; set; } = string.Empty;  // label
}

public class ProductPrediction
{
    [ColumnName("PredictedLabel")]
    public string? Category { get; set; }

    public float[]? Score { get; set; }
}

public class ProductClassifier
{
    private readonly MLContext _mlContext = new(seed: 42);

    public ITransformer Train(IEnumerable<ProductData> data)
    {
        var dataView = _mlContext.Data.LoadFromEnumerable(data);

        var pipeline = _mlContext.Transforms.Conversion
            .MapValueToKey("Label", nameof(ProductData.Category))
            .Append(_mlContext.Transforms.Text
                .FeaturizeText("Features", nameof(ProductData.Description)))
            .Append(_mlContext.MulticlassClassification.Trainers
                .SdcaMaximumEntropy("Label", "Features"))
            .Append(_mlContext.Transforms.Conversion
                .MapKeyToValue("PredictedLabel"));

        return pipeline.Fit(dataView);
    }
}
```

---

## Step 1214: AutoML - Automated Machine Learning

```csharp
using Microsoft.ML.AutoML;

public class AutoMLExample
{
    private readonly MLContext _mlContext = new(seed: 42);

    public async Task RunAutoML(string dataPath)
    {
        var data = _mlContext.Data.LoadFromTextFile<SentimentData>(
            dataPath, hasHeader: true, separatorChar: '\t');

        var split = _mlContext.Data.TrainTestSplit(data, testFraction: 0.2);

        // AutoML experiment
        var experiment = _mlContext.Auto()
            .CreateBinaryClassificationExperiment(new BinaryExperimentSettings
            {
                MaxExperimentTimeInSeconds = 60,
                OptimizingMetric = BinaryClassificationMetric.AreaUnderRocCurve,
            });

        Console.WriteLine("Running AutoML experiment...");
        var result = await experiment.ExecuteAsync(
            split.TrainSet,
            split.TestSet,
            labelColumnName: "Label",
            progressHandler: new BinaryExperimentProgressHandler());

        Console.WriteLine($"\nBest algorithm: {result.BestRun.TrainerName}");
        Console.WriteLine($"Best AUC: {result.BestRun.ValidationMetrics.AreaUnderRocCurve:P2}");

        // Get best model
        var bestModel = result.BestRun.Model;
    }
}

public class BinaryExperimentProgressHandler
    : IProgress<RunDetail<BinaryClassificationMetrics>>
{
    private int _iteration;

    public void Report(RunDetail<BinaryClassificationMetrics> value)
    {
        if (value.ValidationMetrics is not null)
        {
            Console.WriteLine($"Iteration {++_iteration}: " +
                $"Trainer={value.TrainerName,-35} " +
                $"AUC={value.ValidationMetrics.AreaUnderRocCurve:P2}");
        }
    }
}
```

---

## Step 1215: ONNX Runtime - Cross-Platform Inference

### ทำไมต้องใช้ ONNX?

```
Python Training (PyTorch/TensorFlow)
         │
         ▼
   Export to ONNX
         │
         ▼
ONNX Runtime (.NET) ──── GPU Acceleration (CUDA/DirectML)
         │
         ▼
  Production API
```

### ติดตั้ง

```xml
<PackageReference Include="Microsoft.ML.OnnxRuntime" Version="1.18.0" />
<!-- GPU support: -->
<PackageReference Include="Microsoft.ML.OnnxRuntime.Gpu" Version="1.18.0" />
```

### Image Classification with ONNX

```csharp
using Microsoft.ML.OnnxRuntime;
using Microsoft.ML.OnnxRuntime.Tensors;

public class ImageClassifier : IDisposable
{
    private readonly InferenceSession _session;
    private readonly string[] _labels;

    public ImageClassifier(string modelPath, string[] labels)
    {
        var options = new SessionOptions
        {
            ExecutionMode = ExecutionMode.ORT_PARALLEL,
            InterOpNumThreads = Environment.ProcessorCount,
        };

        // Enable GPU if available
        // options.AppendExecutionProvider_CUDA();
        // options.AppendExecutionProvider_DirectML();

        _session = new InferenceSession(modelPath, options);
        _labels = labels;
    }

    public (string Label, float Confidence) Classify(float[] imageData, int height, int width)
    {
        // Create input tensor [batch=1, channels=3, height, width]
        var dimensions = new long[] { 1, 3, height, width };
        var tensor = new DenseTensor<float>(imageData, dimensions);

        var inputs = new List<NamedOnnxValue>
        {
            NamedOnnxValue.CreateFromTensor("input", tensor)
        };

        using var results = _session.Run(inputs);
        var output = results.First().AsEnumerable<float>().ToArray();

        // Softmax
        var softmax = Softmax(output);
        var maxIdx = Array.IndexOf(softmax, softmax.Max());

        return (_labels[maxIdx], softmax[maxIdx]);
    }

    private static float[] Softmax(float[] scores)
    {
        var maxScore = scores.Max();
        var expScores = scores.Select(s => MathF.Exp(s - maxScore)).ToArray();
        var sumExp = expScores.Sum();
        return expScores.Select(e => e / sumExp).ToArray();
    }

    public void Dispose() => _session.Dispose();
}
```

### NLP with BERT ONNX

```csharp
public class BertTextClassifier : IDisposable
{
    private readonly InferenceSession _session;
    private readonly BertTokenizer _tokenizer;
    private const int MaxSequenceLength = 128;

    public BertTextClassifier(string modelPath, string vocabPath)
    {
        _session = new InferenceSession(modelPath);
        _tokenizer = new BertTokenizer(vocabPath);
    }

    public float[] GetEmbedding(string text)
    {
        var tokens = _tokenizer.Tokenize(text, MaxSequenceLength);

        // Input IDs
        var inputIds = new DenseTensor<long>(
            tokens.InputIds.Select(x => (long)x).ToArray(),
            new long[] { 1, MaxSequenceLength });

        // Attention mask
        var attentionMask = new DenseTensor<long>(
            tokens.AttentionMask.Select(x => (long)x).ToArray(),
            new long[] { 1, MaxSequenceLength });

        // Token type IDs
        var tokenTypeIds = new DenseTensor<long>(
            new long[MaxSequenceLength],
            new long[] { 1, MaxSequenceLength });

        var inputs = new List<NamedOnnxValue>
        {
            NamedOnnxValue.CreateFromTensor("input_ids", inputIds),
            NamedOnnxValue.CreateFromTensor("attention_mask", attentionMask),
            NamedOnnxValue.CreateFromTensor("token_type_ids", tokenTypeIds),
        };

        using var results = _session.Run(inputs);

        // [CLS] token embedding (first token = sentence representation)
        var lastHiddenState = results.First()
            .AsEnumerable<float>()
            .ToArray();

        // Extract [CLS] embedding: first 768 values
        return lastHiddenState[..768];
    }

    public void Dispose() => _session.Dispose();
}
```

---

## Step 1216: Microsoft.Extensions.AI - Unified AI Interface

### ทำไมต้องใช้ Microsoft.Extensions.AI?

```
Without MEA:                    With MEA:
─────────────────               ─────────────────────────────
OpenAI SDK                      IChatClient (abstraction)
Azure OpenAI SDK       ──────▶      ├── OpenAI
Ollama Client                       ├── Azure OpenAI
Anthropic SDK                       ├── Ollama
Custom LLM                          └── Any provider
(different APIs!)                (unified interface)
```

### ติดตั้ง

```xml
<PackageReference Include="Microsoft.Extensions.AI" Version="9.0.0" />
<PackageReference Include="Microsoft.Extensions.AI.OpenAI" Version="9.0.0" />
<PackageReference Include="Microsoft.Extensions.AI.Ollama" Version="9.0.0" />
```

### IChatClient Interface

```csharp
using Microsoft.Extensions.AI;

// Unified interface - works with any provider
public class AIChatService(IChatClient chatClient)
{
    public async Task<string> GetResponseAsync(
        string userMessage,
        CancellationToken ct = default)
    {
        var response = await chatClient.CompleteAsync(
            userMessage,
            cancellationToken: ct);

        return response.Message.Text ?? string.Empty;
    }

    public async IAsyncEnumerable<string> StreamResponseAsync(
        string userMessage,
        [EnumeratorCancellation] CancellationToken ct = default)
    {
        await foreach (var update in chatClient.CompleteStreamingAsync(
            userMessage, cancellationToken: ct))
        {
            if (update.Text is not null)
                yield return update.Text;
        }
    }

    public async Task<string> ChatWithHistoryAsync(
        IList<ChatMessage> history,
        string newMessage,
        CancellationToken ct = default)
    {
        history.Add(new ChatMessage(ChatRole.User, newMessage));

        var response = await chatClient.CompleteAsync(history, cancellationToken: ct);
        var assistantMessage = response.Message;

        history.Add(assistantMessage);
        return assistantMessage.Text ?? string.Empty;
    }
}
```

### DI Registration

```csharp
// Program.cs
builder.Services.AddChatClient(builder =>
{
    // Provider 1: OpenAI
    builder.Use(new OpenAIClient(openAIKey)
        .AsChatClient("gpt-4o-mini"));

    // Provider 2: Azure OpenAI
    // builder.Use(new AzureOpenAIClient(...).AsChatClient("gpt-4o"));

    // Provider 3: Ollama (local)
    // builder.Use(new OllamaChatClient("http://localhost:11434", "llama3.2"));

    // Add middleware
    builder.UseLogging();
    builder.UseFunctionInvocation();
});
```

---

## Step 1217: OpenAI SDK for .NET

### ติดตั้ง

```xml
<PackageReference Include="OpenAI" Version="2.0.0" />
<PackageReference Include="Azure.AI.OpenAI" Version="2.0.0" />
```

### Chat Completions

```csharp
using OpenAI;
using OpenAI.Chat;

public class OpenAIService
{
    private readonly ChatClient _chatClient;

    public OpenAIService(IConfiguration config)
    {
        var apiKey = config["OpenAI:ApiKey"]
            ?? throw new InvalidOperationException("OpenAI API key not configured");

        var client = new OpenAIClient(apiKey);
        _chatClient = client.GetChatClient("gpt-4o-mini");
    }

    public async Task<string> CompleteChatAsync(
        string systemPrompt,
        string userMessage)
    {
        var messages = new List<ChatMessage>
        {
            new SystemChatMessage(systemPrompt),
            new UserChatMessage(userMessage),
        };

        var completion = await _chatClient.CompleteChatAsync(messages);
        return completion.Value.Content[0].Text;
    }

    public async IAsyncEnumerable<string> StreamChatAsync(
        string systemPrompt,
        string userMessage,
        [EnumeratorCancellation] CancellationToken ct = default)
    {
        var messages = new List<ChatMessage>
        {
            new SystemChatMessage(systemPrompt),
            new UserChatMessage(userMessage),
        };

        await foreach (var update in _chatClient
            .CompleteChatStreamingAsync(messages)
            .WithCancellation(ct))
        {
            foreach (var part in update.ContentUpdate)
            {
                if (!string.IsNullOrEmpty(part.Text))
                    yield return part.Text;
            }
        }
    }

    // Structured output with JSON Schema
    public async Task<T?> GetStructuredOutputAsync<T>(
        string prompt,
        JsonSerializerOptions? options = null) where T : class
    {
        var schemaOptions = new ChatCompletionOptions
        {
            ResponseFormat = ChatResponseFormat.CreateJsonSchemaFormat(
                jsonSchemaFormatName: typeof(T).Name,
                jsonSchema: BinaryData.FromString(
                    GetJsonSchema<T>())),
        };

        var messages = new List<ChatMessage>
        {
            new UserChatMessage(prompt)
        };

        var completion = await _chatClient.CompleteChatAsync(messages, schemaOptions);
        var json = completion.Value.Content[0].Text;

        return JsonSerializer.Deserialize<T>(json, options);
    }

    private static string GetJsonSchema<T>()
    {
        // Use source generation or reflection to get JSON schema
        return $$"""
        {
            "type": "object",
            "properties": {{GetPropertiesSchema<T>()}},
            "required": {{GetRequiredProperties<T>()}}
        }
        """;
    }

    private static string GetPropertiesSchema<T>() => "{}"; // simplified
    private static string GetRequiredProperties<T>() => "[]"; // simplified
}
```

### Function Calling (Tools)

```csharp
public class WeatherAssistant
{
    private readonly ChatClient _chatClient;

    public WeatherAssistant(ChatClient chatClient)
    {
        _chatClient = chatClient;
    }

    public async Task<string> GetWeatherResponseAsync(string userQuery)
    {
        var tools = new List<ChatTool>
        {
            ChatTool.CreateFunctionTool(
                functionName: "get_weather",
                functionDescription: "Get current weather for a location",
                functionParameters: BinaryData.FromString("""
                {
                    "type": "object",
                    "properties": {
                        "location": {
                            "type": "string",
                            "description": "City name, e.g. 'Bangkok, TH'"
                        },
                        "unit": {
                            "type": "string",
                            "enum": ["celsius", "fahrenheit"]
                        }
                    },
                    "required": ["location"]
                }
                """)),
        };

        var messages = new List<ChatMessage>
        {
            new UserChatMessage(userQuery)
        };

        // Agentic loop
        while (true)
        {
            var options = new ChatCompletionOptions();
            foreach (var tool in tools) options.Tools.Add(tool);

            var completion = await _chatClient.CompleteChatAsync(messages, options);

            if (completion.Value.FinishReason == ChatFinishReason.ToolCalls)
            {
                messages.Add(new AssistantChatMessage(completion.Value));

                foreach (var toolCall in completion.Value.ToolCalls)
                {
                    var result = await ExecuteToolAsync(toolCall);
                    messages.Add(new ToolChatMessage(toolCall.Id, result));
                }
            }
            else
            {
                return completion.Value.Content[0].Text;
            }
        }
    }

    private async Task<string> ExecuteToolAsync(ChatToolCall toolCall)
    {
        if (toolCall.FunctionName == "get_weather")
        {
            using var doc = JsonDocument.Parse(toolCall.FunctionArguments);
            var location = doc.RootElement.GetProperty("location").GetString()!;
            var unit = doc.RootElement.TryGetProperty("unit", out var u)
                ? u.GetString()
                : "celsius";

            // Call real weather API
            return await GetWeatherDataAsync(location, unit ?? "celsius");
        }

        return "Tool not found";
    }

    private Task<string> GetWeatherDataAsync(string location, string unit)
    {
        // Simulate weather API
        return Task.FromResult($$"""
        {"location": "{{location}}", "temperature": 32, "unit": "{{unit}}", "condition": "Sunny"}
        """);
    }
}
```

---

## Step 1218: Embeddings and Vector Search

### สร้าง Embeddings

```csharp
using OpenAI.Embeddings;

public class EmbeddingService
{
    private readonly EmbeddingClient _embeddingClient;

    public EmbeddingService(OpenAIClient client)
    {
        _embeddingClient = client.GetEmbeddingClient("text-embedding-3-small");
    }

    public async Task<float[]> GetEmbeddingAsync(string text)
    {
        var result = await _embeddingClient.GenerateEmbeddingAsync(text);
        return result.Value.ToFloats().ToArray();
    }

    public async Task<IReadOnlyList<float[]>> GetEmbeddingsAsync(
        IEnumerable<string> texts)
    {
        var result = await _embeddingClient.GenerateEmbeddingsAsync(texts.ToList());
        return result.Value.Select(e => e.ToFloats().ToArray()).ToList();
    }

    // Cosine similarity
    public static float CosineSimilarity(float[] a, float[] b)
    {
        if (a.Length != b.Length)
            throw new ArgumentException("Vectors must have same dimension");

        float dotProduct = 0, normA = 0, normB = 0;
        for (int i = 0; i < a.Length; i++)
        {
            dotProduct += a[i] * b[i];
            normA += a[i] * a[i];
            normB += b[i] * b[i];
        }

        return dotProduct / (MathF.Sqrt(normA) * MathF.Sqrt(normB));
    }
}
```

### In-Memory Vector Store

```csharp
public class DocumentChunk
{
    public required string Id { get; init; }
    public required string Content { get; init; }
    public required string Source { get; init; }
    public required float[] Embedding { get; init; }
    public Dictionary<string, string> Metadata { get; init; } = [];
}

public class InMemoryVectorStore
{
    private readonly List<DocumentChunk> _chunks = [];
    private readonly EmbeddingService _embeddingService;

    public InMemoryVectorStore(EmbeddingService embeddingService)
    {
        _embeddingService = embeddingService;
    }

    public async Task AddDocumentAsync(string content, string source)
    {
        var chunks = SplitIntoChunks(content, chunkSize: 500, overlap: 50);

        foreach (var (chunkContent, index) in chunks.Select((c, i) => (c, i)))
        {
            var embedding = await _embeddingService.GetEmbeddingAsync(chunkContent);

            _chunks.Add(new DocumentChunk
            {
                Id = $"{source}_{index}",
                Content = chunkContent,
                Source = source,
                Embedding = embedding,
            });
        }
    }

    public async Task<IReadOnlyList<DocumentChunk>> SearchAsync(
        string query,
        int topK = 5,
        float minScore = 0.7f)
    {
        var queryEmbedding = await _embeddingService.GetEmbeddingAsync(query);

        return _chunks
            .Select(chunk => (
                Chunk: chunk,
                Score: EmbeddingService.CosineSimilarity(queryEmbedding, chunk.Embedding)))
            .Where(x => x.Score >= minScore)
            .OrderByDescending(x => x.Score)
            .Take(topK)
            .Select(x => x.Chunk)
            .ToList();
    }

    private static IEnumerable<string> SplitIntoChunks(
        string text, int chunkSize, int overlap)
    {
        var words = text.Split(' ');
        var step = chunkSize - overlap;

        for (int i = 0; i < words.Length; i += step)
        {
            var chunk = string.Join(" ", words.Skip(i).Take(chunkSize));
            if (!string.IsNullOrWhiteSpace(chunk))
                yield return chunk;
        }
    }
}
```

---

## Step 1219: Semantic Kernel - LLM Orchestration

### ติดตั้ง

```xml
<PackageReference Include="Microsoft.SemanticKernel" Version="1.30.0" />
<PackageReference Include="Microsoft.SemanticKernel.Connectors.OpenAI" Version="1.30.0" />
<PackageReference Include="Microsoft.SemanticKernel.Connectors.AzureOpenAI" Version="1.30.0" />
<PackageReference Include="Microsoft.SemanticKernel.Memory" Version="1.30.0-alpha" />
```

### Kernel Setup and Plugins

```csharp
using Microsoft.SemanticKernel;
using Microsoft.SemanticKernel.ChatCompletion;
using Microsoft.SemanticKernel.Connectors.OpenAI;

public class SemanticKernelService
{
    private readonly Kernel _kernel;

    public SemanticKernelService(IConfiguration config)
    {
        var builder = Kernel.CreateBuilder();

        // Add AI services
        builder.AddOpenAIChatCompletion(
            modelId: "gpt-4o-mini",
            apiKey: config["OpenAI:ApiKey"]!);

        // Add plugins
        builder.Plugins.AddFromType<TimePlugin>();
        builder.Plugins.AddFromType<WeatherPlugin>();

        // Add logging
        builder.Services.AddLogging(l => l.AddConsole());

        _kernel = builder.Build();
    }

    // Semantic Function (prompt template)
    public async Task<string> SummarizeAsync(string text)
    {
        var summarize = _kernel.CreateFunctionFromPrompt(
            """
            Summarize the following text in 3 bullet points:
            
            {{$input}}
            
            Summary:
            """,
            new OpenAIPromptExecutionSettings
            {
                MaxTokens = 200,
                Temperature = 0.3,
            });

        var result = await _kernel.InvokeAsync(summarize,
            new KernelArguments { ["input"] = text });

        return result.GetValue<string>() ?? string.Empty;
    }

    // Chat with auto function calling
    public async Task<string> ChatWithToolsAsync(string userMessage)
    {
        var chatService = _kernel.GetRequiredService<IChatCompletionService>();

        var history = new ChatHistory();
        history.AddSystemMessage(
            "You are a helpful assistant. Use available tools to answer questions.");
        history.AddUserMessage(userMessage);

        var settings = new OpenAIPromptExecutionSettings
        {
            ToolCallBehavior = ToolCallBehavior.AutoInvokeKernelFunctions,
        };

        var response = await chatService.GetChatMessageContentAsync(
            history, settings, _kernel);

        return response.Content ?? string.Empty;
    }
}

// Native Plugin
public class TimePlugin
{
    [KernelFunction("get_current_time")]
    [Description("Returns the current date and time")]
    public string GetCurrentTime() =>
        DateTime.UtcNow.ToString("yyyy-MM-dd HH:mm:ss UTC");

    [KernelFunction("get_time_in_timezone")]
    [Description("Returns the current time in specified timezone")]
    public string GetTimeInTimezone(
        [Description("IANA timezone name, e.g. 'Asia/Bangkok'")] string timezone)
    {
        var tz = TimeZoneInfo.FindSystemTimeZoneById(timezone);
        return TimeZoneInfo.ConvertTimeFromUtc(DateTime.UtcNow, tz)
            .ToString("yyyy-MM-dd HH:mm:ss");
    }
}

public class WeatherPlugin
{
    private readonly HttpClient _httpClient;

    public WeatherPlugin(IHttpClientFactory factory)
    {
        _httpClient = factory.CreateClient();
    }

    [KernelFunction("get_weather")]
    [Description("Gets current weather for a city")]
    public async Task<string> GetWeatherAsync(
        [Description("City name")] string city)
    {
        // Real implementation would call a weather API
        await Task.Delay(10); // simulate API call
        return $"Weather in {city}: 28°C, Partly cloudy";
    }
}
```

---

## Step 1220: RAG (Retrieval-Augmented Generation) Pattern

### RAG Architecture

```
User Query
    │
    ▼
┌──────────────┐      ┌──────────────────────────────────┐
│   Embed      │──────▶     Vector Database              │
│  Query       │      │  (semantic search, top K chunks) │
└──────────────┘      └─────────────┬────────────────────┘
                                    │
                                    ▼
                        ┌──────────────────────┐
                        │   Build Context      │
                        │  (query + chunks)    │
                        └──────────┬───────────┘
                                   │
                                   ▼
                        ┌──────────────────────┐
                        │    LLM (GPT-4o)      │
                        │  "Answer based on    │
                        │   this context..."   │
                        └──────────┬───────────┘
                                   │
                                   ▼
                             Final Answer
```

### RAG Implementation

```csharp
public class RagChatbot
{
    private readonly InMemoryVectorStore _vectorStore;
    private readonly ChatClient _chatClient;
    private const string SystemPrompt = """
        You are a helpful assistant. Answer questions based ONLY on the provided context.
        If the context doesn't contain enough information to answer, say so clearly.
        Always cite the source of your information.
        """;

    public RagChatbot(InMemoryVectorStore vectorStore, ChatClient chatClient)
    {
        _vectorStore = vectorStore;
        _chatClient = chatClient;
    }

    public async Task LoadDocumentsAsync(IEnumerable<(string content, string source)> documents)
    {
        foreach (var (content, source) in documents)
        {
            await _vectorStore.AddDocumentAsync(content, source);
        }
    }

    public async Task<RagResponse> AskAsync(string question)
    {
        // 1. Retrieve relevant context
        var relevantChunks = await _vectorStore.SearchAsync(question, topK: 5);

        if (!relevantChunks.Any())
        {
            return new RagResponse
            {
                Answer = "I couldn't find relevant information to answer your question.",
                Sources = [],
                ContextUsed = [],
            };
        }

        // 2. Build context
        var contextBuilder = new StringBuilder();
        contextBuilder.AppendLine("CONTEXT:");
        foreach (var (chunk, i) in relevantChunks.Select((c, i) => (c, i + 1)))
        {
            contextBuilder.AppendLine($"[{i}] Source: {chunk.Source}");
            contextBuilder.AppendLine(chunk.Content);
            contextBuilder.AppendLine();
        }

        // 3. Generate answer with context
        var messages = new List<ChatMessage>
        {
            new SystemChatMessage(SystemPrompt),
            new UserChatMessage($"{contextBuilder}\n\nQUESTION: {question}"),
        };

        var completion = await _chatClient.CompleteChatAsync(messages);
        var answer = completion.Value.Content[0].Text;

        return new RagResponse
        {
            Answer = answer,
            Sources = relevantChunks.Select(c => c.Source).Distinct().ToList(),
            ContextUsed = relevantChunks.Select(c => c.Content).ToList(),
        };
    }
}

public class RagResponse
{
    public required string Answer { get; init; }
    public required IReadOnlyList<string> Sources { get; init; }
    public required IReadOnlyList<string> ContextUsed { get; init; }
}
```

---

## Step 1221: Qdrant Vector Database Integration

### ติดตั้ง

```xml
<PackageReference Include="Qdrant.Client" Version="1.10.0" />
```

### Qdrant Client Setup

```csharp
using Qdrant.Client;
using Qdrant.Client.Grpc;

public class QdrantVectorStore
{
    private readonly QdrantClient _client;
    private readonly EmbeddingService _embeddingService;
    private const string CollectionName = "documents";
    private const int VectorSize = 1536; // text-embedding-3-small

    public QdrantVectorStore(
        QdrantClient client,
        EmbeddingService embeddingService)
    {
        _client = client;
        _embeddingService = embeddingService;
    }

    public async Task EnsureCollectionExistsAsync()
    {
        var collections = await _client.ListCollectionsAsync();
        if (!collections.Any(c => c == CollectionName))
        {
            await _client.CreateCollectionAsync(
                CollectionName,
                new VectorsConfig
                {
                    Params = new VectorParams
                    {
                        Size = VectorSize,
                        Distance = Distance.Cosine,
                    }
                });

            // Create payload index for filtering
            await _client.CreatePayloadIndexAsync(
                CollectionName,
                "source",
                PayloadSchemaType.Keyword);
        }
    }

    public async Task UpsertAsync(
        string id,
        string content,
        string source,
        Dictionary<string, string>? metadata = null)
    {
        var embedding = await _embeddingService.GetEmbeddingAsync(content);

        var point = new PointStruct
        {
            Id = new PointId { Uuid = id },
            Vectors = embedding,
        };

        point.Payload["content"] = new Value { StringValue = content };
        point.Payload["source"] = new Value { StringValue = source };

        if (metadata is not null)
        {
            foreach (var (key, value) in metadata)
                point.Payload[key] = new Value { StringValue = value };
        }

        await _client.UpsertAsync(CollectionName, [point]);
    }

    public async Task<IReadOnlyList<SearchResult>> SearchAsync(
        string query,
        int limit = 5,
        string? sourceFilter = null)
    {
        var queryEmbedding = await _embeddingService.GetEmbeddingAsync(query);

        Filter? filter = sourceFilter is not null
            ? new Filter
            {
                Must =
                {
                    new Condition
                    {
                        Field = new FieldCondition
                        {
                            Key = "source",
                            Match = new Match
                            {
                                Keyword = sourceFilter
                            }
                        }
                    }
                }
            }
            : null;

        var results = await _client.SearchAsync(
            CollectionName,
            queryEmbedding,
            limit: (ulong)limit,
            filter: filter,
            withPayload: true);

        return results.Select(r => new SearchResult(
            Id: r.Id.Uuid,
            Score: r.Score,
            Content: r.Payload["content"].StringValue,
            Source: r.Payload["source"].StringValue)).ToList();
    }
}

public record SearchResult(string Id, float Score, string Content, string Source);
```

### DI Setup for Vector Store

```csharp
// Program.cs
builder.Services.AddSingleton<QdrantClient>(sp =>
{
    var config = sp.GetRequiredService<IConfiguration>();
    return new QdrantClient(
        host: config["Qdrant:Host"] ?? "localhost",
        port: int.Parse(config["Qdrant:Port"] ?? "6334"),
        https: false);
});

builder.Services.AddSingleton<QdrantVectorStore>();
```

---

## Step 1222: Azure OpenAI Integration

### ติดตั้ง

```xml
<PackageReference Include="Azure.AI.OpenAI" Version="2.0.0" />
<PackageReference Include="Azure.Identity" Version="1.12.0" />
```

### Azure OpenAI Service

```csharp
using Azure;
using Azure.AI.OpenAI;
using Azure.Identity;
using OpenAI.Chat;
using OpenAI.Embeddings;

public class AzureOpenAIService
{
    private readonly AzureOpenAIClient _client;
    private readonly ChatClient _chatClient;
    private readonly EmbeddingClient _embeddingClient;

    public AzureOpenAIService(IConfiguration config)
    {
        var endpoint = new Uri(config["AzureOpenAI:Endpoint"]!);

        // Option 1: API Key
        var apiKey = config["AzureOpenAI:ApiKey"];
        if (!string.IsNullOrEmpty(apiKey))
        {
            _client = new AzureOpenAIClient(endpoint, new AzureKeyCredential(apiKey));
        }
        else
        {
            // Option 2: Managed Identity (recommended for production)
            _client = new AzureOpenAIClient(endpoint, new DefaultAzureCredential());
        }

        _chatClient = _client.GetChatClient(config["AzureOpenAI:ChatDeployment"]!);
        _embeddingClient = _client.GetEmbeddingClient(
            config["AzureOpenAI:EmbeddingDeployment"]!);
    }

    public async Task<string> GetCompletionAsync(
        string systemPrompt,
        string userMessage,
        float temperature = 0.7f)
    {
        var options = new ChatCompletionOptions
        {
            Temperature = temperature,
            MaxOutputTokenCount = 1000,
        };

        var messages = new List<ChatMessage>
        {
            new SystemChatMessage(systemPrompt),
            new UserChatMessage(userMessage),
        };

        var response = await _chatClient.CompleteChatAsync(messages, options);
        return response.Value.Content[0].Text;
    }

    public async Task<float[]> GetEmbeddingAsync(string text)
    {
        var response = await _embeddingClient.GenerateEmbeddingAsync(text);
        return response.Value.ToFloats().ToArray();
    }
}
```

### Content Safety Filter

```csharp
using Azure.AI.ContentSafety;

public class ContentSafetyService
{
    private readonly ContentSafetyClient _client;

    public ContentSafetyService(IConfiguration config)
    {
        _client = new ContentSafetyClient(
            new Uri(config["ContentSafety:Endpoint"]!),
            new AzureKeyCredential(config["ContentSafety:ApiKey"]!));
    }

    public async Task<SafetyAnalysisResult> AnalyzeTextAsync(string text)
    {
        var options = new AnalyzeTextOptions(text);
        var response = await _client.AnalyzeTextAsync(options);

        return new SafetyAnalysisResult
        {
            IsSafe = response.Value.CategoriesAnalysis
                .All(c => c.Severity < 2),
            Hate = response.Value.CategoriesAnalysis
                .FirstOrDefault(c => c.Category == TextCategory.Hate)?.Severity ?? 0,
            Violence = response.Value.CategoriesAnalysis
                .FirstOrDefault(c => c.Category == TextCategory.Violence)?.Severity ?? 0,
            Sexual = response.Value.CategoriesAnalysis
                .FirstOrDefault(c => c.Category == TextCategory.Sexual)?.Severity ?? 0,
            SelfHarm = response.Value.CategoriesAnalysis
                .FirstOrDefault(c => c.Category == TextCategory.SelfHarm)?.Severity ?? 0,
        };
    }
}

public class SafetyAnalysisResult
{
    public bool IsSafe { get; init; }
    public int Hate { get; init; }
    public int Violence { get; init; }
    public int Sexual { get; init; }
    public int SelfHarm { get; init; }
}
```

---

## Step 1223: Local LLM with Ollama

### Setup Ollama

```bash
# Install Ollama
curl -fsSL https://ollama.ai/install.sh | sh

# Pull models
ollama pull llama3.2
ollama pull phi3.5
ollama pull mistral
ollama pull nomic-embed-text  # for embeddings
```

### Ollama .NET Client

```csharp
using Microsoft.Extensions.AI;

public class OllamaService
{
    private readonly IChatClient _chatClient;
    private readonly IEmbeddingGenerator<string, Embedding<float>> _embeddingGenerator;

    public OllamaService(IConfiguration config)
    {
        var baseUrl = config["Ollama:BaseUrl"] ?? "http://localhost:11434";
        var model = config["Ollama:ChatModel"] ?? "llama3.2";
        var embeddingModel = config["Ollama:EmbeddingModel"] ?? "nomic-embed-text";

        _chatClient = new OllamaChatClient(new Uri(baseUrl), model);
        _embeddingGenerator = new OllamaEmbeddingGenerator(
            new Uri(baseUrl), embeddingModel);
    }

    public async Task<string> ChatAsync(string message)
    {
        var response = await _chatClient.CompleteAsync(message);
        return response.Message.Text ?? string.Empty;
    }

    public async IAsyncEnumerable<string> StreamChatAsync(
        string message,
        [EnumeratorCancellation] CancellationToken ct = default)
    {
        await foreach (var update in _chatClient
            .CompleteStreamingAsync(message)
            .WithCancellation(ct))
        {
            if (update.Text is not null)
                yield return update.Text;
        }
    }

    public async Task<float[]> GetEmbeddingAsync(string text)
    {
        var result = await _embeddingGenerator.GenerateAsync([text]);
        return result[0].Vector.ToArray();
    }
}
```

### Fallback Pattern (Cloud → Local)

```csharp
public class HybridAIService
{
    private readonly IChatClient _primary;   // OpenAI
    private readonly IChatClient _fallback;  // Ollama

    public HybridAIService(
        [FromKeyedServices("openai")] IChatClient primary,
        [FromKeyedServices("ollama")] IChatClient fallback)
    {
        _primary = primary;
        _fallback = fallback;
    }

    public async Task<string> CompleteAsync(
        string message,
        CancellationToken ct = default)
    {
        try
        {
            var response = await _primary.CompleteAsync(message, cancellationToken: ct);
            return response.Message.Text ?? string.Empty;
        }
        catch (Exception ex) when (
            ex is HttpRequestException or TimeoutException or TaskCanceledException)
        {
            // Fallback to local model
            var response = await _fallback.CompleteAsync(message, cancellationToken: ct);
            return response.Message.Text ?? string.Empty;
        }
    }
}
```

---

## Step 1224: AI-Powered ASP.NET Core API

### Complete AI Chat API

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Add AI services
builder.Services.AddSingleton(sp =>
{
    var config = sp.GetRequiredService<IConfiguration>();
    return new OpenAIClient(config["OpenAI:ApiKey"]!);
});

builder.Services.AddSingleton(sp =>
{
    var client = sp.GetRequiredService<OpenAIClient>();
    return client.GetChatClient("gpt-4o-mini");
});

builder.Services.AddSingleton<EmbeddingService>();
builder.Services.AddSingleton<InMemoryVectorStore>();
builder.Services.AddSingleton<RagChatbot>();

builder.Services.AddOutputCache(options =>
{
    options.AddPolicy("AiResponse", policy =>
    {
        policy.Expire(TimeSpan.FromMinutes(5));
        policy.VaryByValue(ctx =>
            new KeyValuePair<string, string>("q",
                ctx.HttpContext.Request.Query["q"].ToString()));
    });
});

var app = builder.Build();

// Chat endpoint
app.MapPost("/api/chat", async (
    ChatRequest request,
    RagChatbot chatbot,
    CancellationToken ct) =>
{
    if (string.IsNullOrWhiteSpace(request.Message))
        return Results.BadRequest("Message cannot be empty");

    var response = await chatbot.AskAsync(request.Message);
    return Results.Ok(response);
})
.WithName("Chat")
.Produces<RagResponse>()
.ProducesProblem(StatusCodes.Status400BadRequest);

// Streaming chat
app.MapGet("/api/chat/stream", async (
    string message,
    OpenAIService aiService,
    HttpContext ctx,
    CancellationToken ct) =>
{
    ctx.Response.ContentType = "text/event-stream";
    ctx.Response.Headers.CacheControl = "no-cache";

    await foreach (var chunk in aiService.StreamChatAsync(
        "You are a helpful assistant.", message, ct))
    {
        await ctx.Response.WriteAsync($"data: {JsonSerializer.Serialize(new { chunk })}\n\n", ct);
        await ctx.Response.Body.FlushAsync(ct);
    }
});

// Document upload for RAG
app.MapPost("/api/documents", async (
    IFormFile file,
    RagChatbot chatbot) =>
{
    using var reader = new StreamReader(file.OpenReadStream());
    var content = await reader.ReadToEndAsync();
    await chatbot.LoadDocumentsAsync([(content, file.FileName)]);
    return Results.Ok(new { message = "Document indexed successfully" });
})
.DisableAntiforgery();

app.Run();

public record ChatRequest(string Message, string? SessionId = null);
```

---

## Step 1225: AI Background Jobs and Batch Processing

### AI Batch Processing with Channel

```csharp
public class AiBatchProcessor : BackgroundService
{
    private readonly Channel<EmbeddingJob> _channel;
    private readonly EmbeddingService _embeddingService;
    private readonly ILogger<AiBatchProcessor> _logger;
    private const int BatchSize = 100;
    private const int MaxConcurrency = 5;

    public AiBatchProcessor(
        Channel<EmbeddingJob> channel,
        EmbeddingService embeddingService,
        ILogger<AiBatchProcessor> logger)
    {
        _channel = channel;
        _embeddingService = embeddingService;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        var semaphore = new SemaphoreSlim(MaxConcurrency);
        var batch = new List<EmbeddingJob>(BatchSize);

        await foreach (var job in _channel.Reader.ReadAllAsync(stoppingToken))
        {
            batch.Add(job);

            if (batch.Count >= BatchSize)
            {
                await ProcessBatchAsync(batch, semaphore, stoppingToken);
                batch.Clear();
            }
        }

        // Process remaining
        if (batch.Count > 0)
        {
            await ProcessBatchAsync(batch, semaphore, stoppingToken);
        }
    }

    private async Task ProcessBatchAsync(
        IReadOnlyList<EmbeddingJob> jobs,
        SemaphoreSlim semaphore,
        CancellationToken ct)
    {
        var tasks = jobs.Select(async job =>
        {
            await semaphore.WaitAsync(ct);
            try
            {
                var embedding = await _embeddingService.GetEmbeddingAsync(job.Text);
                await job.ResultSource.SetResultAsync(embedding);
                _logger.LogDebug("Processed embedding for job {JobId}", job.Id);
            }
            catch (Exception ex)
            {
                job.ResultSource.SetException(ex);
                _logger.LogError(ex, "Failed to process job {JobId}", job.Id);
            }
            finally
            {
                semaphore.Release();
            }
        });

        await Task.WhenAll(tasks);
    }
}

public class EmbeddingJob
{
    public string Id { get; } = Guid.NewGuid().ToString();
    public required string Text { get; init; }
    public TaskCompletionSource<float[]> ResultSource { get; } = new();
}
```

---

## Step 1226: Semantic Kernel - Memory and Plugins

### Semantic Memory

```csharp
using Microsoft.SemanticKernel.Memory;
using Microsoft.SemanticKernel.Connectors.OpenAI;

public class SemanticMemoryService
{
    private readonly ISemanticTextMemory _memory;

    public SemanticMemoryService(IConfiguration config)
    {
        _memory = new MemoryBuilder()
            .WithOpenAITextEmbeddingGeneration(
                "text-embedding-3-small",
                config["OpenAI:ApiKey"]!)
            .WithMemoryStore(new VolatileMemoryStore()) // In-memory, use Qdrant for production
            .Build();
    }

    public async Task SaveMemoryAsync(
        string collection,
        string id,
        string text,
        string? description = null)
    {
        await _memory.SaveInformationAsync(
            collection: collection,
            text: text,
            id: id,
            description: description);
    }

    public async Task<IReadOnlyList<MemoryQueryResult>> SearchMemoryAsync(
        string collection,
        string query,
        int limit = 5,
        double minRelevance = 0.7)
    {
        var results = new List<MemoryQueryResult>();

        await foreach (var result in _memory.SearchAsync(
            collection, query, limit, minRelevance))
        {
            results.Add(result);
        }

        return results;
    }
}
```

### Advanced Plugin with State

```csharp
public class OrderManagementPlugin
{
    private readonly IOrderRepository _orderRepository;
    private readonly IOrderService _orderService;

    public OrderManagementPlugin(
        IOrderRepository orderRepository,
        IOrderService orderService)
    {
        _orderRepository = orderRepository;
        _orderService = orderService;
    }

    [KernelFunction("get_order_status")]
    [Description("Get the current status of an order by order ID")]
    public async Task<string> GetOrderStatusAsync(
        [Description("The order ID (GUID format)")] string orderId)
    {
        if (!Guid.TryParse(orderId, out var id))
            return "Invalid order ID format";

        var order = await _orderRepository.FindByIdAsync(new OrderId(id));
        if (order is null)
            return $"Order {orderId} not found";

        return $"Order {orderId}: Status={order.Status}, " +
               $"Total={order.TotalAmount}, " +
               $"CreatedAt={order.CreatedAt:yyyy-MM-dd}";
    }

    [KernelFunction("cancel_order")]
    [Description("Cancel an order if it's in Pending or Confirmed status")]
    public async Task<string> CancelOrderAsync(
        [Description("The order ID to cancel")] string orderId,
        [Description("Reason for cancellation")] string reason)
    {
        if (!Guid.TryParse(orderId, out var id))
            return "Invalid order ID format";

        try
        {
            await _orderService.CancelOrderAsync(new OrderId(id), reason);
            return $"Order {orderId} has been cancelled successfully";
        }
        catch (DomainException ex)
        {
            return $"Cannot cancel order: {ex.Message}";
        }
    }

    [KernelFunction("list_recent_orders")]
    [Description("List recent orders for a customer")]
    public async Task<string> ListRecentOrdersAsync(
        [Description("Customer ID (GUID)")] string customerId,
        [Description("Number of orders to return (max 10)")] int count = 5)
    {
        if (!Guid.TryParse(customerId, out var id))
            return "Invalid customer ID format";

        count = Math.Min(count, 10);
        var orders = await _orderRepository.GetRecentByCustomerAsync(
            new CustomerId(id), count);

        if (!orders.Any())
            return "No orders found for this customer";

        var sb = new StringBuilder();
        sb.AppendLine($"Recent {orders.Count} orders:");
        foreach (var order in orders)
        {
            sb.AppendLine($"- {order.Id}: {order.Status} | {order.TotalAmount} | {order.CreatedAt:yyyy-MM-dd}");
        }

        return sb.ToString();
    }
}
```

---

## Step 1227: AI Middleware and Caching

### Response Caching for AI

```csharp
public class CachedChatClient : IChatClient
{
    private readonly IChatClient _inner;
    private readonly IDistributedCache _cache;
    private readonly TimeSpan _cacheDuration;

    public CachedChatClient(
        IChatClient inner,
        IDistributedCache cache,
        TimeSpan? cacheDuration = null)
    {
        _inner = inner;
        _cache = cache;
        _cacheDuration = cacheDuration ?? TimeSpan.FromHours(24);
    }

    public async Task<ChatCompletion> CompleteAsync(
        IList<ChatMessage> chatMessages,
        ChatOptions? options = null,
        CancellationToken cancellationToken = default)
    {
        var cacheKey = ComputeCacheKey(chatMessages, options);

        var cached = await _cache.GetStringAsync(cacheKey, cancellationToken);
        if (cached is not null)
        {
            return JsonSerializer.Deserialize<ChatCompletion>(cached)!;
        }

        var result = await _inner.CompleteAsync(chatMessages, options, cancellationToken);

        await _cache.SetStringAsync(
            cacheKey,
            JsonSerializer.Serialize(result),
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = _cacheDuration,
            },
            cancellationToken);

        return result;
    }

    public IAsyncEnumerable<StreamingChatCompletionUpdate> CompleteStreamingAsync(
        IList<ChatMessage> chatMessages,
        ChatOptions? options = null,
        CancellationToken cancellationToken = default)
    {
        // Streaming responses are not cached
        return _inner.CompleteStreamingAsync(chatMessages, options, cancellationToken);
    }

    private static string ComputeCacheKey(
        IList<ChatMessage> messages,
        ChatOptions? options)
    {
        var content = messages
            .Select(m => $"{m.Role}:{m.Text}")
            .Aggregate((a, b) => $"{a}|{b}");

        var hash = SHA256.HashData(Encoding.UTF8.GetBytes(content));
        return $"ai:chat:{Convert.ToHexString(hash)}";
    }

    public ChatClientMetadata Metadata => _inner.Metadata;

    public TService? GetService<TService>(object? key = null) where TService : class
        => _inner.GetService<TService>(key);

    public void Dispose() => _inner.Dispose();
}
```

### Rate Limiting Middleware for AI

```csharp
public class AiRateLimitingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly RateLimiter _rateLimiter;

    public AiRateLimitingMiddleware(RequestDelegate next)
    {
        _next = next;
        // Token bucket: 100 requests per minute, burst of 20
        _rateLimiter = new TokenBucketRateLimiter(new TokenBucketRateLimiterOptions
        {
            TokenLimit = 20,
            QueueProcessingOrder = QueueProcessingOrder.OldestFirst,
            QueueLimit = 0,
            ReplenishmentPeriod = TimeSpan.FromSeconds(60),
            TokensPerPeriod = 100,
            AutoReplenishment = true,
        });
    }

    public async Task InvokeAsync(HttpContext context)
    {
        using var lease = await _rateLimiter.AcquireAsync(permitCount: 1);

        if (!lease.IsAcquired)
        {
            context.Response.StatusCode = StatusCodes.Status429TooManyRequests;
            context.Response.Headers.RetryAfter = "60";
            await context.Response.WriteAsJsonAsync(new
            {
                error = "Rate limit exceeded. Please try again later.",
                retryAfter = 60
            });
            return;
        }

        await _next(context);
    }
}
```

---

## Step 1228: Image Generation and Vision

### Image Generation with DALL-E

```csharp
using OpenAI.Images;

public class ImageGenerationService
{
    private readonly ImageClient _imageClient;

    public ImageGenerationService(OpenAIClient client)
    {
        _imageClient = client.GetImageClient("dall-e-3");
    }

    public async Task<string> GenerateImageAsync(
        string prompt,
        ImageSize size = default,
        ImageQuality quality = default)
    {
        var options = new ImageGenerationOptions
        {
            Quality = quality == default ? GeneratedImageQuality.Standard : quality,
            Size = size == default ? GeneratedImageSize.W1024xH1024 : size,
            Style = GeneratedImageStyle.Natural,
            ResponseFormat = GeneratedImageFormat.Uri,
        };

        var response = await _imageClient.GenerateImageAsync(prompt, options);
        return response.Value.ImageUri.AbsoluteUri;
    }

    public async Task<byte[]> GenerateImageBytesAsync(string prompt)
    {
        var options = new ImageGenerationOptions
        {
            Quality = GeneratedImageQuality.Standard,
            Size = GeneratedImageSize.W1024xH1024,
            ResponseFormat = GeneratedImageFormat.Bytes,
        };

        var response = await _imageClient.GenerateImageAsync(prompt, options);
        return response.Value.ImageBytes.ToArray();
    }
}
```

### Vision - Analyze Images

```csharp
public class VisionService
{
    private readonly ChatClient _chatClient;

    public VisionService(OpenAIClient client)
    {
        _chatClient = client.GetChatClient("gpt-4o");
    }

    public async Task<string> AnalyzeImageAsync(
        string imageUrl,
        string question = "What's in this image?")
    {
        var messages = new List<ChatMessage>
        {
            new UserChatMessage(
                ChatMessageContentPart.CreateTextPart(question),
                ChatMessageContentPart.CreateImagePart(new Uri(imageUrl)))
        };

        var response = await _chatClient.CompleteChatAsync(messages);
        return response.Value.Content[0].Text;
    }

    public async Task<string> AnalyzeImageFromBytesAsync(
        byte[] imageBytes,
        string mimeType,
        string question = "Describe this image in detail")
    {
        var messages = new List<ChatMessage>
        {
            new UserChatMessage(
                ChatMessageContentPart.CreateTextPart(question),
                ChatMessageContentPart.CreateImagePart(
                    BinaryData.FromBytes(imageBytes),
                    mimeType))
        };

        var response = await _chatClient.CompleteChatAsync(messages);
        return response.Value.Content[0].Text;
    }

    public async Task<ProductInfo?> ExtractProductInfoAsync(byte[] productImageBytes)
    {
        var messages = new List<ChatMessage>
        {
            new SystemChatMessage("Extract product information from the image. Return JSON only."),
            new UserChatMessage(
                ChatMessageContentPart.CreateTextPart(
                    "Extract product name, price, description, and any visible specifications."),
                ChatMessageContentPart.CreateImagePart(
                    BinaryData.FromBytes(productImageBytes),
                    "image/jpeg"))
        };

        var options = new ChatCompletionOptions
        {
            ResponseFormat = ChatResponseFormat.JsonObject,
        };

        var response = await _chatClient.CompleteChatAsync(messages, options);
        var json = response.Value.Content[0].Text;

        return JsonSerializer.Deserialize<ProductInfo>(json, new JsonSerializerOptions
        {
            PropertyNameCaseInsensitive = true,
        });
    }
}

public class ProductInfo
{
    public string Name { get; set; } = string.Empty;
    public string Price { get; set; } = string.Empty;
    public string Description { get; set; } = string.Empty;
    public List<string> Specifications { get; set; } = [];
}
```

---

## Step 1229: Prompt Engineering and Templates

### PromptTemplate System

```csharp
public class PromptTemplate
{
    private readonly string _template;
    private readonly Dictionary<string, string> _defaults;

    private PromptTemplate(string template, Dictionary<string, string> defaults)
    {
        _template = template;
        _defaults = defaults;
    }

    public static Builder Create(string template) => new(template);

    public string Render(Dictionary<string, string>? variables = null)
    {
        var allVars = new Dictionary<string, string>(_defaults);
        if (variables is not null)
        {
            foreach (var (k, v) in variables)
                allVars[k] = v;
        }

        var result = _template;
        foreach (var (key, value) in allVars)
        {
            result = result.Replace($"{{{{{key}}}}}", value);
        }

        // Check for unresolved placeholders
        var unresolved = System.Text.RegularExpressions.Regex.Matches(
            result, @"\{\{(\w+)\}\}");
        if (unresolved.Count > 0)
        {
            var keys = unresolved.Cast<System.Text.RegularExpressions.Match>()
                .Select(m => m.Groups[1].Value)
                .Distinct();
            throw new InvalidOperationException(
                $"Unresolved template variables: {string.Join(", ", keys)}");
        }

        return result;
    }

    public class Builder
    {
        private readonly string _template;
        private readonly Dictionary<string, string> _defaults = [];

        public Builder(string template) => _template = template;

        public Builder WithDefault(string key, string value)
        {
            _defaults[key] = value;
            return this;
        }

        public PromptTemplate Build() => new(_template, _defaults);
    }
}

// Usage
public static class PromptLibrary
{
    public static readonly PromptTemplate CodeReview = PromptTemplate
        .Create("""
        Review the following {{language}} code for:
        1. Bugs and potential errors
        2. Performance issues
        3. Security vulnerabilities
        4. Code style and best practices

        Code:
        ```{{language}}
        {{code}}
        ```

        Provide specific, actionable feedback. Format as numbered list.
        """)
        .WithDefault("language", "C#")
        .Build();

    public static readonly PromptTemplate Summarize = PromptTemplate
        .Create("""
        Summarize the following text in {{style}} style:
        - Maximum {{max_words}} words
        - Focus on: {{focus}}
        
        Text: {{text}}
        """)
        .WithDefault("style", "concise")
        .WithDefault("max_words", "100")
        .WithDefault("focus", "key points")
        .Build();
}
```

---

## Step 1230: AI Testing Strategies

### Unit Testing AI Services

```csharp
using Microsoft.Extensions.AI;
using NSubstitute;
using Xunit;

public class AiChatServiceTests
{
    private readonly IChatClient _mockChatClient;
    private readonly AIChatService _sut;

    public AiChatServiceTests()
    {
        _mockChatClient = Substitute.For<IChatClient>();
        _sut = new AIChatService(_mockChatClient);
    }

    [Fact]
    public async Task GetResponseAsync_ReturnsMessageText()
    {
        // Arrange
        var expectedResponse = "Hello, how can I help you?";
        _mockChatClient
            .CompleteAsync(
                Arg.Any<IList<ChatMessage>>(),
                Arg.Any<ChatOptions>(),
                Arg.Any<CancellationToken>())
            .Returns(new ChatCompletion(
                new ChatMessage(ChatRole.Assistant, expectedResponse)));

        // Act
        var result = await _sut.GetResponseAsync("Hi");

        // Assert
        result.Should().Be(expectedResponse);
    }

    [Fact]
    public async Task GetResponseAsync_WhenClientThrows_PropagatesException()
    {
        // Arrange
        _mockChatClient
            .CompleteAsync(
                Arg.Any<IList<ChatMessage>>(),
                Arg.Any<ChatOptions>(),
                Arg.Any<CancellationToken>())
            .ThrowsAsync(new HttpRequestException("API unavailable"));

        // Act & Assert
        await Assert.ThrowsAsync<HttpRequestException>(
            () => _sut.GetResponseAsync("Test"));
    }
}
```

### Integration Testing with Real Models

```csharp
[Collection("OpenAI")]
public class OpenAIIntegrationTests(OpenAIFixture fixture)
{
    [Fact]
    [Trait("Category", "Integration")]
    public async Task Chat_WithSimpleQuestion_ReturnsResponse()
    {
        // Arrange
        var service = fixture.GetChatService();

        // Act
        var response = await service.GetResponseAsync(
            "What is 2 + 2? Answer with just the number.");

        // Assert
        response.Should().Contain("4");
    }

    [Fact]
    [Trait("Category", "Integration")]
    public async Task Embedding_ReturnsSameVectorForSameText()
    {
        // Arrange
        var embeddingService = fixture.GetEmbeddingService();
        const string text = "Hello, world!";

        // Act
        var embedding1 = await embeddingService.GetEmbeddingAsync(text);
        var embedding2 = await embeddingService.GetEmbeddingAsync(text);

        // Assert - embeddings should be deterministic
        embedding1.Should().BeEquivalentTo(embedding2);
    }
}

public class OpenAIFixture : IAsyncLifetime
{
    private OpenAIClient? _client;

    public async Task InitializeAsync()
    {
        var apiKey = Environment.GetEnvironmentVariable("OPENAI_API_KEY")
            ?? throw new SkipException("OpenAI API key not configured");

        _client = new OpenAIClient(apiKey);
        await Task.CompletedTask;
    }

    public AIChatService GetChatService()
    {
        var chatClient = _client!.GetChatClient("gpt-4o-mini")
            .AsIChatClient();
        return new AIChatService(chatClient);
    }

    public EmbeddingService GetEmbeddingService()
        => new EmbeddingService(_client!);

    public Task DisposeAsync() => Task.CompletedTask;
}
```

---

## Step 1231: AI Performance and Observability

### OpenTelemetry for AI

```csharp
// Program.cs
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing =>
    {
        tracing
            .AddSource("*") // Capture all sources including AI
            .AddAspNetCoreInstrumentation()
            .AddHttpClientInstrumentation()
            .AddOtlpExporter();
    })
    .WithMetrics(metrics =>
    {
        metrics
            .AddAspNetCoreInstrumentation()
            .AddMeter("Microsoft.Extensions.AI") // AI SDK metrics
            .AddOtlpExporter();
    });

// Add AI with telemetry
builder.Services.AddChatClient(b =>
{
    b.Use(new OpenAIClient(apiKey).AsChatClient("gpt-4o-mini"));
    b.UseOpenTelemetry(configure: o =>
    {
        o.EnableSensitiveData = false; // Don't log prompt content in production
    });
});
```

### Custom AI Metrics

```csharp
public class AiMetricsCollector
{
    private static readonly Meter Meter = new("MyApp.AI");

    private static readonly Counter<long> RequestCount =
        Meter.CreateCounter<long>(
            "ai.requests.total",
            description: "Total AI API requests");

    private static readonly Histogram<double> LatencyHistogram =
        Meter.CreateHistogram<double>(
            "ai.request.duration_ms",
            unit: "ms",
            description: "AI request duration");

    private static readonly Counter<long> TokensUsed =
        Meter.CreateCounter<long>(
            "ai.tokens.used",
            description: "Total tokens consumed");

    public void RecordRequest(
        string provider,
        string model,
        long inputTokens,
        long outputTokens,
        double durationMs,
        bool succeeded)
    {
        var tags = new TagList
        {
            { "provider", provider },
            { "model", model },
            { "success", succeeded.ToString() },
        };

        RequestCount.Add(1, tags);
        LatencyHistogram.Record(durationMs, tags);
        TokensUsed.Add(inputTokens + outputTokens,
            new TagList
            {
                { "provider", provider },
                { "type", "total" },
            });
    }
}
```

---

## Step 1232: Semantic Kernel Planners

### Function Calling Planner

```csharp
using Microsoft.SemanticKernel.Planning;

public class AgentWithPlanner
{
    private readonly Kernel _kernel;

    public AgentWithPlanner(Kernel kernel)
    {
        _kernel = kernel;
    }

    public async Task<string> ExecuteComplexTaskAsync(string goal)
    {
        // FunctionCallingStepwisePlanner breaks complex goals into steps
        var planner = new FunctionCallingStepwisePlanner(
            new FunctionCallingStepwisePlannerOptions
            {
                MaxTokens = 4000,
                MaxIterations = 15,
            });

        var result = await planner.ExecuteAsync(_kernel, goal);

        return result.FinalAnswer ?? "Could not complete the task";
    }
}

// Complex Plugin with Multiple Functions
public class DataAnalysisPlugin
{
    [KernelFunction("load_csv")]
    [Description("Load data from a CSV file path")]
    public async Task<string> LoadCsvAsync(
        [Description("File path to CSV")] string filePath)
    {
        var lines = await File.ReadAllLinesAsync(filePath);
        return $"Loaded {lines.Length - 1} rows with columns: {lines[0]}";
    }

    [KernelFunction("calculate_statistics")]
    [Description("Calculate min, max, mean, and standard deviation for a numeric column")]
    public string CalculateStatistics(
        [Description("Column data as comma-separated values")] string data,
        [Description("Column name")] string columnName)
    {
        var values = data.Split(',')
            .Select(v => double.Parse(v.Trim()))
            .ToArray();

        var mean = values.Average();
        var stdDev = Math.Sqrt(values.Select(v => Math.Pow(v - mean, 2)).Average());

        return $"{columnName}: Min={values.Min():F2}, Max={values.Max():F2}, " +
               $"Mean={mean:F2}, StdDev={stdDev:F2}";
    }

    [KernelFunction("generate_report")]
    [Description("Generate a markdown report from analysis results")]
    public string GenerateReport(
        [Description("Analysis results as JSON")] string analysisJson,
        [Description("Report title")] string title)
    {
        var analysis = JsonSerializer.Deserialize<Dictionary<string, object>>(analysisJson)!;

        var sb = new StringBuilder();
        sb.AppendLine($"# {title}");
        sb.AppendLine();
        sb.AppendLine($"Generated: {DateTime.UtcNow:yyyy-MM-dd HH:mm} UTC");
        sb.AppendLine();
        sb.AppendLine("## Summary");

        foreach (var (key, value) in analysis)
        {
            sb.AppendLine($"- **{key}**: {value}");
        }

        return sb.ToString();
    }
}
```

---

## Step 1233: Document AI Processing Pipeline

### Document Processing with AI

```csharp
public class DocumentProcessingPipeline
{
    private readonly ChatClient _chatClient;
    private readonly EmbeddingService _embeddingService;
    private readonly QdrantVectorStore _vectorStore;

    public DocumentProcessingPipeline(
        ChatClient chatClient,
        EmbeddingService embeddingService,
        QdrantVectorStore vectorStore)
    {
        _chatClient = chatClient;
        _embeddingService = embeddingService;
        _vectorStore = vectorStore;
    }

    public async Task<ProcessedDocument> ProcessAsync(
        string content,
        string source,
        CancellationToken ct = default)
    {
        // 1. Extract metadata
        var metadata = await ExtractMetadataAsync(content, ct);

        // 2. Classify document
        var category = await ClassifyDocumentAsync(content, ct);

        // 3. Generate summary
        var summary = await SummarizeAsync(content, ct);

        // 4. Extract key entities
        var entities = await ExtractEntitiesAsync(content, ct);

        // 5. Chunk and embed
        var chunks = SplitIntoChunks(content, 500, 50);
        foreach (var (chunk, i) in chunks.Select((c, i) => (c, i)))
        {
            await _vectorStore.UpsertAsync(
                id: $"{source}_{i}",
                content: chunk,
                source: source,
                metadata: new Dictionary<string, string>
                {
                    ["category"] = category,
                    ["chunk_index"] = i.ToString(),
                });
        }

        return new ProcessedDocument
        {
            Source = source,
            Category = category,
            Summary = summary,
            Entities = entities,
            Metadata = metadata,
            ChunkCount = chunks.Count,
        };
    }

    private async Task<string> ClassifyDocumentAsync(string content, CancellationToken ct)
    {
        var messages = new List<ChatMessage>
        {
            new SystemChatMessage(
                "Classify the document into one of: Technical, Legal, Financial, Marketing, Other. " +
                "Respond with ONLY the category name."),
            new UserChatMessage(content[..Math.Min(content.Length, 2000)]),
        };

        var response = await _chatClient.CompleteChatAsync(messages);
        return response.Value.Content[0].Text.Trim();
    }

    private async Task<string> SummarizeAsync(string content, CancellationToken ct)
    {
        var messages = new List<ChatMessage>
        {
            new SystemChatMessage("Summarize in 2-3 sentences. Focus on key points."),
            new UserChatMessage(content[..Math.Min(content.Length, 4000)]),
        };

        var response = await _chatClient.CompleteChatAsync(messages);
        return response.Value.Content[0].Text;
    }

    private async Task<List<string>> ExtractEntitiesAsync(string content, CancellationToken ct)
    {
        var messages = new List<ChatMessage>
        {
            new SystemChatMessage(
                "Extract named entities (people, organizations, locations, dates). " +
                "Return as JSON array of strings."),
            new UserChatMessage(content[..Math.Min(content.Length, 2000)]),
        };

        var options = new ChatCompletionOptions
        {
            ResponseFormat = ChatResponseFormat.JsonObject,
        };

        var response = await _chatClient.CompleteChatAsync(messages, options);
        try
        {
            using var doc = JsonDocument.Parse(response.Value.Content[0].Text);
            return doc.RootElement.EnumerateArray()
                .Select(e => e.GetString() ?? "")
                .Where(s => !string.IsNullOrEmpty(s))
                .ToList();
        }
        catch
        {
            return [];
        }
    }

    private async Task<Dictionary<string, string>> ExtractMetadataAsync(
        string content, CancellationToken ct)
    {
        // Extract date, author, title if present
        return new Dictionary<string, string>
        {
            ["word_count"] = content.Split(' ').Length.ToString(),
            ["char_count"] = content.Length.ToString(),
        };
    }

    private static List<string> SplitIntoChunks(string text, int size, int overlap)
    {
        var chunks = new List<string>();
        var words = text.Split(' ');
        var step = size - overlap;

        for (int i = 0; i < words.Length; i += step)
        {
            var chunk = string.Join(" ", words.Skip(i).Take(size));
            if (!string.IsNullOrWhiteSpace(chunk))
                chunks.Add(chunk);
        }

        return chunks;
    }
}

public class ProcessedDocument
{
    public required string Source { get; init; }
    public required string Category { get; init; }
    public required string Summary { get; init; }
    public required List<string> Entities { get; init; }
    public required Dictionary<string, string> Metadata { get; init; }
    public int ChunkCount { get; init; }
}
```

---

## Step 1234: Phi-3 and Small Language Models

### Running Local LLMs with .NET

```csharp
// Install: dotnet add package Microsoft.ML.OnnxRuntimeGenAI
// Download Phi-3 ONNX model from HuggingFace

using Microsoft.ML.OnnxRuntimeGenAI;

public class Phi3LocalModel : IDisposable
{
    private readonly Model _model;
    private readonly Tokenizer _tokenizer;

    public Phi3LocalModel(string modelPath)
    {
        _model = new Model(modelPath);
        _tokenizer = new Tokenizer(_model);
    }

    public async IAsyncEnumerable<string> GenerateAsync(
        string prompt,
        int maxLength = 200,
        float temperature = 0.7f,
        [EnumeratorCancellation] CancellationToken ct = default)
    {
        var inputs = _tokenizer.Encode($"<|user|>\n{prompt}<|end|>\n<|assistant|>\n");

        var params_ = new GeneratorParams(_model);
        params_.SetSearchOption("max_length", maxLength);
        params_.SetSearchOption("temperature", temperature);

        using var generator = new Generator(_model, params_);
        generator.AppendTokenSequences(inputs);

        while (!generator.IsDone())
        {
            ct.ThrowIfCancellationRequested();
            generator.GenerateNextToken();

            var outputTokens = generator.GetSequence(0);
            var newToken = outputTokens[^1];
            var decoded = _tokenizer.Decode([newToken]);

            if (!string.IsNullOrEmpty(decoded))
                yield return decoded;

            await Task.Yield();
        }
    }

    public void Dispose()
    {
        _tokenizer.Dispose();
        _model.Dispose();
    }
}
```

---

## Step 1235: Complete AI Application - Code Review Bot

```csharp
// Complete example: AI-powered code review API

// Models
public record CodeReviewRequest(
    string Code,
    string Language = "csharp",
    string[]? FocusAreas = null);

public record CodeReviewResponse(
    string Summary,
    IReadOnlyList<CodeIssue> Issues,
    IReadOnlyList<string> Improvements,
    string OverallScore);

public record CodeIssue(
    string Type,       // Bug, Performance, Security, Style
    string Severity,   // Critical, High, Medium, Low
    int? LineNumber,
    string Description,
    string? Suggestion);

// Service
public class CodeReviewService
{
    private readonly ChatClient _chatClient;

    public CodeReviewService(ChatClient chatClient)
    {
        _chatClient = chatClient;
    }

    public async Task<CodeReviewResponse> ReviewAsync(
        CodeReviewRequest request,
        CancellationToken ct = default)
    {
        var focusAreas = request.FocusAreas?.Length > 0
            ? string.Join(", ", request.FocusAreas)
            : "bugs, performance, security, code style";

        var systemPrompt = $"""
            You are an expert {request.Language} code reviewer.
            Review code for: {focusAreas}.
            Be specific, actionable, and provide line numbers when possible.
            Return a JSON object with this structure:
            {{
                "summary": "overall assessment",
                "issues": [
                    {{
                        "type": "Bug|Performance|Security|Style",
                        "severity": "Critical|High|Medium|Low",
                        "lineNumber": null or number,
                        "description": "what's wrong",
                        "suggestion": "how to fix it"
                    }}
                ],
                "improvements": ["general improvement suggestion 1", ...],
                "overallScore": "A|B|C|D|F"
            }}
            """;

        var messages = new List<ChatMessage>
        {
            new SystemChatMessage(systemPrompt),
            new UserChatMessage($"Review this {request.Language} code:\n\n```{request.Language}\n{request.Code}\n```"),
        };

        var options = new ChatCompletionOptions
        {
            ResponseFormat = ChatResponseFormat.JsonObject,
            Temperature = 0.3f,
        };

        var response = await _chatClient.CompleteChatAsync(messages, options);
        var json = response.Value.Content[0].Text;

        return ParseReviewResponse(json);
    }

    private static CodeReviewResponse ParseReviewResponse(string json)
    {
        using var doc = JsonDocument.Parse(json);
        var root = doc.RootElement;

        var issues = root.GetProperty("issues").EnumerateArray()
            .Select(issue => new CodeIssue(
                Type: issue.GetProperty("type").GetString() ?? "",
                Severity: issue.GetProperty("severity").GetString() ?? "",
                LineNumber: issue.TryGetProperty("lineNumber", out var ln)
                    ? ln.ValueKind == JsonValueKind.Number ? ln.GetInt32() : null
                    : null,
                Description: issue.GetProperty("description").GetString() ?? "",
                Suggestion: issue.TryGetProperty("suggestion", out var s)
                    ? s.GetString() : null))
            .ToList();

        var improvements = root.GetProperty("improvements").EnumerateArray()
            .Select(i => i.GetString() ?? "")
            .Where(s => !string.IsNullOrEmpty(s))
            .ToList();

        return new CodeReviewResponse(
            Summary: root.GetProperty("summary").GetString() ?? "",
            Issues: issues,
            Improvements: improvements,
            OverallScore: root.GetProperty("overallScore").GetString() ?? "C");
    }
}

// API Endpoint
app.MapPost("/api/code-review", async (
    CodeReviewRequest request,
    CodeReviewService reviewService,
    CancellationToken ct) =>
{
    if (string.IsNullOrWhiteSpace(request.Code))
        return Results.BadRequest("Code cannot be empty");

    if (request.Code.Length > 50_000)
        return Results.BadRequest("Code too large (max 50,000 characters)");

    var review = await reviewService.ReviewAsync(request, ct);
    return Results.Ok(review);
})
.WithName("ReviewCode")
.Produces<CodeReviewResponse>()
.ProducesProblem(StatusCodes.Status400BadRequest);
```

---

## Step 1236: appsettings.json Configuration

```json
{
  "OpenAI": {
    "ApiKey": "${OPENAI_API_KEY}",
    "ChatModel": "gpt-4o-mini",
    "EmbeddingModel": "text-embedding-3-small"
  },
  "AzureOpenAI": {
    "Endpoint": "${AZURE_OPENAI_ENDPOINT}",
    "ApiKey": "${AZURE_OPENAI_API_KEY}",
    "ChatDeployment": "gpt-4o",
    "EmbeddingDeployment": "text-embedding-3-small"
  },
  "Ollama": {
    "BaseUrl": "http://localhost:11434",
    "ChatModel": "llama3.2",
    "EmbeddingModel": "nomic-embed-text"
  },
  "Qdrant": {
    "Host": "localhost",
    "Port": "6334"
  },
  "AI": {
    "MaxContextLength": 8000,
    "DefaultTemperature": 0.7,
    "CacheDurationHours": 24,
    "RateLimitPerMinute": 100
  }
}
```

---

## Step 1237: Complete Project Structure

```
MyAIApp/
├── src/
│   ├── MyAIApp.API/
│   │   ├── Program.cs
│   │   ├── Endpoints/
│   │   │   ├── ChatEndpoints.cs
│   │   │   ├── CodeReviewEndpoints.cs
│   │   │   └── DocumentEndpoints.cs
│   │   ├── Middleware/
│   │   │   ├── AiRateLimitingMiddleware.cs
│   │   │   └── ContentSafetyMiddleware.cs
│   │   └── appsettings.json
│   │
│   ├── MyAIApp.Core/
│   │   ├── AI/
│   │   │   ├── IChatService.cs
│   │   │   ├── IEmbeddingService.cs
│   │   │   └── IVectorStore.cs
│   │   └── Documents/
│   │       ├── ProcessedDocument.cs
│   │       └── IDocumentProcessor.cs
│   │
│   ├── MyAIApp.Infrastructure/
│   │   ├── AI/
│   │   │   ├── OpenAIChatService.cs
│   │   │   ├── AzureOpenAIService.cs
│   │   │   └── OllamaService.cs
│   │   ├── VectorStores/
│   │   │   ├── QdrantVectorStore.cs
│   │   │   └── InMemoryVectorStore.cs
│   │   └── Caching/
│   │       └── CachedChatClient.cs
│   │
│   └── MyAIApp.Application/
│       ├── Services/
│       │   ├── RagChatbot.cs
│       │   ├── CodeReviewService.cs
│       │   └── DocumentProcessingPipeline.cs
│       └── Pipelines/
│           └── AiBatchProcessor.cs
│
├── tests/
│   ├── MyAIApp.UnitTests/
│   │   └── Services/
│   │       └── CodeReviewServiceTests.cs
│   └── MyAIApp.IntegrationTests/
│       └── OpenAIIntegrationTests.cs
│
└── docker-compose.yml
```

### docker-compose.yml

```yaml
version: '3.8'

services:
  api:
    build: .
    ports:
      - "8080:8080"
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - Qdrant__Host=qdrant
    depends_on:
      - qdrant
      - ollama

  qdrant:
    image: qdrant/qdrant:latest
    ports:
      - "6333:6333"
      - "6334:6334"
    volumes:
      - qdrant_storage:/qdrant/storage

  ollama:
    image: ollama/ollama:latest
    ports:
      - "11434:11434"
    volumes:
      - ollama_data:/root/.ollama
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]  # Optional: GPU support

volumes:
  qdrant_storage:
  ollama_data:
```

---

## Step 1238: AI Cost Optimization

### Token Usage Tracking

```csharp
public class TokenUsageTracker
{
    private readonly IDistributedCache _cache;
    private readonly ILogger<TokenUsageTracker> _logger;

    public TokenUsageTracker(IDistributedCache cache, ILogger<TokenUsageTracker> logger)
    {
        _cache = cache;
        _logger = logger;
    }

    public async Task<DailyUsage> GetUsageAsync(string userId, DateOnly date)
    {
        var key = $"usage:{userId}:{date:yyyyMMdd}";
        var json = await _cache.GetStringAsync(key);

        return json is null
            ? new DailyUsage { UserId = userId, Date = date }
            : JsonSerializer.Deserialize<DailyUsage>(json)!;
    }

    public async Task RecordUsageAsync(
        string userId,
        int inputTokens,
        int outputTokens,
        string model)
    {
        var today = DateOnly.FromDateTime(DateTime.UtcNow);
        var key = $"usage:{userId}:{today:yyyyMMdd}";

        var usage = await GetUsageAsync(userId, today);
        usage = usage with
        {
            InputTokens = usage.InputTokens + inputTokens,
            OutputTokens = usage.OutputTokens + outputTokens,
            TotalCost = usage.TotalCost + CalculateCost(inputTokens, outputTokens, model),
            RequestCount = usage.RequestCount + 1,
        };

        await _cache.SetStringAsync(
            key,
            JsonSerializer.Serialize(usage),
            new DistributedCacheEntryOptions
            {
                AbsoluteExpiration = DateTimeOffset.UtcNow.AddDays(30),
            });

        _logger.LogInformation(
            "User {UserId} used {Tokens} tokens on {Model}",
            userId, inputTokens + outputTokens, model);
    }

    private static decimal CalculateCost(int input, int output, string model) =>
        model switch
        {
            "gpt-4o" => (input / 1_000_000m * 5.0m) + (output / 1_000_000m * 15.0m),
            "gpt-4o-mini" => (input / 1_000_000m * 0.15m) + (output / 1_000_000m * 0.60m),
            "gpt-3.5-turbo" => (input / 1_000_000m * 0.50m) + (output / 1_000_000m * 1.50m),
            _ => 0m,
        };
}

public record DailyUsage
{
    public required string UserId { get; init; }
    public required DateOnly Date { get; init; }
    public int InputTokens { get; init; }
    public int OutputTokens { get; init; }
    public decimal TotalCost { get; init; }
    public int RequestCount { get; init; }
}
```

---

## Step 1239: Guardrails and Safety

### Input/Output Validation Pipeline

```csharp
public class SafeAIChatPipeline
{
    private readonly IChatClient _chatClient;
    private readonly ContentSafetyService _contentSafety;
    private readonly IReadOnlyList<string> _blockedTopics;

    public SafeAIChatPipeline(
        IChatClient chatClient,
        ContentSafetyService contentSafety,
        IConfiguration config)
    {
        _chatClient = chatClient;
        _contentSafety = contentSafety;
        _blockedTopics = config.GetSection("AI:BlockedTopics")
            .Get<string[]>() ?? [];
    }

    public async Task<SafeChatResult> ProcessAsync(
        string userInput,
        CancellationToken ct = default)
    {
        // 1. Input validation
        if (string.IsNullOrWhiteSpace(userInput))
            return SafeChatResult.Error("Empty input");

        if (userInput.Length > 5000)
            return SafeChatResult.Error("Input too long");

        // 2. Content safety check
        var safety = await _contentSafety.AnalyzeTextAsync(userInput);
        if (!safety.IsSafe)
            return SafeChatResult.Blocked("Content violates safety policy");

        // 3. Topic blocking
        var lowerInput = userInput.ToLowerInvariant();
        var blockedTopic = _blockedTopics
            .FirstOrDefault(t => lowerInput.Contains(t));
        if (blockedTopic is not null)
            return SafeChatResult.Blocked($"Topic not allowed: {blockedTopic}");

        // 4. Get AI response
        var response = await _chatClient.CompleteAsync(userInput, cancellationToken: ct);
        var outputText = response.Message.Text ?? string.Empty;

        // 5. Output safety check
        var outputSafety = await _contentSafety.AnalyzeTextAsync(outputText);
        if (!outputSafety.IsSafe)
        {
            // Log for review but don't expose to user
            return SafeChatResult.Error("Response could not be generated. Please rephrase your question.");
        }

        return SafeChatResult.Success(outputText);
    }
}

public record SafeChatResult
{
    public bool IsSuccess { get; init; }
    public string? Content { get; init; }
    public string? ErrorMessage { get; init; }
    public bool IsBlocked { get; init; }

    public static SafeChatResult Success(string content) =>
        new() { IsSuccess = true, Content = content };

    public static SafeChatResult Error(string message) =>
        new() { IsSuccess = false, ErrorMessage = message };

    public static SafeChatResult Blocked(string reason) =>
        new() { IsSuccess = false, IsBlocked = true, ErrorMessage = reason };
}
```

---

## Step 1240: Summary - AI/ML ใน .NET

### ภาพรวมที่เรียนในบทนี้

```
AI/ML Integration (.NET 8/9/10)
├── ML.NET
│   ├── Binary/Multi-class Classification
│   ├── Regression
│   └── AutoML
├── ONNX Runtime
│   ├── Cross-platform inference
│   ├── Image Classification
│   └── BERT NLP
├── Microsoft.Extensions.AI
│   ├── IChatClient (unified interface)
│   └── Provider abstraction
├── OpenAI SDK
│   ├── Chat Completions
│   ├── Streaming
│   ├── Function Calling
│   └── Embeddings
├── Semantic Kernel
│   ├── Plugins (Native + Semantic)
│   ├── Memory (Vector search)
│   └── Planners
├── RAG Pattern
│   ├── Document processing
│   ├── Vector stores (Qdrant)
│   └── Retrieval + Generation
├── Local LLMs (Ollama)
│   ├── Phi-3, Llama 3.2
│   └── Privacy-first AI
├── Safety & Guardrails
│   ├── Content safety
│   ├── Rate limiting
│   └── Input/output validation
└── Observability
    ├── OpenTelemetry
    ├── Token tracking
    └── Cost optimization
```

### Quick Decision Guide

| Need | Use |
|------|-----|
| Train custom ML model | ML.NET |
| Use pre-trained Python model | ONNX Runtime |
| Chat with GPT-4o | OpenAI SDK + IChatClient |
| Orchestrate LLM workflows | Semantic Kernel |
| Semantic document search | RAG + Qdrant |
| Privacy-first AI (on-prem) | Ollama + local LLM |
| Azure-hosted AI | Azure OpenAI + Managed Identity |
| Structured AI output | JSON Schema mode |
| Image analysis | GPT-4o Vision |
| Cost control | Caching + Token tracking |

ในบทถัดไปจะเรียนเรื่อง **Cloud-Native .NET (Steps 1251-1290)** ซึ่งครอบคลุม Azure, AWS, GCP, Kubernetes, Service Mesh และ Infrastructure as Code!
