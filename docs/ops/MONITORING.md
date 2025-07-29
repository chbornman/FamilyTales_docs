# FamilyTales System Monitoring

## Overview

FamilyTales monitoring strategy focuses on the unique challenges of our service: ensuring high-quality OCR processing, timely audio generation, and seamless family sharing experiences. We monitor both technical metrics and user experience indicators to maintain service quality.

## Key Metrics

### OCR Processing Metrics

#### Success Rate Monitoring
```rust
// src/monitoring/ocr_metrics.rs
use prometheus::{Counter, Histogram, Registry};

pub struct OcrMetrics {
    pub ocr_requests_total: Counter,
    pub ocr_success_total: Counter,
    pub ocr_failures_total: Counter,
    pub ocr_confidence_score: Histogram,
    pub ocr_processing_time: Histogram,
    pub ocr_document_types: CounterVec,
}

impl OcrMetrics {
    pub fn record_ocr_result(&self, result: &OcrResult) {
        self.ocr_requests_total.inc();
        
        match result.status {
            OcrStatus::Success => {
                self.ocr_success_total.inc();
                self.ocr_confidence_score.observe(result.confidence);
                self.ocr_processing_time.observe(result.duration_ms);
                self.ocr_document_types
                    .with_label_values(&[&result.document_type])
                    .inc();
            }
            OcrStatus::Failed => {
                self.ocr_failures_total.inc();
            }
        }
    }
    
    pub fn success_rate(&self) -> f64 {
        let total = self.ocr_requests_total.get();
        let success = self.ocr_success_total.get();
        if total > 0.0 {
            success / total * 100.0
        } else {
            0.0
        }
    }
}
```

#### OCR Quality Tracking
```yaml
# prometheus-rules.yaml
groups:
  - name: ocr_quality
    interval: 30s
    rules:
      - alert: LowOcrSuccessRate
        expr: |
          (sum(rate(ocr_success_total[5m])) / sum(rate(ocr_requests_total[5m]))) < 0.95
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "OCR success rate below 95%"
          description: "Current success rate: {{ $value | humanizePercentage }}"
      
      - alert: LowOcrConfidence
        expr: |
          histogram_quantile(0.5, ocr_confidence_score) < 0.85
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Median OCR confidence below 85%"
          description: "50% of OCR results have confidence below 85%"
```

### Audio Generation Metrics

```rust
// src/monitoring/audio_metrics.rs
pub struct AudioMetrics {
    pub audio_generation_requests: Counter,
    pub audio_generation_time: Histogram,
    pub audio_file_size_bytes: Histogram,
    pub voice_usage: CounterVec,
    pub mux_upload_time: Histogram,
    pub mux_processing_status: CounterVec,
}

impl AudioMetrics {
    pub fn track_audio_generation(&self, request: &AudioRequest, result: &AudioResult) {
        self.audio_generation_requests.inc();
        self.audio_generation_time.observe(result.generation_time_ms);
        self.audio_file_size_bytes.observe(result.file_size_bytes as f64);
        self.voice_usage
            .with_label_values(&[&request.voice_id])
            .inc();
            
        // Track Mux upload performance
        if let Some(mux_time) = result.mux_upload_time_ms {
            self.mux_upload_time.observe(mux_time);
        }
        
        self.mux_processing_status
            .with_label_values(&[&result.mux_status.to_string()])
            .inc();
    }
}
```

### Family Activity Monitoring

```sql
-- Family engagement tracking
CREATE VIEW family_activity_metrics AS
SELECT 
    f.id as family_id,
    f.name as family_name,
    COUNT(DISTINCT fm.user_id) as active_members,
    COUNT(DISTINCT mb.id) as memory_books_created,
    COUNT(DISTINCT cs.id) as documents_processed,
    SUM(t.total_duration) / 60 as total_audio_minutes,
    MAX(mb.created_at) as last_activity,
    f.subscription_tier
FROM families f
LEFT JOIN family_members fm ON f.id = fm.family_id
LEFT JOIN memory_books mb ON f.id = mb.family_id 
    AND mb.created_at > NOW() - INTERVAL '30 days'
LEFT JOIN threads t ON mb.id = t.book_id
LEFT JOIN content_segments cs ON t.id = cs.thread_id
GROUP BY f.id, f.name, f.subscription_tier;

-- Alert on inactive families
CREATE OR REPLACE FUNCTION check_inactive_families()
RETURNS TABLE(family_id UUID, family_name TEXT, days_inactive INT) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        f.id,
        f.name,
        EXTRACT(DAY FROM NOW() - COALESCE(MAX(mb.created_at), f.created_at))::INT
    FROM families f
    LEFT JOIN memory_books mb ON f.id = mb.family_id
    WHERE f.subscription_tier != 'free_trial'
    GROUP BY f.id, f.name
    HAVING EXTRACT(DAY FROM NOW() - COALESCE(MAX(mb.created_at), f.created_at)) > 60;
END;
$$ LANGUAGE plpgsql;
```

