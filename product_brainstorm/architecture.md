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

### 2. Content Processing & Audio Generation

**Concatenated Audio Architecture**:
```rust
pub struct MemoryBookProcessor {
    ocr_engine: Box<dyn OCREngine>,
    tts_client: GoogleTtsClient,
    mux_client: MuxClient,
}

impl MemoryBookProcessor {
    pub async fn process_memory_book(&self, book: &MemoryBook) -> Result<ProcessedBook> {
        let mut segments = Vec::new();
        let mut full_text = String::new();
        let mut current_position = 0.0; // seconds
        
        // Process each content item in order
        for item in &book.items {
            match item {
                ContentItem::HandwrittenDocument(doc) => {
                    let ocr_result = self.ocr_engine.process(&doc.image).await?;
                    let text = ocr_result.text;
                    let text_length = text.len();
                    
                    segments.push(ContentSegment {
                        id: doc.id,
                        content_type: ContentType::HandwrittenText,
                        original_url: doc.image_url,
                        ocr_text: Some(text.clone()),
                        timestamp_start: current_position,
                        timestamp_end: 0.0, // Will be calculated after audio generation
                        page_number: doc.page_number,
                    });
                    
                    full_text.push_str(&text);
                    full_text.push_str("\n\n"); // Natural pause between documents
                }
                ContentItem::DigitalDocument(doc) => {
                    let text = self.extract_text(doc).await?;
                    segments.push(ContentSegment {
                        id: doc.id,
                        content_type: ContentType::DigitalText,
                        original_url: doc.file_url,
                        ocr_text: Some(text.clone()),
                        timestamp_start: current_position,
                        timestamp_end: 0.0,
                        page_number: doc.page_number,
                    });
                    
                    full_text.push_str(&text);
                    full_text.push_str("\n\n");
                }
                ContentItem::Image(img) => {
                    // Images get timestamped but no text
                    segments.push(ContentSegment {
                        id: img.id,
                        content_type: ContentType::Image,
                        original_url: img.image_url,
                        ocr_text: img.caption.clone(),
                        timestamp_start: current_position,
                        timestamp_end: current_position, // Same as start for images
                        page_number: img.page_number,
                    });
                }
            }
        }
        
        // Generate single audio file from concatenated text
        let audio = self.generate_audio(&full_text, &book.voice_settings).await?;
        
        // Calculate actual timestamps based on speech timing
        self.calculate_timestamps(&mut segments, &audio.word_timings);
        
        // Upload to Mux
        let mux_asset = self.upload_to_mux(audio).await?;
        
        Ok(ProcessedBook {
            segments,
            audio_url: mux_asset.hls_url,
            total_duration: mux_asset.duration,
        })
    }
}
```

**Multi-Format Input Support**:
```rust
pub enum ContentItem {
    HandwrittenDocument {
        id: Uuid,
        image_url: String,
        page_number: u32,
    },
    DigitalDocument {
        id: Uuid,
        file_url: String,
        format: DocumentFormat,
        page_number: u32,
    },
    Image {
        id: Uuid,
        image_url: String,
        caption: Option<String>,
        page_number: u32,
    },
}

pub enum DocumentFormat {
    PDF,
    Word,
    GoogleDocs,
    PlainText,
    EPUB,
    HandwrittenImage,
}
```

**Mux Advantages**:
- **Zero HLS Complexity**: Mux handles all segmentation and packaging
- **Instant Global Delivery**: Built-in CDN with 100+ edge locations
- **Adaptive Bitrates**: Automatically adjusts quality based on connection
- **Audio Normalization**: Consistent volume across all documents
- **Analytics Included**: Track plays, completion rates, bandwidth usage
- **Direct Playback URLs**: No need to manage streaming infrastructure

### 3. Three-View Experience System

