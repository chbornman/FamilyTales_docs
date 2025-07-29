# FamilyTales Scaling Strategies

## Overview

FamilyTales scaling strategy is designed to handle growth from 1,000 beta users to 100,000+ active families while maintaining cost efficiency and performance. Our architecture supports both vertical and horizontal scaling with a focus on the unique challenges of OCR processing, audio generation, and global content distribution.

## Current Architecture Scaling Points

### Rust API Services - Horizontal Scaling

#### Kubernetes Deployment
```yaml
# k8s/familytales-api-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: familytales-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: familytales-api
  template:
    metadata:
      labels:
        app: familytales-api
    spec:
      containers:
      - name: api
        image: familytales/api:latest
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        env:
        - name: RUST_LOG
          value: "info"
        - name: DATABASE_POOL_SIZE
          value: "20"
        ports:
        - containerPort: 8080
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: familytales-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: familytales-api
  minReplicas: 3
  maxReplicas: 50
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
        averageValue: "1000"
```

#### Load Balancing Configuration
```nginx
# nginx/load-balancer.conf
upstream familytales_api {
    least_conn;
    
    server api-1.familytales.internal:8080 weight=10 max_fails=3 fail_timeout=30s;
    server api-2.familytales.internal:8080 weight=10 max_fails=3 fail_timeout=30s;
    server api-3.familytales.internal:8080 weight=10 max_fails=3 fail_timeout=30s;
    
    # Health check
    check interval=5000 rise=2 fall=3 timeout=2000 type=http;
    check_http_send "HEAD /health HTTP/1.1\r\nHost: api.familytales.app\r\n\r\n";
    check_http_expect_alive http_2xx http_3xx;
}

server {
    listen 443 ssl http2;
    server_name api.familytales.app;
    
    location / {
        proxy_pass http://familytales_api;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Connection pooling
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        
        # Timeouts for long-running operations
        proxy_connect_timeout 10s;
        proxy_send_timeout 120s;
        proxy_read_timeout 120s;
    }
}
```

### Database Read Replicas

#### PostgreSQL Read Replica Setup
```sql
-- Primary database configuration
ALTER SYSTEM SET wal_level = replica;
ALTER SYSTEM SET max_wal_senders = 10;
ALTER SYSTEM SET wal_keep_segments = 64;
ALTER SYSTEM SET hot_standby = on;

-- Create replication user
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'secure_password';
GRANT CONNECT ON DATABASE familytales_production TO replicator;
```

#### Application-Level Read/Write Splitting
```rust
// src/database/connection_pool.rs
use sqlx::{Pool, Postgres};
use std::sync::Arc;

pub struct DatabasePool {
    write_pool: Arc<Pool<Postgres>>,
    read_pools: Vec<Arc<Pool<Postgres>>>,
    current_read_index: AtomicUsize,
}

impl DatabasePool {
    pub async fn new(config: &DatabaseConfig) -> Result<Self> {
        // Create write pool (primary)
        let write_pool = PgPoolOptions::new()
            .max_connections(config.write_pool_size)
            .connect(&config.primary_url)
            .await?;
        
        // Create read pools (replicas)
        let mut read_pools = Vec::new();
        for replica_url in &config.replica_urls {
            let read_pool = PgPoolOptions::new()
                .max_connections(config.read_pool_size)
                .connect(replica_url)
                .await?;
            read_pools.push(Arc::new(read_pool));
        }
        
        Ok(Self {
            write_pool: Arc::new(write_pool),
            read_pools,
            current_read_index: AtomicUsize::new(0),
        })
    }
    
    pub fn write_pool(&self) -> &Pool<Postgres> {
        &self.write_pool
    }
    
    pub fn read_pool(&self) -> &Pool<Postgres> {
        // Round-robin load balancing across read replicas
        let index = self.current_read_index.fetch_add(1, Ordering::Relaxed);
        let pool_index = index % self.read_pools.len();
        &self.read_pools[pool_index]
    }
}

// Usage in queries
impl FamilyRepository {
    pub async fn get_family(&self, family_id: Uuid) -> Result<Family> {
        // Read operations use read replica
        sqlx::query_as!(
            Family,
            "SELECT * FROM families WHERE id = $1",
            family_id
        )
        .fetch_one(self.db.read_pool())
        .await
    }
    
    pub async fn update_family(&self, family: &Family) -> Result<()> {
        // Write operations use primary
        sqlx::query!(
            "UPDATE families SET name = $1, updated_at = NOW() WHERE id = $2",
            family.name,
            family.id
        )
        .execute(self.db.write_pool())
        .await?;
        Ok(())
    }
}
```

