# Part 66: Kubernetes & Container Orchestration — Production-Grade .NET Deployment (Steps 1685-1700)

## Steps 1685-1700: Helm, KEDA, Dapr, Probes, and GitOps for .NET Applications

Kubernetes is the production standard for running containerized .NET workloads. This part covers everything from writing production-quality Dockerfiles through Helm charts, event-driven autoscaling with KEDA, sidecar patterns with Dapr, and GitOps with Argo CD.

---

## Step 1685: Production Dockerfile for .NET

```dockerfile
# Build stage — restore and publish in one optimized layer
FROM mcr.microsoft.com/dotnet/sdk:9.0-alpine AS build
WORKDIR /src

# Copy only project files first — Docker layer caching: restore is expensive
COPY ["src/OrderApi/OrderApi.csproj", "src/OrderApi/"]
COPY ["src/OrderApi.Contracts/OrderApi.Contracts.csproj", "src/OrderApi.Contracts/"]
RUN dotnet restore "src/OrderApi/OrderApi.csproj" \
    --runtime linux-musl-x64 \
    --packages /nuget-packages

# Copy full source and build
COPY . .
RUN dotnet publish "src/OrderApi/OrderApi.csproj" \
    --configuration Release \
    --runtime linux-musl-x64 \
    --self-contained true \
    --no-restore \
    -p:PublishSingleFile=true \
    -p:PublishReadyToRun=true \
    -p:StripSymbols=true \
    --output /app/publish

# Runtime stage — minimal distroless image
FROM mcr.microsoft.com/dotnet/runtime-deps:9.0-alpine AS runtime
WORKDIR /app

# Security: run as non-root user
RUN addgroup --system --gid 1001 appgroup && \
    adduser --system --uid 1001 --ingroup appgroup --no-create-home appuser

# Copy published app
COPY --from=build --chown=appuser:appgroup /app/publish .

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=15s --retries=3 \
  CMD wget -qO- http://localhost:8080/health || exit 1

USER appuser
EXPOSE 8080

ENTRYPOINT ["./OrderApi"]
```

```dockerfile
# Alternative: chiseled image (even smaller, fewer packages)
FROM mcr.microsoft.com/dotnet/aspnet:9.0-jammy-chiseled AS runtime
WORKDIR /app
COPY --from=build /app/publish .
USER app
EXPOSE 8080
ENTRYPOINT ["dotnet", "OrderApi.dll"]
```

```bash
# Build and push
docker build -t myregistry.azurecr.io/order-api:v1.2.3 .
docker push myregistry.azurecr.io/order-api:v1.2.3

# Scan image for vulnerabilities before pushing
docker scout cves myregistry.azurecr.io/order-api:v1.2.3
trivy image myregistry.azurecr.io/order-api:v1.2.3
```

---

## Step 1686: Kubernetes Deployment Manifests

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-api
  namespace: myapp
  labels:
    app: order-api
    version: v1.2.3
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0      # zero downtime
      maxSurge: 1            # one extra pod during rollout
  selector:
    matchLabels:
      app: order-api
  template:
    metadata:
      labels:
        app: order-api
        version: v1.2.3
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: order-api-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        runAsGroup: 1001
        fsGroup: 1001
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: order-api
          image: myregistry.azurecr.io/order-api:v1.2.3
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
              name: http
          env:
            - name: ASPNETCORE_ENVIRONMENT
              value: Production
            - name: ASPNETCORE_URLS
              value: http://+:8080
            - name: ConnectionStrings__OrdersDb
              valueFrom:
                secretKeyRef:
                  name: db-secrets
                  key: connection-string
          resources:
            requests:
              memory: "256Mi"
              cpu: "100m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          startupProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 30   # 30 × 5s = 150s max startup time
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir: {}
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: order-api
                topologyKey: kubernetes.io/hostname
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: order-api
```

```yaml
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: order-api
  namespace: myapp
spec:
  selector:
    app: order-api
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
      name: http
  type: ClusterIP

