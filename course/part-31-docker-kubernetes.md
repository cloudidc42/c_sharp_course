# Part 31: Docker, Containers & Kubernetes

## Steps 846-880: Container Orchestration

---

## Step 846: Docker Fundamentals สำหรับ .NET

```dockerfile
# Multi-stage build สำหรับ .NET 9
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /src

# Restore first (cache layer optimization)
COPY ["MyApp/MyApp.csproj", "MyApp/"]
RUN dotnet restore "MyApp/MyApp.csproj"

# Build
COPY . .
WORKDIR "/src/MyApp"
RUN dotnet build "MyApp.csproj" -c Release -o /app/build

# Publish
FROM build AS publish
RUN dotnet publish "MyApp.csproj" -c Release -o /app/publish \
    /p:UseAppHost=false \
    /p:PublishSingleFile=false \
    /p:TrimmedCopyLocalFileList=""

# Runtime image (smallest possible)
FROM mcr.microsoft.com/dotnet/aspnet:9.0-alpine AS final
WORKDIR /app

# Non-root user for security
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

COPY --from=publish --chown=appuser:appgroup /app/publish .
EXPOSE 8080
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

---

## Step 847: Docker Compose สำหรับ Development

```yaml
# docker-compose.yml
version: "3.9"

services:
  # Application
  api:
    build:
      context: .
      dockerfile: src/MyApp.Api/Dockerfile
      target: final
    ports:
      - "5000:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__DefaultConnection=Host=postgres;Database=myapp;Username=postgres;Password=secret
      - ConnectionStrings__Redis=redis:6379
      - Jwt__Secret=my-super-secret-key-minimum-32-characters
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    volumes:
      - ./src:/app/src  # Hot reload
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    networks:
      - app-network

  # Database
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: secret
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./scripts/init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network

  # Cache
  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes --requirepass secret
    volumes:
      - redis-data:/data
    ports:
      - "6379:6379"
    networks:
      - app-network

  # Message Queue
  rabbitmq:
    image: rabbitmq:3.13-management-alpine
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: secret
    volumes:
      - rabbitmq-data:/var/lib/rabbitmq
    ports:
      - "5672:5672"
      - "15672:15672"
    networks:
      - app-network

  # Reverse Proxy (dev/staging)
  nginx:
    image: nginx:alpine
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - api
    networks:
      - app-network

volumes:
  postgres-data:
  redis-data:
  rabbitmq-data:

networks:
  app-network:
    driver: bridge
```

```yaml
# docker-compose.override.yml (Development overrides)
version: "3.9"

services:
  api:
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ASPNETCORE_URLS=http://+:8080
    volumes:
      - ~/.nuget:/root/.nuget  # Share NuGet cache
    command: dotnet watch run --project /app/src/MyApp.Api

  postgres:
    ports:
      - "5432:5432"  # Expose for external tools

  pgadmin:
    image: dpage/pgadmin4
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@example.com
      PGADMIN_DEFAULT_PASSWORD: admin
    ports:
      - "8888:80"
    networks:
      - app-network
```

---

## Step 848: Docker Secrets และ Configs

```yaml
# docker-compose.production.yml
version: "3.9"

services:
  api:
    image: myregistry.azurecr.io/myapp:${IMAGE_TAG}
    secrets:
      - db_password
      - jwt_secret
    configs:
      - source: app_config
        target: /app/appsettings.Production.json
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - DB_PASSWORD_FILE=/run/secrets/db_password
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
        order: start-first
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health/live"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

secrets:
  db_password:
    external: true
  jwt_secret:
    external: true

configs:
  app_config:
    file: ./appsettings.Production.json
```

```csharp
// Reading Docker secrets in .NET
public static class SecretHelper
{
    public static string? GetSecretOrEnv(string secretName, string envName)
    {
        var secretFile = $"/run/secrets/{secretName}";
        if (File.Exists(secretFile))
            return File.ReadAllText(secretFile).Trim();
        return Environment.GetEnvironmentVariable(envName);
    }
}

