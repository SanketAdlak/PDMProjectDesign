# API Specification — TrustStack Module 2: AI Data Discovery & NLP Contract Audit

**Version:** v1
**Date:** 2026-04-22
**Base URL:** `https://api.truststack.in/discovery/v1`
**Format:** OpenAPI 3.1 (narrative form — machine-readable YAML in `/openapi/discovery-v1.yaml`)

---

## Authentication

Two authentication schemes are accepted. The tenant is always derived from the credential — it is never accepted in the request body or path.

### 1. Bearer JWT (Human actors — DPO, CTO, Admin)

```
Authorization: Bearer <signed-jwt>
```

JWTs are issued by TrustStack's auth service (AWS Cognito, ap-south-1). The JWT contains `tenant_id` and `role` claims. Supported roles:

| Role | Description |
|------|-------------|
| `DPO` | Data Protection Officer — full read/write access |
| `CTO` | Engineering lead — connector management + scan management |
| `ANALYST` | Read-only access to findings and data flow map |
| `ADMIN` | TrustStack internal — cross-tenant audit access |

### 2. API Key (Service-to-service — Erasure Engine)

```
X-API-Key: ts_disc_<64-char-hex>
```

API keys are scoped to a specific tenant and a specific service permission set. The Erasure Engine uses keys with the `ERASURE_ENGINE` scope, which grants access only to `/snapshot/*` endpoints. No other endpoint is accessible with an `ERASURE_ENGINE`-scoped key.

API key values are stored as SHA-256 hashes — the raw key is shown once on creation and never again (mirrors the API Key Management pattern documented in the Consent Vault).

---

## Rate Limits

| Endpoint Group | Limit | Window | Applies To |
|----------------|-------|--------|------------|
| `POST /scans` | 10 requests | 1 hour | Per tenant |
| `POST /connectors` | 20 requests | 1 hour | Per tenant |
| `POST /connectors/{id}/test` | 5 requests | 10 minutes | Per connector |
| `POST /contracts/audit` | 50 requests | 24 hours | Per tenant |
| `POST /snapshot/request` | 100 requests | 1 hour | Per API key |
| `GET /snapshot/{token}` | 1 request | Per token | Per token (one-time) |
| `GET /findings/export` | 5 requests | 1 hour | Per tenant |
| All other GETs | 300 requests | 1 minute | Per tenant |

Rate limit headers are returned on every response:

```
X-RateLimit-Limit: 300
X-RateLimit-Remaining: 247
X-RateLimit-Reset: 1745318400
```

When the limit is exceeded, a `429 Too Many Requests` response is returned (RFC 9457 format, see Error Responses section).

---

## Common Headers

| Header | Required | Description |
|--------|----------|-------------|
| `Authorization` | Yes (JWT auth) | `Bearer <token>` |
| `X-API-Key` | Yes (service auth) | Service-to-service API key |
| `X-Request-Id` | Recommended | Client-generated UUID for idempotency and tracing |
| `Content-Type` | Yes (POST/PATCH) | `application/json` or `multipart/form-data` |
| `Accept` | No | Defaults to `application/json` |

---

## Error Responses (RFC 9457 Problem Details)

