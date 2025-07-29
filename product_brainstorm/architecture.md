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

FamilyTales is a cross-platform application that converts handwritten documents (letters, memoirs, journals) into high-quality audio format with instant global family sharing. The architecture leverages Flutter for iOS/Android/Web/Desktop development, self-hosted olmOCR for privacy-focused text recognition, and HLS streaming for efficient audio distribution to family members worldwide.

### Key Architecture Principles
- **Privacy-First**: Self-hosted olmOCR keeps all document processing in-house
- **One-Scan-Many-Listen**: Single family member scans, entire family enjoys instantly via HLS streaming
- **Global Family Connectivity**: Smart CDN distribution with offline download support
- **Collaborative Intelligence**: Family members help improve OCR accuracy together
- **Easy Onboarding**: QR codes, simple invite links, and progressive account creation

## System Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                     Client Applications                          │
│            (Flutter - iOS/Android/Web/Desktop)                   │
├─────────────────────────────────────────────────────────────────┤
│                    Rust Axum API Backend                         │
│         (Business Logic, Auth, Job Orchestration)                │
├─────────────────┬─────────────────┬────────────────────────────┤
│   PostgreSQL     │     Redis       │         MinIO              │
│   (User Data,    │  (Cache, Job    │   (Object Storage for      │
│    Metadata)     │     State)      │    Images, Audio)          │
├─────────────────┴─────────────────┴────────────────────────────┤
│                       RabbitMQ Job Queue                         │
│            (Distributes processing to local PC)                  │
├─────────────────────────────────────────────────────────────────┤
│                  Local PC Processing Worker                      │
│                  (Powerful Desktop Machine)                      │
├─────────────────┬─────────────────┬────────────────────────────┤
│   olmOCR Engine  │  TTS Generation │   HLS Segmentation         │
│  (GPU Accelerated)│  (High Quality) │   & Optimization           │
└─────────────────┴─────────────────┴────────────────────────────┘
```

### High-Level Data Flow

1. **Document Capture**: User photographs handwritten document
2. **Pre-processing**: Image enhancement and optimization  
3. **OCR Processing**: Secure upload to self-hosted olmOCR server
4. **Text Verification**: User reviews and corrects OCR output
5. **Audio Generation**: Cloud-based TTS conversion with HLS segmentation
6. **Global Distribution**: 
   - HLS streams uploaded to CDN
   - Family members notified instantly
   - Audio available for streaming or download
7. **Flexible Playback**:
   - Stream via HLS for instant access
   - Download for offline playback
   - Automatic quality adjustment based on connection

## Core Components

### 1. OCR Engine

**Primary Technology**: Self-hosted olmOCR
- Dedicated on-premise OCR server for maximum privacy
- Optimized for handwritten document recognition
- No data leaves our infrastructure
- Customizable training for family-specific handwriting patterns

**Architecture Benefits**:
- Complete data sovereignty
- No per-request costs
- Ability to fine-tune models on user corrections
- Consistent performance without API rate limits

**Key Features**:
```dart
class OCREngine {
  // Self-hosted olmOCR with intelligent routing
  Future<OCRResult> processImage(ImageFile image) async {
    // Upload to our olmOCR server
    final olmResult = await _processWithOlmOCR(image);
    
    // Apply family-specific model if available
    if (await _hasFamilyModel(userId)) {
      return await _enhanceWithFamilyModel(olmResult, userId);
    }
    
    return olmResult;
  }
  
  // Train family-specific models from corrections
  Future<void> updateFamilyModel(String userId, List<Correction> corrections) async {
    await _olmOCRService.trainModel(
      modelId: 'family_$userId',
      trainingData: corrections
    );
  }
}
```

### 2. Text-to-Speech Engine & Mux Streaming

**Primary Service**: Google Cloud Text-to-Speech with Mux Distribution
- 380+ voices across 50+ languages
- WaveNet and Neural2 voices for natural sound
- SSML support for pronunciation customization
- Mux handles all HLS streaming complexity

**Voice Options**:
- Standard voices for basic conversion
- Premium WaveNet voices for emotional content
- Voice cloning capability (future feature with ElevenLabs)

**Audio Processing & Mux Integration**:
```rust
use mux_rust_sdk::{MuxClient, CreateAssetRequest};

pub struct AudioProcessor {
    tts_client: GoogleTtsClient,
    mux_client: MuxClient,
}

