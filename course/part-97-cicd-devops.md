# Part 97: CI/CD และ DevOps Pipelines

## Steps 2181-2196

การสร้าง production-grade CI/CD pipelines ด้วย GitHub Actions, Docker, Kubernetes deployment, และ infrastructure as code สำหรับ .NET applications

---

## Step 2181: CI/CD Strategy Overview

```
CI/CD Pipeline Flow
====================

Developer Push
     │
     ▼
┌─────────────────────────────────────────────┐
│              CI Pipeline                    │
│  1. Checkout + Restore                     │
│  2. Build (dotnet build)                   │
│  3. Test (unit + integration)              │
│  4. Code Analysis (SonarQube/CodeQL)       │
│  5. Security Scan (Trivy/Snyk)             │
│  6. Docker Build + Scan                   │
│  7. Push to Registry                      │
└─────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────┐
│              CD Pipeline                    │
│  Dev → Staging → Production                │
│  - Deploy to Kubernetes                   │
│  - Run smoke tests                        │
│  - Blue/Green or Canary deploy            │
│  - Health check verification              │
│  - Rollback on failure                    │
└─────────────────────────────────────────────┘
```

---

## Step 2182: GitHub Actions - CI Pipeline

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

env:
  DOTNET_VERSION: '9.0.x'
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build-and-test:
    name: Build and Test
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full history for SonarQube

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}

      - name: Cache NuGet packages
        uses: actions/cache@v4
        with:
          path: ~/.nuget/packages
          key: ${{ runner.os }}-nuget-${{ hashFiles('**/*.csproj') }}
          restore-keys: |
            ${{ runner.os }}-nuget-

      - name: Restore dependencies
        run: dotnet restore

      - name: Build
        run: dotnet build --no-restore --configuration Release

      - name: Run unit tests
        run: |
          dotnet test \
            --no-build \
            --configuration Release \
            --filter "Category=Unit" \
            --logger "trx;LogFileName=unit-tests.trx" \
            --collect:"XPlat Code Coverage" \
            --results-directory ./TestResults/Unit

      - name: Run integration tests
        env:
          ConnectionStrings__DefaultConnection: "Host=localhost;Database=testdb;Username=postgres;Password=postgres"
          ConnectionStrings__Redis: "localhost:6379"
        run: |
          dotnet test \
            --no-build \
            --configuration Release \
            --filter "Category=Integration" \
            --logger "trx;LogFileName=integration-tests.trx" \
            --results-directory ./TestResults/Integration

      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: TestResults/

      - name: Code coverage report
        uses: danielpalme/ReportGenerator-GitHub-Action@5
        with:
          reports: './TestResults/**/*.xml'
          targetdir: './TestResults/CoverageReport'
          reporttypes: 'HtmlInline;Cobertura'

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          directory: ./TestResults
          token: ${{ secrets.CODECOV_TOKEN }}

      - name: Publish test results
        uses: EnricoMi/publish-unit-test-result-action@v2
        if: always()
        with:
          files: './TestResults/**/*.trx'
```

---

## Step 2183: Security Scanning

```yaml
# Continuation of ci.yml

  security-scan:
    name: Security Scan
    runs-on: ubuntu-latest
    needs: build-and-test
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'

      - name: Upload Trivy scan results to GitHub Security tab
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: 'trivy-results.sarif'

      - name: Snyk security scan
        uses: snyk/actions/dotnet@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high

      - name: CodeQL Analysis
        uses: github/codeql-action/init@v3
        with:
          languages: csharp

      - name: Autobuild for CodeQL
        uses: github/codeql-action/autobuild@v3

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3

  docker-build:
    name: Docker Build and Push
    runs-on: ubuntu-latest
    needs: [build-and-test, security-scan]
    if: github.event_name == 'push'
    
    outputs:
      image-digest: ${{ steps.build.outputs.digest }}
      image-tag: ${{ steps.meta.outputs.tags }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=sha,prefix=sha-

      - name: Build and push Docker image
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          platforms: linux/amd64,linux/arm64
          build-args: |
            BUILD_VERSION=${{ github.sha }}
            BUILD_DATE=${{ github.event.head_commit.timestamp }}

      - name: Scan Docker image
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }}
          format: 'table'
          exit-code: '1'
          ignore-unfixed: true
          severity: 'CRITICAL'
```

---

## Step 2184: Optimized Dockerfile

```dockerfile
# Dockerfile
# Stage 1: Build
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
ARG BUILD_VERSION=1.0.0
ARG BUILD_DATE

