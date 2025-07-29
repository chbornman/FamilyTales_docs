# FamilyTales Admin Panel Specification

## Overview

The FamilyTales admin panel serves as the control center for Head of Family users and system administrators. It provides comprehensive family management, usage tracking, subscription control, and content moderation capabilities.

## Table of Contents

1. [User Roles & Access](#user-roles--access)
2. [Head of Family Dashboard](#head-of-family-dashboard)
3. [Family Member Management](#family-member-management)
4. [Usage Analytics & Limits](#usage-analytics--limits)
5. [Subscription Management](#subscription-management)
6. [Content Moderation Tools](#content-moderation-tools)
7. [Support Ticket Integration](#support-ticket-integration)
8. [Technical Implementation](#technical-implementation)

## User Roles & Access

### Head of Family Role

The Head of Family is the primary account holder who pays for the subscription and has ultimate control over the family account.

```rust
pub enum AdminRole {
    HeadOfFamily {
        family_id: Uuid,
        subscription_owner: bool,
    },
    FamilyAdmin {
        family_id: Uuid,
        permissions: Vec<AdminPermission>,
    },
    SystemAdmin {
        access_level: SystemAccessLevel,
    },
}

pub enum AdminPermission {
    InviteMembers,
    RemoveMembers,
    ManageContent,
    ViewAnalytics,
    ModerateContent,
    ManageSubscription,
    TransferOwnership,
}
```

### Access Control Matrix

| Feature | Head of Family | Family Admin | Member | System Admin |
|---------|---------------|--------------|---------|--------------|
| View Dashboard | ✓ | ✓ | ✗ | ✓ |
| Invite Members | ✓ | ✓ | ✗ | ✓ |
| Remove Members | ✓ | ✗ | ✗ | ✓ |
| Manage Subscription | ✓ | ✗ | ✗ | ✓ |
| Transfer Ownership | ✓ | ✗ | ✗ | ✓ |
| View Usage Analytics | ✓ | ✓ | ✗ | ✓ |
| Moderate Content | ✓ | ✓ | ✗ | ✓ |
| Access Support | ✓ | ✓ | ✓ | ✓ |

## Head of Family Dashboard

### Dashboard Overview

The main dashboard provides a comprehensive view of family activity, usage, and health metrics.

```typescript
interface DashboardMetrics {
  // Family Overview
  totalMembers: number;
  activeMembers: number; // Active in last 30 days
  pendingInvites: number;
  
  // Usage Statistics
  documentsScanned: {
    thisMonth: number;
    total: number;
    trend: TrendData;
  };
  audioGenerated: {
    minutes: number;
    trend: TrendData;
  };
  storageUsed: {
    current: StorageSize;
    limit: StorageSize;
    percentage: number;
  };
  
  // Engagement Metrics
  memoryBooksCreated: number;
  averageListeningTime: Duration;
  topListeners: FamilyMember[];
  mostListenedBooks: MemoryBook[];
  
  // Subscription Info
  currentPlan: SubscriptionTier;
  renewalDate: Date;
  paymentStatus: PaymentStatus;
}
```

### Dashboard Components

#### 1. Quick Stats Cards

```dart
class QuickStatsCard extends StatelessWidget {
  final String title;
  final String value;
  final IconData icon;
  final Color color;
  final TrendData? trend;
  
  @override
  Widget build(BuildContext context) {
    return Card(
      child: Padding(
        padding: EdgeInsets.all(16),
        child: Column(
          children: [
            Row(
              children: [
                Icon(icon, color: color, size: 32),
                SizedBox(width: 16),
                Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Text(title, style: TextStyle(fontSize: 14)),
                    Text(value, style: TextStyle(
                      fontSize: 24,
                      fontWeight: FontWeight.bold,
                    )),
                  ],
                ),
              ],
            ),
            if (trend != null) TrendIndicator(trend: trend),
          ],
        ),
      ),
    );
  }
}
```

#### 2. Activity Timeline

Shows recent family activity with filters and search capabilities.

```rust
pub struct ActivityEvent {
    pub id: Uuid,
    pub event_type: EventType,
    pub actor: FamilyMember,
    pub timestamp: DateTime<Utc>,
    pub details: EventDetails,
}

pub enum EventType {
    DocumentScanned,
    AudioGenerated,
    MemoryBookCreated,
    MemberJoined,
    MemberInvited,
    ContentShared,
    SubscriptionChanged,
}
```

#### 3. Usage Charts

Interactive charts showing usage patterns over time.

```dart
class UsageChart extends StatelessWidget {
  final ChartType type;
  final DateRange range;
  final UsageData data;
  
  @override
  Widget build(BuildContext context) {
    return Card(
      child: Padding(
        padding: EdgeInsets.all(16),
        child: Column(
          children: [
            ChartHeader(
              title: _getChartTitle(type),
              onRangeChanged: (range) => _updateRange(range),
            ),
            SizedBox(height: 16),
            if (type == ChartType.line)
              LineChart(data: _prepareLineData(data))
            else if (type == ChartType.bar)
              BarChart(data: _prepareBarData(data))
            else
              PieChart(data: _preparePieData(data)),
          ],
        ),
      ),
    );
  }
}
```

## Family Member Management

### Member List View

Comprehensive view of all family members with management actions.

```rust
pub struct FamilyMemberView {
    pub user: User,
    pub role: FamilyRole,
    pub joined_at: DateTime<Utc>,
    pub last_active: Option<DateTime<Utc>>,
    pub usage_stats: MemberUsageStats,
    pub status: MemberStatus,
}

pub struct MemberUsageStats {
    pub documents_scanned: u32,
    pub memory_books_created: u32,
    pub audio_minutes_listened: f64,
    pub last_scan_date: Option<DateTime<Utc>>,
    pub storage_used_mb: f64,
}

pub enum MemberStatus {
    Active,
    Pending,
    Inactive { since: DateTime<Utc> },
    Suspended { reason: String },
}
```

### Member Management Actions

```dart
class MemberManagementActions extends StatelessWidget {
  final FamilyMember member;
  final bool isHeadOfFamily;
  
  @override
  Widget build(BuildContext context) {
    return PopupMenuButton<MemberAction>(
      onSelected: (action) => _handleAction(action, member),
      itemBuilder: (context) => [
        if (member.status == MemberStatus.pending)
          PopupMenuItem(
            value: MemberAction.resendInvite,
            child: ListTile(
              leading: Icon(Icons.send),
              title: Text('Resend Invite'),
            ),
          ),
        if (isHeadOfFamily && member.role != FamilyRole.headOfFamily)
          PopupMenuItem(
            value: MemberAction.changeRole,
            child: ListTile(
              leading: Icon(Icons.admin_panel_settings),
              title: Text('Change Role'),
            ),
          ),
        if (isHeadOfFamily)
          PopupMenuItem(
            value: MemberAction.remove,
            child: ListTile(
              leading: Icon(Icons.remove_circle, color: Colors.red),
              title: Text('Remove Member'),
              textColor: Colors.red,
            ),
          ),
        PopupMenuItem(
          value: MemberAction.viewActivity,
          child: ListTile(
            leading: Icon(Icons.timeline),
            title: Text('View Activity'),
          ),
        ),
      ],
    );
  }
}
```

### Invitation System

```rust
pub struct InvitationManager {
    pub async fn create_invitation(
        &self,
        family_id: Uuid,
        invited_by: Uuid,
        options: InvitationOptions,
    ) -> Result<Invitation> {
        let invitation = Invitation {
            id: Uuid::new_v4(),
            family_id,
            invite_code: generate_invite_code(),
            invited_by,
            created_at: Utc::now(),
            expires_at: Utc::now() + options.valid_for,
            max_uses: options.max_uses,
            used_count: 0,
            role: options.role,
        };
        
        self.db.save_invitation(&invitation).await?;
        
        if let Some(email) = options.email {
            self.send_email_invitation(&invitation, &email).await?;
        }
        
        Ok(invitation)
    }
}

pub struct InvitationOptions {
    pub valid_for: Duration,
    pub max_uses: Option<u32>,
    pub role: FamilyRole,
    pub email: Option<String>,
    pub custom_message: Option<String>,
}
```

## Usage Analytics & Limits

### Analytics Dashboard

Detailed analytics for understanding family usage patterns.

```typescript
interface UsageAnalytics {
  // Time-based metrics
  dailyActiveUsers: TimeSeriesData[];
  weeklyActiveUsers: TimeSeriesData[];
  monthlyActiveUsers: TimeSeriesData[];
  
  // Content metrics
  documentsProcessed: {
    byType: Record<DocumentType, number>;
    byMonth: TimeSeriesData[];
    averageProcessingTime: Duration;
  };
  
  // Audio metrics
  audioStats: {
    totalMinutesGenerated: number;
    averageListeningDuration: Duration;
    completionRate: number;
    topVoices: VoiceUsage[];
  };
  
  // Storage metrics
  storageBreakdown: {
    documents: StorageSize;
    audio: StorageSize;
    images: StorageSize;
    other: StorageSize;
  };
  
  // Engagement metrics
  engagement: {
    memoryBooksPerUser: number;
    sharingRate: number;
    collaborationScore: number;
  };
}
```

### Usage Limits Management

```rust
pub struct UsageLimits {
    pub tier: SubscriptionTier,
    pub limits: TierLimits,
    pub current_usage: CurrentUsage,
}

pub struct TierLimits {
    pub monthly_scans: Option<u32>,
    pub storage_gb: f64,
    pub family_members: Option<u32>,
    pub audio_minutes: Option<u32>,
    pub api_calls: Option<u32>,
}

pub struct CurrentUsage {
    pub scans_this_month: u32,
    pub storage_used_gb: f64,
    pub active_members: u32,
    pub audio_minutes_this_month: u32,
    pub api_calls_this_month: u32,
}

impl UsageLimits {
    pub fn check_limit(&self, limit_type: LimitType) -> LimitStatus {
        match limit_type {
            LimitType::MonthlyScans => {
                if let Some(limit) = self.limits.monthly_scans {
                    let usage_percent = (self.current_usage.scans_this_month as f64 / limit as f64) * 100.0;
                    if usage_percent >= 100.0 {
                        LimitStatus::Exceeded
                    } else if usage_percent >= 80.0 {
                        LimitStatus::Warning(usage_percent)
                    } else {
                        LimitStatus::Ok(usage_percent)
                    }
                } else {
                    LimitStatus::Unlimited
                }
            }
            // ... other limit types
        }
    }
}
```

### Usage Alerts

```dart
class UsageAlertWidget extends StatelessWidget {
  final UsageLimits limits;
  
  @override
  Widget build(BuildContext context) {
    final alerts = _getActiveAlerts(limits);
    
    if (alerts.isEmpty) return SizedBox.shrink();
    
    return Column(
      children: alerts.map((alert) => AlertCard(
        severity: alert.severity,
        title: alert.title,
        message: alert.message,
        action: alert.action,
      )).toList(),
    );
  }
  
  List<UsageAlert> _getActiveAlerts(UsageLimits limits) {
    final alerts = <UsageAlert>[];
    
    // Check each limit type
    for (final limitType in LimitType.values) {
      final status = limits.checkLimit(limitType);
      
      switch (status) {
        case LimitStatus.exceeded:
          alerts.add(UsageAlert(
            severity: AlertSeverity.error,
            title: '${limitType.displayName} Limit Exceeded',
            message: _getExceededMessage(limitType),
            action: UpgradeAction(),
          ));
          break;
        case LimitStatus.warning:
          if (status.percentage >= 90) {
            alerts.add(UsageAlert(
              severity: AlertSeverity.warning,
              title: '${limitType.displayName} Nearly Exceeded',
              message: _getWarningMessage(limitType, status.percentage),
              action: ViewUsageAction(),
            ));
          }
          break;
      }
    }
    
    return alerts;
  }
}
```

## Subscription Management

### Subscription Overview

```rust
pub struct SubscriptionDashboard {
    pub current_plan: SubscriptionPlan,
    pub billing_info: BillingInfo,
    pub payment_history: Vec<Payment>,
    pub upcoming_invoice: Option<Invoice>,
}

pub struct SubscriptionPlan {
    pub tier: SubscriptionTier,
    pub status: SubscriptionStatus,
    pub started_at: DateTime<Utc>,
    pub current_period_end: DateTime<Utc>,
    pub cancel_at_period_end: bool,
    pub stripe_subscription_id: String,
}

pub struct BillingInfo {
    pub payment_method: PaymentMethod,
    pub billing_address: Address,
    pub tax_info: Option<TaxInfo>,
}

pub enum SubscriptionStatus {
    Active,
    PastDue,
    Cancelled,
    Paused,
    Trialing { ends_at: DateTime<Utc> },
}
```

### Plan Management Interface

```dart
class SubscriptionManagementView extends StatelessWidget {
  final SubscriptionDashboard dashboard;
  
  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        CurrentPlanCard(
          plan: dashboard.currentPlan,
          onUpgrade: () => _showUpgradeDialog(context),
          onCancel: () => _showCancelDialog(context),
        ),
        
        PaymentMethodCard(
          paymentMethod: dashboard.billingInfo.paymentMethod,
          onUpdate: () => _updatePaymentMethod(context),
        ),
        
        if (dashboard.upcomingInvoice != null)
          UpcomingInvoiceCard(
            invoice: dashboard.upcomingInvoice!,
            onViewDetails: () => _showInvoiceDetails(context),
          ),
        
        BillingHistoryList(
          payments: dashboard.paymentHistory,
          onDownloadInvoice: (payment) => _downloadInvoice(payment),
        ),
      ],
    );
  }
}
```

### Upgrade/Downgrade Flow

```rust
impl SubscriptionManager {
    pub async fn change_plan(
        &self,
        family_id: Uuid,
        new_tier: SubscriptionTier,
        proration_behavior: ProrationBehavior,
    ) -> Result<SubscriptionChange> {
        let current_sub = self.get_subscription(family_id).await?;
        
        // Validate plan change
        self.validate_plan_change(&current_sub, &new_tier)?;
        
        // Calculate proration
        let proration = self.calculate_proration(
            &current_sub,
            &new_tier,
            proration_behavior,
        )?;
        
        // Update in Stripe
        let stripe_update = UpdateSubscription {
            items: vec![UpdateSubscriptionItem {
                id: current_sub.stripe_item_id,
                price: new_tier.stripe_price_id(),
            }],
            proration_behavior,
            ..Default::default()
        };
        
        let updated_sub = self.stripe_client
            .update_subscription(&current_sub.stripe_subscription_id, stripe_update)
            .await?;
        
        // Update local database
        self.db.update_subscription(family_id, &updated_sub).await?;
        
        // Send confirmation email
        self.email_service.send_plan_change_confirmation(
            family_id,
            &current_sub.tier,
            &new_tier,
            &proration,
        ).await?;
        
        Ok(SubscriptionChange {
            previous_tier: current_sub.tier,
            new_tier,
            proration,
            effective_date: Utc::now(),
        })
    }
}
```

## Content Moderation Tools

### Content Review Queue

```rust
pub struct ModerationQueue {
    pub pending_items: Vec<ModerationItem>,
    pub filters: ModerationFilters,
    pub sort_by: SortOption,
}

pub struct ModerationItem {
    pub id: Uuid,
    pub content_type: ContentType,
    pub flagged_by: FlagSource,
    pub flag_reason: FlagReason,
    pub content_preview: ContentPreview,
    pub family_id: Uuid,
    pub created_at: DateTime<Utc>,
    pub priority: ModerationPriority,
}

pub enum FlagSource {
    AutomatedDetection { confidence: f32 },
    UserReport { reporter_id: Uuid },
    AdminReview,
}

pub enum FlagReason {
    InappropriateContent,
    PrivacyViolation,
    CopyrightConcern,
    QualityIssue,
    Other(String),
}
```

### Moderation Interface

```dart
class ModerationToolsView extends StatelessWidget {
  final ModerationQueue queue;
  
  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        ModerationFiltersBar(
          currentFilters: queue.filters,
          onFiltersChanged: (filters) => _updateFilters(filters),
        ),
        
        Expanded(
          child: ListView.builder(
            itemCount: queue.pendingItems.length,
            itemBuilder: (context, index) {
              final item = queue.pendingItems[index];
              return ModerationItemCard(
                item: item,
                onApprove: () => _approveContent(item),
                onReject: () => _rejectContent(item),
                onEscalate: () => _escalateToSupport(item),
              );
            },
          ),
        ),
      ],
    );
  }
}

class ModerationItemCard extends StatelessWidget {
  final ModerationItem item;
  final VoidCallback onApprove;
  final VoidCallback onReject;
  final VoidCallback onEscalate;
  
  @override
  Widget build(BuildContext context) {
    return Card(
      color: _getPriorityColor(item.priority),
      child: ExpansionTile(
        title: Text('${item.contentType.name} - ${item.flagReason.name}'),
        subtitle: Text('Flagged by: ${item.flaggedBy.description}'),
        children: [
          ContentPreviewWidget(preview: item.contentPreview),
          
          ButtonBar(
            children: [
              ElevatedButton.icon(
                icon: Icon(Icons.check),
                label: Text('Approve'),
                onPressed: onApprove,
                style: ElevatedButton.styleFrom(
                  backgroundColor: Colors.green,
                ),
              ),
              ElevatedButton.icon(
                icon: Icon(Icons.close),
                label: Text('Reject'),
                onPressed: onReject,
                style: ElevatedButton.styleFrom(
                  backgroundColor: Colors.red,
                ),
              ),
              TextButton.icon(
                icon: Icon(Icons.arrow_upward),
                label: Text('Escalate'),
                onPressed: onEscalate,
              ),
            ],
          ),
        ],
      ),
    );
  }
}
```

### Automated Content Scanning

```rust
pub struct ContentScanner {
    pub async fn scan_document(
        &self,
        document: &Document,
    ) -> Result<ScanResult> {
        let mut issues = Vec::new();
        
        // Text content analysis
        if let Some(text) = &document.ocr_text {
            // Check for PII
            if let Some(pii) = self.detect_pii(text).await? {
                issues.push(ContentIssue {
                    issue_type: IssueType::PiiDetected(pii),
                    severity: Severity::High,
                    location: pii.locations,
                });
            }
            
            // Check for inappropriate content
            if let Some(inappropriate) = self.detect_inappropriate(text).await? {
                issues.push(ContentIssue {
                    issue_type: IssueType::InappropriateContent(inappropriate),
                    severity: inappropriate.severity,
                    location: inappropriate.locations,
                });
            }
        }
        
        // Image analysis
        if let Some(image_url) = &document.image_url {
            let image_issues = self.scan_image(image_url).await?;
            issues.extend(image_issues);
        }
        
        Ok(ScanResult {
            document_id: document.id,
            issues,
            scanned_at: Utc::now(),
        })
    }
    
    async fn detect_pii(&self, text: &str) -> Result<Option<PiiDetection>> {
        // Use regex patterns for common PII
        let patterns = vec![
            (r"\d{3}-\d{2}-\d{4}", PiiType::Ssn),
            (r"\d{16}", PiiType::CreditCard),
            (r"\d{3}\.\d{3}\.\d{4}", PiiType::Phone),
        ];
        
        let mut detections = Vec::new();
        
        for (pattern, pii_type) in patterns {
            let re = Regex::new(pattern)?;
            for mat in re.find_iter(text) {
                detections.push(PiiMatch {
                    pii_type: pii_type.clone(),
                    location: TextLocation {
                        start: mat.start(),
                        end: mat.end(),
                    },
                    redacted_value: self.redact_value(mat.as_str(), &pii_type),
                });
            }
        }
        
        if detections.is_empty() {
            Ok(None)
        } else {
            Ok(Some(PiiDetection { detections }))
        }
    }
}
```

## Support Ticket Integration

### Support Dashboard

```rust
pub struct SupportDashboard {
    pub open_tickets: Vec<SupportTicket>,
    pub ticket_stats: TicketStatistics,
    pub common_issues: Vec<CommonIssue>,
}

pub struct SupportTicket {
    pub id: Uuid,
    pub family_id: Uuid,
    pub submitted_by: Uuid,
    pub category: TicketCategory,
    pub subject: String,
    pub description: String,
    pub priority: TicketPriority,
    pub status: TicketStatus,
    pub assigned_to: Option<Uuid>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
    pub messages: Vec<TicketMessage>,
}

pub enum TicketCategory {
    Technical,
    Billing,
    AccountManagement,
    ContentIssue,
    FeatureRequest,
    Other,
}

pub enum TicketStatus {
    Open,
    InProgress,
    WaitingForCustomer,
    Resolved,
    Closed,
}
```

### Integrated Support Interface

```dart
class SupportIntegration extends StatelessWidget {
  final Family family;
  
  @override
  Widget build(BuildContext context) {
    return DefaultTabController(
      length: 3,
      child: Column(
        children: [
          TabBar(
            tabs: [
              Tab(text: 'Active Tickets'),
              Tab(text: 'History'),
              Tab(text: 'FAQ'),
            ],
          ),
          Expanded(
            child: TabBarView(
              children: [
                ActiveTicketsView(familyId: family.id),
                TicketHistoryView(familyId: family.id),
                FaqView(userContext: family),
              ],
            ),
          ),
        ],
      ),
    );
  }
}

class CreateTicketDialog extends StatefulWidget {
  final Family family;
  final DocumentContext? context;
  
  @override
  State<CreateTicketDialog> createState() => _CreateTicketDialogState();
}

class _CreateTicketDialogState extends State<CreateTicketDialog> {
  final _formKey = GlobalKey<FormState>();
  TicketCategory? _category;
  final _subjectController = TextEditingController();
  final _descriptionController = TextEditingController();
  List<Attachment> _attachments = [];
  
  @override
  Widget build(BuildContext context) {
    return AlertDialog(
      title: Text('Contact Support'),
      content: Form(
        key: _formKey,
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            DropdownButtonFormField<TicketCategory>(
              value: _category,
              decoration: InputDecoration(labelText: 'Category'),
              items: TicketCategory.values.map((cat) => 
                DropdownMenuItem(
                  value: cat,
                  child: Text(cat.displayName),
                ),
              ).toList(),
              onChanged: (value) => setState(() => _category = value),
              validator: (value) => 
                value == null ? 'Please select a category' : null,
            ),
            
            TextFormField(
              controller: _subjectController,
              decoration: InputDecoration(labelText: 'Subject'),
              validator: (value) => 
                value?.isEmpty ?? true ? 'Please enter a subject' : null,
            ),
            
            TextFormField(
              controller: _descriptionController,
              decoration: InputDecoration(labelText: 'Description'),
              maxLines: 4,
              validator: (value) => 
                value?.isEmpty ?? true ? 'Please describe your issue' : null,
            ),
            
            AttachmentPicker(
              attachments: _attachments,
              onAttachmentsChanged: (attachments) => 
                setState(() => _attachments = attachments),
              allowedTypes: ['image/*', 'application/pdf'],
            ),
            
            if (widget.context != null)
              CheckboxListTile(
                title: Text('Include document context'),
                subtitle: Text('Helps support understand your issue'),
                value: _includeContext,
                onChanged: (value) => setState(() => _includeContext = value!),
              ),
          ],
        ),
      ),
      actions: [
        TextButton(
          onPressed: () => Navigator.of(context).pop(),
          child: Text('Cancel'),
        ),
        ElevatedButton(
          onPressed: _submitTicket,
          child: Text('Submit'),
        ),
      ],
    );
  }
  
  Future<void> _submitTicket() async {
    if (!_formKey.currentState!.validate()) return;
    
    final ticket = CreateTicketRequest(
      familyId: widget.family.id,
      category: _category!,
      subject: _subjectController.text,
      description: _descriptionController.text,
      attachments: _attachments,
      context: _includeContext ? widget.context : null,
    );
    
    await supportService.createTicket(ticket);
    
    if (mounted) {
      Navigator.of(context).pop();
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('Support ticket created')),
      );
    }
  }
}
```

### Knowledge Base Integration

```rust
pub struct KnowledgeBase {
    pub async fn search_articles(
        &self,
        query: &str,
        context: Option<UserContext>,
    ) -> Result<Vec<KbArticle>> {
        let mut articles = self.text_search(query).await?;
        
        // Boost articles based on user context
        if let Some(ctx) = context {
            articles = self.rank_by_relevance(articles, &ctx);
        }
        
        // Filter by user's subscription tier
        articles.retain(|article| {
            article.required_tier.is_none() || 
            article.required_tier <= Some(ctx.subscription_tier)
        });
        
        Ok(articles)
    }
    
    pub async fn suggest_articles(
        &self,
        error: &AppError,
    ) -> Result<Vec<KbArticle>> {
        // Map common errors to relevant articles
        let article_ids = match error {
            AppError::OcrFailed(_) => vec!["ocr-troubleshooting", "image-quality"],
            AppError::AudioGenerationFailed(_) => vec!["audio-issues", "voice-selection"],
            AppError::SubscriptionExpired => vec!["renew-subscription", "payment-methods"],
            _ => vec!["general-troubleshooting"],
        };
        
        self.get_articles_by_ids(&article_ids).await
    }
}
```

## Technical Implementation

### Admin Panel Architecture

```rust
// Admin API Routes
pub fn admin_routes() -> Router {
    Router::new()
        .route("/dashboard", get(handlers::admin::get_dashboard))
        .route("/members", get(handlers::admin::list_members))
        .route("/members/:id", delete(handlers::admin::remove_member))
        .route("/members/:id/role", patch(handlers::admin::update_role))
        .route("/analytics", get(handlers::admin::get_analytics))
        .route("/subscription", get(handlers::admin::get_subscription))
        .route("/subscription/change", post(handlers::admin::change_plan))
        .route("/moderation/queue", get(handlers::admin::moderation_queue))
        .route("/moderation/:id/action", post(handlers::admin::moderate_content))
        .route("/support/tickets", get(handlers::admin::list_tickets))
        .route("/support/tickets", post(handlers::admin::create_ticket))
        .layer(middleware::from_fn(auth::require_admin))
}

// Admin middleware
pub async fn require_admin(
    State(state): State<AppState>,
    user: User,
    req: Request,
    next: Next,
) -> Result<Response> {
    let family_id = extract_family_id(&req)?;
    let member = state.db
        .get_family_member(family_id, user.id)
        .await?;
    
    if !member.is_admin() {
        return Err(ApiError::Forbidden);
    }
    
    Ok(next.run(req).await)
}
```

### Real-time Updates

```dart
class AdminDashboardProvider extends StateNotifier<DashboardState> {
  late final StreamSubscription _subscription;
  
  AdminDashboardProvider() : super(DashboardState.loading()) {
    _subscription = adminService.dashboardStream.listen((update) {
      state = state.copyWith(
        metrics: update.metrics,
        lastUpdated: DateTime.now(),
      );
    });
  }
  
  @override
  void dispose() {
    _subscription.cancel();
    super.dispose();
  }
  
  Future<void> refresh() async {
    state = state.copyWith(isLoading: true);
    
    try {
      final dashboard = await adminService.getDashboard();
      state = DashboardState.loaded(dashboard);
    } catch (e) {
      state = DashboardState.error(e.toString());
    }
  }
}
```

### Security Considerations

```rust
impl AdminSecurity {
    pub async fn validate_admin_action(
        &self,
        actor: &User,
        action: AdminAction,
        target_family_id: Uuid,
    ) -> Result<()> {
        // Get actor's role in the family
        let actor_member = self.db
            .get_family_member(target_family_id, actor.id)
            .await?;
        
        // Check permissions based on action
        match action {
            AdminAction::RemoveMember(member_id) => {
                // Only head of family can remove members
                if !actor_member.is_head_of_family() {
                    return Err(ApiError::InsufficientPermissions);
                }
                
                // Cannot remove self
                if member_id == actor.id {
                    return Err(ApiError::CannotRemoveSelf);
                }
                
                // Cannot remove another head of family
                let target_member = self.db
                    .get_family_member(target_family_id, member_id)
                    .await?;
                    
                if target_member.is_head_of_family() {
                    return Err(ApiError::CannotRemoveHeadOfFamily);
                }
            }
            
            AdminAction::TransferOwnership(new_owner_id) => {
                // Only current head can transfer
                if !actor_member.is_head_of_family() {
                    return Err(ApiError::InsufficientPermissions);
                }
                
                // New owner must be existing member
                self.db
                    .get_family_member(target_family_id, new_owner_id)
                    .await?;
            }
            
            AdminAction::CancelSubscription => {
                // Only head of family can cancel
                if !actor_member.is_head_of_family() {
                    return Err(ApiError::InsufficientPermissions);
                }
            }
            
            _ => {
                // Other actions require admin role
                if !actor_member.is_admin() {
                    return Err(ApiError::InsufficientPermissions);
                }
            }
        }
        
        // Log admin action for audit trail
        self.audit_log.log_admin_action(
            actor,
            action,
            target_family_id,
        ).await?;
        
        Ok(())
    }
}
```

### Performance Optimization

```rust
// Cache frequently accessed admin data
pub struct AdminCache {
    redis: RedisConnection,
}

impl AdminCache {
    pub async fn get_dashboard_metrics(
        &self,
        family_id: Uuid,
    ) -> Result<Option<DashboardMetrics>> {
        let key = format!("admin:dashboard:{}", family_id);
        
        if let Some(cached) = self.redis.get(&key).await? {
            return Ok(Some(serde_json::from_str(&cached)?));
        }
        
        Ok(None)
    }
    
    pub async fn set_dashboard_metrics(
        &self,
        family_id: Uuid,
        metrics: &DashboardMetrics,
        ttl: Duration,
    ) -> Result<()> {
        let key = format!("admin:dashboard:{}", family_id);
        let value = serde_json::to_string(metrics)?;
        
        self.redis.setex(&key, ttl.as_secs(), &value).await?;
        
        Ok(())
    }
    
    pub async fn invalidate_family_cache(
        &self,
        family_id: Uuid,
    ) -> Result<()> {
        let pattern = format!("admin:*:{}:*", family_id);
        let keys: Vec<String> = self.redis.keys(&pattern).await?;
        
        if !keys.is_empty() {
            self.redis.del(&keys).await?;
        }
        
        Ok(())
    }
}
```

## Admin Panel UI Components

### Responsive Layout

```dart
class AdminPanelLayout extends StatelessWidget {
  final Widget child;
  final Family family;
  
  @override
  Widget build(BuildContext context) {
    return LayoutBuilder(
      builder: (context, constraints) {
        final isDesktop = constraints.maxWidth >= 1200;
        final isTablet = constraints.maxWidth >= 768;
        
        if (isDesktop) {
          return Row(
            children: [
              AdminSidebar(family: family),
              VerticalDivider(width: 1),
              Expanded(child: child),
            ],
          );
        } else if (isTablet) {
          return Row(
            children: [
              AdminRailNavigation(family: family),
              VerticalDivider(width: 1),
              Expanded(child: child),
            ],
          );
        } else {
          return Scaffold(
            drawer: AdminDrawer(family: family),
            body: child,
          );
        }
      },
    );
  }
}
```

### Data Tables

```dart
class MembersDataTable extends StatelessWidget {
  final List<FamilyMemberView> members;
  final bool isHeadOfFamily;
  
  @override
  Widget build(BuildContext context) {
    return PaginatedDataTable(
      header: Text('Family Members'),
      actions: [
        IconButton(
          icon: Icon(Icons.person_add),
          onPressed: () => _showInviteDialog(context),
          tooltip: 'Invite Member',
        ),
        IconButton(
          icon: Icon(Icons.download),
          onPressed: () => _exportMemberList(),
          tooltip: 'Export List',
        ),
      ],
      columns: [
        DataColumn(label: Text('Name')),
        DataColumn(label: Text('Email')),
        DataColumn(label: Text('Role')),
        DataColumn(label: Text('Joined')),
        DataColumn(label: Text('Last Active')),
        DataColumn(label: Text('Documents')),
        DataColumn(label: Text('Storage')),
        DataColumn(label: Text('Actions')),
      ],
      source: MembersDataSource(
        members: members,
        isHeadOfFamily: isHeadOfFamily,
        onActionSelected: _handleMemberAction,
      ),
      sortColumnIndex: 0,
      sortAscending: true,
      showCheckboxColumn: false,
      rowsPerPage: 10,
    );
  }
}
```

## Conclusion

The FamilyTales admin panel provides comprehensive tools for family management, usage tracking, and content moderation. The Head of Family role ensures clear ownership and control, while delegated admin permissions allow for flexible family management. The integration of analytics, subscription management, and support tools creates a complete administrative experience that scales from small families to large extended family groups.