impl AudioProcessor {
    pub async fn process_document(&self, document: &Document) -> Result<MuxAsset> {
        // Generate audio from text
        let audio_data = self.tts_client
            .synthesize(SynthesizeRequest {
                text: &document.ocr_text,
                voice: document.voice_settings,
                audio_config: AudioConfig {
                    audio_encoding: AudioEncoding::Mp3,
                    speaking_rate: document.speed,
                },
            })
            .await?;
        
        // Upload to Mux - they handle everything else
        let asset = self.mux_client
            .create_asset(CreateAssetRequest {
                input: audio_data,
                playback_policy: vec!["public"],
                audio_only: true,
                normalize_audio: true,
                master_access: "temporary", // For downloads
                mp4_support: "standard",
            })
            .await?;
        
        // Mux automatically creates:
        // - HLS streams with multiple bitrates
        // - Global CDN distribution
        // - Playback URLs for each quality
        
        Ok(MuxAsset {
            playback_id: asset.playback_ids[0].id,
            hls_url: format!("https://stream.mux.com/{}.m3u8", asset.playback_ids[0].id),
            download_url: asset.master_access_url,
            duration: asset.duration,
        })
    }
}
```

**Mux Advantages**:
- **Zero HLS Complexity**: Mux handles all segmentation and packaging
- **Instant Global Delivery**: Built-in CDN with 100+ edge locations
- **Adaptive Bitrates**: Automatically adjusts quality based on connection
- **Audio Normalization**: Consistent volume across all documents
- **Analytics Included**: Track plays, completion rates, bandwidth usage
- **Direct Playback URLs**: No need to manage streaming infrastructure

### 3. Synchronized Playback System

**Visual-Audio Synchronization**:
```dart
class SyncPlaybackEngine {
  // Synchronized display of original document during audio playback
  Future<SyncedPlayback> prepareSyncedContent(
    Document document
  ) async {
    // Generate word-level timestamps
    final wordTimings = await _generateWordTimings(
      document.ocr.text,
      document.audio.duration_seconds
    );
    
    // Create visual highlights for each word/phrase
    final visualMarkers = await _createVisualMarkers(
      document.image,
      document.ocr.boundingBoxes,
      wordTimings
    );
    
    // Prepare synchronized display data
    return SyncedPlayback(
      audioUrl: document.audio.hls_playlist_path,
      imageUrl: document.image.storage_path,
      syncData: SyncData(
        wordTimings: wordTimings,
        visualMarkers: visualMarkers,
        pageBreaks: document.ocr.pageBreaks
      )
    );
  }
  
  // Real-time highlight during playback
  Widget buildSyncedView(SyncedPlayback playback) {
    return AudioImageSync(
      onTimeUpdate: (currentTime) {
        // Highlight current word/line being read
        _highlightCurrentText(currentTime);
        // Pan/zoom to keep current text visible
        _adjustViewport(currentTime);
      }
    );
  }
}
```

**Key Features**:
- **Follow-Along Reading**: Highlighted text moves with audio playback
- **Smart Viewport**: Auto-pan and zoom to keep current text visible
- **Manual Override**: Users can zoom/pan while audio continues
- **Error Recovery**: Visual cues when OCR text doesn't match audio

### 4. Document Management System

**Core Features**:
- Multi-format support (handwritten documents, photos, memorabilia)
- Document categorization (letters, memoirs, journals, photos, recipes)
- Metadata extraction (dates, recipients, authors)
- Version control for corrections
- Real-time family sharing across continents
- One-scan-many-listen architecture

**Enhanced Media Support**:
- **Photo Collections**: Upload and organize family photos
- **Mixed Media Documents**: Link photos to specific document pages
- **Timestamp Linking**: Connect images to audio timestamps
- **Visual Stories**: Create photo slideshows with audio narration

**Geographic Family Sharing**:
- **Instant Global Access**: One family member scans in USA, relatives in Europe/Asia listen immediately
- **Smart CDN Distribution**: Audio files cached at edge locations near family members
- **Time Zone Aware Notifications**: Respectful alerts based on recipient's local time
- **Live Listening Sessions**: Family members can listen together with synchronized playback

**Storage Architecture**:
- Local SQLite for offline access
- Cloud Firestore for real-time sync
- Cloud Storage for images, documents, and master audio files
- Global CDN with HLS streaming for instant playback
- Edge caching in regions where family members are located

### 4. Family Sharing Architecture

**One-Scan-Many-Listen Model**:
```dart
class FamilySharingService {
  // Scan once, share with entire family instantly
  Future<void> shareWithFamily(
    Document document,
    String familyGroupId
  ) async {
    // Get all family members and their locations
    final members = await _getFamilyMembers(familyGroupId);
    final memberLocations = await _getMemberLocations(members);
    
    // Pre-cache content at edge locations
    await _preCacheAtEdgeLocations(
      document.audioUrl,
      memberLocations
    );
    
    // Notify based on time zones
    await _smartNotify(members, document);
    
    // Enable offline download permissions
    await _enableOfflineAccess(members, document);
  }
  