---
# k8s/hpa.yaml — Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-api-hpa
  namespace: myapp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-api
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
          type: AverageValue
          averageValue: 400Mi
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300   # wait 5 min before scaling down
      policies:
        - type: Percent
          value: 25
          periodSeconds: 60             # remove max 25% of pods per minute
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
        - type: Pods
          value: 4
          periodSeconds: 60             # add max 4 pods per minute
```

---

## Step 1687: Health Checks in ASP.NET Core

```csharp
using Microsoft.Extensions.Diagnostics.HealthChecks;
using Microsoft.AspNetCore.Diagnostics.HealthChecks;

// Register health checks
builder.Services.AddHealthChecks()
    .AddNpgsql(                                         // database
        connectionString: builder.Configuration.GetConnectionString("OrdersDb")!,
        name: "postgres",
        tags: ["db", "ready"])
    .AddRedis(                                          // cache
        builder.Configuration.GetConnectionString("Redis")!,
        name: "redis",
        tags: ["cache", "ready"])
    .AddRabbitMQ(                                       // messaging
        builder.Configuration.GetConnectionString("RabbitMQ")!,
        name: "rabbitmq",
        tags: ["messaging", "ready"])
    .AddCheck("disk", () =>                             // custom check
    {
        var drive = DriveInfo.GetDrives().First(d => d.Name == "/");
        var usedPercent = 100.0 - (drive.AvailableFreeSpace * 100.0 / drive.TotalSize);
        return usedPercent < 90
            ? HealthCheckResult.Healthy($"Disk: {usedPercent:F1}% used")
            : HealthCheckResult.Degraded($"Disk nearly full: {usedPercent:F1}% used");
    }, tags: ["infra"])
    .AddCheck<ExternalApiHealthCheck>("external-api", tags: ["external", "ready"]);

// Map endpoints — Kubernetes uses separate liveness and readiness probes
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready"),  // readiness: db, cache, etc.
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false  // liveness: always return 200 if process is alive
});

app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
}).RequireAuthorization("HealthCheckPolicy"); // protect detailed health from public
```

```csharp
// Custom health check
public class ExternalApiHealthCheck(IHttpClientFactory factory) : IHealthCheck
{
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken ct = default)
    {
        try
        {
            var client = factory.CreateClient("external-api");
            var response = await client.GetAsync("/health", ct);
            return response.IsSuccessStatusCode
                ? HealthCheckResult.Healthy()
                : HealthCheckResult.Degraded($"External API returned {response.StatusCode}");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("External API unreachable", ex);
        }
    }
}
```

---

## Step 1688: ConfigMaps and Secrets

```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-api-config
  namespace: myapp
data:
  appsettings.Production.json: |
    {
      "Logging": {
        "LogLevel": {
          "Default": "Warning",
          "Microsoft.AspNetCore": "Warning"
        }
      },
      "FeatureFlags": {
        "NewCheckout": false
      }
    }
  RabbitMQ__VirtualHost: "/"
  RabbitMQ__Port: "5672"
```

```yaml
# k8s/secret.yaml — stored as base64, but use External Secrets Operator or CSI driver in production
apiVersion: v1
kind: Secret
metadata:
  name: db-secrets
  namespace: myapp
type: Opaque
stringData:
  connection-string: "Host=postgres;Database=orders;Username=api;Password=changeme"
  jwt-signing-key: "super-secret-key-32-chars-minimum!"
```

```yaml
# k8s/external-secret.yaml — External Secrets Operator pulling from Azure Key Vault
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: order-api-secrets
  namespace: myapp
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: azure-keyvault
    kind: ClusterSecretStore
  target:
    name: db-secrets
    creationPolicy: Owner
  data:
    - secretKey: connection-string
      remoteRef:
        key: orders-db-connection-string
    - secretKey: jwt-signing-key
      remoteRef:
        key: order-api-jwt-key
