# Contacts API

The Contacts API allows you to create, read, and update contact records. Contacts can have multiple email addresses and phone numbers associated with them.

> **Note:** To associate a contact with a company or account, use the [Associations API](./associations-api.md).

## Base URL

```
/api/v1/contacts
```

## Authentication

All endpoints require a Bearer token in the Authorization header:

```
Authorization: Bearer <your-api-token>
```

---

## Endpoints

### GET /api/v1/contacts/schema

Returns API documentation and schema information.

**Response:**
```json
{
  "description": "Contacts API - Create, read, and update contacts",
  "version": "1.0.0",
  "authentication": { ... },
  "endpoints": { ... },
  "contactFields": { ... },
  "responseFields": { ... }
}
```

---

### POST /api/v1/contacts

Batch create or update multiple contacts. Uses upsert behavior based on `externalId`.

**Required Scope:** `contacts:write`

#### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `contacts` | array | Yes | Array of contact objects |

#### Contact Object

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `externalId` | string | Yes | External unique identifier from integration source |
| `fullName` | string \| null | No | Contact's full name |
| `title` | string \| null | No | Job title |
| `ownerUserId` | string (UUID) \| null | No | Internal user ID who owns this contact |
| `emails` | array | No | Array of email objects |
| `phones` | array | No | Array of phone objects |
| `attributes` | object | No | Custom key-value attributes |
| `properties` | object | No | Custom field values (must match field_definitions) |

#### Email Object

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `email` | string | Yes | Email address |
| `isPrimary` | boolean | No | Whether this is the primary email (default: false) |
| `label` | string | No | Label (e.g., "Work", "Personal") |

#### Phone Object

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `phone` | string | Yes | Phone number (any format) |
| `isPrimary` | boolean | No | Whether this is the primary phone (default: false) |
| `label` | string | No | Label (e.g., "Mobile", "Office") |

#### Important Notes (Email & Phone)

- Only one email can be `isPrimary: true` per contact (same for phone numbers).
- If multiple entries are passed with `isPrimary: true`, only the first is set as primary.
- When a new email/phone is set as primary, the existing primary is automatically set to `isPrimary: false`.

#### Important Notes (Companies)

- Company relationships are managed via the Associations API; the Contacts API does not accept a `companies` array in create/update requests.

#### Example Request

```bash
curl -X POST https://api.arali.ai/api/v1/contacts \
  -H "Authorization: Bearer YOUR_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "contacts": [
      {
        "externalId": "hubspot_contact_12345",
        "fullName": "John Doe",
        "title": "VP of Engineering",
        "emails": [
          { "email": "john.doe@acme.com", "isPrimary": true, "label": "Work" },
          { "email": "johndoe@gmail.com", "isPrimary": false, "label": "Personal" }
        ],
        "phones": [
          { "phone": "+1-555-123-4567", "isPrimary": true, "label": "Mobile" }
        ],
        "attributes": {
          "linkedin": "https://linkedin.com/in/johndoe",
          "timezone": "America/New_York"
        }
      }
    ]
  }'
```

#### Example Response

```json
{
  "success": true,
  "summary": { "total": 1, "success": 1, "failed": 0 },
  "results": [
    {
      "externalId": "hubspot_contact_12345",
      "contactId": "550e8400-e29b-41d4-a716-446655440001",
      "status": "success"
    }
  ]
}
```

---

### GET /api/v1/contacts

Retrieves a paginated list of contacts.

**Required Scope:** `contacts:read`

#### Query Parameters

| Parameter | Type | Default | Description |
|----------|------|---------|-------------|
| `limit` | integer | 50 | Number of contacts to return (max: 100) |
| `offset` | integer | 0 | Number of contacts to skip |

#### Example Request

```bash
curl -X GET "https://api.arali.ai/api/v1/contacts?limit=10&offset=0" \
  -H "Authorization: Bearer YOUR_API_TOKEN"
```

#### Example Response