## Monitoring Infrastructure

### Uptime Monitoring with Uptime Robot

```yaml
# uptime-robot-monitors.yaml
monitors:
  - name: "FamilyTales API Health"
    url: "https://api.familytales.app/health"
    type: "HTTP"
    interval: 60
    timeout: 30
    alerts:
      - email: "oncall@familytales.app"
      - slack: "#alerts"
    
  - name: "FamilyTales Web App"
    url: "https://app.familytales.app"
    type: "HTTP"
    interval: 300
    keyword: "Welcome to FamilyTales"
    
  - name: "Mux Webhook Endpoint"
    url: "https://api.familytales.app/webhooks/mux"
    type: "PORT"
    port: 443
    
  - name: "Database Connection"
    url: "https://api.familytales.app/health/db"
    type: "HTTP"
    interval: 300
    custom_headers:
      Authorization: "Bearer ${MONITORING_TOKEN}"
```

### Application Performance Monitoring (APM)

#### DataDog Configuration
```yaml
# datadog-agent.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: datadog-agent
spec:
  template:
    spec:
      containers:
      - name: datadog-agent
        image: gcr.io/datadoghq/agent:latest
        env:
        - name: DD_API_KEY
          valueFrom:
            secretKeyRef:
              name: datadog-secret
              key: api-key
        - name: DD_SITE
          value: "datadoghq.com"
        - name: DD_APM_ENABLED
          value: "true"
        - name: DD_LOGS_ENABLED
          value: "true"
        - name: DD_PROCESS_AGENT_ENABLED
          value: "true"
        - name: DD_TAGS
          value: "env:production service:familytales"
```

#### Custom APM Instrumentation
```rust
// src/middleware/apm.rs
use datadog_apm::tracer::{Tracer, Span};

pub struct ApmMiddleware {
    tracer: Tracer,
}

impl ApmMiddleware {
    pub async fn trace_ocr_processing<F, T>(
        &self,
        document_id: Uuid,
        operation: F,
    ) -> Result<T>
    where
        F: Future<Output = Result<T>>,
    {
        let mut span = self.tracer.span("ocr.process");
        span.set_tag("document.id", &document_id.to_string());
        span.set_tag("operation.type", "ocr");
        
        let start = Instant::now();
        let result = operation.await;
        let duration = start.elapsed();
        
        span.set_tag("duration.ms", &duration.as_millis().to_string());
        
        match &result {
            Ok(_) => span.set_tag("status", "success"),
            Err(e) => {
                span.set_tag("status", "error");
                span.set_tag("error.message", &e.to_string());
            }
        }
        
        span.finish();
        result
    }
}
```

## Custom Dashboards

### Elder User Experience Dashboard

```json
{
  "dashboard": {
    "title": "Elder User Experience",
    "panels": [
      {
        "title": "Session Duration by Age Group",
        "query": "SELECT age_group, AVG(session_duration_minutes) FROM user_sessions WHERE age_group IN ('65-74', '75+') GROUP BY age_group",
        "visualization": "bar_chart"
      },
      {
        "title": "Error Rate by User Age",
        "query": "SELECT age_group, COUNT(error_events) / COUNT(DISTINCT user_id) as error_rate FROM user_events WHERE event_type = 'error' GROUP BY age_group",
        "visualization": "line_chart"
      },
      {
        "title": "Most Common Elder User Issues",
        "query": "SELECT error_type, COUNT(*) as occurrences FROM error_events WHERE user_age >= 65 GROUP BY error_type ORDER BY occurrences DESC LIMIT 10",
        "visualization": "table"
      },
      {
        "title": "Voice Playback Success Rate",
        "query": "SELECT DATE(timestamp), COUNT(CASE WHEN status = 'success' THEN 1 END) / COUNT(*) as success_rate FROM audio_playback_events WHERE user_age >= 65 GROUP BY DATE(timestamp)",
        "visualization": "time_series"
      }
    ]
  }
}
```