### Mux CDN Utilization

#### Optimizing Mux Delivery
```rust
// src/services/mux_service.rs
use mux_rust_sdk::{MuxClient, CreateAssetRequest, PlaybackPolicy};

pub struct MuxService {
    client: MuxClient,
    cdn_optimization: CdnSettings,
}

impl MuxService {
    pub async fn upload_optimized_audio(&self, audio_data: Vec<u8>) -> Result<MuxAsset> {
        // Create asset with CDN optimization
        let asset = self.client.create_asset(CreateAssetRequest {
            input: audio_data,
            playback_policy: vec![PlaybackPolicy::Public],
            mp4_support: "standard", // Enable MP4 for better compatibility
            normalize_audio: true,    // Consistent volume across content
            master_access: "none",    // Security: no master file access
            
            // CDN optimization settings
            passthrough: json!({
                "cdn_settings": {
                    "cache_control": "public, max-age=31536000", // 1 year
                    "origin_shield": true, // Enable origin shield
                    "geo_blocking": false,
                }
            }).to_string(),
        }).await?;
        
        // Pre-warm CDN cache in family member regions
        self.prewarm_cdn_cache(&asset).await?;
        
        Ok(asset)
    }
    
    async fn prewarm_cdn_cache(&self, asset: &MuxAsset) -> Result<()> {
        // Get family member locations from analytics
        let regions = self.get_family_member_regions(asset.passthrough.family_id).await?;
        
        // Trigger cache warming in each region
        for region in regions {
            let edge_url = format!(
                "https://{}.mux.com/{}/playlist.m3u8",
                region.edge_server,
                asset.playback_id
            );
            
            // Simple HEAD request to trigger caching
            reqwest::Client::new()
                .head(&edge_url)
                .send()
                .await?;
        }
        
        Ok(())
    }
}
```

### Job Queue Scaling with RabbitMQ

#### RabbitMQ Cluster Configuration
```yaml
# docker-compose-rabbitmq.yml
version: '3.8'
services:
  rabbitmq1:
    image: rabbitmq:3.12-management
    hostname: rabbitmq1
    environment:
      RABBITMQ_ERLANG_COOKIE: 'secret_cookie'
      RABBITMQ_DEFAULT_USER: familytales
      RABBITMQ_DEFAULT_PASS: secure_password
    volumes:
      - rabbitmq1_data:/var/lib/rabbitmq
    ports:
      - "5672:5672"
      - "15672:15672"
  
  rabbitmq2:
    image: rabbitmq:3.12-management
    hostname: rabbitmq2
    environment:
      RABBITMQ_ERLANG_COOKIE: 'secret_cookie'
    volumes:
      - rabbitmq2_data:/var/lib/rabbitmq
    depends_on:
      - rabbitmq1
    command: |
      bash -c "sleep 10 && 
               rabbitmqctl stop_app && 
               rabbitmqctl reset && 
               rabbitmqctl join_cluster rabbit@rabbitmq1 && 
               rabbitmqctl start_app"
  
  rabbitmq3:
    image: rabbitmq:3.12-management
    hostname: rabbitmq3
    environment:
      RABBITMQ_ERLANG_COOKIE: 'secret_cookie'
    volumes:
      - rabbitmq3_data:/var/lib/rabbitmq
    depends_on:
      - rabbitmq1
    command: |
      bash -c "sleep 10 && 
               rabbitmqctl stop_app && 
               rabbitmqctl reset && 
               rabbitmqctl join_cluster rabbit@rabbitmq1 && 
               rabbitmqctl start_app"

volumes:
  rabbitmq1_data:
  rabbitmq2_data:
  rabbitmq3_data:
```

