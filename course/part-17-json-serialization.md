# Part 17: JSON Serialization (Steps 511-540)

## เป้าหมายการเรียนรู้
- System.Text.Json (built-in, .NET 5+) 
- Newtonsoft.Json (Json.NET) — industry standard
- Serialization options และ customization
- Custom converters
- JSON Schema validation
- JSON Patch และ Merge Patch

---

## Step 511: System.Text.Json Basics

```csharp
using System.Text.Json;
using System.Text.Json.Serialization;

// Basic serialization
record Person(string Name, int Age, string Email);

var person = new Person("Alice", 30, "alice@example.com");

// Serialize to JSON string
string json = JsonSerializer.Serialize(person);
Console.WriteLine(json); // {"name":"Alice","age":30,"email":"alice@example.com"}

// With pretty printing
var options = new JsonSerializerOptions { WriteIndented = true };
string prettyJson = JsonSerializer.Serialize(person, options);
Console.WriteLine(prettyJson);
// {
//   "name": "Alice",
//   "age": 30,
//   "email": "alice@example.com"
// }

// Deserialize from JSON
string input = """{"Name":"Bob","Age":25,"Email":"bob@example.com"}""";
Person? bob = JsonSerializer.Deserialize<Person>(input);
Console.WriteLine($"{bob?.Name}, {bob?.Age}"); // Bob, 25

// Common options
var commonOptions = new JsonSerializerOptions
{
    WriteIndented = true,
    PropertyNameCaseInsensitive = true,          // case-insensitive property names
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase, // PascalCase → camelCase
    DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull, // skip nulls
    AllowTrailingCommas = true,                  // forgive trailing commas
    ReadCommentHandling = JsonCommentHandling.Skip, // allow //comments
    NumberHandling = JsonNumberHandling.AllowReadingFromString, // "42" → 42
};

// Serialize complex types
class Order
{
    public int Id { get; set; }
    public string Customer { get; set; } = "";
    public List<OrderItem> Items { get; set; } = new();
    public DateTime CreatedAt { get; set; }
    public decimal Total => Items.Sum(i => i.Price * i.Quantity);
    
    [JsonIgnore] // Don't serialize this property
    public string InternalCode { get; set; } = "";
    
    [JsonPropertyName("order_date")] // Custom JSON property name
    public string DateFormatted => CreatedAt.ToString("yyyy-MM-dd");
}

class OrderItem
{
    public string Name { get; set; } = "";
    public int Quantity { get; set; }
    public decimal Price { get; set; }
}

var order = new Order
{
    Id = 1,
    Customer = "Alice",
    CreatedAt = DateTime.Now,
    Items = new()
    {
        new() { Name = "Laptop", Quantity = 1, Price = 1299m },
        new() { Name = "Mouse", Quantity = 2, Price = 29.99m }
    },
    InternalCode = "SECRET" // will be ignored
};

string orderJson = JsonSerializer.Serialize(order, new JsonSerializerOptions { WriteIndented = true });
Console.WriteLine(orderJson);
```

---

## Step 512: JsonSerializer Options and Attributes

