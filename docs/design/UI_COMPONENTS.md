# UI Components Design Specification

## Overview

FamilyTales UI components are designed with elder accessibility as the primary focus, incorporating warm, book-inspired aesthetics that make family story sharing feel natural and welcoming. Every component follows our core principles of generous spacing, high contrast, and intuitive interaction patterns.

## Elder-Friendly Design Principles

### Touch Target Guidelines
- **Minimum Size**: 48px × 48px for all interactive elements
- **Preferred Size**: 56px × 56px for primary actions
- **Spacing**: Minimum 8px between adjacent touch targets
- **Visual Feedback**: Clear pressed states with 200ms transition

### Accessibility Standards
- **Color Contrast**: WCAG AAA compliance (7:1 ratio minimum)
- **Text Size**: 16px minimum, 18px preferred for body text
- **Focus Indicators**: 2px outline in Deep Burgundy (#722F37)
- **Screen Reader**: Full semantic markup for all components

## Button Components

### Primary Button

#### Visual Specifications
```css
.primary-button {
  min-height: 56px;
  padding: 16px 24px;
  background-color: #8B4513; /* Warm Brown */
  color: #FDFCFA; /* Book White */
  border: none;
  border-radius: 8px;
  font-family: 'Source Sans Pro', sans-serif;
  font-size: 18px;
  font-weight: 600;
  box-shadow: 0 2px 4px rgba(139, 69, 19, 0.2);
}

.primary-button:hover {
  background-color: #A0522D;
  box-shadow: 0 4px 8px rgba(139, 69, 19, 0.3);
  transform: translateY(-1px);
}

.primary-button:active {
  transform: translateY(0);
  box-shadow: 0 1px 2px rgba(139, 69, 19, 0.3);
}
```

#### Flutter Implementation
```dart
class PrimaryButton extends StatelessWidget {
  final String text;
  final VoidCallback onPressed;
  final bool isLoading;
  
  const PrimaryButton({
    Key? key,
    required this.text,
    required this.onPressed,
    this.isLoading = false,
  }) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return Container(
      height: 56,
      width: double.infinity,
      child: ElevatedButton(
        onPressed: isLoading ? null : onPressed,
        style: ElevatedButton.styleFrom(
          backgroundColor: const Color(0xFF8B4513),
          foregroundColor: const Color(0xFFFDFCFA),
          elevation: 2,
          shadowColor: const Color(0xFF8B4513).withOpacity(0.2),
          shape: RoundedRectangleBorder(
            borderRadius: BorderRadius.circular(8),
          ),
        ),
        child: isLoading
            ? const CircularProgressIndicator(
                valueColor: AlwaysStoppedAnimation<Color>(Color(0xFFFDFCFA)),
              )
            : Text(
                text,
                style: const TextStyle(
                  fontSize: 18,
                  fontWeight: FontWeight.w600,
                ),
              ),
      ),
    );
  }
}
```

### Secondary Button

#### Visual Specifications
```css
.secondary-button {
  min-height: 56px;
  padding: 16px 24px;
  background-color: transparent;
  color: #8B4513; /* Warm Brown */
  border: 2px solid #8B4513;
  border-radius: 8px;
  font-family: 'Source Sans Pro', sans-serif;
  font-size: 18px;
  font-weight: 600;
}

.secondary-button:hover {
  background-color: #8B4513;
  color: #FDFCFA;
}
```

### Icon Button

#### Specifications
- **Size**: 48px × 48px minimum
- **Icon Size**: 24px within button
- **Padding**: 12px on all sides
- **Border Radius**: 24px (fully rounded)
- **Colors**: Warm Brown icon on Book White background

## Memory Book Card Components

### Story Card

#### Layout Structure
```
┌─────────────────────────────────────┐
│  [Thumbnail]     Story Title        │
│   120x80px       Author Name        │
│                  Duration: 12:45    │
│                  Status Indicator   │
└─────────────────────────────────────┘
```

#### Visual Specifications
```css
.story-card {
  background-color: #FDFCFA; /* Book White */
  border: 1px solid #E5E5E5;
  border-radius: 12px;
  padding: 16px;
  margin-bottom: 12px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  min-height: 120px;
}

.story-card:hover {
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  transform: translateY(-2px);
}
```

#### Flutter Implementation
```dart
class StoryCard extends StatelessWidget {
  final Story story;
  final VoidCallback onTap;
  
  const StoryCard({
    Key? key,
    required this.story,
    required this.onTap,
  }) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return Card(
      elevation: 2,
      color: const Color(0xFFFDFCFA),
      shape: RoundedRectangleBorder(
        borderRadius: BorderRadius.circular(12),
      ),
      child: InkWell(
        onTap: onTap,
        borderRadius: BorderRadius.circular(12),
        child: Padding(
          padding: const EdgeInsets.all(16),
          child: Row(
            children: [
              // Thumbnail
              Container(
                width: 120,
                height: 80,
                decoration: BoxDecoration(
                  borderRadius: BorderRadius.circular(8),
                  color: const Color(0xFFE5E5E5),
                ),
                child: story.thumbnailUrl != null
                    ? ClipRRect(
                        borderRadius: BorderRadius.circular(8),
                        child: Image.network(
                          story.thumbnailUrl!,
                          fit: BoxFit.cover,
                        ),
                      )
                    : const Icon(
                        Icons.auto_stories,
                        size: 32,
                        color: Color(0xFF8B4513),
                      ),
              ),
              const SizedBox(width: 16),
              // Content
              Expanded(
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Text(
                      story.title,
                      style: const TextStyle(
                        fontSize: 20,
                        fontWeight: FontWeight.w500,
                        color: Color(0xFF8B4513),
                      ),
                      maxLines: 2,
                      overflow: TextOverflow.ellipsis,
                    ),
                    const SizedBox(height: 8),
                    Text(
                      'By ${story.authorName}',
                      style: const TextStyle(
                        fontSize: 16,
                        color: Color(0xFF666666),
                      ),
                    ),
                    const SizedBox(height: 4),
                    Text(
                      'Duration: ${_formatDuration(story.duration)}',
                      style: const TextStyle(
                        fontSize: 14,
                        color: Color(0xFF888888),
                      ),
                    ),
                  ],
                ),
              ),
            ],
          ),
        ),
      ),
    );
  }
  
  String _formatDuration(Duration duration) {
    final minutes = duration.inMinutes;
    final seconds = duration.inSeconds % 60;
    return '$minutes:${seconds.toString().padLeft(2, '0')}';
  }
}
```

### Chapter Card

#### Specifications
- **Height**: 80px minimum
- **Layout**: Horizontal with chapter number, title, and duration
- **Interactive**: Clear pressed state for navigation
- **Progress**: Visual indicator for completed chapters

## Family Member Avatar Components

### Avatar Specifications

#### Size Variants
- **Large Avatar**: 120px diameter (profile pages)
- **Medium Avatar**: 80px diameter (story cards)
- **Small Avatar**: 48px diameter (navigation, lists)
- **Mini Avatar**: 32px diameter (comments, metadata)

#### Visual Design
```css
.avatar {
  border-radius: 50%;
  border: 3px solid #D4A574; /* Golden Amber */
  background-color: #F5F5DC; /* Cream */
  overflow: hidden;
}

.avatar.online::after {
  content: '';
  position: absolute;
  bottom: 4px;
  right: 4px;
  width: 12px;
  height: 12px;
  background-color: #4CAF50;
  border: 2px solid #FDFCFA;
  border-radius: 50%;
}
```

#### Flutter Implementation
```dart
class FamilyAvatar extends StatelessWidget {
  final String? imageUrl;
  final String initials;
  final double size;
  final bool showOnlineIndicator;
  
  const FamilyAvatar({
    Key? key,
    this.imageUrl,
    required this.initials,
    this.size = 80,
    this.showOnlineIndicator = false,
  }) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return Stack(
      children: [
        Container(
          width: size,
          height: size,
          decoration: BoxDecoration(
            shape: BoxShape.circle,
            border: Border.all(
              color: const Color(0xFFD4A574),
              width: 3,
            ),
          ),
          child: ClipOval(
            child: imageUrl != null
                ? Image.network(
                    imageUrl!,
                    fit: BoxFit.cover,
                    errorBuilder: (context, error, stackTrace) {
                      return _buildInitialsAvatar();
                    },
                  )
                : _buildInitialsAvatar(),
          ),
        ),
        if (showOnlineIndicator)
          Positioned(
            bottom: size * 0.05,
            right: size * 0.05,
            child: Container(
              width: size * 0.15,
              height: size * 0.15,
              decoration: BoxDecoration(
                color: Colors.green,
                shape: BoxShape.circle,
                border: Border.all(
                  color: const Color(0xFFFDFCFA),
                  width: 2,
                ),
              ),
            ),
          ),
      ],
    );
  }
  
  Widget _buildInitialsAvatar() {
    return Container(
      color: const Color(0xFFF5F5DC),
      child: Center(
        child: Text(
          initials,
          style: TextStyle(
            fontSize: size * 0.4,
            fontWeight: FontWeight.w600,
            color: const Color(0xFF8B4513),
          ),
        ),
      ),
    );
  }
}
```

## Form Input Components

### Text Input Field

#### Design Specifications
```css
.text-input {
  min-height: 56px;
  padding: 16px;
  background-color: #FDFCFA; /* Book White */
  border: 2px solid #E5E5E5;
  border-radius: 8px;
  font-family: 'Source Sans Pro', sans-serif;
  font-size: 18px;
  color: #8B4513;
}

.text-input:focus {
  border-color: #8B4513; /* Warm Brown */
  outline: none;
  box-shadow: 0 0 0 2px rgba(139, 69, 19, 0.2);
}

.text-input::placeholder {
  color: #999999;
  font-style: italic;
}
```

#### Flutter Implementation
```dart
class FamilyTextField extends StatelessWidget {
  final String label;
  final String? hint;
  final TextEditingController controller;
  final bool isRequired;
  final int maxLines;
  final TextInputType keyboardType;
  
  const FamilyTextField({
    Key? key,
    required this.label,
    this.hint,
    required this.controller,
    this.isRequired = false,
    this.maxLines = 1,
    this.keyboardType = TextInputType.text,
  }) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        RichText(
          text: TextSpan(
            style: const TextStyle(
              fontSize: 16,
              fontWeight: FontWeight.w600,
              color: Color(0xFF8B4513),
            ),
            children: [
              TextSpan(text: label),
              if (isRequired)
                const TextSpan(
                  text: ' *',
                  style: TextStyle(color: Color(0xFF722F37)),
                ),
            ],
          ),
        ),
        const SizedBox(height: 8),
        TextFormField(
          controller: controller,
          keyboardType: keyboardType,
          maxLines: maxLines,
          style: const TextStyle(
            fontSize: 18,
            color: Color(0xFF8B4513),
          ),
          decoration: InputDecoration(
            hintText: hint,
            hintStyle: const TextStyle(
              color: Color(0xFF999999),
              fontStyle: FontStyle.italic,
            ),
            filled: true,
            fillColor: const Color(0xFFFDFCFA),
            contentPadding: const EdgeInsets.all(16),
            border: OutlineInputBorder(
              borderRadius: BorderRadius.circular(8),
              borderSide: const BorderSide(
                color: Color(0xFFE5E5E5),
                width: 2,
              ),
            ),
            focusedBorder: OutlineInputBorder(
              borderRadius: BorderRadius.circular(8),
              borderSide: const BorderSide(
                color: Color(0xFF8B4513),
                width: 2,
              ),
            ),
          ),
        ),
      ],
    );
  }
}
```

### File Upload Component

#### Visual Design
- **Drop Zone**: Large, clearly defined area with dashed border
- **Instructions**: Clear, simple language about accepted formats
- **Progress**: Visual progress bar during upload
- **Preview**: Thumbnail preview of uploaded content

#### Specifications
```css
.file-upload {
  min-height: 200px;
  border: 3px dashed #D4A574; /* Golden Amber */
  border-radius: 12px;
  background-color: #FDFCFA;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 32px;
  cursor: pointer;
}

.file-upload:hover {
  border-color: #8B4513; /* Warm Brown */
  background-color: #F5F5DC; /* Cream */
}
```

## Navigation Components

### Bottom Navigation Bar

#### Design Specifications
- **Height**: 80px minimum for elder accessibility
- **Icons**: 32px with text labels below
- **Active State**: Golden Amber background with rounded corners
- **Layout**: Maximum 4 main navigation items

#### Flutter Implementation
```dart
class FamilyBottomNavBar extends StatelessWidget {
  final int currentIndex;
  final Function(int) onTap;
  
  const FamilyBottomNavBar({
    Key? key,
    required this.currentIndex,
    required this.onTap,
  }) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return Container(
      height: 80,
      decoration: const BoxDecoration(
        color: Color(0xFFFDFCFA),
        border: Border(
          top: BorderSide(
            color: Color(0xFFE5E5E5),
            width: 1,
          ),
        ),
      ),
      child: Row(
        mainAxisAlignment: MainAxisAlignment.spaceEvenly,
        children: [
          _buildNavItem(0, Icons.home, 'Home'),
          _buildNavItem(1, Icons.auto_stories, 'Stories'),
          _buildNavItem(2, Icons.family_restroom, 'Family'),
          _buildNavItem(3, Icons.person, 'Profile'),
        ],
      ),
    );
  }
  
  Widget _buildNavItem(int index, IconData icon, String label) {
    final isActive = currentIndex == index;
    
    return GestureDetector(
      onTap: () => onTap(index),
      child: Container(
        padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
        decoration: BoxDecoration(
          color: isActive ? const Color(0xFFD4A574) : Colors.transparent,
          borderRadius: BorderRadius.circular(12),
        ),
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            Icon(
              icon,
              size: 32,
              color: isActive ? const Color(0xFFFDFCFA) : const Color(0xFF8B4513),
            ),
            const SizedBox(height: 4),
            Text(
              label,
              style: TextStyle(
                fontSize: 12,
                fontWeight: FontWeight.w600,
                color: isActive ? const Color(0xFFFDFCFA) : const Color(0xFF8B4513),
              ),
            ),
          ],
        ),
      ),
    );
  }
}
```

### Breadcrumb Navigation

#### Specifications
- **Typography**: 16px Source Sans Pro
- **Separators**: Warm brown chevrons (›)
- **Current Page**: Bold, non-clickable
- **Links**: Underlined on hover

## Loading and Feedback Components

### Loading Spinner

#### Design
```dart
class FamilyLoadingSpinner extends StatelessWidget {
  final double size;
  final String? message;
  
  const FamilyLoadingSpinner({
    Key? key,
    this.size = 48,
    this.message,
  }) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return Column(
      mainAxisSize: MainAxisSize.min,
      children: [
        SizedBox(
          width: size,
          height: size,
          child: CircularProgressIndicator(
            strokeWidth: 4,
            valueColor: const AlwaysStoppedAnimation<Color>(Color(0xFF8B4513)),
          ),
        ),
        if (message != null) ...[
          const SizedBox(height: 16),
          Text(
            message!,
            style: const TextStyle(
              fontSize: 16,
              color: Color(0xFF8B4513),
            ),
            textAlign: TextAlign.center,
          ),
        ],
      ],
    );
  }
}
```

### Toast Notifications

#### Specifications
- **Position**: Top of screen with slide-down animation
- **Duration**: 4 seconds for info, 6 seconds for errors
- **Colors**: Success (green), Warning (amber), Error (burgundy)
- **Dismissible**: Swipe to dismiss functionality

## Audio Player Components

### Mini Player

#### Design Requirements
- **Height**: 80px bottom bar
- **Controls**: Play/pause, skip, title display
- **Progress**: Thin progress bar at top
- **Expandable**: Tap to open full player view

### Full Player

#### Layout Structure
```
┌─────────────────────────────────────┐
│            Story Title              │
│           Author Name               │
│                                     │
│         Waveform Display            │
│                                     │
│        Audio Controls Row           │
│     [<<] [Play/Pause] [>>]         │
│                                     │
│         Speed    Volume             │
│                                     │
│      Chapter Navigation             │
└─────────────────────────────────────┘
```

## Responsive Design Guidelines

### Breakpoints
- **Mobile**: < 768px
- **Tablet**: 768px - 1024px
- **Desktop**: > 1024px

### Adaptive Components
- **Button Sizes**: Larger on mobile, standard on desktop
- **Typography**: Scales appropriately across devices
- **Touch Targets**: Mobile-optimized sizes maintained on all devices
- **Layout**: Single column on mobile, multi-column on larger screens

## Component Testing Guidelines

### Accessibility Testing
- **Screen Reader**: Test with VoiceOver/TalkBack
- **Keyboard Navigation**: Full keyboard accessibility
- **Color Blind Testing**: Verify contrast and color combinations
- **Elder User Testing**: Validate with target demographic

### Performance Requirements
- **Render Time**: < 16ms for smooth 60fps animations
- **Memory Usage**: Efficient component cleanup
- **Touch Response**: < 100ms from touch to visual feedback
- **Image Loading**: Progressive loading with graceful fallbacks

This comprehensive component library ensures that FamilyTales provides a consistently warm, accessible, and delightful experience that honors the precious family memories being shared through our platform.