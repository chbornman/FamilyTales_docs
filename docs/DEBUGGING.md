# FamilyTales Debugging Guide

This guide covers common debugging scenarios and troubleshooting techniques for the FamilyTales application.

## Table of Contents
- [OCR Accuracy Issues](#ocr-accuracy-issues)
- [Audio Generation Debugging](#audio-generation-debugging)
- [Family Sharing Permission Issues](#family-sharing-permission-issues)
- [Flutter-Specific Debugging](#flutter-specific-debugging)
- [Rust Backend Debugging](#rust-backend-debugging)
- [Performance Profiling](#performance-profiling)

## OCR Accuracy Issues

### Common Problems

1. **Poor Image Quality**
   - Check image resolution (minimum 300 DPI recommended)
   - Verify image format (PNG/JPEG preferred)
   - Look for blurry or skewed images

2. **Text Recognition Failures**
   ```rust
   // Enable debug logging for OCR processing
   RUST_LOG=familytales_ocr=debug cargo run
   ```

3. **Debugging OCR Pipeline**
   ```rust
   // Check OCR confidence scores
   let result = ocr_processor.process_image(&image_path)?;
   debug!("OCR confidence: {}", result.confidence);
   debug!("Detected text blocks: {}", result.text_blocks.len());
   ```

### Troubleshooting Steps

1. **Test with Sample Images**
   ```bash
   # Run OCR tests with debug output
   cargo test ocr_tests -- --nocapture
   ```

2. **Inspect Preprocessing**
   - Check image enhancement steps
   - Verify text orientation detection
   - Review binarization results

3. **OCR Engine Configuration**
   ```rust
   // Adjust OCR parameters for better accuracy
   let config = OcrConfig {
       language: "eng",
       page_segmentation_mode: PageSegMode::AutoOsd,
       confidence_threshold: 0.7,
   };
   ```

## Audio Generation Debugging

### Common Issues

1. **Voice Selection Problems**
   ```rust
   // Debug voice assignment
   RUST_LOG=familytales_audio=debug cargo run
   
   // Check available voices
   let voices = audio_service.list_available_voices()?;
   debug!("Available voices: {:?}", voices);
   ```

2. **Audio Quality Issues**
   - Verify audio bitrate settings
   - Check sample rate configuration
   - Review voice model parameters

3. **Performance Problems**
   ```rust
   // Profile audio generation
   let start = Instant::now();
   let audio = generator.generate_audio(&text, &voice_id)?;
   debug!("Audio generation took: {:?}", start.elapsed());
   ```

### Debugging Mux Integration

```rust
// Enable Mux debug logging
std::env::set_var("MUX_DEBUG", "true");

// Check Mux asset status
let asset = mux_client.get_asset(&asset_id).await?;
debug!("Mux asset status: {:?}", asset.status);
debug!("Playback IDs: {:?}", asset.playback_ids);
```

## Family Sharing Permission Issues

### Permission Matrix Debugging

1. **Check User Permissions**
   ```rust
   // Debug permission checks
   let permissions = family_service.get_user_permissions(&user_id, &family_id)?;
   debug!("User {} permissions in family {}: {:?}", user_id, family_id, permissions);
   ```

2. **Family Hierarchy Issues**
   ```sql
   -- Check family relationships
   SELECT 
       f.id as family_id,
       fm.user_id,
       fm.role,
       fm.permissions
   FROM families f
   JOIN family_members fm ON f.id = fm.family_id
   WHERE f.id = $1;
   ```

3. **Story Access Problems**
   ```rust
   // Debug story visibility
   let accessible_stories = story_service.get_accessible_stories(&user_id)?;
   debug!("User {} can access {} stories", user_id, accessible_stories.len());
   ```

### Common Permission Scenarios

1. **Admin Cannot Share Story**
   - Verify family membership
   - Check story ownership
   - Review sharing settings

2. **Member Cannot View Story**
   - Confirm story is shared with family
   - Check member's view permissions
   - Verify story status (published/draft)

## Flutter-Specific Debugging

### Debug Tools

1. **Flutter Inspector**
   ```bash
   # Run with Flutter DevTools
   flutter run --debug
   # Open DevTools
   flutter pub global run devtools
   ```

2. **Widget Inspector**
   - Inspect widget tree
   - Check widget properties
   - Analyze render performance

3. **Network Debugging**
   ```dart
   // Enable network logging
   if (kDebugMode) {
     HttpClient.enableTimelineLogging = true;
   }
   ```

### Common Flutter Issues

1. **State Management Debugging**
   ```dart
   // Debug Riverpod providers
   ref.listen<StoryState>(
     storyProvider,
     (previous, next) {
       debugPrint('Story state changed: $previous -> $next');
     },
   );
   ```

2. **Performance Issues**
   ```dart
   // Enable performance overlay
   MaterialApp(
     showPerformanceOverlay: kDebugMode,
     // ... other properties
   );
   ```

3. **Memory Leaks**
   ```dart
   // Use Flutter DevTools Memory tab
   // Look for:
   // - Retained objects
   // - Growing memory usage
   // - Unreleased resources
   ```

## Rust Backend Debugging

### Logging Configuration

```toml
# In Cargo.toml
[dependencies]
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
```

```rust
// Initialize tracing
tracing_subscriber::fmt()
    .with_env_filter("familytales=debug,tower_http=debug")
    .init();
```

### Database Debugging

1. **Query Performance**
   ```rust
   // Log SQL queries with timing
   sqlx::query!("SELECT * FROM stories WHERE family_id = $1")
       .bind(family_id)
       .fetch_all(&pool)
       .instrument(info_span!("fetch_family_stories"))
       .await?;
   ```

2. **Connection Pool Issues**
   ```rust
   // Monitor connection pool
   let pool_options = PgPoolOptions::new()
       .max_connections(5)
       .acquire_timeout(Duration::from_secs(3))
       .after_connect(|conn| {
           Box::pin(async move {
               debug!("New connection established");
               Ok(())
           })
       });
   ```

### API Debugging

```rust
// Request/Response logging middleware
let app = Router::new()
    .layer(
        TraceLayer::new_for_http()
            .make_span_with(DefaultMakeSpan::default())
            .on_request(|request: &Request<_>, _span: &Span| {
                debug!("Request: {} {}", request.method(), request.uri());
            })
            .on_response(|response: &Response<_>, latency: Duration, _span: &Span| {
                debug!("Response: {} in {:?}", response.status(), latency);
            })
    );
```

## Performance Profiling

### Backend Profiling

1. **CPU Profiling**
   ```bash
   # Use cargo flamegraph
   cargo install flamegraph
   cargo flamegraph --bin familytales-api
   ```

2. **Memory Profiling**
   ```bash
   # Use valgrind
   valgrind --tool=massif target/release/familytales-api
   ms_print massif.out.<pid>
   ```

3. **Async Runtime Analysis**
   ```rust
   // Tokio console
   console_subscriber::init();
   
   // Run with tokio-console
   tokio-console http://localhost:6669
   ```

### Flutter Performance

1. **Frame Rendering**
   ```dart
   // Enable timeline events
   Timeline.startSync('ExpensiveOperation');
   // ... expensive operation
   Timeline.finishSync();
   ```

2. **Build Performance**
   ```dart
   // Identify expensive builds
   class ExpensiveWidget extends StatelessWidget {
     @override
     Widget build(BuildContext context) {
       debugPrint('Building ExpensiveWidget');
       // ... widget tree
     }
   }
   ```

3. **Memory Profiling**
   - Use Flutter DevTools Memory view
   - Track object allocation
   - Identify memory leaks

### Database Performance

1. **Query Analysis**
   ```sql
   -- Explain analyze queries
   EXPLAIN ANALYZE
   SELECT s.*, array_agg(st.tag_name) as tags
   FROM stories s
   LEFT JOIN story_tags st ON s.id = st.story_id
   WHERE s.family_id = $1
   GROUP BY s.id;
   ```

2. **Index Usage**
   ```sql
   -- Check missing indexes
   SELECT 
       schemaname,
       tablename,
       attname,
       n_distinct,
       most_common_vals
   FROM pg_stats
   WHERE tablename = 'stories';
   ```

## Debugging Checklist

### Before Reporting an Issue

- [ ] Check application logs
- [ ] Verify environment configuration
- [ ] Test in isolation (unit test)
- [ ] Reproduce consistently
- [ ] Collect relevant metrics

### Information to Collect

1. **Error Details**
   - Full error message
   - Stack trace
   - Request/Response data

2. **Environment**
   - OS and version
   - Flutter/Dart version
   - Rust version
   - Database version

3. **Steps to Reproduce**
   - Exact sequence of actions
   - Input data used
   - Expected vs actual behavior

### Debug Commands Reference

```bash
# Rust backend
RUST_LOG=debug cargo run
RUST_BACKTRACE=full cargo test failing_test

# Flutter app
flutter run --verbose
flutter logs
flutter analyze

# Database
psql -d familytales -c "SELECT version();"
redis-cli ping

# System
docker-compose logs -f api
docker stats
```