  // Smart download management
  Future<void> downloadForOffline(
    Document document,
    User user
  ) async {
    // Download appropriate quality based on device storage
    final quality = await _determineOptimalQuality(user.device);
    
    // Download with progress tracking
    await _downloadManager.download(
      url: document.getHLSVariant(quality),
      onProgress: (progress) => _updateUI(progress)
    );
    
    // Encrypt for local storage
    await _encryptForOfflineStorage(document, user);
  }
}
```

**Geographic Distribution Features**:
- **Smart Pre-caching**: Content cached near family members before they access
- **Bandwidth Optimization**: Lower quality for mobile data, high quality for WiFi
- **Offline Sync**: Downloaded content syncs playback position across devices
- **Family Analytics**: See who's listened and from where (privacy-respecting)

## Technology Stack

### Mobile Application

**Framework**: Flutter 3.x
- Single codebase for iOS, Android, Web, and Desktop
- Native performance with platform-specific optimizations
- Rich widget library for elderly-friendly UI
- PWA support for instant web access

**Key Dependencies**:
```yaml
dependencies:
  # Core functionality
  camera: ^0.10.5
  image_picker: ^1.0.4
  http: ^1.1.0  # For olmOCR API calls
  
  # Audio handling with HLS support
  just_audio: ^0.9.34  # Native HLS support on iOS/Android/Web
  audio_service: ^0.18.10  # Background playback
  just_audio_background: ^0.0.1  # Background audio metadata
  
  # Storage and sync
  sqflite: ^2.3.0
  cloud_firestore: ^4.13.0
  firebase_storage: ^11.5.0
  
  # Real-time family features
  firebase_messaging: ^14.7.0  # Push notifications
  web_socket_channel: ^2.4.0  # Live listening sessions
  
  # UI/UX and State Management
  flutter_riverpod: ^2.4.0  # Better than Provider - compile-safe, testable
  riverpod_annotation: ^2.3.0  # Code generation for providers
  go_router: ^12.1.0
  flutter_animate: ^4.3.0
```

### Backend Services

**Primary Backend**: Rust with Axum Framework
- High-performance async web framework
- Type-safe API development
- Excellent concurrency for handling multiple requests
- Low memory footprint for cost-effective hosting

**Rust Backend Stack**:
```toml
[dependencies]
axum = "0.7"
tokio = { version = "1", features = ["full"] }
tower = "0.4"
tower-http = { version = "0.5", features = ["cors", "compression"] }
sqlx = { version = "0.7", features = ["postgres", "runtime-tokio-rustls"] }
redis = { version = "0.24", features = ["tokio-comp"] }
serde = { version = "1.0", features = ["derive"] }
uuid = { version = "1.6", features = ["v4", "serde"] }
jsonwebtoken = "9.2"
reqwest = { version = "0.11", features = ["json"] }
tracing = "0.1"
tracing-subscriber = "0.3"

# Job queue management
lapin = "2.3" # RabbitMQ client for job distribution
```

**Job Processing Architecture**:
```rust
// Job queue system for delegating heavy processing
pub struct ProcessingJob {
    pub id: Uuid,
    pub job_type: JobType,
    pub priority: Priority,
    pub payload: JobPayload,
}

pub enum JobType {
    OcrProcessing { 
        document_id: Uuid,
        image_url: String,
        language_hints: Vec<String>,
    },
    AudioGeneration {
        document_id: Uuid,
        text: String,
        voice_settings: VoiceSettings,
    },
    HlsSegmentation {
        audio_file: String,
        output_settings: HlsSettings,
    }
}
```

**Infrastructure Components**:
- **PostgreSQL**: Primary database for relational data
- **Redis**: Session cache and job queue state
- **RabbitMQ**: Job queue for communication with local PC
- **Mux**: Video/audio storage, streaming, and processing
- **Nginx**: Reverse proxy and static file serving
- **Local PC Worker**: Powerful machine running job processor

### Local PC Worker Service

**Worker Implementation** (Rust):
```rust
// Local PC worker that polls for jobs
pub struct LocalWorker {
    rabbit_conn: lapin::Connection,
    ocr_engine: OlmOcrEngine,
    tts_engine: TtsEngine,
    hls_processor: HlsProcessor,
}

