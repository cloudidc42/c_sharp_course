# Part 43: Cloud-Native .NET (Steps 1241-1280)

## Step 1241: Cloud-Native Principles

### 12-Factor App Methodology for .NET

```
┌────────────────────────────────────────────────────────────────┐
│                  12-Factor App Principles                       │
├────────────────────────────────────────────────────────────────┤
│  1. Codebase       One repo, many deploys                      │
│  2. Dependencies   Explicit via NuGet, lock files              │
│  3. Config         Environment variables / Secrets             │
│  4. Backing Services  Attached resources (DB, Cache)           │
│  5. Build/Release/Run  Strict separation stages                │
│  6. Processes      Stateless, share-nothing                    │
│  7. Port Binding   Export services via port (Kestrel)          │
│  8. Concurrency    Scale via process model (K8s pods)          │
│  9. Disposability  Fast startup, graceful shutdown             │
│ 10. Dev/Prod Parity  Keep environments similar                 │
│ 11. Logs           Treat as event streams (stdout)             │
│ 12. Admin Processes  One-off tasks via same codebase           │
└────────────────────────────────────────────────────────────────┘
```

### .NET Cloud-Native Checklist

```csharp
// 1. Health checks (readiness + liveness)
builder.Services.AddHealthChecks()
    .AddDbContextCheck<AppDbContext>()
    .AddRedis(redisConnectionString)
    .AddRabbitMQ();

// 2. Structured logging to stdout
builder.Logging.AddJsonConsole(o =>
{
    o.IncludeScopes = true;
    o.TimestampFormat = "yyyy-MM-ddTHH:mm:ssZ";
});

// 3. OpenTelemetry tracing + metrics
builder.Services.AddOpenTelemetry()
    .WithTracing(t => t.AddAspNetCoreInstrumentation().AddOtlpExporter())
    .WithMetrics(m => m.AddAspNetCoreInstrumentation().AddOtlpExporter());

// 4. Graceful shutdown
builder.Services.Configure<HostOptions>(o =>
{
    o.ShutdownTimeout = TimeSpan.FromSeconds(30);
});

// 5. Configuration from environment
builder.Configuration
    .AddEnvironmentVariables()
    .AddJsonFile("appsettings.json")
    .AddJsonFile($"appsettings.{builder.Environment.EnvironmentName}.json", optional: true);

// 6. Resilience (Polly)
builder.Services.AddHttpClient<IExternalService, ExternalService>()
    .AddStandardResilienceHandler();
```

---

## Step 1242: Docker Best Practices for .NET

### Optimized Dockerfile

```dockerfile
# Stage 1: Restore
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS restore
WORKDIR /src

# Copy only project files for layer caching
COPY ["src/MyApp.API/MyApp.API.csproj", "src/MyApp.API/"]
COPY ["src/MyApp.Core/MyApp.Core.csproj", "src/MyApp.Core/"]
COPY ["src/MyApp.Infrastructure/MyApp.Infrastructure.csproj", "src/MyApp.Infrastructure/"]

RUN dotnet restore "src/MyApp.API/MyApp.API.csproj"

# Stage 2: Build
FROM restore AS build
COPY . .
RUN dotnet build "src/MyApp.API/MyApp.API.csproj" -c Release --no-restore

# Stage 3: Test
FROM build AS test
RUN dotnet test "tests/MyApp.Tests/MyApp.Tests.csproj" \
    -c Release \
    --no-build \
    --logger "trx;LogFileName=/test-results/results.trx" \
    --results-directory /test-results

# Stage 4: Publish
FROM build AS publish
RUN dotnet publish "src/MyApp.API/MyApp.API.csproj" \
    -c Release \
    --no-build \
    -o /app/publish \
    /p:UseAppHost=false

# Stage 5: Runtime (minimal image)
FROM mcr.microsoft.com/dotnet/aspnet:9.0-alpine AS runtime
WORKDIR /app

# Security: run as non-root
RUN adduser -S appuser -G appgroup 2>/dev/null || \
    addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

COPY --from=publish /app/publish .

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD wget -qO- http://localhost:8080/health || exit 1

EXPOSE 8080
ENTRYPOINT ["dotnet", "MyApp.API.dll"]
```

### .dockerignore

```
**/.vs
**/.git
**/bin
**/obj
**/TestResults
**/*.user
**/*.md
.dockerignore
Dockerfile*
docker-compose*.yml
```

### Docker Compose for Development

```yaml
version: '3.8'

services:
  api:
    build:
      context: .
      target: runtime
    ports:
      - "8080:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__Default=Host=postgres;Database=myapp;Username=postgres;Password=dev_password
      - ConnectionStrings__Redis=redis:6379
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - ./src:/app/src:ro  # hot reload in dev

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: dev_password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  seq:
    image: datalust/seq:latest
    ports:
      - "5341:80"
    environment:
      ACCEPT_EULA: Y

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"

volumes:
  postgres_data:
```

---

## Step 1243: Kubernetes Deployment

### Namespace and ConfigMap

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
  labels:
    name: myapp

---
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
  namespace: myapp
data:
  ASPNETCORE_ENVIRONMENT: "Production"
  ASPNETCORE_URLS: "http://+:8080"
  ConnectionStrings__Redis: "redis-service:6379"
  Logging__LogLevel__Default: "Information"
```

### Secret Management

```yaml
# secret.yaml (use sealed-secrets or External Secrets Operator in production)
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secrets
  namespace: myapp
type: Opaque
stringData:
  ConnectionStrings__Default: "Host=postgres-service;Database=myapp;Username=app;Password=secret"
  JWT__Secret: "your-jwt-secret-key"
```

### Deployment

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-api
  namespace: myapp
  labels:
    app: myapp-api
    version: v1
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: myapp-api
  template:
    metadata:
      labels:
        app: myapp-api
        version: v1
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: myapp-sa
      
      # Security context
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
      
      containers:
        - name: api
          image: myregistry.azurecr.io/myapp-api:v1.2.3
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
              name: http
          
          envFrom:
            - configMapRef:
                name: myapp-config
            - secretRef:
                name: myapp-secrets
          
          # Resources
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          
          # Health probes
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
            failureThreshold: 3
          
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 30
            failureThreshold: 3
          
          startupProbe:
            httpGet:
              path: /health/live
              port: 8080
            failureThreshold: 30
            periodSeconds: 2
          
          # Graceful shutdown
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 5"]
          
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      
      volumes:
        - name: tmp
          emptyDir: {}
      
      terminationGracePeriodSeconds: 60
      
      # Topology spread
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: myapp-api
```