### OCR Performance Dashboard

```rust
// src/monitoring/dashboards/ocr_dashboard.rs
pub struct OcrDashboard {
    pub panels: Vec<Panel>,
}

impl OcrDashboard {
    pub fn new() -> Self {
        Self {
            panels: vec![
                Panel {
                    title: "OCR Success Rate (24h)",
                    query: r#"
                        sum(rate(ocr_success_total[5m])) / 
                        sum(rate(ocr_requests_total[5m])) * 100
                    "#,
                    visualization: PanelType::Gauge,
                    thresholds: vec![
                        Threshold { value: 95.0, color: "green" },
                        Threshold { value: 90.0, color: "yellow" },
                        Threshold { value: 0.0, color: "red" },
                    ],
                },
                Panel {
                    title: "OCR Processing Time by Document Type",
                    query: r#"
                        histogram_quantile(0.95, 
                            sum by (document_type, le) (
                                rate(ocr_processing_time_bucket[5m])
                            )
                        )
                    "#,
                    visualization: PanelType::BarChart,
                },
                Panel {
                    title: "Low Confidence Documents",
                    query: r#"
                        count(ocr_confidence_score < 0.85)
                    "#,
                    visualization: PanelType::SingleStat,
                },
            ],
        }
    }
}
```

## Alert Thresholds and Escalation

### Alert Configuration

```yaml
# alerting-rules.yaml
groups:
  - name: familytales_critical
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) / 
          sum(rate(http_requests_total[5m])) > 0.05
        for: 5m
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "High error rate detected"
          runbook: "https://wiki.familytales.app/runbooks/high-error-rate"
      
      - alert: DatabaseConnectionFailure
        expr: |
          up{job="postgres"} == 0
        for: 1m
        labels:
          severity: critical
          team: infrastructure
        annotations:
          summary: "Database connection lost"
          runbook: "https://wiki.familytales.app/runbooks/database-failure"
      
      - alert: MuxWebhookFailures
        expr: |
          sum(rate(mux_webhook_failures_total[5m])) > 5
        for: 10m
        labels:
          severity: high
          team: backend
        annotations:
          summary: "Mux webhook processing failures"
  
  - name: familytales_performance
    rules:
      - alert: SlowOCRProcessing
        expr: |
          histogram_quantile(0.95, ocr_processing_time) > 30000
        for: 15m
        labels:
          severity: warning
          team: ml
        annotations:
          summary: "95th percentile OCR processing time > 30s"
      
      - alert: AudioGenerationSlow
        expr: |
          histogram_quantile(0.95, audio_generation_time) > 60000
        for: 15m
        labels:
          severity: warning
          team: backend
        annotations:
          summary: "Audio generation taking > 60s at p95"
```

### Escalation Matrix

```yaml
# escalation-policy.yaml
escalation_policies:
  critical:
    - level: 1
      delay: 0m
      contacts:
        - oncall_engineer
      methods:
        - pagerduty
        - slack
    - level: 2
      delay: 15m
      contacts:
        - backup_oncall
        - team_lead
      methods:
        - pagerduty
        - phone
    - level: 3
      delay: 30m
      contacts:
        - engineering_manager
        - cto
      methods:
        - phone
  
  high:
    - level: 1
      delay: 0m
      contacts:
        - oncall_engineer
      methods:
        - slack
        - email
    - level: 2
      delay: 30m
      contacts:
        - team_lead
      methods:
        - pagerduty
  
  warning:
    - level: 1
      delay: 0m
      contacts:
        - team_channel
      methods:
        - slack
```

## Real-time Monitoring Implementation

### WebSocket Dashboard for Live Metrics

