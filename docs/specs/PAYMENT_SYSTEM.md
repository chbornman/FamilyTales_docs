# Payment System Specification - MVP

## Overview

The FamilyTales MVP payment system implements a family-based subscription model using Stripe for payment processing. Each family has one designated payer (Family Owner) who manages the subscription for unlimited family members.

**MVP Requirements** (from [`MVP_REQUIREMENTS.md`](../../MVP_REQUIREMENTS.md)):
- 14-day free trial with credit card required upfront
- Family-based billing (not per-user)
- Two tiers: Free (after trial) and Family Plan ($14.99/month)
- Automatic billing after trial unless cancelled
- Grace period for failed payments

## Subscription Tiers

### Free Trial (14 Days)
- **Price**: $0 for 14 days
- **Features**:
  - All Family Plan features during trial
  - Credit card required upfront
  - Automatic conversion to paid plan
  - Can cancel anytime during trial

### Free Tier (Post-Trial)
- **Price**: $0/month
- **Features**:
  - 3 stories per month (resets monthly)
  - Basic TTS voice only
  - Up to 5 family members
  - Watermark on shared content
  - Standard processing (up to 24 hours)

### Family Plan
- **Price**: $14.99/month (MVP has monthly billing only)
- **Features**:
  - Unlimited stories
  - Unlimited family members
  - Premium TTS voice (1 high-quality option)
  - Priority processing (under 5 minutes)
  - No watermarks
  - Email support

## Stripe Integration

### Customer Management

```rust
// Customer creation linked to Head of Family
pub struct StripeCustomer {
    pub id: String,
    pub family_id: Uuid,
    pub head_of_family_id: Uuid,
    pub email: String,
    pub stripe_customer_id: String,
    pub created_at: DateTime<Utc>,
}

// API endpoint for customer creation
#[post("/api/payments/customers")]
async fn create_stripe_customer(
    family_id: Uuid,
    user: AuthenticatedUser,
    stripe: web::Data<StripeClient>,
) -> Result<HttpResponse, Error> {
    // Verify user is head of family
    // Create Stripe customer
    // Store in database
}
```

### Subscription Management

```rust
pub struct Subscription {
    pub id: Uuid,
    pub family_id: Uuid,
    pub stripe_subscription_id: String,
    pub stripe_customer_id: String,
    pub tier: SubscriptionTier,
    pub status: SubscriptionStatus,
    pub current_period_start: DateTime<Utc>,
    pub current_period_end: DateTime<Utc>,
    pub cancel_at_period_end: bool,
}

#[derive(Debug, Serialize, Deserialize)]
pub enum SubscriptionTier {
    Free,
    Family,
    // Legacy tier removed from MVP
}

#[derive(Debug, Serialize, Deserialize)]
pub enum SubscriptionStatus {
    Trialing,    // 14-day free trial (MVP priority)
    Active,
    PastDue,
    Canceled,
    Incomplete,
    IncompleteExpired,
    Unpaid,
}
```

### Webhook Handling

```rust
#[post("/api/webhooks/stripe")]
async fn handle_stripe_webhook(
    payload: web::Bytes,
    signature: web::Header<StripeSignature>,
    db: web::Data<Database>,
) -> Result<HttpResponse, Error> {
    // Verify webhook signature
    let event = stripe::Webhook::construct_event(
        &payload,
        signature.as_str(),
        &webhook_secret,
    )?;
    
    match event.type_ {
        EventType::CustomerSubscriptionCreated => handle_subscription_created(event, db).await?,
        EventType::CustomerSubscriptionUpdated => handle_subscription_updated(event, db).await?,
        EventType::CustomerSubscriptionDeleted => handle_subscription_deleted(event, db).await?,
        EventType::InvoicePaymentSucceeded => handle_payment_succeeded(event, db).await?,
        EventType::InvoicePaymentFailed => handle_payment_failed(event, db).await?,
        _ => {}
    }
    
    Ok(HttpResponse::Ok().finish())
}
```

## Family-Based Billing Model

### One Payer Per Family

```rust
// MVP: Simplified family structure - Owner is the payer
pub struct Family {
    pub id: Uuid,
    pub name: String,
    pub owner_id: Uuid,  // Owner is always the payer in MVP
    pub subscription_id: Option<Uuid>,
    pub created_at: DateTime<Utc>,
}

// Ensure only payer can manage subscription
#[middleware]
async fn require_payer(
    req: ServiceRequest,
    family_id: Uuid,
) -> Result<ServiceRequest, Error> {
    let user = get_authenticated_user(&req)?;
    let family = get_family(family_id)?;
    
    if user.id != family.payer_id {
        return Err(Error::Forbidden);
    }
    
    Ok(req)
}
```

