# Database Schema

## Overview

FamilyTales uses PostgreSQL 16 with TimescaleDB for time-series data optimization. The schema is designed to support multi-tenant family memory preservation with strong data integrity and efficient query patterns.

## Database Configuration

```yaml
Database: PostgreSQL 16
Extensions:
  - TimescaleDB (for time-series optimization)
  - pgvector (for future ML features)
  - uuid-ossp (for UUID generation)
Encoding: UTF8
Collation: en_US.UTF-8
Connection Pool: 25 connections
```

## Tables

### 1. users

Stores user information with Clerk integration.

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    clerk_id VARCHAR(255) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    display_name VARCHAR(255),
    avatar_url TEXT,
    subscription_tier VARCHAR(50) DEFAULT 'free' CHECK (subscription_tier IN ('free', 'premium', 'enterprise')),
    subscription_expires_at TIMESTAMPTZ,
    storage_used_bytes BIGINT DEFAULT 0,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    last_login_at TIMESTAMPTZ,
    metadata JSONB DEFAULT '{}'::jsonb
);

CREATE INDEX idx_users_clerk_id ON users(clerk_id);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_subscription ON users(subscription_tier, subscription_expires_at);
CREATE INDEX idx_users_created_at ON users(created_at);
```

### 2. families

Represents family groups that share memory books.

```sql
CREATE TABLE families (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
    subscription_tier VARCHAR(50) DEFAULT 'free' CHECK (subscription_tier IN ('free', 'premium', 'enterprise')),
    storage_used_bytes BIGINT DEFAULT 0,
    storage_limit_bytes BIGINT DEFAULT 5368709120, -- 5GB default
    member_limit INTEGER DEFAULT 10,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    settings JSONB DEFAULT '{
        "privacy": "private",
        "allow_downloads": true,
        "require_approval": true
    }'::jsonb
);

CREATE INDEX idx_families_owner ON families(owner_id);
CREATE INDEX idx_families_active ON families(is_active);
CREATE INDEX idx_families_created_at ON families(created_at);
```

### 3. family_members

Junction table for users belonging to families with role-based access.

```sql
CREATE TABLE family_members (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    family_id UUID NOT NULL REFERENCES families(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role VARCHAR(50) NOT NULL DEFAULT 'member' CHECK (role IN ('owner', 'admin', 'editor', 'member', 'viewer')),
    joined_at TIMESTAMPTZ DEFAULT NOW(),
    invited_by UUID REFERENCES users(id),
    is_active BOOLEAN DEFAULT true,
    permissions JSONB DEFAULT '{
        "can_edit": false,
        "can_delete": false,
        "can_invite": false,
        "can_download": true
    }'::jsonb,
    UNIQUE(family_id, user_id)
);

CREATE INDEX idx_family_members_family ON family_members(family_id);
CREATE INDEX idx_family_members_user ON family_members(user_id);
CREATE INDEX idx_family_members_role ON family_members(role);
```

### 4. memory_books

Containers for family memories and stories.

```sql
CREATE TABLE memory_books (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    family_id UUID NOT NULL REFERENCES families(id) ON DELETE CASCADE,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    cover_image_url TEXT,
    created_by UUID NOT NULL REFERENCES users(id),
    is_collaborative BOOLEAN DEFAULT true,
    privacy VARCHAR(50) DEFAULT 'family' CHECK (privacy IN ('family', 'private', 'public')),
    thread_count INTEGER DEFAULT 0,
    last_activity_at TIMESTAMPTZ DEFAULT NOW(),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    metadata JSONB DEFAULT '{
        "theme": "default",
        "tags": [],
        "contributors": []
    }'::jsonb
);