impl LocalWorker {
    pub async fn start(&mut self) -> Result<()> {
        let channel = self.rabbit_conn.create_channel().await?;
        let consumer = channel
            .basic_consume(
                "processing_queue",
                "local_pc_worker",
                BasicConsumeOptions::default(),
            )
            .await?;
        
        while let Some(delivery) = consumer.next().await {
            let job: ProcessingJob = serde_json::from_slice(&delivery.data)?;
            
            match job.job_type {
                JobType::OcrProcessing { .. } => {
                    self.process_ocr(job).await?;
                },
                JobType::AudioGeneration { .. } => {
                    self.process_audio(job).await?;
                },
                JobType::HlsSegmentation { .. } => {
                    self.process_hls(job).await?;
                }
            }
            
            delivery.ack(BasicAckOptions::default()).await?;
        }
        Ok(())
    }
}
```

**GPU Acceleration**:
- CUDA support for olmOCR processing
- Hardware-accelerated audio encoding
- Parallel job processing capabilities

## API Design

### Rust Axum API Structure

```rust
use axum::{
    Router,
    routing::{get, post, patch, delete},
    middleware,
};

pub fn create_app() -> Router {
    Router::new()
        // Public routes
        .route("/health", get(handlers::health_check))
        .route("/auth/register", post(handlers::auth::register))
        .route("/auth/login", post(handlers::auth::login))
        
        // Protected routes
        .nest("/api/v1", api_routes())
        .layer(middleware::from_fn(auth::verify_token))
}

fn api_routes() -> Router {
    Router::new()
        // Document routes
        .route("/documents", post(handlers::documents::create))
        .route("/documents/:id", get(handlers::documents::get))
        .route("/documents/:id", patch(handlers::documents::update))
        .route("/documents/:id/process", post(handlers::documents::process))
        
        // Family routes
        .route("/family/invite", post(handlers::family::create_invite))
        .route("/family/join/:code", post(handlers::family::join))
        
        // Streaming routes
        .route("/stream/:id/playlist.m3u8", get(handlers::stream::hls_playlist))
        .route("/stream/:id/:segment", get(handlers::stream::hls_segment))
}
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
# Create family invite link (no email required)
POST /family/invites/create-link
Body: {
  expires_in_days: 7,
  max_uses: 10,
  role: "viewer|editor|admin"
}
Response: {
  invite_link: "https://familytales.app/join/ABC123",
  qr_code: "base64_image",
  share_message: "Join our family on FamilyTales!"
}

# Join via invite link/code
POST /family/join
Body: {
  invite_code: "ABC123"
}

# Share app with family (deep link to app store + invite)
POST /family/share-app
Body: {
  method: "sms|email|whatsapp",
  recipient: "phone_or_email",
  include_invite: true
}
Response: {
  share_link: "https://familytales.app/get?family=ABC123",
  message: "Grandma shared family memories with you! Download FamilyTales to listen: [link]"
}

# Get HLS stream for document
GET /documents/{documentId}/stream.m3u8
Response: HLS playlist file

# Download for offline
POST /documents/{documentId}/download
Body: {
  quality: "low|medium|high"
}
Response: {
  download_url: "signed_url",
  expires_at: "ISO8601",
  file_size_mb: 12.5
}
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

### PostgreSQL Schema

#### Users Table
```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    display_name VARCHAR(255),
    
    -- Subscription info
    subscription_tier VARCHAR(20) DEFAULT 'free' CHECK (subscription_tier IN ('free', 'premium', 'family_legacy')),
    subscription_status VARCHAR(20) DEFAULT 'active',
    subscription_expires_at TIMESTAMPTZ,
    stripe_customer_id VARCHAR(255),
    
    -- Preferences
    default_voice VARCHAR(50),
    default_language VARCHAR(10) DEFAULT 'en',
    auto_enhance_images BOOLEAN DEFAULT true,
    
    -- Usage tracking
    documents_processed INTEGER DEFAULT 0,
    audio_minutes_generated DECIMAL(10,2) DEFAULT 0,
    storage_used_mb DECIMAL(10,2) DEFAULT 0,
    
    -- Timestamps
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    last_login_at TIMESTAMPTZ
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_subscription ON users(subscription_tier, subscription_status);
```

