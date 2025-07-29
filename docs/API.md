# FamilyTales REST API Documentation

## Overview

The FamilyTales API provides comprehensive endpoints for managing family memories, including document scanning, audio generation, family management, and subscription handling. All API endpoints are served over HTTPS and require authentication via Clerk JWT tokens.

## Base URL

```
Production: https://api.familytales.app/v1
Staging: https://staging-api.familytales.app/v1
Local: http://localhost:3000/v1
```

## Authentication

All API requests require authentication using Bearer tokens provided by Clerk.

```http
Authorization: Bearer <clerk_jwt_token>
```

## Common Headers

```http
Content-Type: application/json
X-Family-Context: <family_id> // Optional, specifies current family context
X-Client-Version: 1.0.0
```

## Response Format

All responses follow a consistent structure:

```json
{
  "success": true,
  "data": {...},
  "meta": {
    "request_id": "uuid",
    "timestamp": "2025-01-29T12:00:00Z"
  }
}
```

Error responses:

```json
{
  "success": false,
  "error": {
    "code": "RESOURCE_NOT_FOUND",
    "message": "Memory book not found",
    "details": {...}
  },
  "meta": {...}
}
```

## Rate Limiting

- **Free tier**: 100 requests per hour
- **Premium tier**: 1000 requests per hour
- **Enterprise tier**: 10000 requests per hour

Rate limit headers:

```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1672531200
```

## API Endpoints

### Authentication Endpoints

#### POST /auth/login
Authenticate user with Clerk token.

```http
POST /auth/login
Content-Type: application/json

{
  "clerk_token": "string"
}
```

Response:
```json
{
  "success": true,
  "data": {
    "user": {
      "id": "uuid",
      "email": "user@example.com",
      "display_name": "John Doe",
      "subscription_tier": "premium"
    },
    "families": [
      {
        "id": "uuid",
        "name": "Smith Family",
        "role": "admin"
      }
    ],
    "session_token": "jwt_token"
  }
}
```

#### POST /auth/logout
Invalidate current session.

```http
POST /auth/logout
Authorization: Bearer <token>
```

#### GET /auth/me
Get current user profile.

```http
GET /auth/me
Authorization: Bearer <token>
```

### Family Management Endpoints

#### GET /families
List all families user belongs to.

```http
GET /families
Authorization: Bearer <token>
```

Response:
```json
{
  "success": true,
  "data": {
    "families": [
      {
        "id": "uuid",
        "name": "Smith Family",
        "created_at": "2025-01-29T12:00:00Z",
        "member_count": 12,
        "role": "admin",
        "subscription": {
          "tier": "premium",
          "expires_at": "2025-12-31T23:59:59Z"
        }
      }
    ]
  }
}
```

#### POST /families
Create a new family.

```http
POST /families
Authorization: Bearer <token>
Content-Type: application/json

{
  "name": "Johnson Family",
  "description": "The Johnson family memories"
}
```

#### GET /families/{family_id}
Get family details.

```http
GET /families/{family_id}
Authorization: Bearer <token>
```

#### PUT /families/{family_id}
Update family information.

```http
PUT /families/{family_id}
Authorization: Bearer <token>
Content-Type: application/json

{
  "name": "Updated Family Name",
  "description": "Updated description"
}
```

#### POST /families/{family_id}/invites
Create family invitation.

```http
POST /families/{family_id}/invites
Authorization: Bearer <token>
Content-Type: application/json

{
  "email": "newmember@example.com",
  "role": "member",
  "message": "Join our family memories!"
}
```

Response:
```json
{
  "success": true,
  "data": {
    "invite": {
      "id": "uuid",
      "invite_code": "FAM123ABC",
      "invite_link": "https://familytales.app/join/FAM123ABC",
      "expires_at": "2025-02-05T12:00:00Z"
    }
  }
}
```

#### POST /invites/{invite_code}/accept
Accept family invitation.

```http
POST /invites/{invite_code}/accept
Authorization: Bearer <token>
```

#### GET /families/{family_id}/members
List family members.

```http
GET /families/{family_id}/members
Authorization: Bearer <token>
```

### Memory Book Endpoints

#### GET /families/{family_id}/memory-books
List all memory books in a family.

```http
GET /families/{family_id}/memory-books
Authorization: Bearer <token>
Query Parameters:
  - page: number (default: 1)
  - limit: number (default: 20, max: 100)
  - sort: created_at|updated_at|title (default: updated_at)
  - order: asc|desc (default: desc)
```

#### POST /families/{family_id}/memory-books
Create a new memory book.

```http
POST /families/{family_id}/memory-books
Authorization: Bearer <token>
Content-Type: application/json

{
  "title": "Grandma's Letters",
  "description": "Love letters from World War II",
  "cover_image_url": "https://...",
  "metadata": {
    "year": "1942-1945",
    "tags": ["letters", "war", "love"]
  }
}
```

#### GET /memory-books/{book_id}
Get memory book details.

```http
GET /memory-books/{book_id}
Authorization: Bearer <token>
```