WORKDIR /src

# Copy project files first for layer caching
COPY ["src/OrderService/OrderService.csproj", "src/OrderService/"]
COPY ["src/OrderService.Domain/OrderService.Domain.csproj", "src/OrderService.Domain/"]
COPY ["src/OrderService.Infrastructure/OrderService.Infrastructure.csproj", "src/OrderService.Infrastructure/"]

# Restore (cached layer if .csproj unchanged)
RUN dotnet restore "src/OrderService/OrderService.csproj"

# Copy source
COPY . .

# Build
RUN dotnet build "src/OrderService/OrderService.csproj" \
    -c Release \
    -p:Version=$BUILD_VERSION \
    --no-restore

# Publish
RUN dotnet publish "src/OrderService/OrderService.csproj" \
    -c Release \
    -o /app/publish \
    --no-build \
    -p:UseAppHost=false \
    -p:PublishSingleFile=false \
    -p:PublishReadyToRun=true

# Stage 2: Runtime
FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS runtime

# Security: run as non-root
RUN addgroup --system --gid 1001 appgroup && \
    adduser --system --uid 1001 --ingroup appgroup --shell /bin/false appuser

WORKDIR /app

# Copy published output
COPY --from=build --chown=appuser:appgroup /app/publish .

# Labels
LABEL org.opencontainers.image.created=$BUILD_DATE
LABEL org.opencontainers.image.version=$BUILD_VERSION
LABEL org.opencontainers.image.source="https://github.com/org/ecommerce"

# Non-root user
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health/live || exit 1

EXPOSE 8080
EXPOSE 8081

ENTRYPOINT ["dotnet", "OrderService.dll"]
```

**.dockerignore:**

```
**/.classpath
**/.dockerignore
**/.env
**/.git
**/.gitignore
**/.project
**/.settings
**/.toolstarget
**/.vs
**/.vscode
**/*.*proj.user
**/*.dbmdl
**/*.jfm
**/azds.yaml
**/bin
**/charts
**/docker-compose*
**/Dockerfile*
**/node_modules
**/npm-debug.log
**/obj
**/secrets.dev.yaml
**/values.dev.yaml
LICENSE
README.md
```

---

## Step 2185: CD Pipeline - Deploy to Kubernetes

```yaml
# .github/workflows/cd.yml
name: CD Pipeline

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      image-tag:
        required: true
        type: string

