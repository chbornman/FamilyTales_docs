# MVP Requirements

This document defines the exact scope and boundaries for the FamilyTales Minimum Viable Product (MVP). This is what we're building first, before any additional features.

## Table of Contents
- [MVP Core Features](#mvp-core-features)
- [Platform Support](#platform-support)
- [Authentication & User Management](#authentication--user-management)
- [Payment System](#payment-system)
- [Technical Requirements](#technical-requirements)
- [What's NOT in MVP](#whats-not-in-mvp)
- [Future Considerations](#future-considerations)
- [Success Criteria](#success-criteria)

## MVP Core Features

### 1. User Onboarding
- **Account Creation**: All users must create accounts (no magic links)
- **Family Creation**: Users can create or join one family
- **Profile Setup**: Basic user info (name, email, profile photo)
- **Free Trial**: 2-week trial with credit card required

### 2. Story Creation Workflow
- **Photo Upload**: Upload up to 5 document photos per story
- **OCR Processing**: Google Vision API for text extraction
- **Text Editing**: Simple text editor to correct OCR results
- **Audio Generation**: Single TTS voice option (Google Cloud TTS)
- **Story Publishing**: Publish to family with title and description

### 3. Family Management
- **Single Family**: Users belong to one family only
- **Role-Based Access**: Owner, Admin, Member, Viewer roles
- **Email Invitations**: Invite family members via email
- **Family Dashboard**: View all family stories in chronological order

### 4. Story Consumption
- **Audio Playback**: Stream stories with HLS via Mux
- **Basic Controls**: Play, pause, seek, volume
- **Story List**: Browse all family stories
- **Story Details**: View original photos alongside audio

### 5. Account Management
- **Subscription Management**: Upgrade, downgrade, cancel subscription
- **Payment History**: View billing history
- **Family Settings**: Manage family members and permissions

## Platform Support

**Flutter Multi-Platform from Day One:**
- **iOS App**: Native iOS application
- **Android App**: Native Android application  
- **Web App**: Progressive Web App (PWA)
- **Desktop Apps**: macOS, Windows, Linux support

*Rationale: We want users on all platforms to encourage engagement with future physical products and in-app purchases.*

## Authentication & User Management

### Authentication Strategy
- **Required for All Users**: Everyone must create an account
- **Clerk Integration**: JWT-based authentication
- **Account Types**: 
  - Individual accounts (not shared family accounts)
  - Each person has their own login credentials
- **No Magic Links**: Users must remember credentials or use password reset

### User Roles
```
Owner (1 per family):
- Create/delete family
- Manage subscription and billing
- Add/remove any family member
- All admin and member permissions

Admin (multiple allowed):
- Add/remove members (except Owner)
- Manage family settings
- All member permissions

Member (default role):
- Upload and create stories
- Edit own stories
- View all family stories
- Invite new members

Viewer:
- View and listen to stories only
- Cannot upload or edit
```

## Payment System

### Free Trial Structure
- **Duration**: 14-day free trial
- **Credit Card Required**: Must provide payment method upfront
- **Full Access**: All MVP features available during trial
- **Auto-Billing**: Automatically charges after trial unless cancelled
- **Cancellation**: Can cancel anytime during trial with no charge

### Subscription Tiers

#### Free Tier (After Trial Ends)
- **3 stories per month** (resets monthly)
- **Basic voice only** (1 TTS option)
- **Up to 5 family members**
- **Watermark on shared content**
- **Standard processing** (up to 24 hours)

#### Family Plan - $14.99/month
- **Unlimited stories**
- **Unlimited family members**
- **Premium voice** (1 high-quality option)
- **Priority processing** (under 5 minutes)
- **No watermarks**
- **Email support**

### Payment Features
- **Stripe Integration**: Secure payment processing
- **Billing Management**: View invoices, update payment methods
- **Subscription Changes**: Upgrade/downgrade anytime
- **Failed Payment Handling**: Grace period and account suspension

## Technical Requirements

### Performance Standards
- **OCR Processing**: < 10 seconds per page (MVP target)
- **Audio Generation**: < 30 seconds per story (MVP target)
- **App Startup**: < 5 seconds (more lenient for MVP)
- **API Responses**: < 500ms p95 (MVP target)

### Data Storage
- **Story Limit**: 100 stories per family (MVP limit)
- **File Size**: 10MB max per photo
- **Audio Length**: 30 minutes max per story
- **Family Size**: 20 members max (MVP limit)

### Infrastructure
- **Single Server Deployment**: Docker Compose on single VPS
- **Database**: PostgreSQL with basic backup
- **Media Storage**: Mux for audio, S3-compatible for images
- **Monitoring**: Basic health checks and error logging

## What's NOT in MVP

### Features Explicitly Excluded
- **Multiple Families**: Users can only belong to one family
- **Voice Selection**: Only one TTS voice option available
- **Complex Memory Books**: No multi-thread story collections
- **Advanced Audio Features**: No speed control, bookmarks, or playlists
- **Social Sharing**: No sharing outside the family
- **Advanced Search**: Basic chronological browsing only
- **Data Export**: No ability to export family data
- **Content Moderation**: No automated content filtering
- **Mobile-Specific Features**: No camera integration beyond basic photo picker
- **Offline Mode**: Requires internet connection
- **Advanced Analytics**: Basic usage tracking only
- **API Access**: No public API
- **White-Label Options**: Single-tenant only

### Simplified Features
- **Basic OCR**: Google Vision API only (no custom models)
- **Simple Text Editor**: Basic editing, no rich formatting
- **Email Only**: No SMS or push notifications
- **Standard Support**: Email support only, no phone/chat

## Future Considerations

### Post-MVP Feature Candidates
1. **Magic Links for Convenience**: Passwordless login for frequent family members
2. **Data Export**: Allow families to export all their data
3. **Content Moderation**: Automated filtering for inappropriate content
4. **Multiple Families**: Support for both sides of family tree
5. **Voice Cloning**: Preserve actual family member voices
6. **Physical Products**: Print-on-demand memory books
7. **Advanced Memory Books**: Multi-thread story collections
8. **Mobile Camera Integration**: Direct document scanning
9. **Social Features**: Share favorite stories outside family

### Technical Debt Acceptable for MVP
- **Manual Deployment**: No CI/CD required initially
- **Basic Error Handling**: Simple error messages, no advanced recovery
- **Limited Testing**: Core functionality tested, edge cases acceptable
- **Simple Monitoring**: Basic health checks, advanced observability later
- **Performance**: Acceptable to have slower processing times initially

## Success Criteria

### Launch Readiness
- [ ] All core user flows work end-to-end
- [ ] Payment system processes subscriptions correctly
- [ ] Flutter apps deploy to all target platforms
- [ ] OCR and audio generation complete within time limits
- [ ] Family invitation system works reliably
- [ ] Basic error handling prevents crashes

### Business Metrics (3 months post-launch)
- **User Acquisition**: 1,000 registered users
- **Conversion Rate**: 3% trial-to-paid conversion
- **Retention**: 40% monthly active users
- **Content Creation**: Average 2 stories per active family
- **Platform Distribution**: 60% mobile, 30% web, 10% desktop

### Technical Metrics
- **99% Uptime**: Service availability
- **< 5% Error Rate**: Processing success rate
- **< 10 Second Response**: Average processing time
- **Zero Data Loss**: No stories or user data lost

## MVP Timeline

**Phase 1 (Weeks 1-4): Foundation**
- Backend API with authentication
- Database schema and migrations  
- Basic Flutter app shell
- Payment integration

**Phase 2 (Weeks 5-8): Core Features**
- Photo upload and storage
- OCR processing pipeline
- Family management system
- Basic audio generation

**Phase 3 (Weeks 9-12): Polish & Launch**
- Audio playback and streaming
- UI/UX refinement for elder users
- Cross-platform testing
- Production deployment

*Total MVP Development Time: 12 weeks*