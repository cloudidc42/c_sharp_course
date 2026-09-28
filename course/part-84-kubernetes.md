# Part 84: Kubernetes & Cloud Deployment

## Steps 1973–1988

---

## Step 1973: Container-Ready .NET Application

Before deploying to Kubernetes, applications must be container-ready.

```dockerfile
# Dockerfile — multi-stage build
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /src

# Copy solution and restore (layer caching)
COPY ["Directory.Build.props", "."]
COPY ["Directory.Packages.props", "."]
COPY ["src/Orders.Api/Orders.Api.csproj", "src/Orders.Api/"]
COPY ["src/Orders.Application/Orders.Application.csproj", "src/Orders.Application/"]
COPY ["src/Orders.Domain/Orders.Domain.csproj", "src/Orders.Domain/"]
COPY ["src/Orders.Infrastructure/Orders.Infrastructure.csproj", "src/Orders.Infrastructure/"]

RUN dotnet restore "src/Orders.Api/Orders.Api.csproj"

# Copy source and build
COPY src/ src/
WORKDIR "/src/src/Orders.Api"
RUN dotnet build "Orders.Api.csproj" -c Release -o /app/build

# Publish
FROM build AS publish
RUN dotnet publish "Orders.Api.csproj" -c Release -o /app/publish \
    --no-restore \
    /p:UseAppHost=false \
    /p:PublishTrimmed=false

# Runtime — minimal image
FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS final
WORKDIR /app

# Security: non-root user
RUN addgroup --system --gid 1001 dotnet && \
    adduser --system --uid 1001 --ingroup dotnet dotnet

# Health check binary
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*

COPY --from=publish --chown=dotnet:dotnet /app/publish .

USER dotnet

# Port exposure
EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080
ENV ASPNETCORE_ENVIRONMENT=Production

# Health check
HEALTHCHECK --interval=10s --timeout=5s --start-period=30s --retries=3 \
    CMD curl -f http://localhost:8080/health/live || exit 1

ENTRYPOINT ["dotnet", "Orders.Api.dll"]
```

```yaml
# .dockerignore
**/bin/
**/obj/
**/out/
**/.vs/
**/.git/
*.md
Dockerfile*
docker-compose*
.env*
```

---

## Step 1974: Kubernetes Deployment Manifest

```yaml
# k8s/orders-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-api
  namespace: production
  labels:
    app: orders-api
    version: "1.0"
    component: api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: orders-api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0  # zero-downtime deployment
  template:
    metadata:
      labels:
        app: orders-api
        version: "1.0"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: orders-api-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        fsGroup: 1001
      containers:
        - name: orders-api
          image: myregistry.azurecr.io/orders-api:1.2.3
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
              name: http
          env:
            - name: ASPNETCORE_ENVIRONMENT
              value: "Production"
            - name: ConnectionStrings__Postgres
              valueFrom:
                secretKeyRef:
                  name: orders-secrets
                  key: postgres-connection-string
            - name: ConnectionStrings__Redis
              valueFrom:
                secretKeyRef:
                  name: orders-secrets
                  key: redis-connection-string
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: "http://otel-collector:4317"
            - name: OTEL_SERVICE_NAME
              value: "orders-api"
          resources:
            requests:
              memory: "256Mi"
              cpu: "100m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 3
          startupProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 12  # 60s window for startup
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir: {}
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: orders-api
```

---

## Step 1975: Kubernetes Service & Ingress

```yaml
# k8s/orders-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: orders-api
  namespace: production
  labels:
    app: orders-api
spec:
  selector:
    app: orders-api
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 8080
  type: ClusterIP

---
# k8s/orders-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: orders-api-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.example.com
      secretName: api-tls-secret
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /api/orders(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: orders-api
                port:
                  number: 80
```

---

## Step 1976: ConfigMap & Secrets