jobs:
  deploy:
    name: Deploy to ${{ inputs.environment }}
    runs-on: ubuntu-latest
    environment:
      name: ${{ inputs.environment }}
      url: ${{ vars.APP_URL }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up kubectl
        uses: azure/setup-kubectl@v4

      - name: Configure kubeconfig
        uses: azure/k8s-set-context@v4
        with:
          method: kubeconfig
          kubeconfig: ${{ secrets.KUBECONFIG }}

      - name: Update image tag in manifests
        run: |
          sed -i "s|IMAGE_TAG|${{ inputs.image-tag }}|g" \
            k8s/${{ inputs.environment }}/*.yaml

      - name: Deploy to Kubernetes
        run: |
          kubectl apply -f k8s/${{ inputs.environment }}/
          kubectl rollout status deployment/order-service \
            -n ecommerce \
            --timeout=5m

      - name: Run smoke tests
        run: |
          APP_URL="${{ vars.APP_URL }}"
          
          # Wait for service to be ready
          for i in {1..30}; do
            if curl -sf "${APP_URL}/health/ready" > /dev/null; then
              echo "Service is healthy"
              break
            fi
            echo "Waiting for service... attempt $i"
            sleep 10
          done

          # Run smoke tests
          curl -sf "${APP_URL}/health/ready" || exit 1
          curl -sf "${APP_URL}/api/products" || exit 1

      - name: Rollback on failure
        if: failure()
        run: |
          kubectl rollout undo deployment/order-service -n ecommerce
          echo "Rolled back due to deployment failure"

  deploy-staging:
    uses: ./.github/workflows/cd.yml
    with:
      environment: staging
      image-tag: ${{ needs.docker-build.outputs.image-tag }}
    needs: docker-build
    if: github.ref == 'refs/heads/main'

  deploy-production:
    uses: ./.github/workflows/cd.yml
    with:
      environment: production
      image-tag: ${{ needs.docker-build.outputs.image-tag }}
    needs: deploy-staging
    if: github.ref == 'refs/heads/main'
```

---

## Step 2186: Kubernetes Manifests

```yaml
# k8s/production/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: ecommerce
  labels:
    app: order-service
    version: v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: order-service
        version: v1
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8081"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: order-service
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        fsGroup: 1001
      containers:
        - name: order-service
          image: ghcr.io/org/order-service:IMAGE_TAG
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
              name: http
            - containerPort: 8081
              name: metrics
          env:
            - name: ASPNETCORE_ENVIRONMENT
              value: Production
            - name: ASPNETCORE_URLS
              value: "http://+:8080"
            - name: ConnectionStrings__DefaultConnection
              valueFrom:
                secretKeyRef:
                  name: order-service-secrets
                  key: db-connection-string
            - name: ConnectionStrings__Redis
              valueFrom:
                secretKeyRef:
                  name: order-service-secrets
                  key: redis-connection-string
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
              port: http
            initialDelaySeconds: 30
            periodSeconds: 30
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /health/ready
              port: http
            initialDelaySeconds: 10
            periodSeconds: 10
            failureThreshold: 3
          startupProbe:
            httpGet:
              path: /health/live
              port: http
            failureThreshold: 30
            periodSeconds: 10
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: order-service

---
# k8s/production/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
  namespace: ecommerce
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
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
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 100
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 25
          periodSeconds: 60
```

---

## Step 2187: Helm Chart

```yaml
# helm/order-service/Chart.yaml
apiVersion: v2
name: order-service
description: Order Service Helm Chart
type: application
version: 0.1.0
appVersion: "1.0.0"

# helm/order-service/values.yaml
replicaCount: 3

image:
  repository: ghcr.io/org/order-service
  pullPolicy: Always
  tag: ""

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: true
  className: nginx
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: api.ecommerce.com
      paths:
        - path: /api/orders
          pathType: Prefix
  tls:
    - secretName: api-tls
      hosts:
        - api.ecommerce.com

resources:
  requests:
    memory: 256Mi
    cpu: 100m
  limits:
    memory: 512Mi
    cpu: 500m

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70

env:
  ASPNETCORE_ENVIRONMENT: Production

secrets:
  dbConnectionString: ""
  redisConnectionString: ""
```

```yaml
# helm/order-service/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "order-service.fullname" . }}
  labels: {{- include "order-service.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels: {{- include "order-service.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels: {{- include "order-service.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: 8080
          env:
            {{- range $key, $value := .Values.env }}
            - name: {{ $key }}
              value: {{ $value | quote }}
            {{- end }}
          resources: {{- toYaml .Values.resources | nindent 12 }}
```

```bash
# Deploy with Helm
helm upgrade --install order-service ./helm/order-service \
  --namespace ecommerce \
  --create-namespace \
  --set image.tag=$(git rev-parse --short HEAD) \
  --set secrets.dbConnectionString=$DB_CONN \
  --set secrets.redisConnectionString=$REDIS_CONN \
  --wait \
  --timeout 5m
```

---

## Step 2188: Infrastructure as Code - Terraform

```hcl
# infrastructure/main.tf

terraform {
  required_version = ">= 1.5"
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.0"
    }
  }
  backend "azurerm" {
    resource_group_name  = "terraform-state"
    storage_account_name = "tfstate${var.environment}"
    container_name       = "tfstate"
    key                  = "ecommerce.tfstate"
  }
}

resource "azurerm_resource_group" "main" {
  name     = "rg-ecommerce-${var.environment}"
  location = var.location
}

resource "azurerm_kubernetes_cluster" "aks" {
  name                = "aks-ecommerce-${var.environment}"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  dns_prefix          = "ecommerce-${var.environment}"

  default_node_pool {
    name                = "system"
    node_count          = 3
    vm_size             = "Standard_D4s_v3"
    enable_auto_scaling = true
    min_count           = 3
    max_count           = 10
    os_disk_size_gb     = 100
    type                = "VirtualMachineScaleSets"
    zones               = ["1", "2", "3"]
  }

  identity {
    type = "SystemAssigned"
  }

  network_profile {
    network_plugin    = "azure"
    network_policy    = "calico"
    load_balancer_sku = "standard"
  }

  oms_agent {
    log_analytics_workspace_id = azurerm_log_analytics_workspace.main.id
  }
}

resource "azurerm_postgresql_flexible_server" "main" {
  name                   = "psql-ecommerce-${var.environment}"
  resource_group_name    = azurerm_resource_group.main.name
  location               = azurerm_resource_group.main.location
  version                = "16"
  administrator_login    = var.db_admin_username
  administrator_password = var.db_admin_password

  storage_mb = 32768

  sku_name = var.environment == "production" ? "GP_Standard_D4s_v3" : "B_Standard_B2s"

  high_availability {
    mode = var.environment == "production" ? "ZoneRedundant" : "Disabled"
  }

  maintenance_window {
    day_of_week  = 0
    start_hour   = 2
    start_minute = 0
  }
}
```

---

## Step 2189: Release Management - Semantic Versioning

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    branches: [main]

jobs:
  semantic-release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      issues: write
      pull-requests: write
      packages: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
          persist-credentials: false

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install semantic-release
        run: |
          npm install -g \
            semantic-release \
            @semantic-release/git \
            @semantic-release/changelog \
            @semantic-release/exec

      - name: Release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: npx semantic-release

# .releaserc.json
{
  "branches": ["main"],
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    ["@semantic-release/changelog", {
      "changelogFile": "CHANGELOG.md"
    }],
    ["@semantic-release/exec", {
      "prepareCmd": "dotnet build -c Release -p:Version=${nextRelease.version}"
    }],
    ["@semantic-release/git", {
      "assets": ["CHANGELOG.md", "**/*.csproj"],
      "message": "chore(release): ${nextRelease.version} [skip ci]"
    }],
    "@semantic-release/github"
  ]
}
```

---

## Step 2190: Monitoring และ Alerting

```yaml
# .github/workflows/deploy-monitoring.yml
# Prometheus + Grafana deployment

