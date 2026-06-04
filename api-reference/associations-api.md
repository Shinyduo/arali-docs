# Associations API

The Associations API allows you to create relationships between contacts and companies, accounts, and deals. Use this API to link contacts to their organizations and deals.

## Base URL

```
/api/v1/associations
```

## Authentication

All endpoints require a static API key in the Authorization header:

```
Authorization: Api-Key <api_key>
```

---

## Key Concepts

### Separation of Concerns

| API | Responsibility |
|-----|----------------|
| **Contacts API** | Manage contact data (name, emails, phones, attributes) |
| **Associations API** | Manage relationships (contact→company, contact→account, contact→deal) |

### Association Flows

The API supports five flows for creating associations:

| Flow | Description | Creates |
|------|-------------|---------|
| **Flow 1: Company-only** | Associate contact directly with a company | `customer_company` |
| **Flow 2: Account-based** | Associate contact with an account (company auto-resolved) | `contact_account` + `customer_company` |
| **Flow 3: Account + Company** | Associate contact with account AND additional company | `contact_account` + multiple `customer_company` |
| **Flow 4: Deal-based** | Associate contact with a deal (company auto-resolved from deal) | `deal_contact` + `interaction_deal` + `customer_company` |
| **Flow 5: Deal + Account** | Associate contact with deal AND account | `deal_contact` + `interaction_deal` + `contact_account` + `customer_company` |

---

## Endpoints

### GET /api/v1/associations

Fetch enriched related entities for a contact, deal, company, or account.

**Required Scope:** `associations:read`

#### Query Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `entityType` | string | Yes | Type of entity: `contact`, `deal`, `company`, `account` |
| `external_id` | string | Conditional* | External ID of the entity |
| `internal_id` | string (UUID) | Conditional* | Internal UUID of the entity |
| `providerKey` | string | No | Accepted for backward compatibility only. It is not used for external ID lookup. |

\* Provide exactly one of `external_id` or `internal_id`

#### Response by Entity Type

| Entity Type | Returns |
|-------------|---------|
| **contact** | `companies[]`, `deals[]`, `accounts[]` |
| **deal** | `contacts[]`, `company`, `account` |
| **company** | `contacts[]`, `deals[]`, `accounts[]` |
| **account** | `contacts[]`, `deals[]`, `company` |

All returned entities are fully enriched with `stage`, `fieldValues`, emails, phones, and pipeline/stage details where applicable.

#### Example Request

```bash
curl -G "https://api.arali.ai/api/v1/associations" \
  -H "Authorization: Api-Key YOUR_STATIC_KEY" \
  --data-urlencode "entityType=contact" \
  --data-urlencode "external_id=hubspot_contact_12345"
```

#### Example Response

```json
{
  "success": true,
  "entityType": "contact",
  "entityId": "550e8400-e29b-41d4-a716-446655440000",
  "companies": [...],
  "deals": [...],
  "accounts": [...]
}
```

---

### GET /api/v1/associations/schema

Returns API documentation and schema information.

---

### POST /api/v1/associations

Batch create or update associations between contacts and companies, accounts, and deals.

**Required Scope:** `associations:write`

#### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `associations` | array | Yes | Array of association objects |

#### Association Object

You can use either internal UUIDs or native external IDs for all entities. External IDs resolve from the entity tables directly: `contacts.external_contact_id`, `companies.external_company_id`, `accounts.external_id`, and `deals.external_deal_id`.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `contactId` | string (UUID) | Conditional* | Internal contact ID |
| `externalContactId` | string | Conditional* | Native contact external ID from `contacts.external_contact_id` |
| `companyId` | string (UUID) | Conditional** | Internal company ID |
| `externalCompanyId` | string | Conditional** | Native company external ID from `companies.external_company_id` |
| `accountId` | string (UUID) | Conditional** | Internal account ID |
| `externalAccountId` | string | Conditional** | Native account external ID from `accounts.external_id` |
| `dealId` | string (UUID) | Conditional** | Internal deal ID |
| `externalDealId` | string | Conditional** | Native deal external ID from `deals.external_deal_id` |
| `relation` | enum | No | Relationship type (default: "employee"). Invalid values silently default to `employee` |
| `isPrimary` | boolean | No | Primary company flag (default: false). Forced to `false` for account-based associations |
| `role` | string | No | Role in account (for contact_account) |
| `dealRole` | string | No | Role in deal (for deal_contact). Default: "unknown". Invalid values silently default to `unknown` |

\* Must provide either `contactId` OR `externalContactId`  
\** Must provide at least one of: `companyId`, `externalCompanyId`, `accountId`, `externalAccountId`, `dealId`, `externalDealId`

#### Relation Types

| Value | Description |
|-------|-------------|
| `employee` | Full-time or part-time employee (default) |
| `consultant` | Consultant |
| `contractor` | Contractor |
| `partner` | Partner |
| `agency` | Agency |
| `other` | Other relationship |

#### Deal Role Types

| Value | Description |
|-------|-------------|
| `decision_maker` | Key decision maker |
| `economic_buyer` | Economic buyer |
| `champion` | Internal champion |
| `influencer` | Influencer |
| `user` | End user |
| `stakeholder` | Stakeholder |
| `admin` | Admin |
| `other` | Other role |
| `unknown` | Unknown role (default) |