### Service and Ingress

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-api-service
  namespace: myapp
spec:
  selector:
    app: myapp-api
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
  type: ClusterIP

---
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: myapp
  annotations:
    nginx.ingress.kubernetes.io/rate-limit: "100"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
    - hosts:
        - api.myapp.com
      secretName: myapp-tls
  rules:
    - host: api.myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp-api-service
                port:
                  number: 80

---
# HPA (Horizontal Pod Autoscaler)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-api-hpa
  namespace: myapp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp-api
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

---

## Step 1244: Azure - App Service and Container Apps

### Azure Container Apps Bicep

```bicep
// main.bicep
param location string = resourceGroup().location
param environmentName string = 'prod'
param appName string = 'myapp-api'
param containerImage string
param containerRegistry string

// Container Apps Environment
resource containerAppsEnv 'Microsoft.App/managedEnvironments@2024-03-01' = {
  name: '${environmentName}-env'
  location: location
  properties: {
    appLogsConfiguration: {
      destination: 'log-analytics'
      logAnalyticsConfiguration: {
        customerId: logAnalytics.properties.customerId
        sharedKey: logAnalytics.listKeys().primarySharedKey
      }
    }
  }
}

// Container App
resource containerApp 'Microsoft.App/containerApps@2024-03-01' = {
  name: appName
  location: location
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    managedEnvironmentId: containerAppsEnv.id
    configuration: {
      ingress: {
        external: true
        targetPort: 8080
        transport: 'http'
        traffic: [
          {
            weight: 100
            latestRevision: true
          }
        ]
      }
      registries: [
        {
          server: containerRegistry
          identity: 'system'
        }
      ]
      secrets: [
        {
          name: 'db-connection-string'
          keyVaultUrl: 'https://${keyVault.name}.vault.azure.net/secrets/DbConnectionString'
          identity: 'system'
        }
      ]
    }
    template: {
      revisionSuffix: 'v1'
      containers: [
        {
          name: appName
          image: containerImage
          resources: {
            cpu: json('0.5')
            memory: '1.0Gi'
          }
          env: [
            {
              name: 'ASPNETCORE_ENVIRONMENT'
              value: 'Production'
            }
            {
              name: 'ConnectionStrings__Default'
              secretRef: 'db-connection-string'
            }
          ]
          probes: [
            {
              type: 'Readiness'
              httpGet: {
                path: '/health/ready'
                port: 8080
              }
            }
          ]
        }
      ]
      scale: {
        minReplicas: 1
        maxReplicas: 10
        rules: [
          {
            name: 'http-scaling'
            http: {
              metadata: {
                concurrentRequests: '100'
              }
            }
          }
        ]
      }
    }
  }
}

output containerAppUrl string = 'https://${containerApp.properties.latestRevisionFqdn}'
```

---

## Step 1245: Azure Key Vault Integration

### Key Vault Configuration Provider

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Add Azure Key Vault as config source
if (builder.Environment.IsProduction())
{
    var keyVaultUri = builder.Configuration["KeyVault:Uri"]
        ?? throw new InvalidOperationException("Key Vault URI not configured");

    builder.Configuration.AddAzureKeyVault(
        new Uri(keyVaultUri),
        new DefaultAzureCredential());
}

builder.Services.AddAzureClients(clients =>
{
    // Secret client
    clients.AddSecretClient(
        new Uri(builder.Configuration["KeyVault:Uri"]!));

    // Service Bus
    clients.AddServiceBusClient(
        builder.Configuration["ServiceBus:ConnectionString"]!);

    // Configure default credentials
    clients.UseCredential(new DefaultAzureCredential());
});
```

### Managed Identity Pattern

```csharp
using Azure.Identity;
using Azure.Security.KeyVault.Secrets;

public class SecretManager
{
    private readonly SecretClient _secretClient;
    private readonly IMemoryCache _cache;

    public SecretManager(SecretClient secretClient, IMemoryCache cache)
    {
        _secretClient = secretClient;
        _cache = cache;
    }

    public async Task<string> GetSecretAsync(string secretName)
    {
        return await _cache.GetOrCreateAsync(
            $"secret:{secretName}",
            async entry =>
            {
                entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5);

                var secret = await _secretClient.GetSecretAsync(secretName);
                return secret.Value.Value;
            }) ?? throw new KeyNotFoundException($"Secret '{secretName}' not found");
    }

    public async Task SetSecretAsync(string secretName, string value)
    {
        await _secretClient.SetSecretAsync(secretName, value);
        _cache.Remove($"secret:{secretName}");
    }
}
```

---

## Step 1246: Azure Service Bus

### Service Bus Publisher

```csharp
using Azure.Messaging.ServiceBus;
using System.Text.Json;

public class ServiceBusPublisher : IAsyncDisposable
{
    private readonly ServiceBusClient _client;
    private readonly ServiceBusSender _sender;

    public ServiceBusPublisher(ServiceBusClient client, string queueOrTopicName)
    {
        _client = client;
        _sender = client.CreateSender(queueOrTopicName);
    }

    public async Task PublishAsync<T>(
        T message,
        string? sessionId = null,
        string? correlationId = null,
        TimeSpan? timeToLive = null,
        CancellationToken ct = default) where T : class
    {
        var body = JsonSerializer.SerializeToUtf8Bytes(message);
        var sbMessage = new ServiceBusMessage(body)
        {
            ContentType = "application/json",
            Subject = typeof(T).Name,
            CorrelationId = correlationId ?? Guid.NewGuid().ToString(),
            SessionId = sessionId,
        };

        if (timeToLive.HasValue)
            sbMessage.TimeToLive = timeToLive.Value;

        await _sender.SendMessageAsync(sbMessage, ct);
    }