// Program.cs
builder.Configuration.AddKeyPerFile("/run/secrets", optional: true);
// or
var dbPassword = SecretHelper.GetSecretOrEnv("db_password", "DB_PASSWORD");
```

---

## Step 849: NGINX Configuration

```nginx
# nginx.conf
events {
    worker_connections 1024;
}

http {
    upstream api_backend {
        server api:8080;
        keepalive 32;
    }
    
    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
    
    server {
        listen 80;
        server_name myapp.example.com;
        return 301 https://$server_name$request_uri;
    }
    
    server {
        listen 443 ssl http2;
        server_name myapp.example.com;
        
        ssl_certificate /etc/nginx/ssl/cert.pem;
        ssl_certificate_key /etc/nginx/ssl/key.pem;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers HIGH:!aNULL:!MD5;
        
        # Security headers
        add_header X-Frame-Options DENY;
        add_header X-Content-Type-Options nosniff;
        add_header X-XSS-Protection "1; mode=block";
        add_header Strict-Transport-Security "max-age=31536000" always;
        
        location /api {
            limit_req zone=api_limit burst=20 nodelay;
            
            proxy_pass http://api_backend;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection keep-alive;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_cache_bypass $http_upgrade;
            
            proxy_read_timeout 30s;
            proxy_connect_timeout 5s;
        }
        
        # WebSocket support (SignalR)
        location /hubs {
            proxy_pass http://api_backend;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "Upgrade";
            proxy_set_header Host $host;
            proxy_read_timeout 3600s;
        }
        
        # Static files
        location / {
            root /usr/share/nginx/html;
            try_files $uri $uri/ /index.html;
            
            location ~* \.(js|css|png|jpg|ico|svg)$ {
                expires 1y;
                add_header Cache-Control "public, immutable";
            }
        }
    }
}
```

---

## Step 850: Kubernetes Basics

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
  labels:
    app: myapp
    env: production
```

```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: myapp
data:
  ASPNETCORE_ENVIRONMENT: "Production"
  Logging__LogLevel__Default: "Warning"
  AllowedHosts: "*"
  Redis__Configuration: "redis-service:6379"
```

```yaml
# k8s/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: myapp
type: Opaque
data:
  # base64 encoded values
  DB_CONNECTION_STRING: cG9zdGdyZXM6Ly91c2VyOnBhc3NAaG9zdC9kYg==
  JWT_SECRET: c3VwZXItc2VjcmV0LWtleS0yNTYtYml0cw==
  REDIS_PASSWORD: cmVkaXNwYXNz
```

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
  namespace: myapp
  labels:
    app: api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: api
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: api-service-account
      
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
          
          env:
            - name: ASPNETCORE_ENVIRONMENT
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: ASPNETCORE_ENVIRONMENT
            - name: ConnectionStrings__Default
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: DB_CONNECTION_STRING
          
          envFrom:
            - configMapRef:
                name: app-config
          
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
            initialDelaySeconds: 15
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
            periodSeconds: 10
          
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
      
      terminationGracePeriodSeconds: 60
      
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: api
```

---

## Step 851: Kubernetes Service และ Ingress

```yaml
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: myapp
spec:
  selector:
    app: api
  ports:
    - name: http
      port: 80
      targetPort: 8080
  type: ClusterIP
---
# Internal service for database
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  namespace: myapp
spec:
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
  type: ClusterIP
```

```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: myapp
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "100m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "60"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.example.com
      secretName: api-tls
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
```

---

## Step 852: StatefulSet สำหรับ Databases

```yaml
# k8s/postgres-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: myapp
spec:
  serviceName: postgres-headless
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          env:
            - name: POSTGRES_DB
              value: "myapp"
            - name: POSTGRES_USER
              value: "postgres"
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: DB_PASSWORD
          ports:
            - containerPort: 5432
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "1000m"
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
          readinessProbe:
            exec:
              command:
                - /bin/sh
                - -c
                - pg_isready -U postgres
            initialDelaySeconds: 10
            periodSeconds: 5
  volumeClaimTemplates:
    - metadata:
        name: postgres-data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: "fast-ssd"
        resources:
          requests:
            storage: 20Gi
