# API Specification — SahmatOS Consent Vault

**Version:** v1  
**Date:** 2026-04-22  
**Base URL:** `https://api.sahmat.truststack.in/v1`  
**Format:** OpenAPI 3.1  

---

## Authentication

All API calls require one of:
1. **API Key** (`X-API-Key` header) — for Data Fiduciary backends
2. **Bearer JWT** (`Authorization: Bearer <token>`) — for consumer dashboard and admin

API keys are scoped to a single tenant. A tenant cannot access another tenant's resources regardless of key.

```
X-API-Key: ts_live_<32-char-hex>
X-Tenant-Id: <uuid>    # Required with API key auth
```

---

## Common Headers

| Header | Required | Description |
|--------|----------|-------------|
| `X-API-Key` | Yes (backend) | Data Fiduciary API key |
| `X-Tenant-Id` | Yes | Tenant UUID |
| `X-Request-Id` | Recommended | Client-generated UUID for idempotency |
| `X-DP-Language` | Optional | Override detected language (BCP 47) |
| `X-Device-Locale` | Optional | Device locale for language detection |
| `Accept-Language` | Optional | Standard browser language header |

---

## 1. Consent Endpoints

### 1.1 Grant or Withdraw Consent

**Use case:** Called by Data Fiduciary backend or widget after user action.

```
POST /v1/consent
```

**Request Body:**

```json
{
  "tenantId": "550e8400-e29b-41d4-a716-446655440000",
  "dpIdentifier": "+919876543210",
  "dpIdentifierType": "phone",
  "purposeId": "7c3e1d2a-8f9b-4a5c-b6e7-123456789abc",
  "action": "GRANTED",
  "collectionMethod": "WIDGET",
  "languageCode": "hi",
  "channelEvidence": null
}
```

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `tenantId` | UUID | Yes | Must match authenticated tenant |
| `dpIdentifier` | string | Yes | Raw identifier (stored encrypted; transmitted over TLS) |
| `dpIdentifierType` | enum | Yes | `phone` \| `email` \| `user_id` \| `aadhaar_virtual_id` |
| `purposeId` | UUID | Yes | Must exist and be approved for this tenant |
| `action` | enum | Yes | `GRANTED` \| `WITHDRAWN` |
| `collectionMethod` | enum | Yes | `WIDGET` \| `API` \| `WHATSAPP` \| `VERBAL_AADHAAR` |
| `languageCode` | string | No | BCP 47. If omitted, auto-detected from request. |
| `channelEvidence` | object | No | Required for `WHATSAPP` method |

**WhatsApp channel evidence:**
```json
{
  "wabaId": "1234567890",
  "messageId": "wamid.HBgL...",
  "messageTimestamp": "2026-04-22T10:30:00Z",
  "readReceiptTimestamp": "2026-04-22T10:30:05Z"
}
```

**Response 201 Created:**

```json
{
  "artifactId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "tenantId": "550e8400-e29b-41d4-a716-446655440000",
  "purposeId": "7c3e1d2a-8f9b-4a5c-b6e7-123456789abc",
  "action": "GRANTED",
  "languageCode": "hi",
  "collectionMethod": "WIDGET",
  "signature": "MEYCIQDmF3J...",
  "signingKeyId": "arn:aws:kms:ap-south-1:123456789:key/abc123",
  "createdAt": "2026-04-22T10:30:00.123Z",
  "noticeVersion": 3,
  "expiresAt": null
}
```

**Errors:**

| Status | Code | Description |
|--------|------|-------------|
| 400 | `INVALID_PURPOSE` | Purpose does not exist or not approved |
| 400 | `MINOR_WITHOUT_GUARDIAN` | DP is under 18 and no guardian consent linked |
| 400 | `PURPOSE_NOT_TRANSLATED` | Purpose lacks approved translation for requested language |
| 401 | `INVALID_API_KEY` | API key missing or invalid |
| 403 | `TENANT_MISMATCH` | tenantId in body doesn't match authenticated tenant |
| 409 | `DUPLICATE_REQUEST` | Same X-Request-Id already processed (idempotency) |
| 422 | `INVALID_LANGUAGE_CODE` | Language code not in supported list |
| 429 | `RATE_LIMITED` | Too many requests |

---

### 1.2 Check Consent Status

**Use case:** Called by DF backend before every data processing operation. High-frequency, cached.

```
GET /v1/consent/check?tenantId={id}&dpIdentifier={id}&purposeId={id}
```

**Query Parameters:**

| Parameter | Type | Required | Notes |
|-----------|------|----------|-------|
| `tenantId` | UUID | Yes | |
| `dpIdentifier` | string | Yes | URL-encoded |
| `purposeId` | UUID | Yes | |

