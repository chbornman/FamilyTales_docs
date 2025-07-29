# FamilyTales Backup and Disaster Recovery

## Overview

FamilyTales handles precious family memories that are often irreplaceable. Our backup and disaster recovery strategy ensures that family data is protected against all forms of data loss, from accidental deletion to regional disasters.

## Architecture Components

### Critical Data Types

1. **PostgreSQL Database** (Primary Data Store)
   - User accounts and family relationships
   - Memory book metadata and organization
   - Document processing history
   - Subscription and billing data
   - OCR corrections and improvements

2. **Mux Media Storage**
   - Original document images (JPEG, PNG, PDF)
   - Generated audio files (HLS streams)
   - Processed video thumbnails
   - Document preview images

3. **Redis Cache**
   - Session data
   - Job queue state
   - Temporary processing data
   - API response cache

4. **Application Configuration**
   - Environment variables
   - Service credentials
   - Feature flags
   - Infrastructure as Code

## Backup Strategies

### PostgreSQL Backup Strategy

#### Continuous Replication
```bash
# Primary-Replica Setup with Streaming Replication
# Master: postgres-primary.familytales.internal
# Replica: postgres-replica.familytales.internal

# On replica server
pg_basebackup -h postgres-primary.familytales.internal \
  -D /var/lib/postgresql/14/main \
  -U replicator \
  -v -P -W \
  -X stream
```

#### Automated Daily Backups
```yaml
# backup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
spec:
  schedule: "0 2 * * *"  # 2 AM daily
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: postgres-backup
            image: postgres:14
            env:
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: password
            command:
            - /bin/bash
            - -c
            - |
              DATE=$(date +%Y%m%d_%H%M%S)
              pg_dump -h postgres-primary \
                -U familytales \
                -d familytales_production \
                --format=custom \
                --verbose \
                --file=/backups/familytales_${DATE}.dump
              
              # Upload to S3
              aws s3 cp /backups/familytales_${DATE}.dump \
                s3://familytales-backups/postgres/${DATE}/
              
              # Keep only last 30 days locally
              find /backups -name "*.dump" -mtime +30 -delete
```

#### Point-in-Time Recovery (PITR)
```bash
# Enable WAL archiving in postgresql.conf
archive_mode = on
archive_command = 'aws s3 cp %p s3://familytales-backups/wal/%f'
wal_level = replica
max_wal_senders = 3

# Recovery configuration
restore_command = 'aws s3 cp s3://familytales-backups/wal/%f %p'
recovery_target_time = '2024-01-15 14:30:00'
```

### Mux Media Backup

Since Mux provides built-in redundancy and durability, we focus on metadata backup and recovery procedures:

```rust
// Mux asset inventory backup
pub async fn backup_mux_inventory() -> Result<()> {
    let mux_client = MuxClient::new(&config.mux_token_id, &config.mux_token_secret);
    
    // List all assets
    let assets = mux_client.list_assets().await?;
    
    // Create inventory manifest
    let manifest = AssetManifest {
        timestamp: Utc::now(),
        total_assets: assets.len(),
        assets: assets.iter().map(|asset| AssetRecord {
            id: asset.id.clone(),
            playback_id: asset.playback_ids[0].id.clone(),
            created_at: asset.created_at,
            duration: asset.duration,
            metadata: asset.passthrough.clone(),
        }).collect(),
    };
    
    // Save to S3
    let manifest_json = serde_json::to_string_pretty(&manifest)?;
    s3_client.put_object()
        .bucket("familytales-backups")
        .key(format!("mux-inventory/{}.json", Utc::now().format("%Y%m%d")))
        .body(manifest_json.into_bytes().into())
        .send()
        .await?;
    
    Ok(())
}
```

### Redis Backup Strategy

Redis data is transient but we backup for faster recovery:

```bash
# Redis persistence configuration
# redis.conf
save 900 1      # Save after 900 sec if at least 1 key changed
save 300 10     # Save after 300 sec if at least 10 keys changed
save 60 10000   # Save after 60 sec if at least 10000 keys changed

appendonly yes
appendfsync everysec

# Backup script
#!/bin/bash
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
redis-cli --rdb /backups/redis_${TIMESTAMP}.rdb
aws s3 cp /backups/redis_${TIMESTAMP}.rdb s3://familytales-backups/redis/
```

## Cross-Region Replication