```

---

## Step 853: HPA and VPA

```yaml
# k8s/hpa.yaml - Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
  namespace: myapp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-deployment
  minReplicas: 2
  maxReplicas: 20
  metrics:
    # CPU
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    # Memory
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
    # Custom metric (requests per second)
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 4
          periodSeconds: 60
        - type: Percent
          value: 100
          periodSeconds: 60
      selectPolicy: Max
```

---

## Step 854: Network Policies

```yaml
# k8s/network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-network-policy
  namespace: myapp
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # Allow from ingress controller
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - protocol: TCP
          port: 8080
    # Allow from other services in same namespace
    - from:
        - podSelector:
            matchLabels:
              role: frontend
  egress:
    # Allow to database
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432
    # Allow to Redis
    - to:
        - podSelector:
            matchLabels:
              app: redis
      ports:
        - protocol: TCP
          port: 6379
    # Allow DNS
    - to: []
      ports:
        - protocol: UDP
          port: 53
```

---

## Step 855: PodDisruptionBudget

```yaml
# k8s/pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-pdb
  namespace: myapp
spec:
  minAvailable: 2  # หรือใช้ maxUnavailable: 1
  selector:
    matchLabels:
      app: api
```

---

## Step 856: ConfigMap และ Secret ขั้นสูง

```yaml
# k8s/external-secrets.yaml (External Secrets Operator)
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: api-secrets
  namespace: myapp
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: azure-keyvault
    kind: ClusterSecretStore
  target:
    name: app-secrets
    creationPolicy: Owner
  data:
    - secretKey: DB_CONNECTION_STRING
      remoteRef:
        key: api-db-connection
    - secretKey: JWT_SECRET
      remoteRef:
        key: api-jwt-secret
```

---

## Step 857: Helm Charts

```
myapp-chart/
├── Chart.yaml
├── values.yaml
├── values.production.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── hpa.yaml
    ├── configmap.yaml
    └── _helpers.tpl
```

```yaml
# Chart.yaml
apiVersion: v2
name: myapp
description: A .NET web application
type: application
version: 0.1.0
appVersion: "1.0.0"
```

```yaml
# values.yaml
replicaCount: 2

image:
  repository: myregistry.azurecr.io/myapp-api
  pullPolicy: IfNotPresent
  tag: "latest"

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  className: "nginx"
  host: myapp.example.com
  tls: true

resources:
  requests:
    memory: "256Mi"
    cpu: "100m"
  limits:
    memory: "512Mi"
    cpu: "500m"

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

env:
  ASPNETCORE_ENVIRONMENT: "Production"

secrets: {}
  # DB_CONNECTION_STRING: ""
  # JWT_SECRET: ""
```

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "myapp.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: 8080
          env:
            {{- range $key, $value := .Values.env }}
            - name: {{ $key }}
              value: {{ $value | quote }}
            {{- end }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

```bash
# Helm commands
helm create myapp-chart
helm install myapp ./myapp-chart -f values.production.yaml
helm upgrade myapp ./myapp-chart -f values.production.yaml
helm rollback myapp 1
helm uninstall myapp
helm history myapp
```

---

## Step 858: Kubernetes Operators

```csharp
// Custom Resource Definition (CRD) managed via C# operator
// Install KubeOps
dotnet add package KubeOps.Operator

// CRD Entity
[KubernetesEntity(Group = "myapp.io", ApiVersion = "v1", Kind = "DatabaseBackup")]
public class DatabaseBackupEntity : CustomKubernetesEntity<DatabaseBackupSpec, DatabaseBackupStatus>
{
}

public class DatabaseBackupSpec
{
    public string DatabaseName { get; set; } = string.Empty;
    public string Schedule { get; set; } = "0 * * * *"; // hourly
    public string StorageBucket { get; set; } = string.Empty;
    public int RetentionDays { get; set; } = 7;
}