All errors use [RFC 9457 Problem Details](https://www.rfc-editor.org/rfc/rfc9457) format. The `Content-Type` of error responses is `application/problem+json`.

```json
{
  "type": "https://api.truststack.in/problems/connector-auth-failed",
  "title": "Connector Authentication Failed",
  "status": 422,
  "detail": "AWS Secrets Manager could not resolve the credentials ARN for connector 'zoho-crm-prod'. Verify the ARN and IAM permissions.",
  "instance": "/discovery/v1/connectors/abc123/test",
  "traceId": "01HXYZ789ABCDEF012345",
  "timestamp": "2026-04-22T11:45:00Z"
}
```

Standard error types:

| HTTP Status | type slug | Common cause |
|-------------|-----------|--------------|
| 400 | `validation-error` | Request body fails schema validation |
| 401 | `unauthenticated` | Missing or expired JWT/API key |
| 403 | `forbidden` | Valid credential but insufficient scope/role |
| 404 | `not-found` | Resource does not exist or belongs to another tenant |
| 409 | `conflict` | Duplicate resource (same connector vendor type + same credentials ref) |
| 410 | `snapshot-expired-or-consumed` | Snapshot token already used or past 15-min TTL |
| 422 | `connector-auth-failed` | Connector credentials invalid or IAM permission error |
| 429 | `rate-limit-exceeded` | Rate limit hit (includes `Retry-After` header) |
| 500 | `internal-error` | Unexpected server error (includes traceId for support) |
| 503 | `scan-engine-unavailable` | Temporal.io workflow engine is unreachable |

---

## Pagination

List endpoints use cursor-based pagination. All list responses include a `pagination` object:

```json
{
  "data": [...],
  "pagination": {
    "cursor": "eyJjcmVhdGVkX2F0IjoiMjAyNi0wNC0yMlQxMDozMDowMFoiLCJpZCI6ImFiYzEyMyJ9",
    "hasMore": true,
    "pageSize": 50
  }
}
```

Pass the `cursor` as a query parameter (`?cursor=<value>`) to fetch the next page. Cursors are opaque base64-encoded values encoding the last-seen `created_at` and `id`.

---

## Connector Management

### `GET /connectors`

**Description:** List all configured connectors for the authenticated tenant, including current status and last scan time.

**Required role:** `DPO`, `CTO`, `ANALYST`

**Query parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `status` | enum | Filter by `ACTIVE`, `ERROR`, `PAUSED`, `PENDING_AUTH` |
| `vendor_type` | string | Filter by vendor (e.g., `ZOHO_CRM`) |
| `cursor` | string | Pagination cursor |
| `pageSize` | integer | Default 50, max 100 |

**Response 200:**

```json
{
  "data": [
    {
      "connectorId": "c1d2e3f4-1111-2222-3333-444455556666",
      "vendorType": "ZOHO_CRM",
      "status": "ACTIVE",
      "lastScanAt": "2026-04-21T03:00:00Z",
      "scanScope": {
        "include": ["Contacts", "Leads", "Deals"],
        "exclude": []
      },
      "createdBy": "u1a2b3c4-aaaa-bbbb-cccc-ddddeeeeeeee",
      "createdAt": "2026-03-15T09:00:00Z",
      "updatedAt": "2026-04-21T03:45:12Z"
    }
  ],
  "pagination": {
    "cursor": null,
    "hasMore": false,
    "pageSize": 50
  }
}
```

---

### `POST /connectors`

**Description:** Add a new connector. The `credentialsRef` must be an AWS Secrets Manager ARN in the tenant's account. TrustStack reads the ARN at scan time — raw credentials are never sent to or stored by TrustStack.

**Required role:** `DPO`, `CTO`

**Rate limit:** 20/hour per tenant

**Request body:**

```json
{
  "vendorType": "RAZORPAY",
  "credentialsRef": "arn:aws:secretsmanager:ap-south-1:123456789012:secret:razorpay-live-key-Xk9pQ2",
  "scanScope": {
    "include": ["payments", "customers", "subscriptions"],
    "exclude": ["webhooks"]
  }
}
```

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `vendorType` | enum | Yes | One of 13 supported vendor types |
| `credentialsRef` | string | Yes | AWS Secrets Manager ARN — never a raw credential |
| `scanScope` | object | No | Defaults to all available resources |

**Response 201:**

```json
{
  "connectorId": "d4e5f6a7-7777-8888-9999-000011112222",
  "vendorType": "RAZORPAY",
  "status": "PENDING_AUTH",
  "credentialsRef": "arn:aws:secretsmanager:ap-south-1:123456789012:secret:razorpay-live-key-Xk9pQ2",
  "scanScope": {
    "include": ["payments", "customers", "subscriptions"],
    "exclude": ["webhooks"]
  },
  "createdAt": "2026-04-22T11:00:00Z"
}
```

**Error responses:**

| Status | type | Condition |
|--------|------|-----------|
| 400 | `validation-error` | `credentialsRef` is not a valid Secrets Manager ARN format |
| 409 | `conflict` | Connector with same `vendorType` and `credentialsRef` already exists |

---

### `PATCH /connectors/{connectorId}`

**Description:** Update the scan scope of an existing connector or trigger re-authentication (e.g., after rotating credentials in Secrets Manager).

**Required role:** `DPO`, `CTO`

**Path parameters:** `connectorId` (UUID)

**Request body (all fields optional):**

```json
{
  "scanScope": {
    "include": ["Contacts", "Leads"],
    "exclude": ["InternalNotes"]
  },
  "credentialsRef": "arn:aws:secretsmanager:ap-south-1:123456789012:secret:zoho-api-key-v2-Yz7mR3",
  "status": "ACTIVE"
}
```

**Response 200:** Updated connector object (same schema as `POST /connectors` response).

**Error responses:**

| Status | type | Condition |
|--------|------|-----------|
| 404 | `not-found` | `connectorId` does not exist or belongs to another tenant |
| 422 | `connector-auth-failed` | New `credentialsRef` ARN could not be resolved |

---

### `DELETE /connectors/{connectorId}`

**Description:** Remove a connector configuration. Existing `pii_findings` and `data_flow_nodes` that reference this connector are **retained** — deletion of a connector does not erase the evidence of what was discovered. The connector's `connector_id` FK in `data_flow_nodes` is set to NULL (ON DELETE SET NULL).

**Required role:** `DPO`, `CTO`

**Path parameters:** `connectorId` (UUID)

**Response 204:** No body.

**Error responses:**

| Status | type | Condition |
|--------|------|-----------|
| 404 | `not-found` | Connector not found |
| 409 | `conflict` | Connector has a scan currently in `RUNNING` status — cancel scan first |

---

### `POST /connectors/{connectorId}/test`

**Description:** Test connectivity to the vendor system and return a summary of accessible resources. This endpoint **does not** start a scan and **does not** retrieve or store any PII. It returns only counts and structural metadata.

**Required role:** `DPO`, `CTO`

**Rate limit:** 5/10 minutes per connector

**Path parameters:** `connectorId` (UUID)

**Response 200:**

```json
{
  "connectorId": "c1d2e3f4-1111-2222-3333-444455556666",
  "vendorType": "ZOHO_CRM",
  "connectionStatus": "OK",
  "latencyMs": 143,
  "accessibleResources": [
    { "name": "Contacts", "estimatedObjectCount": 48200 },
    { "name": "Leads", "estimatedObjectCount": 12450 },
    { "name": "Deals", "estimatedObjectCount": 3100 }
  ],
  "testedAt": "2026-04-22T11:05:00Z"
}
```

**Error responses:**

| Status | type | Condition |
|--------|------|-----------|
| 422 | `connector-auth-failed` | Credentials invalid, expired, or IAM policy insufficient |

---

## Scan Management

### `POST /scans`

**Description:** Trigger a new scan job. For `FULL_SCAN` and `INCREMENTAL_SCAN`, `connectorIds` is optional — if omitted, all `ACTIVE` connectors are included. For `CONNECTOR_SCAN`, exactly one `connectorId` is required. `DP_TARGETED_SCAN` is reserved for internal use by the Erasure Engine via the Snapshot API.

**Required role:** `DPO`, `CTO`

**Rate limit:** 10/hour per tenant

**Request body:**

```json
{
  "jobType": "INCREMENTAL_SCAN",
  "connectorIds": [
    "c1d2e3f4-1111-2222-3333-444455556666",
    "d4e5f6a7-7777-8888-9999-000011112222"
  ]
}
```

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `jobType` | enum | Yes | `FULL_SCAN` \| `INCREMENTAL_SCAN` \| `CONNECTOR_SCAN` |
| `connectorIds` | UUID[] | No | Empty = all active connectors |

**Response 202 Accepted:**

```json
{
  "jobId": "e5f6a7b8-aaaa-bbbb-cccc-ddddeeeeeeee",
  "jobType": "INCREMENTAL_SCAN",
  "status": "QUEUED",
  "connectorsIncluded": [
    "c1d2e3f4-1111-2222-3333-444455556666",
    "d4e5f6a7-7777-8888-9999-000011112222"
  ],
  "temporalWorkflowId": "discovery-incremental-e5f6a7b8",
  "triggeredBy": "MANUAL",
  "createdAt": "2026-04-22T11:10:00Z"
}
```

**Error responses:**

| Status | type | Condition |
|--------|------|-----------|
| 400 | `validation-error` | `CONNECTOR_SCAN` requested without exactly one `connectorId` |
| 409 | `conflict` | A `FULL_SCAN` or `INCREMENTAL_SCAN` is already `RUNNING` for this tenant |
| 503 | `scan-engine-unavailable` | Temporal.io workflow engine unreachable |

---

### `GET /scans`

**Description:** List scan jobs for the tenant, newest first.

**Required role:** `DPO`, `CTO`, `ANALYST`

**Query parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `status` | enum | Filter by job status |
| `jobType` | enum | Filter by job type |
| `connectorId` | UUID | Filter scans that included this connector |
| `from` | ISO 8601 | Filter scans started after this timestamp |
| `to` | ISO 8601 | Filter scans started before this timestamp |
| `cursor` | string | Pagination cursor |
| `pageSize` | integer | Default 20, max 100 |

**Response 200:**

```json
{
  "data": [
    {
      "jobId": "e5f6a7b8-aaaa-bbbb-cccc-ddddeeeeeeee",
      "jobType": "INCREMENTAL_SCAN",
      "status": "COMPLETED",
      "connectorsIncluded": ["c1d2e3f4-1111-2222-3333-444455556666"],
      "findingsCount": 342,
      "triggeredBy": "MANUAL",
      "startedAt": "2026-04-22T11:10:05Z",
      "completedAt": "2026-04-22T11:23:41Z",
      "createdAt": "2026-04-22T11:10:00Z"
    }
  ],
  "pagination": {
    "cursor": null,
    "hasMore": false,
    "pageSize": 20
  }
}
```

---

### `GET /scans/{jobId}`

**Description:** Get real-time status and progress of a scan job.

**Required role:** `DPO`, `CTO`, `ANALYST`

**Path parameters:** `jobId` (UUID)

**Response 200:**

```json
{
  "jobId": "e5f6a7b8-aaaa-bbbb-cccc-ddddeeeeeeee",
  "jobType": "INCREMENTAL_SCAN",
  "status": "RUNNING",
  "connectorsIncluded": [
    "c1d2e3f4-1111-2222-3333-444455556666",
    "d4e5f6a7-7777-8888-9999-000011112222"
  ],
  "progress": {
    "connectorsCompleted": 1,
    "connectorsTotal": 2,
    "currentConnector": "RAZORPAY",
    "findingsSoFar": 189
  },
  "temporalWorkflowId": "discovery-incremental-e5f6a7b8",
  "triggeredBy": "MANUAL",
  "startedAt": "2026-04-22T11:10:05Z",
  "completedAt": null,
  "createdAt": "2026-04-22T11:10:00Z"
}
```

---

### `POST /scans/{jobId}/cancel`

**Description:** Request cancellation of a `QUEUED` or `RUNNING` scan. Partial results already written to `pii_findings` are retained. Cancellation is propagated to the Temporal.io workflow via a signal.

**Required role:** `DPO`, `CTO`

**Path parameters:** `jobId` (UUID)

**Response 202:**

```json
{
  "jobId": "e5f6a7b8-aaaa-bbbb-cccc-ddddeeeeeeee",
  "status": "CANCELLED",
  "message": "Cancellation signal sent to Temporal workflow. Partial results retained."
}
```

**Error responses:**

| Status | type | Condition |
|--------|------|-----------|
| 409 | `conflict` | Job is already `COMPLETED`, `FAILED`, or `CANCELLED` |

---

### `GET /scans/{jobId}/summary`

**Description:** Aggregate findings summary for a completed scan. Returns counts and breakdowns — no raw PII, no value hashes.

**Required role:** `DPO`, `CTO`, `ANALYST`

**Path parameters:** `jobId` (UUID)

**Response 200:**

```json
{
  "jobId": "e5f6a7b8-aaaa-bbbb-cccc-ddddeeeeeeee",
  "status": "COMPLETED",
  "startedAt": "2026-04-22T11:10:05Z",
  "completedAt": "2026-04-22T11:23:41Z",
  "totalFindings": 342,
  "byRiskLevel": {
    "CRITICAL": 12,
    "HIGH": 87,
    "MEDIUM": 194,
    "LOW": 49
  },
  "byDpdpaCategory": {
    "FINANCIAL": 103,
    "PERSONAL_IDENTITY": 89,
    "CONTACT": 77,
    "BEHAVIORAL": 43,
    "TECHNICAL": 30
  },
  "byConnector": [
    { "connectorId": "c1d2e3f4-1111-2222-3333-444455556666", "vendorType": "ZOHO_CRM", "findingsCount": 201 },
    { "connectorId": "d4e5f6a7-7777-8888-9999-000011112222", "vendorType": "RAZORPAY", "findingsCount": 141 }
  ],
  "newFindings": 58,
  "resolvedSinceLastScan": 14,
  "minorDataFindingsCount": 3
}
```

**Error responses:**

| Status | type | Condition |
|--------|------|-----------|
| 409 | `conflict` | Job is not yet in `COMPLETED` status |

---

## PII Findings

### `GET /findings`

**Description:** List PII findings for the tenant. Returns structural metadata and classification — never raw PII values or even hashes (hashes are stored for deduplication but not exposed in list responses).

**Required role:** `DPO`, `CTO`, `ANALYST`

**Query parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `connectorId` | UUID | Filter by connector |
| `dpdpaCategory` | enum | Filter by DPDPA category |
| `riskLevel` | enum | `CRITICAL` \| `HIGH` \| `MEDIUM` \| `LOW` |
| `status` | enum | `OPEN` \| `ACKNOWLEDGED` \| `SUPPRESSED` \| `REMEDIATED` |
| `entityType` | enum | e.g., `AADHAAR_NUMBER`, `CREDIT_CARD` |
| `from` | ISO 8601 | Filter by `found_at` after this date |
| `to` | ISO 8601 | Filter by `found_at` before this date |
| `cursor` | string | Pagination cursor |
| `pageSize` | integer | Default 50, max 200 |

**Response 200:**

```json
{
  "data": [
    {
      "findingId": "f1a2b3c4-1234-5678-9012-abcdef012345",
      "jobId": "e5f6a7b8-aaaa-bbbb-cccc-ddddeeeeeeee",
      "connectorId": "c1d2e3f4-1111-2222-3333-444455556666",
      "entityType": "AADHAAR_NUMBER",
      "dpdpaCategory": "PERSONAL_IDENTITY",
      "sourceSystem": "Zoho CRM — Production",
      "sourceLocation": {
        "table": "Contacts",
        "field": "custom_aadhaar_field",
        "estimatedRecordCount": 1420
      },
      "confidence": 0.97,
      "riskLevel": "CRITICAL",
      "status": "OPEN",
      "regulatoryConflict": null,
      "foundAt": "2026-04-22T11:22:07Z"
    }
  ],
  "pagination": {
    "cursor": "eyJmb3VuZF9hdCI6IjIwMjYtMDQtMjJUMTE6MjI6MDdaIiwiaWQiOiJmMWEyYjNjNC0xMjM0LTU2NzgtOTAxMi1hYmNkZWYwMTIzNDUifQ==",
    "hasMore": true,
    "pageSize": 50
  }
}
```

---

### `GET /findings/{findingId}`

**Description:** Get full details of a single finding, including the `value_hash` (SHA-256 only — never the raw value).

**Required role:** `DPO`, `CTO`, `ANALYST`

**Path parameters:** `findingId` (UUID)

**Response 200:**

```json
{
  "findingId": "f1a2b3c4-1234-5678-9012-abcdef012345",
  "jobId": "e5f6a7b8-aaaa-bbbb-cccc-ddddeeeeeeee",
  "connectorId": "c1d2e3f4-1111-2222-3333-444455556666",
  "entityType": "AADHAAR_NUMBER",
  "dpdpaCategory": "PERSONAL_IDENTITY",
  "valueHash": "a665a45920422f9d417e4867efdc4fb8a04a1f3fff1fa07e998e86f7f7a27ae3",
  "sourceSystem": "Zoho CRM — Production",
  "sourceLocation": {
    "table": "Contacts",
    "field": "custom_aadhaar_field",
    "estimatedRecordCount": 1420
  },
  "confidence": 0.97,
  "riskLevel": "CRITICAL",
  "status": "OPEN",
  "acknowledgedBy": null,
  "acknowledgedAt": null,
  "suppressionReason": null,
  "regulatoryConflict": null,
  "foundAt": "2026-04-22T11:22:07Z"
}
```

---

### `POST /findings/{findingId}/acknowledge`

**Description:** Mark a finding as acknowledged. This signals that the DPO has reviewed the finding and is aware of it. It does not suppress or remediate the finding — those are separate actions.

**Required role:** `DPO`

**Path parameters:** `findingId` (UUID)

**Request body:**

```json
{
  "reason": "Confirmed — Aadhaar numbers captured via KYC flow in 2024. Mapping to consent purpose 'kyc_verification' is in progress."
}
```

**Response 200:**

```json
{
  "findingId": "f1a2b3c4-1234-5678-9012-abcdef012345",
  "status": "ACKNOWLEDGED",
  "acknowledgedBy": "u1a2b3c4-aaaa-bbbb-cccc-ddddeeeeeeee",
  "acknowledgedAt": "2026-04-22T11:45:00Z"
}
```

---

### `POST /findings/{findingId}/suppress`

**Description:** Suppress a finding with a documented regulatory justification. Used when another Indian law requires retention of this data beyond the DPDPA erasure window (e.g., RBI 5-year record-keeping rule conflicts with DPDPA consent withdrawal).

DPBI auditors can review `regulatoryConflict` entries to understand why specific data was retained past the DPDPA erasure window. The `suppressionReason` field is a mandatory plain-language explanation.

**Required role:** `DPO`

**Path parameters:** `findingId` (UUID)

**Request body:**

```json
{
  "regulatoryConflict": "RBI_5YR",
  "suppressionReason": "Payment transaction records containing PAN must be retained for 5 years per RBI Master Direction on KYC, 2016. DPDPA erasure cannot be executed on this field until RBI retention period lapses."
}
```

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `regulatoryConflict` | enum | Yes | `RBI_5YR` \| `GST_6YR` \| `SEBI_5YR` \| `CERT_IN_LOGS` |
| `suppressionReason` | string | Yes | Plain-language explanation for DPBI audit |

**Response 200:**

```json
{
  "findingId": "f1a2b3c4-1234-5678-9012-abcdef012345",
  "status": "SUPPRESSED",
  "regulatoryConflict": "RBI_5YR",
  "suppressionReason": "Payment transaction records containing PAN must be retained for 5 years per RBI Master Direction on KYC, 2016. DPDPA erasure cannot be executed on this field until RBI retention period lapses.",
  "suppressedBy": "u1a2b3c4-aaaa-bbbb-cccc-ddddeeeeeeee",
  "suppressedAt": "2026-04-22T11:50:00Z"
}
```

---

### `GET /findings/export`

**Description:** Export findings as a CSV file. No raw PII is included — values are represented by their SHA-256 hashes and structural metadata only. The export is generated asynchronously for large datasets; the response includes a signed S3 pre-signed URL valid for 15 minutes.

**Required role:** `DPO`

**Rate limit:** 5/hour per tenant

**Query parameters:** Same filters as `GET /findings`.

**Response 200:**

```json
{
  "exportId": "exp-a1b2c3d4",
  "downloadUrl": "https://ts-exports-ap-south-1.s3.amazonaws.com/tenant-abc/findings-export-2026-04-22.csv?X-Amz-Expires=900&...",
  "expiresAt": "2026-04-22T12:10:00Z",
  "rowCount": 342,
  "generatedAt": "2026-04-22T11:55:00Z",
  "note": "Raw PII values are not included. value_hash column contains SHA-256 hashes only."
}
```

**CSV columns:**
`finding_id`, `entity_type`, `dpdpa_category`, `risk_level`, `status`, `source_system`, `source_table`, `source_field`, `estimated_record_count`, `confidence`, `regulatory_conflict`, `found_at`, `value_hash`

---

## Data Flow Map

### `GET /data-flow-map`

**Description:** Get the full Data Flow Map — all nodes and edges — for the tenant. This is the complete picture of where personal data flows across all connected and manually-added systems.

**Required role:** `DPO`, `CTO`, `ANALYST`

**Response 200:**

```json
{
  "tenantId": "550e8400-e29b-41d4-a716-446655440000",
  "generatedAt": "2026-04-22T11:23:41Z",
  "nodes": [
    {
      "nodeId": "n1a2b3c4-1111-2222-3333-444455556666",
      "systemName": "Zoho CRM — Production",
      "systemType": "VENDOR_CRM",
      "connectorId": "c1d2e3f4-1111-2222-3333-444455556666",
      "riskScore": 0.82,
      "metadata": {
        "region": "ap-south-1",
        "cloudProvider": "AWS",
        "crossBorderFlag": false
      },
      "lastUpdatedAt": "2026-04-22T11:23:41Z"
    },
    {
      "nodeId": "n2b3c4d5-2222-3333-4444-555566667777",
      "systemName": "Razorpay",
      "systemType": "VENDOR_PAYMENT",
      "connectorId": "d4e5f6a7-7777-8888-9999-000011112222",
      "riskScore": 0.91,
      "metadata": {
        "region": "ap-south-1",
        "cloudProvider": "AWS",
        "crossBorderFlag": false
      },
      "lastUpdatedAt": "2026-04-22T11:23:41Z"
    }
  ],
  "edges": [
    {
      "edgeId": "e1a2b3c4-aaaa-bbbb-cccc-ddddeeee0001",
      "sourceNodeId": "n1a2b3c4-1111-2222-3333-444455556666",
      "destinationNodeId": "n2b3c4d5-2222-3333-4444-555566667777",
      "dataCategories": ["FINANCIAL", "PERSONAL_IDENTITY"],
      "flowType": "INTENTIONAL",
      "purposes": ["payment_processing"],
      "estimatedRecordsPerDay": 1200,
      "crossBorder": false,
      "riskScore": 0.75,
      "lastDataSeenAt": "2026-04-22T06:00:00Z"
    }
  ],
  "summary": {
    "totalNodes": 8,
    "totalEdges": 14,
    "crossBorderEdges": 0,
    "unintentionalLeakEdges": 2,
    "overallRiskScore": 0.74
  }
}
```

---

### `GET /data-flow-map/nodes`

**Description:** List all system nodes. Supports filtering and pagination.

**Required role:** `DPO`, `CTO`, `ANALYST`

**Query parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `systemType` | enum | Filter by system type |
| `crossBorder` | boolean | Filter cross-border systems only |
| `riskScoreGte` | decimal | Minimum risk score (e.g., `0.7`) |
| `cursor` | string | Pagination cursor |
| `pageSize` | integer | Default 50, max 200 |

**Response 200:** `{ "data": [<node objects>], "pagination": {...} }`

---

### `GET /data-flow-map/edges`

**Description:** List all data flow edges. Useful for identifying unintentional leaks or cross-border flows.

**Required role:** `DPO`, `CTO`, `ANALYST`

**Query parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `flowType` | enum | e.g., `UNINTENTIONAL_LEAK` |
| `crossBorder` | boolean | Filter cross-border flows (Section 16) |
| `dataCategory` | enum | Filter edges carrying a specific DPDPA category |
| `sourceNodeId` | UUID | Edges from a specific node |
| `destinationNodeId` | UUID | Edges into a specific node |
| `cursor` | string | Pagination cursor |

**Response 200:** `{ "data": [<edge objects>], "pagination": {...} }`

---

### `GET /data-flow-map/risk-summary`

**Description:** Risk breakdown by system and data category. Designed for the DPO executive dashboard.

**Required role:** `DPO`, `CTO`, `ANALYST`

**Response 200:**

```json
{
  "generatedAt": "2026-04-22T11:23:41Z",
  "bySystem": [
    {
      "nodeId": "n2b3c4d5-2222-3333-4444-555566667777",
      "systemName": "Razorpay",
      "riskScore": 0.91,
      "openFindingsCount": 103,
      "criticalFindingsCount": 8,
      "dataCategories": ["FINANCIAL", "PERSONAL_IDENTITY"]
    }
  ],
  "byDpdpaCategory": [
    { "category": "FINANCIAL", "totalFindings": 103, "criticalFindings": 8, "affectedSystems": 2 },
    { "category": "PERSONAL_IDENTITY", "totalFindings": 89, "criticalFindings": 4, "affectedSystems": 3 }
  ],
  "crossBorderSystems": [],
  "unintentionalLeaks": [
    {
      "edgeId": "e1a2b3c4-aaaa-bbbb-cccc-ddddeeee0009",
      "sourceSystem": "Zoho CRM — Production",
      "destinationSystem": "AWS CloudWatch Logs",
      "dataCategories": ["CONTACT", "PERSONAL_IDENTITY"],
      "riskScore": 0.88
    }
  ]
}
```

---

### `GET /data-flow-map/export`

**Description:** Export the full Data Flow Map as a JSON file, compatible with D3.js, Graphviz, and Mermaid diagram tools. The export includes nodes and edges but never PII values.

**Required role:** `DPO`, `CTO`

**Response 200:**

```json
{
  "exportId": "dfm-exp-b2c3d4e5",
  "downloadUrl": "https://ts-exports-ap-south-1.s3.amazonaws.com/tenant-abc/dfm-export-2026-04-22.json?X-Amz-Expires=900&...",
  "expiresAt": "2026-04-22T12:15:00Z",
  "format": "json",
  "nodeCount": 8,
  "edgeCount": 14,
  "generatedAt": "2026-04-22T11:58:00Z"
}
```

---

## Snapshot API

The Snapshot API is the exclusive cross-module boundary between Module 2 (Discovery) and Module 3 (Erasure Engine). It allows the Erasure Engine to retrieve a current Data Flow Map for a specific Data Principal before initiating deletion.

**Why this design (Section 2(g) dual-role prohibition):**
TrustStack cannot act as both Consent Manager and Data Processor for the same Data Principal. The Snapshot API ensures the Discovery module only *reports* data locations outward to the Erasure Engine — it never executes deletion itself, and it never gives the Erasure Engine access to raw consent records. The one-time token (15-minute TTL) ensures that each erasure event is tied to a single, fresh, auditable snapshot. Replaying a token to get a stale data map is architecturally impossible.

**Authentication:** `X-API-Key` with `ERASURE_ENGINE` scope. JWT-authenticated users cannot call these endpoints. An `ERASURE_ENGINE`-scoped key cannot call any other endpoint in this API.

---

### `POST /snapshot/request`

**Description:** Request a one-time snapshot token for a specific Data Principal. The Erasure Engine calls this before initiating a deletion workflow. The response contains a signed JWT token that the Erasure Engine will use to consume the snapshot.

**Authentication:** `X-API-Key` with `ERASURE_ENGINE` scope only

**Rate limit:** 100/hour per API key

**Request body:**

```json
{
  "dpIdentifierHash": "a665a45920422f9d417e4867efdc4fb8a04a1f3fff1fa07e998e86f7f7a27ae3",
  "justification": "ERASURE_REQUEST"
}
```

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `dpIdentifierHash` | string | Yes | SHA-256 of the DP's identifier. **Raw identifier must never be sent.** |
| `justification` | enum | Yes | `ERASURE_REQUEST` \| `CONSENT_WITHDRAWN` \| `DP_REQUEST` |

**Response 201:**

```json
{
  "snapshotToken": "eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCJ9.eyJ0b2tlbklkIjoiZ2gxMjM0NTYtNzg5MC1hYmNkLWVmMTItMzQ1Njc4OTBhYmNkIiwidGVuYW50SWQiOiI1NTBlODQwMC1lMjliLTQxZDQtYTcxNi00NDY2NTU0NDAwMDAiLCJleHBpcmVzQXQiOiIyMDI2LTA0LTIyVDEyOjI1OjAwWiJ9.MEYCIQDmF3J...",
  "expiresAt": "2026-04-22T12:10:00Z",
  "systemCount": 5,
  "createdAt": "2026-04-22T11:55:00Z"
}
```

| Field | Description |
|-------|-------------|
| `snapshotToken` | Signed JWT (ECDSA P-256). One-time use — will return 410 after consumption or expiry. |
| `expiresAt` | 15 minutes from `createdAt`. After this, the token cannot be consumed. |
| `systemCount` | Number of `data_flow_nodes` nodes in the snapshot (not the systems' names — no PII). |

**Error responses:**

| Status | type | Condition |
|--------|------|-----------|
| 400 | `validation-error` | `dpIdentifierHash` is not a valid SHA-256 hex string (64 chars) |
| 403 | `forbidden` | API key lacks `ERASURE_ENGINE` scope |
| 404 | `not-found` | No data flow map entries found for this DP hash in this tenant |

**Audit:** Every call to this endpoint is written to `discovery_audit_log` with `action = 'SNAPSHOT_REQUESTED'`, `actor_type = 'ERASURE_ENGINE'`, and the `justification` in `metadata`. The `dp_identifier_hash` is **not** logged — only the `token_id` and `systemCount` are recorded, preserving Zero-Knowledge properties in the audit trail.

---

### `GET /snapshot/{token}`

**Description:** Consume a snapshot token and retrieve the Data Flow Map for the specific Data Principal. This endpoint can be called **exactly once** — on the second call (or after the 15-minute TTL), it returns `410 Gone`.

The response contains node metadata (system names, types, risk scores, data categories) but never raw PII. The Erasure Engine uses this to know which vendor systems to target for deletion.

**Authentication:** `X-API-Key` with `ERASURE_ENGINE` scope only

**Rate limit:** 1 per token (enforced at the database level — see `snapshot_tokens` trigger in DATA_MODEL.md)

**Path parameters:** `token` — the full signed JWT value from `POST /snapshot/request`

**Response 200:**

```json
{
  "snapshotId": "gh123456-7890-abcd-ef12-34567890abcd",
  "tenantId": "550e8400-e29b-41d4-a716-446655440000",
  "dpIdentifierHash": "a665a45920422f9d417e4867efdc4fb8a04a1f3fff1fa07e998e86f7f7a27ae3",
  "justification": "ERASURE_REQUEST",
  "snapshotGeneratedAt": "2026-04-22T11:55:00Z",
  "systems": [
    {
      "nodeId": "n1a2b3c4-1111-2222-3333-444455556666",
      "systemName": "Zoho CRM — Production",
      "systemType": "VENDOR_CRM",
      "vendorType": "ZOHO_CRM",
      "connectorId": "c1d2e3f4-1111-2222-3333-444455556666",
      "dataCategories": ["PERSONAL_IDENTITY", "CONTACT"],
      "riskScore": 0.82,
      "crossBorder": false,
      "metadata": {
        "region": "ap-south-1",
        "cloudProvider": "AWS"
      }
    },
    {
      "nodeId": "n2b3c4d5-2222-3333-4444-555566667777",
      "systemName": "Razorpay",
      "systemType": "VENDOR_PAYMENT",
      "vendorType": "RAZORPAY",
      "connectorId": "d4e5f6a7-7777-8888-9999-000011112222",
      "dataCategories": ["FINANCIAL", "PERSONAL_IDENTITY"],
      "riskScore": 0.91,
      "crossBorder": false,
      "regulatoryConflicts": ["RBI_5YR"],
      "metadata": {
        "region": "ap-south-1",
        "cloudProvider": "AWS"
      }
    }
  ],
  "consumedAt": "2026-04-22T11:55:03Z"
}
```

**Note on `regulatoryConflicts`:** When a system has findings for this DP that are `SUPPRESSED` due to a regulatory conflict, the conflict type is included in the snapshot so the Erasure Engine knows erasure from that system must be skipped (or partial) with the appropriate regulatory exception documented.

**Response 410 Gone (token already consumed or expired):**

```json
{
  "type": "https://api.truststack.in/problems/snapshot-expired-or-consumed",
  "title": "Snapshot Token Expired or Already Consumed",
  "status": 410,
  "detail": "This snapshot token has already been consumed or has passed its 15-minute expiry window. Request a new token via POST /snapshot/request.",
  "instance": "/discovery/v1/snapshot/eyJ...",
  "traceId": "01HXYZ789ABCDEF012345",
  "timestamp": "2026-04-22T12:30:00Z"
}
```

**Audit:** Consumption is written to `discovery_audit_log` with `action = 'SNAPSHOT_CONSUMED'`. The `consumed_by` field on the `snapshot_tokens` row is set to the Erasure Engine's service identifier. The database trigger (see DATA_MODEL.md, `snapshot_tokens`) prevents the `consumed` flag from ever being reset to `false`.

---

## Contract Audit

### `POST /contracts/audit`

**Description:** Upload a vendor contract for NLP-based DPDPA compliance audit. Accepts PDF or DOCX files in any of the 22 Eighth Schedule languages. The contract is stored encrypted in S3 (AES-256); the audit runs asynchronously. The response returns an `auditId` to poll for results.

**Required role:** `DPO`, `CTO`

**Rate limit:** 50/24 hours per tenant

**Request:** `multipart/form-data`

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `file` | binary | Yes | PDF or DOCX. Max 50MB. |
| `vendorName` | string | Yes | Human-readable vendor name |
| `languageHint` | string | No | ISO 639-1/3 code hint (e.g., `hi`, `ta`). Auto-detected if omitted. |

**Response 202 Accepted:**

```json
{
  "auditId": "a1b2c3d4-1111-2222-3333-444455556666",
  "vendorName": "Zoho Corporation India Pvt Ltd",
  "contractFilename": "zoho-dpa-2026.pdf",
  "contractHash": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
  "status": "PROCESSING",
  "detectedLanguage": null,
  "fileSizeBytes": 284672,
  "createdAt": "2026-04-22T12:00:00Z",
  "estimatedCompletionSeconds": 45
}
```

**Error responses:**

| Status | type | Condition |
|--------|------|-----------|
| 400 | `validation-error` | Unsupported file format or file exceeds 50MB |
| 409 | `conflict` | Contract with the same SHA-256 hash already audited (returns existing `auditId`) |

---

### `GET /contracts`

**Description:** List all contract audits for the tenant, newest first.

**Required role:** `DPO`, `CTO`, `ANALYST`

**Query parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `status` | enum | Filter by audit status |
| `vendorName` | string | Filter by vendor name (substring match) |
| `dpaGenerated` | boolean | Filter contracts with/without generated DPA |
| `cursor` | string | Pagination cursor |
| `pageSize` | integer | Default 20, max 100 |

**Response 200:**

```json
{
  "data": [
    {
      "auditId": "a1b2c3d4-1111-2222-3333-444455556666",
      "vendorName": "Zoho Corporation India Pvt Ltd",
      "contractFilename": "zoho-dpa-2026.pdf",
      "detectedLanguage": "en",
      "status": "COMPLETED",
      "overallComplianceScore": 61.5,
      "criticalFlagsCount": 2,
      "highFlagsCount": 5,
      "dpaGenerated": false,
      "completedAt": "2026-04-22T12:00:47Z",
      "createdAt": "2026-04-22T12:00:00Z"
    }
  ],
  "pagination": { "cursor": null, "hasMore": false, "pageSize": 20 }
}
```

---

### `GET /contracts/{auditId}`

**Description:** Get the full audit report including all extracted clauses and their compliance flags.

**Required role:** `DPO`, `CTO`, `ANALYST`

**Path parameters:** `auditId` (UUID)

**Response 200:**

```json
{
  "auditId": "a1b2c3d4-1111-2222-3333-444455556666",
  "vendorName": "Zoho Corporation India Pvt Ltd",
  "contractFilename": "zoho-dpa-2026.pdf",
  "detectedLanguage": "en",
  "pageCount": 12,
  "fileSizeBytes": 284672,
  "status": "COMPLETED",
  "overallComplianceScore": 61.5,
  "criticalFlagsCount": 2,
  "highFlagsCount": 5,
  "dpaGenerated": false,
  "ocrUsed": false,
  "startedAt": "2026-04-22T12:00:00Z",
  "completedAt": "2026-04-22T12:00:47Z",
  "clauses": [
    {
      "clauseId": "cl001-aaaa-bbbb-cccc-ddddeeeeeeee",
      "clauseType": "BREACH_NOTIFICATION",
      "originalText": "Vendor shall notify the Company of any data breach within 30 business days.",
      "translatedText": null,
      "pageNumber": 7,
      "complianceStatus": "NON_COMPLIANT",
      "complianceFlags": [
        {
          "flagId": "fl001-aaaa-bbbb-cccc-ddddeeeeeeee",
          "severity": "CRITICAL",
          "dpdpaSection": "Rule 7, DPDP Rules 2025",
          "description": "Breach notification timeline of '30 business days' exceeds DPDPA Rule 7 requirement of 72 hours for DPBI and immediate notification to Data Principals.",
          "remediation": "Replace '30 business days' with '72 hours for regulatory notification and immediate notification to affected Data Principals.'"
        }
      ]
    },
    {
      "clauseId": "cl002-aaaa-bbbb-cccc-ddddeeeeffff",
      "clauseType": "RETENTION",
      "originalText": "Personal data shall be retained for 5 years from the date of collection.",
      "translatedText": null,
      "pageNumber": 4,
      "complianceStatus": "PARTIAL",
      "complianceFlags": [
        {
          "flagId": "fl002-aaaa-bbbb-cccc-ddddeeeeeeee",
          "severity": "HIGH",
          "dpdpaSection": "Section 8(7), DPDPA 2023 and Third Schedule, DPDP Rules 2025",
          "description": "Fixed 5-year retention does not align with DPDPA class-specific timelines. For non-Significant Data Fiduciaries, retention must end when purpose is fulfilled or consent is withdrawn — not on a fixed timeline.",
          "remediation": "Add purpose-linked retention clause: 'Data shall be retained until the purpose for which it was collected is fulfilled or consent is withdrawn, whichever is earlier, unless a specific retention period is mandated by law.'"
        }
      ]
    },
    {
      "clauseId": "cl003-aaaa-bbbb-cccc-ddddeeee0003",
      "clauseType": "DP_RIGHTS",
      "originalText": null,
      "translatedText": null,
      "pageNumber": null,
      "complianceStatus": "MISSING",
      "complianceFlags": [
        {
          "flagId": "fl003-aaaa-bbbb-cccc-ddddeeeeeeee",
          "severity": "CRITICAL",
          "dpdpaSection": "Section 12-14, DPDPA 2023",
          "description": "Contract contains no clause addressing Data Principal rights (right to access, right to correction, right to erasure, right to grievance redressal). This is a mandatory omission under DPDPA.",
          "remediation": "Add a Data Principal Rights clause specifying that the vendor will support the Data Fiduciary in responding to Data Principal access, correction, and erasure requests within 30 days."
        }
      ]
    }
  ]
}
```

---

### `GET /contracts/{auditId}/summary`

**Description:** Executive summary of a contract audit — overall score and critical issue count only. Designed for the management dashboard.

**Required role:** `DPO`, `CTO`, `ANALYST`

**Path parameters:** `auditId` (UUID)

**Response 200:**

```json
{
  "auditId": "a1b2c3d4-1111-2222-3333-444455556666",
  "vendorName": "Zoho Corporation India Pvt Ltd",
  "overallComplianceScore": 61.5,
  "verdict": "NON_COMPLIANT",
  "criticalFlagsCount": 2,
  "highFlagsCount": 5,
  "mediumFlagsCount": 3,
  "missingClauses": ["DP_RIGHTS"],
  "dpaGenerationAvailable": true,
  "completedAt": "2026-04-22T12:00:47Z"
}
```

---

### `GET /contracts/{auditId}/clauses`

**Description:** Paginated list of extracted clauses with compliance status. Supports filtering.

**Required role:** `DPO`, `CTO`, `ANALYST`

**Path parameters:** `auditId` (UUID)

**Query parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `complianceStatus` | enum | Filter by `COMPLIANT`, `NON_COMPLIANT`, `MISSING`, `PARTIAL`, `NEEDS_REVIEW` |
| `clauseType` | enum | Filter by clause type |
| `severity` | string | Filter clauses with at least one flag of this severity (`CRITICAL`, `HIGH`, etc.) |
| `cursor` | string | Pagination cursor |
| `pageSize` | integer | Default 20, max 50 |

**Response 200:** `{ "data": [<clause objects>], "pagination": {...} }`

---

### `POST /contracts/{auditId}/dpa/generate`

**Description:** Generate a compliant Data Processing Agreement from the audit results. The DPA is generated in the same language as the contract's `detectedLanguage` (Rule 3: notice in user's language). The generated DPA addresses all identified non-compliant and missing clauses. Generation is asynchronous — poll `GET /contracts/{auditId}/dpa` for the result.

**Required role:** `DPO`

**Path parameters:** `auditId` (UUID)

**Request body:**

```json
{
  "templateVersion": "v2.1",
  "customClauses": [
    {
      "clauseType": "LIABILITY",
      "customText": "Vendor liability for data breaches is capped at ₹50,00,000 per incident."
    }
  ]
}
```

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `templateVersion` | string | No | Defaults to latest stable DPA template |
| `customClauses` | array | No | Optional custom clause text to insert verbatim |

**Response 202:**

```json
{
  "dpaId": "dpa-d1e2f3a4-1111-2222-3333-444455556666",
  "auditId": "a1b2c3d4-1111-2222-3333-444455556666",
  "status": "GENERATING",
  "generatedLanguage": "en",
  "templateVersion": "v2.1",
  "estimatedCompletionSeconds": 20,
  "createdAt": "2026-04-22T12:05:00Z"
}
```

**Error responses:**

| Status | type | Condition |
|--------|------|-----------|
| 409 | `conflict` | DPA already generated for this audit — use `GET /contracts/{auditId}/dpa` to download |
| 422 | `validation-error` | Audit status is not `COMPLETED` |

---

### `GET /contracts/{auditId}/dpa`

**Description:** Download the generated DPA. Returns pre-signed S3 URLs for both DOCX and signed PDF versions. The PDF is ECDSA-signed with AWS KMS — the signing key ID is included so the receiver can verify the signature independently.

**Required role:** `DPO`, `CTO`

**Path parameters:** `auditId` (UUID)

**Response 200:**

```json
{
  "dpaId": "dpa-d1e2f3a4-1111-2222-3333-444455556666",
  "auditId": "a1b2c3d4-1111-2222-3333-444455556666",
  "vendorName": "Zoho Corporation India Pvt Ltd",
  "generatedLanguage": "en",
  "templateVersion": "v2.1",
  "downloads": {
    "docx": {
      "url": "https://ts-dpa-ap-south-1.s3.amazonaws.com/tenant-abc/dpa-d1e2f3a4.docx?X-Amz-Expires=900&...",
      "expiresAt": "2026-04-22T12:20:00Z"
    },
    "pdf": {
      "url": "https://ts-dpa-ap-south-1.s3.amazonaws.com/tenant-abc/dpa-d1e2f3a4-signed.pdf?X-Amz-Expires=900&...",
      "expiresAt": "2026-04-22T12:20:00Z"
    }
  },
  "signature": {
    "algorithm": "ECDSA_SHA_256",
    "signingKeyId": "arn:aws:kms:ap-south-1:123456789012:key/abc-def-ghi",
    "ecdsaSignature": "MEYCIQDmF3J4KLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789..."
  },
  "createdAt": "2026-04-22T12:05:21Z"
}
```

**Error responses:**

| Status | type | Condition |
|--------|------|-----------|
| 404 | `not-found` | DPA not yet generated — call `POST /contracts/{auditId}/dpa/generate` first |
| 409 | `conflict` | DPA generation is still in progress — retry after `estimatedCompletionSeconds` |

---

## Scheduling

### `GET /schedules`

**Description:** List all configured scan schedules for the tenant.

**Required role:** `DPO`, `CTO`, `ANALYST`

**Response 200:**

```json
{
  "data": [
    {
      "scheduleId": "sch-s1a2b3c4-1111-2222-3333-444455556666",
      "cronExpression": "0 3 * * 1",
      "description": "Weekly incremental scan — every Monday at 3:00 AM IST",
      "jobType": "INCREMENTAL_SCAN",
      "connectorIds": [],
      "temporalScheduleId": "ts-disc-sch-s1a2b3c4",
      "active": true,
      "nextRunAt": "2026-04-27T21:30:00Z",
      "lastRunAt": "2026-04-20T21:30:00Z",
      "lastRunStatus": "COMPLETED",
      "createdAt": "2026-03-15T09:00:00Z"
    }
  ],
  "pagination": { "cursor": null, "hasMore": false, "pageSize": 20 }
}
```

---

### `POST /schedules`

**Description:** Create a new scheduled scan. The schedule is backed by a Temporal.io Schedule (not a cron daemon) — this gives durable, at-least-once execution guarantees. Cron expressions use UTC; IST offsets should be computed by the caller.

**Required role:** `DPO`, `CTO`

**Request body:**

```json
{
  "cronExpression": "0 3 * * 1",
  "description": "Weekly incremental scan — every Monday at 3:00 AM IST",
  "jobType": "INCREMENTAL_SCAN",
  "connectorIds": []
}
```

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `cronExpression` | string | Yes | Standard 5-field cron. UTC timezone. |
| `description` | string | Yes | Human-readable label for the schedule |
| `jobType` | enum | Yes | `FULL_SCAN` \| `INCREMENTAL_SCAN` |
| `connectorIds` | UUID[] | No | Empty = all active connectors |

**Response 201:**

```json
{
  "scheduleId": "sch-s1a2b3c4-1111-2222-3333-444455556666",
  "cronExpression": "0 3 * * 1",
  "description": "Weekly incremental scan — every Monday at 3:00 AM IST",
  "jobType": "INCREMENTAL_SCAN",
  "connectorIds": [],
  "active": true,
  "nextRunAt": "2026-04-27T21:30:00Z",
  "temporalScheduleId": "ts-disc-sch-s1a2b3c4",
  "createdAt": "2026-04-22T12:10:00Z"
}
```

**Error responses:**

| Status | type | Condition |
|--------|------|-----------|
| 400 | `validation-error` | `cronExpression` is not a valid 5-field cron expression |
| 409 | `conflict` | A `FULL_SCAN` schedule already exists (only one full-scan schedule per tenant) |

---

### `DELETE /schedules/{scheduleId}`

**Description:** Remove a scan schedule. Any currently running scan triggered by this schedule is not affected — it continues to completion. Future scheduled runs are cancelled.

**Required role:** `DPO`, `CTO`

**Path parameters:** `scheduleId` (UUID)

**Response 204:** No body.

**Error responses:**

| Status | type | Condition |
|--------|------|-----------|
| 404 | `not-found` | Schedule not found |

---

## Idempotency

All `POST` requests that create resources support idempotency via the `X-Request-Id` header. If the same `X-Request-Id` is sent within 24 hours, the original response is returned without creating a duplicate resource. The `X-Request-Id` must be a client-generated UUID.

```
X-Request-Id: 7c3e1d2a-8f9b-4a5c-b6e7-123456789abc
```

When an idempotent replay is detected, the response includes:

```
X-Idempotent-Replay: true
```

---

## Webhook Events

The following events are emitted to the tenant's configured webhook URL (configured in the Consent Vault module, shared across all TrustStack modules):

| Event | Trigger |
|-------|---------|
| `discovery.scan.completed` | Scan job reaches `COMPLETED` status |
| `discovery.scan.failed` | Scan job reaches `FAILED` status |
| `discovery.finding.critical` | A `CRITICAL` risk finding is detected |
| `discovery.finding.minor_data` | A `MINOR_DATA` entity type is detected (Section 9 escalation) |
| `discovery.contract.audit_completed` | Contract audit reaches `COMPLETED` or `REQUIRES_REVIEW` status |
| `discovery.snapshot.consumed` | Erasure Engine consumes a snapshot token |

Webhook payloads are signed with HMAC-SHA256 using the tenant's webhook secret. Each payload includes `X-TrustStack-Signature` and `X-TrustStack-Event` headers.

Example `discovery.finding.critical` payload:

```json
{
  "event": "discovery.finding.critical",
  "tenantId": "550e8400-e29b-41d4-a716-446655440000",
  "timestamp": "2026-04-22T11:22:07Z",
  "data": {
    "findingId": "f1a2b3c4-1234-5678-9012-abcdef012345",
    "entityType": "AADHAAR_NUMBER",
    "dpdpaCategory": "PERSONAL_IDENTITY",
    "riskLevel": "CRITICAL",
    "sourceSystem": "Zoho CRM — Production",
    "jobId": "e5f6a7b8-aaaa-bbbb-cccc-ddddeeeeeeee"
  }
}
```

Note: The `discovery.finding.minor_data` event triggers an automatic escalation workflow in the Regulatory War Room (Module 3) for Section 9 review. No targeted advertising or profiling configuration should reference systems that have received this event.