**Modern Multi-Modal Viewer**:
```dart
enum ViewMode {
  original,  // Scanned images, PDFs, original format
  text,      // OCR'd text in readable format
  audio,     // Audio player with visual feedback
}

class MemoryBookViewer extends StatefulWidget {
  final MemoryBook book;
  final ViewMode initialMode;
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Column(
        children: [
          // Swipeable view modes
          TabBar(
            tabs: [
              Tab(icon: Icon(Icons.photo), text: 'Original'),
              Tab(icon: Icon(Icons.text_fields), text: 'Text'),
              Tab(icon: Icon(Icons.headphones), text: 'Audio'),
            ],
          ),
          Expanded(
            child: TabBarView(
              children: [
                OriginalView(book: book),
                TextView(book: book),
                AudioView(book: book),
              ],
            ),
          ),
        ],
      ),
    );
  }
}

// Original View - Shows scanned documents/images
class OriginalView extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return PhotoViewGallery.builder(
      itemCount: book.segments.length,
      builder: (context, index) {
        final segment = book.segments[index];
        return PhotoViewGalleryPageOptions(
          imageProvider: NetworkImage(segment.original_url),
          minScale: PhotoViewComputedScale.contained,
          maxScale: PhotoViewComputedScale.covered * 3,
        );
      },
      onPageChanged: (index) {
        // Sync audio position if playing
        if (audioPlayer.isPlaying) {
          audioPlayer.seek(book.segments[index].timestamp_start);
        }
      },
    );
  }
}

// Text View - Clean, readable OCR text
class TextView extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scrollable(
      child: SelectableText.rich(
        TextSpan(
          children: book.segments.map((segment) {
            return TextSpan(
              text: segment.ocr_text ?? '',
              style: TextStyle(
                fontSize: _userPreferredSize,
                height: 1.6,
                backgroundColor: _isCurrentSegment(segment) 
                  ? Colors.yellow.withOpacity(0.3) 
                  : null,
              ),
              recognizer: TapGestureRecognizer()
                ..onTap = () => audioPlayer.seek(segment.timestamp_start),
            );
          }).toList(),
        ),
      ),
    );
  }
}

// Audio View - Visual audio player
class AudioView extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        // Current segment image
        Expanded(
          child: AnimatedSwitcher(
            duration: Duration(milliseconds: 300),
            child: Image.network(
              currentSegment.original_url,
              key: ValueKey(currentSegment.id),
            ),
          ),
        ),
        // Waveform visualization
        AudioWaveform(
          audioUrl: book.audio_url,
          segments: book.segments,
          onSeek: (position) => audioPlayer.seek(position),
        ),
        // Playback controls
        PlaybackControls(
          onPlay: () => audioPlayer.play(),
          onPause: () => audioPlayer.pause(),
          onSkipToSegment: (segmentId) {
            final segment = book.segments.firstWhere((s) => s.id == segmentId);
            audioPlayer.seek(segment.timestamp_start);
          },
        ),
        // Segment list
        SegmentList(
          segments: book.segments,
          currentSegment: currentSegment,
          onTap: (segment) => audioPlayer.seek(segment.timestamp_start),
        ),
      ],
    );
  }
}
```

**Key Features**:
- **Seamless Switching**: Swipe between views without losing position
- **Synchronized Navigation**: Jump to same content across all views
- **Accessibility**: Each view optimized for different needs
- **Modern UX**: Follows Material/iOS design patterns

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

## Content Organization Model

### Flexible Collection Structure

The key architectural decision is to support both simple and complex use cases through a "Memory Book → Thread → Segment" hierarchy:

```rust
// A Memory Book can contain multiple Threads (narrative sections)
// Each Thread is a cohesive audio file with its own segments

pub struct MemoryBook {
    pub id: Uuid,
    pub family_id: Uuid,
    pub title: String,
    pub description: Option<String>,
    pub threads: Vec<Thread>,
    pub cover_image_url: Option<String>,
    pub created_at: DateTime<Utc>,
    pub settings: BookSettings,
}

pub struct Thread {
    pub id: Uuid,
    pub book_id: Uuid,
    pub title: String,              // e.g., "Letters from War", "Recipe Collection"
    pub sequence_number: i32,       // Order within the book
    pub audio_url: String,          // Single concatenated audio file
    pub total_duration: f64,
    pub segments: Vec<ContentSegment>,
    pub narrator_voice: VoiceSettings,
}

pub struct ContentSegment {
    pub id: Uuid,
    pub thread_id: Uuid,
    pub sequence_number: i32,       // Order within thread
    pub content_type: ContentType,
    pub original_url: String,       // Image/PDF page URL
    pub ocr_text: Option<String>,
    pub timestamp_start: f64,       // Position in thread audio
    pub timestamp_end: f64,
    pub metadata: SegmentMetadata,
}
```

### Organization Benefits

1. **Maximum Flexibility**
   - Single thread = Simple audio book (most common use case)
   - Multiple threads = Organized sections (advanced users)
   - Mix content types within threads

2. **Natural Groupings**
   - "Dad's War Letters" (Thread 1)
   - "Mom's Recipes" (Thread 2)  
   - "Family Photos & Stories" (Thread 3)

3. **Print-Ready Structure**
   - Each thread becomes a chapter
   - Table of contents with page numbers
   - QR code links to specific thread timestamps

4. **Progressive Complexity**
   - Start simple: One book = one thread
   - Grow naturally: Add threads as needed
   - AI suggestions: "These letters seem related..."

### Example Use Cases

#### Simple Use Case: "Grandma's Letters"
```
Memory Book: "Grandma's Letters"
└── Thread 1: "All Letters" (single audio file)
    ├── Segment 1: Letter from 1943
    ├── Segment 2: Letter from 1944
    └── Segment 3: Letter from 1945
```

#### Complex Use Case: "Family Heritage Collection"
```
Memory Book: "Smith Family Heritage"
├── Thread 1: "Grandpa's War Years" (15 min audio)
│   ├── Segment 1: Enlistment letter
│   ├── Segment 2: Letters from France
│   └── Segment 3: Victory announcement
├── Thread 2: "Grandma's Recipes" (8 min audio)
│   ├── Segment 1: Handwritten recipe cards
│   └── Segment 2: Recipe notes and stories
└── Thread 3: "Family Photos & Captions" (12 min audio)
    ├── Segment 1: Wedding photo description
    └── Segment 2: Baby photos with notes
```