#### PUT /memory-books/{book_id}
Update memory book.

```http
PUT /memory-books/{book_id}
Authorization: Bearer <token>
Content-Type: application/json

{
  "title": "Updated Title",
  "description": "Updated description"
}
```

#### DELETE /memory-books/{book_id}
Delete memory book.

```http
DELETE /memory-books/{book_id}
Authorization: Bearer <token>
```

### Document Processing Endpoints

#### POST /memory-books/{book_id}/documents
Upload and process a document.

```http
POST /memory-books/{book_id}/documents
Authorization: Bearer <token>
Content-Type: multipart/form-data

{
  "file": <binary>,
  "title": "Letter from 1943",
  "metadata": {
    "date": "1943-06-15",
    "author": "Grandpa Joe"
  }
}
```

Response:
```json
{
  "success": true,
  "data": {
    "document": {
      "id": "uuid",
      "status": "processing",
      "title": "Letter from 1943",
      "processing_job_id": "job_123"
    }
  }
}
```

#### GET /documents/{document_id}
Get document details.

```http
GET /documents/{document_id}
Authorization: Bearer <token>
```

Response:
```json
{
  "success": true,
  "data": {
    "document": {
      "id": "uuid",
      "title": "Letter from 1943",
      "status": "completed",
      "original_url": "https://...",
      "extracted_text": "Dear Mary...",
      "audio_url": "https://...",
      "duration_seconds": 180,
      "metadata": {...}
    }
  }
}
```

#### PUT /documents/{document_id}/text
Update extracted text (OCR corrections).

```http
PUT /documents/{document_id}/text
Authorization: Bearer <token>
Content-Type: application/json

{
  "text": "Corrected text content...",
  "regenerate_audio": true
}
```

#### POST /documents/{document_id}/regenerate-audio
Regenerate audio with different voice.

```http
POST /documents/{document_id}/regenerate-audio
Authorization: Bearer <token>
Content-Type: application/json

{
  "voice_id": "en-US-standard-B",
  "speed": 1.0
}
```

### Media Endpoints

#### POST /upload/presigned-url
Get presigned URL for direct upload to Mux.

```http
POST /upload/presigned-url
Authorization: Bearer <token>
Content-Type: application/json

{
  "filename": "grandma_letter.jpg",
  "content_type": "image/jpeg",
  "size_bytes": 2048000
}
```

Response:
```json
{
  "success": true,
  "data": {
    "upload_url": "https://storage.mux.com/...",
    "upload_id": "upload_123",
    "expires_at": "2025-01-29T13:00:00Z"
  }
}
```

#### POST /upload/{upload_id}/complete
Confirm upload completion.

```http
POST /upload/{upload_id}/complete
Authorization: Bearer <token>
```

### Subscription Endpoints

#### GET /subscription
Get current subscription details.

```http
GET /subscription
Authorization: Bearer <token>
```

Response:
```json
{
  "success": true,
  "data": {
    "subscription": {
      "tier": "premium",
      "status": "active",
      "expires_at": "2025-12-31T23:59:59Z",
      "features": {
        "max_families": 2,
        "max_storage_gb": 100,
        "premium_voices": true,
        "bulk_upload": true
      },
      "usage": {
        "storage_used_gb": 23.5,
        "documents_processed": 156,
        "audio_minutes_generated": 1250
      }
    }
  }
}
```

#### POST /subscription/checkout
Create Stripe checkout session.

```http
POST /subscription/checkout
Authorization: Bearer <token>
Content-Type: application/json

{
  "plan": "premium_yearly",
  "success_url": "https://app.familytales.app/subscription/success",
  "cancel_url": "https://app.familytales.app/subscription"
}
```

Response:
```json
{
  "success": true,
  "data": {
    "checkout_url": "https://checkout.stripe.com/...",
    "session_id": "cs_test_123"
  }
}
```

#### POST /subscription/cancel
Cancel subscription.

```http
POST /subscription/cancel
Authorization: Bearer <token>
```

### Search Endpoints

#### GET /search
Search across all accessible content.

```http
GET /search
Authorization: Bearer <token>
Query Parameters:
  - q: string (search query)
  - type: all|documents|memory_books (default: all)
  - family_id: uuid (optional)
  - from: date (optional)
  - to: date (optional)
  - page: number
  - limit: number
```

### Analytics Endpoints

#### GET /analytics/usage
Get usage statistics.

```http
GET /analytics/usage
Authorization: Bearer <token>
X-Family-Context: <family_id>
Query Parameters:
  - period: day|week|month|year
  - from: date
  - to: date
```

Response:
```json
{
  "success": true,
  "data": {
    "usage": {
      "documents_scanned": 45,
      "audio_minutes_generated": 180,
      "storage_used_mb": 2048,
      "family_members_active": 8,
      "most_active_members": [...],
      "popular_memory_books": [...]
    }
  }
}
```

## Webhook Events

FamilyTales sends webhooks for important events:

### Document Processing Complete
```json
{
  "event": "document.processing.complete",
  "data": {
    "document_id": "uuid",
    "memory_book_id": "uuid",
    "family_id": "uuid",
    "status": "success",
    "audio_url": "https://..."
  }
}
```

### Subscription Updated
```json
{
  "event": "subscription.updated",
  "data": {
    "user_id": "uuid",
    "old_tier": "free",
    "new_tier": "premium",
    "expires_at": "2025-12-31T23:59:59Z"
  }
}
```

### Family Member Joined
```json
{
  "event": "family.member.joined",
  "data": {
    "family_id": "uuid",
    "user_id": "uuid",
    "invited_by": "uuid",
    "role": "member"
  }
}
```

## Error Codes

| Code | Description |
|------|-------------|
| AUTH_INVALID_TOKEN | Invalid or expired authentication token |
| AUTH_INSUFFICIENT_PERMISSIONS | User lacks required permissions |
| RESOURCE_NOT_FOUND | Requested resource not found |
| VALIDATION_ERROR | Request validation failed |
| RATE_LIMIT_EXCEEDED | Too many requests |
| STORAGE_LIMIT_EXCEEDED | Storage quota exceeded |
| PROCESSING_FAILED | Document processing failed |
| PAYMENT_FAILED | Payment processing failed |
| FAMILY_LIMIT_REACHED | Maximum families reached for tier |

## OpenAPI Specification

```yaml
openapi: 3.0.0
info:
  title: FamilyTales API
  version: 1.0.0
  description: API for preserving and sharing family memories through audio
servers:
  - url: https://api.familytales.app/v1
    description: Production server
security:
  - bearerAuth: []
components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
  schemas:
    User:
      type: object
      properties:
        id:
          type: string
          format: uuid
        email:
          type: string
          format: email
        display_name:
          type: string
        subscription_tier:
          type: string
          enum: [free, premium, enterprise]
    Family:
      type: object
      properties:
        id:
          type: string
          format: uuid
        name:
          type: string
        created_at:
          type: string
          format: date-time
        member_count:
          type: integer
    MemoryBook:
      type: object
      properties:
        id:
          type: string
          format: uuid
        title:
          type: string
        description:
          type: string
        document_count:
          type: integer
        total_duration_seconds:
          type: integer
    Document:
      type: object
      properties:
        id:
          type: string
          format: uuid
        title:
          type: string
        status:
          type: string
          enum: [pending, processing, completed, failed]
        original_url:
          type: string
        audio_url:
          type: string
        extracted_text:
          type: string
paths:
  /auth/login:
    post:
      summary: Authenticate user
      tags: [Authentication]
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [clerk_token]
              properties:
                clerk_token:
                  type: string
      responses:
        200:
          description: Authentication successful
        401:
          description: Invalid credentials
```

## SDK Examples

### JavaScript/TypeScript

```typescript
import { FamilyTalesClient } from '@familytales/sdk';

const client = new FamilyTalesClient({
  apiKey: process.env.FAMILYTALES_API_KEY,
  baseUrl: 'https://api.familytales.app/v1'
});

// Upload and process a document
const memoryBook = await client.memoryBooks.get('book_123');
const document = await client.documents.upload(memoryBook.id, {
  file: fileBuffer,
  title: 'Grandma\'s Letter',
  metadata: {
    date: '1943-06-15',
    author: 'Grandma Rose'
  }
});

// Poll for processing completion
const processed = await client.documents.waitForProcessing(document.id);
console.log('Audio URL:', processed.audio_url);
```

### Python

```python
from familytales import FamilyTalesClient

client = FamilyTalesClient(
    api_key=os.environ['FAMILYTALES_API_KEY'],
    base_url='https://api.familytales.app/v1'
)

# Create a memory book
memory_book = client.memory_books.create(
    family_id='family_123',
    title="Grandma's Letters",
    description="Love letters from World War II"
)

# Upload document
with open('letter.jpg', 'rb') as f:
    document = client.documents.upload(
        memory_book_id=memory_book.id,
        file=f,
        title='Letter from 1943'
    )

# Get processing status
status = client.documents.get(document.id)
print(f"Status: {status.status}")
```

### cURL

```bash
# Upload a document
curl -X POST https://api.familytales.app/v1/memory-books/book_123/documents \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -F "file=@grandma_letter.jpg" \
  -F "title=Letter from 1943" \
  -F 'metadata={"date":"1943-06-15","author":"Grandma Rose"}'

# Get document status
curl https://api.familytales.app/v1/documents/doc_456 \
  -H "Authorization: Bearer YOUR_TOKEN"
```

## Best Practices

1. **Authentication**: Always validate tokens server-side
2. **Rate Limiting**: Implement exponential backoff for retries
3. **File Uploads**: Use presigned URLs for large files
4. **Webhooks**: Verify webhook signatures
5. **Pagination**: Always paginate list endpoints
6. **Error Handling**: Check for specific error codes
7. **Caching**: Cache family and user data appropriately

## Support

For API support:
- Email: api-support@familytales.app
- Developer Portal: https://developers.familytales.app
- Status Page: https://status.familytales.app