#### Dynamic Worker Scaling
```rust
// src/workers/autoscaler.rs
use lapin::{Connection, Channel};
use tokio::time::{interval, Duration};

pub struct WorkerAutoscaler {
    rabbitmq_conn: Connection,
    min_workers: usize,
    max_workers: usize,
    current_workers: Arc<Mutex<Vec<WorkerHandle>>>,
}

impl WorkerAutoscaler {
    pub async fn start_autoscaling(&self) {
        let mut interval = interval(Duration::from_secs(30));
        
        loop {
            interval.tick().await;
            
            let queue_depth = self.get_queue_depth().await.unwrap_or(0);
            let current_count = self.current_workers.lock().await.len();
            
            let desired_workers = self.calculate_desired_workers(queue_depth);
            
            if desired_workers > current_count {
                self.scale_up(desired_workers - current_count).await;
            } else if desired_workers < current_count {
                self.scale_down(current_count - desired_workers).await;
            }
        }
    }
    
    fn calculate_desired_workers(&self, queue_depth: u32) -> usize {
        // Scale based on queue depth
        let workers_needed = (queue_depth as f64 / 100.0).ceil() as usize;
        workers_needed.clamp(self.min_workers, self.max_workers)
    }
    
    async fn scale_up(&self, count: usize) {
        let mut workers = self.current_workers.lock().await;
        
        for _ in 0..count {
            let worker = spawn_worker(self.rabbitmq_conn.clone()).await;
            workers.push(worker);
        }
        
        info!("Scaled up to {} workers", workers.len());
    }
    
    async fn scale_down(&self, count: usize) {
        let mut workers = self.current_workers.lock().await;
        
        for _ in 0..count {
            if let Some(worker) = workers.pop() {
                worker.graceful_shutdown().await;
            }
        }
        
        info!("Scaled down to {} workers", workers.len());
    }
}
```

## Cost Optimization Strategies

### Tiered Processing Architecture

```rust
// src/processing/tiered_processor.rs
pub struct TieredProcessor {
    free_tier_processor: CloudProcessor,
    premium_tier_processor: LocalProcessor,
    usage_tracker: UsageTracker,
}

impl TieredProcessor {
    pub async fn process_document(&self, request: ProcessRequest) -> Result<ProcessResult> {
        let user = self.get_user(request.user_id).await?;
        
        match user.subscription_tier {
            SubscriptionTier::FreeTrial { .. } => {
                // Use cloud APIs (higher cost per document)
                self.free_tier_processor.process(request).await
            }
            SubscriptionTier::FamilyPlan | SubscriptionTier::FamilyLegacy => {
                // Use local PC processing (lower cost)
                self.premium_tier_processor.process(request).await
            }
        }
    }
}

// Cost tracking
impl UsageTracker {
    pub async fn track_processing_cost(&self, result: &ProcessResult) {
        let cost = match result.processing_method {
            ProcessingMethod::GoogleVision => 0.0015, // $1.50 per 1000
            ProcessingMethod::AmazonTextract => 0.0015,
            ProcessingMethod::LocalOCR => 0.0001, // Electricity cost only
        };
        
        self.record_cost(result.user_id, cost, "ocr_processing").await;
    }
}
```

### Storage Optimization

```rust
// src/storage/lifecycle_manager.rs
pub struct StorageLifecycleManager {
    s3_client: aws_sdk_s3::Client,
    db: DatabasePool,
}

impl StorageLifecycleManager {
    pub async fn optimize_storage(&self) -> Result<()> {
        // Move old audio to cheaper storage
        self.archive_old_audio().await?;
        
        // Compress images after processing
        self.compress_processed_images().await?;
        
        // Clean up orphaned files
        self.cleanup_orphaned_files().await?;
        
        Ok(())
    }
    
    async fn archive_old_audio(&self) -> Result<()> {
        // Find audio files not accessed in 90 days
        let old_audio = sqlx::query!(
            r#"
            SELECT cs.id, cs.original_url, cs.thread_id
            FROM content_segments cs
            JOIN threads t ON cs.thread_id = t.id
            WHERE t.last_accessed_at < NOW() - INTERVAL '90 days'
            AND cs.storage_class = 'STANDARD'
            "#
        )
        .fetch_all(self.db.read_pool())
        .await?;
        
        for segment in old_audio {
            // Move to Glacier storage
            self.s3_client.copy_object()
                .copy_source(&segment.original_url)
                .bucket("familytales-archive")
                .key(&format!("audio/{}", segment.id))
                .storage_class(StorageClass::GlacierInstantRetrieval)
                .send()
                .await?;
            
            // Update database
            sqlx::query!(
                "UPDATE content_segments SET storage_class = 'GLACIER' WHERE id = $1",
                segment.id
            )
            .execute(self.db.write_pool())
            .await?;
        }
        
        Ok(())
    }
}
```

### CDN Cost Optimization