```csharp
using System.Text.Json;
using System.Text.Json.Serialization;

// JsonSerializerContext for source generation (AOT-friendly, faster)
[JsonSerializable(typeof(Product))]
[JsonSerializable(typeof(List<Product>))]
[JsonSourceGenerationOptions(
    WriteIndented = true,
    PropertyNamingPolicy = JsonKnownNamingPolicy.CamelCase,
    DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull
)]
partial class AppJsonContext : JsonSerializerContext { }

// Attributes
class Product
{
    [JsonPropertyName("product_id")]     // custom name in JSON
    public int Id { get; set; }
    
    public string Name { get; set; } = "";
    
    [JsonPropertyOrder(1)]              // ordering in output
    public decimal Price { get; set; }
    
    [JsonIgnore]                         // exclude from JSON
    public string InternalSku { get; set; } = "";
    
    [JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingNull)]
    public string? Description { get; set; }
    
    [JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingDefault)]
    public int Stock { get; set; }
    
    [JsonExtensionData]                  // capture unknown properties
    public Dictionary<string, JsonElement>? Extra { get; set; }
}

// Read-only and required
class Config
{
    [JsonRequired]
    public string ConnectionString { get; set; } = "";
    
    public int Timeout { get; set; } = 30;
    
    [JsonInclude]  // include non-public (needs to be visible to serializer)
    public string? SecretValue { get; private set; }
}

// Enums
class StatusObject
{
    [JsonConverter(typeof(JsonStringEnumConverter))]
    public Status Status { get; set; }
}

enum Status { Active, Inactive, Suspended }

// DateTimeOffset with custom format
class EventLog
{
    public string Message { get; set; } = "";
    
    [JsonConverter(typeof(DateTimeConverter))]
    public DateTime Timestamp { get; set; }
}

class DateTimeConverter : System.Text.Json.Serialization.JsonConverter<DateTime>
{
    private const string Format = "yyyy-MM-dd HH:mm:ss";
    
    public override DateTime Read(ref Utf8JsonReader reader, Type typeToConvert, JsonSerializerOptions options)
    {
        string? value = reader.GetString();
        return DateTime.ParseExact(value!, Format, null);
    }
    
    public override void Write(Utf8JsonWriter writer, DateTime value, JsonSerializerOptions options)
    {
        writer.WriteStringValue(value.ToString(Format));
    }
}

// Using source generation (fast, AOT-compatible)
var product = new Product { Id = 1, Name = "Widget", Price = 9.99m };
string json = JsonSerializer.Serialize(product, AppJsonContext.Default.Product);
Product? loaded = JsonSerializer.Deserialize(json, AppJsonContext.Default.Product);
```

---

## Step 513: JsonNode and JSON DOM

```csharp
using System.Text.Json.Nodes;

// JsonNode — dynamic JSON manipulation
string jsonStr = """
    {
        "name": "Alice",
        "age": 30,
        "address": {
            "city": "Bangkok",
            "country": "Thailand"
        },
        "scores": [95, 87, 92]
    }
    """;

// Parse JSON
JsonNode? root = JsonNode.Parse(jsonStr);

// Access properties
string name = root!["name"]!.GetValue<string>();
int age = root["age"]!.GetValue<int>();
string city = root["address"]!["city"]!.GetValue<string>();

Console.WriteLine($"{name}, {age}, {city}");

// Array access
JsonArray scores = root["scores"]!.AsArray();
foreach (JsonNode? score in scores)
    Console.Write($"{score?.GetValue<int>()} ");
Console.WriteLine();

// Modify
root["age"] = 31;
root["email"] = "alice@example.com";
root["address"]!["zip"] = "10110";

// Add to array
scores.Add(98);

// Create from scratch
JsonObject newObj = new JsonObject
{
    ["name"] = "Bob",
    ["age"] = 25,
    ["active"] = true,
    ["tags"] = new JsonArray("c#", "dotnet", "developer"),
    ["address"] = new JsonObject
    {
        ["city"] = "Bangkok",
        ["country"] = "Thailand"
    }
};

Console.WriteLine(newObj.ToJsonString(new JsonSerializerOptions { WriteIndented = true }));

// JsonElement (read-only, from JsonSerializer)
using JsonDocument doc = JsonDocument.Parse(jsonStr);
JsonElement element = doc.RootElement;

// Enumerate object properties
foreach (JsonProperty prop in element.EnumerateObject())
    Console.WriteLine($"{prop.Name}: {prop.Value.ValueKind}");

// Safe access
if (element.TryGetProperty("name", out JsonElement nameEl))
    Console.WriteLine($"Name: {nameEl.GetString()}");

// Array enumeration  
if (element.TryGetProperty("scores", out JsonElement scoresEl))
    foreach (JsonElement score in scoresEl.EnumerateArray())
        Console.Write($"{score.GetInt32()} ");

// Convert to object
using JsonDocument doc2 = JsonDocument.Parse(jsonStr);
JsonElement root2 = doc2.RootElement;
string json2 = root2.GetRawText();
```