#### Documents Table
```sql
CREATE TABLE documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    type VARCHAR(20) CHECK (type IN ('letter', 'memoir', 'journal', 'recipe', 'photo_annotation')),
    status VARCHAR(20) DEFAULT 'uploading' CHECK (status IN ('uploading', 'processing', 'ready', 'error')),
    
    -- Organization
    folder_path VARCHAR(500),
    
    -- Image data (stored in Mux)
    image_mux_asset_id VARCHAR(255),
    image_mux_playback_id VARCHAR(255),
    image_width INTEGER,
    image_height INTEGER,
    image_size_bytes BIGINT,
    
    -- OCR results
    ocr_text TEXT,
    ocr_confidence DECIMAL(3,2),
    ocr_language VARCHAR(10),
    ocr_processing_time_ms INTEGER,
    ocr_engine VARCHAR(50),
    ocr_bounding_boxes JSONB, -- Array of word boundaries
    ocr_page_breaks INTEGER[], -- Character positions
    
    -- Audio data (Mux handles all streaming)
    audio_mux_asset_id VARCHAR(255),
    audio_mux_playback_id VARCHAR(255),
    audio_duration_seconds DECIMAL(10,2),
    audio_voice_id VARCHAR(50),
    audio_mux_download_url TEXT,
    audio_size_bytes BIGINT,
    
    -- Metadata
    title VARCHAR(500),
    author VARCHAR(255),
    recipient VARCHAR(255),
    date_written DATE,
    
    -- Sharing
    visibility VARCHAR(20) DEFAULT 'private' CHECK (visibility IN ('private', 'family', 'public')),
    share_link VARCHAR(100) UNIQUE,
    download_allowed BOOLEAN DEFAULT true,
    listen_count INTEGER DEFAULT 0,
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Separate tables for many-to-many relationships
CREATE TABLE document_collections (
    document_id UUID REFERENCES documents(id) ON DELETE CASCADE,
    collection_id UUID REFERENCES collections(id) ON DELETE CASCADE,
    added_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (document_id, collection_id)
);

CREATE TABLE document_tags (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id UUID REFERENCES documents(id) ON DELETE CASCADE,
    tag_type VARCHAR(20) CHECK (tag_type IN ('people', 'topics', 'locations', 'time_periods', 'custom')),
    tag_value VARCHAR(255),
    confidence DECIMAL(3,2),
    UNIQUE(document_id, tag_type, tag_value)
);

CREATE TABLE document_corrections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id UUID REFERENCES documents(id) ON DELETE CASCADE,
    original_text TEXT,
    corrected_text TEXT,
    position_start INTEGER,
    position_end INTEGER,
    corrected_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE document_shares (
    document_id UUID REFERENCES documents(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    shared_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (document_id, user_id)
);

CREATE INDEX idx_documents_user ON documents(user_id);
CREATE INDEX idx_documents_status ON documents(status);
CREATE INDEX idx_documents_folder ON documents(folder_path);
```

#### Collections Table
```sql
CREATE TABLE collections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    cover_image_mux_playback_id VARCHAR(255),
    visibility VARCHAR(20) DEFAULT 'private' CHECK (visibility IN ('private', 'family', 'public')),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE collection_collaborators (
    collection_id UUID REFERENCES collections(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    role VARCHAR(20) CHECK (role IN ('viewer', 'editor', 'admin')),
    added_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (collection_id, user_id)
);

CREATE INDEX idx_collections_user ON collections(user_id);
```

#### Family Groups Table
```sql
CREATE TABLE family_groups (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    owner_id UUID REFERENCES users(id) ON DELETE CASCADE,
    invite_code VARCHAR(10) UNIQUE,
    
    -- Settings
    auto_share_new_documents BOOLEAN DEFAULT false,
    require_approval_for_edits BOOLEAN DEFAULT false,
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE family_members (
    family_group_id UUID REFERENCES family_groups(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    role VARCHAR(20) CHECK (role IN ('admin', 'member')),
    joined_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (family_group_id, user_id)
);

CREATE INDEX idx_family_groups_owner ON family_groups(owner_id);
CREATE INDEX idx_family_groups_invite ON family_groups(invite_code);
```

