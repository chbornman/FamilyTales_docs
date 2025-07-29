# FamilyTales Security Operations

## Overview

FamilyTales handles deeply personal family memories, making security paramount. Our security operations focus on protecting family data privacy, ensuring compliance with healthcare regulations (given the elderly user base), and maintaining the trust families place in us with their irreplaceable memories.

## Security Architecture

### Defense in Depth Strategy

```
┌─────────────────────────────────────────────────────────────┐
│                    External Security Layer                   │
│  • WAF (Web Application Firewall)                          │
│  • DDoS Protection                                         │
│  • Rate Limiting                                           │
├─────────────────────────────────────────────────────────────┤
│                    Network Security Layer                    │
│  • VPC with Private Subnets                               │
│  • Network ACLs                                            │
│  • Security Groups                                         │
├─────────────────────────────────────────────────────────────┤
│                  Application Security Layer                  │
│  • JWT Authentication                                      │
│  • RBAC Authorization                                      │
│  • Input Validation                                        │
├─────────────────────────────────────────────────────────────┤
│                     Data Security Layer                      │
│  • Encryption at Rest (AES-256)                           │
│  • Encryption in Transit (TLS 1.3)                        │
│  • Key Management (AWS KMS)                               │
└─────────────────────────────────────────────────────────────┘
```

## Family Data Privacy Protection

### Data Classification

```rust
// src/security/data_classification.rs
#[derive(Debug, Clone, PartialEq)]
pub enum DataClassification {
    PublicShareable,      // Data explicitly marked for public sharing
    FamilyPrivate,        // Default - only accessible to family members
    PersonalConfidential, // User-specific data (auth tokens, personal settings)
    HighlySensitive,      // Medical information, legal documents
}

pub struct DataClassifier {
    nlp_analyzer: NlpAnalyzer,
}

impl DataClassifier {
    pub async fn classify_document(&self, content: &str) -> DataClassification {
        // Check for sensitive content patterns
        let sensitivity_score = self.nlp_analyzer.analyze_sensitivity(content).await;
        
        // Look for medical/legal terminology
        let contains_medical = self.contains_medical_terms(content);
        let contains_legal = self.contains_legal_terms(content);
        
        match (sensitivity_score, contains_medical, contains_legal) {
            (score, true, _) if score > 0.8 => DataClassification::HighlySensitive,
            (score, _, true) if score > 0.8 => DataClassification::HighlySensitive,
            (score, _, _) if score > 0.5 => DataClassification::PersonalConfidential,
            _ => DataClassification::FamilyPrivate,
        }
    }
    
    fn contains_medical_terms(&self, content: &str) -> bool {
        const MEDICAL_TERMS: &[&str] = &[
            "diagnosis", "medication", "prescription", "medical", 
            "doctor", "hospital", "treatment", "surgery"
        ];
        
        let content_lower = content.to_lowercase();
        MEDICAL_TERMS.iter().any(|term| content_lower.contains(term))
    }
}
```

### Privacy-Preserving OCR

```rust
// src/security/private_ocr.rs
pub struct PrivateOcrProcessor {
    ocr_engine: OlmOcrEngine,
    encryption_service: EncryptionService,
    audit_logger: AuditLogger,
}

impl PrivateOcrProcessor {
    pub async fn process_document(&self, document: &Document, user_id: Uuid) -> Result<OcrResult> {
        // Audit log the access
        self.audit_logger.log_document_access(user_id, document.id, "ocr_processing").await;
        
        // Process in memory only - no disk writes
        let image_data = self.load_encrypted_image(document).await?;
        
        // Perform OCR in isolated environment
        let ocr_result = tokio::task::spawn_blocking(move || {
            // Run in separate thread with restricted permissions
            self.ocr_engine.process_isolated(image_data)
        }).await??;
        
        // Immediately encrypt results
        let encrypted_text = self.encryption_service.encrypt_text(&ocr_result.text).await?;
        
        // Clear sensitive data from memory
        self.secure_clear(&ocr_result.text);
        
        Ok(OcrResult {
            encrypted_text,
            confidence: ocr_result.confidence,
            processing_time: ocr_result.processing_time,
        })
    }
    
    fn secure_clear(&self, sensitive_data: &str) {
        // Overwrite memory with random data
        unsafe {
            let ptr = sensitive_data.as_ptr() as *mut u8;
            let len = sensitive_data.len();
            for i in 0..len {
                ptr.add(i).write_volatile(rand::random::<u8>());
            }
        }
    }
}
```