```yaml
# k8s/orders-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: orders-config
  namespace: production
data:
  ASPNETCORE_ENVIRONMENT: "Production"
  Logging__LogLevel__Default: "Information"
  Logging__LogLevel__Microsoft.AspNetCore: "Warning"
  FeatureFlags__EnableNewCheckout: "true"
  RateLimiting__DefaultLimit: "100"

---
# k8s/orders-secrets.yaml (values are base64-encoded)
# DO NOT commit actual secrets — use Sealed Secrets, Vault, or External Secrets Operator
apiVersion: v1
kind: Secret
metadata:
  name: orders-secrets
  namespace: production
type: Opaque
stringData:
  postgres-connection-string: "Host=postgres-svc;Database=orders;Username=app;Password=changeme"
  redis-connection-string: "redis-svc:6379"
  jwt-secret-base64: "change-this-in-production"
```

```yaml
# External Secrets Operator — pull from Azure Key Vault
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: orders-secrets
  namespace: production
spec:
  refreshInterval: 5m
  secretStoreRef:
    kind: ClusterSecretStore
    name: azure-keyvault
  target:
    name: orders-secrets
    creationPolicy: Owner
  data:
    - secretKey: postgres-connection-string
      remoteRef:
        key: orders-postgres-connection-string
    - secretKey: jwt-secret-base64
      remoteRef:
        key: orders-jwt-secret
```

---

## Step 1977: Horizontal Pod Autoscaler

```yaml
# k8s/orders-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: orders-api-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: orders-api
  minReplicas: 3
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
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
        - type: Pods
          value: 2
          periodSeconds: 30
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 1
          periodSeconds: 60
```

---

## Step 1978: Pod Disruption Budget

```yaml
# Ensure minimum available pods during rolling updates or node drains
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: orders-api-pdb
  namespace: production
spec:
  minAvailable: 2   # always keep at least 2 pods
  selector:
    matchLabels:
      app: orders-api
```

---

## Step 1979: GitHub Actions CI/CD Pipeline

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: myregistry.azurecr.io
  IMAGE_NAME: orders-api
  K8S_NAMESPACE: production

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: orders_test
        ports: ["5432:5432"]
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '9.x'

      - name: Cache NuGet packages
        uses: actions/cache@v4
        with:
          path: ~/.nuget/packages
          key: ${{ runner.os }}-nuget-${{ hashFiles('**/*.csproj') }}

      - name: Restore
        run: dotnet restore

      - name: Build
        run: dotnet build --no-restore -c Release

      - name: Test
        run: |
          dotnet test --no-build -c Release \
            --logger "trx;LogFileName=test-results.trx" \
            --collect:"XPlat Code Coverage" \
            -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.Format=cobertura
        env:
          ConnectionStrings__Postgres: "Host=localhost;Database=orders_test;Username=postgres;Password=testpass"

      - name: Publish test results
        uses: dorny/test-reporter@v1
        if: always()
        with:
          name: .NET Tests
          path: '**/*.trx'
          reporter: dotnet-trx

      - name: NuGet audit
        run: dotnet list package --vulnerable --include-transitive

  build-and-push:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.version }}
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4

      - name: Log in to container registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ secrets.ACR_USERNAME }}
          password: ${{ secrets.ACR_PASSWORD }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=semver,pattern={{version}}
            type=sha,prefix=sha-,format=short
            type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          file: src/Orders.Api/Dockerfile
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=registry,ref=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:cache
          cache-to: type=registry,ref=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:cache,mode=max

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Set up kubectl
        uses: azure/setup-kubectl@v3

      - name: Azure login
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Set AKS context
        uses: azure/aks-set-context@v3
        with:
          resource-group: my-rg
          cluster-name: my-aks-cluster

      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/orders-api \
            orders-api=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ needs.build-and-push.outputs.image-tag }} \
            -n ${{ env.K8S_NAMESPACE }}

          kubectl rollout status deployment/orders-api \
            -n ${{ env.K8S_NAMESPACE }} \
            --timeout=300s

      - name: Run smoke tests
        run: |
          sleep 10
          curl -f https://api.example.com/health/ready || exit 1