    public async Task PublishBatchAsync<T>(
        IReadOnlyList<T> messages,
        CancellationToken ct = default) where T : class
    {
        using var batch = await _sender.CreateMessageBatchAsync(ct);

        foreach (var message in messages)
        {
            var body = JsonSerializer.SerializeToUtf8Bytes(message);
            var sbMessage = new ServiceBusMessage(body)
            {
                ContentType = "application/json",
                Subject = typeof(T).Name,
            };

            if (!batch.TryAddMessage(sbMessage))
            {
                // Batch full, send and create new
                await _sender.SendMessagesAsync(batch, ct);
            }
        }

        if (batch.Count > 0)
            await _sender.SendMessagesAsync(batch, ct);
    }

    public async ValueTask DisposeAsync()
    {
        await _sender.DisposeAsync();
        await _client.DisposeAsync();
    }
}
```

### Service Bus Consumer

```csharp
public class OrderProcessorWorker : BackgroundService
{
    private readonly ServiceBusClient _client;
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<OrderProcessorWorker> _logger;
    private ServiceBusProcessor? _processor;

    public OrderProcessorWorker(
        ServiceBusClient client,
        IServiceProvider serviceProvider,
        ILogger<OrderProcessorWorker> logger)
    {
        _client = client;
        _serviceProvider = serviceProvider;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _processor = _client.CreateProcessor("orders", new ServiceBusProcessorOptions
        {
            MaxConcurrentCalls = 10,
            AutoCompleteMessages = false,
            MaxAutoLockRenewalDuration = TimeSpan.FromMinutes(5),
        });

        _processor.ProcessMessageAsync += ProcessMessageAsync;
        _processor.ProcessErrorAsync += ProcessErrorAsync;

        await _processor.StartProcessingAsync(stoppingToken);

        await Task.Delay(Timeout.Infinite, stoppingToken);
    }

    private async Task ProcessMessageAsync(ProcessMessageEventArgs args)
    {
        try
        {
            await using var scope = _serviceProvider.CreateAsyncScope();
            var handler = scope.ServiceProvider.GetRequiredService<IOrderEventHandler>();

            var order = args.Message.Body.ToObjectFromJson<OrderCreatedEvent>();
            await handler.HandleAsync(order, args.CancellationToken);

            await args.CompleteMessageAsync(args.Message);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error processing message {MessageId}",
                args.Message.MessageId);

            if (args.Message.DeliveryCount >= 3)
            {
                await args.DeadLetterMessageAsync(args.Message,
                    deadLetterReason: "MaxDeliveryCountExceeded",
                    deadLetterErrorDescription: ex.Message);
            }
            else
            {
                await args.AbandonMessageAsync(args.Message);
            }
        }
    }

    private Task ProcessErrorAsync(ProcessErrorEventArgs args)
    {
        _logger.LogError(args.Exception,
            "Service Bus error. Source={Source}, Entity={Entity}",
            args.ErrorSource, args.EntityPath);
        return Task.CompletedTask;
    }

    public override async Task StopAsync(CancellationToken cancellationToken)
    {
        if (_processor is not null)
            await _processor.StopProcessingAsync(cancellationToken);
        await base.StopAsync(cancellationToken);
    }
}
```

---

## Step 1247: AWS Lambda with .NET

### Lambda Function Setup

```xml
<!-- MyLambda.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <GenerateRuntimeConfigurationFiles>true</GenerateRuntimeConfigurationFiles>
    <AWSProjectType>Lambda</AWSProjectType>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <!-- AOT for cold start improvement -->
    <PublishAot>true</PublishAot>
  </PropertyGroup>

  <PackageReference Include="Amazon.Lambda.Core" Version="2.5.0" />
  <PackageReference Include="Amazon.Lambda.APIGatewayEvents" Version="2.7.0" />
  <PackageReference Include="Amazon.Lambda.RuntimeSupport" Version="1.10.0" />
  <PackageReference Include="Amazon.Lambda.Serialization.SystemTextJson" Version="2.4.4" />
</Project>
```

### Lambda Handler

```csharp
using Amazon.Lambda.APIGatewayEvents;
using Amazon.Lambda.Core;
using Amazon.Lambda.RuntimeSupport;
using Amazon.Lambda.Serialization.SystemTextJson;
using System.Text.Json;
using System.Text.Json.Serialization;

// AOT-compatible JSON context
[JsonSerializable(typeof(APIGatewayHttpApiV2ProxyRequest))]
[JsonSerializable(typeof(APIGatewayHttpApiV2ProxyResponse))]
[JsonSerializable(typeof(OrderRequest))]
[JsonSerializable(typeof(OrderResponse))]
internal partial class LambdaJsonContext : JsonSerializerContext { }

// Function handler
public class Function
{
    private static readonly LambdaJsonContext JsonContext = LambdaJsonContext.Default;

    public static async Task<APIGatewayHttpApiV2ProxyResponse> FunctionHandler(
        APIGatewayHttpApiV2ProxyRequest request,
        ILambdaContext context)
    {
        context.Logger.LogInformation(
            "Processing request {RequestId} for path {Path}",
            request.RequestContext.RequestId,
            request.RequestContext.Http.Path);

        try
        {
            var orderRequest = JsonSerializer.Deserialize(
                request.Body,
                JsonContext.OrderRequest)!;

            // Process order
            var order = await ProcessOrderAsync(orderRequest);

            var responseBody = JsonSerializer.Serialize(order, JsonContext.OrderResponse);

            return new APIGatewayHttpApiV2ProxyResponse
            {
                StatusCode = 200,
                Body = responseBody,
                Headers = new Dictionary<string, string>
                {
                    ["Content-Type"] = "application/json",
                    ["X-Request-Id"] = request.RequestContext.RequestId,
                }
            };
        }
        catch (Exception ex)
        {
            context.Logger.LogError(ex, "Error processing order");

            return new APIGatewayHttpApiV2ProxyResponse
            {
                StatusCode = 500,
                Body = JsonSerializer.Serialize(
                    new { error = "Internal server error" }),
                Headers = new Dictionary<string, string>
                {
                    ["Content-Type"] = "application/json",
                }
            };
        }
    }

    private static Task<OrderResponse> ProcessOrderAsync(OrderRequest request)
    {
        // Business logic here
        return Task.FromResult(new OrderResponse
        {
            OrderId = Guid.NewGuid().ToString(),
            Status = "Created",
            CreatedAt = DateTime.UtcNow,
        });
    }

    // Entry point for Lambda
    private static async Task Main()
    {
        await LambdaBootstrapBuilder
            .Create(FunctionHandler,
                new SourceGeneratorLambdaJsonSerializer<LambdaJsonContext>())
            .Build()
            .RunAsync();
    }
}

