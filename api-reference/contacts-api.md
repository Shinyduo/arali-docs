# Contacts API

The Contacts API allows you to create, read, and update contact records. Contacts can have multiple email addresses and phone numbers associated with them.

> **Note:** To associate a contact with a company or account, use the [Associations API](./associations-api.md).

## Base URL

```
/api/v1/contacts
```

## Authentication

All endpoints require a static API key in the Authorization header:

```
Authorization: Api-Key <static_key>
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

Batch create or update multiple contacts. Uses upsert behavior based on `externalContactId`.

**Required Scope:** `contacts:write`

#### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `contacts` | array | Yes | Array of contact objects |

#### Contact Object

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `externalContactId` | string | Yes | External unique identifier stored on `contacts.external_contact_id` and used for idempotent upserts |
| `fullName` | string \| null | No | Contact's full name |
| `title` | string \| null | No | Job title |
| `ownerUserId` | string (UUID) \| null | No | Internal user ID who owns this contact |
| `emails` | array | No | Array of email objects |
| `phones` | array | No | Array of phone objects |
| `attributes` | object | No | Custom key-value attributes |
| `properties` | object | No | Custom field values (must match field_definitions) |
| `touchpoint` | object | No | Marketing touchpoint (ad attribution) to record for this contact. See [Marketing Touchpoint Object](#marketing-touchpoint-object). |

#### Marketing Touchpoint Object

A `touchpoint` records one row in `marketing_touchpoint`, linking the contact to ad spend.
When `touchpoint` is omitted, the API can still derive attribution from legacy keys:
`attributes.meta_*` (Meta webhook), or `properties` containing `ad_id`, `utm_*`, `fbclid`/`gclid`/`ttclid`, `landing_page_url`, etc.

| Field | Type | Description |
|-------|------|-------------|
| `provider` | string \| null | Ad network: `meta`, `google`, `tiktok`, `website`, `whatsapp`. Default: `website`. |
| `lead_id` | string \| null | Network's own lead id (e.g. Meta leadgen id). Used with `provider` as the unique key. |
| `external_id_kind` | string \| null | `leadgen_id`, `google_lead_id`, `tiktok_lead_id`. Inferred from `provider` when absent. |
| `ad_id` | string \| null | Numeric network ad id. Non-numeric values are stored as `utm.content`, never as an ad id. |
| `adset_id` | string \| null | Ad set / ad group id. |
| `campaign_id` | string \| null | Campaign id. |
| `ad_account_id` | string \| null | Ad account id. |
| `ad_name` | string \| null | Human-readable ad name. |
| `adset_name` | string \| null | Human-readable ad set name. |
| `campaign_name` | string \| null | Human-readable campaign name. |
| `form_id` | string \| null | Lead-form id (Meta instant forms). |
| `platform` | string \| null | Placement: `fb`, `ig`, `placement`. |
| `is_organic` | boolean \| null | Whether the touch was organic. |
| `click_id` | string \| null | Click id: `fbclid`, `gclid`, `ttclid`, `ctwa_clid`. |
| `click_id_kind` | string \| null | Inferred from `click_id` / `provider` when absent. |
| `utm` | object \| null | UTM parameters: `{ source, medium, campaign, term, content }`. |
| `landing_url` | string \| null | Landing page URL. |
| `referrer` | string \| null | Referrer URL. |
| `visitor_id` | string \| null | Anonymous visitor id. |
| `session_id` | string \| null | Session id. |
| `touch_type` | string \| null | `lead_form`, `web_form`, `wa_message`, `ad_click`. Default: `web_form` for explicit blocks. |
| `occurred_at` | string \| null | ISO 8601 timestamp of the touch. Defaults to contact `createdAt`. |
| `ingest_source` | string \| null | `meta_webhook`, `contacts_api`, `backfill_attributes`, `backfill_field_values`, `snippet`. Default: `contacts_api`. |

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
  -H "Authorization: Api-Key YOUR_STATIC_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "contacts": [
      {
        "externalContactId": "hubspot_contact_12345",
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
      "externalContactId": "hubspot_contact_12345",
      "contactId": "550e8400-e29b-41d4-a716-446655440001",
      "status": "success"
    }
  ]
}
```

#### Example Request with Marketing Touchpoint

```bash
curl -X POST https://api.arali.ai/api/v1/contacts \
  -H "Authorization: Api-Key YOUR_STATIC_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "contacts": [
      {
        "externalContactId": "meta_lead_1234567890",
        "fullName": "John Smith",
        "title": "Product Manager",
        "emails": [
          { "email": "john@acme.com", "isPrimary": true, "label": "Work" }
        ],
        "phones": [
          { "phone": "+1234567890", "isPrimary": true, "label": "Mobile" }
        ],
        "touchpoint": {
          "provider": "meta",
          "lead_id": "1234567890",
          "ad_id": "123456789",
          "adset_id": "987654321",
          "campaign_id": "555555555",
          "ad_account_id": "act_123456789",
          "form_id": "form_987654321",
          "platform": "fb",
          "is_organic": false,
          "utm": {
            "source": "facebook",
            "medium": "paid_social",
            "campaign": "summer_2026",
            "term": "sales",
            "content": "carousel_v1"
          },
          "landing_url": "https://arali.ai/sales",
          "referrer": "https://facebook.com",
          "visitor_id": "vis_abc123",
          "session_id": "sess_xyz789",
          "touch_type": "lead_form",
          "occurred_at": "2026-09-05T14:30:00Z",
          "ingest_source": "contacts_api"
        }
      }
    ]
  }'
```

A replay of the same `provider` + `lead_id` (or a minted key when no `lead_id` exists) will not create a duplicate `marketing_touchpoint` row; later writes only fill in missing ids.

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
  -H "Authorization: Api-Key YOUR_STATIC_KEY"
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
  -H "Authorization: Api-Key YOUR_STATIC_KEY"
```

#### Example Response

If present, `companies` reflects relationships created via the Associations API.

```json
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440001",
    "enterpriseId": "550e8400-e29b-41d4-a716-446655440000",
    "externalContactId": "hubspot_contact_12345",
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
  -H "Authorization: Api-Key YOUR_STATIC_KEY" \
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
  -H "Authorization: Api-Key YOUR_STATIC_KEY"
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
