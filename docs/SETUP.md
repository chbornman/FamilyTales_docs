# FamilyTales Development Setup Guide

This guide will help you set up your local development environment for FamilyTales.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Environment Setup](#environment-setup)
- [Docker Compose Configuration](#docker-compose-configuration)
- [Environment Variables](#environment-variables)
- [Clerk Authentication Setup](#clerk-authentication-setup)
- [Mux Account Setup](#mux-account-setup)
- [Database Migrations](#database-migrations)
- [Running the Application](#running-the-application)
- [Troubleshooting](#troubleshooting)

## Prerequisites

### Required Software

1. **Rust** (1.75 or later)
   ```bash
   # Install Rust via rustup
   curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
   
   # Verify installation
   rustc --version
   cargo --version
   ```

2. **Flutter** (3.16 or later)
   ```bash
   # Download Flutter SDK from https://flutter.dev
   # Add to PATH
   export PATH="$PATH:/path/to/flutter/bin"
   
   # Verify installation
   flutter doctor
   ```

3. **PostgreSQL** (15 or later)
   ```bash
   # macOS with Homebrew
   brew install postgresql@15
   brew services start postgresql@15
   
   # Ubuntu/Debian
   sudo apt install postgresql-15 postgresql-contrib
   
   # Verify installation
   psql --version
   ```

4. **Redis** (7.0 or later)
   ```bash
   # macOS with Homebrew
   brew install redis
   brew services start redis
   
   # Ubuntu/Debian
   sudo apt install redis-server
   
   # Verify installation
   redis-cli ping
   ```

5. **Docker & Docker Compose**
   ```bash
   # Install Docker Desktop from https://www.docker.com/products/docker-desktop
   
   # Verify installation
   docker --version
   docker compose version
   ```

6. **Additional Tools**
   ```bash
   # Install sqlx-cli for database migrations
   cargo install sqlx-cli --features postgres
   
   # Install cargo-watch for development
   cargo install cargo-watch
   
   # Install Flutter dependencies
   flutter pub global activate fvm  # Flutter Version Management
   ```

### System Dependencies

```bash
# macOS
brew install pkg-config openssl tesseract

# Ubuntu/Debian
sudo apt-get install pkg-config libssl-dev tesseract-ocr libtesseract-dev
```

## Environment Setup

### Clone the Repository

```bash
git clone https://github.com/your-org/familytales.git
cd familytales
```

### Project Structure

```
familytales/
├── backend/            # Rust API server
│   ├── api/           # HTTP endpoints
│   ├── core/          # Business logic
│   ├── db/            # Database layer
│   └── services/      # External services
├── mobile/            # Flutter mobile app
│   ├── lib/           # Application code
│   ├── test/          # Tests
│   └── assets/        # Static assets
├── docker/            # Docker configurations
├── migrations/        # Database migrations
└── scripts/           # Development scripts
```

## Docker Compose Configuration

### Development Environment

```yaml
# docker-compose.yml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: familytales
      POSTGRES_PASSWORD: familytales_dev
      POSTGRES_DB: familytales_dev
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./scripts/init-db.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U familytales"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  minio:
    image: minio/minio:latest
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - minio_data:/data
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    command: server /data --console-address ":9001"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3

  mailhog:
    image: mailhog/mailhog
    ports:
      - "1025:1025"  # SMTP server
      - "8025:8025"  # Web UI

volumes:
  postgres_data:
  redis_data:
  minio_data:
```

### Start Services

```bash
# Start all services
docker compose up -d

# Check service status
docker compose ps

# View logs
docker compose logs -f

# Stop services
docker compose down
```

## Environment Variables

### Backend Configuration

Create `.env` file in the `backend` directory:

```bash
# Database
DATABASE_URL=postgresql://familytales:familytales_dev@localhost:5432/familytales_dev
DATABASE_MAX_CONNECTIONS=10
DATABASE_MIN_CONNECTIONS=2

# Redis
REDIS_URL=redis://localhost:6379
REDIS_MAX_CONNECTIONS=10

# Server
HOST=0.0.0.0
PORT=8080
ENV=development
RUST_LOG=info,familytales=debug

# Authentication (Clerk)
CLERK_SECRET_KEY=sk_test_your_secret_key_here
CLERK_PUBLISHABLE_KEY=pk_test_your_publishable_key_here
CLERK_JWT_VERIFICATION_KEY=your_jwt_key_here

# Storage (MinIO for local, S3 for production)
STORAGE_TYPE=minio
STORAGE_ENDPOINT=http://localhost:9000
STORAGE_ACCESS_KEY=minioadmin
STORAGE_SECRET_KEY=minioadmin
STORAGE_BUCKET=familytales
STORAGE_REGION=us-east-1

# Media Processing (Mux)
MUX_TOKEN_ID=your_mux_token_id
MUX_TOKEN_SECRET=your_mux_token_secret
MUX_WEBHOOK_SECRET=your_webhook_secret

# Email (Development)
SMTP_HOST=localhost
SMTP_PORT=1025
SMTP_FROM=noreply@familytales.local

# Error Tracking
SENTRY_DSN=https://your-sentry-dsn@sentry.io/project-id

# Feature Flags
ENABLE_PREMIUM_FEATURES=false
ENABLE_BETA_FEATURES=true
```

### Flutter Configuration

Create `.env` file in the `mobile` directory:

```bash
# API Configuration
API_BASE_URL=http://localhost:8080
API_TIMEOUT=30000

# Authentication (Clerk)
CLERK_PUBLISHABLE_KEY=pk_test_your_publishable_key_here

# Feature Flags
ENABLE_PREMIUM_FEATURES=false
ENABLE_BETA_FEATURES=true

# Analytics
ANALYTICS_ENABLED=false

# Sentry
SENTRY_DSN=https://your-sentry-dsn@sentry.io/project-id
```

Create `lib/config/env.dart`:

```dart
class Environment {
  static const String apiBaseUrl = String.fromEnvironment(
    'API_BASE_URL',
    defaultValue: 'http://localhost:8080',
  );
  
  static const String clerkPublishableKey = String.fromEnvironment(
    'CLERK_PUBLISHABLE_KEY',
    defaultValue: '',
  );
  
  static const bool enablePremiumFeatures = bool.fromEnvironment(
    'ENABLE_PREMIUM_FEATURES',
    defaultValue: false,
  );
}
```

## Clerk Authentication Setup

### 1. Create Clerk Account

1. Sign up at [clerk.com](https://clerk.com)
2. Create a new application
3. Choose "Mobile" as your platform

### 2. Configure Clerk

```typescript
// Clerk Dashboard Settings
{
  "application_name": "FamilyTales",
  "sign_in_options": ["email", "google", "apple"],
  "user_attributes": {
    "first_name": "required",
    "last_name": "required",
    "profile_image_url": "optional"
  },
  "jwt_template": {
    "claims": {
      "email": "{{user.primary_email_address}}",
      "user_id": "{{user.id}}",
      "metadata": "{{user.public_metadata}}"
    }
  }
}
```

### 3. Configure Webhooks

In Clerk Dashboard:
1. Go to Webhooks
2. Add endpoint: `https://your-api.com/webhooks/clerk`
3. Select events:
   - `user.created`
   - `user.updated`
   - `user.deleted`

### 4. Flutter Integration

```dart
// pubspec.yaml
dependencies:
  clerk_flutter: ^0.1.0

// lib/services/auth_service.dart
import 'package:clerk_flutter/clerk_flutter.dart';

class AuthService {
  late final Clerk _clerk;
  
  Future<void> initialize() async {
    _clerk = Clerk(
      publishableKey: Environment.clerkPublishableKey,
    );
    await _clerk.load();
  }
  
  Future<void> signIn({required String email}) async {
    await _clerk.signIn.create(
      strategy: SignInStrategy.emailCode,
      identifier: email,
    );
  }
}
```

## Mux Account Setup

### 1. Create Mux Account

1. Sign up at [mux.com](https://mux.com)
2. Create a new environment (Development)

### 2. Get API Credentials

1. Go to Settings → Access Tokens
2. Create a new token pair
3. Copy Token ID and Token Secret

### 3. Configure Webhooks

1. Go to Settings → Webhooks
2. Add endpoint: `https://your-api.com/webhooks/mux`
3. Select events:
   - `video.asset.created`
   - `video.asset.ready`
   - `video.asset.errored`

### 4. Backend Integration

```rust
// backend/src/services/mux.rs
use mux_rust_sdk::{MuxClient, CreateAssetRequest};

pub struct MuxService {
    client: MuxClient,
}

impl MuxService {
    pub fn new(token_id: &str, token_secret: &str) -> Self {
        let client = MuxClient::new(token_id, token_secret);
        Self { client }
    }
    
    pub async fn create_asset(&self, input_url: &str) -> Result<Asset> {
        let request = CreateAssetRequest {
            input: vec![AssetInput {
                url: input_url.to_string(),
            }],
            playback_policy: vec![PlaybackPolicy::Public],
            mp4_support: Some(Mp4Support::Standard),
        };
        
        self.client.create_asset(request).await
    }
}
```

## Database Migrations

### Initial Setup

```bash
# Create migration
sqlx migrate add initial_schema

# Edit migration file
vim migrations/20240101000000_initial_schema.sql
```

### Migration Files

```sql
-- migrations/20240101000000_initial_schema.sql
-- Enable extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- Create families table
CREATE TABLE families (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Create users table
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    clerk_id VARCHAR(255) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    first_name VARCHAR(255),
    last_name VARCHAR(255),
    profile_image_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Create family_members table
CREATE TABLE family_members (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    family_id UUID NOT NULL REFERENCES families(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role VARCHAR(50) NOT NULL CHECK (role IN ('admin', 'member', 'viewer')),
    joined_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(family_id, user_id)
);

-- Create stories table
CREATE TABLE stories (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    family_id UUID NOT NULL REFERENCES families(id) ON DELETE CASCADE,
    created_by UUID NOT NULL REFERENCES users(id),
    title VARCHAR(255) NOT NULL,
    content TEXT,
    narrator_voice VARCHAR(50),
    audio_url TEXT,
    mux_asset_id VARCHAR(255),
    mux_playback_id VARCHAR(255),
    duration INTEGER,
    status VARCHAR(50) NOT NULL DEFAULT 'draft' CHECK (status IN ('draft', 'processing', 'published', 'failed')),
    published_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Create story_pages table
CREATE TABLE story_pages (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    story_id UUID NOT NULL REFERENCES stories(id) ON DELETE CASCADE,
    page_number INTEGER NOT NULL,
    image_url TEXT NOT NULL,
    ocr_text TEXT,
    edited_text TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(story_id, page_number)
);

-- Create indexes
CREATE INDEX idx_family_members_family_id ON family_members(family_id);
CREATE INDEX idx_family_members_user_id ON family_members(user_id);
CREATE INDEX idx_stories_family_id ON stories(family_id);
CREATE INDEX idx_stories_created_by ON stories(created_by);
CREATE INDEX idx_stories_status ON stories(status);
CREATE INDEX idx_story_pages_story_id ON story_pages(story_id);

-- Create updated_at trigger function
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ language 'plpgsql';

-- Apply updated_at triggers
CREATE TRIGGER update_families_updated_at BEFORE UPDATE ON families
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_users_updated_at BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_stories_updated_at BEFORE UPDATE ON stories
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

### Run Migrations

```bash
# Check migration status
sqlx migrate info

# Run pending migrations
sqlx migrate run

# Revert last migration
sqlx migrate revert

# Create database if it doesn't exist
sqlx database create

# Drop and recreate database (WARNING: destructive)
sqlx database drop
sqlx database create
sqlx migrate run
```

## Running the Application

### Backend Development

```bash
# Navigate to backend directory
cd backend

# Install dependencies
cargo build

# Run migrations
sqlx migrate run

# Start development server with hot reload
cargo watch -x run

# Or run directly
cargo run

# Run tests
cargo test

# Run with specific log level
RUST_LOG=debug cargo run
```

### Flutter Development

```bash
# Navigate to mobile directory
cd mobile

# Install dependencies
flutter pub get

# Generate code (if using code generation)
flutter pub run build_runner build

# Run on iOS simulator
flutter run -d ios

# Run on Android emulator
flutter run -d android

# Run with specific environment
flutter run --dart-define=API_BASE_URL=http://localhost:8080

# Run tests
flutter test

# Build for release
flutter build ios --release
flutter build apk --release
```

### Full Stack Development

```bash
# Start all backend services
docker compose up -d

# In terminal 1: Run backend
cd backend && cargo watch -x run

# In terminal 2: Run Flutter app
cd mobile && flutter run

# In terminal 3: Watch logs
docker compose logs -f
```

## Troubleshooting

### Common Issues

1. **Database Connection Errors**
   ```bash
   # Check PostgreSQL is running
   docker compose ps postgres
   
   # Check connection
   psql $DATABASE_URL -c "SELECT 1"
   
   # Reset database
   sqlx database drop
   sqlx database create
   sqlx migrate run
   ```

2. **Redis Connection Errors**
   ```bash
   # Check Redis is running
   docker compose ps redis
   
   # Test connection
   redis-cli ping
   ```

3. **Port Conflicts**
   ```bash
   # Find process using port
   lsof -i :8080
   
   # Kill process
   kill -9 <PID>
   ```

4. **Flutter Issues**
   ```bash
   # Clean Flutter build
   flutter clean
   flutter pub get
   
   # Reset iOS pods
   cd ios && pod deintegrate && pod install
   
   # Clear build cache
   flutter pub cache clean
   ```

5. **Rust Compilation Errors**
   ```bash
   # Clean build
   cargo clean
   
   # Update dependencies
   cargo update
   
   # Check for outdated dependencies
   cargo outdated
   ```

### Development Tips

1. **Use Development Scripts**
   ```bash
   # Create useful scripts in scripts/
   ./scripts/setup.sh      # Initial setup
   ./scripts/reset-db.sh   # Reset database
   ./scripts/seed-db.sh    # Seed test data
   ```

2. **Enable Debug Logging**
   ```bash
   # Rust
   RUST_LOG=debug,sqlx=debug cargo run
   
   # Flutter
   flutter logs
   ```

3. **Use Docker Compose Profiles**
   ```yaml
   # docker-compose.yml
   services:
     monitoring:
       profiles: ["monitoring"]
       # ... monitoring services
   ```

4. **Hot Reload**
   - Backend: Use `cargo-watch`
   - Flutter: Press 'r' in terminal

5. **Database GUI**
   - Use TablePlus, DBeaver, or pgAdmin
   - Connection string from `.env`

### Getting Help

1. Check logs first
2. Search existing issues
3. Include full error messages
4. Provide environment details
5. Steps to reproduce