### Multi-Region Setup
```yaml
# terraform/multi-region.tf
resource "aws_s3_bucket" "backup_primary" {
  bucket = "familytales-backups-us-east-1"
  region = "us-east-1"
}

resource "aws_s3_bucket" "backup_replica" {
  bucket = "familytales-backups-eu-west-1"
  region = "eu-west-1"
}

resource "aws_s3_bucket_replication_configuration" "backup_replication" {
  role   = aws_iam_role.replication.arn
  bucket = aws_s3_bucket.backup_primary.id

  rule {
    id     = "cross-region-backup"
    status = "Enabled"

    destination {
      bucket        = aws_s3_bucket.backup_replica.arn
      storage_class = "GLACIER_IR"
      
      replication_time {
        status = "Enabled"
        time {
          minutes = 15
        }
      }
    }
  }
}
```

## Disaster Recovery Procedures

### Recovery Time Objectives (RTO) and Recovery Point Objectives (RPO)

| Component | RPO | RTO | Strategy |
|-----------|-----|-----|----------|
| PostgreSQL | 1 hour | 2 hours | Streaming replication + hourly snapshots |
| Mux Media | 0 (Mux handles) | 0 | Built-in Mux redundancy |
| Redis Cache | 24 hours | 30 minutes | Can rebuild from database |
| Application | 0 | 15 minutes | Blue-green deployment |

### Recovery Runbooks

#### Scenario 1: PostgreSQL Primary Failure

```bash
#!/bin/bash
# postgres-failover.sh

echo "Starting PostgreSQL failover procedure..."

# 1. Promote replica to primary
ssh postgres-replica "sudo -u postgres pg_ctl promote -D /var/lib/postgresql/14/main"

# 2. Update application configuration
kubectl set env deployment/familytales-api \
  DATABASE_HOST=postgres-replica.familytales.internal

# 3. Verify new primary is accepting connections
PGPASSWORD=$DB_PASSWORD psql -h postgres-replica.familytales.internal \
  -U familytales -d familytales_production -c "SELECT NOW();"

# 4. Start new replica from latest backup
./restore-postgres-replica.sh

echo "Failover complete. New primary: postgres-replica"
```

#### Scenario 2: Complete Regional Failure

```bash
#!/bin/bash
# regional-failover.sh

BACKUP_REGION="eu-west-1"
PRIMARY_REGION="us-east-1"

echo "Initiating regional failover to ${BACKUP_REGION}..."

# 1. Update DNS to point to backup region
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch file://failover-dns-changes.json

# 2. Restore database in backup region
kubectl --context=${BACKUP_REGION} apply -f postgres-restore-job.yaml

# 3. Update Mux webhooks to backup region
./update-mux-webhooks.sh ${BACKUP_REGION}

# 4. Scale up services in backup region
kubectl --context=${BACKUP_REGION} scale deployment familytales-api --replicas=5

# 5. Verify health
curl -f https://api-${BACKUP_REGION}.familytales.app/health || exit 1

echo "Regional failover complete"
```

#### Scenario 3: Data Corruption Recovery

```sql
-- Point-in-time recovery procedure
-- 1. Stop application to prevent new writes
-- 2. Identify corruption time
-- 3. Restore to point before corruption

-- Create recovery database
CREATE DATABASE familytales_recovery;

-- Restore from backup before corruption
pg_restore -h localhost -U postgres \
  -d familytales_recovery \
  /backups/familytales_20240115_020000.dump

-- Apply WAL logs up to corruption point
SELECT pg_create_restore_point('before_corruption');

-- Verify data integrity
SELECT COUNT(*) FROM memory_books WHERE created_at < '2024-01-15 14:00:00';

-- Once verified, swap databases
ALTER DATABASE familytales_production RENAME TO familytales_corrupted;
ALTER DATABASE familytales_recovery RENAME TO familytales_production;
```

## Automated Backup Monitoring

### Backup Health Checks
```rust
// src/monitoring/backup_health.rs
use chrono::{DateTime, Utc, Duration};

pub struct BackupMonitor {
    s3_client: aws_sdk_s3::Client,
    alerting: AlertingService,
}

impl BackupMonitor {
    pub async fn check_backup_freshness(&self) -> Result<()> {
        let backup_configs = vec![
            BackupConfig {
                name: "PostgreSQL Daily",
                bucket: "familytales-backups",
                prefix: "postgres/",
                max_age: Duration::hours(26), // Allow 2-hour buffer
            },
            BackupConfig {
                name: "Mux Inventory",
                bucket: "familytales-backups",
                prefix: "mux-inventory/",
                max_age: Duration::hours(25),
            },
        ];
        
        for config in backup_configs {
            let latest = self.get_latest_backup(&config).await?;
            
            if let Some(backup) = latest {
                let age = Utc::now() - backup.last_modified;
                
                if age > config.max_age {
                    self.alerting.send_alert(Alert {
                        severity: Severity::High,
                        title: format!("{} backup is stale", config.name),
                        message: format!(
                            "Latest backup is {} hours old (threshold: {} hours)",
                            age.num_hours(),
                            config.max_age.num_hours()
                        ),
                    }).await?;
                }
            } else {
                self.alerting.send_alert(Alert {
                    severity: Severity::Critical,
                    title: format!("No {} backups found", config.name),
                    message: "No backups found in expected location".to_string(),
                }).await?;
            }
        }
        
        Ok(())
    }
}
```