#### Recipe Collection Use Case: "Grandma's Fixin's"
```
Memory Book: "Grandma's Southern Kitchen" (Perfect Christmas Gift!)
├── Thread 1: "Holiday Favorites" (12 min audio)
│   ├── Segment 1: Famous Pecan Pie (with her secret ingredient notes)
│   ├── Segment 2: Christmas Morning Cinnamon Rolls
│   └── Segment 3: Thanksgiving Cornbread Dressing
├── Thread 2: "Sunday Suppers" (10 min audio)
│   ├── Segment 1: Fried Chicken & Gravy
│   ├── Segment 2: Buttermilk Biscuits
│   └── Segment 3: Green Bean Casserole
├── Thread 3: "Sweet Treats" (8 min audio)
│   ├── Segment 1: Chocolate Chess Pie
│   ├── Segment 2: Tea Cakes
│   └── Segment 3: Blackberry Cobbler
└── Thread 4: "Family Stories Behind the Food" (15 min audio)
    ├── Segment 1: How she learned from her mother
    ├── Segment 2: Feeding the farmhands stories
    └── Segment 3: Sunday dinner traditions
```

### Recipe-Specific Features

1. **Recipe Mode Processing**
   ```rust
   pub struct RecipeProcessor {
       // Special OCR training for common recipe terms
       // Handles fractions, measurements, temperatures
       ocr_engine: RecipeOptimizedOCR,
       
       // Recipe-specific voice narration
       voice_settings: RecipeVoiceSettings {
           pace: "slower_for_ingredients",
           emphasis: "measurements_and_timing",
           pauses: "after_each_step"
       }
   }
   ```

2. **Print-Ready Recipe Books**
   - Professional cookbook layout
   - Original handwriting alongside typed version
   - QR codes for audio cooking instructions
   - Spiral binding option for kitchen use
   - Laminated pages available

3. **Marketing Angles**
   - "Give the gift of Grandma's cooking this Christmas"
   - "Turn recipe cards into a family cookbook"
   - Mother's Day special: "Mom's Kitchen Memories"
   - "Preserve the secret ingredients and stories"

## Database Schema

### PostgreSQL Schema

