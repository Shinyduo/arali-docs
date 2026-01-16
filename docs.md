# Arali Backend Public API Documentation

This document provides comprehensive documentation for all public APIs exposed under `/api/v1`.

**Base URL:** `https://your-api-domain.com`  
**API Version:** `1.0.0`  
**Authentication:** Bearer Token

---

## Table of Contents

1. [Authentication](#authentication)
2. [Companies API](#companies-api)
3. [Accounts API](#accounts-api)
4. [Contacts API](#contacts-api)
5. [Associations API](#associations-api)
6. [Calls API](#calls-api)
7. [Meetings API](#meetings-api)
8. [Fields API](#fields-api)

---

## Authentication

All API endpoints require authentication via Bearer Token.

**Header Format:**
```
Authorization: Bearer <your-api-token>
```

The token automatically provides:
- `enterpriseId` - Your enterprise identifier
- `integrationAccountId` - Integration account context
- `scopes` - Authorized permissions

---

## Companies API

Manage company records for your enterprise.

### Endpoints

| Method | Endpoint | Description | Required Scope |
|--------|----------|-------------|----------------|
| POST | `/api/v1/companies` | Batch create/update companies | `companies:write` |
| GET | `/api/v1/companies` | List all companies | `companies:read` |
| GET | `/api/v1/companies/:id` | Get single company | `companies:read` |
| PUT | `/api/v1/companies/:id` | Update single company | `companies:write` |
| DELETE | `/api/v1/companies/:id` | Delete a company | `companies:delete` |
| GET | `/api/v1/companies/schema` | Get API schema | - |

### Company Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Company display name |
| `externalId` | string | Yes | **Primary External ID**. Used for integration mapping lookup. |
| `externalCompanyId` | string | No | **Secondary External ID**. Stored on the record. Defaults to `externalId` if not provided. |
| `domain` | string \| null | No | Company website domain |
| `lifecycleStage` | enum | No | `lead`, `prospect`, `customer`, `expansion`, `churned`, `lost`, `partner`, `other` (default: `lead`) |
| `ownerUserId` | UUID \| null | No | Owner user ID |
| `healthScore` | integer \| null | No | Health score (0-100) |
| `healthScoreComputedAt` | ISO 8601 datetime \| null | No | When health score was computed |
| `healthScoreBreakdown` | object \| null | No | Health score component breakdown |
| `arr` | integer \| null | No | Annual Recurring Revenue |
| `attributes` | object | No | Custom key-value attributes |
| `properties` | object | No | Custom field values (maps to `field_values` table) |

### Upsert Logic
1. **Mapping Lookup**: The API first checks for an existing mapping using `externalId`. If found, it updates the linked company.
2. **DB Constraint**: If no mapping is found, it attempts to insert. If a company with the same `externalCompanyId` exists in the database, it updates that record (and creates a new mapping).
3. **Insert**: If neither is found, a new company is created.

### cURL Examples

#### Create/Update Companies (Batch)

```bash
curl -X POST 'https://your-api-domain.com/api/v1/companies' \
  -H 'Authorization: Bearer YOUR_API_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
    "companies": [
      {
        "name": "Acme Corporation",
        "externalId": "hubspot_12345", 
        "externalCompanyId": "CRM_001", 
        "domain": "acme.com",
        "lifecycleStage": "customer",
        "healthScore": 85,
        "arr": 120000
      }
    ]
  }'
```

> **Note:** `providerKey` is optional and defaults to `"api"` if not provided.

#### List All Companies

```bash
curl -X GET 'https://your-api-domain.com/api/v1/companies' \
  -H 'Authorization: Bearer YOUR_API_TOKEN'
```

#### Get Single Company

```bash
curl -X GET 'https://your-api-domain.com/api/v1/companies/COMPANY_UUID' \
  -H 'Authorization: Bearer YOUR_API_TOKEN'
```

#### Update Single Company

```bash
curl -X PUT 'https://your-api-domain.com/api/v1/companies/COMPANY_UUID' \
  -H 'Authorization: Bearer YOUR_API_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "Acme Corp Updated",
    "healthScore": 90,
    "lifecycleStage": "expansion"
  }'
```

#### Delete Company

```bash
curl -X DELETE 'https://your-api-domain.com/api/v1/companies/COMPANY_UUID' \
  -H 'Authorization: Bearer YOUR_API_TOKEN'
```

---

## Accounts API

Manage account records linked to companies.

### Endpoints

| Method | Endpoint | Description | Required Scope |
|--------|----------|-------------|----------------|
| POST | `/api/v1/accounts` | Batch create/update accounts | `accounts:write` |
| GET | `/api/v1/accounts` | List all accounts | `accounts:read` |
| GET | `/api/v1/accounts/:id` | Get single account | `accounts:read` |
| PUT | `/api/v1/accounts/:id` | Update single account | `accounts:write` |
| DELETE | `/api/v1/accounts/:id` | Delete an account | `accounts:delete` |
| GET | `/api/v1/accounts/schema` | Get API schema | - |

### Account Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Account display name |
| `externalId` | string | Yes | **Primary External ID**. Used for integration mapping lookup. |
| `externalAccountId` | string | No | **Secondary External ID**. Stored on the record. Defaults to `externalId` if not provided. |
| `externalCompanyId` | string | Yes | External ID of the **parent company**. |
| `domain` | string \| null | No | Account domain |
| `accountType` | string \| null | No | Account type/tier (e.g., `enterprise`, `startup`, `smb`) |
| `ownerUserId` | UUID \| null | No | Owner user ID |
| `healthScore` | integer \| null | No | Health score (0-100) |
| `healthScoreComputedAt` | ISO 8601 datetime \| null | No | When health score was computed |
| `healthScoreBreakdown` | object \| null | No | Health score component breakdown |
| `arr` | integer \| null | No | Annual Recurring Revenue |
| `metadata` | object | No | Custom key-value metadata |
| `properties` | object | No | Custom field values (maps to `field_values` table) |

### Upsert Logic
1. **Mapping Lookup**: Checks for mapping using `externalId`.
2. **DB Constraint**: Validates/Upserts on `externalAccountId` (defaults to `externalId`).
3. **Company Link**: Must resolve `externalCompanyId` to an existing parent company.

### cURL Examples

#### Create/Update Accounts (Batch)

```bash
curl -X POST 'https://your-api-domain.com/api/v1/accounts' \
  -H 'Authorization: Bearer YOUR_API_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
    "accounts": [
      {
        "name": "Acme Enterprise Account",
        "externalId": "vitally_acc_12345",
        "externalAccountId": "CRM_ACC_001",
        "externalCompanyId": "hubspot_12345",
        "domain": "acme.com",
        "accountType": "enterprise",
        "healthScore": 85,
        "arr": 120000,
        "metadata": {
          "tier": "premium",
          "contract_start": "2024-01-01"
        }
      }
    ]
  }'
```

> **Note:** `providerKey` is optional and defaults to `"api"` if not provided.

#### List All Accounts

```bash
curl -X GET 'https://your-api-domain.com/api/v1/accounts' \
  -H 'Authorization: Bearer YOUR_API_TOKEN'
```

#### Get Single Account

```bash
curl -X GET 'https://your-api-domain.com/api/v1/accounts/ACCOUNT_UUID' \
  -H 'Authorization: Bearer YOUR_API_TOKEN'
```

#### Update Single Account

```bash
curl -X PUT 'https://your-api-domain.com/api/v1/accounts/ACCOUNT_UUID' \
  -H 'Authorization: Bearer YOUR_API_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "Acme Enterprise Account Updated",
    "healthScore": 92,
    "arr": 150000
  }'
```

#### Delete Account

```bash
curl -X DELETE 'https://your-api-domain.com/api/v1/accounts/ACCOUNT_UUID' \
  -H 'Authorization: Bearer YOUR_API_TOKEN'
```

---

## Contacts API

Manage contact records for your enterprise.

### Endpoints

| Method | Endpoint | Description | Required Scope |
|--------|----------|-------------|----------------|
| POST | `/api/v1/contacts` | Batch create/update contacts | `contacts:write` |
| GET | `/api/v1/contacts` | List all contacts | `contacts:read` |
| GET | `/api/v1/contacts/:id` | Get single contact (with emails, phones, companies) | `contacts:read` |
| PUT | `/api/v1/contacts/:id` | Update single contact | `contacts:write` |
| DELETE | `/api/v1/contacts/:id` | Delete a contact | `contacts:delete` |
| GET | `/api/v1/contacts/schema` | Get API schema | - |

### Contact Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `externalId` | string | Yes | External unique ID from integration source |
| `fullName` | string \| null | No | Contact's full name |
| `primaryEmail` | string \| null | No | Primary email address |
| `primaryPhone` | string \| null | No | Primary phone number |
| `title` | string \| null | No | Job title |
| `ownerUserId` | UUID \| null | No | Owner user ID |
| `emails` | array | No | List of emails (see Email Object) |
| `phones` | array | No | List of phones (see Phone Object) |
| `companies` | array | No | List of company associations (see Company Association Object) |
| `attributes` | object | No | Custom key-value attributes |
| `properties` | object | No | Custom field values (maps to `field_values` table) |

#### Email Object

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `email` | string | Yes | Email address |
| `isPrimary` | boolean | No | Is primary email (default: `false`) |
| `label` | string | No | Label (e.g., "Work", "Personal") |

#### Phone Object

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `phone` | string | Yes | Phone number |
| `isPrimary` | boolean | No | Is primary phone (default: `false`) |
| `label` | string | No | Label (e.g., "Mobile", "Office") |

#### Company Association Object

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `companyId` | UUID | Yes | Company UUID |
| `relation` | enum | No | `employee`, `consultant`, `agency`, `partner`, `contractor`, `other` (default: `employee`) |
| `isPrimary` | boolean | No | Is primary company (default: `false`) |
| `attributes` | object | No | Relationship attributes |

### cURL Examples

#### Create/Update Contacts (Batch)

```bash
curl -X POST 'https://your-api-domain.com/api/v1/contacts' \
  -H 'Authorization: Bearer YOUR_API_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
    "contacts": [
      {
        "externalId": "contact_12345",
        "fullName": "John Doe",
        "primaryEmail": "john.doe@acme.com",
        "primaryPhone": "+1-555-123-4567",
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

> **Note:** `providerKey` is optional and defaults to `"api"` if not provided.

#### List All Contacts

```bash
curl -X GET 'https://your-api-domain.com/api/v1/contacts' \
  -H 'Authorization: Bearer YOUR_API_TOKEN'
```

#### Get Single Contact

```bash
curl -X GET 'https://your-api-domain.com/api/v1/contacts/CONTACT_UUID' \
  -H 'Authorization: Bearer YOUR_API_TOKEN'
```

#### Update Single Contact

```bash
curl -X PUT 'https://your-api-domain.com/api/v1/contacts/CONTACT_UUID' \
  -H 'Authorization: Bearer YOUR_API_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
    "fullName": "John Doe Jr.",
    "title": "Senior VP of Engineering",
    "phones": [
      { "phone": "+1-555-987-6543", "isPrimary": true, "label": "Mobile" }
    ]
  }'
```

#### Delete Contact

```bash
curl -X DELETE 'https://your-api-domain.com/api/v1/contacts/CONTACT_UUID' \
  -H 'Authorization: Bearer YOUR_API_TOKEN'
```

---

## Associations API

Create contact-company and contact-account relationships. Supports both external IDs (via integration mappings) and direct internal database IDs.

### Endpoints

| Method | Endpoint | Description | Required Scope |
|--------|----------|-------------|----------------|
| POST | `/api/v1/associations` | Batch create/update associations | `associations:write` |
| GET | `/api/v1/associations/schema` | Get API schema | - |

### Association Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `contactId` | UUID | No | Internal UUID of the contact (bypasses mapping lookup; optional if `externalContactId` provided) |
| `externalContactId` | string | No | External ID of the contact (via integration mapping; optional if `contactId` provided) |
| `companyId` | UUID | No | Internal UUID of the company (bypasses mapping lookup; optional if `externalCompanyId` provided) |
| `externalCompanyId` | string | No | External ID of the company (creates `customer_company`; optional if `companyId` provided) |
| `accountId` | UUID | No | Internal UUID of the account (bypasses mapping lookup and auto-resolves company; optional if `externalAccountId` provided) |
| `externalAccountId` | string | No | External ID of the account (creates `contact_account` and auto-resolves company; optional if `accountId` provided) |
| `relation` | enum | No | `employee`, `consultant`, `contractor`, `partner`, `agency`, `other` (default: `employee`) |
| `isPrimary` | boolean | No | Is primary company for contact (default: `false`) |
| `role` | string | No | Role for the contact_account association |

### Validation Rules

1. **Contact Identifier:** Must provide either `contactId` OR `externalContactId`
2. **Company/Account Identifier:** Must provide at least one of:
   - `companyId` or `externalCompanyId` (for company-only associations)
   - `accountId` or `externalAccountId` (for account-based associations)
   - Both (for associations with both company and account)

### Association Flows

| Flow | Required Fields | Creates |
|------|-----------------|---------|
| **Company-only** | `contactId` or `externalContactId` + `companyId` or `externalCompanyId` | `customer_company` |
| **Account-based** | `contactId` or `externalContactId` + `accountId` or `externalAccountId` | `contact_account`, `customer_company` (from account.companyId) |
| **Account + Company** | `contactId` or `externalContactId` + `accountId` or `externalAccountId` + `companyId` or `externalCompanyId` | `contact_account`, `customer_company` (for both companies) |

### cURL Examples

#### Create/Update Associations with External IDs (Batch)

```bash
curl -X POST 'https://your-api-domain.com/api/v1/associations' \
  -H 'Authorization: Bearer YOUR_API_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
    "associations": [
      {
        "externalContactId": "contact_12345",
        "externalCompanyId": "company_67890",
        "relation": "employee",
        "isPrimary": true
      },
      {
        "externalContactId": "contact_12345",
        "externalAccountId": "acc_abc123",
        "role": "decision_maker"
      }
    ]
  }'
```

#### Create/Update Associations with Internal IDs (Batch)

```bash
curl -X POST 'https://your-api-domain.com/api/v1/associations' \
  -H 'Authorization: Bearer YOUR_API_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
    "associations": [
      {
        "contactId": "550e8400-e29b-41d4-a716-446655440000",
        "companyId": "660e8400-e29b-41d4-a716-446655440111",
        "relation": "employee",
        "isPrimary": true
      },
      {
        "contactId": "550e8400-e29b-41d4-a716-446655440000",
        "accountId": "770e8400-e29b-41d4-a716-446655440222",
        "role": "decision_maker"
      }
    ]
  }'
```

#### Mixed ID Types (External ID for contact, Internal ID for company)

```bash
curl -X POST 'https://your-api-domain.com/api/v1/associations' \
  -H 'Authorization: Bearer YOUR_API_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
    "associations": [
      {
        "externalContactId": "contact_12345",
        "companyId": "660e8400-e29b-41d4-a716-446655440111"
      }
    ]
  }'
```

> **Note:** `providerKey` is optional and defaults to `"api"` if not provided. Direct internal IDs (UUID format) bypass integration mapping lookups, making associations faster and more reliable when you have the database IDs available.

---

## Calls API

Create and manage call records.

### Endpoints

| Method | Endpoint | Description | Required Scope |
|--------|----------|-------------|----------------|
| POST | `/api/v1/calls` | Batch create call records | `calls:write` |
| GET | `/api/v1/calls` | List all calls | `calls:read` |
| GET | `/api/v1/calls/:id` | Get single call | `calls:read` |
| GET | `/api/v1/calls/schema` | Get API schema | - |

### Call Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `externalCallId` | string | Yes | External unique ID from call provider |
| `direction` | enum | No | `inbound`, `outbound`, `unknown` (default: `unknown`) |
| `status` | enum | No | `completed`, `missed`, `voicemail`, `failed`, `busy`, `cancelled`, `unknown` (default: `completed`) |
| `fromNumber` | string \| null | No | Phone number the call originated from |
| `toNumber` | string \| null | No | Phone number the call was made to |
| `durationSeconds` | number \| null | No | Call duration in seconds |
| `ringDurationSeconds` | number \| null | No | Ring duration before answer |
| `startTimestamp` | ISO 8601 \| null | No | Call start time |
| `endTimestamp` | ISO 8601 \| null | No | Call end time |
| `recordingUrl` | string \| null | No | URL to call recording |
| `customer` | object | No | Customer info (`phone`, `email`, `name`) |
| `orgUnitIds` | UUID[] | No | Org unit IDs this call belongs to |
| `orgUnitId` | UUID | No | Single org unit ID (alternative to `orgUnitIds`) |
| `metadata` | object | No | Additional metadata |
| `title` | string \| null | No | Call title (auto-generated if not provided) |

### cURL Examples

#### Create Calls (Batch)

```bash
curl -X POST 'https://your-api-domain.com/api/v1/calls' \
  -H 'Authorization: Bearer YOUR_API_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
    "calls": [
      {
        "externalCallId": "call_12345",
        "direction": "inbound",
        "status": "completed",
        "fromNumber": "+1-555-123-4567",
        "toNumber": "+1-555-987-6543",
        "durationSeconds": 300,
        "startTimestamp": "2024-01-15T10:00:00Z",
        "endTimestamp": "2024-01-15T10:05:00Z",
        "recordingUrl": "https://recordings.example.com/call_12345.mp3",
        "customer": {
          "phone": "+1-555-123-4567",
          "name": "John Doe",
          "email": "john.doe@acme.com"
        },
        "title": "Sales Follow-up Call"
      }
    ]
  }'
```

> **Note:** `providerKey` is optional and defaults to `"api"` if not provided.

#### List All Calls

```bash
curl -X GET 'https://your-api-domain.com/api/v1/calls' \
  -H 'Authorization: Bearer YOUR_API_TOKEN'
```

#### Get Single Call

```bash
curl -X GET 'https://your-api-domain.com/api/v1/calls/CALL_UUID' \
  -H 'Authorization: Bearer YOUR_API_TOKEN'
```

---

## Meetings API

Create and manage meeting records.

### Endpoints

| Method | Endpoint | Description | Required Scope |
|--------|----------|-------------|----------------|
| POST | `/api/v1/meetings` | Create/update a meeting | `meetings:write` |
| GET | `/api/v1/meetings` | List all meetings | `meetings:read` |
| GET | `/api/v1/meetings/:id` | Get single meeting | `meetings:read` |
| DELETE | `/api/v1/meetings/:id` | Delete/cancel a meeting | `meetings:delete` |
| GET | `/api/v1/meetings/schema` | Get API schema | - |

### Meeting Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `scheduledStartAt` | ISO 8601 | Yes | Scheduled start time |
| `scheduledEndAt` | ISO 8601 | Yes | Scheduled end time (must be after start and in the future) |
| `title` | string | No | Meeting title/subject (default: `empty title`) |
| `channel` | enum | No | `gmeet`, `zoom`, `teams`, `offline` |
| `joinUrl` | string \| null | No | URL to join the meeting |
| `status` | enum | No | `scheduled`, `skip`, `in_progress`, `cancelled`, `completed`, `deleted`, `error` (default: `scheduled`) |
| `externalMeetingId` | string \| null | No | External unique ID from calendar source (used for upsert) |
| `hostEmail` | string \| null | No | Email of meeting host/organizer |
| `participantEmails` | string[] | No | Array of participant email addresses |
| `orgUnitIds` | UUID[] | No | Org unit IDs this meeting belongs to |
| `orgUnitId` | UUID | No | Single org unit ID (alternative to `orgUnitIds`) |
| `recurringMeetingId` | string \| null | No | Recurring meeting reference ID |
| `type` | enum | No | `external`, `internal` (auto-determined from participant domains) |
| `metadata` | object | No | Additional metadata |

### cURL Examples

#### Create/Update Meeting

```bash
curl -X POST 'https://your-api-domain.com/api/v1/meetings' \
  -H 'Authorization: Bearer YOUR_API_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
    "title": "Weekly Team Sync",
    "channel": "gmeet",
    "joinUrl": "https://meet.google.com/abc-defg-hij",
    "scheduledStartAt": "2024-01-20T10:00:00Z",
    "scheduledEndAt": "2024-01-20T11:00:00Z",
    "status": "scheduled",
    "externalMeetingId": "calendar_event_abc123",
    "hostEmail": "john@example.com",
    "participantEmails": ["john@example.com", "jane@example.com"],
    "type": "internal"
  }'
```

#### List All Meetings

```bash
curl -X GET 'https://your-api-domain.com/api/v1/meetings' \
  -H 'Authorization: Bearer YOUR_API_TOKEN'
```

#### Get Single Meeting

```bash
curl -X GET 'https://your-api-domain.com/api/v1/meetings/MEETING_UUID' \
  -H 'Authorization: Bearer YOUR_API_TOKEN'
```

#### Delete/Cancel Meeting

```bash
curl -X DELETE 'https://your-api-domain.com/api/v1/meetings/MEETING_UUID' \
  -H 'Authorization: Bearer YOUR_API_TOKEN'
```

---

## Fields API

Manage custom field definitions and values for entities (companies, accounts, contacts).

### Endpoints

| Method | Endpoint | Description | Required Scope |
|--------|----------|-------------|----------------|
| GET | `/api/v1/fields/definitions` | List field definitions for entity type | `fields:read` |
| POST | `/api/v1/fields/definitions` | Create field definition | `fields:write` |
| DELETE | `/api/v1/fields/definitions/:id` | Delete field definition | `fields:delete` |
| GET | `/api/v1/fields/values` | Get field values for an entity | `fields:read` |
| POST | `/api/v1/fields/values` | Set field values for an entity | `fields:write` |
| POST | `/api/v1/fields/values/batch` | Set field values for multiple entities | `fields:write` |
| GET | `/api/v1/fields/schema` | Get API schema | - |

### Field Definition Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `fieldKey` | string | Yes | Unique key for the field (snake_case recommended) |
| `fieldName` | string | Yes | Display name for the field |
| `fieldType` | enum | Yes | `text`, `number`, `date`, `boolean`, `json`, `enum` |
| `entityType` | enum | Yes | `company`, `account`, `contact` |
| `enumOptions` | string[] | No | Valid options for enum field type |
| `isRequired` | boolean | No | Is field required (default: `false`) |
| `displayOrder` | integer | No | Order for UI display |

### Field Value Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `fieldKey` | string | Yes | Key of the field definition |
| `value` | any | Yes | Value matching the field type |

### Valid Types

- **Entity Types:** `company`, `account`, `contact`
- **Field Types:** `text`, `number`, `date`, `boolean`, `json`, `enum`