public record OrderRequest(string ProductId, int Quantity, string CustomerId);
public record OrderResponse(string OrderId, string Status, DateTime CreatedAt);
```

### SAM Template

```yaml
# template.yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Globals:
  Function:
    Timeout: 30
    MemorySize: 256
    Runtime: provided.al2023
    Architectures:
      - arm64
    Environment:
      Variables:
        ASPNETCORE_ENVIRONMENT: !Ref Environment

Parameters:
  Environment:
    Type: String
    Default: dev
    AllowedValues: [dev, staging, prod]

Resources:
  OrderFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: ./src/MyLambda/
      Handler: bootstrap
      Events:
        CreateOrder:
          Type: HttpApi
          Properties:
            Path: /orders
            Method: POST
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref OrdersTable
        - SQSSendMessagePolicy:
            QueueName: !GetAtt OrderQueue.QueueName

  OrdersTable:
    Type: AWS::DynamoDB::Table
    Properties:
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - AttributeName: PK
          AttributeType: S
        - AttributeName: SK
          AttributeType: S
      KeySchema:
        - AttributeName: PK
          KeyType: HASH
        - AttributeName: SK
          KeyType: RANGE

  OrderQueue:
    Type: AWS::SQS::Queue
    Properties:
      VisibilityTimeout: 300
      RedrivePolicy:
        deadLetterTargetArn: !GetAtt OrderDLQ.Arn
        maxReceiveCount: 3

  OrderDLQ:
    Type: AWS::SQS::Queue
```

---

## Step 1248: AWS DynamoDB with .NET

### DynamoDB Service

```csharp
using Amazon.DynamoDBv2;
using Amazon.DynamoDBv2.DocumentModel;
using Amazon.DynamoDBv2.Model;

public class DynamoOrderRepository : IOrderRepository
{
    private readonly IAmazonDynamoDB _dynamoDB;
    private readonly Table _table;
    private const string TableName = "Orders";

    public DynamoOrderRepository(IAmazonDynamoDB dynamoDB)
    {
        _dynamoDB = dynamoDB;
        _table = Table.LoadTable(dynamoDB, TableName);
    }

    public async Task SaveAsync(Order order, CancellationToken ct = default)
    {
        var doc = new Document
        {
            ["PK"] = $"CUSTOMER#{order.CustomerId}",
            ["SK"] = $"ORDER#{order.Id}",
            ["OrderId"] = order.Id.ToString(),
            ["CustomerId"] = order.CustomerId.ToString(),
            ["Status"] = order.Status.ToString(),
            ["TotalAmount"] = order.TotalAmount.Amount,
            ["Currency"] = order.TotalAmount.Currency,
            ["CreatedAt"] = order.CreatedAt.ToString("O"),
            ["GSI1PK"] = $"STATUS#{order.Status}",
            ["GSI1SK"] = $"DATE#{order.CreatedAt:yyyy-MM-dd}",
        };

        await _table.PutItemAsync(doc, ct);
    }

    public async Task<Order?> FindByIdAsync(
        OrderId orderId,
        CustomerId customerId,
        CancellationToken ct = default)
    {
        var doc = await _table.GetItemAsync(
            new Primitive($"CUSTOMER#{customerId}"),
            new Primitive($"ORDER#{orderId}"),
            ct);

        return doc is null ? null : MapToOrder(doc);
    }

    public async Task<IReadOnlyList<Order>> GetByStatusAsync(
        OrderStatus status,
        DateOnly date,
        CancellationToken ct = default)
    {
        var request = new QueryRequest
        {
            TableName = TableName,
            IndexName = "GSI1",
            KeyConditionExpression = "GSI1PK = :pk AND begins_with(GSI1SK, :sk)",
            ExpressionAttributeValues = new Dictionary<string, AttributeValue>
            {
                [":pk"] = new AttributeValue($"STATUS#{status}"),
                [":sk"] = new AttributeValue($"DATE#{date:yyyy-MM-dd}"),
            }
        };

        var response = await _dynamoDB.QueryAsync(request, ct);
        return response.Items.Select(MapAttributesToOrder).ToList();
    }

    public async Task UpdateStatusAsync(
        OrderId orderId,
        CustomerId customerId,
        OrderStatus newStatus,
        CancellationToken ct = default)
    {
        var request = new UpdateItemRequest
        {
            TableName = TableName,
            Key = new Dictionary<string, AttributeValue>
            {
                ["PK"] = new AttributeValue($"CUSTOMER#{customerId}"),
                ["SK"] = new AttributeValue($"ORDER#{orderId}"),
            },
            UpdateExpression = "SET #status = :newStatus, UpdatedAt = :now",
            ConditionExpression = "attribute_exists(PK)",
            ExpressionAttributeNames = new Dictionary<string, string>
            {
                ["#status"] = "Status"
            },
            ExpressionAttributeValues = new Dictionary<string, AttributeValue>
            {
                [":newStatus"] = new AttributeValue(newStatus.ToString()),
                [":now"] = new AttributeValue(DateTime.UtcNow.ToString("O")),
            }
        };

        await _dynamoDB.UpdateItemAsync(request, ct);
    }

    private static Order MapToOrder(Document doc)
    {
        // Map DynamoDB document back to Order aggregate
        throw new NotImplementedException("Map document to domain entity");
    }

    private static Order MapAttributesToOrder(Dictionary<string, AttributeValue> attrs)
    {
        throw new NotImplementedException("Map attributes to domain entity");
    }
}
```

---

## Step 1249: Google Cloud Platform - Cloud Run

### Cloud Run Deployment

```yaml
# cloudbuild.yaml
steps:
  # Build Docker image
  - name: 'gcr.io/cloud-builders/docker'
    args:
      - 'build'
      - '-t'
      - '$_REGION-docker.pkg.dev/$PROJECT_ID/$_REPO/$_SERVICE:$COMMIT_SHA'
      - '--cache-from'
      - '$_REGION-docker.pkg.dev/$PROJECT_ID/$_REPO/$_SERVICE:latest'
      - '.'

  # Push to Artifact Registry
  - name: 'gcr.io/cloud-builders/docker'
    args:
      - 'push'
      - '$_REGION-docker.pkg.dev/$PROJECT_ID/$_REPO/$_SERVICE:$COMMIT_SHA'

  # Deploy to Cloud Run
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    entrypoint: gcloud
    args:
      - 'run'
      - 'deploy'
      - '$_SERVICE'
      - '--image=$_REGION-docker.pkg.dev/$PROJECT_ID/$_REPO/$_SERVICE:$COMMIT_SHA'
      - '--region=$_REGION'
      - '--platform=managed'
      - '--service-account=myapp-sa@$PROJECT_ID.iam.gserviceaccount.com'
      - '--min-instances=1'
      - '--max-instances=100'
      - '--memory=512Mi'
      - '--cpu=1'
      - '--concurrency=100'
      - '--timeout=30s'
      - '--set-env-vars=ASPNETCORE_ENVIRONMENT=Production'
      - '--set-secrets=DB_CONNECTION=db-connection:latest'