CREATE INDEX idx_memory_books_family ON memory_books(family_id);
CREATE INDEX idx_memory_books_creator ON memory_books(created_by);
CREATE INDEX idx_memory_books_activity ON memory_books(last_activity_at DESC);
CREATE INDEX idx_memory_books_privacy ON memory_books(privacy);
```

### 5. threads

Conversation threads within memory books.

```sql
CREATE TABLE threads (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    memory_book_id UUID NOT NULL REFERENCES memory_books(id) ON DELETE CASCADE,
    title VARCHAR(255),
    started_by UUID NOT NULL REFERENCES users(id),
    participant_count INTEGER DEFAULT 1,
    segment_count INTEGER DEFAULT 0,
    last_segment_at TIMESTAMPTZ,
    is_locked BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    metadata JSONB DEFAULT '{
        "topic": null,
        "mood": null,
        "participants": []
    }'::jsonb
);

CREATE INDEX idx_threads_memory_book ON threads(memory_book_id);
CREATE INDEX idx_threads_starter ON threads(started_by);
CREATE INDEX idx_threads_activity ON threads(last_segment_at DESC);
CREATE INDEX idx_threads_created_at ON threads(created_at);
```

### 6. content_segments

Individual pieces of content (text, audio, photos) within threads.
This table is partitioned by created_at for performance.

```sql
CREATE TABLE content_segments (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    thread_id UUID NOT NULL REFERENCES threads(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id),
    content_type VARCHAR(50) NOT NULL CHECK (content_type IN ('text', 'audio', 'photo', 'mixed')),
    sequence_number INTEGER NOT NULL,
    
    -- Content fields
    text_content TEXT,
    audio_url TEXT,
    audio_duration_seconds INTEGER,
    audio_transcript TEXT,
    photo_urls TEXT[], -- Array of photo URLs
    
    -- Metadata
    is_edited BOOLEAN DEFAULT false,
    edited_at TIMESTAMPTZ,
    is_deleted BOOLEAN DEFAULT false,
    deleted_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    
    -- Analytics
    view_count INTEGER DEFAULT 0,
    reaction_count INTEGER DEFAULT 0,
    
    metadata JSONB DEFAULT '{
        "device_type": null,
        "location": null,
        "weather": null,
        "ai_insights": {}
    }'::jsonb,
    
    UNIQUE(thread_id, sequence_number)
) PARTITION BY RANGE (created_at);

-- Create partitions for each month
CREATE TABLE content_segments_2024_01 PARTITION OF content_segments
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
-- Continue creating monthly partitions...

-- Indexes on parent table (inherited by partitions)
CREATE INDEX idx_content_segments_thread ON content_segments(thread_id);
CREATE INDEX idx_content_segments_user ON content_segments(user_id);
CREATE INDEX idx_content_segments_type ON content_segments(content_type);
CREATE INDEX idx_content_segments_created ON content_segments(created_at);
```

### 7. photo_narrations

AI-generated or user-provided narrations for photos.

```sql
CREATE TABLE photo_narrations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    content_segment_id UUID NOT NULL REFERENCES content_segments(id) ON DELETE CASCADE,
    photo_index INTEGER NOT NULL, -- Which photo in the array
    narration_text TEXT NOT NULL,
    narration_type VARCHAR(50) DEFAULT 'ai' CHECK (narration_type IN ('ai', 'user', 'edited')),
    confidence_score FLOAT,
    language VARCHAR(10) DEFAULT 'en',
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    metadata JSONB DEFAULT '{
        "model_version": null,
        "detected_objects": [],
        "detected_people": []
    }'::jsonb,
    UNIQUE(content_segment_id, photo_index)
);

CREATE INDEX idx_photo_narrations_segment ON photo_narrations(content_segment_id);
CREATE INDEX idx_photo_narrations_type ON photo_narrations(narration_type);
```

### 8. content_links

Relationships between content segments (replies, references).

```sql
CREATE TABLE content_links (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    source_segment_id UUID NOT NULL REFERENCES content_segments(id) ON DELETE CASCADE,
    target_segment_id UUID NOT NULL REFERENCES content_segments(id) ON DELETE CASCADE,
    link_type VARCHAR(50) NOT NULL CHECK (link_type IN ('reply', 'reference', 'continuation')),
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    metadata JSONB DEFAULT '{}'::jsonb,
    UNIQUE(source_segment_id, target_segment_id, link_type)
);