```

---

## Step 1689: Helm Chart for .NET Applications

```bash
helm create order-api
```

```
order-api/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── serviceaccount.yaml
│   ├── configmap.yaml
│   └── _helpers.tpl
```

```yaml
# Chart.yaml
apiVersion: v2
name: order-api
description: Order API Helm chart
type: application
version: 0.1.0
appVersion: "1.2.3"
dependencies:
  - name: postgresql
    version: "15.5.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
```

```yaml
# values.yaml
replicaCount: 3

image:
  repository: myregistry.azurecr.io/order-api
  pullPolicy: Always
  tag: ""  # overridden at deploy time via --set image.tag=v1.2.3

serviceAccount:
  create: true
  name: ""

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/rate-limit: "100"
  hosts:
    - host: api.myapp.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: order-api-tls
      hosts:
        - api.myapp.com

resources:
  requests:
    memory: 256Mi
    cpu: 100m
  limits:
    memory: 512Mi
    cpu: 500m

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70

env:
  ASPNETCORE_ENVIRONMENT: Production
  ASPNETCORE_URLS: http://+:8080

postgresql:
  enabled: false  # use external database in production

podAnnotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "8080"
```

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "order-api.fullname" . }}
  labels:
    {{- include "order-api.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "order-api.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      annotations:
        {{- with .Values.podAnnotations }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
      labels:
        {{- include "order-api.selectorLabels" . | nindent 8 }}
    spec:
      serviceAccountName: {{ include "order-api.serviceAccountName" . }}
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: 8080
              protocol: TCP
          env:
            {{- range $key, $val := .Values.env }}
            - name: {{ $key }}
              value: {{ $val | quote }}
            {{- end }}
            - name: ConnectionStrings__OrdersDb
              valueFrom:
                secretKeyRef:
                  name: {{ include "order-api.fullname" . }}-secrets
                  key: connection-string
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          readinessProbe:
            httpGet: { path: /health/ready, port: 8080 }
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:
            httpGet: { path: /health/live, port: 8080 }
            initialDelaySeconds: 30
            periodSeconds: 10
```

```bash
# Deploy
helm upgrade --install order-api ./order-api \
  --namespace myapp \
  --create-namespace \
  --set image.tag=v1.2.3 \
  --values values.prod.yaml \
  --wait \
  --atomic  # roll back automatically if deployment fails
```

---

## Step 1690: KEDA — Event-Driven Autoscaling

KEDA extends HPA to scale based on external event sources: Azure Service Bus queue depth, RabbitMQ message count, Kafka consumer lag, Redis list length, etc.

```bash
# Install KEDA
helm repo add kedacore https://kedacore.github.io/charts
helm install keda kedacore/keda --namespace keda --create-namespace
```

```yaml
# ScaledObject: scale order-processor based on RabbitMQ queue depth
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-processor-scaler
  namespace: myapp
spec:
  scaleTargetRef:
    name: order-processor
  minReplicaCount: 0    # scale to zero when queue is empty!
  maxReplicaCount: 50
  pollingInterval: 15   # check every 15 seconds
  cooldownPeriod: 60    # wait 60s before scaling down
  triggers:
    - type: rabbitmq
      metadata:
        protocol: amqp
        queueName: order-events
        mode: QueueLength
        value: "100"          # 1 pod per 100 messages
        host: amqp://rabbitmq.myapp.svc.cluster.local
      authenticationRef:
        name: rabbitmq-auth
    - type: cpu
      metadata:
        type: Utilization
        value: "80"
```

```yaml
# ScaledObject: scale based on Azure Service Bus
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: notification-worker-scaler
  namespace: myapp
spec:
  scaleTargetRef:
    name: notification-worker
  minReplicaCount: 1
  maxReplicaCount: 30
  triggers:
    - type: azure-servicebus
      metadata:
        queueName: notifications
        messageCount: "50"      # 1 pod per 50 messages
      authenticationRef:
        name: azure-servicebus-auth

---
# TriggerAuthentication for Service Bus connection string
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: azure-servicebus-auth
  namespace: myapp
spec:
  secretTargetRef:
    - parameter: connection
      name: servicebus-secrets
      key: connection-string
```