public class DatabaseBackupStatus
{
    public string Phase { get; set; } = "Pending";
    public DateTime? LastBackupAt { get; set; }
    public string? LastBackupFile { get; set; }
    public int FailureCount { get; set; }
}

// Controller
[EntityRbac(typeof(DatabaseBackupEntity), Verbs = RbacVerb.All)]
public class DatabaseBackupController(
    ILogger<DatabaseBackupController> logger,
    IBackupService backupService)
    : IResourceController<DatabaseBackupEntity>
{
    public async Task ReconcileAsync(DatabaseBackupEntity entity, CancellationToken ct)
    {
        logger.LogInformation(
            "Reconciling DatabaseBackup {Name}", entity.Name());
        
        entity.Status.Phase = "Running";
        
        try
        {
            var backupFile = await backupService.CreateBackupAsync(
                entity.Spec.DatabaseName,
                entity.Spec.StorageBucket,
                ct);
            
            entity.Status.Phase = "Completed";
            entity.Status.LastBackupAt = DateTime.UtcNow;
            entity.Status.LastBackupFile = backupFile;
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "Backup failed");
            entity.Status.Phase = "Failed";
            entity.Status.FailureCount++;
        }
    }
    
    public Task DeletedAsync(DatabaseBackupEntity entity, CancellationToken ct)
    {
        logger.LogInformation("DatabaseBackup deleted: {Name}", entity.Name());
        return Task.CompletedTask;
    }
}
```

---

## Step 859: GitOps with ArgoCD

```yaml
# argocd/application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  
  source:
    repoURL: https://github.com/myorg/myapp-deploy
    targetRevision: HEAD
    path: k8s/overlays/production
  
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp
  
  syncPolicy:
    automated:
      prune: true        # Delete resources not in Git
      selfHeal: true     # Auto-fix drift
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - ApplyOutOfSyncOnly=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

```yaml
# kustomize/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - ingress.yaml
  - hpa.yaml

commonLabels:
  app: myapp
  managed-by: kustomize

# kustomize/overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
  - ../../base

namespace: myapp-production

patchesStrategicMerge:
  - deployment-patch.yaml

images:
  - name: myapp-api
    newTag: v1.2.3
```

---

## Step 860: CI/CD Pipeline

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy

on:
  push:
    branches: [main]
    tags: ['v*']
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_PASSWORD: test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '9.0.x'
      - name: Restore
        run: dotnet restore
      - name: Test
        run: dotnet test --no-restore --logger "trx;LogFileName=test-results.trx"
        env:
          ConnectionStrings__TestDb: "Host=localhost;Database=testdb;Username=postgres;Password=test"
      - name: Publish Test Results
        uses: dorny/test-reporter@v1
        if: always()
        with:
          name: Tests
          path: '**/*.trx'
          reporter: dotnet-trx
  
  build:
    needs: test
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    steps:
      - uses: actions/checkout@v4
      - name: Log in to Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=sha,prefix={{branch}}-
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
  
  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production
    steps:
      - uses: actions/checkout@v4
      - name: Configure kubectl
        uses: azure/k8s-set-context@v4
        with:
          method: kubeconfig
          kubeconfig: ${{ secrets.KUBECONFIG }}
      - name: Deploy to Kubernetes
        uses: azure/k8s-deploy@v5
        with:
          namespace: myapp
          manifests: |
            k8s/deployment.yaml
            k8s/service.yaml
          images: ${{ needs.build.outputs.image-tag }}
          strategy: rolling
```

---

## Step 861: Observability Stack

```yaml
# k8s/prometheus-stack.yaml (Prometheus + Grafana)
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: kube-prometheus-stack
  namespace: monitoring