substitutions:
  _REGION: asia-southeast1
  _REPO: myapp
  _SERVICE: myapp-api
```

### Cloud Spanner Integration

```csharp
using Google.Cloud.Spanner.Data;

public class SpannerOrderRepository
{
    private readonly string _connectionString;

    public SpannerOrderRepository(IConfiguration config)
    {
        _connectionString = config["Spanner:ConnectionString"]!;
    }

    public async Task<Order?> FindByIdAsync(Guid orderId)
    {
        await using var connection = new SpannerConnection(_connectionString);
        await connection.OpenAsync();

        var command = connection.CreateSelectCommand(
            "SELECT OrderId, CustomerId, Status, TotalAmount, CreatedAt " +
            "FROM Orders WHERE OrderId = @orderId",
            new SpannerParameterCollection
            {
                ["orderId"] = SpannerDbType.String,
            });

        command.Parameters["orderId"].Value = orderId.ToString();

        await using var reader = await command.ExecuteReaderAsync();
        if (!await reader.ReadAsync()) return null;

        return new Order
        {
            // Map columns to domain model
        };
    }

    public async Task SaveAsync(Order order)
    {
        await using var connection = new SpannerConnection(_connectionString);
        await connection.OpenAsync();

        var mutation = SpannerMutation.Insert(
            tableName: "Orders",
            columns: ["OrderId", "CustomerId", "Status", "TotalAmount", "CreatedAt"],
            values: [
                order.Id.ToString(),
                order.CustomerId.ToString(),
                order.Status.ToString(),
                order.TotalAmount.Amount,
                order.CreatedAt,
            ]);

        await connection.RunWithRetriableTransactionAsync(tx =>
            tx.Commit([mutation]));
    }
}
```

---

## Step 1250: Service Mesh with Istio

### Istio Configuration for .NET Services

```yaml
# virtual-service.yaml - Traffic management
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp-api
  namespace: myapp
spec:
  hosts:
    - myapp-api-service
  http:
    # Canary: 10% traffic to v2
    - route:
        - destination:
            host: myapp-api-service
            subset: v1
          weight: 90
        - destination:
            host: myapp-api-service
            subset: v2
          weight: 10
      timeout: 30s
      retries:
        attempts: 3
        perTryTimeout: 10s
        retryOn: 5xx,reset,connect-failure

---
# destination-rule.yaml - Load balancing + circuit breaking
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: myapp-api
  namespace: myapp
spec:
  host: myapp-api-service
  trafficPolicy:
    connectionPool:
      http:
        h2UpgradePolicy: UPGRADE
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
    outlierDetection:
      consecutiveGatewayErrors: 5
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
```

### .NET Distributed Tracing with Istio

```csharp
// Propagate Istio headers for distributed tracing
public class IstioTracingMiddleware
{
    private static readonly string[] TracingHeaders =
    [
        "x-request-id",
        "x-b3-traceid",
        "x-b3-spanid",
        "x-b3-parentspanid",
        "x-b3-sampled",
        "x-b3-flags",
        "x-ot-span-context",
    ];

    private readonly RequestDelegate _next;

    public IstioTracingMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(HttpContext context, IHttpClientFactory factory)
    {
        // The headers are automatically propagated by OpenTelemetry
        // but we can also manually propagate them for downstream calls
        await _next(context);
    }
}

// HttpClient propagates trace headers automatically with OpenTelemetry
builder.Services.AddHttpClient("downstream")
    .AddHttpMessageHandler<TracingDelegatingHandler>();

public class TracingDelegatingHandler : DelegatingHandler
{
    private static readonly string[] IstioHeaders =
    [
        "x-request-id", "x-b3-traceid", "x-b3-spanid",
        "x-b3-sampled", "x-b3-flags",
    ];

    private readonly IHttpContextAccessor _httpContextAccessor;

    public TracingDelegatingHandler(IHttpContextAccessor httpContextAccessor)
    {
        _httpContextAccessor = httpContextAccessor;
    }

    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request,
        CancellationToken cancellationToken)
    {
        var incomingHeaders = _httpContextAccessor.HttpContext?.Request.Headers;
        if (incomingHeaders is not null)
        {
            foreach (var header in IstioHeaders)
            {
                if (incomingHeaders.TryGetValue(header, out var value))
                {
                    request.Headers.TryAddWithoutValidation(header, (string?)value);
                }
            }
        }

        return await base.SendAsync(request, cancellationToken);
    }
}
```

---

## Step 1251: Helm Charts for .NET

### Helm Chart Structure

```
myapp-chart/
├── Chart.yaml
├── values.yaml
├── values.prod.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── hpa.yaml
│   ├── configmap.yaml
│   ├── serviceaccount.yaml
│   └── _helpers.tpl
└── charts/
    └── redis-18.x.x.tgz
```

### values.yaml

```yaml
# values.yaml
replicaCount: 2

image:
  repository: myregistry.azurecr.io/myapp-api
  tag: "latest"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: api.myapp.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: myapp-tls
      hosts:
        - api.myapp.com

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70

config:
  environment: production
  logLevel: Information

redis:
  enabled: true
  auth:
    enabled: false
  master:
    persistence:
      enabled: false
```

---

## Step 1252: GitOps with ArgoCD

### ArgoCD Application

```yaml
# argocd-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/myapp.git
    targetRevision: main
    path: deploy/helm
    helm:
      releaseName: myapp
      valueFiles:
        - values.yaml
        - values.prod.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
  revisionHistoryLimit: 10