## HIPAA Compliance Considerations

### Technical Safeguards

```rust
// src/security/hipaa_compliance.rs
pub struct HipaaCompliance {
    encryption: EncryptionService,
    access_control: AccessControlService,
    audit_logger: AuditLogger,
}

impl HipaaCompliance {
    pub async fn ensure_compliance(&self, request: &Request) -> Result<()> {
        // Access Control (164.312(a)(1))
        self.verify_user_authentication(request).await?;
        self.verify_user_authorization(request).await?;
        
        // Audit Controls (164.312(b))
        self.log_access_attempt(request).await;
        
        // Integrity Controls (164.312(c)(1))
        self.verify_data_integrity(request).await?;
        
        // Transmission Security (164.312(e)(1))
        self.verify_encrypted_transmission(request)?;
        
        Ok(())
    }
    
    async fn verify_user_authentication(&self, request: &Request) -> Result<()> {
        // Verify JWT token
        let token = request.headers().get("Authorization")
            .ok_or(SecurityError::MissingAuth)?;
        
        let claims = self.verify_jwt(token).await?;
        
        // Enforce session timeout (30 minutes of inactivity)
        if claims.last_activity < Utc::now() - Duration::minutes(30) {
            return Err(SecurityError::SessionExpired);
        }
        
        Ok(())
    }
    
    async fn log_access_attempt(&self, request: &Request) {
        let log_entry = AuditLogEntry {
            timestamp: Utc::now(),
            user_id: request.user_id(),
            ip_address: request.remote_addr(),
            action: request.method().to_string(),
            resource: request.uri().path().to_string(),
            user_agent: request.headers().get("User-Agent").map(|v| v.to_str().unwrap_or("").to_string()),
            success: true, // Will be updated if request fails
        };
        
        self.audit_logger.log(log_entry).await;
    }
}
```

### Administrative Safeguards

```yaml
# hipaa-policies.yaml
administrative_safeguards:
  security_officer:
    name: "Chief Security Officer"
    responsibilities:
      - "Develop and implement security policies"
      - "Conduct risk assessments"
      - "Manage security incidents"
      - "Oversee security training"
  
  workforce_training:
    frequency: "Annual"
    topics:
      - "HIPAA Privacy Rule"
      - "HIPAA Security Rule"
      - "Handling of PHI"
      - "Incident reporting"
      - "Password policies"
    
  access_management:
    provisioning:
      - "Role-based access only"
      - "Principle of least privilege"
      - "Manager approval required"
    
    deprovisioning:
      - "Immediate on termination"
      - "Audit of accessed data"
      - "Return of company assets"
```

## Encryption Implementation

### Encryption at Rest

```rust
// src/security/encryption.rs
use ring::aead::{Aad, BoundKey, Nonce, NonceSequence, SealingKey, UnboundKey, AES_256_GCM};
use aws_sdk_kms::Client as KmsClient;

pub struct EncryptionService {
    kms_client: KmsClient,
    data_key_cache: Arc<Mutex<HashMap<String, CachedDataKey>>>,
}

impl EncryptionService {
    pub async fn encrypt_document(&self, 
        document: &[u8], 
        family_id: Uuid
    ) -> Result<EncryptedDocument> {
        // Get or create data encryption key for family
        let data_key = self.get_or_create_data_key(family_id).await?;
        
        // Generate nonce
        let nonce = self.generate_nonce();
        
        // Encrypt document
        let mut in_out = document.to_vec();
        let tag = data_key.sealing_key
            .seal_in_place_separate_tag(
                Nonce::try_assume_unique_for_key(&nonce)?,
                Aad::from(&family_id.as_bytes()[..]),
                &mut in_out
            )?;
        
        Ok(EncryptedDocument {
            ciphertext: in_out,
            nonce: nonce.to_vec(),
            tag: tag.as_ref().to_vec(),
            key_id: data_key.key_id.clone(),
            algorithm: "AES-256-GCM".to_string(),
        })
    }
    
    async fn get_or_create_data_key(&self, family_id: Uuid) -> Result<DataKey> {
        let cache_key = format!("family_key_{}", family_id);
        
        // Check cache
        if let Some(cached) = self.data_key_cache.lock().await.get(&cache_key) {
            if cached.expires_at > Utc::now() {
                return Ok(cached.data_key.clone());
            }
        }
        
        // Generate new data key using KMS
        let response = self.kms_client
            .generate_data_key()
            .key_id(&self.get_master_key_id())
            .key_spec(DataKeySpec::Aes256)
            .send()
            .await?;
        
        let data_key = DataKey {
            key_id: Uuid::new_v4().to_string(),
            plaintext: response.plaintext.unwrap(),
            ciphertext: response.ciphertext_blob.unwrap(),
        };
        
        // Cache for 24 hours
        self.data_key_cache.lock().await.insert(
            cache_key,
            CachedDataKey {
                data_key: data_key.clone(),
                expires_at: Utc::now() + Duration::hours(24),
            }
        );
        
        Ok(data_key)
    }
}
```