**Response 200 OK:**

```json
{
  "tenantId": "550e8400-...",
  "purposeId": "7c3e1d2a-...",
  "status": "ACTIVE",
  "grantedAt": "2026-04-22T10:30:00Z",
  "expiresAt": null,
  "languageCode": "hi",
  "artifactId": "a1b2c3d4-..."
}
```

**Status values:**

| Status | Meaning | DF should... |
|--------|---------|-------------|
| `ACTIVE` | Valid consent exists | Proceed with processing |
| `WITHDRAWN` | Consent was withdrawn | Stop all processing for this purpose |
| `EXPIRED` | Time-bound consent expired | Re-collect consent |
| `NOT_FOUND` | No consent record exists | Collect consent before processing |
| `PENDING_GUARDIAN` | Minor awaiting guardian approval | Wait for guardian consent |

**Response latency:** p50 < 50ms (Redis cache hit), p99 < 200ms (cache miss + DB)

---

### 1.3 List Consent Records (Consumer Dashboard)

**Use case:** Consumer viewing all their consents.

```
GET /v1/consent?dpIdentifier={id}&languageCode={code}
```

Authentication: Consumer JWT (not DF API key)

**Response 200 OK:**

```json
{
  "total": 12,
  "items": [
    {
      "artifactId": "a1b2c3d4-...",
      "tenantName": "Meesho",
      "purposeTitle": "मार्केटिंग संदेश भेजना",
      "purposeDescription": "आपको ऑफर और नई चीज़ों के बारे में SMS भेजना",
      "action": "GRANTED",
      "grantedAt": "2026-04-22T10:30:00Z",
      "languageCode": "hi",
      "canWithdraw": true
    }
  ],
  "nextCursor": "eyJpZCI6..."
}
```

Language of `purposeTitle` and `purposeDescription` follows `languageCode` parameter.

---

### 1.4 Get Consent Artifact (Audit)

**Use case:** DF retrieving ECDSA-signed artifact for DPBI submission.

```
GET /v1/consent/{artifactId}
```

**Response 200 OK:**

```json
{
  "artifactId": "a1b2c3d4-...",
  "tenantId": "550e8400-...",
  "purposeId": "7c3e1d2a-...",
  "dpIdentifierHash": "sha256:abc123...",
  "action": "GRANTED",
  "languageCode": "hi",
  "noticeText": "Meesho आपको ऑफर और नई चीज़ों के बारे में SMS भेजने के लिए...",
  "collectionMethod": "WIDGET",
  "signature": "MEYCIQDmF3J...",
  "signingKeyId": "arn:aws:kms:...",
  "createdAt": "2026-04-22T10:30:00.123Z",
  "dataResidencyRegion": "ap-south-1",
  "verificationUrl": "https://verify.truststack.in/consent/a1b2c3d4"
}
```

---

## 2. Widget Rendering

### 2.1 Get Widget Data (Server-side render)

**Use case:** DF backend fetching consent widget content to render in their app.

```
GET /v1/widget/consent?tenantId={id}&purposeId={id}&languageCode={code}
```

**Response 200 OK:**

```json
{
  "purposeId": "7c3e1d2a-...",
  "languageCode": "hi",
  "textDirection": "ltr",
  "fontFamily": "Noto Sans Devanagari",
  "fontUrl": "https://fonts.truststack.in/NotoSansDevanagari.woff2",
  "content": {
    "title": "सहमति दें",
    "businessName": "Meesho",
    "noticeText": "Meesho आपसे यह डेटा इस उद्देश्य के लिए माँग रहा है:",
    "purposeTitle": "मार्केटिंग संदेश",
    "purposeDescription": "आपको ऑफर और नई चीज़ों के बारे में SMS भेजना। यह आपकी पसंद पर आधारित होगा।",
    "dataCategories": ["मोबाइल नंबर", "खरीदारी का इतिहास"],
    "retentionPeriod": "2 साल तक",
    "yourRights": "आप किसी भी समय अपनी सहमति वापस ले सकते हैं",
    "thirdParties": null,
    "withdrawalText": "सहमति वापस लेने के लिए 'मना करूँगा' बटन दबाएँ"
  },
  "actions": {
    "grantLabel": "मैं सहमत हूँ ✓",
    "denyLabel": "मैं मना करता हूँ ✗",
    "learnMoreLabel": "और जानें",
    "languageChangeLabel": "भाषा बदलें"
  }
}
```

---

### 2.2 Widget JavaScript SDK

**Embed in DF's web app:**