---

## Step 514: Custom JSON Converters

```csharp
using System.Text.Json;
using System.Text.Json.Serialization;

// Custom converter for complex types
class ColorConverter : JsonConverter<Color>
{
    // Serialize Color as hex string: "#FF5733"
    public override Color Read(ref Utf8JsonReader reader, Type typeToConvert, JsonSerializerOptions options)
    {
        string? hex = reader.GetString();
        if (hex == null) return new Color(0, 0, 0);
        
        hex = hex.TrimStart('#');
        if (hex.Length != 6) throw new JsonException($"Invalid color: #{hex}");
        
        return new Color(
            Convert.ToByte(hex[0..2], 16),
            Convert.ToByte(hex[2..4], 16),
            Convert.ToByte(hex[4..6], 16)
        );
    }
    
    public override void Write(Utf8JsonWriter writer, Color value, JsonSerializerOptions options)
    {
        writer.WriteStringValue($"#{value.R:X2}{value.G:X2}{value.B:X2}");
    }
}

record Color(byte R, byte G, byte B);

// Polymorphic converter
[JsonDerivedType(typeof(Dog), "dog")]
[JsonDerivedType(typeof(Cat), "cat")]
[JsonPolymorphic(TypeDiscriminatorPropertyName = "type")]
abstract class Animal
{
    public string Name { get; set; } = "";
}

class Dog : Animal { public string Breed { get; set; } = ""; }
class Cat : Animal { public bool IsIndoor { get; set; } }

// Union type converter
class StringOrNumber
{
    public object Value { get; set; }
    public StringOrNumber(object value) => Value = value;
    
    public override string ToString() => Value.ToString() ?? "";
}

class StringOrNumberConverter : JsonConverter<StringOrNumber>
{
    public override StringOrNumber Read(ref Utf8JsonReader reader, Type typeToConvert, JsonSerializerOptions options)
    {
        return reader.TokenType switch
        {
            JsonTokenType.String => new StringOrNumber(reader.GetString()!),
            JsonTokenType.Number => reader.TryGetInt64(out long i) 
                ? new StringOrNumber(i) 
                : new StringOrNumber(reader.GetDouble()),
            _ => throw new JsonException($"Unexpected token: {reader.TokenType}")
        };
    }
    
    public override void Write(Utf8JsonWriter writer, StringOrNumber value, JsonSerializerOptions options)
    {
        switch (value.Value)
        {
            case string s: writer.WriteStringValue(s); break;
            case long l:   writer.WriteNumberValue(l); break;
            case double d: writer.WriteNumberValue(d); break;
            default: writer.WriteStringValue(value.ToString()); break;
        }
    }
}

// IPAddress converter
class IPAddressConverter : JsonConverter<System.Net.IPAddress>
{
    public override System.Net.IPAddress Read(ref Utf8JsonReader reader, Type type, JsonSerializerOptions opts)
        => System.Net.IPAddress.Parse(reader.GetString()!);
    
    public override void Write(Utf8JsonWriter writer, System.Net.IPAddress value, JsonSerializerOptions opts)
        => writer.WriteStringValue(value.ToString());
}

// Usage
var options = new JsonSerializerOptions();
options.Converters.Add(new JsonStringEnumConverter());
options.Converters.Add(new ColorConverter());
options.Converters.Add(new IPAddressConverter());

// Polymorphism
Animal[] animals = { new Dog { Name = "Rex", Breed = "Lab" }, new Cat { Name = "Whiskers", IsIndoor = true } };
string animalsJson = JsonSerializer.Serialize(animals, options);
Console.WriteLine(animalsJson);
// [{"type":"dog","Name":"Rex","Breed":"Lab"},{"type":"cat","Name":"Whiskers","IsIndoor":true}]

Animal[]? loaded = JsonSerializer.Deserialize<Animal[]>(animalsJson, options);
```