```

---

## Step 1980: Helm Chart

```yaml
# charts/orders-api/Chart.yaml
apiVersion: v2
name: orders-api
description: Orders microservice Helm chart
type: application
version: 0.1.0
appVersion: "1.0.0"
```

```yaml
# charts/orders-api/values.yaml
replicaCount: 3

image:
  repository: myregistry.azurecr.io/orders-api
  pullPolicy: IfNotPresent
  tag: ""  # Overridden by CI

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: true
  className: nginx
  hosts:
    - host: api.example.com
      paths:
        - path: /api/orders
          pathType: Prefix
  tls:
    - secretName: api-tls
      hosts:
        - api.example.com

resources:
  requests:
    memory: "256Mi"
    cpu: "100m"
  limits:
    memory: "512Mi"
    cpu: "500m"

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70

env:
  ASPNETCORE_ENVIRONMENT: Production

secrets:
  postgresConnectionString: ""  # Provided at deploy time
  jwtSecretBase64: ""
```

```yaml
# charts/orders-api/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "orders-api.fullname" . }}
  labels:
    {{- include "orders-api.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "orders-api.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "orders-api.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.targetPort }}
          env:
            {{- range $key, $value := .Values.env }}
            - name: {{ $key }}
              value: {{ $value | quote }}
            {{- end }}
            - name: ConnectionStrings__Postgres
              valueFrom:
                secretKeyRef:
                  name: {{ include "orders-api.fullname" . }}-secrets
                  key: postgres-connection-string
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          livenessProbe:
            httpGet:
              path: /health/live
              port: {{ .Values.service.targetPort }}
            initialDelaySeconds: 10
          readinessProbe:
            httpGet:
              path: /health/ready
              port: {{ .Values.service.targetPort }}
            initialDelaySeconds: 5
```

---

## Step 1981: Graceful Shutdown in .NET

```csharp
// Program.cs — graceful shutdown
var app = builder.Build();

// Handle SIGTERM (Kubernetes sends this before killing pod)
var lifetime = app.Services.GetRequiredService<IHostApplicationLifetime>();

lifetime.ApplicationStopping.Register(() =>
{
    app.Services.GetRequiredService<ILogger<Program>>()
        .LogInformation("Application is shutting down. Draining connections...");
});

// Configure shutdown timeout
builder.Host.ConfigureHostOptions(opts =>
{
    opts.ShutdownTimeout = TimeSpan.FromSeconds(30);
});

// In-flight requests are completed before shutdown
// Kubernetes terminationGracePeriodSeconds should be > ShutdownTimeout + preStop sleep
```

```yaml
# k8s — ensure graceful shutdown
containers:
  - name: orders-api
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "sleep 5"]  # give load balancer time to deregister
    terminationMessagePolicy: FallbackToLogsOnError
spec:
  terminationGracePeriodSeconds: 60  # > ShutdownTimeout(30) + preStop(5)
```

---

## Step 1982: Network Policies

```yaml
# Restrict ingress/egress — defense in depth
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: orders-api-network-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: orders-api
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # Allow traffic from ingress controller
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - protocol: TCP
          port: 8080
    # Allow traffic from other services in the same namespace
    - from:
        - podSelector: {}
      ports:
        - protocol: TCP
          port: 8080
  egress:
    # Allow DNS
    - ports:
        - protocol: UDP
          port: 53
    # Allow PostgreSQL
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432
    # Allow Redis
    - to:
        - podSelector:
            matchLabels:
              app: redis
      ports:
        - protocol: TCP
          port: 6379
    # Allow RabbitMQ
    - to:
        - podSelector:
            matchLabels:
              app: rabbitmq
      ports:
        - protocol: TCP
          port: 5672
    # Allow OpenTelemetry collector
    - to:
        - podSelector:
            matchLabels:
              app: otel-collector
      ports:
        - protocol: TCP
          port: 4317
```

---

## Step 1983: Prometheus Metrics

```bash
dotnet add package prometheus-net.AspNetCore
dotnet add package prometheus-net.AspNetCore.HealthChecks
```

```csharp
// Program.cs
builder.Services.AddMetrics();

app.UseHttpMetrics(opts =>
{
    opts.AddCustomLabel("service", _ => "orders-api");
    opts.ReduceStatusCodeCardinality(); // 2xx, 4xx, 5xx groups
});

app.MapMetrics("/metrics"); // Prometheus scrape endpoint

// Custom metrics
public static class OrderMetrics
{
    private static readonly Counter OrdersPlaced = Metrics.CreateCounter(
        "orders_placed_total",
        "Total number of orders placed",
        new CounterConfiguration
        {
            LabelNames = ["customer_tier", "channel"]
        });

    private static readonly Histogram OrderProcessingDuration = Metrics.CreateHistogram(
        "order_processing_duration_seconds",
        "Time to process an order",
        new HistogramConfiguration
        {
            Buckets = Histogram.PowersOfTenDividedBuckets(-3, 1, 4) // 1ms to 10s
        });

    private static readonly Gauge ActiveOrders = Metrics.CreateGauge(
        "orders_active",
        "Number of orders currently being processed");

    public static void RecordOrderPlaced(string customerTier, string channel)
    {
        OrdersPlaced.WithLabels(customerTier, channel).Inc();
        ActiveOrders.Inc();
    }

    public static IDisposable TrackOrderProcessing() =>
        OrderProcessingDuration.NewTimer();

    public static void RecordOrderCompleted()
    {
        ActiveOrders.Dec();
    }
}
```

---

## Step 1984: Kubernetes RBAC

```yaml
# ServiceAccount for the application pod
apiVersion: v1
kind: ServiceAccount
metadata:
  name: orders-api-sa
  namespace: production

---
# Role — minimal permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: orders-api-role
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["configmaps", "secrets"]
    resourceNames: ["orders-config", "orders-secrets"]
    verbs: ["get"]

---
# Bind role to service account
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: orders-api-rolebinding
  namespace: production
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: orders-api-role
subjects:
  - kind: ServiceAccount
    name: orders-api-sa
    namespace: production
```

---

## Step 1985: Zero-Downtime Database Migrations

```csharp
// InitContainers pattern — run migrations before app starts
// OR: migrate on startup but make it backward-compatible

// Backward-compatible migration strategy:
// 1. Deploy new code that handles BOTH old and new schema
// 2. Run migration (additive only — new column nullable)
// 3. Verify deployment
// 4. Separate migration to set NOT NULL / remove old column

// Startup migration (only for small apps)
public static async Task MigrateDatabase(IHost host)
{
    using var scope = host.Services.CreateScope();
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    var logger = scope.ServiceProvider.GetRequiredService<ILogger<Program>>();

    logger.LogInformation("Running database migrations...");
    await db.Database.MigrateAsync();
    logger.LogInformation("Database migrations complete.");
}
```

```yaml
# InitContainer pattern — migrations as a separate job
apiVersion: batch/v1
kind: Job
metadata:
  name: orders-api-migrate-{{ .Release.Revision }}
  namespace: production
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  backoffLimit: 3
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: migrate
          image: myregistry.azurecr.io/orders-api:{{ .Values.image.tag }}
          command: ["dotnet", "Orders.Api.dll", "--migrate-only"]
          env:
            - name: ConnectionStrings__Postgres
              valueFrom:
                secretKeyRef:
                  name: orders-secrets
                  key: postgres-connection-string
```

---

## Step 1986: Multi-Environment Configuration

```csharp
// Program.cs — environment-based configuration loading
var builder = WebApplication.CreateBuilder(args);

// appsettings.json → appsettings.Production.json → env vars → user secrets
// This is the default ASP.NET Core chain — already correct

// Kubernetes: inject environment-specific values via ConfigMap
// app can read any env var as a config key (__ = : separator)
```

```
Environment Variables in Kubernetes:
ASPNETCORE_ENVIRONMENT        = Production
Logging__LogLevel__Default    = Information  (maps to Logging:LogLevel:Default)
FeatureFlags__NewCheckout     = true
ConnectionStrings__Postgres   = Host=pg-svc;...
```

---

## Step 1987: Health Check Endpoints for Kubernetes

```csharp
// Program.cs — Kubernetes probe types
builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy(), tags: ["live"])
    .AddNpgSql(config.GetConnectionString("Postgres")!, tags: ["ready", "db"])
    .AddRedis(config.GetConnectionString("Redis")!, tags: ["ready", "cache"])
    .AddCheck<MessageBusHealthCheck>("rabbitmq", tags: ["ready", "messaging"])
    .AddCheck<ExternalApiHealthCheck>("payment-api", tags: ["ready"]);