```html
<!-- SahmatOS Consent Widget -->
<script src="https://cdn.truststack.in/sahmat/v1/widget.js" async></script>
<div
  id="sahmat-widget"
  data-tenant-id="550e8400-e29b-41d4-a716-446655440000"
  data-purpose-id="7c3e1d2a-8f9b-4a5c-b6e7-123456789abc"
  data-dp-identifier="+919876543210"
  data-dp-identifier-type="phone"
></div>
<script>
SahmatOS.init({
  onGrant: (artifact) => {
    console.log('Consent granted:', artifact.artifactId);
    // Enable the protected feature
  },
  onDeny: () => {
    console.log('Consent denied');
    // Do not process data
  },
  onLanguageChange: (langCode) => {
    console.log('User switched to:', langCode);
  }
});
</script>
```

---

## 3. Erasure Endpoints

### 3.1 Schedule Erasure

```
POST /v1/erasure
```

**Request Body:**

```json
{
  "tenantId": "550e8400-...",
  "dpIdentifier": "+919876543210",
  "dpIdentifierType": "phone",
  "purposeId": "7c3e1d2a-...",
  "triggeredBy": "CONSENT_WITHDRAWN",
  "notificationLanguage": "hi"
}
```

| `triggeredBy` value | When to use |
|--------------------|-------------|
| `CONSENT_WITHDRAWN` | DP withdrew consent |
| `PURPOSE_FULFILLED` | Processing purpose is complete |
| `DP_REQUEST` | Direct right-to-erasure request |
| `CLASS_TIMELINE` | Third Schedule class deadline reached |

**Response 202 Accepted:**

```json
{
  "erasureId": "e7f8g9h0-...",
  "status": "SCHEDULED",
  "scheduledAt": "2026-04-22T10:30:00Z",
  "notificationDueAt": "2026-04-22T10:30:00Z",
  "executeAt": "2026-04-22T10:30:00Z",
  "dfClass": "STANDARD",
  "temporalWorkflowId": "erasure-e7f8g9h0-..."
}
```

Note: `executeAt` equals `scheduledAt` for `STANDARD` class DFs. For `ECOMMERCE_GT_2CR`, `GAMING_GT_50L`, `SOCIAL_GT_2CR`, it will be up to 3 years from last activity.

---

### 3.2 Get Erasure Status

```
GET /v1/erasure/{erasureId}
```

**Response 200 OK:**

```json
{
  "erasureId": "e7f8g9h0-...",
  "status": "COMPLETED",
  "scheduledAt": "2026-04-22T10:30:00Z",
  "notificationSentAt": "2026-04-24T10:30:00Z",
  "executedAt": "2026-04-26T10:32:15Z",
  "certificate": {
    "certificateId": "cert-...",
    "deletionTargets": [
      {"system": "tenant_database", "status": "deleted"},
      {"system": "razorpay", "status": "deleted"},
      {"system": "tally", "status": "deleted"}
    ],
    "signature": "MEYCIQDmF3J...",
    "issuedAt": "2026-04-26T10:32:15Z"
  }
}
```

---

## 4. Breach Notification

### 4.1 Trigger Breach Workflow

```
POST /v1/breach
```

**Request Body:**

```json
{
  "tenantId": "550e8400-...",
  "detectedAt": "2026-04-22T08:00:00Z",
  "breachType": "UNAUTHORISED_ACCESS",
  "breachDescription": "Database credentials leaked in public GitHub repository",
  "affectedDataCategories": ["phone", "email", "purchase_history"],
  "estimatedAffectedCount": 15000,
  "notificationLanguages": ["hi", "ta", "bn"]
}
```

**Response 202 Accepted:**

```json
{
  "breachId": "b1c2d3e4-...",
  "status": "NOTIFYING",
  "certInDeadline": "2026-04-22T14:00:00Z",
  "dpbiDeadline": "2026-04-25T08:00:00Z",
  "temporalWorkflowId": "breach-b1c2d3e4-..."
}
```

**IMPORTANT:** This endpoint triggers a time-critical workflow (6hr CERT-In deadline). Calling it triggers immediate automated notifications. Call only for genuine security incidents.

---

### 4.2 Get Breach Status

```
GET /v1/breach/{breachId}
```

**Response 200 OK:**

```json
{
  "breachId": "b1c2d3e4-...",
  "status": "DPBI_SUBMITTED",
  "detectedAt": "2026-04-22T08:00:00Z",
  "certInSubmittedAt": "2026-04-22T13:45:00Z",
  "certInReportId": "CERT-IN-2026-04-22-0042",
  "dpbiSubmittedAt": "2026-04-25T07:30:00Z",
  "dpbiReportId": "DPBI-2026-04-0042",
  "dpNotificationCount": 15000,
  "dpNotificationSentAt": "2026-04-22T10:00:00Z"
}
```