CREATE INDEX idx_content_links_source ON content_links(source_segment_id);
CREATE INDEX idx_content_links_target ON content_links(target_segment_id);
CREATE INDEX idx_content_links_type ON content_links(link_type);
```

### 9. content_tags

User-generated tags for content discovery.

```sql
CREATE TABLE content_tags (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    content_segment_id UUID NOT NULL REFERENCES content_segments(id) ON DELETE CASCADE,
    tag_name VARCHAR(100) NOT NULL,
    tagged_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(content_segment_id, tag_name, tagged_by)
);

CREATE INDEX idx_content_tags_segment ON content_tags(content_segment_id);
CREATE INDEX idx_content_tags_name ON content_tags(tag_name);
CREATE INDEX idx_content_tags_user ON content_tags(tagged_by);
```

### 10. family_invites

Pending invitations to join families.

```sql
CREATE TABLE family_invites (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    family_id UUID NOT NULL REFERENCES families(id) ON DELETE CASCADE,
    email VARCHAR(255) NOT NULL,
    role VARCHAR(50) DEFAULT 'member' CHECK (role IN ('admin', 'editor', 'member', 'viewer')),
    invited_by UUID NOT NULL REFERENCES users(id),
    invite_code VARCHAR(255) UNIQUE NOT NULL,
    expires_at TIMESTAMPTZ NOT NULL,
    accepted_at TIMESTAMPTZ,
    accepted_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    metadata JSONB DEFAULT '{
        "message": null,
        "permissions": {}
    }'::jsonb
);

CREATE INDEX idx_family_invites_family ON family_invites(family_id);
CREATE INDEX idx_family_invites_email ON family_invites(email);
CREATE INDEX idx_family_invites_code ON family_invites(invite_code);
CREATE INDEX idx_family_invites_expires ON family_invites(expires_at);
```

## Views

### active_family_members

Convenient view for active family memberships.

```sql
CREATE VIEW active_family_members AS
SELECT 
    fm.*,
    u.display_name,
    u.email,
    u.avatar_url,
    f.name as family_name
FROM family_members fm
JOIN users u ON fm.user_id = u.id
JOIN families f ON fm.family_id = f.id
WHERE fm.is_active = true 
  AND u.is_active = true 
  AND f.is_active = true;
```

### memory_book_stats

Aggregated statistics for memory books.

```sql
CREATE VIEW memory_book_stats AS
SELECT 
    mb.id,
    mb.family_id,
    mb.title,
    COUNT(DISTINCT t.id) as thread_count,
    COUNT(DISTINCT cs.id) as segment_count,
    COUNT(DISTINCT cs.user_id) as contributor_count,
    MAX(cs.created_at) as last_activity,
    SUM(CASE WHEN cs.content_type = 'photo' THEN array_length(cs.photo_urls, 1) ELSE 0 END) as photo_count,
    SUM(CASE WHEN cs.content_type = 'audio' THEN 1 ELSE 0 END) as audio_count
FROM memory_books mb
LEFT JOIN threads t ON mb.id = t.memory_book_id
LEFT JOIN content_segments cs ON t.id = cs.thread_id
GROUP BY mb.id, mb.family_id, mb.title;
```

## Migration Strategy

### Using sqlx

```rust
// migrations/001_initial_schema.sql
-- Create extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "timescaledb";

-- Create tables in dependency order
-- 1. Independent tables first
CREATE TABLE users ...
CREATE TABLE families ...

-- 2. Tables with foreign keys
CREATE TABLE family_members ...
CREATE TABLE memory_books ...

-- migrations/002_create_partitions.sql
-- Create initial partitions for content_segments
DO $$
DECLARE
    start_date date := '2024-01-01';
    end_date date := '2025-01-01';
    partition_date date;
BEGIN
    partition_date := start_date;
    WHILE partition_date < end_date LOOP
        EXECUTE format('CREATE TABLE IF NOT EXISTS content_segments_%s PARTITION OF content_segments FOR VALUES FROM (%L) TO (%L)',
            to_char(partition_date, 'YYYY_MM'),
            partition_date,
            partition_date + interval '1 month'
        );
        partition_date := partition_date + interval '1 month';
    END LOOP;
END $$;