```yaml
# ScaledJob: scale Kubernetes Jobs for batch processing
apiVersion: keda.sh/v1alpha1
kind: ScaledJob
metadata:
  name: report-generator
  namespace: myapp
spec:
  jobTargetRef:
    template:
      spec:
        containers:
          - name: report-generator
            image: myregistry.azurecr.io/report-generator:v1.0.0
            env:
              - name: REPORT_ID
                valueFrom:
                  fieldRef:
                    fieldPath: metadata.annotations['keda.sh/job-id']
  maxReplicaCount: 20
  triggers:
    - type: azure-storage-queue
      metadata:
        queueName: report-requests
        queueLength: "1"          # 1 job per 1 message
        accountName: mystorageacct
      authenticationRef:
        name: azure-storage-auth
```

---

## Step 1691: Dapr — Distributed Application Runtime

Dapr provides building blocks (pub/sub, state, bindings, service invocation) as sidecar containers, abstracting the underlying infrastructure.

```bash
# Install Dapr in Kubernetes
dapr init --kubernetes --wait
```

```yaml
# k8s/deployment.yaml with Dapr annotations
metadata:
  annotations:
    dapr.io/enabled: "true"
    dapr.io/app-id: "order-api"
    dapr.io/app-port: "8080"
    dapr.io/log-level: "warn"
    dapr.io/enable-api-logging: "false"
    dapr.io/config: "tracing-config"
```

```csharp
// Install: Dapr.AspNetCore, Dapr.Client
using Dapr;
using Dapr.Client;

var builder = WebApplication.CreateBuilder(args);

// Register Dapr client
builder.Services.AddDaprClient();
builder.Services.AddControllers().AddDapr(); // adds CloudEvents support

var app = builder.Build();
app.UseCloudEvents(); // deserializes CloudEvents envelope
app.MapSubscribeHandler(); // registers /dapr/subscribe endpoint
app.MapControllers();
```

```csharp
// Pub/Sub — publish
public class OrderService(DaprClient dapr)
{
    public async Task PublishOrderPlacedAsync(OrderPlacedEvent evt, CancellationToken ct)
    {
        await dapr.PublishEventAsync(
            pubsubName: "pubsub",   // component name from Dapr config
            topicName: "order-placed",
            data: evt,
            cancellationToken: ct);
    }
}

// Pub/Sub — subscribe
[ApiController]
[Route("[controller]")]
public class OrderEventsController : ControllerBase
{
    [Topic("pubsub", "order-placed")]  // subscribes to the topic
    [HttpPost("order-placed")]
    public async Task<IActionResult> HandleOrderPlaced(
        OrderPlacedEvent evt,
        [FromServices] INotificationService notifications,
        CancellationToken ct)
    {
        await notifications.SendConfirmationAsync(evt.CustomerId, evt.OrderId, ct);
        return Ok(); // 200 = ACK; non-2xx = NACK (retry)
    }
}
```

```yaml
# Dapr component: RabbitMQ pub/sub
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: pubsub
  namespace: myapp
spec:
  type: pubsub.rabbitmq
  version: v1
  metadata:
    - name: host
      value: amqp://rabbitmq.myapp.svc.cluster.local
    - name: durable
      value: "true"
    - name: deletedWhenUnused
      value: "false"
    - name: autoAck
      value: "false"
    - name: reconnectWait
      value: "2"
    - name: concurrency
      value: "parallel"
```