---

## 5. Audit Endpoints

### 5.1 Export Audit Log

**Use case:** Compliance team preparing DPBI annual report.

```
GET /v1/audit?tenantId={id}&from={iso8601}&to={iso8601}&eventType={type}&format={format}
```

| Parameter | Default | Notes |
|-----------|---------|-------|
| `from` | 1 year ago | ISO 8601 |
| `to` | now | ISO 8601 |
| `eventType` | all | Filter: `consent.granted`, `erasure.executed`, etc. |
| `format` | `json` | `json` \| `csv` \| `dpbi_xml` (DPBI submission format) |
| `cursor` | | Pagination cursor |

**Response 200 OK (`format=json`):**

```json
{
  "total": 42000,
  "items": [
    {
      "auditId": "...",
      "eventType": "consent.granted",
      "tenantId": "550e8400-...",
      "dpIdentifierHash": "sha256:abc123...",
      "purposeId": "7c3e1d2a-...",
      "languageCode": "hi",
      "collectionMethod": "WIDGET",
      "createdAt": "2026-04-22T10:30:00Z"
    }
  ],
  "nextCursor": "eyJpZCI6..."
}
```

---

## 6. Webhooks

SahmatOS sends webhooks to the DF's `webhook_url` for important events.

### 6.1 Webhook Payload Structure

```json
{
  "webhookId": "wh-...",
  "eventType": "consent.withdrawn",
  "tenantId": "550e8400-...",
  "occurredAt": "2026-04-22T10:30:00Z",
  "data": {
    "artifactId": "a1b2c3d4-...",
    "dpIdentifierHash": "sha256:abc123...",
    "purposeId": "7c3e1d2a-...",
    "action": "WITHDRAWN"
  },
  "signature": "sha256=<HMAC-SHA256(payload, webhook_secret)>"
}
```

### 6.2 Webhook Events

| Event | Trigger | Action required by DF |
|-------|---------|----------------------|
| `consent.granted` | New consent collected | Enable processing |
| `consent.withdrawn` | Consent withdrawn | **Stop all processing immediately** |
| `erasure.notification_sent` | 48hr pre-deletion notice sent | Prepare for deletion |
| `erasure.completed` | Data deleted, certificate ready | Retrieve certificate |
| `breach.cert_in_submitted` | CERT-In report filed | Inform internal stakeholders |
| `breach.dpbi_submitted` | DPBI report filed | |

### 6.3 Webhook Verification

```typescript
// Verify webhook authenticity
import { createHmac } from 'crypto';

function verifyWebhook(payload: string, signature: string, secret: string): boolean {
  const expected = `sha256=${createHmac('sha256', secret).update(payload).digest('hex')}`;
  return signature === expected; // Use timing-safe comparison in production
}
```

---

## 7. Rate Limits

| Endpoint Group | Limit | Window |
|---------------|-------|--------|
| `POST /v1/consent` | 1,000 req | per minute per tenant |
| `GET /v1/consent/check` | 10,000 req | per minute per tenant |
| `POST /v1/erasure` | 100 req | per minute per tenant |
| `POST /v1/breach` | 10 req | per hour per tenant |
| `GET /v1/audit` | 60 req | per minute per tenant |
| Widget render | 5,000 req | per minute per tenant |

Rate limit headers returned on all responses:
```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 847
X-RateLimit-Reset: 1745321460
```

---

## 8. Error Format

All errors follow RFC 9457 (Problem Details for HTTP APIs):

```json
{
  "type": "https://api.truststack.in/errors/invalid-purpose",
  "title": "Invalid Purpose",
  "status": 400,
  "detail": "Purpose 7c3e1d2a-... does not exist or is not approved for tenant 550e8400-...",
  "instance": "/v1/consent",
  "requestId": "req-abc123",
  "timestamp": "2026-04-22T10:30:00Z"
}
```

---

## 9. Idempotency

`POST /v1/consent` and `POST /v1/erasure` support idempotency via `X-Request-Id` header.

- If the same `X-Request-Id` is seen within 24 hours, return the original response (status 200 with original body, not 201)
- After 24 hours, the idempotency key expires and the request would be processed as new

---

## 10. Language Override

Any endpoint that returns user-facing content (widget data, consent notices, error messages for consumer-facing flows) respects the language preference in this order:

1. `X-DP-Language` header (explicit override)
2. `languageCode` query/body parameter
3. `Accept-Language` header
4. `X-Device-Locale` header
5. DP's registered language preference (from `dp_identifiers.preferred_language`)
6. Tenant's `default_language`
7. `hi` (Hindi — national link language fallback)

Consumer-facing error messages are returned in the detected language. API errors for DF backends are always in English.