-- migrations/003_create_indexes.sql
-- Create all indexes after data is loaded
```

### Migration Commands

```bash
# Run migrations
sqlx migrate run

# Create new migration
sqlx migrate add <migration_name>

# Revert last migration
sqlx migrate revert
```

## Data Retention Policies

### 7-Year Retention Policy

```sql
-- Function to archive old data
CREATE OR REPLACE FUNCTION archive_old_content() RETURNS void AS $$
DECLARE
    archive_date TIMESTAMPTZ := NOW() - INTERVAL '7 years';
BEGIN
    -- Move to archive tables
    INSERT INTO archived_content_segments 
    SELECT * FROM content_segments 
    WHERE created_at < archive_date;
    
    -- Delete from main table
    DELETE FROM content_segments 
    WHERE created_at < archive_date;
    
    -- Update family storage metrics
    UPDATE families f
    SET storage_used_bytes = (
        SELECT COALESCE(SUM(
            LENGTH(cs.text_content) + 
            LENGTH(cs.audio_transcript) +
            LENGTH(cs.audio_url::text) * 1000000 + -- Estimate 1MB per audio
            ARRAY_LENGTH(cs.photo_urls, 1) * 500000  -- Estimate 500KB per photo
        ), 0)
        FROM content_segments cs
        JOIN threads t ON cs.thread_id = t.id
        JOIN memory_books mb ON t.memory_book_id = mb.id
        WHERE mb.family_id = f.id
    );
END;
$$ LANGUAGE plpgsql;

-- Schedule monthly execution
SELECT cron.schedule('archive-old-content', '0 0 1 * *', 'SELECT archive_old_content()');
```

### Partition Maintenance

```sql
-- Function to create future partitions
CREATE OR REPLACE FUNCTION create_monthly_partitions() RETURNS void AS $$
DECLARE
    start_date date;
    end_date date;
    partition_name text;
BEGIN
    start_date := date_trunc('month', NOW() + interval '1 month');
    end_date := start_date + interval '1 month';
    partition_name := 'content_segments_' || to_char(start_date, 'YYYY_MM');
    
    EXECUTE format('CREATE TABLE IF NOT EXISTS %I PARTITION OF content_segments FOR VALUES FROM (%L) TO (%L)',
        partition_name, start_date, end_date);
END;
$$ LANGUAGE plpgsql;

-- Schedule monthly execution
SELECT cron.schedule('create-partitions', '0 0 25 * *', 'SELECT create_monthly_partitions()');
```

## Common Query Examples

### Get Family Memory Books with Stats

```sql
-- Get all memory books for a family with statistics
SELECT 
    mb.*,
    mbs.thread_count,
    mbs.segment_count,
    mbs.contributor_count,
    mbs.last_activity,
    mbs.photo_count,
    mbs.audio_count,
    u.display_name as created_by_name
FROM memory_books mb
JOIN memory_book_stats mbs ON mb.id = mbs.id
JOIN users u ON mb.created_by = u.id
WHERE mb.family_id = $1
  AND mb.privacy != 'private'
ORDER BY mbs.last_activity DESC;
```

### Get Thread with Content

```sql
-- Get thread with all content segments
WITH thread_content AS (
    SELECT 
        cs.*,
        u.display_name,
        u.avatar_url,
        array_agg(
            json_build_object(
                'index', pn.photo_index,
                'narration', pn.narration_text,
                'type', pn.narration_type
            ) ORDER BY pn.photo_index
        ) FILTER (WHERE pn.id IS NOT NULL) as photo_narrations
    FROM content_segments cs
    JOIN users u ON cs.user_id = u.id
    LEFT JOIN photo_narrations pn ON cs.id = pn.content_segment_id
    WHERE cs.thread_id = $1
      AND cs.is_deleted = false
    GROUP BY cs.id, u.display_name, u.avatar_url
)
SELECT 
    t.*,
    json_agg(
        tc ORDER BY tc.sequence_number
    ) as segments