### Usage Tracking

```rust
pub struct FamilyUsage {
    pub id: Uuid,
    pub family_id: Uuid,
    pub month: DateTime<Utc>,
    pub stories_created: i32,
    pub storage_bytes: i64,
    pub api_calls: i32,
    pub family_members_active: i32,
}

// Track usage per action
async fn track_story_creation(
    family_id: Uuid,
    db: &Database,
) -> Result<(), Error> {
    // Increment story count
    // Check against tier limits
    // Store usage metrics
}
```

## Advanced Features

### Gift Subscriptions (POST-MVP)

**Note**: Gift subscriptions are not included in MVP scope.

```rust
// POST-MVP: Gift subscriptions for future implementation
pub struct GiftSubscription {
    pub id: Uuid,
    pub gifter_email: String,
    pub recipient_family_id: Uuid,
    pub tier: SubscriptionTier,
    pub duration_months: i32,
    pub redemption_code: String,
    pub redeemed_at: Option<DateTime<Utc>>,
    pub expires_at: DateTime<Utc>,
}
```

### Payment Method Management

**MVP Scope**: Single payment method per family (credit card required for trial signup).

```rust
// MVP: Simplified payment method management
#[get("/api/payments/method")]
async fn get_payment_method(
    user: AuthenticatedUser,
    stripe: web::Data<StripeClient>,
) -> Result<HttpResponse, Error> {
    // Get default payment method for family
    // Return sanitized data (last 4 digits, brand)
}

#[put("/api/payments/method")]
async fn update_payment_method(
    payment_method_id: String,
    user: AuthenticatedUser,
    stripe: web::Data<StripeClient>,
) -> Result<HttpResponse, Error> {
    // Replace current payment method
    // Update Stripe customer default
}
```

### Failed Payment Handling

```rust
pub struct PaymentRetryConfig {
    pub max_retries: i32,
    pub retry_intervals: Vec<Duration>,
    pub dunning_email_schedule: Vec<i32>, // Days before/after failure
}

async fn handle_payment_failed(
    event: Event,
    db: web::Data<Database>,
) -> Result<(), Error> {
    let invoice = parse_invoice(event)?;
    
    // Update subscription status
    // Send dunning email
    // Schedule retry
    // Log for support team
    
    if invoice.attempt_count >= MAX_RETRIES {
        // Downgrade to free tier
        // Notify family members
    }
}
```

### Proration Handling

```rust
async fn handle_subscription_upgrade(
    family_id: Uuid,
    new_tier: SubscriptionTier,
    stripe: web::Data<StripeClient>,
) -> Result<Subscription, Error> {
    // Get current subscription
    // Calculate proration
    // Update subscription with proration_behavior: "create_prorations"
    // Update database
}
```

## Flutter Frontend Implementation

### Payment UI Components

```dart
// Subscription management screen
class SubscriptionScreen extends StatefulWidget {
  final Family family;
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: StreamBuilder<Subscription>(
        stream: paymentService.subscriptionStream(family.id),
        builder: (context, snapshot) {
          if (!snapshot.hasData) return LoadingIndicator();
          
          final subscription = snapshot.data!;
          return Column(
            children: [
              SubscriptionTierCard(
                tier: subscription.tier,
                status: subscription.status,
                nextBillingDate: subscription.currentPeriodEnd,
              ),
              PaymentMethodsList(familyId: family.id),
              BillingHistoryList(familyId: family.id),
              if (subscription.tier != SubscriptionTier.legacy)
                UpgradePrompt(currentTier: subscription.tier),
            ],
          );
        },
      ),
    );
  }
}
```

### Stripe Integration

```dart
class PaymentService {
  final Dio _dio;
  final stripe.Stripe _stripe;
  
  Future<void> attachPaymentMethod(String paymentMethodId) async {
    try {
      // Create setup intent on backend
      final response = await _dio.post('/api/payments/setup-intent');
      final clientSecret = response.data['client_secret'];
      
      // Confirm setup intent
      await _stripe.confirmSetupIntent(
        clientSecret,
        PaymentMethodParams.card(),
      );
      
      // Attach to customer on backend
      await _dio.post('/api/payments/methods', data: {
        'payment_method_id': paymentMethodId,
      });
    } catch (e) {
      throw PaymentException('Failed to add payment method: $e');
    }
  }
  
  Future<void> updateSubscription(SubscriptionTier newTier) async {
    final response = await _dio.post('/api/payments/subscription/update', data: {
      'tier': newTier.toString(),
    });
    
    if (response.data['requires_payment']) {
      // Handle payment requirement
      await _handlePaymentRequired(response.data['client_secret']);
    }
  }
}
```

