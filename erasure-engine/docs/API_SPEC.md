# API Specification — Automatic Erasure Engine

**Version:** v1
**Date:** 2026-04-22
**Module:** 3.2 — Erasure Engine (TrustStack)
**Base URL:** `https://api.truststack.in/erasure/v1`
**Format:** OpenAPI 3.1

---

## Authentication

The Erasure Engine API supports two authentication schemes:

### Bearer JWT (Human users — DPO, CTO, Compliance Officer)

```
Authorization: Bearer <JWT>
```

JWTs are issued by TrustStack's Auth service (Auth0-compatible). Claims include:
- `sub`: user UUID
- `tenant_id`: tenant UUID
- `roles`: array of RBAC roles (see Role table below)
- `exp`: expiry (1 hour max)

### API Key (Service-to-service)

```
X-API-Key: ts_erasure_live_<64-char-hex>
X-Tenant-Id: <tenant-uuid>
```

API keys are issued per tenant and scoped to the Erasure Engine API only. The Consent Vault Kafka consumer uses this scheme when triggering erasures programmatically after a `consent.withdrawn` event.

---

## RBAC Roles

| Role | Description | Can Trigger | Can Approve | Can View | Can Override Plan |
|---|---|---|---|---|---|
| `ERASURE_ADMIN` | DPO / Compliance Head | Yes | Yes | Yes | Yes |
| `ERASURE_APPROVER` | IT Manager / CTO | No | Yes | Yes | No |
| `ERASURE_VIEWER` | Regulatory War Room read-only | No | No | Yes | No |
| `SERVICE_ACCOUNT` | Internal service (API key auth) | Yes | No | No | No |

---

## Common Headers

| Header | Required | Description |
|---|---|---|
| `Authorization` | Yes (JWT auth) | `Bearer <JWT>` |
| `X-API-Key` | Yes (service auth) | Service API key |
| `X-Tenant-Id` | Yes (service auth) | Tenant UUID |
| `X-Request-Id` | Recommended on all POST/PATCH | Client UUID for idempotency. Duplicate requests with the same ID within 24hr are de-duplicated. |
| `Content-Type` | Yes (POST/PATCH) | `application/json` |

---

## Rate Limits

| Operation class | Limit |
|---|---|
| Erasure triggers (`POST /erasures`) | 100 / hour per tenant |
| Status reads (`GET /erasures/*`) | 1,000 / hour per tenant |
| Approval decisions | 500 / hour per tenant |
| Certificate downloads | 200 / hour per tenant |
| Audit log reads | 500 / hour per tenant |