FROM threads t
JOIN thread_content tc ON true
WHERE t.id = $1
GROUP BY t.id;
```

### Search Content by Tags

```sql
-- Search content segments by tags
SELECT DISTINCT
    cs.*,
    mb.title as memory_book_title,
    t.title as thread_title,
    array_agg(DISTINCT ct.tag_name) as tags
FROM content_segments cs
JOIN content_tags ct ON cs.id = ct.content_segment_id
JOIN threads t ON cs.thread_id = t.id
JOIN memory_books mb ON t.memory_book_id = mb.id
JOIN family_members fm ON mb.family_id = fm.family_id
WHERE fm.user_id = $1
  AND fm.is_active = true
  AND ct.tag_name = ANY($2::text[])
  AND cs.is_deleted = false
GROUP BY cs.id, mb.title, t.title
ORDER BY cs.created_at DESC
LIMIT 50;
```

### Family Storage Usage

```sql
-- Calculate detailed storage usage for a family
WITH storage_details AS (
    SELECT 
        mb.id as memory_book_id,
        mb.title,
        COUNT(DISTINCT cs.id) as segment_count,
        SUM(LENGTH(cs.text_content)) as text_bytes,
        SUM(LENGTH(cs.audio_transcript)) as transcript_bytes,
        SUM(CASE WHEN cs.audio_url IS NOT NULL THEN 1 ELSE 0 END) as audio_count,
        SUM(ARRAY_LENGTH(cs.photo_urls, 1)) as photo_count,
        -- Estimate: 1MB per audio, 500KB per photo
        SUM(CASE WHEN cs.audio_url IS NOT NULL THEN 1048576 ELSE 0 END) as estimated_audio_bytes,
        SUM(ARRAY_LENGTH(cs.photo_urls, 1) * 524288) as estimated_photo_bytes
    FROM memory_books mb
    JOIN threads t ON mb.id = t.memory_book_id
    JOIN content_segments cs ON t.id = cs.thread_id
    WHERE mb.family_id = $1
      AND cs.is_deleted = false
    GROUP BY mb.id, mb.title
)
SELECT 
    f.id,
    f.name,
    f.storage_limit_bytes,
    COALESCE(SUM(
        sd.text_bytes + 
        sd.transcript_bytes + 
        sd.estimated_audio_bytes + 
        sd.estimated_photo_bytes
    ), 0) as total_used_bytes,
    json_agg(
        json_build_object(
            'memory_book_id', sd.memory_book_id,
            'title', sd.title,
            'segment_count', sd.segment_count,
            'text_bytes', sd.text_bytes,
            'audio_count', sd.audio_count,
            'photo_count', sd.photo_count,
            'total_bytes', sd.text_bytes + sd.transcript_bytes + 
                          sd.estimated_audio_bytes + sd.estimated_photo_bytes
        ) ORDER BY sd.text_bytes + sd.transcript_bytes + 
                   sd.estimated_audio_bytes + sd.estimated_photo_bytes DESC
    ) as breakdown
FROM families f
LEFT JOIN storage_details sd ON true
WHERE f.id = $1
GROUP BY f.id, f.name, f.storage_limit_bytes;
```

### Active User Sessions

```sql
-- Get active family members in the last 30 days
SELECT 
    fm.family_id,
    fm.user_id,
    fm.role,
    u.display_name,
    u.last_login_at,
    COUNT(DISTINCT cs.id) as recent_segments,
    MAX(cs.created_at) as last_contribution
FROM family_members fm
JOIN users u ON fm.user_id = u.id
LEFT JOIN content_segments cs ON cs.user_id = u.id 
    AND cs.created_at > NOW() - INTERVAL '30 days'
WHERE fm.family_id = $1
  AND fm.is_active = true
  AND u.last_login_at > NOW() - INTERVAL '30 days'
GROUP BY fm.family_id, fm.user_id, fm.role, u.display_name, u.last_login_at
ORDER BY u.last_login_at DESC;
```

## Database Sizing Estimates

### Storage Calculations

```yaml
Average Sizes:
  Text Segment: 500 bytes
  Audio Segment: 1 MB (60 seconds @ 128kbps)
  Photo: 500 KB (compressed)
  Metadata: 200 bytes per record