```

### GitHub Actions CI/CD

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '9.0.x'

      - name: Restore
        run: dotnet restore

      - name: Build
        run: dotnet build --no-restore -c Release

      - name: Test
        run: |
          dotnet test --no-build -c Release \
            --logger "trx;LogFileName=results.trx" \
            --collect:"XPlat Code Coverage"

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}

  build-push:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.build.outputs.digest }}

    steps:
      - uses: actions/checkout@v4

      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=sha-
            type=ref,event=branch
            type=semver,pattern={{version}}

      - name: Build and push
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          provenance: true
          sbom: true

  deploy:
    needs: build-push
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Update Helm values
        run: |
          cd deploy/helm
          yq e '.image.tag = "${{ needs.build-push.outputs.image-tag }}"' \
            -i values.prod.yaml

      - name: Commit and push
        run: |
          git config user.name "GitHub Actions"
          git config user.email "actions@github.com"
          git add deploy/helm/values.prod.yaml
          git commit -m "chore: deploy ${{ github.sha }}"
          git push
```

---

## Step 1253: Observability Stack

### OpenTelemetry Complete Setup

```csharp
// Program.cs - Complete observability setup
using OpenTelemetry;
using OpenTelemetry.Metrics;
using OpenTelemetry.Resources;
using OpenTelemetry.Trace;

var builder = WebApplication.CreateBuilder(args);

var serviceName = "myapp-api";
var serviceVersion = Assembly.GetExecutingAssembly()
    .GetName()
    .Version?.ToString() ?? "1.0.0";

builder.Services.AddOpenTelemetry()
    .ConfigureResource(resource =>
    {
        resource.AddService(
            serviceName: serviceName,
            serviceVersion: serviceVersion,
            serviceInstanceId: Environment.MachineName);

        resource.AddAttributes([
            new("deployment.environment", builder.Environment.EnvironmentName),
            new("cloud.provider", "azure"),
        ]);
    })
    .WithTracing(tracing =>
    {
        tracing
            .AddAspNetCoreInstrumentation(o =>
            {
                o.RecordException = true;
                o.Filter = ctx => !ctx.Request.Path.StartsWithSegments("/health");
            })
            .AddHttpClientInstrumentation(o =>
            {
                o.RecordException = true;
            })
            .AddEntityFrameworkCoreInstrumentation(o =>
            {
                o.SetDbStatementForText = true;
            })
            .AddRedisInstrumentation()
            .AddSource(serviceName)
            .AddOtlpExporter(o =>
            {
                o.Endpoint = new Uri(builder.Configuration["OTEL_EXPORTER_OTLP_ENDPOINT"]!);
            });
    })
    .WithMetrics(metrics =>
    {
        metrics
            .AddAspNetCoreInstrumentation()
            .AddHttpClientInstrumentation()
            .AddRuntimeInstrumentation()
            .AddProcessInstrumentation()
            .AddMeter(serviceName)
            .AddOtlpExporter();
    });

// Custom metrics
builder.Services.AddSingleton<AppMetrics>();

// Logging with OpenTelemetry
builder.Logging.AddOpenTelemetry(logging =>
{
    logging.IncludeScopes = true;
    logging.IncludeFormattedMessage = true;
    logging.AddOtlpExporter();
});
```

### Custom Metrics

```csharp
using System.Diagnostics;
using System.Diagnostics.Metrics;

public class AppMetrics
{
    public static readonly ActivitySource ActivitySource =
        new ActivitySource("MyApp.API");

    private readonly Meter _meter;
    private readonly Counter<long> _ordersCreated;
    private readonly Histogram<double> _orderProcessingTime;
    private readonly ObservableGauge<int> _activeConnections;

    private int _activeConnectionCount;

    public AppMetrics()
    {
        _meter = new Meter("MyApp.API", "1.0.0");

        _ordersCreated = _meter.CreateCounter<long>(
            "orders.created",
            unit: "{orders}",
            description: "Number of orders created");

        _orderProcessingTime = _meter.CreateHistogram<double>(
            "orders.processing_duration",
            unit: "ms",
            description: "Order processing duration");

        _activeConnections = _meter.CreateObservableGauge<int>(
            "connections.active",
            () => _activeConnectionCount,
            unit: "{connections}",
            description: "Active WebSocket connections");
    }

    public void RecordOrderCreated(string customerId, string status)
    {
        _ordersCreated.Add(1,
            new TagList
            {
                { "customer.tier", GetCustomerTier(customerId) },
                { "order.status", status },
            });
    }

    public IDisposable MeasureOrderProcessing()
    {
        var sw = Stopwatch.StartNew();
        return new DurationRecorder(() =>
            _orderProcessingTime.Record(sw.Elapsed.TotalMilliseconds));
    }

    public void IncrementConnections() =>
        Interlocked.Increment(ref _activeConnectionCount);

    public void DecrementConnections() =>
        Interlocked.Decrement(ref _activeConnectionCount);

    private static string GetCustomerTier(string customerId) => "standard";

    private sealed class DurationRecorder(Action onDispose) : IDisposable
    {
        public void Dispose() => onDispose();
    }
}
```

---

## Step 1254: Multi-Region Deployment

### Global Traffic Manager Pattern

```csharp
// Multi-region health check
public class RegionHealthService : BackgroundService
{
    private readonly HttpClient _httpClient;
    private readonly ILogger<RegionHealthService> _logger;
    private readonly Dictionary<string, RegionStatus> _regionStatuses = [];

    private static readonly string[] Regions =
    [
        "https://us-east.myapp.com",
        "https://eu-west.myapp.com",
        "https://ap-southeast.myapp.com",
    ];

    public RegionHealthService(
        IHttpClientFactory factory,
        ILogger<RegionHealthService> logger)
    {
        _httpClient = factory.CreateClient();
        _httpClient.Timeout = TimeSpan.FromSeconds(5);
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            var checks = Regions.Select(region => CheckRegionAsync(region, stoppingToken));
            await Task.WhenAll(checks);

            await Task.Delay(TimeSpan.FromSeconds(30), stoppingToken);
        }
    }

    private async Task CheckRegionAsync(string region, CancellationToken ct)
    {
        try
        {
            var response = await _httpClient.GetAsync($"{region}/health", ct);
            _regionStatuses[region] = new RegionStatus
            {
                IsHealthy = response.IsSuccessStatusCode,
                LastChecked = DateTimeOffset.UtcNow,
                StatusCode = (int)response.StatusCode,
            };
        }
        catch (Exception ex)
        {
            _regionStatuses[region] = new RegionStatus
            {
                IsHealthy = false,
                LastChecked = DateTimeOffset.UtcNow,
                Error = ex.Message,
            };
            _logger.LogWarning("Region {Region} is unhealthy: {Error}", region, ex.Message);
        }
    }

    public IReadOnlyDictionary<string, RegionStatus> GetRegionStatuses()
        => _regionStatuses;
}

public class RegionStatus
{
    public bool IsHealthy { get; init; }
    public DateTimeOffset LastChecked { get; init; }
    public int StatusCode { get; init; }
    public string? Error { get; init; }
}
```