### Encryption in Transit

```nginx
# nginx/tls-config.conf
server {
    listen 443 ssl http2;
    server_name api.familytales.app;
    
    # TLS 1.3 only
    ssl_protocols TLSv1.3;
    ssl_ciphers 'TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256';
    
    # Strong DH parameters
    ssl_dhparam /etc/ssl/certs/dhparam-4096.pem;
    
    # OCSP stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/ssl/certs/chain.pem;
    
    # Security headers
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
    add_header X-Frame-Options "DENY" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval' https://cdn.jsdelivr.net; style-src 'self' 'unsafe-inline';" always;
}
```

## Access Control and Authorization

### Role-Based Access Control (RBAC)

```rust
// src/security/rbac.rs
#[derive(Debug, Clone, PartialEq)]
pub enum Permission {
    // Document permissions
    DocumentView,
    DocumentCreate,
    DocumentEdit,
    DocumentDelete,
    DocumentShare,
    
    // Family permissions
    FamilyInvite,
    FamilyRemoveMember,
    FamilyManageSubscription,
    FamilyViewAnalytics,
    
    // Admin permissions
    AdminViewAllFamilies,
    AdminManageUsers,
    AdminViewAuditLogs,
}

pub struct RbacService {
    db: DatabasePool,
    cache: Redis,
}

impl RbacService {
    pub async fn check_permission(
        &self,
        user_id: Uuid,
        family_id: Uuid,
        permission: Permission,
    ) -> Result<bool> {
        // Get user's role in family
        let role = self.get_user_role(user_id, family_id).await?;
        
        // Check if role has permission
        match (role, permission) {
            // Head of family has all permissions
            (FamilyRole::HeadOfFamily, _) => Ok(true),
            
            // Admin permissions
            (FamilyRole::Admin, Permission::FamilyInvite) => Ok(true),
            (FamilyRole::Admin, Permission::DocumentCreate) => Ok(true),
            (FamilyRole::Admin, Permission::DocumentEdit) => Ok(true),
            (FamilyRole::Admin, Permission::DocumentShare) => Ok(true),
            
            // Member permissions
            (FamilyRole::Member, Permission::DocumentView) => Ok(true),
            (FamilyRole::Member, Permission::DocumentCreate) => Ok(true),
            
            // Deny by default
            _ => Ok(false),
        }
    }
    
    pub async fn check_document_access(
        &self,
        user_id: Uuid,
        document_id: Uuid,
    ) -> Result<bool> {
        // Get document's family
        let document = sqlx::query!(
            "SELECT family_id FROM memory_books mb 
             JOIN threads t ON mb.id = t.book_id 
             JOIN content_segments cs ON t.id = cs.thread_id 
             WHERE cs.id = $1",
            document_id
        )
        .fetch_one(self.db.read_pool())
        .await?;
        
        // Check if user is in the family
        let is_member = sqlx::query!(
            "SELECT EXISTS(SELECT 1 FROM family_members WHERE family_id = $1 AND user_id = $2)",
            document.family_id,
            user_id
        )
        .fetch_one(self.db.read_pool())
        .await?
        .exists.unwrap_or(false);
        
        Ok(is_member)
    }
}
```

### Audit Logging