# k8s/monitoring/prometheus-values.yaml
prometheus:
  prometheusSpec:
    retention: 30d
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: premium-lrs
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 50Gi
    additionalScrapeConfigs:
      - job_name: 'ecommerce-services'
        kubernetes_sd_configs:
          - role: pod
            namespaces:
              names: ['ecommerce']
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
            action: keep
            regex: "true"

alertmanager:
  config:
    route:
      group_by: ['alertname', 'cluster', 'service']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
      receiver: 'slack-notifications'
      routes:
        - match:
            severity: critical
          receiver: 'pagerduty'

    receivers:
      - name: 'slack-notifications'
        slack_configs:
          - api_url: '{{ slack_api_url }}'
            channel: '#alerts'
            title: '{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'
            text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

      - name: 'pagerduty'
        pagerduty_configs:
          - service_key: '{{ pagerduty_key }}'
```

**Custom Prometheus alerts:**

```yaml
# k8s/monitoring/alerts.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: ecommerce-alerts
  namespace: ecommerce
spec:
  groups:
    - name: order-service
      interval: 30s
      rules:
        - alert: HighErrorRate
          expr: |
            (
              sum(rate(http_requests_total{job="order-service",status=~"5.."}[5m]))
              /
              sum(rate(http_requests_total{job="order-service"}[5m]))
            ) > 0.05
          for: 2m
          labels:
            severity: critical
          annotations:
            summary: "High error rate on Order Service"
            description: "Error rate is {{ $value | humanizePercentage }} for the last 5 minutes"

        - alert: HighLatency
          expr: |
            histogram_quantile(0.95,
              sum(rate(http_request_duration_seconds_bucket{job="order-service"}[5m]))
              by (le)
            ) > 2
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "High latency on Order Service"
            description: "P95 latency is {{ $value }}s"

        - alert: PodNotReady
          expr: |
            kube_pod_status_ready{namespace="ecommerce",condition="true"} == 0
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Pod {{ $labels.pod }} is not ready"
```

---

## Step 2191: Database Migrations in CI/CD

```csharp
// DbMigrator project for running migrations
// ECommerceAspire.DbMigrator/Program.cs
using Microsoft.EntityFrameworkCore;

var host = Host.CreateDefaultBuilder(args)
    .ConfigureServices((context, services) =>
    {
        services.AddDbContext<OrderDbContext>(options =>
            options.UseNpgsql(context.Configuration.GetConnectionString("DefaultConnection")));
        services.AddDbContext<CatalogDbContext>(options =>
            options.UseNpgsql(context.Configuration.GetConnectionString("CatalogConnection")));
    })
    .Build();

using var scope = host.Services.CreateScope();

await MigrateAsync<OrderDbContext>(scope.ServiceProvider);
await MigrateAsync<CatalogDbContext>(scope.ServiceProvider);