```yaml
# cloudfront-distribution.yaml
AWSTemplateFormatVersion: '2010-09-09'
Resources:
  FamilyTalesCDN:
    Type: AWS::CloudFront::Distribution
    Properties:
      DistributionConfig:
        Origins:
          - Id: MuxOrigin
            DomainName: stream.mux.com
            CustomOriginConfig:
              OriginProtocolPolicy: https-only
        
        DefaultCacheBehavior:
          TargetOriginId: MuxOrigin
          ViewerProtocolPolicy: redirect-to-https
          CachePolicyId: !Ref OptimizedCachePolicy
          
        PriceClass: PriceClass_100  # Use only cheapest edge locations
        
  OptimizedCachePolicy:
    Type: AWS::CloudFront::CachePolicy
    Properties:
      CachePolicyConfig:
        DefaultTTL: 86400      # 1 day
        MaxTTL: 31536000      # 1 year
        MinTTL: 1
        ParametersInCacheKeyAndForwardedToOrigin:
          EnableAcceptEncodingGzip: true
          EnableAcceptEncodingBrotli: true
```

## Load Testing Procedures

### Artillery Load Test Configuration

```yaml
# load-tests/familytales-load-test.yml
config:
  target: "https://api.familytales.app"
  phases:
    - duration: 60
      arrivalRate: 10
      name: "Warm up"
    - duration: 300
      arrivalRate: 50
      name: "Ramp up"
    - duration: 600
      arrivalRate: 100
      name: "Sustained load"
    - duration: 120
      arrivalRate: 200
      name: "Spike test"
  
  processor: "./load-test-processor.js"
  
scenarios:
  - name: "Family User Journey"
    weight: 70
    flow:
      - post:
          url: "/api/v1/auth/login"
          json:
            email: "{{ $randomEmail() }}"
            password: "{{ $randomPassword() }}"
          capture:
            - json: "$.token"
              as: "authToken"
      
      - post:
          url: "/api/v1/documents"
          headers:
            Authorization: "Bearer {{ authToken }}"
          formData:
            image: "{{ $randomImage() }}"
            metadata: '{"type": "letter"}'
          capture:
            - json: "$.document_id"
              as: "documentId"
      
      - loop:
          count: 10
          actions:
            - get:
                url: "/api/v1/documents/{{ documentId }}/status"
                headers:
                  Authorization: "Bearer {{ authToken }}"
            - think: 2
      
      - post:
          url: "/api/v1/documents/{{ documentId }}/generate-audio"
          headers:
            Authorization: "Bearer {{ authToken }}"
          json:
            voice_id: "en-US-Wavenet-F"
  
  - name: "Audio Streaming"
    weight: 30
    flow:
      - get:
          url: "/api/v1/stream/{{ $randomDocumentId() }}/playlist.m3u8"
          capture:
            - regexp: "(.+\.ts)"
              as: "segmentUrl"
      
      - loop:
          count: 5
          actions:
            - get:
                url: "/api/v1/stream/{{ $randomDocumentId() }}/{{ segmentUrl }}"
            - think: 1
```

### Load Test Analysis

```rust
// src/testing/load_test_analyzer.rs
pub struct LoadTestAnalyzer {
    results_path: PathBuf,
}

impl LoadTestAnalyzer {
    pub fn analyze_results(&self) -> LoadTestReport {
        let raw_results = self.parse_artillery_results();
        
        LoadTestReport {
            summary: TestSummary {
                total_requests: raw_results.aggregate.counters.requests,
                success_rate: self.calculate_success_rate(&raw_results),
                avg_latency: raw_results.aggregate.latency.mean,
                p95_latency: raw_results.aggregate.latency.p95,
                p99_latency: raw_results.aggregate.latency.p99,
                requests_per_second: raw_results.aggregate.rates.mean,
            },
            
            bottlenecks: self.identify_bottlenecks(&raw_results),
            
            scaling_recommendations: self.generate_recommendations(&raw_results),
            
            cost_projection: self.project_costs_at_scale(&raw_results),
        }
    }
    
    fn identify_bottlenecks(&self, results: &ArtilleryResults) -> Vec<Bottleneck> {
        let mut bottlenecks = Vec::new();
        
        // Check for database bottlenecks
        if results.custom_metrics.db_connection_wait_time > 100.0 {
            bottlenecks.push(Bottleneck {
                component: "Database Connection Pool",
                impact: "High latency on read operations",
                recommendation: "Increase connection pool size or add read replicas",
            });
        }
        
        // Check for OCR processing bottlenecks
        if results.scenarios["ocr_processing"].p95 > 30000.0 {
            bottlenecks.push(Bottleneck {
                component: "OCR Processing",
                impact: "Slow document processing",
                recommendation: "Scale up worker nodes or optimize OCR pipeline",
            });
        }
        
        bottlenecks
    }
}
```