#### Users Table
```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255),
    display_name VARCHAR(255),
    
    -- User preferences
    default_voice VARCHAR(50),
    default_language VARCHAR(10) DEFAULT 'en',
    auto_enhance_images BOOLEAN DEFAULT true,
    
    -- Current/default family toggle
    current_family_id UUID, -- Which family context they're viewing
    
    -- Timestamps
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    last_login_at TIMESTAMPTZ
);

-- Family accounts table (one subscription per family)
CREATE TABLE families (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL, -- "The Smith Family"
    
    -- Subscription info (ONE per family)
    subscription_tier VARCHAR(20) DEFAULT 'free_trial',
    subscription_status VARCHAR(20) DEFAULT 'active',
    subscription_expires_at TIMESTAMPTZ,
    stripe_customer_id VARCHAR(255),
    stripe_subscription_id VARCHAR(255),
    
    -- Free trial tracking
    trial_scans_used INTEGER DEFAULT 0,
    trial_expires_at TIMESTAMPTZ DEFAULT (NOW() + INTERVAL '7 days'),
    
    -- Usage tracking (for the whole family)
    documents_processed INTEGER DEFAULT 0,
    audio_minutes_generated DECIMAL(10,2) DEFAULT 0,
    storage_used_mb DECIMAL(10,2) DEFAULT 0,
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Many-to-many: Users can be in multiple families
CREATE TABLE family_members (
    family_id UUID REFERENCES families(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    role VARCHAR(20) CHECK (role IN ('owner', 'admin', 'member')),
    joined_at TIMESTAMPTZ DEFAULT NOW(),
    invited_by UUID REFERENCES users(id),
    PRIMARY KEY (family_id, user_id)
);

-- Memory Books (the collections)
CREATE TABLE memory_books (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    family_id UUID REFERENCES families(id) ON DELETE CASCADE,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    cover_image_url VARCHAR(500),
    
    -- Settings
    auto_organize BOOLEAN DEFAULT false, -- AI suggests thread groupings
    sharing_enabled BOOLEAN DEFAULT true,
    print_ready BOOLEAN DEFAULT false,
    default_voice VARCHAR(50),
    
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Threads (audio narratives within books)
CREATE TABLE threads (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    book_id UUID REFERENCES memory_books(id) ON DELETE CASCADE,
    title VARCHAR(255) NOT NULL,
    sequence_number INTEGER NOT NULL,
    
    -- Audio information
    audio_mux_asset_id VARCHAR(255),
    audio_url VARCHAR(500),
    total_duration DECIMAL(10,3), -- seconds
    
    -- Voice settings for this thread
    voice_id VARCHAR(50),
    voice_speed DECIMAL(3,2) DEFAULT 1.0,
    voice_pitch DECIMAL(3,2) DEFAULT 1.0,
    
    processing_status VARCHAR(50) DEFAULT 'pending',
    processing_started_at TIMESTAMPTZ,
    processing_completed_at TIMESTAMPTZ,
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(book_id, sequence_number)
);

-- Content Segments (individual items within threads)
CREATE TABLE content_segments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    thread_id UUID REFERENCES threads(id) ON DELETE CASCADE,
    sequence_number INTEGER NOT NULL,
    
    -- Content type and source
    content_type VARCHAR(30) CHECK (content_type IN (
        'handwritten_document', 'typed_document', 'photo', 'recipe', 'mixed'
    )),
    source_format VARCHAR(20), -- PDF, JPEG, PNG, DOCX, etc.
    
    -- Original content (stored in Mux)
    original_mux_asset_id VARCHAR(255),
    original_url VARCHAR(500), -- Direct playback URL
    
    -- OCR/Text extraction
    ocr_text TEXT,
    ocr_confidence DECIMAL(3,2),
    ocr_method VARCHAR(50), -- google_vision, olmocr, pdf_extract, etc.
    
    -- Position in audio
    timestamp_start DECIMAL(10,3) NOT NULL, -- seconds
    timestamp_end DECIMAL(10,3) NOT NULL,
    
    -- Metadata
    title VARCHAR(500),
    page_number INTEGER, -- For multi-page documents
    date_written DATE,
    author VARCHAR(255),
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(thread_id, sequence_number)
);

-- Family invites
CREATE TABLE family_invites (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    family_id UUID REFERENCES families(id) ON DELETE CASCADE,
    invite_code VARCHAR(10) UNIQUE,
    email VARCHAR(255), -- Optional: specific email invite
    created_by UUID REFERENCES users(id),
    expires_at TIMESTAMPTZ DEFAULT (NOW() + INTERVAL '7 days'),
    used_at TIMESTAMPTZ,
    used_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Photo narrations (for standalone photos with audio)
CREATE TABLE photo_narrations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    segment_id UUID REFERENCES content_segments(id) ON DELETE CASCADE,
    narrator_id UUID REFERENCES users(id),
    narration_text TEXT,
    narration_audio_url VARCHAR(500),
    narration_duration DECIMAL(10,3),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Content links (for connecting photos to documents)
CREATE TABLE content_links (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    from_segment_id UUID REFERENCES content_segments(id) ON DELETE CASCADE,
    to_segment_id UUID REFERENCES content_segments(id) ON DELETE CASCADE,
    link_type VARCHAR(30) CHECK (link_type IN ('illustration', 'reference', 'companion')),
    timestamp_link DECIMAL(10,3), -- Optional: specific timestamp in audio
    created_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(from_segment_id, to_segment_id)
);

-- Tags for all content
CREATE TABLE content_tags (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    segment_id UUID REFERENCES content_segments(id) ON DELETE CASCADE,
    tag_type VARCHAR(20) CHECK (tag_type IN ('people', 'topics', 'locations', 'time_periods', 'custom')),
    tag_value VARCHAR(255),
    confidence DECIMAL(3,2),
    UNIQUE(segment_id, tag_type, tag_value)
);

-- Add foreign key constraints
ALTER TABLE users ADD CONSTRAINT fk_current_family
    FOREIGN KEY (current_family_id) REFERENCES families(id);

-- Indexes for performance
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_family_members_user ON family_members(user_id);
CREATE INDEX idx_memory_books_family ON memory_books(family_id);
CREATE INDEX idx_threads_book ON threads(book_id);
CREATE INDEX idx_segments_thread ON content_segments(thread_id);
CREATE INDEX idx_segments_timestamp ON content_segments(thread_id, timestamp_start);
CREATE INDEX idx_invites_code ON family_invites(invite_code);
CREATE INDEX idx_invites_family ON family_invites(family_id);
CREATE INDEX idx_photo_narrations_segment ON photo_narrations(segment_id);
CREATE INDEX idx_content_links_from ON content_links(from_segment_id);
CREATE INDEX idx_content_links_to ON content_links(to_segment_id);
CREATE INDEX idx_content_tags_segment ON content_tags(segment_id);
```

## Photo & Mixed Media Features

### Photo Narration System

Photos can have their own audio narrations, separate from document processing:

```rust
pub struct PhotoNarration {
    // User records audio describing the photo
    pub async fn record_narration(
        &self,
        photo: &ContentSegment,
        audio_data: Vec<u8>,
        transcript: String,
    ) -> Result<Narration> {
        // Upload audio to Mux
        let audio_asset = self.mux_client.upload_audio(audio_data).await?;
        
        // Save narration
        let narration = Narration {
            segment_id: photo.id,
            narrator_id: current_user.id,
            audio_url: audio_asset.playback_url,
            transcript,
            duration: audio_asset.duration,
        };
        
        Ok(narration)
    }
}
```

### Mixed Media Memory Books

Combine different content types in meaningful ways:

```typescript
interface MixedMediaThread {
  // Example: "Our Wedding Day"
  segments: [
    {
      type: 'photo',
      url: 'wedding-ceremony.jpg',
      narration: 'This was the moment we said I do...',
      timestamp: { start: 0, end: 15 }
    },
    {
      type: 'handwritten_document',
      url: 'wedding-vows.jpg',
      ocr_text: 'I promise to love you...',
      timestamp: { start: 15, end: 45 }
    },
    {
      type: 'photo',
      url: 'first-dance.jpg',
      narration: 'Our first dance as husband and wife',
      timestamp: { start: 45, end: 60 }
    }
  ]
}
```

### Smart Photo Features

1. **Auto-grouping by Date/Event**
   - AI detects related photos
   - Suggests thread creation
   - Maintains chronological order

2. **Face Detection & Tagging**
   - Auto-identify family members
   - Link to family tree
   - Search by person

