# API Documentation

> Complete API reference for the Career Options Explorer application

## Base URL

```
Development: http://localhost:3001/api/v1
Production: https://your-domain.com/api/v1
```

## Authentication

Admin endpoints require authentication. Currently using JWT tokens:

```http
Authorization: Bearer <jwt_token>
```

## Response Format

All API responses follow this standard format:

```json
{
  "success": true,
  "data": {},
  "message": "Optional message",
  "pagination": {}, // Only for paginated responses
  "facets": {} // Only for search responses
}
```

## Error Responses

```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "Human readable error message",
    "details": {} // Optional additional error details
  }
}
```

## Public Endpoints

### Careers

#### GET /careers

Retrieve a list of published careers with optional filtering and pagination.

**Query Parameters:**
- `q` (string): Search query
- `category` (string): Filter by category (comma-separated)
- `skill` (string): Filter by skills (comma-separated)
- `education` (string): Filter by education level
- `page` (number): Page number (default: 1)
- `limit` (number): Items per page (default: 12, max: 100)
- `sort` (string): Sort order (`recent`, `title`, `popular`)

**Response:**
```json
{
  "success": true,
  "data": {
    "careers": [
      {
        "id": 1,
        "title": "Data Scientist",
        "slug": "data-scientist",
        "snapshot": "Analyze complex datasets to extract actionable insights...",
        "tags": [
          {
            "name": "Python",
            "category": "skill",
            "relevanceScore": 0.95
          }
        ],
        "createdAt": "2024-01-01T00:00:00Z",
        "updatedAt": "2024-01-01T00:00:00Z"
      }
    ]
  },
  "pagination": {
    "currentPage": 1,
    "totalPages": 5,
    "totalItems": 50,
    "itemsPerPage": 12
  },
  "facets": {
    "categories": [
      { "name": "Technology", "count": 15 },
      { "name": "Healthcare", "count": 8 }
    ],
    "skills": [
      { "name": "Python", "count": 12 },
      { "name": "JavaScript", "count": 8 }
    ]
  }
}
```

#### GET /careers/:slug

Retrieve detailed information for a specific career.

**Parameters:**
- `slug` (string): Career slug identifier

**Response:**
```json
{
  "success": true,
  "data": {
    "career": {
      "id": 1,
      "title": "Data Scientist",
      "slug": "data-scientist",
      "snapshot": "Brief career overview...",
      "overview": "Detailed career description...",
      "sections": [
        {
          "id": 1,
          "title": "Skills Needed",
          "content": "Required skills and competencies...",
          "displayOrder": 1,
          "sectionType": "SKILLS"
        }
      ],
      "tags": [
        {
          "name": "Python",
          "category": "skill",
          "relevanceScore": 0.95
        }
      ],
      "relatedCareers": [
        {
          "id": 2,
          "title": "Machine Learning Engineer",
          "slug": "machine-learning-engineer",
          "snapshot": "Brief overview..."
        }
      ]
    }
  }
}
```

### Tags

#### GET /tags

Retrieve available tags for filtering.

**Query Parameters:**
- `category` (string): Filter by tag category
- `popular` (boolean): Return only popular tags
- `limit` (number): Maximum number of tags to return

**Response:**
```json
{
  "success": true,
  "data": {
    "tags": [
      {
        "id": 1,
        "name": "Python",
        "category": "skill",
        "careerCount": 15
      }
    ]
  }
}
```

### Feedback

#### POST /feedback

Submit user feedback for a career page.

**Request Body:**
```json
{
  "careerId": 1,
  "pageUrl": "/careers/data-scientist",
  "isHelpful": true,
  "comment": "Very informative and well-structured!"
}
```

**Response:**
```json
{
  "success": true,
  "message": "Feedback submitted successfully"
}
```

## Admin Endpoints

> All admin endpoints require authentication

### Career Management

#### GET /admin/careers

Retrieve all careers including drafts and archived items.

**Query Parameters:**
- `status` (string): Filter by status (`draft`, `published`, `archived`, `parsing_issue`)
- `page` (number): Page number
- `limit` (number): Items per page
- `sort` (string): Sort order

