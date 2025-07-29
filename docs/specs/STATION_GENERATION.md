# Audio Generation Stations/Workers Specification

## Overview

This document outlines the worker architecture for processing family stories through various generation stations. Each station handles a specific part of the pipeline from image ingestion to final audio delivery.

## 1. OCR Processing Station

### Purpose
Extract text from scanned images of family stories, letters, and documents with high accuracy and confidence scoring.

### Image Preprocessing Pipeline

1. **Input Validation**
   - Supported formats: JPEG, PNG, PDF, TIFF
   - Maximum file size: 20MB per image
   - Minimum resolution: 300 DPI recommended

2. **Preprocessing Steps**
   ```
   Raw Image → Orientation Detection → Auto-Rotation → 
   Noise Reduction → Contrast Enhancement → Binarization →
   Skew Correction → Border Removal → Ready for OCR
   ```

3. **Enhancement Techniques**
   - Adaptive thresholding for varying lighting conditions
   - Morphological operations for text clarity
   - Background subtraction for aged documents
   - Smart cropping to focus on text regions

### OCR Engine Selection

**MVP: Google Cloud Vision API**
- Proven accuracy for handwritten and typed text
- Built-in language detection
- Confidence scoring per word/block
- Support for 50+ languages

**Future Considerations:**
- Tesseract 5 for on-premise processing
- Amazon Textract for form extraction
- Azure Computer Vision for specialized fonts

### Text Extraction and Confidence Scoring

1. **Extraction Process**
   ```json
   {
     "blocks": [
       {
         "text": "Dear Sarah,",
         "confidence": 0.98,
         "bounding_box": {...},
         "language": "en"
       }
     ],
     "overall_confidence": 0.94,
     "detected_languages": ["en"],
     "page_number": 1
   }
   ```

2. **Confidence Thresholds**
   - High confidence: > 0.9 (automatic processing)
   - Medium confidence: 0.7-0.9 (flagged for review)
   - Low confidence: < 0.7 (manual review required)

### Error Handling and Manual Review Queue

1. **Error Categories**
   - Image quality issues (blur, low resolution)
   - Unsupported languages
   - Handwriting legibility problems
   - Mixed content (text + images)

2. **Manual Review Queue**
   - Priority based on confidence scores
   - Side-by-side view (image + extracted text)
   - Edit capabilities with change tracking
   - Approval workflow with versioning

3. **Recovery Mechanisms**
   - Retry with enhanced preprocessing
   - Alternative OCR engine fallback
   - Partial text recovery options
   - Manual transcription assignment

## 2. Audio Generation Station

### Purpose
Convert extracted text into natural-sounding audio narrations with appropriate voice selection and post-processing.

### Text Preprocessing for TTS

1. **Text Normalization**
   - Expand abbreviations (Mr. → Mister)
   - Convert numbers to words
   - Handle dates intelligently (1942 → nineteen forty-two)
   - Process special characters and punctuation

2. **Prosody Markup**
   ```xml
   <speak>
     <p>Dear Sarah,</p>
     <break time="500ms"/>
     <p>I hope this letter finds you well.</p>
     <emphasis level="moderate">I miss you dearly.</emphasis>
   </speak>
   ```

3. **Context Analysis**
   - Sentiment detection for emotional tone
   - Named entity recognition for proper nouns
   - Pause insertion at natural breaks
   - Emphasis detection for important passages

### Voice Selection Logic

1. **Voice Attributes**
   ```json
   {
     "gender": "female",
     "age_range": "adult",
     "accent": "en-US",
     "speaking_rate": 1.0,
     "pitch": 0,
     "emotional_tone": "warm"
   }
   ```

2. **Selection Criteria**
   - Match narrator to story author when known
   - Consider time period and location
   - User preferences and family voice bank
   - Consistency across related stories

3. **Voice Rotation**
   - Multiple narrators for conversations
   - Distinct voices for different time periods
   - Child voices for young authors

### TTS Provider Integration

1. **Google Cloud Text-to-Speech (Primary)**
   - WaveNet voices for natural sound
   - SSML support
   - 40+ languages and variants
   - Real-time synthesis