## Revenue Tracking and Reporting

### Database Schema

```sql
-- Monthly recurring revenue tracking
CREATE TABLE revenue_metrics (
    id UUID PRIMARY KEY,
    month DATE NOT NULL,
    mrr_total DECIMAL(10,2) NOT NULL,
    mrr_new DECIMAL(10,2) NOT NULL,
    mrr_expansion DECIMAL(10,2) NOT NULL,
    mrr_contraction DECIMAL(10,2) NOT NULL,
    mrr_churn DECIMAL(10,2) NOT NULL,
    active_subscriptions INTEGER NOT NULL,
    new_subscriptions INTEGER NOT NULL,
    churned_subscriptions INTEGER NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Detailed transaction log
CREATE TABLE payment_transactions (
    id UUID PRIMARY KEY,
    family_id UUID REFERENCES families(id),
    stripe_charge_id VARCHAR(255),
    amount DECIMAL(10,2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'USD',
    status VARCHAR(50) NOT NULL,
    description TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

### Revenue Calculation

```rust
pub struct RevenueMetrics {
    pub mrr_total: Decimal,
    pub mrr_new: Decimal,
    pub mrr_expansion: Decimal,
    pub mrr_contraction: Decimal,
    pub mrr_churn: Decimal,
    pub active_subscriptions: i32,
    pub ltv_average: Decimal,
    pub churn_rate: Decimal,
}

async fn calculate_monthly_metrics(
    month: DateTime<Utc>,
    db: &Database,
) -> Result<RevenueMetrics, Error> {
    // Calculate MRR components
    // Track subscription changes
    // Calculate churn and LTV
}
```

### Admin Dashboard

```rust
#[get("/api/admin/revenue/dashboard")]
async fn revenue_dashboard(
    _admin: AdminUser,
    db: web::Data<Database>,
) -> Result<HttpResponse, Error> {
    let current_metrics = calculate_monthly_metrics(Utc::now(), &db).await?;
    let historical_metrics = get_historical_metrics(&db).await?;
    let cohort_analysis = calculate_cohort_retention(&db).await?;
    
    Ok(HttpResponse::Ok().json(RevenueDashboard {
        current: current_metrics,
        historical: historical_metrics,
        cohorts: cohort_analysis,
        forecasts: calculate_revenue_forecasts(&historical_metrics),
    }))
}
```

## Security Considerations

1. **PCI Compliance**: Never store card details; use Stripe tokens
2. **Webhook Security**: Verify all webhook signatures
3. **Access Control**: Only family payer can modify subscription
4. **Audit Trail**: Log all payment-related actions
5. **Data Encryption**: Encrypt sensitive payment data at rest

## Implementation Timeline (MVP)

**Reference**: [`SPRINTS.md`](../../SPRINTS.md) Sprint 5 (Weeks 9-10 in overall MVP timeline)

### Sprint 5: Payment System & Subscriptions (Weeks 9-10)

#### Backend Tasks (Week 1)
- Stripe integration and webhook setup
- 14-day trial implementation with automatic billing
- Subscription enforcement middleware (usage limits)
- Failed payment handling and grace periods

#### Frontend Tasks (Week 2)  
- Trial signup flow with credit card collection
- Subscription management UI (cancel, update payment method)
- Usage tracking display (stories remaining for free tier)
- Billing history and invoice access

### Key MVP Deliverables
- ✅ Credit card required for 14-day free trial
- ✅ Automatic conversion to Family Plan ($14.99/month)
- ✅ Free tier with 3 stories/month after trial ends
- ✅ Single payment method per family (no multiple cards)
- ✅ Basic failed payment recovery (3-day grace period)

### Post-MVP Features (Future Sprints)
- Gift subscriptions
- Annual billing discounts  
- Multiple payment methods
- Revenue analytics dashboard
- Advanced dunning management

## Testing Strategy

1. **Unit Tests**: All payment calculation logic
2. **Integration Tests**: Stripe API interactions
3. **End-to-End Tests**: Complete payment flows
4. **Webhook Tests**: All event types
5. **Load Tests**: Concurrent subscription updates
6. **Security Tests**: Authorization and data protection