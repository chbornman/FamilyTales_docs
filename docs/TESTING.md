# FamilyTales Testing Strategy

This guide outlines the comprehensive testing approach for FamilyTales, covering all aspects from unit tests to elder user acceptance testing.

## Table of Contents
- [Testing Philosophy](#testing-philosophy)
- [Unit Tests for OCR Processing](#unit-tests-for-ocr-processing)
- [Integration Tests for Family Management](#integration-tests-for-family-management)
- [Flutter Widget Tests](#flutter-widget-tests)
- [End-to-End Tests](#end-to-end-tests)
- [Performance Testing](#performance-testing)
- [Elder User Acceptance Testing](#elder-user-acceptance-testing)
- [CI/CD Integration](#cicd-integration)

## Testing Philosophy

### Core Principles

1. **Test Pyramid**
   - Many unit tests (fast, isolated)
   - Moderate integration tests (feature verification)
   - Few E2E tests (critical user journeys)

2. **Elder-First Testing**
   - Accessibility is not optional
   - Simplicity over features
   - Clear feedback always

3. **Performance Standards**
   - OCR: < 2 seconds per page
   - Audio generation: < 5 seconds per story
   - App startup: < 3 seconds

## Unit Tests for OCR Processing

### Test Structure

```rust
// backend/tests/unit/ocr_tests.rs
use familytales_core::ocr::{OcrProcessor, OcrConfig, ImagePreprocessor};
use image::{DynamicImage, ImageBuffer};

mod ocr_processor_tests {
    use super::*;
    
    #[test]
    fn test_basic_text_extraction() {
        // Arrange
        let processor = OcrProcessor::new(OcrConfig::default());
        let test_image = load_test_image("simple_text.png");
        
        // Act
        let result = processor.extract_text(&test_image).unwrap();
        
        // Assert
        assert!(result.confidence > 0.8);
        assert!(result.text.contains("Once upon a time"));
        assert_eq!(result.text_blocks.len(), 1);
    }
    
    #[test]
    fn test_handwriting_detection() {
        let processor = OcrProcessor::new(OcrConfig {
            enable_handwriting: true,
            ..Default::default()
        });
        
        let handwritten_image = load_test_image("handwritten_note.jpg");
        let result = processor.extract_text(&handwritten_image).unwrap();
        
        assert!(result.is_handwritten);
        assert!(result.confidence > 0.6); // Lower threshold for handwriting
    }
    
    #[test]
    fn test_multi_column_layout() {
        let processor = OcrProcessor::new(OcrConfig::default());
        let newspaper_image = load_test_image("newspaper_layout.png");
        
        let result = processor.extract_text(&newspaper_image).unwrap();
        
        assert!(result.text_blocks.len() > 1);
        // Verify correct reading order
        let first_block = &result.text_blocks[0];
        let second_block = &result.text_blocks[1];
        assert!(first_block.bounding_box.y < second_block.bounding_box.y ||
                first_block.bounding_box.x < second_block.bounding_box.x);
    }
}

mod image_preprocessing_tests {
    use super::*;
    
    #[test]
    fn test_image_enhancement() {
        let preprocessor = ImagePreprocessor::new();
        let blurry_image = load_test_image("blurry_photo.jpg");
        
        let enhanced = preprocessor.enhance(&blurry_image).unwrap();
        
        // Check sharpness improvement
        let original_sharpness = calculate_sharpness(&blurry_image);
        let enhanced_sharpness = calculate_sharpness(&enhanced);
        assert!(enhanced_sharpness > original_sharpness * 1.2);
    }
    
    #[test]
    fn test_skew_correction() {
        let preprocessor = ImagePreprocessor::new();
        let skewed_image = create_skewed_text_image(15.0); // 15 degree skew
        
        let corrected = preprocessor.correct_skew(&skewed_image).unwrap();
        
        let detected_skew = preprocessor.detect_skew_angle(&corrected).unwrap();
        assert!(detected_skew.abs() < 1.0); // Less than 1 degree remaining
    }
    
    #[test]
    fn test_noise_reduction() {
        let preprocessor = ImagePreprocessor::new();
        let noisy_image = add_salt_pepper_noise(&load_test_image("clean_text.png"), 0.1);
        
        let denoised = preprocessor.reduce_noise(&noisy_image).unwrap();
        
        let psnr = calculate_psnr(&load_test_image("clean_text.png"), &denoised);
        assert!(psnr > 30.0); // Good quality threshold
    }
}

mod ocr_accuracy_tests {
    use super::*;
    
    #[test]
    fn test_accuracy_on_printed_text() {
        let processor = OcrProcessor::new(OcrConfig::default());
        let test_cases = vec![
            ("arial_12pt.png", "The quick brown fox jumps over the lazy dog", 0.95),
            ("times_10pt.png", "In a hole in the ground there lived a hobbit", 0.93),
            ("comic_sans_14pt.png", "Not all those who wander are lost", 0.90),
        ];
        
        for (image_file, expected_text, min_accuracy) in test_cases {
            let image = load_test_image(image_file);
            let result = processor.extract_text(&image).unwrap();
            
            let accuracy = calculate_text_similarity(&result.text, expected_text);
            assert!(
                accuracy >= min_accuracy,
                "OCR accuracy {} below threshold {} for {}",
                accuracy, min_accuracy, image_file
            );
        }
    }
    
    #[test]
    fn test_special_characters_and_punctuation() {
        let processor = OcrProcessor::new(OcrConfig::default());
        let test_text = "Hello, World! How are you? I'm fine @ 100% :)";
        let image = create_text_image(test_text);
        
        let result = processor.extract_text(&image).unwrap();
        
        assert_eq!(result.text.trim(), test_text);
    }
    
    #[test]
    fn test_mixed_language_support() {
        let processor = OcrProcessor::new(OcrConfig {
            languages: vec!["eng", "spa", "fra"],
            ..Default::default()
        });
        
        let mixed_text = "Hello! ¡Hola! Bonjour! Welcome, Bienvenido, Bienvenue!";
        let image = create_text_image(mixed_text);
        
        let result = processor.extract_text(&image).unwrap();
        
        assert!(result.text.contains("Hello"));
        assert!(result.text.contains("¡Hola!"));
        assert!(result.text.contains("Bonjour"));
    }
}

// Helper functions
fn load_test_image(filename: &str) -> DynamicImage {
    let path = format!("tests/fixtures/ocr/{}", filename);
    image::open(path).expect("Failed to load test image")
}

fn calculate_text_similarity(text1: &str, text2: &str) -> f64 {
    use edit_distance::edit_distance;
    let distance = edit_distance(text1, text2);
    let max_len = text1.len().max(text2.len());
    1.0 - (distance as f64 / max_len as f64)
}
```

### Property-Based Tests

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn test_ocr_never_panics(
        width in 100..2000u32,
        height in 100..2000u32,
        noise_level in 0.0..0.5f32,
    ) {
        let image = generate_random_text_image(width, height, noise_level);
        let processor = OcrProcessor::new(OcrConfig::default());
        
        // Should never panic, regardless of input
        let _ = processor.extract_text(&image);
    }
    
    #[test]
    fn test_preprocessing_maintains_aspect_ratio(
        width in 100..4000u32,
        height in 100..4000u32,
    ) {
        let image = DynamicImage::new_rgb8(width, height);
        let preprocessor = ImagePreprocessor::new();
        
        let processed = preprocessor.enhance(&image).unwrap();
        
        let original_ratio = width as f64 / height as f64;
        let processed_ratio = processed.width() as f64 / processed.height() as f64;
        
        prop_assert!((original_ratio - processed_ratio).abs() < 0.01);
    }
}
```

## Integration Tests for Family Management

### Database Integration Tests

```rust
// backend/tests/integration/family_management_tests.rs
use familytales::test_helpers::{TestDb, create_test_user, create_test_family};
use familytales_core::services::{FamilyService, PermissionService};

#[tokio::test]
async fn test_create_family_with_admin() {
    // Setup
    let db = TestDb::new().await;
    let family_service = FamilyService::new(db.pool());
    let user = create_test_user(&db, "test@example.com").await;
    
    // Act
    let family = family_service.create_family(
        &user.id,
        "The Smiths",
        Some("Our family stories"),
    ).await.unwrap();
    
    // Assert
    assert_eq!(family.name, "The Smiths");
    
    // Verify creator is admin
    let member = family_service
        .get_member(&family.id, &user.id)
        .await
        .unwrap();
    assert_eq!(member.role, "admin");
}

#[tokio::test]
async fn test_invite_family_member() {
    let db = TestDb::new().await;
    let family_service = FamilyService::new(db.pool());
    let permission_service = PermissionService::new(db.pool());
    
    // Setup family with admin
    let admin = create_test_user(&db, "admin@example.com").await;
    let family = create_test_family(&db, &admin.id, "Test Family").await;
    
    // Invite new member
    let new_member = create_test_user(&db, "member@example.com").await;
    let invitation = family_service.invite_member(
        &family.id,
        &admin.id,
        &new_member.email,
        "member",
    ).await.unwrap();
    
    // Accept invitation
    family_service.accept_invitation(
        &invitation.token,
        &new_member.id,
    ).await.unwrap();
    
    // Verify permissions
    let can_view = permission_service.can_view_family(
        &new_member.id,
        &family.id,
    ).await.unwrap();
    assert!(can_view);
    
    let can_admin = permission_service.can_admin_family(
        &new_member.id,
        &family.id,
    ).await.unwrap();
    assert!(!can_admin);
}

#[tokio::test]
async fn test_family_story_sharing() {
    let db = TestDb::new().await;
    let family_service = FamilyService::new(db.pool());
    let story_service = StoryService::new(db.pool());
    
    // Create two families
    let user1 = create_test_user(&db, "user1@example.com").await;
    let family1 = create_test_family(&db, &user1.id, "Family 1").await;
    
    let user2 = create_test_user(&db, "user2@example.com").await;
    let family2 = create_test_family(&db, &user2.id, "Family 2").await;
    
    // Create story in family1
    let story = story_service.create_story(
        &user1.id,
        &family1.id,
        "Private Story",
        "Content",
    ).await.unwrap();
    
    // User2 shouldn't see the story
    let user2_stories = story_service
        .get_accessible_stories(&user2.id)
        .await
        .unwrap();
    assert!(!user2_stories.iter().any(|s| s.id == story.id));
    
    // Add user2 to family1
    family_service.add_member(
        &family1.id,
        &user1.id,
        &user2.id,
        "member",
    ).await.unwrap();
    
    // Now user2 should see the story
    let user2_stories = story_service
        .get_accessible_stories(&user2.id)
        .await
        .unwrap();
    assert!(user2_stories.iter().any(|s| s.id == story.id));
}

#[tokio::test]
async fn test_remove_family_member_cascade() {
    let db = TestDb::new().await;
    let family_service = FamilyService::new(db.pool());
    
    // Setup
    let admin = create_test_user(&db, "admin@example.com").await;
    let member = create_test_user(&db, "member@example.com").await;
    let family = create_test_family(&db, &admin.id, "Test Family").await;
    
    family_service.add_member(
        &family.id,
        &admin.id,
        &member.id,
        "member",
    ).await.unwrap();
    
    // Create member's story
    let story = story_service.create_story(
        &member.id,
        &family.id,
        "Member's Story",
        "Content",
    ).await.unwrap();
    
    // Remove member
    family_service.remove_member(
        &family.id,
        &admin.id,
        &member.id,
    ).await.unwrap();
    
    // Verify member can't access family anymore
    let can_view = permission_service.can_view_family(
        &member.id,
        &family.id,
    ).await.unwrap();
    assert!(!can_view);
    
    // Verify story still exists but member can't access it
    let story_exists = story_service.get_story(&story.id).await.is_ok();
    assert!(story_exists);
    
    let member_stories = story_service
        .get_accessible_stories(&member.id)
        .await
        .unwrap();
    assert!(!member_stories.iter().any(|s| s.id == story.id));
}
```

### API Integration Tests

```rust
// backend/tests/integration/api_tests.rs
use axum::http::StatusCode;
use familytales::test_helpers::{TestApp, create_auth_header};

#[tokio::test]
async fn test_family_api_workflow() {
    let app = TestApp::new().await;
    let user_token = app.create_test_user("test@example.com").await;
    
    // Create family
    let response = app.client
        .post("/api/families")
        .header("Authorization", create_auth_header(&user_token))
        .json(&json!({
            "name": "Test Family",
            "description": "Integration test family"
        }))
        .send()
        .await
        .unwrap();
    
    assert_eq!(response.status(), StatusCode::CREATED);
    let family: Family = response.json().await.unwrap();
    
    // List user's families
    let response = app.client
        .get("/api/families")
        .header("Authorization", create_auth_header(&user_token))
        .send()
        .await
        .unwrap();
    
    assert_eq!(response.status(), StatusCode::OK);
    let families: Vec<Family> = response.json().await.unwrap();
    assert_eq!(families.len(), 1);
    assert_eq!(families[0].id, family.id);
    
    // Invite member
    let response = app.client
        .post(&format!("/api/families/{}/invitations", family.id))
        .header("Authorization", create_auth_header(&user_token))
        .json(&json!({
            "email": "newmember@example.com",
            "role": "member"
        }))
        .send()
        .await
        .unwrap();
    
    assert_eq!(response.status(), StatusCode::CREATED);
}

#[tokio::test]
async fn test_story_lifecycle() {
    let app = TestApp::new().await;
    let user_token = app.create_test_user("storyteller@example.com").await;
    let family_id = app.create_test_family(&user_token, "Story Family").await;
    
    // Upload story images
    let image_urls = vec![];
    for i in 1..=3 {
        let form = multipart::Form::new()
            .part(
                "file",
                multipart::Part::bytes(include_bytes!("../fixtures/page1.jpg"))
                    .file_name(format!("page{}.jpg", i))
                    .mime_str("image/jpeg")
                    .unwrap(),
            );
        
        let response = app.client
            .post("/api/upload")
            .header("Authorization", create_auth_header(&user_token))
            .multipart(form)
            .send()
            .await
            .unwrap();
        
        let upload_result: UploadResult = response.json().await.unwrap();
        image_urls.push(upload_result.url);
    }
    
    // Create story with images
    let response = app.client
        .post("/api/stories")
        .header("Authorization", create_auth_header(&user_token))
        .json(&json!({
            "family_id": family_id,
            "title": "Test Story",
            "pages": image_urls.iter().enumerate().map(|(i, url)| {
                json!({
                    "page_number": i + 1,
                    "image_url": url
                })
            }).collect::<Vec<_>>()
        }))
        .send()
        .await
        .unwrap();
    
    assert_eq!(response.status(), StatusCode::CREATED);
    let story: Story = response.json().await.unwrap();
    
    // Wait for OCR processing
    tokio::time::sleep(Duration::from_secs(5)).await;
    
    // Check story status
    let response = app.client
        .get(&format!("/api/stories/{}", story.id))
        .header("Authorization", create_auth_header(&user_token))
        .send()
        .await
        .unwrap();
    
    let updated_story: Story = response.json().await.unwrap();
    assert_eq!(updated_story.status, "processing");
    
    // Edit OCR text
    let response = app.client
        .put(&format!("/api/stories/{}/pages/1", story.id))
        .header("Authorization", create_auth_header(&user_token))
        .json(&json!({
            "edited_text": "Once upon a time in a land far away..."
        }))
        .send()
        .await
        .unwrap();
    
    assert_eq!(response.status(), StatusCode::OK);
    
    // Generate audio
    let response = app.client
        .post(&format!("/api/stories/{}/generate-audio", story.id))
        .header("Authorization", create_auth_header(&user_token))
        .json(&json!({
            "narrator_voice": "grandpa_warm"
        }))
        .send()
        .await
        .unwrap();
    
    assert_eq!(response.status(), StatusCode::ACCEPTED);
}
```

## Flutter Widget Tests

### Basic Widget Tests

```dart
// mobile/test/widgets/story_card_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:familytales/widgets/story_card.dart';
import 'package:familytales/models/story.dart';

void main() {
  group('StoryCard Widget', () {
    testWidgets('displays story title and thumbnail', (tester) async {
      // Arrange
      final story = Story(
        id: '123',
        title: 'Test Story',
        thumbnailUrl: 'https://example.com/thumb.jpg',
        duration: 180,
        createdAt: DateTime.now(),
      );
      
      // Act
      await tester.pumpWidget(
        MaterialApp(
          home: Scaffold(
            body: StoryCard(story: story),
          ),
        ),
      );
      
      // Assert
      expect(find.text('Test Story'), findsOneWidget);
      expect(find.byType(Image), findsOneWidget);
      expect(find.text('3:00'), findsOneWidget); // Duration
    });
    
    testWidgets('shows play button on hover', (tester) async {
      final story = Story(
        id: '123',
        title: 'Test Story',
        thumbnailUrl: 'https://example.com/thumb.jpg',
      );
      
      await tester.pumpWidget(
        MaterialApp(
          home: Scaffold(
            body: StoryCard(story: story),
          ),
        ),
      );
      
      // Initially no play button
      expect(find.byIcon(Icons.play_circle_filled), findsNothing);
      
      // Hover over card
      final gesture = await tester.createGesture(kind: PointerDeviceKind.mouse);
      await gesture.addPointer();
      await gesture.moveTo(tester.getCenter(find.byType(StoryCard)));
      await tester.pumpAndSettle();
      
      // Play button should appear
      expect(find.byIcon(Icons.play_circle_filled), findsOneWidget);
    });
    
    testWidgets('navigates to story detail on tap', (tester) async {
      final story = Story(id: '123', title: 'Test Story');
      bool navigated = false;
      
      await tester.pumpWidget(
        MaterialApp(
          home: Scaffold(
            body: StoryCard(
              story: story,
              onTap: () => navigated = true,
            ),
          ),
        ),
      );
      
      await tester.tap(find.byType(StoryCard));
      expect(navigated, isTrue);
    });
  });
}
```

### Complex Widget Tests

```dart
// mobile/test/widgets/story_editor_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mockito/mockito.dart';
import 'package:familytales/widgets/story_editor.dart';
import 'package:familytales/services/story_service.dart';

class MockStoryService extends Mock implements StoryService {}

void main() {
  late MockStoryService mockStoryService;
  
  setUp(() {
    mockStoryService = MockStoryService();
  });
  
  group('StoryEditor Widget', () {
    testWidgets('displays OCR text and allows editing', (tester) async {
      // Arrange
      final pages = [
        StoryPage(
          pageNumber: 1,
          imageUrl: 'https://example.com/page1.jpg',
          ocrText: 'Original OCR text',
          editedText: null,
        ),
      ];
      
      when(mockStoryService.getStoryPages(any))
          .thenAnswer((_) async => pages);
      
      // Act
      await tester.pumpWidget(
        MaterialApp(
          home: StoryEditor(
            storyId: '123',
            storyService: mockStoryService,
          ),
        ),
      );
      await tester.pumpAndSettle();
      
      // Assert - OCR text is displayed
      expect(find.text('Original OCR text'), findsOneWidget);
      
      // Edit text
      await tester.enterText(
        find.byType(TextField),
        'Edited text by user',
      );
      await tester.testTextInput.receiveAction(TextInputAction.done);
      await tester.pumpAndSettle();
      
      // Verify save was called
      verify(mockStoryService.updatePageText(
        '123',
        1,
        'Edited text by user',
      )).called(1);
    });
    
    testWidgets('supports page navigation', (tester) async {
      // Arrange
      final pages = List.generate(
        5,
        (i) => StoryPage(
          pageNumber: i + 1,
          imageUrl: 'https://example.com/page${i + 1}.jpg',
          ocrText: 'Page ${i + 1} text',
        ),
      );
      
      when(mockStoryService.getStoryPages(any))
          .thenAnswer((_) async => pages);
      
      // Act
      await tester.pumpWidget(
        MaterialApp(
          home: StoryEditor(
            storyId: '123',
            storyService: mockStoryService,
          ),
        ),
      );
      await tester.pumpAndSettle();
      
      // Assert - First page is shown
      expect(find.text('Page 1 text'), findsOneWidget);
      expect(find.text('Page 1 of 5'), findsOneWidget);
      
      // Navigate to next page
      await tester.tap(find.byIcon(Icons.arrow_forward));
      await tester.pumpAndSettle();
      
      expect(find.text('Page 2 text'), findsOneWidget);
      expect(find.text('Page 2 of 5'), findsOneWidget);
      
      // Navigate to last page
      await tester.tap(find.byIcon(Icons.last_page));
      await tester.pumpAndSettle();
      
      expect(find.text('Page 5 text'), findsOneWidget);
      expect(find.text('Page 5 of 5'), findsOneWidget);
    });
  });
}
```

### Accessibility Widget Tests

```dart
// mobile/test/widgets/accessibility_test.dart
void main() {
  group('Accessibility Tests', () {
    testWidgets('all interactive elements have semantic labels', (tester) async {
      await tester.pumpWidget(
        MaterialApp(
          home: StoryListScreen(),
        ),
      );
      await tester.pumpAndSettle();
      
      // Check all buttons have labels
      final buttons = find.byType(IconButton);
      for (int i = 0; i < buttons.evaluate().length; i++) {
        final button = buttons.at(i);
        final semantics = tester.getSemantics(button);
        expect(semantics.label, isNotEmpty);
      }
      
      // Check images have descriptions
      final images = find.byType(Image);
      for (int i = 0; i < images.evaluate().length; i++) {
        final image = images.at(i);
        final semantics = tester.getSemantics(image);
        expect(semantics.label ?? semantics.value, isNotEmpty);
      }
    });
    
    testWidgets('supports large text scaling', (tester) async {
      await tester.pumpWidget(
        MaterialApp(
          home: MediaQuery(
            data: const MediaQueryData(textScaleFactor: 2.0),
            child: StoryCard(
              story: Story(id: '1', title: 'Test Story'),
            ),
          ),
        ),
      );
      
      // Verify text doesn't overflow
      expect(find.byType(Text), findsWidgets);
      expect(tester.takeException(), isNull);
    });
    
    testWidgets('supports screen reader navigation', (tester) async {
      final semanticsHandle = tester.ensureSemantics();
      
      await tester.pumpWidget(
        MaterialApp(
          home: StoryDetailScreen(storyId: '123'),
        ),
      );
      await tester.pumpAndSettle();
      
      // Get semantic tree
      final semantics = tester.getSemantics(find.byType(StoryDetailScreen));
      
      // Verify proper heading structure
      expect(semantics.hasFlag(ui.SemanticsFlag.isHeader), isTrue);
      
      semanticsHandle.dispose();
    });
  });
}
```

## End-to-End Tests

### Critical User Journeys

```dart
// mobile/integration_test/app_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';
import 'package:familytales/main.dart' as app;

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();
  
  group('End-to-End Tests', () {
    testWidgets('complete story creation flow', (tester) async {
      // Start app
      app.main();
      await tester.pumpAndSettle();
      
      // Login
      await _performLogin(tester, 'test@example.com', 'password123');
      
      // Navigate to create story
      await tester.tap(find.byIcon(Icons.add));
      await tester.pumpAndSettle();
      
      // Select family
      await tester.tap(find.text('The Smiths'));
      await tester.pumpAndSettle();
      
      // Add photos
      await tester.tap(find.text('Add Photos'));
      await tester.pumpAndSettle();
      
      // Select photos from gallery (mocked in test mode)
      await tester.tap(find.byKey(Key('photo_1')));
      await tester.tap(find.byKey(Key('photo_2')));
      await tester.tap(find.byKey(Key('photo_3')));
      await tester.tap(find.text('Done'));
      await tester.pumpAndSettle();
      
      // Enter story title
      await tester.enterText(
        find.byKey(Key('story_title_field')),
        'Grandma\'s Birthday',
      );
      
      // Start processing
      await tester.tap(find.text('Create Story'));
      await tester.pumpAndSettle();
      
      // Wait for OCR (with timeout)
      await tester.pump(Duration(seconds: 10));
      
      // Verify OCR results shown
      expect(find.text('Edit Story Text'), findsOneWidget);
      
      // Edit first page
      final firstPageField = find.byKey(Key('page_1_text'));
      await tester.tap(firstPageField);
      await tester.enterText(
        firstPageField,
        'It was grandma\'s 80th birthday...',
      );
      
      // Generate audio
      await tester.tap(find.text('Generate Audio'));
      await tester.pumpAndSettle();
      
      // Select voice
      await tester.tap(find.text('Grandpa - Warm'));
      await tester.tap(find.text('Generate'));
      await tester.pumpAndSettle();
      
      // Wait for audio generation
      await tester.pump(Duration(seconds: 15));
      
      // Verify success
      expect(find.text('Story Ready!'), findsOneWidget);
      expect(find.byIcon(Icons.play_arrow), findsOneWidget);
    });
    
    testWidgets('elder-friendly playback flow', (tester) async {
      app.main();
      await tester.pumpAndSettle();
      
      // Simple login
      await _performElderLogin(tester);
      
      // Large touch targets
      final playButton = find.byKey(Key('large_play_button'));
      expect(playButton, findsOneWidget);
      
      // Get button size
      final buttonSize = tester.getSize(playButton);
      expect(buttonSize.width, greaterThanOrEqualTo(60));
      expect(buttonSize.height, greaterThanOrEqualTo(60));
      
      // Tap to play
      await tester.tap(playButton);
      await tester.pumpAndSettle();
      
      // Verify audio controls are visible and large
      expect(find.byKey(Key('audio_progress')), findsOneWidget);
      expect(find.byKey(Key('large_pause_button')), findsOneWidget);
      
      // Test gesture controls
      await tester.drag(
        find.byKey(Key('audio_progress')),
        Offset(100, 0),
      );
      await tester.pumpAndSettle();
      
      // Verify seeking worked
      expect(find.text('0:30'), findsOneWidget); // Current time
    });
  });
}

Future<void> _performLogin(
  WidgetTester tester,
  String email,
  String password,
) async {
  await tester.enterText(find.byKey(Key('email_field')), email);
  await tester.enterText(find.byKey(Key('password_field')), password);
  await tester.tap(find.text('Login'));
  await tester.pumpAndSettle();
}

Future<void> _performElderLogin(WidgetTester tester) async {
  // Simplified login with PIN
  for (final digit in '1234'.split('')) {
    await tester.tap(find.text(digit));
    await tester.pump(Duration(milliseconds: 200));
  }
  await tester.pumpAndSettle();
}
```

## Performance Testing

### OCR Performance Tests

```rust
// backend/tests/performance/ocr_bench.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion, BenchmarkId};
use familytales_core::ocr::OcrProcessor;

fn benchmark_ocr_processing(c: &mut Criterion) {
    let processor = OcrProcessor::new(Default::default());
    let test_images = vec![
        ("small", load_test_image("small_300dpi.png")),    // 1 page
        ("medium", load_test_image("medium_300dpi.png")),  // 5 pages
        ("large", load_test_image("large_300dpi.png")),    // 20 pages
    ];
    
    let mut group = c.benchmark_group("ocr_processing");
    
    for (name, image) in test_images {
        group.bench_with_input(
            BenchmarkId::from_parameter(name),
            &image,
            |b, img| {
                b.iter(|| {
                    processor.extract_text(black_box(img))
                });
            },
        );
    }
    
    group.finish();
}

fn benchmark_preprocessing(c: &mut Criterion) {
    let preprocessor = ImagePreprocessor::new();
    
    c.bench_function("denoise_image", |b| {
        let noisy_image = create_noisy_image(1920, 1080, 0.1);
        b.iter(|| {
            preprocessor.reduce_noise(black_box(&noisy_image))
        });
    });
    
    c.bench_function("deskew_image", |b| {
        let skewed_image = create_skewed_image(2048, 2048, 15.0);
        b.iter(|| {
            preprocessor.correct_skew(black_box(&skewed_image))
        });
    });
}

criterion_group!(benches, benchmark_ocr_processing, benchmark_preprocessing);
criterion_main!(benches);
```

### Audio Generation Performance

```rust
// backend/tests/performance/audio_bench.rs
#[tokio::test]
async fn test_audio_generation_performance() {
    let audio_service = AudioService::new(MuxClient::new("token", "secret"));
    
    let test_texts = vec![
        ("short", "Once upon a time."),  // ~2 seconds
        ("medium", include_str!("../fixtures/medium_story.txt")),  // ~1 minute
        ("long", include_str!("../fixtures/long_story.txt")),  // ~5 minutes
    ];
    
    for (name, text) in test_texts {
        let start = Instant::now();
        
        let result = audio_service
            .generate_audio(text, "grandma_gentle")
            .await
            .unwrap();
        
        let duration = start.elapsed();
        
        println!("{} story generation took: {:?}", name, duration);
        
        // Assert performance requirements
        match name {
            "short" => assert!(duration < Duration::from_secs(3)),
            "medium" => assert!(duration < Duration::from_secs(10)),
            "long" => assert!(duration < Duration::from_secs(30)),
            _ => unreachable!(),
        }
    }
}
```

### Flutter Performance Tests

```dart
// mobile/test/performance/app_performance_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';

void main() {
  final binding = IntegrationTestWidgetsFlutterBinding.ensureInitialized();
  
  testWidgets('app startup performance', (tester) async {
    final stopwatch = Stopwatch()..start();
    
    await tester.pumpWidget(MyApp());
    await tester.pumpAndSettle();
    
    stopwatch.stop();
    
    print('App startup took: ${stopwatch.elapsedMilliseconds}ms');
    expect(stopwatch.elapsedMilliseconds, lessThan(3000));
  });
  
  testWidgets('story list scroll performance', (tester) async {
    await tester.pumpWidget(MyApp());
    await tester.pumpAndSettle();
    
    // Navigate to story list
    await tester.tap(find.text('Stories'));
    await tester.pumpAndSettle();
    
    // Measure frame rendering during scroll
    await binding.watchPerformance(() async {
      await tester.fling(
        find.byType(ListView),
        Offset(0, -300),
        1000,
      );
      await tester.pumpAndSettle();
    }, reportKey: 'story_list_scroll');
  });
  
  testWidgets('image loading performance', (tester) async {
    await tester.pumpWidget(MyApp());
    await tester.pumpAndSettle();
    
    // Navigate to story with images
    await tester.tap(find.text('Photo Story'));
    await tester.pumpAndSettle();
    
    final stopwatch = Stopwatch()..start();
    
    // Wait for all images to load
    await tester.pump(Duration(seconds: 5));
    
    stopwatch.stop();
    
    // All images should load within 5 seconds
    final images = find.byType(Image);
    expect(images, findsWidgets);
    
    print('Image loading took: ${stopwatch.elapsedMilliseconds}ms');
  });
}
```

## Elder User Acceptance Testing

### Test Scenarios

```yaml
# test/uat/elder_test_scenarios.yaml
scenarios:
  - name: "First Time Setup"
    description: "Elder user sets up the app for the first time"
    participants:
      min_age: 65
      tech_experience: "minimal"
    steps:
      - "Download and open the app"
      - "Create account with email"
      - "Join family using invitation code"
      - "View first story"
    success_criteria:
      - "Completes setup in under 10 minutes"
      - "No assistance required for critical steps"
      - "Understands how to play stories"
    
  - name: "Daily Story Playback"
    description: "Elder user plays their daily story"
    participants:
      min_age: 70
      vision: "requires reading glasses"
    steps:
      - "Open app"
      - "Find today's story"
      - "Play story"
      - "Pause and resume"
      - "Adjust volume"
    success_criteria:
      - "Locates story in under 30 seconds"
      - "All controls clearly visible"
      - "Can pause/resume without confusion"
      
  - name: "Low Vision Usage"
    description: "User with low vision navigates the app"
    participants:
      vision: "20/200 or worse"
      uses_magnifier: true
    steps:
      - "Increase text size"
      - "Navigate to stories"
      - "Play story with audio"
      - "Return to home"
    success_criteria:
      - "All text readable at max size"
      - "High contrast mode effective"
      - "Audio descriptions helpful"
```

### UAT Test Protocol

```dart
// test/uat/elder_uat_framework.dart
class ElderUATFramework {
  final List<UATScenario> scenarios;
  final UATRecorder recorder;
  
  Future<UATResults> runSession({
    required Participant participant,
    required List<String> scenarioIds,
  }) async {
    final results = UATResults(
      participant: participant,
      startTime: DateTime.now(),
    );
    
    // Pre-test questionnaire
    results.preTestResponses = await conductPreTest(participant);
    
    // Run scenarios
    for (final scenarioId in scenarioIds) {
      final scenario = scenarios.firstWhere((s) => s.id == scenarioId);
      
      // Start recording
      await recorder.startRecording(
        scenarioId: scenario.id,
        participantId: participant.id,
      );
      
      // Present scenario
      await presentScenario(scenario);
      
      // Observe and record
      final observations = <Observation>[];
      final timer = Stopwatch()..start();
      
      for (final step in scenario.steps) {
        final observation = await observeStep(step);
        observation.duration = timer.elapsed;
        observations.add(observation);
        
        if (observation.required_assistance) {
          results.assistancePoints.add(AssistancePoint(
            step: step,
            reason: observation.assistance_reason,
            time: timer.elapsed,
          ));
        }
      }
      
      timer.stop();
      
      // Post-scenario questions
      final feedback = await collectScenarioFeedback(scenario);
      
      results.scenarioResults.add(ScenarioResult(
        scenario: scenario,
        observations: observations,
        totalTime: timer.elapsed,
        feedback: feedback,
        success: evaluateSuccess(observations, scenario.successCriteria),
      ));
      
      await recorder.stopRecording();
    }
    
    // Post-test questionnaire
    results.postTestResponses = await conductPostTest(participant);
    
    // System Usability Scale (SUS)
    results.susScore = await conductSUS(participant);
    
    return results;
  }
  
  Future<Map<String, dynamic>> conductPreTest(Participant participant) async {
    return await showQuestionnaire([
      Question(
        text: "How comfortable are you with smartphones?",
        type: QuestionType.scale,
        scale: 1..5,
        labels: {"1": "Not at all", "5": "Very comfortable"},
      ),
      Question(
        text: "Do you use any assistive technologies?",
        type: QuestionType.multipleChoice,
        options: ["Screen reader", "Magnifier", "Large text", "None"],
      ),
      Question(
        text: "How often do you use apps to view photos?",
        type: QuestionType.multipleChoice,
        options: ["Daily", "Weekly", "Monthly", "Rarely", "Never"],
      ),
    ]);
  }
  
  double calculateSUSScore(Map<String, int> responses) {
    // Standard SUS calculation
    var score = 0.0;
    
    // Odd questions (positive)
    for (int i = 1; i <= 10; i += 2) {
      score += (responses['q$i']! - 1);
    }
    
    // Even questions (negative)
    for (int i = 2; i <= 10; i += 2) {
      score += (5 - responses['q$i']!);
    }
    
    return score * 2.5; // Scale to 0-100
  }
}
```

### Accessibility Metrics

```dart
// test/uat/accessibility_metrics.dart
class AccessibilityMetrics {
  static Future<AccessibilityReport> analyze(UATResults results) async {
    return AccessibilityReport(
      touchTargetCompliance: _analyzeTouchTargets(results),
      contrastRatios: _analyzeContrast(results),
      textSizeAdequacy: _analyzeTextSizes(results),
      navigationClarity: _analyzeNavigation(results),
      errorRecovery: _analyzeErrorRecovery(results),
      cognitiveLoad: _analyzeCognitiveLoad(results),
    );
  }
  
  static double _analyzeTouchTargets(UATResults results) {
    final violations = results.observations
        .where((obs) => obs.touchTargetTooSmall)
        .length;
    
    final totalTargets = results.observations
        .where((obs) => obs.involvedTouchTarget)
        .length;
    
    return 1.0 - (violations / totalTargets);
  }
  
  static Map<String, double> _analyzeContrast(UATResults results) {
    return {
      'text_background': results.averageContrast('text'),
      'button_background': results.averageContrast('button'),
      'icon_background': results.averageContrast('icon'),
    };
  }
  
  static TextSizeAnalysis _analyzeTextSizes(UATResults results) {
    return TextSizeAnalysis(
      minSize: results.minTextSize,
      avgSize: results.avgTextSize,
      readabilityScore: _calculateReadability(results),
      scalingEffective: results.textScalingTests.all((t) => t.passed),
    );
  }
}
```

## CI/CD Integration

### GitHub Actions Workflow

```yaml
# .github/workflows/test.yml
name: Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  rust-tests:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
          
    steps:
      - uses: actions/checkout@v3
      
      - name: Install Rust
        uses: actions-rs/toolchain@v1
        with:
          toolchain: stable
          override: true
          
      - name: Cache dependencies
        uses: actions/cache@v3
        with:
          path: |
            ~/.cargo/registry
            ~/.cargo/git
            target
          key: ${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}
          
      - name: Install system dependencies
        run: |
          sudo apt-get update
          sudo apt-get install -y tesseract-ocr libtesseract-dev
          
      - name: Run migrations
        env:
          DATABASE_URL: postgres://postgres:postgres@localhost/familytales_test
        run: |
          cargo install sqlx-cli --no-default-features --features postgres
          sqlx database create
          sqlx migrate run
          
      - name: Run tests
        env:
          DATABASE_URL: postgres://postgres:postgres@localhost/familytales_test
        run: |
          cargo test --all-features
          
      - name: Run benchmarks
        run: cargo bench --no-run
        
  flutter-tests:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.16.0'
          channel: 'stable'
          
      - name: Install dependencies
        run: |
          cd mobile
          flutter pub get
          
      - name: Analyze code
        run: |
          cd mobile
          flutter analyze
          
      - name: Run tests
        run: |
          cd mobile
          flutter test --coverage
          
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: mobile/coverage/lcov.info
          
  integration-tests:
    runs-on: ubuntu-latest
    needs: [rust-tests, flutter-tests]
    steps:
      - uses: actions/checkout@v3
      
      - name: Start services
        run: docker-compose up -d
        
      - name: Wait for services
        run: |
          timeout 30 sh -c 'until nc -z localhost 5432; do sleep 1; done'
          timeout 30 sh -c 'until nc -z localhost 6379; do sleep 1; done'
          
      - name: Run E2E tests
        run: |
          cd mobile
          flutter drive --target=test_driver/app.dart
          
  performance-tests:
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v3
      
      - name: Run performance benchmarks
        run: |
          cd backend
          cargo bench -- --output-format bencher | tee output.txt
          
      - name: Store benchmark result
        uses: benchmark-action/github-action-benchmark@v1
        with:
          tool: 'cargo'
          output-file-path: backend/output.txt
          github-token: ${{ secrets.GITHUB_TOKEN }}
          auto-push: true
```

### Test Reports

```yaml
# .github/workflows/test-report.yml
name: Test Report

on:
  workflow_run:
    workflows: ["Tests"]
    types:
      - completed

jobs:
  report:
    runs-on: ubuntu-latest
    steps:
      - name: Download artifacts
        uses: actions/download-artifact@v3
        
      - name: Generate test report
        run: |
          # Combine test results
          cat rust-tests.xml flutter-tests.xml > all-tests.xml
          
      - name: Publish test results
        uses: EnricoMi/publish-unit-test-result-action@v2
        with:
          files: |
            **/*-tests.xml
            **/test-results/*.xml
          
      - name: Comment PR
        uses: actions/github-script@v6
        if: github.event.workflow_run.event == 'pull_request'
        with:
          script: |
            const fs = require('fs');
            const testResults = JSON.parse(
              fs.readFileSync('test-summary.json', 'utf8')
            );
            
            const comment = `## Test Results
            
            ✅ **${testResults.passed}** passed
            ❌ **${testResults.failed}** failed
            ⏭️ **${testResults.skipped}** skipped
            
            Coverage: **${testResults.coverage}%**
            
            [View full report](${testResults.reportUrl})`;
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: comment
            });
```