// Separate endpoints for K8s probes
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("live"),
    ResponseWriter = (ctx, report) =>
    {
        ctx.Response.ContentType = "application/json";
        return ctx.Response.WriteAsync(
            report.Status == HealthStatus.Healthy ? """{"status":"ok"}""" : """{"status":"degraded"}""");
    }
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready"),
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

// /health — full report (authenticated)
app.MapHealthChecks("/health", new HealthCheckOptions
{
    Predicate = _ => true,
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
}).RequireAuthorization("AdminOnly");
```

---

## Step 1988: Deployment Checklist

```
Pre-deployment:
□ Docker image built with multi-stage, non-root user
□ Health check endpoints: /health/live and /health/ready
□ Graceful shutdown: ShutdownTimeout ≥ 15s, preStop sleep ≥ 5s
□ terminationGracePeriodSeconds > ShutdownTimeout + preStop
□ Resource requests and limits defined
□ Liveness and readiness probes configured
□ Startup probe for slow-starting apps
□ HPA configured (CPU + custom metrics)
□ PDB configured (minAvailable: 2)
□ Secrets in K8s Secret / External Secrets Operator
□ No secrets in ConfigMap or Deployment env
□ Network policies restrict ingress/egress
□ RBAC: minimal ServiceAccount permissions
□ Image pinned to exact digest or immutable tag (not :latest)