2. **Amazon Polly (Fallback)**
   - Neural voices
   - Lexicon support for names
   - Breathing and speaking styles
   - Cost-effective for long content

3. **ElevenLabs (Premium)**
   - Voice cloning capabilities
   - Emotional range
   - Ultra-realistic output
   - Custom voice creation

### Audio Post-Processing and Normalization

1. **Processing Pipeline**
   ```
   Raw TTS → Noise Gate → EQ → Compression → 
   Normalization → Fade In/Out → Final Audio
   ```

2. **Quality Standards**
   - Target loudness: -16 LUFS (streaming standard)
   - Peak limiting: -1 dBTP
   - Frequency response: 50Hz-15kHz
   - Bit depth: 24-bit processing, 16-bit output

3. **Enhancement Features**
   - Background ambiance (optional)
   - Music bed for introductions
   - Chapter markers and metadata
   - Multiple format exports (MP3, M4A, WAV)

## 3. Media Processing Station

### Purpose
Prepare audio files for streaming distribution with proper encoding, thumbnail generation, and CDN deployment.

### Mux Upload and Encoding

1. **Upload Process**
   - Chunked upload for large files
   - Resume capability
   - Checksum verification
   - Metadata attachment

2. **Encoding Profiles**
   ```json
   {
     "audio_profiles": [
       {
         "name": "high_quality",
         "codec": "aac",
         "bitrate": "192k",
         "sample_rate": 48000
       },
       {
         "name": "standard",
         "codec": "aac",
         "bitrate": "128k",
         "sample_rate": 44100
       },
       {
         "name": "low_bandwidth",
         "codec": "aac",
         "bitrate": "64k",
         "sample_rate": 22050
       }
     ]
   }
   ```

3. **Adaptive Bitrate**
   - Multiple quality levels
   - Automatic switching based on bandwidth
   - Smooth transitions between qualities

### HLS Stream Generation

1. **Segmentation**
   - 10-second segments for optimal loading
   - Byte-range requests support
   - Playlist generation (m3u8)

2. **Manifest Structure**
   ```
   #EXTM3U
   #EXT-X-VERSION:3
   #EXT-X-TARGETDURATION:10
   #EXT-X-MEDIA-SEQUENCE:0
   #EXTINF:10.0,
   segment0.ts
   #EXTINF:10.0,
   segment1.ts
   ```

3. **Security Features**
   - Token-based authentication
   - Geo-restriction capabilities
   - Expiring URLs
   - DRM support (future)

### Thumbnail Generation

1. **Visual Representation**
   - Waveform visualization
   - Story metadata overlay
   - Family photo integration
   - Branded templates

2. **Generation Process**
   - Extract key moments from audio
   - Generate multiple options
   - A/B testing for engagement
   - Responsive sizing (multiple resolutions)

### CDN Distribution

1. **Multi-Region Deployment**
   - Primary: CloudFront
   - Fallback: Fastly
   - Geographic distribution based on user base

2. **Caching Strategy**
   - Audio segments: 1 year cache
   - Manifests: 5 minute cache
   - Thumbnails: 30 day cache
   - Purge API for updates

3. **Performance Optimization**
   - Pre-warming popular content
   - Predictive caching
   - Edge compression
   - HTTP/3 support

## 4. Worker Architecture

### RabbitMQ Job Distribution

1. **Queue Structure**
   ```
   ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
   │ OCR Queue   │     │ Audio Queue │     │ Media Queue │
   └─────────────┘     └─────────────┘     └─────────────┘
         │                   │                   │
         ├───────────────────┴───────────────────┤
         │                                       │
         │            RabbitMQ Broker            │
         │                                       │
         └───────────────────────────────────────┘
   ```

2. **Message Format**
   ```json
   {
     "job_id": "uuid",
     "type": "ocr_processing",
     "priority": 5,
     "payload": {
       "image_url": "s3://bucket/image.jpg",
       "options": {...}
     },
     "metadata": {
       "user_id": "123",
       "story_id": "456",
       "retry_count": 0
     }
   }
   ```

