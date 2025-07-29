# FamilyTales Deployment Guide

## Overview

This guide covers the complete deployment process for FamilyTales, from local development to production deployment on Kubernetes with full CI/CD automation.

## Table of Contents

1. [Infrastructure Requirements](#infrastructure-requirements)
2. [Environment Configuration](#environment-configuration)
3. [Docker Containerization](#docker-containerization)
4. [Kubernetes Deployment](#kubernetes-deployment)
5. [CI/CD Pipeline](#cicd-pipeline)
6. [Monitoring Setup](#monitoring-setup)
7. [Security Hardening](#security-hardening)
8. [Troubleshooting](#troubleshooting)

## Infrastructure Requirements

### Minimum Production Requirements

#### API Servers (3 nodes minimum)
- **CPU**: 4 vCPUs per node
- **Memory**: 8GB RAM per node
- **Storage**: 50GB SSD
- **Network**: 1Gbps

#### Database Server
- **CPU**: 8 vCPUs
- **Memory**: 32GB RAM
- **Storage**: 500GB SSD (NVMe preferred)
- **Backup**: Daily snapshots

#### Redis Cluster
- **CPU**: 2 vCPUs per node
- **Memory**: 8GB RAM per node
- **Nodes**: 3 (1 master, 2 replicas)

#### Worker Nodes (Auto-scaling)
- **CPU**: 2 vCPUs per node
- **Memory**: 4GB RAM per node
- **Scaling**: 2-20 nodes based on queue depth

### Cloud Provider Recommendations

#### AWS
```yaml
# terraform/aws/main.tf
resource "aws_eks_cluster" "familytales" {
  name     = "familytales-production"
  role_arn = aws_iam_role.eks_cluster.arn

  vpc_config {
    subnet_ids = aws_subnet.private[*].id
    security_group_ids = [aws_security_group.eks.id]
  }

  enabled_cluster_log_types = ["api", "audit", "authenticator"]
}

resource "aws_rds_cluster" "postgres" {
  cluster_identifier      = "familytales-db"
  engine                  = "aurora-postgresql"
  engine_version          = "16.1"
  master_username         = "familytales"
  master_password         = var.db_password
  backup_retention_period = 30
  preferred_backup_window = "03:00-04:00"
}
```

#### Google Cloud
```yaml
# terraform/gcp/main.tf
resource "google_container_cluster" "familytales" {
  name     = "familytales-production"
  location = "us-central1"
  
  node_pool {
    name       = "default-pool"
    node_count = 3
    
    node_config {
      machine_type = "n1-standard-4"
      disk_size_gb = 100
    }
  }
}
```

## Environment Configuration

### Environment Variables

```bash
# .env.production
# Database
DATABASE_URL=postgresql://user:pass@host:5432/familytales
DATABASE_POOL_SIZE=25
DATABASE_TIMEOUT=30s

# Redis
REDIS_URL=redis://redis-cluster:6379
REDIS_POOL_SIZE=10

# Authentication (Clerk)
CLERK_PUBLISHABLE_KEY=pk_live_xxx
CLERK_SECRET_KEY=sk_live_xxx

# Media Storage (Mux)
MUX_TOKEN_ID=xxx
MUX_TOKEN_SECRET=xxx
MUX_WEBHOOK_SECRET=xxx

# Email (SendGrid)
SENDGRID_API_KEY=SG.xxx
SENDGRID_FROM_EMAIL=hello@familytales.app
SENDGRID_TEMPLATE_WELCOME=d-xxx
SENDGRID_TEMPLATE_INVITE=d-xxx

# Payment (Stripe)
STRIPE_PUBLISHABLE_KEY=pk_live_xxx
STRIPE_SECRET_KEY=sk_live_xxx
STRIPE_WEBHOOK_SECRET=whsec_xxx

# OCR Service
GOOGLE_VISION_API_KEY=xxx
OCR_TIMEOUT=60s

# TTS Service
GOOGLE_CLOUD_TTS_KEY=xxx
TTS_DEFAULT_VOICE=en-US-Standard-D
TTS_TIMEOUT=120s

# Application
RUST_LOG=info
API_PORT=3000
WORKER_CONCURRENCY=10
CORS_ALLOWED_ORIGINS=https://app.familytales.app,https://familytales.app

# Monitoring
PROMETHEUS_PORT=9090
JAEGER_ENDPOINT=http://jaeger:14268/api/traces
```

### Secrets Management

```yaml
# kubernetes/secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: familytales-secrets
type: Opaque
stringData:
  database-url: "postgresql://..."
  clerk-secret: "sk_live_..."
  stripe-secret: "sk_live_..."
  mux-secret: "..."
```

## Docker Containerization

### API Server Dockerfile

```dockerfile
# docker/api/Dockerfile
FROM rust:1.75-slim AS builder

WORKDIR /app
COPY Cargo.toml Cargo.lock ./
COPY src ./src

# Build dependencies separately for caching
RUN cargo build --release --bin api

# Runtime stage
FROM debian:bookworm-slim

RUN apt-get update && apt-get install -y \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

COPY --from=builder /app/target/release/api /usr/local/bin/api

EXPOSE 3000
CMD ["api"]
```

### Worker Dockerfile

```dockerfile
# docker/worker/Dockerfile
FROM rust:1.75-slim AS builder

WORKDIR /app
COPY Cargo.toml Cargo.lock ./
COPY src ./src

RUN cargo build --release --bin worker

# Runtime stage
FROM debian:bookworm-slim

RUN apt-get update && apt-get install -y \
    ca-certificates \
    imagemagick \
    ffmpeg \
    && rm -rf /var/lib/apt/lists/*

COPY --from=builder /app/target/release/worker /usr/local/bin/worker

CMD ["worker"]
```

### Flutter Web Dockerfile

```dockerfile
# docker/web/Dockerfile
FROM dart:stable AS build

WORKDIR /app
COPY pubspec.* ./
RUN dart pub get

COPY . .
RUN dart pub get --offline
RUN dart run build_runner build --delete-conflicting-outputs
RUN flutter build web --release --web-renderer canvaskit

# Serve with nginx
FROM nginx:alpine

COPY --from=build /app/build/web /usr/share/nginx/html
COPY docker/web/nginx.conf /etc/nginx/nginx.conf

EXPOSE 80
```

### Docker Compose (Development)

```yaml
# docker-compose.yml
version: '3.8'

services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: familytales
      POSTGRES_USER: familytales
      POSTGRES_PASSWORD: localpass
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "5672:5672"
      - "15672:15672"

  api:
    build:
      context: .
      dockerfile: docker/api/Dockerfile
    environment:
      DATABASE_URL: postgresql://familytales:localpass@postgres/familytales
      REDIS_URL: redis://redis:6379
    ports:
      - "3000:3000"
    depends_on:
      - postgres
      - redis

  worker:
    build:
      context: .
      dockerfile: docker/worker/Dockerfile
    environment:
      DATABASE_URL: postgresql://familytales:localpass@postgres/familytales
      REDIS_URL: redis://redis:6379
      RABBITMQ_URL: amqp://guest:guest@rabbitmq:5672
    depends_on:
      - postgres
      - redis
      - rabbitmq

volumes:
  postgres_data:
```

## Kubernetes Deployment

### Namespace and ConfigMap

```yaml
# kubernetes/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: familytales

---
# kubernetes/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: familytales-config
  namespace: familytales
data:
  API_PORT: "3000"
  RUST_LOG: "info"
  CORS_ALLOWED_ORIGINS: "https://app.familytales.app"
```

### API Deployment

```yaml
# kubernetes/deployments/api.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: familytales-api
  namespace: familytales
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: familytales/api:latest
        ports:
        - containerPort: 3000
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: familytales-secrets
              key: database-url
        - name: REDIS_URL
          value: redis://redis-service:6379
        envFrom:
        - configMapRef:
            name: familytales-config
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
```

### Worker Deployment

```yaml
# kubernetes/deployments/worker.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: familytales-worker
  namespace: familytales
spec:
  replicas: 2
  selector:
    matchLabels:
      app: worker
  template:
    metadata:
      labels:
        app: worker
    spec:
      containers:
      - name: worker
        image: familytales/worker:latest
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: familytales-secrets
              key: database-url
        - name: WORKER_CONCURRENCY
          value: "10"
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
```

### Services

```yaml
# kubernetes/services/api.yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: familytales
spec:
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 3000
  type: ClusterIP

---
# kubernetes/services/redis.yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-service
  namespace: familytales
spec:
  selector:
    app: redis
  ports:
  - port: 6379
  type: ClusterIP
```

### Ingress Configuration

```yaml
# kubernetes/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: familytales-ingress
  namespace: familytales
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rate-limit: "100"
spec:
  tls:
  - hosts:
    - api.familytales.app
    secretName: familytales-tls
  rules:
  - host: api.familytales.app
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

### Horizontal Pod Autoscaler

```yaml
# kubernetes/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
  namespace: familytales
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: familytales-api
  minReplicas: 3
  maxReplicas: 10
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

## CI/CD Pipeline

### GitHub Actions Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

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
    - uses: actions/checkout@v3
    
    - name: Setup Rust
      uses: actions-rs/toolchain@v1
      with:
        toolchain: stable
        override: true
    
    - name: Run tests
      run: |
        cargo test --all-features
        cargo clippy -- -D warnings
    
    - name: Security audit
      run: cargo audit

  build:
    needs: test
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Log in to Container Registry
      uses: docker/login-action@v2
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    
    - name: Build and push API image
      uses: docker/build-push-action@v4
      with:
        context: .
        file: docker/api/Dockerfile
        push: true
        tags: |
          ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/api:latest
          ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/api:${{ github.sha }}
    
    - name: Build and push Worker image
      uses: docker/build-push-action@v4
      with:
        context: .
        file: docker/worker/Dockerfile
        push: true
        tags: |
          ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/worker:latest
          ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/worker:${{ github.sha }}

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Configure kubectl
      uses: azure/setup-kubectl@v3
      with:
        version: 'latest'
    
    - name: Set up Kubeconfig
      run: |
        echo "${{ secrets.KUBECONFIG }}" | base64 -d > kubeconfig
        export KUBECONFIG=kubeconfig
    
    - name: Deploy to Kubernetes
      run: |
        kubectl set image deployment/familytales-api \
          api=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/api:${{ github.sha }} \
          -n familytales
        
        kubectl set image deployment/familytales-worker \
          worker=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/worker:${{ github.sha }} \
          -n familytales
        
        kubectl rollout status deployment/familytales-api -n familytales
        kubectl rollout status deployment/familytales-worker -n familytales
    
    - name: Run smoke tests
      run: |
        curl -f https://api.familytales.app/health || exit 1
```

### GitOps with ArgoCD

```yaml
# argocd/application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: familytales
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/familytales/infrastructure
    targetRevision: HEAD
    path: kubernetes
  destination:
    server: https://kubernetes.default.svc
    namespace: familytales
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

## Monitoring Setup

### Prometheus Configuration

```yaml
# monitoring/prometheus/config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
    
    scrape_configs:
    - job_name: 'familytales-api'
      kubernetes_sd_configs:
      - role: pod
        namespaces:
          names:
          - familytales
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        action: keep
        regex: api
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod
```

### Grafana Dashboards

```json
{
  "dashboard": {
    "title": "FamilyTales Production",
    "panels": [
      {
        "title": "API Request Rate",
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])"
          }
        ]
      },
      {
        "title": "Processing Queue Depth",
        "targets": [
          {
            "expr": "rabbitmq_queue_messages"
          }
        ]
      },
      {
        "title": "Error Rate",
        "targets": [
          {
            "expr": "rate(http_requests_total{status=~\"5..\"}[5m])"
          }
        ]
      }
    ]
  }
}
```

### Alerting Rules

```yaml
# monitoring/alerts.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: familytales-alerts
spec:
  groups:
  - name: familytales
    rules:
    - alert: HighErrorRate
      expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
      for: 5m
      annotations:
        summary: "High error rate detected"
        
    - alert: ProcessingQueueBacklog
      expr: rabbitmq_queue_messages > 1000
      for: 10m
      annotations:
        summary: "Processing queue backlog detected"
        
    - alert: DatabaseConnectionPoolExhausted
      expr: pg_stat_database_numbackends / pg_settings_max_connections > 0.8
      for: 5m
      annotations:
        summary: "Database connection pool nearly exhausted"
```

## Security Hardening

### Network Policies

```yaml
# kubernetes/network-policies.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-network-policy
  namespace: familytales
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 3000
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
```

### Pod Security Policies

```yaml
# kubernetes/pod-security.yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: familytales-psp
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
  - ALL
  volumes:
  - 'configMap'
  - 'emptyDir'
  - 'projected'
  - 'secret'
  - 'downwardAPI'
  - 'persistentVolumeClaim'
  hostNetwork: false
  hostIPC: false
  hostPID: false
  runAsUser:
    rule: 'MustRunAsNonRoot'
  seLinux:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
```

## Troubleshooting

### Common Issues

#### 1. Pod CrashLoopBackOff
```bash
# Check pod logs
kubectl logs -n familytales deployment/familytales-api

# Describe pod for events
kubectl describe pod -n familytales <pod-name>

# Check resource limits
kubectl top pods -n familytales
```

#### 2. Database Connection Issues
```bash
# Test database connectivity
kubectl run -it --rm debug --image=postgres:16 --restart=Never -- psql -h <db-host> -U familytales

# Check connection pool
kubectl exec -n familytales deployment/familytales-api -- curl localhost:9090/metrics | grep db_connections
```

#### 3. Slow Performance
```bash
# Check CPU/Memory usage
kubectl top nodes
kubectl top pods -n familytales

# Review slow queries
kubectl exec -n familytales postgres-0 -- psql -U familytales -c "SELECT * FROM pg_stat_statements ORDER BY mean_exec_time DESC LIMIT 10;"
```

### Deployment Rollback

```bash
# View deployment history
kubectl rollout history deployment/familytales-api -n familytales

# Rollback to previous version
kubectl rollout undo deployment/familytales-api -n familytales

# Rollback to specific revision
kubectl rollout undo deployment/familytales-api -n familytales --to-revision=3
```

### Emergency Procedures

#### 1. Circuit Breaker Activation
```bash
# Temporarily route traffic away from problematic service
kubectl patch service api-service -n familytales -p '{"spec":{"selector":{"app":"maintenance"}}}'
```

#### 2. Database Emergency Backup
```bash
# Create immediate backup
kubectl exec -n familytales postgres-0 -- pg_dump -U familytales familytales | gzip > emergency-backup-$(date +%Y%m%d-%H%M%S).sql.gz
```

#### 3. Scale to Zero
```bash
# Stop all processing
kubectl scale deployment familytales-worker -n familytales --replicas=0
```

## Maintenance Windows

### Planned Maintenance Procedure

1. **Announce maintenance** (24 hours prior)
2. **Enable maintenance mode**
   ```bash
   kubectl apply -f kubernetes/maintenance-page.yaml
   ```
3. **Drain worker queues**
   ```bash
   kubectl scale deployment familytales-worker --replicas=0
   ```
4. **Perform maintenance**
5. **Run smoke tests**
6. **Restore services**
7. **Monitor for 30 minutes**

## Support Contacts

- **On-call Engineer**: ops@familytales.app
- **Escalation**: cto@familytales.app
- **Status Page**: https://status.familytales.app
- **Incident Response**: Use PagerDuty