---

## Step 515: Newtonsoft.Json (Json.NET)

```csharp
// Newtonsoft.Json — more feature-rich, industry standard
// Install: dotnet add package Newtonsoft.Json

using Newtonsoft.Json;
using Newtonsoft.Json.Linq;
using Newtonsoft.Json.Converters;
using Newtonsoft.Json.Serialization;

// Basic serialize/deserialize
var person = new { Name = "Alice", Age = 30, Email = "alice@example.com" };
string json = JsonConvert.SerializeObject(person, Formatting.Indented);
Console.WriteLine(json);

// Deserialize
string input = """{"Name":"Bob","Age":25}""";
dynamic obj = JsonConvert.DeserializeObject(input)!;
Console.WriteLine($"{obj.Name}: {obj.Age}");

// Type-safe deserialize
var alice = JsonConvert.DeserializeObject<Person>(input);

// Settings
var settings = new JsonSerializerSettings
{
    Formatting = Formatting.Indented,
    ContractResolver = new CamelCasePropertyNamesContractResolver(),
    NullValueHandling = NullValueHandling.Ignore,
    DefaultValueHandling = DefaultValueHandling.Ignore,
    DateFormatHandling = DateFormatHandling.IsoDateFormat,
    ReferenceLoopHandling = ReferenceLoopHandling.Ignore, // handle circular refs
    TypeNameHandling = TypeNameHandling.Auto,             // preserve type info
    Converters = new List<JsonConverter>
    {
        new StringEnumConverter(),
        new IsoDateTimeConverter { DateTimeFormat = "yyyy-MM-dd" }
    }
};

// JObject — dynamic JSON
JObject jObj = JObject.Parse(json);

// Access properties
string name = (string)jObj["Name"]!;
int age = (int)jObj["Age"]!;

// Modify
jObj["Name"] = "Updated Name";
jObj.Add("NewProp", JToken.FromObject(42));
jObj.Remove("Email");

// JArray
JArray jArr = JArray.Parse("[1, 2, 3, 4, 5]");
jArr.Add(6);
foreach (var item in jArr)
    Console.Write($"{(int)item} ");

// LINQ to JSON
JObject data = JObject.Parse("""
    {
        "products": [
            {"name": "Laptop", "price": 1299, "category": "Electronics"},
            {"name": "Desk", "price": 399, "category": "Furniture"},
            {"name": "Phone", "price": 799, "category": "Electronics"}
        ]
    }
    """);

var electronics = data["products"]!
    .Where(p => (string)p["category"]! == "Electronics")
    .OrderBy(p => (decimal)p["price"]!)
    .Select(p => $"{p["name"]}: ${p["price"]}")
    .ToList();

foreach (var e in electronics) Console.WriteLine(e);

// Custom JsonConverter (Newtonsoft)
class MoneyConverter : JsonConverter<decimal>
{
    public override decimal ReadJson(JsonReader reader, Type objectType, decimal existingValue, 
        bool hasExistingValue, JsonSerializer serializer)
    {
        // Read "1,234.56" format
        string? value = reader.Value?.ToString();
        return value != null ? decimal.Parse(value.Replace(",", ""), System.Globalization.CultureInfo.InvariantCulture) : 0;
    }
    
    public override void WriteJson(JsonWriter writer, decimal value, JsonSerializer serializer)
    {
        writer.WriteValue(value.ToString("N2"));
    }
}

record Person(string Name, int Age, string? Email = null);
```

---

## Step 516: JSON with APIs and HTTP