**Response:**
```json
{
  "success": true,
  "data": {
    "careers": [
      {
        "id": 1,
        "title": "Data Scientist",
        "slug": "data-scientist",
        "status": "published",
        "qualityScore": 0.85,
        "templateVersion": "v1.0",
        "sourceDocxFilename": "data-scientist.docx",
        "lastParsedAt": "2024-01-01T00:00:00Z",
        "parsingNotes": "Successfully parsed all sections",
        "createdAt": "2024-01-01T00:00:00Z",
        "updatedAt": "2024-01-01T00:00:00Z"
      }
    ]
  },
  "pagination": {
    "currentPage": 1,
    "totalPages": 3,
    "totalItems": 25,
    "itemsPerPage": 10
  }
}
```

#### POST /admin/careers

Create a new career manually.

**Request Body:**
```json
{
  "title": "Software Engineer",
  "slug": "software-engineer",
  "snapshot": "Brief overview...",
  "overview": "Detailed description...",
  "status": "draft",
  "sections": [
    {
      "title": "Skills Needed",
      "content": "Programming skills...",
      "displayOrder": 1,
      "sectionType": "SKILLS"
    }
  ],
  "tags": [
    {
      "name": "JavaScript",
      "category": "skill",
      "relevanceScore": 0.9
    }
  ]
}
```

#### PUT /admin/careers/:id

Update an existing career.

**Parameters:**
- `id` (number): Career ID

**Request Body:** Same as POST /admin/careers

#### PATCH /admin/careers/:id/archive

Archive a career (soft delete).

#### PATCH /admin/careers/:id/restore

Restore an archived career.

#### DELETE /admin/careers/:id/permanently

Permanently delete a career (hard delete).

### File Upload

#### POST /admin/upload/preview

Preview parsed content from a DOCX file without saving.

**Request:**
- Content-Type: `multipart/form-data`
- Field: `docxFile` (file)

**Response:**
```json
{
  "success": true,
  "data": {
    "parsedCareer": {
      "title": "Data Scientist",
      "slug": "data-scientist",
      "snapshot": "Generated snapshot...",
      "overview": "Generated overview...",
      "sections": [...],
      "tags": [...],
      "qualityScore": 0.85,
      "parsingNotes": ["Successfully parsed all sections"],
      "templateVersion": "v1.0"
    }
  }
}
```

#### POST /admin/upload

Parse and import a DOCX file into the database.

**Request:**
- Content-Type: `multipart/form-data`
- Field: `docxFile` (file)
- Optional: `templateVersion` (string)
- Optional: `status` (string, default: "draft")

**Response:**
```json
{
  "success": true,
  "data": {
    "career": {
      "id": 123,
      "title": "Data Scientist",
      "slug": "data-scientist",
      "status": "draft"
    }
  },
  "message": "Career imported successfully"
}
```

### Feedback Management

#### GET /admin/feedback

Retrieve user feedback for review.

**Query Parameters:**
- `careerId` (number): Filter by career ID
- `isHelpful` (boolean): Filter by helpful/not helpful
- `page` (number): Page number
- `limit` (number): Items per page

**Response:**
```json
{
  "success": true,
  "data": {
    "feedback": [
      {
        "id": 1,
        "careerId": 1,
        "careerTitle": "Data Scientist",
        "pageUrl": "/careers/data-scientist",
        "isHelpful": true,
        "comment": "Very informative!",
        "submittedAt": "2024-01-01T00:00:00Z"
      }
    ]
  },
  "pagination": {...}
}
```

## Status Codes

- `200` - Success
- `201` - Created
- `400` - Bad Request
- `401` - Unauthorized
- `403` - Forbidden
- `404` - Not Found
- `422` - Validation Error
- `500` - Internal Server Error

## Rate Limiting

- Public endpoints: 100 requests per minute per IP
- Admin endpoints: 200 requests per minute per authenticated user

## Validation

All endpoints validate input data. Validation errors return a 422 status with details:

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": {
      "title": ["Title is required"],
      "slug": ["Slug must be unique"]
    }
  }
}
```

## Examples

### Search for Technology Careers

```bash
curl "http://localhost:3001/api/v1/careers?category=Technology&limit=5"
```

### Upload and Preview DOCX

```bash
curl -X POST \
  -H "Authorization: Bearer <token>" \
  -F "docxFile=@career.docx" \
  "http://localhost:3001/api/v1/admin/upload/preview"
```

### Submit Feedback

```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -d '{"careerId": 1, "isHelpful": true, "comment": "Great info!"}' \
  "http://localhost:3001/api/v1/feedback"
```

---

*For more examples and interactive testing, see the Swagger documentation at `/api/docs` when running the development server.*