Rate limit headers returned on every response:
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 87
X-RateLimit-Reset: 1745321400
```

---

## Error Format (RFC 9457)

All errors follow the Problem Details standard:

```json
{
  "type": "https://api.truststack.in/errors/erasure/operation-not-found",
  "title": "Erasure Operation Not Found",
  "status": 404,
  "detail": "No erasure operation exists with id '550e8400-e29b-41d4-a716-446655440001' for the authenticated tenant.",
  "instance": "/erasure/v1/erasures/550e8400-e29b-41d4-a716-446655440001",
  "requestId": "req_01HZ8K3WQYV7N4EPGXRT52JPBS"
}
```

Common error types:

| HTTP Status | type (suffix) | When |
|---|---|---|
| 400 | `invalid-request` | Schema validation failure |
| 401 | `unauthenticated` | Missing or expired credentials |
| 403 | `forbidden` | Valid credentials, insufficient role |
| 404 | `not-found` | Resource does not exist for this tenant |
| 409 | `conflict` | Duplicate `X-Request-Id` or active operation already exists for this DP |
| 422 | `unprocessable` | Request structurally valid but semantically invalid (e.g., scheduled_at in the past) |
| 429 | `rate-limited` | Rate limit exceeded |
| 500 | `internal-error` | Unexpected server error |
| 503 | `service-unavailable` | Temporal.io or database temporarily unavailable |

---

## Endpoints

---

### 1. Erasure Operations

---

#### `POST /erasures`

**Trigger a new erasure operation.**

Initiates the 5-agent pipeline for a Data Principal. Creates an `erasure_operations` record, starts a Temporal workflow, and immediately returns. The operation progresses asynchronously.

For `STANDARD` class tenants: execution begins after Safety review, optional approval, and the mandatory 48hr pre-deletion notification window.

For `ECOMMERCE_GT_2CR` / `GAMING_GT_50L` / `SOCIAL_GT_2CR` tenants: the Temporal workflow sleeps until `scheduled_at` (up to 3 years), then sends the 48hr notification, then executes.

**DPDPA note:** The `dp_identifier_hash` must be computed by the caller as `HMAC-SHA-256(raw_identifier, tenant_secret)`. Raw identifiers must never appear in API requests. This is enforced server-side by verifying the hash format; no raw PII is logged.

**Required role:** `ERASURE_ADMIN` or `SERVICE_ACCOUNT`

**Request body:**

```json
{
  "dp_identifier_hash": "a3f2c1d9e8b7a6f5e4d3c2b1a0f9e8d7c6b5a4f3e2d1c0b9a8f7e6d5c4b3a2",
  "trigger_type": "CONSENT_WITHDRAWN",
  "erasure_class_override": null,
  "scheduled_at": null,
  "dry_run": false,
  "metadata": {
    "source_consent_id": "7c3e1d2a-8f9b-4a5c-b6e7-123456789abc",
    "source_purpose_id": "3a1b2c4d-5e6f-7a8b-9c0d-1e2f3a4b5c6d"
  }
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `dp_identifier_hash` | string | Yes | HMAC-SHA-256 hex string (64 chars). Server validates format. |
| `trigger_type` | enum | Yes | `CONSENT_WITHDRAWN` \| `DP_REQUEST` \| `PURPOSE_FULFILLED` \| `SCHEDULED_RETENTION` \| `REGULATORY_AUDIT` |
| `erasure_class_override` | enum \| null | No | Override the tenant's default erasure class for this operation only. |
| `scheduled_at` | ISO 8601 \| null | No | For deferred erasures. Must be in the future. If null, uses the tenant's class default. |
| `dry_run` | boolean | No | Default `false`. If `true`, runs full Planner + Safety pipeline but does not execute or issue certificate. Returns plan and safety review. |
| `metadata` | object | No | Arbitrary key-value pairs (non-PII) passed through to audit log. |

**Success response: `202 Accepted`**

```json
{
  "operation_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "status": "PENDING_PLAN",
  "tenant_id": "550e8400-e29b-41d4-a716-446655440000",
  "trigger_type": "CONSENT_WITHDRAWN",
  "erasure_class": "STANDARD",
  "dry_run": false,
  "temporal_workflow_id": "erasure-f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "scheduled_at": null,
  "estimated_completion_at": null,
  "created_at": "2026-04-22T10:30:00Z",
  "_links": {
    "self": "/erasure/v1/erasures/f47ac10b-58cc-4372-a567-0e02b2c3d479",
    "plan": "/erasure/v1/erasures/f47ac10b-58cc-4372-a567-0e02b2c3d479/plan",
    "audit_log": "/erasure/v1/erasures/f47ac10b-58cc-4372-a567-0e02b2c3d479/audit-log"
  }
}
```

**Error responses:**

| Status | Condition |
|---|---|
| 400 | `dp_identifier_hash` is not a valid 64-char hex string |
| 409 | An active (non-terminal) erasure operation already exists for this `dp_identifier_hash` on this tenant |
| 422 | `scheduled_at` is in the past |
| 503 | Temporal.io workflow service unavailable |

---

#### `GET /erasures`

**List all erasure operations for the authenticated tenant.**

**Required role:** `ERASURE_ADMIN`, `ERASURE_APPROVER`, or `ERASURE_VIEWER`

**Query parameters:**

| Parameter | Type | Default | Notes |
|---|---|---|---|
| `status` | enum | (all) | Filter by status. Comma-separated for multiple: `?status=PENDING_APPROVAL,EXECUTING` |
| `trigger_type` | enum | (all) | Filter by trigger type. |
| `erasure_class` | enum | (all) | Filter by erasure class. |
| `from` | ISO 8601 | 30 days ago | `created_at` lower bound. |
| `to` | ISO 8601 | now | `created_at` upper bound. |
| `page` | integer | 1 | 1-based page number. |
| `per_page` | integer | 20 | Max 100. |
| `sort` | string | `created_at:desc` | `created_at:asc\|desc` or `scheduled_at:asc\|desc` |

**Success response: `200 OK`**

```json
{
  "data": [
    {
      "operation_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
      "status": "PENDING_APPROVAL",
      "trigger_type": "DP_REQUEST",
      "erasure_class": "ECOMMERCE_GT_2CR",
      "scheduled_at": "2029-04-20T12:00:00Z",
      "notification_sent_at": null,
      "started_at": null,
      "completed_at": null,
      "created_at": "2026-04-22T10:30:00Z",
      "has_certificate": false,
      "has_pending_approval": true
    }
  ],
  "pagination": {
    "page": 1,
    "per_page": 20,
    "total_count": 347,
    "total_pages": 18
  }
}
```

---

#### `GET /erasures/{operationId}`

**Get full status of a single erasure operation, including all pipeline stages.**

**Required role:** `ERASURE_ADMIN`, `ERASURE_APPROVER`, or `ERASURE_VIEWER`

**Path parameters:**

| Parameter | Type | Notes |
|---|---|---|
| `operationId` | UUID | The `operation_id` from `POST /erasures`. |

**Success response: `200 OK`**

```json
{
  "operation_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "tenant_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "EXECUTING",
  "trigger_type": "DP_REQUEST",
  "erasure_class": "STANDARD",
  "dry_run": false,
  "temporal_workflow_id": "erasure-f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "scheduled_at": null,
  "notification_sent_at": "2026-04-20T14:00:00Z",
  "started_at": "2026-04-22T14:00:00Z",
  "completed_at": null,
  "created_at": "2026-04-20T14:00:00Z",
  "pipeline": {
    "plan": {
      "plan_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "status": "approved",
      "systems_count": 7,
      "total_estimated_records": 1847,
      "generated_at": "2026-04-20T14:01:23Z",
      "approved_at": "2026-04-20T15:30:00Z"
    },
    "safety_review": {
      "review_id": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
      "overall_result": "REQUIRES_HUMAN",
      "hard_violations_count": 0,
      "soft_warnings_count": 2,
      "regulatory_conflicts_count": 1,
      "reviewed_at": "2026-04-20T14:02:11Z"
    },
    "approval": {
      "approval_id": "c3d4e5f6-a7b8-9012-cdef-123456789012",
      "status": "APPROVED",
      "approved_by": "dpo-user-uuid",
      "decided_at": "2026-04-20T15:30:00Z"
    },
    "execution": {
      "systems_total": 7,
      "systems_completed": 4,
      "systems_failed": 0,
      "systems_in_progress": 1,
      "systems_pending": 2,
      "systems_skipped_regulatory_hold": 1
    },
    "certificate": null
  },
  "_links": {
    "self": "/erasure/v1/erasures/f47ac10b-58cc-4372-a567-0e02b2c3d479",
    "plan": "/erasure/v1/erasures/f47ac10b-58cc-4372-a567-0e02b2c3d479/plan",
    "systems": "/erasure/v1/erasures/f47ac10b-58cc-4372-a567-0e02b2c3d479/systems",
    "audit_log": "/erasure/v1/erasures/f47ac10b-58cc-4372-a567-0e02b2c3d479/audit-log"
  }
}
```

**Error responses:**

| Status | Condition |
|---|---|
| 404 | Operation does not exist for the authenticated tenant |

---

#### `POST /erasures/{operationId}/stop`

**Emergency stop — halt the erasure at the next safe checkpoint.**

Sends a Temporal signal to the running workflow. Execution halts before the next system deletion. Systems that have already been deleted are not reversed (DPDPA prohibits soft-delete workarounds). A partial deletion certificate is issued for completed systems.

**DPDPA note:** This endpoint must be used with care. Any system already deleted before the stop cannot be restored. The audit log records the stop signal, reason, and the user who triggered it.

**Required role:** `ERASURE_ADMIN`

**Request body:**

```json
{
  "reason": "Legal hold received — court order pending. Stopping to avoid destroying evidence."
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `reason` | string | Yes | Human-readable reason. Stored in audit log. Max 1000 chars. |

**Success response: `200 OK`**

```json
{
  "operation_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "status": "STOPPED",
  "stopped_at": "2026-04-22T14:32:10Z",
  "stopped_by": "dpo-user-uuid",
  "reason": "Legal hold received — court order pending.",
  "systems_completed_before_stop": 4,
  "systems_not_executed": 3,
  "partial_certificate_available": true,
  "_links": {
    "certificate": "/erasure/v1/erasures/f47ac10b-58cc-4372-a567-0e02b2c3d479/certificate"
  }
}
```

**Error responses:**

| Status | Condition |
|---|---|
| 404 | Operation not found |
| 409 | Operation is already in a terminal state (`COMPLETED`, `FAILED`, `STOPPED`) |

---

#### `POST /erasures/dry-run`

**Simulate a complete erasure without deleting anything.**

Runs the full Planner Agent and Safety Agent pipeline. Returns what would be erased, in what order, and any safety flags — without triggering any deletions or creating a live `erasure_operations` record. Useful for DPOs previewing the scope before initiating a real erasure.

**DPDPA note:** Dry runs do not start the 48-hour notification clock and do not appear in the DP's consent dashboard.

**Required role:** `ERASURE_ADMIN`

**Request body:** Same schema as `POST /erasures` (the `dry_run` field is implied true and ignored if passed).

**Success response: `200 OK`**

```json
{
  "dry_run": true,
  "dp_identifier_hash": "a3f2c1d9e8b7a6f5e4d3c2b1a0f9e8d7c6b5a4f3e2d1c0b9a8f7e6d5c4b3a2",
  "erasure_class": "STANDARD",
  "plan": {
    "systems": [
      {
        "system_id": "zoho_crm",
        "system_name": "Zoho CRM",
        "connector": "ZOHO_CRM",
        "data_categories": ["email", "phone", "contact_notes"],
        "estimated_records": 47,
        "execution_order": 1,
        "deletion_method": "AUTO_API",
        "risk_level": "LOW",
        "requires_approval": false
      },
      {
        "system_id": "razorpay",
        "system_name": "Razorpay",
        "connector": "RAZORPAY",
        "data_categories": ["payment_records", "kyc_documents"],
        "estimated_records": 12,
        "execution_order": 2,
        "deletion_method": "CONNECTOR",
        "risk_level": "HIGH",
        "requires_approval": true,
        "regulatory_conflict": {
          "reason": "RBI_5YR_PAYMENT",
          "action": "ANONYMISE_PII",
          "records_can_anonymize": 3,
          "records_must_retain": 12
        }
      }
    ],
    "total_estimated_records": 1847,
    "execution_order": ["zoho_crm", "whatsapp_business", "aws_s3", "tenant_postgres", "razorpay"],
    "planner_reasoning": "Zoho CRM and WhatsApp contain contact data with no cross-system dependencies and should be deleted first. Tenant PostgreSQL contains order records that reference Razorpay transaction IDs — Razorpay is therefore placed last. Razorpay transaction records cannot be hard-deleted due to RBI 5yr retention; customer PII fields will be anonymised."
  },
  "safety_review": {
    "overall_result": "REQUIRES_HUMAN",
    "hard_rule_violations": [],
    "soft_rule_warnings": [
      {
        "rule_id": "HIGH_VOLUME",
        "rule_name": "High-Volume Deletion",
        "description": "Total estimated records (1847) exceeds the 1000-record human approval threshold.",
        "risk_level": "MEDIUM",
        "recommendation": "Review the system list and record counts before approving."
      }
    ],
    "regulatory_conflicts": [
      {
        "system": "razorpay",
        "conflict": "RBI_5YR_PAYMENT",
        "affected_categories": ["payment_records", "kyc_documents"],
        "regulation": "RBI Payment System Regulations 2018",
        "action": "ANONYMISE_PII"
      }
    ]
  },
  "would_require_approval": true,
  "approval_reasons": ["RECORD_COUNT_EXCEEDS_1000", "FINANCIAL_DATA"],
  "simulated_at": "2026-04-22T10:30:00Z"
}
```

---

### 2. Plan Management

---

#### `GET /erasures/{operationId}/plan`

**Retrieve the Planner Agent's erasure plan for an operation.**

**Required role:** `ERASURE_ADMIN`, `ERASURE_APPROVER`, or `ERASURE_VIEWER`

**Success response: `200 OK`**

```json
{
  "plan_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "operation_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "plan_version": 1,
  "generated_by": "PLANNER_AGENT",
  "generated_at": "2026-04-22T10:31:23Z",
  "approved_at": null,
  "planner_reasoning": "...",
  "systems_targeted": [
    {
      "system_id": "zoho_crm",
      "system_name": "Zoho CRM",
      "connector": "ZOHO_CRM",
      "data_categories": ["email", "phone", "contact_notes"],
      "execution_order": 1,
      "estimated_records": 47,
      "deletion_method": "AUTO_API",
      "risk_level": "LOW",
      "requires_approval": false
    }
  ],
  "execution_order": ["zoho_crm", "whatsapp_business", "aws_s3", "tenant_postgres"],
  "total_estimated_records": 1847
}
```

**Error responses:**

| Status | Condition |
|---|---|
| 404 | Operation not found, or plan not yet generated (still in `PENDING_PLAN`) |

---

#### `PATCH /erasures/{operationId}/plan`

**Override the Planner Agent's plan (DPO manual adjustment).**

Allows a DPO to add or remove systems from the plan, or change execution order. Creates a new plan version (`plan_version` + 1) marked `MANUAL_OVERRIDE`. Any in-progress approval is invalidated and must be re-requested.

**DPDPA note:** Manual overrides are fully logged in the audit log with the DPO's identity. This maintains the chain of accountability required for DPBI audits.

**Required role:** `ERASURE_ADMIN`

**Request body:**

```json
{
  "override_reason": "Tally data requires manual handling per our accountant's advice — removing from auto-deletion.",
  "systems_to_remove": ["tally"],
  "systems_to_add": [],
  "execution_order_override": null
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `override_reason` | string | Yes | Stored in audit log. Max 2000 chars. |
| `systems_to_remove` | string[] | No | Array of `system_id` strings to exclude from execution. |
| `systems_to_add` | object[] | No | Additional systems not in the original plan (same schema as `systems_targeted`). |
| `execution_order_override` | string[] \| null | No | Full replacement of `execution_order`. Must include all non-removed systems. |

**Success response: `200 OK`** — Returns updated plan (same schema as `GET /erasures/{operationId}/plan`).

**Error responses:**

| Status | Condition |
|---|---|
| 403 | Caller does not have `ERASURE_ADMIN` role |
| 409 | Operation is in `EXECUTING`, `VERIFYING`, `COMPLETED`, `FAILED`, or `STOPPED` state (plan cannot be changed) |

---

### 3. Approvals

---

#### `GET /approvals`

**List all pending approval requests for the authenticated tenant.**

Intended for the DPO/CTO approver dashboard. Returns only `PENDING` requests by default.

**Required role:** `ERASURE_ADMIN` or `ERASURE_APPROVER`

**Query parameters:**

| Parameter | Type | Default | Notes |
|---|---|---|---|
| `status` | enum | `PENDING` | `PENDING` \| `APPROVED` \| `REJECTED` \| `EXPIRED` |
| `page` | integer | 1 | |
| `per_page` | integer | 20 | Max 100. |

**Success response: `200 OK`**

```json
{
  "data": [
    {
      "approval_id": "c3d4e5f6-a7b8-9012-cdef-123456789012",
      "operation_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
      "status": "PENDING",
      "required_because": ["RECORD_COUNT_EXCEEDS_1000", "FINANCIAL_DATA"],
      "expires_at": "2026-04-23T10:30:00Z",
      "hours_until_expiry": 23.5,
      "total_systems": 7,
      "total_estimated_records": 1847,
      "requested_at": "2026-04-22T10:30:00Z",
      "_links": {
        "detail": "/erasure/v1/approvals/c3d4e5f6-a7b8-9012-cdef-123456789012",
        "operation": "/erasure/v1/erasures/f47ac10b-58cc-4372-a567-0e02b2c3d479"
      }
    }
  ],
  "pagination": {
    "page": 1,
    "per_page": 20,
    "total_count": 3,
    "total_pages": 1
  }
}
```

---

#### `GET /approvals/{approvalId}`

**Get full approval request detail — plan, safety review, and approval reasons.**

Designed for the DPO decision screen: shows everything the approver needs to make an informed decision.

**Required role:** `ERASURE_ADMIN` or `ERASURE_APPROVER`

**Success response: `200 OK`**

```json
{
  "approval_id": "c3d4e5f6-a7b8-9012-cdef-123456789012",
  "operation_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "status": "PENDING",
  "required_because": ["RECORD_COUNT_EXCEEDS_1000", "FINANCIAL_DATA"],
  "expires_at": "2026-04-23T10:30:00Z",
  "notification_channels": ["SLACK", "EMAIL"],
  "requested_at": "2026-04-22T10:30:00Z",
  "plan_summary": {
    "plan_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "plan_version": 1,
    "generated_by": "PLANNER_AGENT",
    "planner_reasoning": "...",
    "systems_targeted": [ ],
    "total_estimated_records": 1847,
    "execution_order": ["zoho_crm", "whatsapp_business", "tenant_postgres", "razorpay"]
  },
  "safety_summary": {
    "overall_result": "REQUIRES_HUMAN",
    "hard_violations_count": 0,
    "soft_warnings": [
      {
        "rule_id": "HIGH_VOLUME",
        "rule_name": "High-Volume Deletion",
        "description": "1847 records across 7 systems exceeds the 1000-record threshold.",
        "risk_level": "MEDIUM",
        "recommendation": "Review each system carefully."
      }
    ],
    "regulatory_conflicts": [
      {
        "system": "razorpay",
        "conflict": "RBI_5YR_PAYMENT",
        "regulation": "RBI Payment System Regulations 2018",
        "action": "ANONYMISE_PII"
      }
    ]
  }
}
```

---

#### `POST /approvals/{approvalId}/approve`

**Approve the erasure operation.**

Sends an approval signal to the Temporal workflow. The workflow proceeds to the 48-hour pre-deletion notification step.

**DPDPA note:** Approval is recorded with the approver's user ID and timestamp. This forms part of the immutable audit trail. Once approved, the 48-hour window cannot be reduced — it is hardcoded in the Temporal workflow per DPDPA Third Schedule requirements.

**Required role:** `ERASURE_ADMIN` or `ERASURE_APPROVER`

**Request body:**

```json
{
  "note": "Reviewed all 7 systems. Razorpay PII anonymisation confirmed with finance team. Proceeding."
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `note` | string | No | Optional approver note stored in audit log. Max 2000 chars. |

**Success response: `200 OK`**

```json
{
  "approval_id": "c3d4e5f6-a7b8-9012-cdef-123456789012",
  "operation_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "status": "APPROVED",
  "approved_by": "dpo-user-uuid",
  "decided_at": "2026-04-22T15:30:00Z",
  "next_step": "48-hour pre-deletion notification will be sent to the Data Principal. Execution begins after the notification window.",
  "notification_scheduled_for": "2026-04-22T15:30:00Z",
  "execution_earliest": "2026-04-24T15:30:00Z"
}
```

**Error responses:**

| Status | Condition |
|---|---|
| 404 | Approval not found |
| 409 | Approval already decided (`APPROVED`, `REJECTED`, or `EXPIRED`) |

---

#### `POST /approvals/{approvalId}/reject`

**Reject the erasure operation.**

The erasure workflow is cancelled. The Data Principal is notified that their erasure request could not be processed at this time (DPDPA Section 12 requires a reason be given). The rejection reason is mandatory.

**Required role:** `ERASURE_ADMIN` or `ERASURE_APPROVER`

**Request body:**

```json
{
  "reason": "Active court order prevents destruction of records related to case CAS-2026-001. Legal team notified. Erasure will be re-initiated once court order is lifted."
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `reason` | string | Yes | Stored in audit log and included in DP notification. Max 2000 chars. |

**Success response: `200 OK`**

```json
{
  "approval_id": "c3d4e5f6-a7b8-9012-cdef-123456789012",
  "operation_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "status": "REJECTED",
  "rejected_by": "dpo-user-uuid",
  "decided_at": "2026-04-22T15:30:00Z",
  "reason": "Active court order prevents destruction of records..."
}
```

---

### 4. Execution Status

---

#### `GET /erasures/{operationId}/systems`

**List per-system erasure status for an operation.**

Returns one `system_erasure_record` per target system. Used for the real-time execution progress view in the DPO dashboard.

**Required role:** `ERASURE_ADMIN`, `ERASURE_APPROVER`, or `ERASURE_VIEWER`

**Success response: `200 OK`**

```json
{
  "operation_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "systems": [
    {
      "record_id": "d4e5f6a7-b8c9-0123-defg-23456789012a",
      "system_name": "Zoho CRM",
      "connector_type": "ZOHO_CRM",
      "execution_order": 1,
      "status": "VERIFIED",
      "records_targeted": 47,
      "records_deleted": 47,
      "records_anonymized": 0,
      "records_retained": 0,
      "regulatory_hold_reason": null,
      "verification_result": {
        "records_found_after_deletion": 0,
        "verified_at": "2026-04-22T14:35:02Z",
        "method": "API_LOOKUP"
      },
      "started_at": "2026-04-22T14:32:00Z",
      "completed_at": "2026-04-22T14:34:55Z"
    },
    {
      "record_id": "e5f6a7b8-c9d0-1234-efgh-34567890123b",
      "system_name": "Razorpay",
      "connector_type": "RAZORPAY",
      "execution_order": 4,
      "status": "COMPLETED",
      "records_targeted": 15,
      "records_deleted": 0,
      "records_anonymized": 3,
      "records_retained": 12,
      "regulatory_hold_reason": "RBI_5YR_PAYMENT",
      "verification_result": null,
      "started_at": "2026-04-22T14:36:00Z",
      "completed_at": "2026-04-22T14:36:45Z"
    }
  ],
  "summary": {
    "total": 7,
    "verified": 3,
    "completed": 1,
    "in_progress": 1,
    "pending": 2,
    "failed": 0,
    "skipped_regulatory_hold": 0
  }
}
```

---

#### `GET /erasures/{operationId}/systems/{systemId}`

**Get erasure status for a single target system.**

**Required role:** `ERASURE_ADMIN`, `ERASURE_APPROVER`, or `ERASURE_VIEWER`

**Path parameters:**

| Parameter | Type | Notes |
|---|---|---|
| `systemId` | string | The `system_id` value from the plan (e.g., `zoho_crm`, `razorpay`). |

**Success response: `200 OK`** — Returns a single system object (same schema as items in `GET /erasures/{operationId}/systems`).

---

### 5. Certificates

---

#### `GET /erasures/{operationId}/certificate`

**Get the deletion certificate for a completed operation.**

**DPDPA note:** Deletion certificates are ECDSA P-256 signed with the tenant's KMS key. They are the primary evidence artefact for DPBI compliance audits and can be submitted directly to DPBI as proof of erasure. Certificates are retained for 7 years per DPDPA Rule 4.

**Required role:** `ERASURE_ADMIN`, `ERASURE_APPROVER`, or `ERASURE_VIEWER`

**Success response: `200 OK`**

```json
{
  "certificate_id": "cert-01HZ8K3WQYV7N4EPGXRT52JPBS",
  "operation_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "tenant_id": "550e8400-e29b-41d4-a716-446655440000",
  "version": "2.0",
  "dp_identifier_hash": "a3f2c1d9e8b7a6f5e4d3c2b1a0f9e8d7c6b5a4f3e2d1c0b9a8f7e6d5c4b3a2",
  "dpdpa_section": "Section 12 (Right to Erasure)",
  "third_schedule_class": "STANDARD",
  "summary": {
    "total_systems_targeted": 7,
    "systems_deleted": 5,
    "systems_with_regulatory_hold": 1,
    "systems_failed": 0,
    "total_records_deleted": 1802,
    "total_records_anonymized": 3,
    "total_records_retained": 12
  },
  "systems_deleted": [
    {
      "system_name": "Zoho CRM",
      "connector_type": "ZOHO_CRM",
      "status": "DELETED",
      "records_deleted": 47,
      "deletion_timestamp": "2026-04-22T14:34:55Z",
      "verification_status": "VERIFIED"
    }
  ],
  "systems_with_regulatory_hold": [
    {
      "system_name": "Razorpay",
      "hold_reason": "RBI_5YR_PAYMENT",
      "regulation": "RBI Payment System Regulations 2018",
      "records_retained": 12,
      "records_anonymized": 3,
      "expected_release_date": "2031-04-22"
    }
  ],
  "initiated_at": "2026-04-20T14:00:00Z",
  "completed_at": "2026-04-22T14:45:00Z",
  "issued_at": "2026-04-22T14:45:30Z",
  "sent_to_dp_at": "2026-04-22T14:46:00Z",
  "sent_channel": "WHATSAPP",
  "certificate_hash": "sha256:9f86d08188...",
  "ecdsa_signature": "MEQCIBx3...",
  "signing_key_id": "arn:aws:kms:ap-south-1:123456789:key/abc-def-ghi",
  "signature_algorithm": "ECDSA_SHA_256"
}
```

**Error responses:**

| Status | Condition |
|---|---|
| 404 | Operation not found, or certificate not yet issued (operation not `COMPLETED` or `FAILED`) |

---

#### `GET /certificates/{certificateId}`

**Download a deletion certificate by its ID.**

Supports both JSON and PDF formats via the `Accept` header.

**Required role:** `ERASURE_ADMIN`, `ERASURE_APPROVER`, or `ERASURE_VIEWER`

**Headers:**

| Header | Values | Notes |
|---|---|---|
| `Accept` | `application/json` (default) or `application/pdf` | PDF is a formatted, legally-styled document suitable for email attachment. |

**Success response: `200 OK`**

- `application/json`: Same schema as `GET /erasures/{operationId}/certificate`.
- `application/pdf`: Binary PDF stream. Headers include `Content-Disposition: attachment; filename="truststack-deletion-cert-{certificateId}.pdf"`.

---

#### `POST /certificates/{certificateId}/verify`

**Independently verify the ECDSA signature of a deletion certificate.**

Verifies that `certificate_hash` matches the certificate content, and that `ecdsa_signature` is a valid ECDSA P-256 signature over `certificate_hash` using the tenant's public key.

This endpoint is also callable without authentication — designed for DPBI auditors or third parties who hold a certificate and want to verify it without TrustStack access.

**Required role:** None (public endpoint for signature verification)

**Request body:**

```json
{
  "certificate_id": "cert-01HZ8K3WQYV7N4EPGXRT52JPBS",
  "certificate_json": { }
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `certificate_id` | string | Yes | |
| `certificate_json` | object | No | If provided, verifies this payload against the stored signature. If omitted, verifies the stored certificate against its stored signature. |

**Success response: `200 OK`**

```json
{
  "certificate_id": "cert-01HZ8K3WQYV7N4EPGXRT52JPBS",
  "signature_valid": true,
  "hash_matches": true,
  "signing_key_id": "arn:aws:kms:ap-south-1:123456789:key/abc-def-ghi",
  "signature_algorithm": "ECDSA_SHA_256",
  "issued_at": "2026-04-22T14:45:30Z",
  "verified_at": "2026-04-22T16:00:00Z"
}
```

**Tampered certificate response:**

```json
{
  "certificate_id": "cert-01HZ8K3WQYV7N4EPGXRT52JPBS",
  "signature_valid": false,
  "hash_matches": false,
  "failure_reason": "Certificate content does not match hash. Certificate may have been tampered with.",
  "verified_at": "2026-04-22T16:00:00Z"
}
```

---

#### `GET /erasures/{operationId}/certificate/send`

**Re-send the deletion certificate to the Data Principal.**

Delivers the certificate again via the DP's registered channel. Useful if the DP reports not receiving it. Creates an audit log entry.

**Required role:** `ERASURE_ADMIN`

**Query parameters:**

| Parameter | Type | Notes |
|---|---|---|
| `channel` | enum | Optional override: `EMAIL` \| `WHATSAPP` \| `IN_APP` \| `SMS` \| `API_CALLBACK`. Defaults to original channel. |

**Success response: `200 OK`**

```json
{
  "certificate_id": "cert-01HZ8K3WQYV7N4EPGXRT52JPBS",
  "resent": true,
  "channel": "EMAIL",
  "sent_at": "2026-04-22T16:05:00Z"
}
```

---

### 6. Notifications

---

#### `GET /erasures/{operationId}/notifications`

**List all notifications sent for an operation** (pre-deletion notices and certificate delivery).

**Required role:** `ERASURE_ADMIN`, `ERASURE_APPROVER`, or `ERASURE_VIEWER`

**Success response: `200 OK`**

```json
{
  "operation_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "notifications": [
    {
      "notification_id": "f6a7b8c9-d0e1-2345-fghi-456789012345",
      "type": "PRE_DELETION",
      "notification_language": "hi",
      "notification_channel": "WHATSAPP",
      "status": "DELIVERED",
      "scheduled_for": "2026-04-22T14:00:00Z",
      "sent_at": "2026-04-22T14:00:05Z",
      "delivery_confirmed_at": "2026-04-22T14:00:08Z",
      "opt_out_received": false
    },
    {
      "notification_id": "a7b8c9d0-e1f2-3456-ghij-567890123456",
      "type": "CERTIFICATE",
      "notification_language": "hi",
      "notification_channel": "WHATSAPP",
      "status": "DELIVERED",
      "sent_at": "2026-04-22T14:46:00Z",
      "delivery_confirmed_at": "2026-04-22T14:46:03Z"
    }
  ]
}
```

---

#### `POST /erasures/{operationId}/notifications/preview`

**Preview the 48-hour pre-deletion notification in the DP's language.**

Returns the rendered notification content (subject + body) as it will be sent to the DP. Useful for the DPO to verify the vernacular translation is correct before approving.

**DPDPA note:** Section 5, Rule 3 requires notices to be in the user's interface language. This preview endpoint allows DPOs to validate vernacular content before it reaches the DP.

**Required role:** `ERASURE_ADMIN`

**Request body:**

```json
{
  "language": "ta",
  "channel": "WHATSAPP"
}
```

**Success response: `200 OK`**

```json
{
  "language": "ta",
  "channel": "WHATSAPP",
  "subject": null,
  "body": "அன்புள்ள தரவு அதிபர், உங்கள் தரவு நீக்கும் கோரிக்கை பதிவு செய்யப்பட்டது. 48 மணி நேரத்திற்குள் உங்கள் தரவு நீக்கப்படும்...",
  "body_english_translation": "Dear Data Principal, your data deletion request has been registered. Your data will be deleted within 48 hours...",
  "contains_opt_out_instructions": true,
  "preview_generated_at": "2026-04-22T10:30:00Z"
}
```

---

### 7. Audit Log

---

#### `GET /erasures/{operationId}/audit-log`

**Get the immutable audit trail for a single erasure operation.**

Returns all audit events for this operation in chronological order. The audit log is append-only — no events can be deleted or modified.

**DPDPA note:** This log is the evidentiary record for DPBI audits. It shows every decision made by every agent and every human in the erasure pipeline, in order, with timestamps and actor identity.

**Required role:** `ERASURE_ADMIN`, `ERASURE_APPROVER`, or `ERASURE_VIEWER`

**Query parameters:**

| Parameter | Type | Default | Notes |
|---|---|---|---|
| `from` | ISO 8601 | operation start | |
| `to` | ISO 8601 | now | |
| `actor` | enum | (all) | Filter by actor type. |
| `event_type` | string | (all) | Filter by event type. |

**Success response: `200 OK`**

```json
{
  "operation_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "events": [
    {
      "log_id": "1a2b3c4d-5e6f-7890-abcd-ef1234567890",
      "event_type": "OPERATION_CREATED",
      "actor": "API_CLIENT",
      "actor_id": "service-consent-vault",
      "details": {
        "trigger_type": "CONSENT_WITHDRAWN",
        "erasure_class": "STANDARD",
        "source_consent_id": "7c3e1d2a-8f9b-4a5c-b6e7-123456789abc"
      },
      "previous_status": null,
      "new_status": "PENDING_PLAN",
      "created_at": "2026-04-22T10:30:00Z"
    },
    {
      "log_id": "2b3c4d5e-6f7a-8901-bcde-f12345678901",
      "event_type": "PLAN_GENERATED",
      "actor": "PLANNER_AGENT",
      "actor_id": "erasure-engine-sagemaker-endpoint",
      "details": {
        "plan_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
        "systems_count": 7,
        "total_estimated_records": 1847,
        "llm_model": "mistral-7b-erasure-v1",
        "llm_tokens_input": 4200,
        "llm_tokens_output": 890
      },
      "previous_status": "PENDING_PLAN",
      "new_status": "PENDING_SAFETY",
      "created_at": "2026-04-22T10:31:23Z"
    },
    {
      "log_id": "3c4d5e6f-7a8b-9012-cdef-123456789012",
      "event_type": "HUMAN_APPROVED",
      "actor": "HUMAN_APPROVER",
      "actor_id": "dpo-user-uuid",
      "details": {
        "approval_id": "c3d4e5f6-a7b8-9012-cdef-123456789012",
        "note": "Reviewed all 7 systems. Proceeding."
      },
      "previous_status": "PENDING_APPROVAL",
      "new_status": "APPROVED",
      "created_at": "2026-04-22T15:30:00Z"
    }
  ],
  "total_events": 23
}
```

---

#### `GET /audit-log`

**Global audit log for the tenant (all operations, paginated).**

**Required role:** `ERASURE_ADMIN`

**Query parameters:** Same as operation-level audit log, plus:

| Parameter | Type | Notes |
|---|---|---|
| `operation_id` | UUID | Filter to a specific operation. |
| `page` | integer | |
| `per_page` | integer | Max 100. |

**Success response: `200 OK`** — Same schema as operation-level, with `operation_id` on each event and `pagination` object appended.

---

### 8. Webhooks

The Erasure Engine emits webhook events to the tenant's registered webhook URL for all significant lifecycle transitions. Webhook payloads are signed with `HMAC-SHA-256(payload, tenant_webhook_secret)` in the `X-TrustStack-Signature` header.

**Event catalogue:**

| Event | Fired when |
|---|---|
| `erasure.initiated` | A new erasure operation is created |
| `erasure.plan.ready` | Planner Agent has generated a plan |
| `erasure.approval.required` | Operation is waiting at Human Approval Gate |
| `erasure.approved` | DPO or approver approved the operation |
| `erasure.rejected` | DPO or approver rejected the operation |
| `erasure.notification.sent` | 48-hour pre-deletion notice sent to DP |
| `erasure.executing` | Executor Agent has started deletions |
| `erasure.system.completed` | A single target system was deleted successfully |
| `erasure.system.failed` | A single target system deletion failed |
| `erasure.completed` | All systems processed; certificate issued |
| `erasure.failed` | Operation failed (partial certificate issued if any systems succeeded) |
| `erasure.stopped` | Emergency stop was triggered |
| `erasure.certificate.issued` | Deletion certificate signed and stored |

**Webhook payload schema:**

```json
{
  "event": "erasure.completed",
  "event_id": "evt_01HZ8K3WQYV7N4EPGXRT52JPBS",
  "tenant_id": "550e8400-e29b-41d4-a716-446655440000",
  "operation_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "occurred_at": "2026-04-22T14:45:00Z",
  "data": {
    "status": "COMPLETED",
    "systems_deleted": 6,
    "systems_with_regulatory_hold": 1,
    "certificate_id": "cert-01HZ8K3WQYV7N4EPGXRT52JPBS"
  }
}
```

Delivery: HTTPS POST to `tenant.webhook_url`. Retried with exponential backoff (5 attempts, max 1hr). Webhook failures are logged in the audit log. Events older than 7 days that were never successfully delivered are recorded as `WEBHOOK_DELIVERY_FAILED` audit events.