```rust
// src/monitoring/realtime.rs
use axum::extract::ws::{WebSocket, WebSocketUpgrade};
use tokio::time::{interval, Duration};

pub async fn websocket_metrics_handler(
    ws: WebSocketUpgrade,
    State(metrics): State<Arc<Metrics>>,
) -> Response {
    ws.on_upgrade(|socket| handle_socket(socket, metrics))
}

async fn handle_socket(mut socket: WebSocket, metrics: Arc<Metrics>) {
    let mut interval = interval(Duration::from_secs(1));
    
    loop {
        interval.tick().await;
        
        let snapshot = MetricsSnapshot {
            timestamp: Utc::now(),
            ocr_success_rate: metrics.ocr.success_rate(),
            active_users: metrics.get_active_users_count(),
            processing_queue_size: metrics.get_queue_size(),
            recent_errors: metrics.get_recent_errors(10),
            audio_generation_p95: metrics.audio.get_p95_latency(),
        };
        
        let message = serde_json::to_string(&snapshot).unwrap();
        
        if socket.send(Message::Text(message)).await.is_err() {
            break;
        }
    }
}
```

### Grafana Dashboard Configuration

```json
{
  "dashboard": {
    "id": null,
    "title": "FamilyTales Operations",
    "tags": ["familytales", "production"],
    "timezone": "browser",
    "panels": [
      {
        "gridPos": {"h": 8, "w": 12, "x": 0, "y": 0},
        "id": 1,
        "title": "OCR Success Rate",
        "targets": [
          {
            "expr": "sum(rate(ocr_success_total[5m])) / sum(rate(ocr_requests_total[5m])) * 100",
            "refId": "A"
          }
        ],
        "type": "graph",
        "yaxes": [
          {
            "format": "percent",
            "max": 100,
            "min": 0
          }
        ]
      },
      {
        "gridPos": {"h": 8, "w": 12, "x": 12, "y": 0},
        "id": 2,
        "title": "Audio Generation Time (p95)",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, sum(rate(audio_generation_time_bucket[5m])) by (le))",
            "refId": "A"
          }
        ],
        "type": "graph",
        "yaxes": [
          {
            "format": "ms"
          }
        ]
      },
      {
        "gridPos": {"h": 8, "w": 24, "x": 0, "y": 8},
        "id": 3,
        "title": "Family Activity Heatmap",
        "targets": [
          {
            "expr": "sum by (hour) (increase(family_activity_events[1h]))",
            "format": "heatmap",
            "refId": "A"
          }
        ],
        "type": "heatmap"
      }
    ]
  }
}
```

## Synthetic Monitoring

### End-to-End Test Scenarios

```rust
// src/monitoring/synthetic.rs
pub struct SyntheticMonitor {
    client: HttpClient,
    test_family_id: Uuid,
}

impl SyntheticMonitor {
    pub async fn run_full_user_journey(&self) -> Result<SyntheticResult> {
        let start = Instant::now();
        let mut results = Vec::new();
        
        // 1. Upload document
        let upload_result = self.test_document_upload().await?;
        results.push(("document_upload", upload_result));
        
        // 2. Wait for OCR processing
        let ocr_result = self.wait_for_ocr_completion(upload_result.document_id).await?;
        results.push(("ocr_processing", ocr_result));
        
        // 3. Generate audio
        let audio_result = self.test_audio_generation(upload_result.document_id).await?;
        results.push(("audio_generation", audio_result));
        
        // 4. Test playback
        let playback_result = self.test_audio_playback(audio_result.audio_url).await?;
        results.push(("audio_playback", playback_result));
        
        // 5. Test family sharing
        let share_result = self.test_family_sharing(upload_result.document_id).await?;
        results.push(("family_sharing", share_result));
        
        Ok(SyntheticResult {
            duration: start.elapsed(),
            steps: results,
            success: true,
        })
    }
}
```

## Performance Monitoring Best Practices

1. **Baseline Establishment**
   - Track metrics for 2 weeks before setting alerts
   - Document expected ranges for each metric
   - Review baselines quarterly

2. **Alert Fatigue Prevention**
   - No more than 5 critical alerts
   - Automatic alert suppression during maintenance
   - Regular alert effectiveness reviews

3. **Dashboard Organization**
   - Executive dashboard (business metrics)
   - Operations dashboard (technical health)
   - Debug dashboard (detailed metrics)

4. **Data Retention**
   - Raw metrics: 30 days
   - 5-minute aggregates: 90 days
   - Hourly aggregates: 2 years

5. **Regular Reviews**
   - Weekly metrics review meeting
   - Monthly trend analysis
   - Quarterly capacity planning