## Auto-Scaling Configuration

### Kubernetes Cluster Autoscaler

```yaml
# k8s/cluster-autoscaler.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  template:
    spec:
      containers:
      - image: k8s.gcr.io/autoscaling/cluster-autoscaler:v1.21.0
        name: cluster-autoscaler
        command:
        - ./cluster-autoscaler
        - --v=4
        - --stderrthreshold=info
        - --cloud-provider=aws
        - --skip-nodes-with-local-storage=false
        - --expander=least-waste
        - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/familytales
        - --balance-similar-node-groups
        - --skip-nodes-with-system-pods=false
        env:
        - name: AWS_REGION
          value: us-east-1
```

### Application-Level Auto-Scaling

```rust
// src/autoscaling/predictor.rs
use prophet::{Prophet, PredictionRequest};

pub struct ScalingPredictor {
    prophet_model: Prophet,
    metrics_store: MetricsStore,
}

impl ScalingPredictor {
    pub async fn predict_scaling_needs(&self) -> ScalingPrediction {
        // Get historical metrics
        let historical_data = self.metrics_store
            .get_metrics("requests_per_minute", Duration::days(30))
            .await
            .unwrap();
        
        // Prepare data for Prophet
        let prophet_data = historical_data.into_iter()
            .map(|point| (point.timestamp, point.value))
            .collect();
        
        // Make prediction for next 24 hours
        let prediction = self.prophet_model.predict(PredictionRequest {
            historical_data: prophet_data,
            periods: 24 * 60, // 24 hours in minutes
            frequency: "T",   // Per minute
        });
        
        // Calculate required capacity
        let peak_load = prediction.yhat.iter().max_by(|a, b| a.partial_cmp(b).unwrap()).unwrap();
        let required_instances = (*peak_load / 1000.0).ceil() as usize; // 1000 req/min per instance
        
        ScalingPrediction {
            timestamp: Utc::now(),
            predicted_peak: *peak_load,
            recommended_instances: required_instances,
            confidence_interval: prediction.confidence_interval,
        }
    }
}
```

## Geographic Distribution Strategy

### Multi-Region Deployment

```terraform
# terraform/multi-region.tf
variable "regions" {
  default = ["us-east-1", "eu-west-1", "ap-southeast-1"]
}

module "familytales_deployment" {
  for_each = toset(var.regions)
  source   = "./modules/regional-deployment"
  
  region = each.value
  
  api_instance_count = var.instance_counts[each.value]
  db_read_replicas   = var.read_replica_counts[each.value]
  
  # Regional configuration
  enable_cross_region_replication = true
  primary_region = "us-east-1"
}

# Global load balancer
resource "aws_route53_record" "api" {
  zone_id = aws_route53_zone.familytales.zone_id
  name    = "api.familytales.app"
  type    = "A"
  
  set_identifier = "latency-routing"
  
  alias {
    name                   = module.familytales_deployment["us-east-1"].alb_dns_name
    zone_id                = module.familytales_deployment["us-east-1"].alb_zone_id
    evaluate_target_health = true
  }
  
  latency_routing_policy {
    region = "us-east-1"
  }
}
```

## Capacity Planning

### Growth Projections and Resource Requirements