Usage Patterns:
  Active Families: 10,000
  Avg Members per Family: 8
  Avg Memory Books per Family: 5
  Avg Threads per Book: 20
  Avg Segments per Thread: 15
  
Monthly Growth:
  New Families: 500
  New Segments: 150,000
  Storage Growth: ~100 GB/month

First Year Projections:
  Total Families: 16,000
  Total Users: 128,000
  Total Segments: 1.8M
  Database Size: 1.2 TB
  Index Size: 200 GB
  Total Size: 1.4 TB
```

### Performance Targets

```yaml
Query Performance:
  Thread Load: < 50ms
  Memory Book List: < 100ms
  Search: < 200ms
  Family Dashboard: < 150ms

Concurrent Users: 10,000
Requests/Second: 1,000
Database Connections: 100
Connection Pool: 25 per service
```

### Backup Strategy

```yaml
Backup Schedule:
  Full Backup: Daily at 2 AM UTC
  Incremental: Every 6 hours
  WAL Archiving: Continuous
  
Retention:
  Daily Backups: 7 days
  Weekly Backups: 4 weeks
  Monthly Backups: 12 months
  
Storage:
  Primary: AWS RDS
  Backup: S3 with Glacier transition
  Disaster Recovery: Cross-region replication
```

## Security Considerations

### Row-Level Security

```sql
-- Enable RLS on sensitive tables
ALTER TABLE content_segments ENABLE ROW LEVEL SECURITY;

-- Policy: Users can only see content from their families
CREATE POLICY family_content_access ON content_segments
    FOR SELECT
    USING (
        thread_id IN (
            SELECT t.id 
            FROM threads t
            JOIN memory_books mb ON t.memory_book_id = mb.id
            JOIN family_members fm ON mb.family_id = fm.family_id
            WHERE fm.user_id = current_user_id()
              AND fm.is_active = true
        )
    );
```

### Audit Logging

```sql
-- Audit table for sensitive operations
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL,
    action VARCHAR(100) NOT NULL,
    table_name VARCHAR(100),
    record_id UUID,
    old_values JSONB,
    new_values JSONB,
    ip_address INET,
    user_agent TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_audit_log_user ON audit_log(user_id);
CREATE INDEX idx_audit_log_action ON audit_log(action);
CREATE INDEX idx_audit_log_created ON audit_log(created_at);

-- Partition by month for performance
ALTER TABLE audit_log PARTITION BY RANGE (created_at);
```

## Maintenance Scripts

### Vacuum and Analyze

```sql
-- Regular maintenance function
CREATE OR REPLACE FUNCTION perform_maintenance() RETURNS void AS $$
BEGIN
    -- Vacuum and analyze main tables
    VACUUM ANALYZE users;
    VACUUM ANALYZE families;
    VACUUM ANALYZE family_members;
    VACUUM ANALYZE memory_books;
    VACUUM ANALYZE threads;
    
    -- Vacuum partitioned tables
    EXECUTE 'VACUUM ANALYZE content_segments_' || to_char(NOW(), 'YYYY_MM');
    EXECUTE 'VACUUM ANALYZE content_segments_' || to_char(NOW() - interval '1 month', 'YYYY_MM');
    
    -- Update table statistics
    ANALYZE;
END;
$$ LANGUAGE plpgsql;

-- Schedule weekly execution
SELECT cron.schedule('maintenance', '0 0 * * 0', 'SELECT perform_maintenance()');
```

### Index Maintenance

```sql
-- Rebuild indexes that are bloated
CREATE OR REPLACE FUNCTION rebuild_bloated_indexes() RETURNS void AS $$
DECLARE
    idx record;
BEGIN
    FOR idx IN 
        SELECT schemaname, tablename, indexname 
        FROM pg_stat_user_indexes
        WHERE pg_relation_size(indexrelid) > 100000000 -- 100MB
          AND idx_scan < 100
    LOOP
        EXECUTE format('REINDEX INDEX CONCURRENTLY %I.%I', 
                      idx.schemaname, idx.indexname);
    END LOOP;
END;
$$ LANGUAGE plpgsql;
```