## Family Data Export

Users have the right to export their family's data at any time:

```rust
// src/api/family_export.rs
pub async fn export_family_data(
    family_id: Uuid,
    user_id: Uuid,
) -> Result<ExportBundle> {
    // Verify user has permission
    verify_family_member(family_id, user_id).await?;
    
    // Start background job
    let job_id = queue_export_job(ExportJob {
        family_id,
        requested_by: user_id,
        include_media: true,
        format: ExportFormat::Archive,
    }).await?;
    
    Ok(ExportBundle {
        job_id,
        status_url: format!("/api/v1/exports/{}/status", job_id),
        estimated_time_minutes: 30,
    })
}

// Background worker
pub async fn process_export_job(job: ExportJob) -> Result<()> {
    let temp_dir = create_temp_directory()?;
    
    // 1. Export database data
    let family_data = export_family_database_data(job.family_id).await?;
    write_json(&temp_dir.join("family_data.json"), &family_data)?;
    
    // 2. Export memory books structure
    let memory_books = export_memory_books(job.family_id).await?;
    write_json(&temp_dir.join("memory_books.json"), &memory_books)?;
    
    // 3. Download media from Mux
    if job.include_media {
        let media_dir = temp_dir.join("media");
        create_dir_all(&media_dir)?;
        
        for book in &memory_books {
            for thread in &book.threads {
                // Download original documents
                for segment in &thread.segments {
                    let filename = format!("{}.jpg", segment.id);
                    download_from_mux(&segment.original_url, &media_dir.join(filename)).await?;
                }
                
                // Download audio
                let audio_file = format!("{}.mp3", thread.id);
                download_audio_from_mux(&thread.audio_url, &media_dir.join(audio_file)).await?;
            }
        }
    }
    
    // 4. Create ZIP archive
    let archive_path = format!("/tmp/family_export_{}.zip", job.family_id);
    create_zip_archive(&temp_dir, &archive_path)?;
    
    // 5. Upload to S3 with signed URL
    let download_url = upload_export_with_expiry(&archive_path, Duration::days(7)).await?;
    
    // 6. Notify user
    send_export_ready_email(job.requested_by, download_url).await?;
    
    Ok(())
}
```

## Testing and Validation

### Monthly Disaster Recovery Drills

```yaml
# dr-drill-checklist.yaml
disaster_recovery_drill:
  schedule: "First Saturday of each month"
  
  pre_drill:
    - Notify on-call team
    - Create drill documentation ticket
    - Snapshot production metrics
  
  drill_scenarios:
    - database_failover:
        steps:
          - Failover to replica
          - Verify application connectivity
          - Run smoke tests
          - Failover back to primary
        success_criteria:
          - Failover completes in < 5 minutes
          - No data loss
          - All health checks pass
    
    - backup_restoration:
        steps:
          - Restore database to test environment
          - Restore Redis snapshot
          - Verify Mux asset accessibility
          - Run data integrity checks
        success_criteria:
          - Restoration completes in < 2 hours
          - All family data intact
          - Media files accessible
    
    - regional_failover:
        steps:
          - Simulate region failure
          - Execute regional failover
          - Verify cross-region functionality
          - Restore to primary region
        success_criteria:
          - DNS updates propagate in < 5 minutes
          - Services available in backup region
          - No customer impact
  
  post_drill:
    - Document results
    - Update runbooks
    - Address any issues found
    - Schedule follow-up if needed
```

## Best Practices

1. **3-2-1 Backup Rule**
   - 3 copies of data (production + 2 backups)
   - 2 different storage types (S3 + Glacier)
   - 1 offsite backup (cross-region replication)

2. **Encryption**
   - All backups encrypted at rest
   - Encryption keys stored in AWS KMS
   - Separate keys per data type

3. **Access Control**
   - Backup buckets have versioning enabled
   - MFA delete protection on critical backups
   - IAM roles with least privilege

4. **Regular Testing**
   - Automated backup verification daily
   - Manual restoration testing weekly
   - Full DR drill monthly

5. **Documentation**
   - All procedures documented and versioned
   - Runbooks accessible offline
   - Contact information kept current