```csharp
using System.Net.Http;
using System.Net.Http.Json;
using System.Text.Json;

// HttpClient with JSON
static async Task HttpJsonDemo()
{
    using var http = new HttpClient
    {
        BaseAddress = new Uri("https://jsonplaceholder.typicode.com/")
    };
    
    // GET with JSON deserialization
    var posts = await http.GetFromJsonAsync<List<Post>>("/posts");
    Console.WriteLine($"Got {posts?.Count} posts");
    
    // POST with JSON body
    var newPost = new Post { Title = "New Post", Body = "Content here", UserId = 1 };
    var response = await http.PostAsJsonAsync("/posts", newPost);
    var created = await response.Content.ReadFromJsonAsync<Post>();
    Console.WriteLine($"Created post ID: {created?.Id}");
    
    // PUT
    var update = new Post { Id = 1, Title = "Updated", Body = "New content", UserId = 1 };
    await http.PutAsJsonAsync("/posts/1", update);
    
    // With custom options
    var options = new JsonSerializerOptions
    {
        PropertyNameCaseInsensitive = true,
        WriteIndented = false
    };
    var post = await http.GetFromJsonAsync<Post>("/posts/1", options);
}

record Post
{
    public int Id { get; init; }
    public int UserId { get; init; }
    public string Title { get; init; } = "";
    public string Body { get; init; } = "";
}

// JSON over WebSocket (simplified example)
static async Task JsonWebSocketDemo()
{
    using var ws = new System.Net.WebSockets.ClientWebSocket();
    
    // Create message
    var message = new { type = "subscribe", channel = "prices", symbol = "BTC" };
    string jsonMsg = JsonSerializer.Serialize(message);
    byte[] msgBytes = Encoding.UTF8.GetBytes(jsonMsg);
    
    // Send
    // await ws.SendAsync(msgBytes, WebSocketMessageType.Text, true, CancellationToken.None);
    
    // Receive and parse
    var buffer = new byte[4096];
    // var result = await ws.ReceiveAsync(buffer, CancellationToken.None);
    // var received = JsonSerializer.Deserialize<ServerMessage>(buffer[..result.Count]);
}

// Strongly-typed API client
class JsonPlaceholderClient
{
    private readonly HttpClient _http;
    private static readonly JsonSerializerOptions _options = new()
    {
        PropertyNameCaseInsensitive = true,
        WriteIndented = false
    };
    
    public JsonPlaceholderClient(HttpClient http)
    {
        _http = http;
        _http.BaseAddress = new Uri("https://jsonplaceholder.typicode.com/");
    }
    
    public Task<List<Post>?> GetPostsAsync(int? userId = null, CancellationToken ct = default)
    {
        string url = userId.HasValue ? $"/posts?userId={userId}" : "/posts";
        return _http.GetFromJsonAsync<List<Post>>(url, _options, ct);
    }
    
    public async Task<Post?> GetPostAsync(int id, CancellationToken ct = default)
    {
        try { return await _http.GetFromJsonAsync<Post>($"/posts/{id}", _options, ct); }
        catch (HttpRequestException ex) when (ex.StatusCode == System.Net.HttpStatusCode.NotFound)
        { return null; }
    }
    
    public async Task<Post> CreatePostAsync(Post post, CancellationToken ct = default)
    {
        var response = await _http.PostAsJsonAsync("/posts", post, _options, ct);
        response.EnsureSuccessStatusCode();
        return (await response.Content.ReadFromJsonAsync<Post>(_options, ct))!;
    }
}
```

---

## Step 517: โปรแกรมตัวอย่าง — Configuration System