#### Folders Table
```sql
CREATE TABLE folders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    path VARCHAR(1000) NOT NULL,
    parent_folder_id UUID REFERENCES folders(id) ON DELETE CASCADE,
    family_group_id UUID REFERENCES family_groups(id) ON DELETE CASCADE,
    
    -- Customization
    icon VARCHAR(50),
    color VARCHAR(7), -- Hex color
    description TEXT,
    
    -- Auto-organization rules stored as JSONB
    auto_tag_rules JSONB,
    
    -- Stats (can be computed but cached for performance)
    document_count INTEGER DEFAULT 0,
    photo_count INTEGER DEFAULT 0,
    total_duration_minutes DECIMAL(10,2) DEFAULT 0,
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    created_by UUID REFERENCES users(id),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Folder permissions
CREATE TABLE folder_permissions (
    folder_id UUID REFERENCES folders(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    permission VARCHAR(20) CHECK (permission IN ('add', 'organize', 'delete')),
    PRIMARY KEY (folder_id, user_id, permission)
);

CREATE INDEX idx_folders_path ON folders(path);
CREATE INDEX idx_folders_family ON folders(family_group_id);
```

#### Photos Table
```sql
CREATE TABLE photos (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    family_group_id UUID REFERENCES family_groups(id),
    
    -- Organization
    folder_path VARCHAR(1000),
    
    -- Image data (Mux stores images too)
    image_mux_asset_id VARCHAR(255) NOT NULL,
    image_mux_playback_id VARCHAR(255) NOT NULL,
    thumbnail_mux_playback_id VARCHAR(255),
    width INTEGER,
    height INTEGER,
    format VARCHAR(10),
    size_bytes BIGINT,
    
    -- Metadata
    title VARCHAR(500),
    description TEXT,
    date_taken DATE,
    location_name VARCHAR(500),
    location_lat DECIMAL(10,8),
    location_lng DECIMAL(11,8),
    
    -- AI detection results
    detected_text TEXT,
    scene_description TEXT,
    
    -- Narration (optional)
    narration_mux_asset_id VARCHAR(255),
    narration_mux_playback_id VARCHAR(255),
    narration_duration_seconds DECIMAL(10,2),
    narration_transcript TEXT,
    narrator_id UUID REFERENCES users(id),
    
    -- Sharing
    visibility VARCHAR(20) DEFAULT 'private' CHECK (visibility IN ('private', 'family', 'public')),
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Photo to document links
CREATE TABLE photo_document_links (
    photo_id UUID REFERENCES photos(id) ON DELETE CASCADE,
    document_id UUID REFERENCES documents(id) ON DELETE CASCADE,
    page_number INTEGER,
    timestamp_seconds DECIMAL(10,2),
    relationship VARCHAR(20) CHECK (relationship IN ('illustration', 'reference', 'companion')),
    PRIMARY KEY (photo_id, document_id)
);

-- Photo tags (same structure as document tags)
CREATE TABLE photo_tags (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    photo_id UUID REFERENCES photos(id) ON DELETE CASCADE,
    tag_type VARCHAR(20) CHECK (tag_type IN ('people', 'topics', 'locations', 'time_periods', 'custom')),
    tag_value VARCHAR(255),
    confidence DECIMAL(3,2),
    UNIQUE(photo_id, tag_type, tag_value)
);

-- Detected faces
CREATE TABLE photo_faces (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    photo_id UUID REFERENCES photos(id) ON DELETE CASCADE,
    person_name VARCHAR(255),
    confidence DECIMAL(3,2),
    bbox_x INTEGER,
    bbox_y INTEGER,
    bbox_width INTEGER,
    bbox_height INTEGER
);

CREATE INDEX idx_photos_user ON photos(user_id);
CREATE INDEX idx_photos_family ON photos(family_group_id);
CREATE INDEX idx_photos_folder ON photos(folder_path);
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

**Problem**: OCR accuracy varies significantly with handwriting quality

**Solution**:
1. **Self-Hosted olmOCR Advantages**
   - Train custom models on family-specific handwriting
   - Improve accuracy over time with corrections
   - No API limits or costs per document

2. **Family Collaborative Correction**
   ```dart
   class CollaborativeCorrection {
     // Family members can help correct difficult passages
     Future<void> requestFamilyHelp(
       Document doc,
       List<LowConfidenceRegion> regions
     ) async {
       // Notify family members who might recognize handwriting
       await _notifyFamilyExperts(doc.authorId, regions);
       
       // Aggregate corrections from multiple family members
       final corrections = await _collectCorrections();
       
       // Update olmOCR training data
       await _olmOCR.updateTrainingData(corrections);
     }
   }
   ```

3. **Interactive Correction UI**
   - Side-by-side image and text view
   - Word-level confidence highlighting
   - Smart suggestions based on family's writing patterns

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

### Challenge 5: Easy Family Onboarding for Non-Tech-Savvy Users

**Problem**: Elderly users struggle with app downloads and account creation

**Solution**:
1. **One-Click Join Links**
   ```dart
   class EasyOnboarding {
     // Generate simple join codes
     String generateFamilyCode() {
       // 6-digit codes easy to share over phone
       return _generateNumericCode(6); // e.g., "428-319"
     }
     
     // QR codes for in-person sharing
     Future<QRCode> generateQRInvite(Family family) async {
       final deepLink = DeepLink(
         action: 'join_family',
         familyId: family.id,
         skipOnboarding: true
       );
       return QRCode.generate(deepLink);
     }
   }
   ```

2. **Progressive Onboarding**
   - Join family first, create account later
   - Listen to shared memories immediately
   - Gradual feature discovery

3. **Multi-Channel Invites**
   - SMS with direct app store links
   - WhatsApp integration
   - Email with large "Join Family" button
   - Physical QR code cards for reunions

## MVP Cost Analysis: Third-Party vs Self-Hosted

### Option 1: Third-Party Services (MVP Approach)

**OCR Services Cost Comparison**:
```
Google Cloud Vision API:
- First 1,000 units/month: Free
- 1,001-5,000,000 units/month: $1.50 per 1,000
- Average: ~500 chars per document = ~$0.0015 per document