---

## Step 1255: Feature Flags with Azure App Configuration

### Feature Flag Setup

```csharp
// Program.cs
builder.Configuration
    .AddAzureAppConfiguration(options =>
    {
        options.Connect(builder.Configuration["AppConfig:ConnectionString"])
            .UseFeatureFlags(ff =>
            {
                ff.CacheExpirationInterval = TimeSpan.FromMinutes(5);
            });
    });

builder.Services.AddFeatureManagement()
    .AddFeatureFilter<TargetingFilter>()
    .AddFeatureFilter<TimeWindowFilter>()
    .AddFeatureFilter<PercentageFilter>();
```

### Feature Flag Usage

```csharp
using Microsoft.FeatureManagement;

public class OrderController(
    IOrderService orderService,
    IFeatureManager featureManager) : ControllerBase
{
    [HttpPost]
    public async Task<IActionResult> CreateOrderAsync(
        CreateOrderRequest request,
        CancellationToken ct)
    {
        // Feature flag check
        if (await featureManager.IsEnabledAsync("NewOrderWorkflow"))
        {
            return await CreateOrderV2Async(request, ct);
        }

        return await CreateOrderV1Async(request, ct);
    }

    [HttpGet("beta-feature")]
    [FeatureGate("BetaFeature")]
    public IActionResult BetaFeature()
    {
        return Ok(new { message = "You have access to beta!" });
    }

    private Task<IActionResult> CreateOrderV1Async(
        CreateOrderRequest request, CancellationToken ct)
        => Task.FromResult<IActionResult>(Ok());

    private Task<IActionResult> CreateOrderV2Async(
        CreateOrderRequest request, CancellationToken ct)
        => Task.FromResult<IActionResult>(Ok());
}
```

---

## Step 1256: Distributed Configuration

### Consul Integration

```csharp
using Winton.Extensions.Configuration.Consul;

// Program.cs
builder.Configuration.AddConsul(
    "myapp/config",
    options =>
    {
        options.ConsulConfigurationOptions = o =>
        {
            o.Address = new Uri("http://consul:8500");
        };
        options.ReloadOnChange = true;
        options.Optional = false;
    });
```

### Vault Integration (HashiCorp)

```csharp
using VaultSharp;
using VaultSharp.V1.AuthMethods.Kubernetes;

public class VaultSecretProvider
{
    private readonly IVaultClient _vaultClient;

    public VaultSecretProvider(IConfiguration config)
    {
        var authMethod = new KubernetesAuthMethodInfo(
            roleName: "myapp",
            jwt: File.ReadAllText("/var/run/secrets/kubernetes.io/serviceaccount/token"));

        var settings = new VaultClientSettings(
            config["Vault:Address"]!,
            authMethod);

        _vaultClient = new VaultClient(settings);
    }

    public async Task<Dictionary<string, string>> GetSecretsAsync(string path)
    {
        var secret = await _vaultClient.V1.Secrets.KeyValue.V2
            .ReadSecretAsync(path, mountPoint: "secret");

        return secret.Data.Data
            .ToDictionary(kv => kv.Key, kv => kv.Value?.ToString() ?? "");
    }
}
```

---

## Step 1257: Cloud Cost Optimization for .NET

### Resource Usage Monitoring

```csharp
public class ResourceMonitor : BackgroundService
{
    private readonly ILogger<ResourceMonitor> _logger;
    private readonly AppMetrics _metrics;
    private static readonly Process CurrentProcess = Process.GetCurrentProcess();

    public ResourceMonitor(ILogger<ResourceMonitor> logger, AppMetrics metrics)
    {
        _logger = logger;
        _metrics = metrics;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            CurrentProcess.Refresh();

            var workingSet = CurrentProcess.WorkingSet64 / (1024 * 1024);
            var threadCount = CurrentProcess.Threads.Count;
            var gcMemory = GC.GetTotalMemory(false) / (1024 * 1024);
            var gcInfo = GC.GetGCMemoryInfo();

            _logger.LogInformation(
                "Resources: WorkingSet={WorkingSet}MB, GC={GCMemory}MB, " +
                "Threads={Threads}, Gen0={Gen0}, Gen1={Gen1}, Gen2={Gen2}",
                workingSet,
                gcMemory,
                threadCount,
                GC.CollectionCount(0),
                GC.CollectionCount(1),
                GC.CollectionCount(2));

            await Task.Delay(TimeSpan.FromMinutes(1), stoppingToken);
        }
    }
}
```

---

## Step 1258: Zero-Downtime Deployments

### Rolling Update Strategy

```csharp
// Graceful shutdown with drain support
public class GracefulShutdownMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IHostApplicationLifetime _lifetime;
    private int _activeRequests;

    public GracefulShutdownMiddleware(
        RequestDelegate next,
        IHostApplicationLifetime lifetime)
    {
        _next = next;
        _lifetime = lifetime;

        lifetime.ApplicationStopping.Register(() =>
        {
            // Wait for active requests to complete
            var timeout = DateTime.UtcNow.AddSeconds(30);
            while (Volatile.Read(ref _activeRequests) > 0 && DateTime.UtcNow < timeout)
            {
                Thread.Sleep(100);
            }
        });
    }

    public async Task InvokeAsync(HttpContext context)
    {
        if (_lifetime.ApplicationStopping.IsCancellationRequested)
        {
            context.Response.StatusCode = StatusCodes.Status503ServiceUnavailable;
            context.Response.Headers.RetryAfter = "30";
            return;
        }

        Interlocked.Increment(ref _activeRequests);
        try
        {
            await _next(context);
        }
        finally
        {
            Interlocked.Decrement(ref _activeRequests);
        }
    }
}
```