```csharp
// Dapr State Store
public class OrderStateService(DaprClient dapr)
{
    private const string StoreName = "statestore";

    public async Task SaveOrderAsync(Order order, CancellationToken ct)
    {
        await dapr.SaveStateAsync(StoreName, order.Id.ToString(), order,
            cancellationToken: ct);
    }

    public async Task<Order?> GetOrderAsync(Guid id, CancellationToken ct)
        => await dapr.GetStateAsync<Order>(StoreName, id.ToString(),
            cancellationToken: ct);

    // Optimistic concurrency with ETags
    public async Task UpdateOrderAsync(Order order, string etag, CancellationToken ct)
    {
        var options = new StateOptions { Concurrency = ConcurrencyMode.FirstWrite };
        await dapr.TrySaveStateAsync(StoreName, order.Id.ToString(),
            order, etag, options, cancellationToken: ct);
    }
}
```

---

## Step 1692: Ingress with TLS and Rate Limiting

```yaml
# k8s/ingress.yaml — NGINX Ingress with cert-manager
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: order-api-ingress
  namespace: myapp
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/use-regex: "true"
    # Rate limiting per IP
    nginx.ingress.kubernetes.io/limit-connections: "20"
    nginx.ingress.kubernetes.io/limit-rpm: "100"
    nginx.ingress.kubernetes.io/limit-burst-multiplier: "5"
    # Security headers
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Content-Type-Options: nosniff";
      more_set_headers "X-Frame-Options: DENY";
      more_set_headers "Referrer-Policy: strict-origin-when-cross-origin";
spec:
  tls:
    - hosts:
        - api.myapp.com
      secretName: order-api-tls
  rules:
    - host: api.myapp.com
      http:
        paths:
          - path: /api/v1/orders
            pathType: Prefix
            backend:
              service:
                name: order-api
                port:
                  number: 80
          - path: /api/v1/products
            pathType: Prefix
            backend:
              service:
                name: product-api
                port:
                  number: 80
```

```yaml
# ClusterIssuer for Let's Encrypt
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: ops@myapp.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            class: nginx
```

---

## Step 1693: Network Policies — Zero-Trust Networking

```yaml
# Allow order-api to talk to postgres, redis, and rabbitmq — deny all else
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: order-api-network-policy
  namespace: myapp
spec:
  podSelector:
    matchLabels:
      app: order-api
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
        - podSelector:
            matchLabels:
              app: order-api-gateway
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432
    - to:
        - podSelector:
            matchLabels:
              app: redis
      ports:
        - protocol: TCP
          port: 6379
    - to:
        - podSelector:
            matchLabels:
              app: rabbitmq
      ports:
        - protocol: TCP
          port: 5672
    # Allow DNS
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
```

---

## Step 1694: GitOps with Argo CD

```bash
# Install Argo CD
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

```yaml
# argocd/application.yaml — defines what to deploy and from where
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: order-api
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/myapp-infra
    targetRevision: main
    path: k8s/order-api
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp
  syncPolicy:
    automated:
      prune: true          # delete resources removed from Git
      selfHeal: true       # revert manual kubectl changes
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  revisionHistoryLimit: 5
```

```yaml
# GitOps workflow: image update automation with Argo CD Image Updater
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: order-api
  annotations:
    argocd-image-updater.argoproj.io/image-list: myapp=myregistry.azurecr.io/order-api
    argocd-image-updater.argoproj.io/myapp.update-strategy: semver
    argocd-image-updater.argoproj.io/myapp.semver-constraint: "^1.0.0"
    argocd-image-updater.argoproj.io/write-back-method: git
    argocd-image-updater.argoproj.io/git-branch: main
```

---

## Step 1695: Monitoring with Prometheus and Grafana

```csharp
// Install: OpenTelemetry.Exporter.Prometheus.AspNetCore
// prometheus-net.AspNetCore

builder.Services.AddOpenTelemetry()
    .WithMetrics(metrics =>
    {
        metrics
            .AddAspNetCoreInstrumentation()
            .AddHttpClientInstrumentation()
            .AddRuntimeInstrumentation()
            .AddMeter("MyApp.Orders") // custom meter
            .AddPrometheusExporter();
    });

// Expose metrics endpoint
app.MapPrometheusScrapingEndpoint("/metrics"); // scraped by Prometheus
```

```csharp
// Custom metrics
using System.Diagnostics.Metrics;