```csharp
using System;
using System.IO;
using System.Text.Json;
using System.Text.Json.Nodes;
using System.Collections.Generic;
using System.Threading.Tasks;

// ============================================================
// JSON Configuration System
// ============================================================

class AppConfig
{
    public DatabaseConfig Database { get; set; } = new();
    public ServerConfig Server { get; set; } = new();
    public LoggingConfig Logging { get; set; } = new();
    public Dictionary<string, string> Features { get; set; } = new();
}

class DatabaseConfig
{
    public string Host { get; set; } = "localhost";
    public int Port { get; set; } = 5432;
    public string Database { get; set; } = "myapp";
    public string Username { get; set; } = "admin";
    
    [System.Text.Json.Serialization.JsonIgnore]
    public string? Password { get; set; } // Not serialized
    
    public int MaxConnections { get; set; } = 20;
    public int CommandTimeout { get; set; } = 30;
    
    public string ConnectionString => $"Host={Host};Port={Port};Database={Database};Username={Username}";
}

class ServerConfig
{
    public string Host { get; set; } = "0.0.0.0";
    public int Port { get; set; } = 8080;
    public bool Https { get; set; } = false;
    public string? CertPath { get; set; }
    public int MaxRequestSize { get; set; } = 10 * 1024 * 1024; // 10MB
    public List<string> AllowedOrigins { get; set; } = new() { "*" };
}

class LoggingConfig
{
    public string Level { get; set; } = "Information";
    public bool ConsoleEnabled { get; set; } = true;
    public bool FileEnabled { get; set; } = true;
    public string FilePath { get; set; } = "logs/app.log";
    public int MaxFileSizeMb { get; set; } = 10;
    public int RetainedFiles { get; set; } = 5;
}

class ConfigManager
{
    private static readonly JsonSerializerOptions _options = new()
    {
        WriteIndented = true,
        PropertyNameCaseInsensitive = true,
        AllowTrailingCommas = true,
        ReadCommentHandling = JsonCommentHandling.Skip,
    };
    
    private readonly string _configPath;
    private AppConfig _config;
    
    public AppConfig Config => _config;
    
    public ConfigManager(string configPath = "appsettings.json")
    {
        _configPath = configPath;
        _config = LoadOrCreate();
    }
    
    private AppConfig LoadOrCreate()
    {
        if (!File.Exists(_configPath))
        {
            Console.WriteLine($"Config not found, creating defaults at {_configPath}");
            var defaults = CreateDefaults();
            Save(defaults);
            return defaults;
        }
        
        try
        {
            string json = File.ReadAllText(_configPath);
            return JsonSerializer.Deserialize<AppConfig>(json, _options) ?? CreateDefaults();
        }
        catch (JsonException ex)
        {
            Console.WriteLine($"Warning: Invalid config JSON ({ex.Message}), using defaults");
            return CreateDefaults();
        }
    }
    
    private static AppConfig CreateDefaults() => new()
    {
        Features = new Dictionary<string, string>
        {
            ["new-ui"] = "false",
            ["analytics"] = "true",
            ["beta-features"] = "false"
        }
    };
    
    public void Save(AppConfig? config = null)
    {
        string json = JsonSerializer.Serialize(config ?? _config, _options);
        File.WriteAllText(_configPath, json);
    }
    
    // Override with environment-specific config
    public void MergeWith(string envConfigPath)
    {
        if (!File.Exists(envConfigPath)) return;
        
        try
        {
            string baseJson = File.ReadAllText(_configPath);
            string envJson = File.ReadAllText(envConfigPath);
            
            var baseNode = JsonNode.Parse(baseJson)?.AsObject();
            var envNode = JsonNode.Parse(envJson)?.AsObject();
            
            if (baseNode != null && envNode != null)
            {
                MergeObjects(baseNode, envNode);
                _config = JsonSerializer.Deserialize<AppConfig>(baseNode.ToJsonString(), _options) ?? _config;
            }
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Warning: Failed to merge env config: {ex.Message}");
        }
    }
    
    private static void MergeObjects(JsonObject target, JsonObject source)
    {
        foreach (var property in source)
        {
            if (property.Value is JsonObject sourceObj && 
                target[property.Key] is JsonObject targetObj)
            {
                MergeObjects(targetObj, sourceObj);
            }
            else
            {
                target[property.Key] = property.Value?.DeepClone();
            }
        }
    }
    
    // Feature flag check
    public bool IsFeatureEnabled(string feature) =>
        _config.Features.TryGetValue(feature, out string? value) &&
        bool.TryParse(value, out bool enabled) && enabled;
    
    // Update a single setting path
    public void UpdateSetting(string path, string value)
    {
        string json = File.ReadAllText(_configPath);
        var node = JsonNode.Parse(json)?.AsObject();
        if (node == null) return;
        
        string[] parts = path.Split('.');
        JsonObject current = node;
        
        for (int i = 0; i < parts.Length - 1; i++)
        {
            if (current[parts[i]] is not JsonObject next)
            {
                next = new JsonObject();
                current[parts[i]] = next;
            }
            current = next;
        }
        
        current[parts[^1]] = value;
        File.WriteAllText(_configPath, node.ToJsonString(new JsonSerializerOptions { WriteIndented = true }));
        _config = LoadOrCreate();
    }
}

// Demo
string configPath = Path.Combine(Path.GetTempPath(), "demo_config.json");

var mgr = new ConfigManager(configPath);
var config = mgr.Config;

Console.WriteLine("=== Configuration Loaded ===");
Console.WriteLine($"DB:     {config.Database.ConnectionString}");
Console.WriteLine($"Server: {config.Server.Host}:{config.Server.Port}");
Console.WriteLine($"Log:    Level={config.Logging.Level}, File={config.Logging.FilePath}");
Console.WriteLine($"Features: {string.Join(", ", config.Features.Select(f => $"{f.Key}={f.Value}"))}");

Console.WriteLine($"\nFeature 'analytics' enabled: {mgr.IsFeatureEnabled("analytics")}");
Console.WriteLine($"Feature 'new-ui' enabled: {mgr.IsFeatureEnabled("new-ui")}");

// Update setting
mgr.UpdateSetting("Server.Port", "9090");
Console.WriteLine($"\nUpdated port: {mgr.Config.Server.Port}");

// Print config as JSON
string outputJson = JsonSerializer.Serialize(mgr.Config, new JsonSerializerOptions { WriteIndented = true });
Console.WriteLine("\n=== Full Config ===");
Console.WriteLine(outputJson);

File.Delete(configPath);
```