```rust
// src/security/audit_logger.rs
pub struct AuditLogger {
    db: DatabasePool,
    encryption: EncryptionService,
}

impl AuditLogger {
    pub async fn log_sensitive_access(&self, event: SensitiveAccessEvent) -> Result<()> {
        // Encrypt sensitive details
        let encrypted_details = self.encryption.encrypt_json(&event.details).await?;
        
        // Insert audit log
        sqlx::query!(
            r#"
            INSERT INTO audit_logs (
                id, timestamp, user_id, action, resource_type, 
                resource_id, ip_address, user_agent, success, 
                encrypted_details, risk_score
            ) VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11)
            "#,
            Uuid::new_v4(),
            event.timestamp,
            event.user_id,
            event.action,
            event.resource_type,
            event.resource_id,
            event.ip_address,
            event.user_agent,
            event.success,
            encrypted_details,
            self.calculate_risk_score(&event)
        )
        .execute(self.db.write_pool())
        .await?;
        
        // Alert on high-risk events
        if event.risk_score > 0.8 {
            self.send_security_alert(event).await?;
        }
        
        Ok(())
    }
    
    fn calculate_risk_score(&self, event: &SensitiveAccessEvent) -> f32 {
        let mut score = 0.0;
        
        // Failed authentication attempts
        if !event.success && event.action == "login" {
            score += 0.3;
        }
        
        // Access from new location
        if event.is_new_location {
            score += 0.2;
        }
        
        // Bulk data access
        if event.records_accessed > 100 {
            score += 0.3;
        }
        
        // Outside business hours
        if event.is_outside_business_hours {
            score += 0.2;
        }
        
        score.min(1.0)
    }
}
```

## Incident Response Procedures

### Incident Response Plan

```yaml
# incident-response-plan.yaml
incident_response_team:
  incident_commander:
    role: "Overall incident coordination"
    contact: "+1-555-0001"
    
  security_lead:
    role: "Security analysis and containment"
    contact: "+1-555-0002"
    
  engineering_lead:
    role: "Technical response and recovery"
    contact: "+1-555-0003"
    
  communications_lead:
    role: "Internal and external communications"
    contact: "+1-555-0004"

response_procedures:
  detection:
    - "Security monitoring alert"
    - "User report"
    - "Automated anomaly detection"
    
  triage:
    severity_levels:
      critical:
        description: "Data breach or system compromise"
        response_time: "15 minutes"
        
      high:
        description: "Attempted breach or vulnerability"
        response_time: "1 hour"
        
      medium:
        description: "Suspicious activity"
        response_time: "4 hours"
        
      low:
        description: "Policy violation"
        response_time: "24 hours"
  
  containment:
    immediate_actions:
      - "Isolate affected systems"
      - "Disable compromised accounts"
      - "Block malicious IPs"
      - "Preserve evidence"
    
  investigation:
    - "Collect logs and artifacts"
    - "Timeline reconstruction"
    - "Root cause analysis"
    - "Impact assessment"
  
  recovery:
    - "Remove threats"
    - "Patch vulnerabilities"
    - "Restore from backups"
    - "Verify system integrity"
  
  post_incident:
    - "Incident report"
    - "Lessons learned"
    - "Update procedures"
    - "User notification (if required)"
```

### Automated Incident Response

```rust
// src/security/incident_response.rs
pub struct IncidentResponder {
    alert_service: AlertService,
    containment_service: ContainmentService,
    forensics_service: ForensicsService,
}

impl IncidentResponder {
    pub async fn handle_security_event(&self, event: SecurityEvent) -> Result<()> {
        let incident_id = Uuid::new_v4();
        
        // Classify incident
        let severity = self.classify_incident(&event);
        
        // Create incident record
        let incident = Incident {
            id: incident_id,
            severity,
            detected_at: Utc::now(),
            event_type: event.event_type.clone(),
            affected_resources: event.affected_resources.clone(),
            status: IncidentStatus::Active,
        };
        
        // Immediate containment for critical incidents
        if severity == Severity::Critical {
            self.execute_containment(&incident).await?;
        }
        
        // Alert response team
        self.alert_response_team(&incident).await?;
        
        // Start evidence collection
        tokio::spawn(async move {
            self.forensics_service.collect_evidence(incident_id).await
        });
        
        Ok(())
    }
    
    async fn execute_containment(&self, incident: &Incident) -> Result<()> {
        match &incident.event_type {
            EventType::DataBreach { user_id, .. } => {
                // Disable user account
                self.containment_service.disable_user(*user_id).await?;
                
                // Revoke all sessions
                self.containment_service.revoke_all_sessions(*user_id).await?;
                
                // Block IP if available
                if let Some(ip) = &incident.source_ip {
                    self.containment_service.block_ip(ip).await?;
                }
            }
            
            EventType::BruteForceAttempt { target_account, source_ip } => {
                // Temporarily lock account
                self.containment_service.lock_account(target_account, Duration::hours(1)).await?;
                
                // Block source IP
                self.containment_service.block_ip(source_ip).await?;
            }
            
            _ => {}
        }
        
        Ok(())
    }
}
```