public class OrderMetrics
{
    private readonly Counter<long> _ordersPlaced;
    private readonly Histogram<double> _orderProcessingDuration;
    private readonly UpDownCounter<int> _activeOrders;

    public OrderMetrics(IMeterFactory meterFactory)
    {
        var meter = meterFactory.Create("MyApp.Orders");
        _ordersPlaced = meter.CreateCounter<long>(
            "orders_placed_total",
            description: "Total number of orders placed");
        _orderProcessingDuration = meter.CreateHistogram<double>(
            "order_processing_duration_seconds",
            unit: "s",
            description: "Order processing duration");
        _activeOrders = meter.CreateUpDownCounter<int>(
            "orders_active",
            description: "Currently active orders");
    }

    public void RecordOrderPlaced(string customerId, string tier)
    {
        _ordersPlaced.Add(1,
            new("customer.tier", tier),
            new("region", "us-east-1"));
    }

    public void RecordProcessingDuration(double seconds, string status)
    {
        _orderProcessingDuration.Record(seconds, new("status", status));
    }
}
```

```yaml
# prometheus/rule.yaml — alerting rules
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: order-api-alerts
  namespace: myapp
spec:
  groups:
    - name: order-api
      interval: 30s
      rules:
        - alert: HighErrorRate
          expr: |
            rate(http_requests_total{app="order-api",status=~"5.."}[5m])
            / rate(http_requests_total{app="order-api"}[5m]) > 0.05
          for: 2m
          labels:
            severity: critical
          annotations:
            summary: "High error rate on order-api"
            description: "Error rate is {{ $value | humanizePercentage }} over the last 5 minutes"

        - alert: HighOrderProcessingLatency
          expr: |
            histogram_quantile(0.99,
              rate(order_processing_duration_seconds_bucket{app="order-api"}[5m])) > 2
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "p99 order processing latency > 2s"
```

---

## Step 1696: Pod Disruption Budget and Priority Classes

```yaml
# PodDisruptionBudget — ensure minimum availability during node drains
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: order-api-pdb
  namespace: myapp
spec:
  minAvailable: 2   # always keep at least 2 pods running
  selector:
    matchLabels:
      app: order-api

---
# PriorityClass — ensure critical workloads are not evicted
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority-app
value: 100000
globalDefault: false
description: "High priority for critical application pods"
```

---

## Step 1697: Resource Quotas and Limit Ranges

```yaml
# Namespace-level resource quota
apiVersion: v1
kind: ResourceQuota
metadata:
  name: myapp-quota
  namespace: myapp
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "100"
    services.loadbalancers: "2"
    persistentvolumeclaims: "20"

---
# LimitRange — default requests/limits for pods that don't specify
apiVersion: v1
kind: LimitRange
metadata:
  name: myapp-limitrange
  namespace: myapp
spec:
  limits:
    - type: Container
      default:
        memory: 256Mi
        cpu: 200m
      defaultRequest:
        memory: 128Mi
        cpu: 100m
      max:
        memory: 2Gi
        cpu: "2"
      min:
        memory: 64Mi
        cpu: 50m
```

---

## Step 1698: Service Mesh with Istio

```bash
# Install Istio
istioctl install --set profile=production -y
kubectl label namespace myapp istio-injection=enabled
```

```yaml
# VirtualService — traffic splitting for canary deployment
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: order-api
  namespace: myapp
spec:
  hosts:
    - order-api
  http:
    - route:
        - destination:
            host: order-api
            subset: stable
          weight: 90
        - destination:
            host: order-api
            subset: canary
          weight: 10
      retries:
        attempts: 3
        perTryTimeout: 2s
        retryOn: gateway-error,connect-failure,retriable-4xx
      timeout: 5s

---
# DestinationRule — circuit breaker
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: order-api
  namespace: myapp