3. **Scene Understanding**
   - "Find all Christmas photos"
   - "Show beach vacations"
   - "Military uniforms"

## Multiple Input Format Support

### Supported Formats

FamilyTales processes various document types to accommodate all family memory formats:

```rust
pub enum SupportedFormat {
    // Images
    JPEG,
    PNG,
    HEIC,    // iOS photos
    WEBP,
    
    // Documents
    PDF,     // Scanned documents, multi-page support
    DOCX,    // Modern Word documents
    DOC,     // Legacy Word documents
    TXT,     // Plain text files
    RTF,     // Rich text format
    
    // Specialized
    RECIPE,  // Auto-detected recipe cards
    LETTER,  // Auto-detected handwritten letters
}

pub struct FormatProcessor {
    pub async fn process_document(&self, file: UploadedFile) -> Result<ProcessedContent> {
        match file.format {
            // Direct image processing
            JPEG | PNG | HEIC | WEBP => {
                let image = self.normalize_image(file).await?;
                self.extract_text_from_image(image).await
            }
            
            // Multi-page document handling
            PDF => {
                let pages = self.extract_pdf_pages(file).await?;
                let mut segments = Vec::new();
                
                for (idx, page) in pages.iter().enumerate() {
                    let text = if page.is_scanned_image() {
                        self.extract_text_from_image(page.to_image()).await?
                    } else {
                        page.extract_embedded_text()?
                    };
                    
                    segments.push(ContentSegment {
                        page_number: idx + 1,
                        ocr_text: text,
                        ..Default::default()
                    });
                }
                
                Ok(ProcessedContent { segments })
            }
            
            // Word documents
            DOCX | DOC => {
                let text = self.extract_word_text(file).await?;
                let images = self.extract_word_images(file).await?;
                self.create_mixed_content(text, images)
            }
            
            // Recipe-specific processing
            RECIPE => {
                let recipe = self.recipe_processor.extract_recipe(file).await?;
                Ok(ProcessedContent {
                    segments: vec![ContentSegment {
                        ocr_text: recipe.formatted_text(),
                        metadata: recipe.ingredients_and_steps(),
                        content_type: ContentType::Recipe,
                        ..Default::default()
                    }]
                })
            }
            
            _ => self.default_processor.process(file).await
        }
    }
}
```

### Smart Format Detection

```rust
impl FormatDetector {
    pub fn detect_content_type(&self, file: &UploadedFile) -> ContentType {
        // Check file extension first
        let content_type = match file.extension.to_lowercase().as_str() {
            "jpg" | "jpeg" | "png" | "heic" => {
                // Analyze image content
                if self.looks_like_recipe(&file.preview) {
                    ContentType::Recipe
                } else if self.looks_like_letter(&file.preview) {
                    ContentType::HandwrittenDocument
                } else {
                    ContentType::Photo
                }
            }
            "pdf" => ContentType::TypedDocument,
            "docx" | "doc" => ContentType::TypedDocument,
            _ => ContentType::Mixed
        };
        
        content_type
    }
    
    fn looks_like_recipe(&self, preview: &ImagePreview) -> bool {
        // ML model trained on recipe cards
        // Looks for: ingredient lists, measurements, cooking terms
        self.recipe_classifier.predict(preview) > 0.8
    }
}
```

## Authentication & Family Management

### Authentication with Clerk

FamilyTales uses Clerk for authentication, providing a seamless experience across web and mobile:

```typescript
// Clerk integration for Flutter/Web
class AuthService {
  final ClerkClient clerk;
  
  // Seamless invite flow
  async joinFamilyViaInvite(inviteToken: string) {
    // Decode JWT invite token
    const invite = await this.decodeInviteToken(inviteToken);
    
    // If not logged in, show Clerk sign-up with pre-filled family
    if (!clerk.user) {
      return clerk.signUp({
        redirectUrl: `/join-family?token=${inviteToken}`,
        afterSignUpUrl: `/families/${invite.familyId}/welcome`
      });
    }
    
    // If logged in, add to family immediately
    await this.addUserToFamily(clerk.user.id, invite.familyId);
  }
}
```

### Family Invitation System

Two methods for inviting family members:

1. **Direct Link Invitations**
   ```
   https://familytales.app/join/abc123def456
   - JWT encoded with family_id, expires_at
   - Works on web, deep links to mobile app
   - No email required, instant join
   ```

2. **Email Invitations (via SendGrid)**
   ```typescript
   async inviteViaEmail(email: string, familyName: string) {
     const inviteToken = generateInviteToken(familyId);
     
     await sendgrid.send({
       to: email,
       template: 'family-invite',
       data: {
         familyName,
         inviterName,
         joinUrl: `https://familytales.app/join/${inviteToken}`,
         appStoreUrl: 'https://apps.apple.com/familytales',
         playStoreUrl: 'https://play.google.com/familytales'
       }
     });
   }
   ```

### Family Roles & Permissions

```rust
pub enum FamilyRole {
    HeadOfFamily,  // Created family or transferred ownership
    Admin,         // Can invite/remove members
    Member,        // Can view/upload content
}

pub struct FamilyPermissions {
    head_of_family: UserId,  // Pays subscription, ultimate authority
    