spec:
  interval: 30m
  chart:
    spec:
      chart: kube-prometheus-stack
      sourceRef:
        kind: HelmRepository
        name: prometheus-community
  values:
    grafana:
      adminPassword: "changeme"
      ingress:
        enabled: true
        hosts:
          - grafana.example.com
    prometheus:
      prometheusSpec:
        retention: 30d
        storageSpec:
          volumeClaimTemplate:
            spec:
              storageClassName: fast-ssd
              accessModes: ["ReadWriteOnce"]
              resources:
                requests:
                  storage: 50Gi
```

```csharp
// .NET Prometheus metrics
builder.Services.AddOpenTelemetry()
    .WithMetrics(metrics =>
    {
        metrics
            .AddAspNetCoreInstrumentation()
            .AddHttpClientInstrumentation()
            .AddRuntimeInstrumentation()
            .AddPrometheusExporter();
    });

app.MapPrometheusScrapingEndpoint("/metrics");

// Custom metric
var meter = new Meter("MyApp.Api");
var requestCounter = meter.CreateCounter<long>(
    "api.requests.total",
    description: "Total API requests");

app.Use(async (context, next) =>
{
    requestCounter.Add(1, 
        new TagList
        {
            { "method", context.Request.Method },
            { "path", context.Request.Path.Value }
        });
    await next();
});
```

---

## Step 862: Loki (Log Aggregation)

```csharp
// Send logs to Loki via OpenTelemetry
builder.Logging.AddOpenTelemetry(logging =>
{
    logging.IncludeFormattedMessage = true;
    logging.IncludeScopes = true;
    logging.ParseStateValues = true;
});

builder.Services.AddOpenTelemetry()
    .WithLogging(logging =>
    {
        logging.AddOtlpExporter(opts =>
        {
            opts.Endpoint = new Uri("http://loki:4317");
        });
    });

// Structured logging
logger.LogInformation("Processing order {OrderId} for user {UserId}", orderId, userId);
// This creates: {"OrderId": "abc", "UserId": "123", "message": "Processing order..."}
```

---

## Step 863: Distributed Tracing with Jaeger

```csharp
// Program.cs
builder.Services.AddOpenTelemetry()
    .ConfigureResource(resource =>
    {
        resource.AddService("myapp-api",
            serviceVersion: "1.0.0",
            serviceInstanceId: Environment.MachineName);
    })
    .WithTracing(tracing =>
    {
        tracing
            .AddAspNetCoreInstrumentation(opts =>
            {
                opts.RecordException = true;
                opts.Filter = httpContext =>
                    !httpContext.Request.Path.StartsWithSegments("/health");
            })
            .AddHttpClientInstrumentation()
            .AddEntityFrameworkCoreInstrumentation(opts =>
            {
                opts.SetDbStatementForText = true;
            })
            .AddJaegerExporter(opts =>
            {
                opts.AgentHost = builder.Configuration["Jaeger:Host"] ?? "jaeger";
                opts.AgentPort = 6831;
            })
            .AddOtlpExporter();
    });
```

---

## Step 864: Security Scanning

```yaml
# .github/workflows/security.yml
name: Security Scan

on:
  schedule:
    - cron: '0 6 * * 1'  # Weekly Monday 6AM
  push:
    branches: [main]

jobs:
  container-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build image
        run: docker build -t myapp:scan .
      
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'myapp:scan'
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
      
      - name: Upload Trivy results
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: 'trivy-results.sarif'
  
  code-scan:
    runs-on: ubuntu-latest
    permissions:
      actions: read
      contents: read
      security-events: write
    steps:
      - uses: actions/checkout@v4
      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: csharp
      - name: Build
        run: dotnet build
      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3
  
  dependency-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '9.0.x'
      - name: Check for vulnerable packages
        run: dotnet list package --vulnerable --include-transitive