---

## สรุป Part 17

### System.Text.Json
- ✅ `JsonSerializer.Serialize`/`Deserialize`
- ✅ `JsonSerializerOptions` — indent, naming policy, null handling
- ✅ Attributes: `[JsonPropertyName]`, `[JsonIgnore]`, `[JsonRequired]`
- ✅ `[JsonConverter]` for custom type conversion
- ✅ `[JsonPolymorphic]` for polymorphism (C# 7+)
- ✅ Source generators `[JsonSerializable]` — faster, AOT-friendly

### JSON DOM
- ✅ `JsonNode`, `JsonObject`, `JsonArray` — mutable DOM
- ✅ `JsonDocument`, `JsonElement` — immutable, zero-copy
- ✅ Deep merge of JSON objects

### Newtonsoft.Json
- ✅ `JsonConvert.SerializeObject`/`DeserializeObject`
- ✅ `JObject`, `JArray`, `JToken` — dynamic manipulation
- ✅ LINQ to JSON — query with Where/Select
- ✅ `JsonSerializerSettings` — comprehensive options

### HTTP + JSON
- ✅ `GetFromJsonAsync`, `PostAsJsonAsync`, `ReadFromJsonAsync`
- ✅ Typed HTTP clients
- ✅ Custom serialization options per request

---

## แบบฝึกหัด

1. สร้าง `JsonMapper<TSource, TDest>` ที่แปลง JSON structure ระหว่าง 2 formats
2. เขียน `JsonValidator` ที่ validate JSON ตาม JSON Schema
3. สร้าง `JsonDiff` ที่เปรียบเทียบ 2 JSON objects และ return รายการ differences
4. Implement `JsonPatch` (RFC 6902) ที่ apply patch operations ไปยัง JSON document
5. สร้าง strongly-typed API wrapper สำหรับ OpenWeatherMap API โดยใช้ `IHttpClientFactory`