    // Head of Family exclusive permissions
    can_remove_members: bool,           // true only for head
    can_transfer_ownership: bool,       // true only for head
    can_cancel_subscription: bool,      // true only for head
    can_create_second_family: bool,     // Premium perk
    
    // Admin permissions (including head)
    can_invite_members: bool,
    can_manage_content: bool,
    can_create_memory_books: bool,
    
    // Member permissions (everyone)
    can_upload_content: bool,           // Everyone can scan/upload!
    can_view_content: bool,
    can_share_publicly: bool,
}
```

### Multiple Family Support

Users can belong to multiple families with smart context switching:

```dart
class FamilyContext extends StateNotifier<Family> {
  // Auto-switch based on content being viewed
  void autoSwitchFamily(String contentId) {
    final family = findFamilyForContent(contentId);
    if (family != null && family.id != state.id) {
      state = family;
      // Update UI to show current family context
    }
  }
  
  // Manual family switcher in app header
  Widget familySwitcher() {
    return DropdownButton<Family>(
      value: currentFamily,
      items: userFamilies.map((family) => 
        DropdownMenuItem(
          value: family,
          child: Row(
            children: [
              if (family.isPremium) PremiumBadge(),
              Text(family.name),
            ],
          ),
        ),
      ).toList(),
      onChanged: (family) => switchToFamily(family),
    );
  }
}
```

## Freemium Model Details

### Free Tier - "Family Memories Starter"
**Perfect for trying out the service**

- **3 document scans per month** (resets monthly)
- **Basic voice only** (one male, one female voice)
- **Up to 5 family members**
- **7-day trial of premium features** on sign-up
- **Watermark on shared content** ("Created with FamilyTales")
- **Standard processing speed** (up to 24 hours)
- **View-only for premium family content** (if invited to premium family)

### Family Plan - $14.99/month
**The sweet spot for most families**

- **Unlimited document scans**
- **Premium voices** (10+ options including accents)
- **Unlimited family members**
- **Instant processing** (under 5 minutes)
- **No watermarks**
- **Public sharing** (social media friendly)
- **Offline downloads**
- **Priority support**
- **Memory Book templates**

### Family Legacy - $29.99/month
**For serious family historians**

Everything in Family Plan plus:
- **Two family groups** (e.g., both sides of the family)
- **Voice cloning** (preserve Dad's actual voice)
- **Bulk upload mode** (process entire boxes)
- **API access** for developers
- **White-label sharing** (your-family.familytales.app)
- **Phone support**
- **Print-on-demand credits** ($10/month)
- **Early access features**

### Smart Freemium Conversion Tactics

1. **Emotional Triggers**
   - After 3rd scan: "You've preserved 3 memories! Upgrade to preserve them all."
   - Voice preview: "Hear Grandma's letter in a premium voice" (30-second preview)

2. **Family Viral Loop**
   - Free users can join premium families and access all content
   - They see "Shared by [Premium Family Name]" encouraging upgrade
   - "Your Smith family has premium, but your Jones family doesn't"

3. **Seasonal Campaigns**
   - Mother's Day: "Give Mom unlimited memories"
   - Christmas: "Gift a year of family memories"
   - Grandparents Day: Free premium weekend

## Voice Generation Strategy

### Text-to-Speech Options

**For MVP (Third-party services):**

1. **Google Cloud Text-to-Speech**
   - Cost: ~$16 per 1M characters
   - Quality: Excellent, WaveNet voices
   - Voices: 380+ voices in 50+ languages
   - Neural voices sound very natural

2. **Amazon Polly**
   - Cost: ~$4 per 1M characters (standard), $16 (neural)
   - Quality: Good to excellent
   - Voices: 60+ voices
   - Real-time streaming capability

3. **ElevenLabs** (Premium option)
   - Cost: ~$99/month for 500k characters
   - Quality: State-of-the-art, most natural
   - Voice cloning capability
   - Perfect for Family Legacy tier

**Implementation Strategy:**
```rust
pub struct VoiceProcessor {
    // Generate once, store multiple versions
    pub async fn generate_audio(&self, text: &str, book_settings: &BookSettings) -> Result<AudioAsset> {
        // For free tier: Generate with basic voice only
        if book_settings.tier == Tier::Free {
            return self.generate_single_voice(text, Voice::BasicFemale).await;
        }
        
        // For premium: Generate with selected voice
        // We do NOT store multiple versions - user selects voice during book creation
        let voice = book_settings.selected_voice;
        let audio = self.generate_single_voice(text, voice).await?;
        
        // Upload to Mux once
        let mux_asset = self.mux_client.upload_audio(audio).await?;
        
        Ok(AudioAsset {
            mux_playback_id: mux_asset.playback_id,
            voice_used: voice,
            duration: audio.duration,
        })
    }
}
```

### Voice Selection UX

```dart
// During Memory Book creation
class VoiceSelectionScreen extends StatelessWidget {
  Widget build(context) {
    return Column(
      children: [
        Text("Choose a narrator voice for '${book.title}'"),
        
        // Preview with actual content
        for (voice in availableVoices)
          ListTile(
            title: Text(voice.name),
            subtitle: Text(voice.description), // "Warm grandmother", "Gentle male"
            trailing: PlayButton(
              onTap: () => previewVoice(
                voice: voice,
                text: book.segments.first.text.substring(0, 100) // First 100 chars
              ),
            ),
            onTap: () => selectVoice(voice),
          ),
      ],
    );
  }
}
```

### Storage Efficiency

- **One audio file per thread** (not per voice)
- **Voice selection locked** at thread creation
- **Change voice = regenerate thread** (premium only)
- **No multi-voice storage** (saves 90% storage costs)

## Future Feature: Interactive Family Tree

### Vision
An interactive family tree that connects faces, voices, and stories - helping younger generations understand their heritage visually and emotionally.

```typescript
interface FamilyTreeNode {
  id: string;
  name: string;
  birthYear?: number;
  deathYear?: number;
  profilePhotoUrl?: string;
  relationshipType: 'parent' | 'child' | 'spouse' | 'sibling';
  