```

---

## Step 865: Production Checklist

```csharp
// Health checks comprehensive
builder.Services.AddHealthChecks()
    .AddDbContextCheck<AppDbContext>(
        name: "database",
        tags: ["db", "ready"])
    .AddRedis(
        redisConnectionString: builder.Configuration["Redis:ConnectionString"]!,
        name: "redis",
        tags: ["cache", "ready"])
    .AddRabbitMQ(
        rabbitConnectionString: builder.Configuration["RabbitMQ:ConnectionString"]!,
        name: "rabbitmq",
        tags: ["messaging", "ready"])
    .AddUrlGroup(
        new Uri("https://api.external-service.com/health"),
        name: "external-api",
        tags: ["external"])
    .AddCheck<DiskSpaceHealthCheck>(
        name: "disk-space",
        tags: ["infra"]);

// Custom disk space health check
public class DiskSpaceHealthCheck : IHealthCheck
{
    public Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken ct)
    {
        var drive = new DriveInfo(Path.GetPathRoot(Directory.GetCurrentDirectory())!);
        var freeGB = drive.AvailableFreeSpace / 1024.0 / 1024.0 / 1024.0;
        
        if (freeGB < 1)
            return Task.FromResult(HealthCheckResult.Unhealthy(
                $"Low disk space: {freeGB:F1}GB remaining"));
        
        if (freeGB < 5)
            return Task.FromResult(HealthCheckResult.Degraded(
                $"Disk space warning: {freeGB:F1}GB remaining"));
        
        return Task.FromResult(HealthCheckResult.Healthy(
            $"Disk space OK: {freeGB:F1}GB remaining"));
    }
}

// Graceful shutdown
builder.Services.AddHostedService<GracefulShutdownService>();

public class GracefulShutdownService(
    ILogger<GracefulShutdownService> logger,
    IHostApplicationLifetime lifetime)
    : IHostedService
{
    public Task StartAsync(CancellationToken ct)
    {
        lifetime.ApplicationStopping.Register(() =>
        {
            logger.LogInformation("Application stopping, completing in-flight requests...");
        });
        
        lifetime.ApplicationStopped.Register(() =>
        {
            logger.LogInformation("Application stopped cleanly");
        });
        
        return Task.CompletedTask;
    }
    
    public Task StopAsync(CancellationToken ct) => Task.CompletedTask;
}

// Configure graceful shutdown timeout
builder.Host.ConfigureHostOptions(opts =>
{
    opts.ShutdownTimeout = TimeSpan.FromSeconds(60);
});
```

---

## Step 866-880: สรุป Container & Kubernetes

### Docker Best Practices
1. **Multi-stage builds** - ลด image size
2. **Non-root user** - เพิ่ม security
3. **Layer caching** - COPY project file ก่อน restore
4. **.dockerignore** - ไม่ copy ไฟล์ที่ไม่จำเป็น
5. **Health checks** - ให้ orchestrator รู้สถานะ

### Kubernetes Best Practices
1. **Resource limits** - กำหนด CPU/Memory ทุก pod
2. **Liveness/Readiness probes** - ตรวจสถานะ
3. **PodDisruptionBudget** - maintain availability ระหว่าง updates
4. **NetworkPolicies** - จำกัด traffic
5. **RBAC** - principle of least privilege
6. **Namespace** - แยก environments

### Deployment Strategies

| Strategy | Downtime | Risk | Rollback |
|----------|----------|------|----------|
| Recreate | Yes | High | Slow |
| Rolling Update | No | Medium | Automatic |
| Blue-Green | No | Low | Instant |
| Canary | No | Very Low | Easy |
| A/B Testing | No | Managed | Flexible |

### Production Checklist
- [ ] Docker image scanned for vulnerabilities
- [ ] Non-root container user
- [ ] Resource requests/limits set
- [ ] Liveness/Readiness probes configured
- [ ] PodDisruptionBudget set
- [ ] HPA configured
- [ ] Network policies applied
- [ ] Secrets from secret manager (not hardcoded)
- [ ] Health check endpoints working
- [ ] Graceful shutdown implemented
- [ ] Observability (metrics, logs, traces) configured
- [ ] CI/CD pipeline with automated tests

---

*จบ Part 31: Docker, Containers & Kubernetes*
*ต่อไป Part 32: Performance Optimization & Profiling*
