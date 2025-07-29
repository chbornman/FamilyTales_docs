# FamilyTales Architecture Document

## Table of Contents
1. [Executive Summary](#executive-summary)
2. [System Architecture Overview](#system-architecture-overview)
3. [Core Components](#core-components)
4. [Technology Stack](#technology-stack)
5. [API Design](#api-design)
6. [Database Schema](#database-schema)
7. [Security & Privacy Considerations](#security--privacy-considerations)
8. [Scalability Considerations](#scalability-considerations)
9. [Technical Challenges and Solutions](#technical-challenges-and-solutions)
10. [Implementation Phases](#implementation-phases)

## Executive Summary

FamilyTales is a mobile application that converts handwritten documents (letters, memoirs, journals) into high-quality audio format, targeting the $50M market opportunity at the intersection of genealogy, digital preservation, and accessibility. The architecture leverages Flutter for cross-platform mobile development, Google's ML Kit for on-device OCR processing, and cloud-based text-to-speech services for audio generation.

### Key Architecture Principles
- **Privacy-First**: On-device OCR processing for sensitive family documents
- **Offline-First**: Core functionality available without internet connectivity
- **Scalable**: Cloud infrastructure for compute-intensive operations
- **Modular**: Microservices architecture for independent scaling
- **Secure**: End-to-end encryption for document storage and transmission

## System Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        Mobile Application                        │
│                     (Flutter - iOS/Android)                      │
├─────────────────────────────────────────────────────────────────┤
│                          Core Services                           │
├──────────────────┬──────────────────┬──────────────────────────┤
│   OCR Engine     │  Audio Engine    │    Storage Service       │
│  (Google ML Kit) │   (TTS APIs)     │  (Local + Cloud)         │
├──────────────────┴──────────────────┴──────────────────────────┤
│                      Backend Services                            │
│                   (Google Cloud Platform)                        │
├──────────────────┬──────────────────┬──────────────────────────┤
│   API Gateway    │  Microservices   │   Database Layer         │
│  (Cloud Endpoints)│  (Cloud Run)     │  (Firestore + Storage)   │
└──────────────────┴──────────────────┴──────────────────────────┘
```

### High-Level Data Flow

1. **Document Capture**: User photographs handwritten document
2. **Pre-processing**: Image enhancement and optimization
3. **OCR Processing**: On-device text extraction using Google ML Kit
4. **Text Verification**: User reviews and corrects OCR output
5. **Audio Generation**: Cloud-based TTS conversion
6. **Storage & Playback**: Secure storage with streaming capabilities

## Core Components

### 1. OCR Engine

**Primary Technology**: Google ML Kit Text Recognition v2
- On-device processing for privacy and speed
- Support for 50+ languages including cursive scripts
- Confidence scoring for accuracy assessment

**Fallback Options**:
- Cloud Vision API for complex documents
- Tesseract OCR for offline-only mode
- Manual transcription service integration

**Key Features**:
```dart
class OCREngine {
  // Multi-engine approach for maximum accuracy
  Future<OCRResult> processImage(ImageFile image) async {
    // Primary: ML Kit on-device
    final mlKitResult = await _processWithMLKit(image);
    
    // If confidence < threshold, use cloud backup
    if (mlKitResult.confidence < 0.7) {
      return await _processWithCloudVision(image);
    }
    
    return mlKitResult;
  }
}
```

### 2. Text-to-Speech Engine

**Primary Service**: Google Cloud Text-to-Speech
- 380+ voices across 50+ languages
- WaveNet and Neural2 voices for natural sound
- SSML support for pronunciation customization

**Voice Options**:
- Standard voices for basic conversion
- Premium WaveNet voices for emotional content
- Voice cloning capability (future feature with ElevenLabs)

**Audio Processing Pipeline**:
```dart
class AudioEngine {
  // Configurable voice settings per document type
  Future<AudioFile> generateAudio(
    String text, 
    VoiceSettings settings,
    DocumentType type
  ) async {
    // Apply document-specific processing
    final processedText = await _preprocessText(text, type);
    
    // Generate audio with appropriate voice
    final audio = await _ttsService.synthesize(
      text: processedText,
      voice: settings.voiceId,
      speed: settings.speed,
      pitch: settings.pitch
    );
    
    // Post-process for optimal quality
    return await _enhanceAudio(audio);
  }
}
```

### 3. Document Management System

**Core Features**:
- Document categorization (letters, memoirs, journals)
- Metadata extraction (dates, recipients, authors)
- Version control for corrections
- Family sharing capabilities

**Storage Architecture**:
- Local SQLite for offline access
- Cloud Firestore for sync and backup
- Cloud Storage for images and audio files
- CDN for global audio streaming

## Technology Stack

### Mobile Application

**Framework**: Flutter 3.x
- Single codebase for iOS and Android
- Native performance with platform-specific optimizations
- Rich widget library for elderly-friendly UI

**Key Dependencies**:
```yaml
dependencies:
  # Core functionality
  google_mlkit_text_recognition: ^0.11.0
  camera: ^0.10.5
  image_picker: ^1.0.4
  
  # Audio handling
  just_audio: ^0.9.34
  audio_service: ^0.18.10
  
  # Storage and sync
  sqflite: ^2.3.0
  cloud_firestore: ^4.13.0
  firebase_storage: ^11.5.0
  
  # UI/UX
  provider: ^6.1.0
  go_router: ^12.1.0
  flutter_animate: ^4.3.0
```

### Backend Services

**Cloud Platform**: Google Cloud Platform
- Proven scalability and reliability
- Integrated AI/ML services
- Global infrastructure

**Core Services**:
- **Cloud Run**: Containerized microservices
- **Cloud Endpoints**: API management
- **Firestore**: NoSQL document database
- **Cloud Storage**: Binary file storage
- **Cloud CDN**: Global content delivery
- **Cloud Tasks**: Async job processing

**Supporting Services**:
- **Cloud Functions**: Event-driven processing
- **Cloud Pub/Sub**: Service communication
- **Cloud Monitoring**: Observability
- **Cloud IAM**: Access control

## API Design

### RESTful API Structure

```
Base URL: https://api.familytales.app/v1

Authentication: Bearer Token (Firebase Auth)
```

### Core Endpoints

#### Documents API
```http
# Upload new document
POST /documents
Content-Type: multipart/form-data
Body: {
  image: binary,
  metadata: {
    type: "letter|memoir|journal",
    date_written: "ISO8601",
    author: "string"
  }
}

# Get document details
GET /documents/{documentId}
Response: {
  id: "string",
  status: "processing|ready|error",
  text: "string",
  confidence: 0.85,
  audio_url: "string",
  created_at: "ISO8601"
}

# Update document text
PATCH /documents/{documentId}/text
Body: {
  text: "corrected text",
  corrections: [{
    original: "string",
    corrected: "string",
    position: {start: 0, end: 10}
  }]
}

# Generate audio
POST /documents/{documentId}/generate-audio
Body: {
  voice_id: "en-US-Wavenet-D",
  speed: 1.0,
  pitch: 0
}
```

#### Collections API
```http
# Create collection
POST /collections
Body: {
  name: "Grandma's Love Letters",
  description: "string",
  privacy: "private|family|public"
}

# Add document to collection
POST /collections/{collectionId}/documents
Body: {
  document_id: "string",
  order: 1
}
```

#### Family Sharing API
```http
# Invite family member
POST /family/invites
Body: {
  email: "string",
  role: "viewer|editor|admin",
  collections: ["collectionId1", "collectionId2"]
}

# Accept invitation
POST /family/invites/{inviteId}/accept
```

### WebSocket API for Real-time Updates

```javascript
// Real-time OCR progress
ws://api.familytales.app/ws/ocr/{documentId}

// Message format
{
  type: "progress|complete|error",
  progress: 0.75,
  processed_regions: 10,
  total_regions: 15,
  current_text: "partial extracted text..."
}
```

## Database Schema

### Firestore Collections

#### Users Collection
```javascript
users/{userId} {
  email: string,
  display_name: string,
  subscription: {
    tier: "free|premium|family",
    status: "active|cancelled|past_due",
    expires_at: timestamp
  },
  preferences: {
    default_voice: string,
    default_language: string,
    auto_enhance_images: boolean
  },
  usage: {
    documents_processed: number,
    audio_minutes_generated: number,
    storage_used_mb: number
  },
  created_at: timestamp,
  updated_at: timestamp
}
```

#### Documents Collection
```javascript
documents/{documentId} {
  user_id: string,
  type: "letter|memoir|journal",
  status: "uploading|processing|ready|error",
  
  // Original image data
  image: {
    storage_path: string,
    width: number,
    height: number,
    size_bytes: number
  },
  
  // OCR results
  ocr: {
    text: string,
    confidence: number,
    language: string,
    processing_time_ms: number,
    engine_used: "mlkit|cloud_vision|manual"
  },
  
  // User corrections
  corrections: [{
    original: string,
    corrected: string,
    position: {start: number, end: number},
    timestamp: timestamp
  }],
  
  // Audio data
  audio: {
    storage_path: string,
    duration_seconds: number,
    voice_id: string,
    format: "mp3|m4a",
    size_bytes: number
  },
  
  // Metadata
  metadata: {
    title: string,
    author: string,
    recipient: string,
    date_written: date,
    tags: [string]
  },
  
  // Sharing
  sharing: {
    visibility: "private|family|public",
    shared_with: [userId],
    share_link: string
  },
  
  created_at: timestamp,
  updated_at: timestamp
}
```

#### Collections Collection
```javascript
collections/{collectionId} {
  user_id: string,
  name: string,
  description: string,
  cover_image: string,
  
  documents: [{
    document_id: string,
    order: number,
    added_at: timestamp
  }],
  
  sharing: {
    visibility: "private|family|public",
    collaborators: [{
      user_id: string,
      role: "viewer|editor|admin",
      added_at: timestamp
    }]
  },
  
  created_at: timestamp,
  updated_at: timestamp
}
```

#### Family Groups Collection
```javascript
family_groups/{groupId} {
  name: string,
  owner_id: string,
  
  members: [{
    user_id: string,
    role: "admin|member",
    joined_at: timestamp
  }],
  
  shared_collections: [collectionId],
  
  settings: {
    auto_share_new_documents: boolean,
    require_approval_for_edits: boolean
  },
  
  created_at: timestamp,
  updated_at: timestamp
}
```

### Local SQLite Schema

```sql
-- Offline document cache
CREATE TABLE documents (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL,
  type TEXT NOT NULL,
  status TEXT NOT NULL,
  text TEXT,
  audio_path TEXT,
  metadata JSON,
  sync_status TEXT DEFAULT 'pending',
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL
);

-- Offline queue for sync
CREATE TABLE sync_queue (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  operation TEXT NOT NULL, -- 'create', 'update', 'delete'
  resource_type TEXT NOT NULL, -- 'document', 'collection', etc
  resource_id TEXT NOT NULL,
  payload JSON,
  retry_count INTEGER DEFAULT 0,
  created_at INTEGER NOT NULL
);

-- Recent playback history
CREATE TABLE playback_history (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  document_id TEXT NOT NULL,
  position_seconds REAL NOT NULL,
  completed BOOLEAN DEFAULT FALSE,
  last_played_at INTEGER NOT NULL
);
```

## Security & Privacy Considerations

### Data Protection

1. **End-to-End Encryption**
   - Documents encrypted at rest using AES-256
   - TLS 1.3 for all data in transit
   - Client-side encryption option for ultra-sensitive documents

2. **Authentication & Authorization**
   - Firebase Authentication with multi-factor support
   - OAuth2 integration (Google, Apple, Facebook)
   - Role-based access control (RBAC)
   - Session management with refresh tokens

3. **Privacy Features**
   - On-device OCR processing by default
   - Optional cloud processing with explicit consent
   - Data retention policies (auto-delete after X days)
   - GDPR-compliant data export and deletion

### Security Implementation

```dart
class SecurityManager {
  // Document encryption before upload
  Future<EncryptedDocument> encryptDocument(
    Document doc,
    EncryptionKey userKey
  ) async {
    // Generate document-specific key
    final docKey = await _generateKey();
    
    // Encrypt document content
    final encryptedContent = await _aesEncrypt(
      doc.content,
      docKey
    );
    
    // Encrypt document key with user's key
    final encryptedKey = await _rsaEncrypt(
      docKey,
      userKey
    );
    
    return EncryptedDocument(
      content: encryptedContent,
      encryptedKey: encryptedKey
    );
  }
}
```

### Compliance Requirements

1. **HIPAA Compliance** (for healthcare partnerships)
   - BAA agreements with cloud providers
   - Audit logging for all data access
   - Encryption requirements met

2. **COPPA Compliance**
   - Age verification for users under 13
   - Parental consent workflows
   - Limited data collection for minors

3. **Accessibility Standards**
   - WCAG 2.1 AA compliance
   - Screen reader support
   - High contrast mode
   - Adjustable font sizes

## Scalability Considerations

### Horizontal Scaling Strategy

1. **Microservices Architecture**
   ```
   ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
   │   OCR       │     │    Audio    │     │  Document   │
   │  Service    │     │   Service   │     │   Service   │
   │ (Cloud Run) │     │ (Cloud Run) │     │ (Cloud Run) │
   └─────────────┘     └─────────────┘     └─────────────┘
          │                    │                    │
          └────────────────────┴────────────────────┘
                              │
                    ┌─────────────────┐
                    │   Cloud Tasks    │
                    │  (Job Queuing)   │
                    └─────────────────┘
   ```

2. **Auto-scaling Policies**
   - CPU-based scaling (target 70% utilization)
   - Request-based scaling (max 1000 concurrent requests)
   - Geographic distribution across regions

3. **Caching Strategy**
   - CDN for static assets and audio files
   - Redis for session and API response caching
   - Edge caching for frequently accessed documents

### Performance Optimization

1. **Image Processing Pipeline**
   ```dart
   class ImageOptimizer {
     // Progressive enhancement for large images
     Future<ProcessedImage> optimizeForOCR(
       ImageFile original
     ) async {
       // Resize if needed (max 4096x4096)
       final resized = await _smartResize(original);
       
       // Enhance contrast and sharpness
       final enhanced = await _enhanceForText(resized);
       
       // Convert to grayscale for better OCR
       final grayscale = await _toGrayscale(enhanced);
       
       // Compress without losing text quality
       return await _optimizeSize(grayscale);
     }
   }
   ```

2. **Audio Streaming**
   - Adaptive bitrate streaming
   - Preloading next document in collection
   - Offline caching of recently played audio

### Load Handling

1. **Peak Load Management**
   - Cloud Tasks for async processing
   - Priority queues for premium users
   - Graceful degradation for free tier

2. **Database Optimization**
   - Firestore composite indexes for common queries
   - Sharding strategy for user data
   - Read replicas for analytics queries

## Technical Challenges and Solutions

### Challenge 1: Handwriting Recognition Accuracy

**Problem**: OCR accuracy drops to 60-70% for cursive or aged documents

**Solution**:
1. **Multi-Engine Approach**
   - Primary: Google ML Kit (fast, on-device)
   - Secondary: Cloud Vision API (more accurate)
   - Tertiary: Specialized services (Transkribus for historical)

2. **Confidence-Based Processing**
   ```dart
   if (confidence < 0.6) {
     // Offer manual transcription service
     // Or crowdsource to family members
   } else if (confidence < 0.8) {
     // Highlight low-confidence words for review
   }
   ```

3. **Interactive Correction UI**
   - Side-by-side image and text view
   - Word-level confidence highlighting
   - Smart suggestions based on context

### Challenge 2: Voice Quality for Emotional Content

**Problem**: Standard TTS lacks emotional nuance for personal letters

**Solution**:
1. **Voice Selection Algorithm**
   - Analyze document sentiment
   - Match voice characteristics to content
   - User preference learning

2. **SSML Enhancement**
   ```xml
   <speak>
     <prosody rate="slow" pitch="+2st">
       My dearest <emphasis>Sarah</emphasis>,
     </prosody>
     <break time="500ms"/>
     <prosody rate="medium" volume="soft">
       I miss you more than words can say...
     </prosody>
   </speak>
   ```

3. **Future: Voice Cloning**
   - Record family member reading sample
   - Train custom voice model
   - Apply to historical documents

### Challenge 3: Storage Costs at Scale

**Problem**: High-resolution images and audio files consume significant storage

**Solution**:
1. **Intelligent Compression**
   - HEIC/WebP for images (50% smaller)
   - Opus codec for audio (better than MP3)
   - Progressive quality tiers

2. **Lifecycle Management**
   - Hot storage for recent documents
   - Cold storage for archives
   - Automatic compression after 30 days

3. **User Storage Quotas**
   ```dart
   class StorageManager {
     final quotas = {
       'free': 1 * GB,
       'premium': 50 * GB,
       'family': 200 * GB
     };
     
     Future<bool> checkQuota(User user, int sizeBytes) async {
       final used = await getUsedStorage(user);
       final limit = quotas[user.subscription.tier];
       return (used + sizeBytes) <= limit;
     }
   }
   ```

### Challenge 4: Offline Functionality

**Problem**: Users need access to documents without internet

**Solution**:
1. **Selective Sync**
   - Smart caching of frequently accessed documents
   - Predictive preloading based on usage patterns
   - User-defined offline collections

2. **Conflict Resolution**
   - Last-write-wins for simple conflicts
   - Three-way merge for text corrections
   - User prompts for complex conflicts

3. **Progressive Web App Features**
   - Service workers for offline access
   - Background sync when connection returns
   - IndexedDB for large offline storage

## Implementation Phases

### Phase 1: MVP (Months 1-3)
- Basic photo capture and OCR
- Simple text-to-speech conversion
- Local storage only
- iOS launch

### Phase 2: Core Features (Months 4-6)
- Cloud sync and backup
- Family sharing
- Android launch
- Premium subscription

### Phase 3: Enhancement (Months 7-9)
- Multiple OCR engines
- Voice customization
- Collections and organization
- Offline mode

### Phase 4: Scale (Months 10-12)
- B2B partnerships
- Voice cloning beta
- International expansion
- API for third-party integration

## Monitoring and Analytics

### Key Metrics
1. **Technical Metrics**
   - OCR accuracy rates by document type
   - Audio generation time
   - API response times
   - Error rates

2. **Business Metrics**
   - Documents processed per user
   - Conversion rate (free to paid)
   - Audio playback minutes
   - Family sharing adoption

3. **Infrastructure Metrics**
   - Cloud costs per user
   - Storage utilization
   - CDN cache hit rates
   - Database query performance

### Monitoring Stack
- **Application Performance**: Firebase Performance
- **Error Tracking**: Sentry
- **Analytics**: Google Analytics + Mixpanel
- **Infrastructure**: Google Cloud Monitoring
- **User Feedback**: In-app feedback widget

## Conclusion

This architecture provides a solid foundation for FamilyTales to capture the $50M market opportunity in handwritten document preservation. The privacy-first approach with on-device OCR, combined with cloud-based audio generation and family sharing features, creates a unique value proposition. The modular architecture allows for rapid iteration while maintaining scalability and security as the user base grows.

The technical challenges around OCR accuracy and voice quality are addressable through the multi-engine approach and continuous improvement based on user feedback. With the phased implementation plan, FamilyTales can launch quickly with core features and evolve based on market response.