CI/CD:
□ Tests run before build
□ NuGet audit runs in CI
□ Docker image scanned (Trivy/Snyk)
□ Image signed (Cosign/Notary)
□ Rollout verified with kubectl rollout status
□ Smoke test after deployment
□ Rollback strategy documented

Observability:
□ Structured logging with correlation IDs
□ OpenTelemetry traces exported to Jaeger/Zipkin
□ Prometheus metrics at /metrics
□ Alerts configured for error rate, latency p99, memory
□ PagerDuty / on-call rotation configured
```

---

## Summary

| Resource | Purpose | Key Fields |
|----------|---------|-----------|
| `Deployment` | Run pods with replica management | `replicas`, `strategy`, probes |
| `Service` | Stable internal DNS + load balance | `type: ClusterIP`, `targetPort` |
| `Ingress` | External HTTP routing + TLS | `rules`, `tls`, `annotations` |
| `ConfigMap` | Non-secret configuration | `data` key-value pairs |
| `Secret` | Sensitive configuration | `stringData`, External Secrets |
| `HPA` | Auto-scale based on metrics | `minReplicas`, `targetCPUUtilization` |
| `PDB` | Maintain availability during disruption | `minAvailable` |
| `NetworkPolicy` | Firewall between pods | `ingress`/`egress` selectors |
| `ServiceAccount` | Pod identity for RBAC | mounted token |
| `Job` | One-off tasks (migrations) | `backoffLimit`, `restartPolicy` |

**Next**: Part 85 — Advanced Observability: OpenTelemetry, Grafana & Alerting