spec:
  host: order-api
  subsets:
    - name: stable
      labels:
        version: stable
    - name: canary
      labels:
        version: canary
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1000
      http:
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
```

---

## Step 1699: CI/CD Pipeline for Kubernetes

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy

on:
  push:
    branches: [main]

permissions:
  contents: read
  id-token: write   # for OIDC authentication to Azure

env:
  REGISTRY: myregistry.azurecr.io
  IMAGE_NAME: order-api

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.version }}
    steps:
      - uses: actions/checkout@v4

      - name: Login to ACR
        uses: azure/docker-login@v1
        with:
          login-server: ${{ env.REGISTRY }}
          username: ${{ secrets.ACR_USERNAME }}
          password: ${{ secrets.ACR_PASSWORD }}

      - name: Extract Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=semver,pattern={{version}}
            type=sha,format=long

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Scan image
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.meta.outputs.version }}
          exit-code: '1'
          severity: CRITICAL

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Update Helm values
        run: |
          sed -i "s/tag: .*/tag: ${{ needs.build.outputs.image-tag }}/" k8s/order-api/values.yaml
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add k8s/order-api/values.yaml
          git commit -m "chore: update order-api image to ${{ needs.build.outputs.image-tag }}"
          git push
      # Argo CD auto-syncs from the git change above
```

---

## Step 1700: .NET App Graceful Shutdown in Kubernetes

```csharp
// Kubernetes sends SIGTERM before SIGKILL — handle it properly
var app = builder.Build();

// Register shutdown handling
app.Lifetime.ApplicationStopping.Register(() =>
{
    var logger = app.Services.GetRequiredService<ILogger<Program>>();
    logger.LogInformation("SIGTERM received — draining requests...");
});

// Configure graceful shutdown timeout
builder.Services.Configure<HostOptions>(options =>
{
    // Wait up to 30s for in-flight requests to complete
    options.ShutdownTimeout = TimeSpan.FromSeconds(30);
});

// Ensure readiness probe fails immediately on shutdown
// so load balancer stops sending new traffic
builder.Services.AddHealthChecks()
    .AddCheck("shutdown", () =>
    {
        // Check cancellation token of the host
        if (!app.Lifetime.ApplicationStopping.IsCancellationRequested)
            return HealthCheckResult.Healthy();
        return HealthCheckResult.Unhealthy("Application is shutting down");
    }, tags: ["ready"]);

app.Run();
```

```yaml
# Kubernetes termination grace period must exceed app shutdown timeout
spec:
  terminationGracePeriodSeconds: 60    # k8s waits 60s before SIGKILL
  containers:
    - name: order-api
      lifecycle:
        preStop:
          exec:
            command: ["/bin/sh", "-c", "sleep 5"]  # let load balancer remove pod first
```

---

## Kubernetes Deployment Checklist

| Area | Requirement |
|---|---|
| Image | Distroless or Alpine, non-root user, read-only root filesystem |
| Resources | Always set `requests` and `limits` |
| Probes | Separate liveness, readiness, and startup probes |
| Security | `seccompProfile`, drop all Linux capabilities |
| Scaling | HPA (CPU/memory) + KEDA (event-driven) |
| Availability | PodDisruptionBudget, `topologySpreadConstraints`, anti-affinity |
| Networking | NetworkPolicy (zero-trust), TLS at ingress |
| Config | ConfigMap for config, External Secrets for secrets |
| Packaging | Helm chart with parameterized values |
| GitOps | Argo CD sync + image updater |
| Monitoring | Prometheus metrics + alerting rules |
| CI/CD | Build → scan → push → update values → auto-sync |
| Shutdown | Graceful SIGTERM handling, `preStop` sleep hook |

### What Next?

- **Part 67**: Messaging Patterns — MassTransit, Wolverine, Outbox pattern, Saga choreography vs orchestration
- **Part 68**: Database Performance — EF Core compiled queries, raw SQL, Dapper, index strategy, connection pooling
- **Part 69**: Observability Deep Dive — OpenTelemetry traces, metrics, logs; Jaeger, Grafana Tempo