  // Content connections
  memoryBooks: MemoryBookRef[];
  documentsWritten: DocumentRef[];
  documentsMentioned: DocumentRef[];
  photosAppearing: PhotoRef[];
  voiceSample?: AudioRef;  // For voice cloning later
}

interface FamilyTreeView {
  // Visual representation
  layout: 'traditional' | 'circular' | 'timeline';
  
  // Interactive features
  onNodeClick: (person: FamilyTreeNode) => {
    showPersonOverlay({
      photos: getPhotosOfPerson(person),
      letters: getLettersFrom(person),
      mentions: getDocumentsMentioning(person),
      audio: getAudioNarrationBy(person)
    });
  };
  
  // Storytelling integration
  playFamilyStory: () => {
    // Chronological audio journey through the tree
    const timeline = generateFamilyTimeline();
    playAudioJourney(timeline);
  };
}
```

### Smart Connections

```rust
pub struct FamilyTreeBuilder {
    // Auto-detect relationships from document content
    pub fn extract_relationships(&self, segments: &[ContentSegment]) -> Vec<Relationship> {
        let mut relationships = Vec::new();
        
        for segment in segments {
            // Natural language processing to find relationships
            let mentions = self.nlp.extract_family_mentions(&segment.ocr_text);
            
            for mention in mentions {
                relationships.push(Relationship {
                    from_person: segment.author,
                    to_person: mention.person_name,
                    relationship_type: mention.detected_relation, // "my daughter", "dear mother"
                    confidence: mention.confidence,
                    source_document_id: segment.id,
                });
            }
        }
        
        relationships
    }
}
```

### Integration with Memory Books

1. **Person-Centric Views**
   - Click on Grandma in the tree → See all her letters, recipes, photos
   - Timeline view of her life through documents
   - Audio compilation of mentions across family documents

2. **Generational Stories**
   - "The Women of Our Family" - matrilineal audio journey
   - "Father to Son Letters" - paternal wisdom through generations
   - "Sibling Stories" - lateral family connections

3. **Educational Features for Youth**
   ```dart
   class FamilyTreeForKids extends StatelessWidget {
     Widget buildPersonCard(FamilyTreeNode person) {
       return InteractiveCard(
         child: Column(
           children: [
             CircleAvatar(
               backgroundImage: NetworkImage(person.profilePhotoUrl),
               radius: 60,
             ),
             Text(person.name, style: kidFriendlyFont),
             Text(_getRelationshipText(person)), // "Your Great-Grandma"
             
             PlayButton(
               label: "Hear their story",
               onPressed: () => playPersonAudio(person),
             ),
             
             if (person.documentsWritten.isNotEmpty)
               TextButton(
                 child: Text("Read their letters"),
                 onPressed: () => showDocuments(person.documentsWritten),
               ),
           ],
         ),
       );
     }
   }
   ```

### Future AI Enhancements

- **Face Recognition**: Auto-tag people in photos to build tree
- **Handwriting Matching**: Identify unsigned documents by writing style
- **Story Generation**: AI creates connecting narratives between family members
- **Missing Links**: Suggest questions to ask living relatives

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

## Family-Based Subscription Model

### Core Concept: One Pays, All Play
The family administrator (usually Mom/Dad) purchases ONE subscription that covers the ENTIRE family. No individual billing, no seat counting - true family sharing.

### Tier Structure

#### Free Trial - "Test with Grandma"
**Limits**:
- 3 document scans total
- 1 week access for whole family
- Basic TTS voices only
- See if Grandma can use it

**Purpose**: Risk-free family onboarding

#### Family Plan ($14.99/month or $119/year)
**The Sweet Spot - Everything Most Families Need**:
- Unlimited family members (seriously, invite everyone)
- Unlimited document scans
- Premium natural voices
- Automatic family-wide sharing
- Offline downloads for all
- Synchronized playback
- Email/chat support
- No seat limits, no user counting

**Target**: 95% of families fit here

#### Family Legacy ($29.99/month or $299/year)
**For Families Preserving Serious History**:
- Everything in Family Plan
- Voice cloning (preserve voices before they're gone)
- Bulk scanning mode (entire boxes at once)
- White-label domain (memories.smithfamily.com)
- API access for tech-savvy family members
- Phone support for elderly members
- Early access to new features
- Priority processing queue

**Target**: Large families, family historians, estate planning

### Family Account Implementation

```rust
#[derive(Debug, Clone)]
pub struct FamilyAccount {
    pub id: Uuid,
    pub name: String, // "The Smith Family"
    pub subscription: FamilySubscription,
    pub admin_id: Uuid, // Who pays
    pub members: Vec<FamilyMember>,
    pub invite_code: String, // Easy 6-digit code
}