3. **Exchange Configuration**
   - Topic exchange for routing flexibility
   - Dead letter queues for failed jobs
   - TTL for time-sensitive tasks
   - Durable queues for persistence

### Worker Scaling Strategies

1. **Horizontal Scaling**
   - CPU-based autoscaling (target 70%)
   - Queue depth triggers
   - Time-based scaling for peaks
   - Minimum workers per station

2. **Scaling Rules**
   ```yaml
   scaling_rules:
     ocr_workers:
       min: 2
       max: 20
       scale_up_threshold: 100  # queue depth
       scale_down_threshold: 10
     audio_workers:
       min: 3
       max: 30
       scale_up_threshold: 50
       scale_down_threshold: 5
     media_workers:
       min: 2
       max: 15
       scale_up_threshold: 75
       scale_down_threshold: 15
   ```

3. **Resource Allocation**
   - OCR workers: CPU-optimized instances
   - Audio workers: Memory-optimized instances
   - Media workers: Network-optimized instances

### Job Prioritization

1. **Priority Levels**
   - Critical (1): Premium user requests
   - High (3): Standard user real-time requests
   - Normal (5): Batch processing
   - Low (7): Background reprocessing

2. **Prioritization Factors**
   - User tier (premium vs free)
   - Job age (prevent starvation)
   - Resource availability
   - Dependencies completion

3. **Fair Queuing**
   - Weighted round-robin per user
   - Burst allowance for new users
   - Rate limiting per account
   - Priority boost for waiting jobs

### Monitoring and Metrics

1. **Key Performance Indicators**
   ```
   - Job completion rate
   - Average processing time per stage
   - Queue depth and latency
   - Worker utilization
   - Error rates by type
   - Cost per processed item
   ```

2. **Monitoring Stack**
   - Prometheus for metrics collection
   - Grafana for visualization
   - AlertManager for notifications
   - ELK stack for log analysis

3. **Alerting Thresholds**
   - Queue depth > 1000 items
   - Processing time > 2x average
   - Error rate > 5%
   - Worker crashes > 3 in 5 minutes
   - Memory usage > 90%
   - Disk usage > 85%

4. **Health Checks**
   ```json
   {
     "status": "healthy",
     "workers": {
       "ocr": {"active": 5, "idle": 2},
       "audio": {"active": 8, "idle": 3},
       "media": {"active": 3, "idle": 1}
     },
     "queue_depths": {
       "ocr": 45,
       "audio": 123,
       "media": 67
     },
     "processing_times": {
       "ocr_p95": "2.3s",
       "audio_p95": "5.7s",
       "media_p95": "8.2s"
     }
   }
   ```

## Implementation Phases

### Phase 1: MVP (Months 1-2)
- Basic OCR with Google Vision
- Single voice TTS with Google
- Simple HLS streaming
- Manual scaling

### Phase 2: Enhancement (Months 3-4)
- Advanced preprocessing
- Multiple TTS providers
- CDN integration
- Auto-scaling implementation

### Phase 3: Optimization (Months 5-6)
- ML-based voice selection
- Advanced audio processing
- Performance optimization
- Comprehensive monitoring

## Security Considerations

1. **Data Protection**
   - Encryption at rest and in transit
   - PII detection and masking
   - Audit logging for all operations
   - GDPR compliance for data retention

2. **Access Control**
   - Service-to-service authentication
   - API key rotation
   - Network isolation per station
   - Least privilege principles

3. **Input Validation**
   - File type verification
   - Size limits enforcement
   - Content scanning for malware
   - Rate limiting per user

## Cost Optimization

1. **Resource Management**
   - Spot instances for batch jobs
   - Reserved instances for baseline
   - Automatic shutdown during idle
   - Region-based processing

2. **Provider Strategy**
   - Negotiated rates for volume
   - Multi-provider arbitrage
   - Caching to reduce API calls
   - Batch processing discounts

3. **Monitoring Costs**
   - Cost per story processed
   - Provider comparison dashboard
   - Budget alerts and caps
   - Usage forecasting