### Database Migration in Kubernetes

```csharp
// Migration job - runs before deployment
public class MigrationHostedService : IHostedService
{
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<MigrationHostedService> _logger;
    private readonly IHostApplicationLifetime _lifetime;

    public MigrationHostedService(
        IServiceProvider serviceProvider,
        ILogger<MigrationHostedService> logger,
        IHostApplicationLifetime lifetime)
    {
        _serviceProvider = serviceProvider;
        _logger = logger;
        _lifetime = lifetime;
    }

    public async Task StartAsync(CancellationToken cancellationToken)
    {
        try
        {
            using var scope = _serviceProvider.CreateScope();
            var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();

            _logger.LogInformation("Running database migrations...");
            await db.Database.MigrateAsync(cancellationToken);
            _logger.LogInformation("Migrations completed successfully");
        }
        catch (Exception ex)
        {
            _logger.LogCritical(ex, "Migration failed");
            _lifetime.StopApplication();
            throw;
        }
    }

    public Task StopAsync(CancellationToken cancellationToken) => Task.CompletedTask;
}
```

---

## Step 1259: Cloud-Native Testing

### Testcontainers for Integration Tests

```csharp
// Install: dotnet add package Testcontainers
// Install: dotnet add package Testcontainers.PostgreSql
// Install: dotnet add package Testcontainers.Redis

using Testcontainers.PostgreSql;
using Testcontainers.Redis;

public class IntegrationTestBase : IAsyncLifetime
{
    private readonly PostgreSqlContainer _postgres = new PostgreSqlBuilder()
        .WithImage("postgres:16-alpine")
        .WithDatabase("testdb")
        .WithUsername("test")
        .WithPassword("test")
        .Build();

    private readonly RedisContainer _redis = new RedisBuilder()
        .WithImage("redis:7-alpine")
        .Build();

    protected WebApplicationFactory<Program>? Factory { get; private set; }

    public async Task InitializeAsync()
    {
        // Start containers in parallel
        await Task.WhenAll(
            _postgres.StartAsync(),
            _redis.StartAsync());

        Factory = new WebApplicationFactory<Program>()
            .WithWebHostBuilder(builder =>
            {
                builder.ConfigureAppConfiguration((_, config) =>
                {
                    config.AddInMemoryCollection([
                        new("ConnectionStrings:Default",
                            _postgres.GetConnectionString()),
                        new("ConnectionStrings:Redis",
                            _redis.GetConnectionString()),
                    ]);
                });

                builder.ConfigureServices(services =>
                {
                    // Replace real services with test versions
                    var descriptor = services.SingleOrDefault(
                        d => d.ServiceType == typeof(IEmailService));
                    if (descriptor is not null)
                        services.Remove(descriptor);

                    services.AddSingleton<IEmailService, FakeEmailService>();
                });
            });

        // Run migrations
        using var scope = Factory.Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        await db.Database.MigrateAsync();
    }

    public async Task DisposeAsync()
    {
        Factory?.Dispose();
        await Task.WhenAll(
            _postgres.DisposeAsync().AsTask(),
            _redis.DisposeAsync().AsTask());
    }
}

public class OrderApiIntegrationTests : IntegrationTestBase
{
    [Fact]
    public async Task CreateOrder_ReturnsCreated()
    {
        // Arrange
        var client = Factory!.CreateClient();
        var request = new CreateOrderRequest("product-1", 2);

        // Act
        var response = await client.PostAsJsonAsync("/api/orders", request);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Created);
        var order = await response.Content.ReadFromJsonAsync<OrderResponse>();
        order.Should().NotBeNull();
        order!.Status.Should().Be("Draft");
    }
}
```

---

## Step 1260: Summary - Cloud-Native .NET

### ภาพรวมที่เรียนในบทนี้

```
Cloud-Native .NET (8/9/10)
│
├── Containerization
│   ├── Optimized Dockerfile (multi-stage)
│   └── Docker Compose for local dev
│
├── Kubernetes
│   ├── Deployment + Service + Ingress
│   ├── HPA (auto-scaling)
│   ├── Health probes (readiness/liveness/startup)
│   └── Security contexts
│
├── Azure
│   ├── Container Apps (serverless containers)
│   ├── Key Vault (secrets management)
│   ├── Service Bus (messaging)
│   └── App Configuration (feature flags)
│
├── AWS
│   ├── Lambda (Native AOT)
│   ├── DynamoDB (NoSQL)
│   └── SAM (serverless framework)
│
├── GCP
│   ├── Cloud Run (serverless containers)
│   └── Cloud Spanner (global SQL)
│
├── Service Mesh
│   ├── Istio (traffic management)
│   ├── Circuit breakers
│   └── Canary deployments
│
├── GitOps
│   ├── ArgoCD (continuous delivery)
│   └── GitHub Actions CI/CD
│
├── Observability
│   ├── OpenTelemetry (tracing + metrics)
│   ├── Custom metrics (Meter API)
│   └── Structured logging
│
└── Testing
    ├── Testcontainers
    └── Integration tests with real dependencies
```

### Cloud Provider Quick Reference

| Feature | Azure | AWS | GCP |
|---------|-------|-----|-----|
| Serverless Containers | Container Apps | ECS Fargate | Cloud Run |
| Functions | Azure Functions | Lambda | Cloud Functions |
| SQL Database | Azure SQL | RDS | Cloud SQL |
| NoSQL | Cosmos DB | DynamoDB | Firestore |
| Object Storage | Blob Storage | S3 | GCS |
| Message Queue | Service Bus | SQS/SNS | Pub/Sub |
| Container Registry | ACR | ECR | Artifact Registry |
| Secrets | Key Vault | Secrets Manager | Secret Manager |
| CI/CD | Azure DevOps | CodePipeline | Cloud Build |
| Kubernetes | AKS | EKS | GKE |

ในบทถัดไปจะเรียนเรื่อง **Advanced DevOps & Platform Engineering (Steps 1261-1300)** ซึ่งครอบคลุม Infrastructure as Code, Platform Engineering, SRE practices และ chaos engineering!