```python
# capacity_planning.py
import pandas as pd
import numpy as np
from datetime import datetime, timedelta

class CapacityPlanner:
    def __init__(self):
        self.user_growth_rate = 0.15  # 15% monthly growth
        self.docs_per_user_month = 10
        self.avg_doc_size_mb = 2.5
        self.audio_compression_ratio = 0.1  # Audio is 10% of image size
        
    def project_capacity_needs(self, months=12):
        projections = []
        current_users = 1000  # Starting point
        
        for month in range(months):
            users = current_users * (1 + self.user_growth_rate) ** month
            
            # Calculate resource needs
            monthly_docs = users * self.docs_per_user_month
            storage_gb = (monthly_docs * self.avg_doc_size_mb * 
                         (1 + self.audio_compression_ratio)) / 1024
            
            # API capacity (requests per second)
            peak_rps = users * 0.001  # 0.1% of users active simultaneously
            
            # Database connections
            db_connections = min(users * 0.01, 1000)  # 1% active, max 1000
            
            # Worker nodes for processing
            worker_nodes = max(1, monthly_docs / (30 * 24 * 60 * 10))  # 10 docs/min per worker
            
            projections.append({
                'month': month,
                'users': int(users),
                'monthly_documents': int(monthly_docs),
                'storage_gb': round(storage_gb, 2),
                'peak_rps': round(peak_rps, 2),
                'db_connections': int(db_connections),
                'worker_nodes': int(worker_nodes),
                'estimated_cost_usd': self.calculate_cost(
                    storage_gb, peak_rps, worker_nodes
                )
            })
        
        return pd.DataFrame(projections)
    
    def calculate_cost(self, storage_gb, peak_rps, worker_nodes):
        # Simplified cost model
        storage_cost = storage_gb * 0.023  # S3 standard
        compute_cost = (peak_rps / 100) * 50  # EC2 instances
        worker_cost = worker_nodes * 100  # Local PC costs
        
        return round(storage_cost + compute_cost + worker_cost, 2)
```

## Performance Optimization at Scale

### Caching Strategy

```rust
// src/caching/multi_tier_cache.rs
pub struct MultiTierCache {
    local_cache: Arc<Mutex<lru::LruCache<String, CachedItem>>>,
    redis_cache: RedisCache,
    cdn_cache: CdnCache,
}

impl MultiTierCache {
    pub async fn get_with_fallback<T, F>(&self, key: &str, fetch_fn: F) -> Result<T>
    where
        T: Serialize + DeserializeOwned + Clone,
        F: Future<Output = Result<T>>,
    {
        // Check local cache first (L1)
        if let Some(cached) = self.local_cache.lock().await.get(key) {
            if !cached.is_expired() {
                return Ok(cached.data.clone());
            }
        }
        
        // Check Redis cache (L2)
        if let Ok(Some(data)) = self.redis_cache.get::<T>(key).await {
            self.local_cache.lock().await.put(key.to_string(), CachedItem::new(data.clone()));
            return Ok(data);
        }
        
        // Check CDN cache (L3)
        if let Ok(Some(data)) = self.cdn_cache.get::<T>(key).await {
            self.redis_cache.set(key, &data, Duration::hours(1)).await?;
            self.local_cache.lock().await.put(key.to_string(), CachedItem::new(data.clone()));
            return Ok(data);
        }
        
        // Fetch from source
        let data = fetch_fn.await?;
        
        // Populate all cache tiers
        self.cdn_cache.set(key, &data, Duration::days(7)).await?;
        self.redis_cache.set(key, &data, Duration::hours(24)).await?;
        self.local_cache.lock().await.put(key.to_string(), CachedItem::new(data.clone()));
        
        Ok(data)
    }
}
```

## Scaling Milestones and Triggers

| Users | Infrastructure Changes | Estimated Cost |
|-------|------------------------|----------------|
| 1K-10K | Single region, 3 API servers, 1 DB | $500/month |
| 10K-50K | Add read replicas, CDN, 10 API servers | $2,500/month |
| 50K-100K | Multi-region, 30 API servers, sharded DB | $8,000/month |
| 100K+ | Global distribution, 100+ servers, dedicated clusters | $25,000+/month |

## Monitoring Scaling Effectiveness

```rust
// src/monitoring/scaling_metrics.rs
pub struct ScalingMetrics {
    prometheus: PrometheusClient,
}

impl ScalingMetrics {
    pub async fn report_scaling_effectiveness(&self) {
        // Resource utilization
        self.prometheus.gauge("cpu_utilization_percent", self.get_cpu_usage());
        self.prometheus.gauge("memory_utilization_percent", self.get_memory_usage());
        
        // Scaling events
        self.prometheus.counter("autoscale_up_events", 1);
        self.prometheus.counter("autoscale_down_events", 1);
        
        // Cost efficiency
        let cost_per_request = self.calculate_cost_per_request().await;
        self.prometheus.gauge("cost_per_request_cents", cost_per_request);
        
        // Performance under load
        self.prometheus.histogram("response_time_under_load", response_time);
        self.prometheus.gauge("queue_depth", queue_size);
    }
}
```