AWS Textract:
- $1.50 per 1,000 pages
- Average: $0.0015 per document

Azure Computer Vision:
- $1.00 per 1,000 transactions
- Average: $0.001 per document

HandwritingOCR.com:
- $30-300/month plans
- API: ~$0.01 per page
```

**Text-to-Speech Cost Comparison**:
```
Google Cloud TTS:
- First 1M chars/month: Free (Standard voices)
- WaveNet voices: $16 per 1M chars
- Average doc (2,000 chars): $0.032 per document

Amazon Polly:
- Standard: $4 per 1M chars
- Neural: $16 per 1M chars
- Average: $0.008-0.032 per document

Azure Speech:
- Standard: $4 per 1M chars
- Neural: $16 per 1M chars
```

**MVP Monthly Cost Projection** (1,000 active users, 10 docs/user/month):
- OCR: 10,000 docs × $0.002 = $20
- TTS: 10,000 docs × $0.016 = $160
- Storage (S3): ~$50
- **Total: ~$230/month + infrastructure**

### Option 2: Self-Hosted with Local PC

**One-Time Setup**:
- Already have powerful PC: $0
- olmOCR setup: Free (open source)
- TTS models: Free (open source)
- Time investment: ~40 hours

**Ongoing Costs**:
- Electricity: ~$30/month
- Internet bandwidth: Minimal additional
- Maintenance time: ~5 hours/month
- **Total: ~$30/month + time**

### Recommendation for MVP

**Start with Third-Party Services** for MVP because:
1. **Faster Time to Market**: Launch in weeks vs months
2. **Proven Reliability**: 99.9% uptime SLAs
3. **No DevOps Overhead**: Focus on product-market fit
4. **Scalable**: Handle unexpected viral growth
5. **Low Initial Cost**: ~$230/month is manageable

## Freemium Business Model

### Tier Structure

#### Free Tier (Forever Free)
**Limits**:
- 3 document scans per month
- Standard quality OCR only
- Basic TTS voices (robotic)
- No family sharing
- 7-day storage for audio files
- Watermark on exports

**Purpose**: Let users experience core value, create habit

#### Premium Tier ($9.99/month or $79/year)
**Features**:
- Unlimited document scans
- High-accuracy OCR with corrections
- Premium natural voices
- Family sharing (up to 5 members)
- Unlimited cloud storage
- Download audio files
- Synchronized playback
- Email/chat support

**Target**: Individual users and small families

#### Family Legacy Tier ($19.99/month or $179/year)
**Everything in Premium plus**:
- Unlimited family members
- Voice cloning (preserve grandpa's voice)
- Bulk scanning mode
- API access
- White-label sharing pages
- Priority processing
- Video tutorials for elderly
- Phone support

**Target**: Large families, family historians

### Feature Comparison Implementation

```rust
#[derive(Debug, Clone)]
pub struct UserTier {
    pub tier: TierLevel,
    pub limits: TierLimits,
}