---

## Example Requests

### Flow 1: Associate Contact with Company

```bash
curl -X POST https://api.arali.ai/api/v1/associations \
  -H "Authorization: Api-Key YOUR_STATIC_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "associations": [
      {
        "externalContactId": "hubspot_contact_12345",
        "externalCompanyId": "hubspot_company_67890",
        "relation": "employee",
        "isPrimary": true
      }
    ]
  }'
```

### Flow 2: Associate Contact with Account (Company Auto-Resolved)

When you associate a contact with an account, the system automatically:
1. Creates a `contact_account` association
2. Looks up the company linked to that account
3. Creates a `customer_company` association with that company

```bash
curl -X POST https://api.arali.ai/api/v1/associations \
  -H "Authorization: Api-Key YOUR_STATIC_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "associations": [
      {
        "externalContactId": "vitally_user_abc123",
        "externalAccountId": "vitally_acc_xyz789",
        "role": "decision_maker"
      }
    ]
  }'
```

### Flow 3: Associate Contact with Deal (Company Auto-Resolved)

When you associate a contact with a deal, the system automatically:
1. Creates a `deal_contact` association
2. Looks up the company linked to that deal
3. Creates a `customer_company` association with that company
4. Creates `interaction_deal` records for all interactions the contact participates in

```bash
curl -X POST https://api.arali.ai/api/v1/associations \
  -H "Authorization: Api-Key YOUR_STATIC_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "associations": [
      {
        "externalContactId": "hubspot_contact_12345",
        "externalDealId": "hubspot_deal_99999",
        "dealRole": "champion",
        "isPrimary": true
      }
    ]
  }'
```

### Flow 4: Using Direct UUIDs

If you already have internal UUIDs, use them directly:

```bash
curl -X POST https://api.arali.ai/api/v1/associations \
  -H "Authorization: Api-Key YOUR_STATIC_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "associations": [
      {
        "contactId": "550e8400-e29b-41d4-a716-446655440000",
        "companyId": "661e8400-e29b-41d4-a716-446655440111",
        "relation": "employee",
        "isPrimary": false
      }
    ]
  }'
```

### Batch: Multiple Associations

```bash
curl -X POST https://api.arali.ai/api/v1/associations \
  -H "Authorization: Api-Key YOUR_STATIC_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "associations": [
      {
        "externalContactId": "sf_contact_001",
        "externalCompanyId": "sf_company_001",
        "relation": "employee",
        "isPrimary": true
      },
      {
        "externalContactId": "sf_contact_001",
        "externalCompanyId": "sf_company_002",
        "relation": "consultant",
        "isPrimary": false
      },
      {
        "externalContactId": "sf_contact_002",
        "externalAccountId": "sf_account_001",
        "role": "admin"
      }
    ]
  }'
```

---

## Example Response

```json
{
  "status": "success",
  "summary": { "total": 3, "successful": 3, "skipped": 0, "failed": 0 },
  "processingTimeMs": 245,
  "results": [
    {
      "status": "success",
      "flow": "company_direct"
    },
    {
      "status": "success",
      "flow": "account_based"
    },
    {
      "status": "success",
      "flow": "deal_direct"
    }
  ]
}
```

---

## Upsert Behavior

The API uses **upsert** (insert or update) semantics:

| Scenario | Behavior |
|----------|----------|
| New association | Insert new record |
| Existing association (same contact + company) | Update relation, isPrimary, attributes |
| Existing association (same contact + account) | Update role, isPrimary |
| Existing association (same contact + deal) | Update dealRole, isPrimary |

### Conflict Resolution

- **Conflict key for `customer_company`:** `[contactId, companyId]`
- **Conflict key for `contact_account`:** `[contactId, accountId]`
- **Conflict key for `deal_contact`:** `[contactId, dealId]`

---

## Error Responses

| Status Code | Description |
|-------------|-------------|
| `400` | Bad Request - Missing required fields |
| `401` | Unauthorized - Invalid or missing token |
| `403` | Forbidden - Insufficient permissions |
| `404` | Not Found - Contact, company, or account not found |
| `500` | Internal Server Error |

### Skipped Results

Some associations may be skipped (not failed) with reasons:

| Reason | Description |
|--------|-------------|
| `Contact not found` | External ID not found in `contacts.external_contact_id` |
| `Company not found` | External ID not found in `companies.external_company_id` |
| `Account not found` | External ID not found in `accounts.external_id` |
| `Deal not found` | External ID not found in `deals.external_deal_id` |
| `Account has no associated company` | Account exists but has no companyId |
| `Missing contact identifier` | Neither contactId nor externalContactId provided |
| `Missing company/account/deal identifier` | No company, account, or deal identifier provided |

---

## Database Tables

| Table | Purpose |
|-------|---------|
| `customer_company` | Links contacts to companies with relationship metadata |
| `contact_account` | Links contacts to accounts with role information |
| `interaction_company` | Links interactions to companies |
| `interaction_account` | Links interactions to accounts |
| `interaction_deal` | Links interactions to deals |
| `deal_contact` | Links contacts to deals with role information |

---

## Related APIs

- **[Contacts API](./contacts-api.md)** - Manage contact records (name, emails, phones)
- **Companies API** - Manage company records
- **Accounts API** - Manage account records
- **Deals API** - Manage deal records