## Regular Security Audits

### Automated Security Scanning

```yaml
# .github/workflows/security-scan.yml
name: Security Scan
on:
  schedule:
    - cron: '0 2 * * *'  # Daily at 2 AM
  push:
    branches: [main, develop]

jobs:
  dependency-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Run cargo-audit
        run: |
          cargo install cargo-audit
          cargo audit
      
      - name: Run npm audit
        run: |
          cd frontend
          npm audit --production
      
      - name: OWASP Dependency Check
        uses: dependency-check/Dependency-Check_Action@main
        with:
          project: 'FamilyTales'
          format: 'HTML'
          args: >
            --enableRetired
            --enableExperimental
  
  sast-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Run Semgrep
        uses: returntocorp/semgrep-action@v1
        with:
          config: >-
            p/security-audit
            p/secrets
            p/owasp-top-ten
      
      - name: Run CodeQL
        uses: github/codeql-action/analyze@v2
        with:
          languages: 'rust,javascript'
  
  container-scan:
    runs-on: ubuntu-latest
    steps:
      - name: Run Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'familytales/api:latest'
          format: 'sarif'
          severity: 'CRITICAL,HIGH'
```

### Manual Security Assessment Checklist

```markdown
## Quarterly Security Assessment

### Application Security
- [ ] Review and update security headers
- [ ] Test for SQL injection vulnerabilities
- [ ] Test for XSS vulnerabilities
- [ ] Review authentication mechanisms
- [ ] Test authorization controls
- [ ] Review API rate limiting

### Infrastructure Security
- [ ] Review firewall rules
- [ ] Check for unnecessary open ports
- [ ] Review IAM permissions
- [ ] Validate backup encryption
- [ ] Test disaster recovery procedures
- [ ] Review logging configuration

### Data Security
- [ ] Verify encryption at rest
- [ ] Test data deletion procedures
- [ ] Review data retention policies
- [ ] Validate GDPR compliance
- [ ] Check for PII exposure in logs

### Access Management
- [ ] Review user access rights
- [ ] Audit service account permissions
- [ ] Check for inactive accounts
- [ ] Review API key usage
- [ ] Validate MFA enforcement

### Compliance
- [ ] HIPAA compliance check
- [ ] GDPR compliance check
- [ ] COPPA compliance check
- [ ] Review privacy policy
- [ ] Update security documentation
```

## Security Monitoring and Alerting

### Real-time Threat Detection

```rust
// src/security/threat_detection.rs
pub struct ThreatDetector {
    ml_model: SecurityMlModel,
    rules_engine: RulesEngine,
    alert_service: AlertService,
}

impl ThreatDetector {
    pub async fn analyze_request(&self, request: &Request) -> ThreatAnalysis {
        let features = self.extract_features(request);
        
        // ML-based anomaly detection
        let ml_score = self.ml_model.predict_threat_score(&features).await;
        
        // Rule-based detection
        let rule_violations = self.rules_engine.check_rules(&features);
        
        // Combine scores
        let threat_score = (ml_score * 0.7 + rule_violations.len() as f32 * 0.3).min(1.0);
        
        ThreatAnalysis {
            threat_score,
            ml_score,
            rule_violations,
            recommended_action: self.determine_action(threat_score),
        }
    }
    
    fn extract_features(&self, request: &Request) -> SecurityFeatures {
        SecurityFeatures {
            ip_reputation: self.check_ip_reputation(&request.remote_addr()),
            geo_anomaly: self.check_geo_anomaly(&request.remote_addr(), &request.user_id()),
            rate_limit_score: self.check_rate_limits(&request.user_id()),
            unusual_pattern: self.detect_unusual_patterns(request),
            time_anomaly: self.check_time_anomaly(&request.user_id()),
        }
    }
}
```

## Security Best Practices Implementation

1. **Principle of Least Privilege**
   - Users only access their family's data
   - Service accounts have minimal permissions
   - Regular permission audits

2. **Zero Trust Architecture**
   - Verify every request
   - No implicit trust based on network location
   - Continuous authentication

3. **Security by Design**
   - Threat modeling for new features
   - Security review in PR process
   - Automated security testing

4. **Data Minimization**
   - Only collect necessary data
   - Regular data purging
   - Anonymous analytics

5. **Incident Preparedness**
   - Regular drills
   - Updated runbooks
   - Clear escalation paths