```json
{
  "success": true,
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440001",
      "enterpriseId": "550e8400-e29b-41d4-a716-446655440000",
      "fullName": "John Doe",
      "title": "VP of Engineering",
      "ownerUserId": null,
      "attributes": {},
      "emails": [{ "email": "john.doe@acme.com", "isPrimary": true, "label": "Work" }],
      "phones": [{ "phone": "+1-555-123-4567", "isPrimary": true, "label": "Mobile" }],
      "createdAt": "2026-02-03T10:00:00.000Z",
      "updatedAt": "2026-02-03T10:00:00.000Z"
    }
  ],
  "pagination": { "limit": 10, "offset": 0, "total": 1 }
}
```

---

### GET /api/v1/contacts/{id}

Retrieve a single contact by ID, including emails, phones, and company associations.

**Required Scope:** `contacts:read`

#### Example Request

```bash
curl -X GET https://api.arali.ai/api/v1/contacts/550e8400-e29b-41d4-a716-446655440001 \
  -H "Authorization: Bearer YOUR_API_TOKEN"
```

#### Example Response

If present, `companies` reflects relationships created via the Associations API.

```json
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440001",
    "enterpriseId": "550e8400-e29b-41d4-a716-446655440000",
    "externalId": "hubspot_contact_12345",
    "fullName": "John Doe",
    "title": "VP of Engineering",
    "ownerUserId": null,
    "attributes": {
      "linkedin": "https://linkedin.com/in/johndoe"
    },
    "emails": [
      { "email": "john.doe@acme.com", "isPrimary": true, "label": "Work" }
    ],
    "phones": [
      { "phone": "+1-555-123-4567", "isPrimary": true, "label": "Mobile" }
    ],
    "companies": [
      {
        "companyId": "550e8400-e29b-41d4-a716-446655440000",
        "relation": "employee",
        "isPrimary": true
      }
    ],
    "createdAt": "2026-02-03T10:00:00.000Z",
    "updatedAt": "2026-02-03T10:00:00.000Z"
  }
}
```

---

### PUT /api/v1/contacts/{id}

Update a single contact by ID.

**Required Scope:** `contacts:write`

#### Request Body

| Field | Type | Description |
|-------|------|-------------|
| `fullName` | string \| null | Contact's full name |
| `title` | string \| null | Job title |
| `ownerUserId` | string (UUID) \| null | Owner user ID |
| `emails` | array | Email objects (will be upserted) |
| `phones` | array | Phone objects (will be upserted) |
| `attributes` | object | Custom attributes |
| `properties` | object | Custom field values |

#### Example Request

```bash
curl -X PUT https://api.arali.ai/api/v1/contacts/550e8400-e29b-41d4-a716-446655440001 \
  -H "Authorization: Bearer YOUR_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "CTO",
    "phones": [
      { "phone": "+1-555-999-8888", "isPrimary": true, "label": "Office" }
    ]
  }'
```

#### Example Response

```json
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440001",
    "title": "CTO"
  }
}
```

---

### DELETE /api/v1/contacts/{id}

Delete a contact by ID.

**Required Scope:** `contacts:delete`

#### Example Request

```bash
curl -X DELETE https://api.arali.ai/api/v1/contacts/550e8400-e29b-41d4-a716-446655440001 \
  -H "Authorization: Bearer YOUR_API_TOKEN"
```

#### Example Response

```json
{
  "success": true,
  "message": "Contact deleted successfully",
  "deletedId": "550e8400-e29b-41d4-a716-446655440001"
}
```

---

## Error Responses

| Status Code | Description |
|-------------|-------------|
| `400` | Bad Request - Invalid request body or missing required fields |
| `401` | Unauthorized - Invalid or missing token |
| `403` | Forbidden - Insufficient permissions (missing required scope) |
| `404` | Not Found - Contact not found |
| `500` | Internal Server Error |

---

## Related APIs

- **[Associations API](./associations-api.md)** - Create contact→company and contact→account associations
- **Companies API** - Manage company records
- **Accounts API** - Manage account records
