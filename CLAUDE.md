# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

FamilyTales is a family memory preservation platform that transforms handwritten documents into audio stories. The project uses a modern tech stack with Rust backend, Flutter frontend, and focuses on elder-friendly design.

## Repository Structure

This is primarily a documentation and planning repository containing:

- `docs/` - Comprehensive technical documentation
- `README.md` - Project overview, features, and business model
- `docs/MVP_REQUIREMENTS.md` - **Read First** - Clear MVP scope and boundaries
- `docs/SPRINTS.md` - Detailed, AI-agent-friendly sprint planning
- `docs/MVP_USER_FLOWS.md` - Complete user journey documentation for MVP features
- `docs/specs/USER_FLOWS.md` - Comprehensive user flows for all features
- `original_research/` - Market research and competitive analysis
- `product_brainstorm/` - Architecture, business case, and technical planning

The actual implementation will follow this structure:
```
backend/            # Rust API server (Axum + PostgreSQL)
mobile/            # Flutter mobile app
docker/            # Docker configurations
migrations/        # Database migrations
scripts/           # Development scripts
```

## Key Technologies

- **Backend**: Rust with Axum framework, PostgreSQL, Redis
- **Frontend**: Flutter 3.x (iOS/Android/Web/Desktop)
- **Media**: Mux for audio streaming and storage
- **Authentication**: Clerk
- **Deployment**: Simple Docker Compose (MVP), Kubernetes (future)

## Development Commands

Since this is currently a documentation repository, the main development activities involve:

### Documentation Site
```bash
# Serve documentation locally with docsify (recommended)
npx docsify serve docs

# Alternative: serve with Python
cd docs && python -m http.server 8000
```

### Current Development Commands
Since this is a documentation repository with a docsify setup:

```bash
# View documentation locally
npx docsify serve docs

# The documentation includes:
# - Complete MVP specifications in docs/MVP_REQUIREMENTS.md
# - Sprint planning in docs/SPRINTS.md with AI-agent-friendly tasks
# - User flows in docs/MVP_USER_FLOWS.md (MVP) and docs/specs/USER_FLOWS.md (comprehensive)
# - Architecture docs in docs/ARCHITECTURE.md and docs/specs/
```

### Future Development Commands (when implementation begins)

**Backend (Rust):**
```bash
# Setup
cargo build
sqlx migrate run

# Development
cargo watch -x run
cargo test
cargo clippy --all-features -- -D warnings

# Database
sqlx migrate add <migration_name>
sqlx migrate run
sqlx migrate revert
```

**Frontend (Flutter):**
```bash
# Setup
flutter pub get
flutter pub run build_runner build

# Development
flutter run
flutter test
flutter analyze

# Build
flutter build ios --release
flutter build apk --release
```

**Full Stack:**
```bash
# Start all services
docker compose up -d

# Run tests
make test  # Will run both Rust and Flutter tests
make lint  # Will run clippy and dart analyze
make fmt   # Will format all code
```

## Architecture Highlights

### Core Concepts
- **Families**: Groups of users who share memories
- **Memory Books**: Collections of related documents/stories  
- **Stories**: Individual documents converted to audio
- **Processing Pipeline**: OCR → Text Editing → Audio Generation → Distribution

### Key Services
- **OCR Processing**: Google Vision API (MVP) → Custom olmOCR (later)
- **Audio Generation**: Google Cloud TTS with multiple voice options
- **Media Streaming**: Mux handles all audio storage and HLS delivery
- **Family Management**: Role-based permissions (Owner/Admin/Member/Viewer)

### Database Schema
- **PostgreSQL** with extensions (uuid-ossp, pgvector for future ML)
- **Redis** for caching, sessions, and real-time features
- **Structured around families** with proper foreign key relationships

## Testing Strategy

### Test Categories
- **Unit Tests**: OCR accuracy, audio generation, business logic
- **Integration Tests**: API endpoints, database operations, family workflows
- **Widget Tests**: Flutter UI components and interactions
- **E2E Tests**: Complete user journeys (story creation, playback)
- **Performance Tests**: OCR processing speed, audio generation time
- **Elder UAT**: Accessibility and usability testing with target demographic

### Performance Requirements (MVP)
- OCR: < 10 seconds per page (MVP target)
- Audio generation: < 30 seconds per story (MVP target)
- App startup: < 5 seconds (more lenient for MVP)
- API responses: < 500ms p95 (MVP target)

## Code Quality Standards

### Rust
- Use `clippy` with pedantic warnings
- Structured logging with `tracing`
- Error handling with `Result` types
- Database queries with `sqlx` compile-time checking

### Flutter/Dart
- Follow official Flutter lints
- Riverpod for state management
- Structured logging for debugging
- Accessibility compliance (WCAG 2.1 AA)

### Common Patterns
- **Error Handling**: Always use proper error types, never unwrap in production
- **Logging**: Structured logs with correlation IDs and contextual fields
- **Testing**: Comprehensive test coverage with focus on business logic
- **Security**: Never log sensitive data, use environment variables for secrets

## Deployment

The system is designed for Kubernetes deployment with:
- **Horizontal scaling** for API servers and workers
- **Multi-region** support for global families
- **Monitoring** with Prometheus, Grafana, and ELK stack
- **Security** with TLS, JWT authentication, and data encryption

## Business Context

This is a **freemium B2C SaaS** targeting:
- **Primary**: Adults 65+ preserving family memories
- **Secondary**: Adult children (35-55) buying for parents
- **Pricing**: 14-day free trial, then $14.99/month family plan (MVP has single tier)

Key success metrics:
- 5% conversion rate (free to paid)
- 60% monthly active users
- $240 customer lifetime value
- 70+ Net Promoter Score

## Documentation Structure

The documentation is organized in a docsify site under `docs/` with:
- **Getting Started**: MVP requirements, sprint planning, user flows, architecture
- **Feature Specifications**: Detailed technical specs in `docs/specs/`
- **Operations**: Deployment, monitoring, security in `docs/ops/` 
- **Design System**: Brand guide, UI components, audio UX in `docs/design/`

Key files for AI agents:
- `docs/MVP_REQUIREMENTS.md` - **Read first** for scope boundaries
- `docs/SPRINTS.md` - Granular development tasks with file references
- `docs/specs/DATABASE_SCHEMA.md` - Complete data model
- `docs/specs/FRONTEND_ARCHITECTURE.md` - Flutter app structure

## Important Notes

- **Elder-first design**: Large touch targets, high contrast, simple navigation
- **Family-centric**: One subscription covers entire extended family
- **Privacy focused**: Self-hosted processing, GDPR compliance
- **Emotional product**: Focus on preserving irreplaceable memories
- **Performance critical**: Fast processing builds trust with older users
- **Documentation-first**: This repository contains comprehensive planning before implementation

When working with this codebase, always reference the MVP boundaries in `docs/MVP_REQUIREMENTS.md` and follow the sprint tasks in `docs/SPRINTS.md` which include specific file paths and implementation guidance.