static async Task MigrateAsync<TContext>(IServiceProvider services)
    where TContext : DbContext
{
    var logger = services.GetRequiredService<ILogger<TContext>>();
    var db = services.GetRequiredService<TContext>();

    try
    {
        logger.LogInformation("Migrating {ContextName}...", typeof(TContext).Name);
        await db.Database.MigrateAsync();
        logger.LogInformation("{ContextName} migrated successfully", typeof(TContext).Name);
    }
    catch (Exception ex)
    {
        logger.LogError(ex, "Failed to migrate {ContextName}", typeof(TContext).Name);
        throw;
    }
}
```

```yaml
# In CD pipeline - run migrations before deploying app
- name: Run database migrations
  run: |
    docker run --rm \
      --network host \
      -e ConnectionStrings__DefaultConnection="${{ secrets.DB_CONNECTION }}" \
      ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}-migrator:${{ inputs.image-tag }}
```

---

## Step 2192: Feature Flags with Azure App Configuration

```csharp
// Feature flags in production
// Package: Microsoft.Azure.AppConfiguration.AspNetCore
// Package: Microsoft.FeatureManagement.AspNetCore

builder.AddAzureAppConfiguration(options =>
{
    options.Connect(builder.Configuration["AzureAppConfig:ConnectionString"])
        .Select("ECommerce:*", LabelFilter.Null)
        .Select("ECommerce:*", environmentLabel)
        .ConfigureRefresh(refresh =>
        {
            refresh.Register("ECommerce:Sentinel", refreshAll: true)
                   .SetCacheExpiration(TimeSpan.FromSeconds(30));
        })
        .UseFeatureFlags(featureOptions =>
        {
            featureOptions.Label = environmentLabel;
            featureOptions.CacheExpirationInterval = TimeSpan.FromSeconds(30);
        });
});

builder.Services.AddFeatureManagement()
    .AddFeatureFilter<PercentageFilter>()
    .AddFeatureFilter<TargetingFilter>()
    .AddFeatureFilter<TimeWindowFilter>();

// Feature flag usage
public class CheckoutService(IFeatureManager featureManager)
{
    public async Task<CheckoutResult> ProcessCheckoutAsync(CheckoutRequest request, CancellationToken ct)
    {
        if (await featureManager.IsEnabledAsync("NewCheckoutFlow", ct))
        {
            return await ProcessNewCheckoutAsync(request, ct);
        }
        return await ProcessLegacyCheckoutAsync(request, ct);
    }
}

// Targeted rollout - 10% of users
[FeatureGate("NewRecommendationEngine")]
public async Task<IActionResult> GetRecommendations(Guid userId) { ... }
```

---

## Step 2193: Blue/Green Deployment

```yaml
# k8s/blue-green/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: ecommerce
spec:
  selector:
    app: order-service
    slot: green  # Switch between 'blue' and 'green'
  ports:
    - port: 80
      targetPort: 8080

---
# Blue deployment (current)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service-blue
  namespace: ecommerce
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
      slot: blue
  template:
    metadata:
      labels:
        app: order-service
        slot: blue
    spec:
      containers:
        - name: order-service
          image: ghcr.io/org/order-service:v1.2.0

---
# Green deployment (new version, inactive)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service-green
  namespace: ecommerce
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
      slot: green
  template:
    metadata:
      labels:
        app: order-service
        slot: green
    spec:
      containers:
        - name: order-service
          image: ghcr.io/org/order-service:v1.3.0
```

```bash
# Blue/Green switch script
#!/bin/bash
CURRENT_SLOT=$(kubectl get svc order-service -n ecommerce \
  -o jsonpath='{.spec.selector.slot}')

if [ "$CURRENT_SLOT" == "blue" ]; then
  NEW_SLOT="green"
else
  NEW_SLOT="blue"
fi

echo "Switching from $CURRENT_SLOT to $NEW_SLOT"

# Switch traffic
kubectl patch svc order-service -n ecommerce \
  -p "{\"spec\":{\"selector\":{\"slot\":\"$NEW_SLOT\"}}}"

# Wait and verify
sleep 10
if curl -sf https://api.ecommerce.com/health/ready; then
  echo "Deployment successful!"
  kubectl scale deployment order-service-$CURRENT_SLOT -n ecommerce --replicas=0
else
  echo "Deployment failed, rolling back..."
  kubectl patch svc order-service -n ecommerce \
    -p "{\"spec\":{\"selector\":{\"slot\":\"$CURRENT_SLOT\"}}}"