#[derive(Debug, Clone)]
pub struct FamilySubscription {
    pub tier: SubscriptionTier,
    pub status: SubscriptionStatus,
    pub paid_by: Uuid, // Usually Mom or Dad
    pub stripe_subscription_id: String,
    pub expires_at: DateTime<Utc>,
}

#[derive(Debug, Clone)]
pub enum SubscriptionTier {
    FreeTrial { 
        scans_remaining: u8,
        expires_at: DateTime<Utc>,
    },
    FamilyPlan, // $14.99 - unlimited everything
    FamilyLegacy, // $29.99 - plus voice cloning, bulk mode
}

// Context-aware feature access
pub struct UserContext {
    pub user: User,
    pub current_family: FamilyAccount,
    pub all_families: Vec<FamilyAccount>,
}

impl UserContext {
    pub fn can_use_premium_features(&self) -> bool {
        // Only in the context of a paid family
        matches!(
            self.current_family.subscription.tier,
            SubscriptionTier::FamilyPlan | SubscriptionTier::FamilyLegacy
        ) && self.current_family.subscription.status == SubscriptionStatus::Active
    }
    
    pub fn can_scan(&self) -> bool {
        match &self.current_family.subscription.tier {
            SubscriptionTier::FreeTrial { scans_remaining, expires_at } => {
                *scans_remaining > 0 && Utc::now() < *expires_at
            },
            SubscriptionTier::FamilyPlan | SubscriptionTier::FamilyLegacy => {
                self.current_family.subscription.status == SubscriptionStatus::Active
            }
        }
    }
    
    pub fn available_voices(&self) -> Vec<Voice> {
        if self.can_use_premium_features() {
            Voice::all_premium_voices()
        } else {
            Voice::basic_voices_only()
        }
    }
    
    pub fn switch_family(&mut self, family_id: Uuid) -> Result<()> {
        if let Some(family) = self.all_families.iter().find(|f| f.id == family_id) {
            self.current_family = family.clone();
            Ok(())
        } else {
            Err(ApiError::FamilyNotFound)
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

### Phase 5: Physical Products Integration (Months 13-18)

**Print-on-Demand Memory Books**:
```rust
pub struct PrintProduct {
    pub memory_book_id: Uuid,
    pub layout: BookLayout,
    pub options: PrintOptions,
}

pub struct BookLayout {
    pub format: BookFormat, // Hardcover, Softcover, Premium
    pub size: BookSize,     // 8x10, 11x14, Coffee Table
    pub pages: Vec<PageLayout>,
}

pub struct PageLayout {
    pub document_id: Uuid,
    pub layout_type: LayoutType,
    pub elements: Vec<PageElement>,
}

pub enum PageElement {
    OriginalHandwriting {
        image_url: String,
        position: Rectangle,
        page_number: u32,      // Corresponds to app navigation
    },
    TranscribedText {
        text: String,
        font: FontStyle,
        position: Rectangle,
    },
    Photo {
        mux_playback_id: String,
        caption: Option<String>,
        position: Rectangle,
    },
    PageNumber {
        number: u32,
        position: Point,
    },
}

// Single QR code for the entire book
pub struct MemoryBookQR {
    pub memory_book_id: Uuid,
    pub deep_link: String,  // familytales://book/{id}
    pub placement: QRPlacement,
}

pub enum QRPlacement {
    FrontCover,
    BackCover, 
    FirstPage,
    Custom(u32), // Specific page number
}
```

**Implementation Architecture**:
- **Print Partner Integration**: Blurb, Lulu, or custom fulfillment
- **Single QR Code**: One code on cover/first page opens Memory Book in app
- **Page Number Navigation**: In-app page selector jumps to audio timestamps
- **Layout Engine**: Auto-layout with manual override options
- **Preview System**: 3D book preview before ordering
- **Quality Control**: High-res image requirements, text legibility checks

**App Integration**:
```rust
// Page-to-timestamp mapping stored with Memory Book
pub struct PageMapping {
    pub page_number: u32,
    pub document_id: Uuid,
    pub audio_timestamp: f64,  // Seconds into combined audio
    pub image_reference: String,
}

impl MemoryBookPlayer {
    pub fn jump_to_page(&mut self, page_number: u32) {
        if let Some(mapping) = self.page_mappings.get(&page_number) {
            self.audio_player.seek_to(mapping.audio_timestamp);
            self.current_image = mapping.image_reference;
            self.highlight_page_number(page_number);
        }
    }
}
```

**Business Model**:
- Base price: $39 for 20 pages
- Additional pages: $0.50 each
- Premium options: Leather cover (+$40), Gift box (+$20)
- Family discounts: 20% off 5+ copies
- Estimated margin: 40-50%

**Use Cases**:
- Memorial books for funerals
- Anniversary gifts
- Family reunion souvenirs
- Holiday presents
- Estate planning packages

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