#[derive(Debug, Clone)]
pub struct TierLimits {
    pub monthly_scans: Option<u32>,
    pub family_members: Option<u32>,
    pub storage_days: Option<u32>,
    pub ocr_quality: OcrQuality,
    pub voice_quality: VoiceQuality,
    pub features: HashSet<Feature>,
}

impl TierLimits {
    pub fn free() -> Self {
        Self {
            monthly_scans: Some(3),
            family_members: Some(0),
            storage_days: Some(7),
            ocr_quality: OcrQuality::Basic,
            voice_quality: VoiceQuality::Standard,
            features: HashSet::new(),
        }
    }
    
    pub fn premium() -> Self {
        Self {
            monthly_scans: None, // Unlimited
            family_members: Some(5),
            storage_days: None, // Unlimited
            ocr_quality: OcrQuality::Advanced,
            voice_quality: VoiceQuality::Premium,
            features: hashset![
                Feature::FamilySharing,
                Feature::Downloads,
                Feature::SyncPlayback,
                Feature::CloudStorage,
            ],
        }
    }
}

// Middleware to check tier limits
pub async fn check_tier_limits(
    State(app_state): State<AppState>,
    user: User,
    request: Request,
    next: Next,
) -> Result<Response> {
    let tier_limits = get_user_limits(&user);
    
    // Check monthly scan limit
    if request.uri().path() == "/api/v1/documents/process" {
        let monthly_scans = get_monthly_scan_count(&user).await?;
        if let Some(limit) = tier_limits.monthly_scans {
            if monthly_scans >= limit {
                return Err(ApiError::TierLimitExceeded {
                    limit_type: "monthly_scans",
                    upgrade_url: "/pricing",
                });
            }
        }
    }
    
    Ok(next.run(request).await)
}
```

### Conversion Strategy

**Free → Premium Hooks**:
1. Hit scan limit → "Unlock unlimited scans"
2. Try to share → "Share with family members"
3. Audio expires → "Keep your memories forever"
4. Low quality voice → "Hear in natural voice"

**Premium → Family Legacy Hooks**:
1. Add 6th family member → "Add unlimited family"
2. Want better quality → "Try voice cloning"
3. Need bulk processing → "Save hours with bulk mode"

**Migration Path**:
```
Months 1-3: Third-party services (validate market)
Months 4-6: Hybrid approach (heavy users on local PC)
Months 7+: Full self-hosted (once revenue justifies it)
```

## Implementation Phases

### Phase 1: MVP with Rust Backend (Months 1-3)
**Infrastructure**:
- Rust Axum web server on VPS/Cloud
- PostgreSQL database
- Redis for caching and sessions
- Mux for media storage and streaming
- Third-party OCR (Google Vision) and TTS APIs
- Stripe for payments

**Architecture Setup**:
```rust
// Cargo.toml for MVP
[dependencies]
axum = "0.7"
tokio = { version = "1", features = ["full"] }
sqlx = { version = "0.7", features = ["postgres", "runtime-tokio-rustls", "migrate"] }
redis = { version = "0.24", features = ["tokio-comp", "connection-manager"] }
stripe-rust = "0.13"
reqwest = { version = "0.11", features = ["json"] }
jsonwebtoken = "9.2"
argon2 = "0.5"

# Mux integration (using their REST API)
serde_json = "1.0"
```

**Mux Integration Benefits**:
- Built-in HLS streaming (no manual segmentation needed)
- Automatic quality variants
- Global CDN included
- Audio normalization
- Thumbnail generation
- Usage analytics

**Deployment**:
- Single VPS or small Kubernetes cluster
- Docker containers for easy deployment
- GitHub Actions for CI/CD
- Target: 1,000 beta users

### Phase 2: Hybrid Processing (Months 4-6)
- Add RabbitMQ for job queue
- Set up local PC worker for premium users
- Implement tiered processing (free = cloud, premium = local)
- Add monitoring and analytics
- Cost reduction by 60%
- Target: 10,000 users

### Phase 3: Full Self-Hosted (Months 7-9)
- Complete migration to olmOCR on local PC
- Multiple worker machines for redundancy
- Custom TTS models for better quality
- Advanced caching strategies
- Target: 50,000 users

### Phase 4: Scale & Innovation (Months 10-12)
- Distributed processing network
- Voice cloning implementation
- B2B API marketplace
- International expansion
- Target: 100,000+ users

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