fi
```

---

## Step 2194: Observability Stack

```yaml
# docker-compose.observability.yml
version: '3.8'
services:
  # OpenTelemetry Collector
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    volumes:
      - ./otel-config.yaml:/etc/otel/config.yaml
    command: ["--config", "/etc/otel/config.yaml"]
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
      - "8888:8888"   # Prometheus metrics

  # Jaeger for distributed tracing
  jaeger:
    image: jaegertracing/all-in-one:1.57
    ports:
      - "16686:16686"  # Jaeger UI
      - "14250:14250"  # gRPC

  # Prometheus
  prometheus:
    image: prom/prometheus:v2.50.1
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  # Grafana
  grafana:
    image: grafana/grafana:10.3.3
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards
      - grafana-data:/var/lib/grafana
    ports:
      - "3000:3000"

  # Loki for logs
  loki:
    image: grafana/loki:2.9.5
    ports:
      - "3100:3100"

# otel-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

exporters:
  jaeger:
    endpoint: jaeger:14250
    tls:
      insecure: true
  prometheus:
    endpoint: "0.0.0.0:8889"
  loki:
    endpoint: http://loki:3100/loki/api/v1/push

service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [jaeger]
    metrics:
      receivers: [otlp]
      exporters: [prometheus]
    logs:
      receivers: [otlp]
      exporters: [loki]
```

---

## Step 2195: Environment Configuration Strategy

```csharp
// Configuration hierarchy (highest priority first):
// 1. Environment variables (production secrets)
// 2. Azure Key Vault / AWS Secrets Manager
// 3. appsettings.{Environment}.json
// 4. appsettings.json (defaults)
// 5. User secrets (development only)

// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Azure Key Vault in production
if (builder.Environment.IsProduction())
{
    var keyVaultUri = new Uri(builder.Configuration["AzureKeyVault:Uri"]!);
    builder.Configuration.AddAzureKeyVault(keyVaultUri, new DefaultAzureCredential());
}

// Validate required config at startup
builder.Services.AddOptions<DatabaseOptions>()
    .BindConfiguration("Database")
    .ValidateDataAnnotations()
    .ValidateOnStart();

builder.Services.AddOptions<JwtOptions>()
    .BindConfiguration("Jwt")
    .Validate(o => !string.IsNullOrEmpty(o.SecretKey) && o.SecretKey.Length >= 32,
        "JWT SecretKey must be at least 32 characters")
    .ValidateOnStart();

// appsettings.json
{
  "Database": {
    "MaxPoolSize": 100,
    "CommandTimeout": 30
  },
  "Jwt": {
    "Issuer": "https://auth.ecommerce.com",
    "Audience": "https://api.ecommerce.com",
    "ExpiryMinutes": 15
  },
  "FeatureManagement": {
    "NewCheckoutFlow": false,
    "AIRecommendations": {
      "EnabledFor": [
        {
          "Name": "Percentage",
          "Parameters": { "Value": 10 }
        }
      ]
    }
  }
}
```

---

## Step 2196: Complete CI/CD Summary

```
Production-Grade CI/CD Pipeline
=================================

Pull Request:
  ✓ Automated build verification
  ✓ Unit + Integration tests
  ✓ Code coverage gate (>80%)
  ✓ Security scanning (CodeQL + Trivy + Snyk)
  ✓ Required reviewers
  
Merge to main:
  ✓ Semantic versioning (conventional commits)
  ✓ CHANGELOG generation
  ✓ Multi-platform Docker build (amd64 + arm64)
  ✓ Docker image security scan
  ✓ Push to container registry
  
Deploy to Staging:
  ✓ Database migration (zero-downtime)
  ✓ Blue/Green or Rolling update
  ✓ Smoke tests
  ✓ Auto-rollback on failure
  
Deploy to Production:
  ✓ Manual approval gate
  ✓ Blue/Green deployment
  ✓ Canary analysis (optional)
  ✓ Prometheus alerts active
  ✓ Grafana dashboards
  
Tools Used:
  - GitHub Actions (CI/CD orchestration)
  - Docker + BuildKit (containerization)
  - Helm (Kubernetes packaging)
  - Terraform (infrastructure)
  - Prometheus + Grafana (monitoring)
  - Jaeger (tracing)
  - Loki (logs)
  - Azure App Configuration (feature flags)
  - Semantic Release (versioning)
```
