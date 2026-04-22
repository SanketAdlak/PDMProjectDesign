# System Design — SahmatOS Consent Vault

**Version:** 1.0  
**Date:** 2026-04-22  

---

## 1. System Context

SahmatOS is Module 3 of TrustStack. It operates as infrastructure — a set of APIs, event streams, and workflows that Data Fiduciaries (businesses) embed into their products to collect, manage, and prove consent from Data Principals (users).

### 1.1 Actors

| Actor | Role |
|-------|------|
| Data Principal (DP) | The individual whose personal data is being processed. Interacts via consent widget, WhatsApp, or consumer dashboard. |
| Data Fiduciary (DF) | The business collecting and processing personal data. Integrates SahmatOS via SDK or API. |
| TrustStack | Registered Consent Manager (Section 2(g) DPDPA). Operates SahmatOS infrastructure. |
| DPBI | Data Protection Board of India. Regulator that may audit consent records or receive breach reports. |
| CERT-In | Indian Computer Emergency Response Team. Receives 6-hour incident reports. |

### 1.2 Context Diagram

```
                    ┌──────────────────────────────────────────────┐
                    │              TrustStack Platform              │
                    │                                               │
     DP Widget      │  ┌──────────────┐     ┌──────────────────┐  │
   ───────────────► │  │  SahmatOS    │────►│  Consent Vault   │  │
     WhatsApp       │  │  API Gateway │     │  (Ledger + Vault) │  │
   ───────────────► │  └──────┬───────┘     └──────────────────┘  │
     Consumer App   │         │                      │             │
   ───────────────► │         │              ┌───────▼──────────┐  │
                    │         │              │  Erasure Engine  │  │
     DF Backend     │         │              └───────┬──────────┘  │
   ───────────────► │         │                      │             │
     DF Webhook     │         │              ┌───────▼──────────┐  │
   ◄─────────────── │         │              │  Breach Workflow │  │
                    │         │              └──────────────────┘  │
                    │         │                                     │
                    └─────────┼─────────────────────────────────────┘
                              │
                    ┌─────────▼─────────┐
                    │  Module 2         │  (separate VPC/account)
                    │  Discovery Engine │  CANNOT access Consent Vault
                    └───────────────────┘
```

---

## 2. Architecture Overview

### 2.1 Architectural Style: Modular Monolith

SahmatOS is built as a **modular monolith** within its own repository — a single deployable unit divided into clearly bounded modules (Consent, Erasure, Breach, Language, Auth). Cross-module interaction happens only via:
1. Internal service calls (TypeScript interfaces, never HTTP within the monolith)
2. Kafka events (for asynchronous workflows that span consent → erasure)

**Why not microservices?**

| Factor | Modular Monolith | Microservices |
|--------|-----------------|--------------|
| Team size (v1) | 5-8 engineers — monolith wins | Adds distributed systems overhead |
| Regulatory need | All consent data in one DB (audit) | Cross-service joins are hard to audit |
| Operational | Single deployment, simpler | Many services, complex orchestration |
| Extraction path | Clear module boundaries | Start here, extract later if needed |

ADR-01 in `docs/SPEC.md` covers this decision in full.

### 2.2 Deployment Target

- **Primary:** AWS ap-south-1 (Mumbai) — all data stored here
- **DR:** GCP asia-south1 (Mumbai) — Temporal.io workers replicated here
- **No data ever leaves Indian cloud regions** (Section 16 DPDPA)

---

## 3. Component Architecture

### 3.1 Service Map

```
┌─────────────────────────────────────────────────────────────────────┐
│                         SahmatOS Service                             │
│                                                                       │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────────┐ │
│  │   API Layer  │  │  Language    │  │     Auth & Tenant Layer    │ │
│  │  (Express +  │  │   Engine     │  │  (JWT + API Keys + RBAC)  │ │
│  │  OpenAPI 3.1)│  │  (i18n +     │  │                            │ │
│  └──────┬───────┘  │  Translation)│  └────────────┬───────────────┘ │
│         │          └──────┬───────┘               │                  │
│         ▼                 │                        │                  │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    Domain Services                            │   │
│  │                                                               │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌──────────────────────┐ │   │
│  │  │  Consent    │  │  Erasure    │  │  Breach Notification │ │   │
│  │  │  Service    │  │  Service    │  │  Service             │ │   │
│  │  │             │  │             │  │                      │ │   │
│  │  │ - grant()   │  │ - schedule()│  │ - trigger()          │ │   │
│  │  │ - withdraw()│  │ - notify()  │  │ - certIn()           │ │   │
│  │  │ - check()   │  │ - execute() │  │ - dpbi()             │ │   │
│  │  │ - list()    │  │ - certify() │  │                      │ │   │
│  │  └──────┬──────┘  └──────┬──────┘  └──────────┬───────────┘ │   │
│  │         │                │                      │             │   │
│  └─────────┼────────────────┼──────────────────────┼─────────────┘   │
│            │                │                      │                  │
│  ┌─────────▼────────────────▼──────────────────────▼─────────────┐   │
│  │                    Infrastructure Layer                         │   │
│  │                                                                 │   │
│  │  ┌──────────┐  ┌───────┐  ┌──────────┐  ┌─────────────────┐  │   │
│  │  │PostgreSQL│  │ Redis │  │  Kafka   │  │   Temporal.io   │  │   │
│  │  │ (Consent │  │(Cache)│  │  (Event  │  │  (Erasure +     │  │   │
│  │  │  Ledger) │  │       │  │   Bus)   │  │   Breach WF)    │  │   │
│  │  └──────────┘  └───────┘  └──────────┘  └─────────────────┘  │   │
│  │                                                                 │   │
│  │  ┌────────────────────────────────────────────────────────┐    │   │
│  │  │  AWS KMS (ap-south-1) — ECDSA P-256 per-tenant keys   │    │   │
│  │  └────────────────────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 API Layer

**Technology:** Express.js on Node.js 20 LTS, TypeScript ESM, OpenAPI 3.1

**Entry points:**
1. `POST /v1/consent` — Grant or withdraw consent (called by DF backend or widget)
2. `GET /v1/consent/check` — Check if DP has valid consent for a purpose (high-frequency, cached)
3. `GET /v1/consent` — List consent records for a DP (consumer dashboard)
4. `POST /v1/erasure` — Schedule erasure for a DP
5. `GET /v1/erasure/:id/status` — Erasure status
6. `POST /v1/breach` — Trigger breach notification workflow
7. `GET /v1/audit` — Audit log export (DF-facing)
8. `GET /v1/widget/render` — Render consent widget HTML/JSON for a purpose

All endpoints documented in `docs/API_SPEC.md`.

### 3.3 Consent Service

Responsibilities:
- Validate consent request (purpose exists, DF is authorized, DP not a minor without guardian consent)
- Detect language (from headers, locale, or explicit param)
- Retrieve translated consent notice in DP's language
- Generate ECDSA-signed consent artifact
- Write to PostgreSQL append-only ledger
- Emit `consent.granted` or `consent.withdrawn` Kafka event
- Update Redis cache for consent status

```typescript
// consent/service.ts — key flow
export class ConsentService {
  async grant(req: ConsentGrantRequest): Promise<ConsentArtifact> {
    // 1. Validate tenant and purpose
    const purpose = await this.purposeRepo.findById(req.purposeId);
    const tenant = await this.tenantRepo.findById(req.tenantId);
    
    // 2. Check age — route to parental consent if under 18 (Section 9)
    if (await this.isMinor(req.dpIdentifier)) {
      return this.parentalConsentService.initiateGuardianFlow(req);
    }
    
    // 3. Get consent notice in DP's language
    const language = await this.languageEngine.detect(req);
    const notice = await this.translationService.getConsentNotice(
      purpose.id, language.code
    );
    
    // 4. Sign consent artifact
    const artifact = await this.signer.sign({
      tenantId: req.tenantId,
      dpIdentifierHash: await this.hashDpIdentifier(req.dpIdentifier),
      purposeId: req.purposeId,
      purposeDescription: notice.description,
      action: 'GRANTED',
      languageCode: language.code,
      collectionMethod: req.collectionMethod,
      timestamp: new Date().toISOString(),
    });
    
    // 5. Persist (append-only)
    await this.consentRepo.insert(artifact);
    
    // 6. Emit event
    await this.kafka.emit('consent.granted', artifact);
    
    // 7. Update cache
    await this.cache.setConsentStatus(
      req.tenantId, req.dpIdentifier, req.purposeId, 'ACTIVE'
    );
    
    return artifact;
  }
}
```

### 3.4 Language Engine

Responsibilities:
- Language detection (HTTP headers → device locale → IP → tenant default → Hindi)
- Font manifest generation (which fonts to load for this language)
- Translation retrieval (from cache or DB)
- RTL flag determination

```typescript
// language/engine.ts
export class LanguageEngine {
  async detect(req: Request): Promise<LanguageContext> {
    // Priority chain
    const code = 
      this.fromExplicitParam(req) ||
      this.fromAcceptLanguage(req.headers['accept-language']) ||
      this.fromDeviceLocale(req.headers['x-device-locale']) ||
      await this.fromIpGeolocation(req.ip) ||
      this.fromTenantDefault(req.tenantId) ||
      'hi'; // Hindi fallback
    
    const normalised = LANGUAGE_CODE_MAP[code] ?? 'hi';
    
    return {
      code: normalised,
      script: SCRIPT_MAP[normalised],
      direction: RTL_LANGUAGES.includes(normalised) ? 'rtl' : 'ltr',
      fontFamily: FONT_MAP[normalised].fontFamily,
      fontUrl: FONT_MAP[normalised].url,
    };
  }
}
```

### 3.5 Erasure Service (with Temporal.io)

The Erasure Service wraps Temporal.io workflows. All erasure execution happens inside durable workflows — not ad-hoc job queues.

```typescript
// erasure/workflows.ts — Temporal.io workflow
export async function ErasureWorkflow(input: ErasureInput): Promise<ErasureCertificate> {
  // Determine class-specific timeline (Third Schedule)
  const schedule = await determineErasureSchedule(input.tenantId, input.purposeId);
  
  if (schedule.delayMs > 0) {
    // Wait until 48hr before deletion (may be up to 3 years for E-commerce/Gaming/Social)
    await sleep(schedule.executeAt - 48 * HOURS);
  }
  
  // Send 48hr pre-deletion notification to DP in their language
  await notifyDataPrincipal({
    dpIdentifier: input.dpIdentifier,
    language: input.language,
    scheduledDeletion: schedule.executeAt,
    purpose: input.purposeDescription,
  });
  
  // Wait for deletion time
  await sleep(schedule.executeAt - Date.now());
  
  // Propagate deletion to all connected data stores
  const deletionResults = await Promise.all([
    deleteFromTenantDatabase(input),
    deleteFromConnectedIntegrations(input),
    revokeConsentManagerRecords(input),
  ]);
  
  // Generate verifiable deletion certificate (ECDSA signed)
  const certificate = await generateDeletionCertificate({
    ...input,
    deletionTimestamp: new Date().toISOString(),
    results: deletionResults,
  });
  
  // Store certificate (retained for 7 years per Rule 4)
  await storeCertificate(certificate);
  await emitKafkaEvent('erasure.executed', certificate);
  
  return certificate;
}

// Determine erasure timeline — Third Schedule class rules
async function determineErasureSchedule(tenantId: string, purposeId: string): Promise<ErasureSchedule> {
  const tenant = await getTenantClassification(tenantId);
  
  switch (tenant.dfClass) {
    case 'ECOMMERCE_GT_2CR':
    case 'GAMING_GT_50L':
    case 'SOCIAL_GT_2CR':
      // Third Schedule: 3 years from last transaction/login
      const lastActivity = await getLastActivity(tenantId, purposeId);
      return { executeAt: addYears(lastActivity, 3), class: tenant.dfClass };
    
    default:
      // All other DFs: erase immediately (when purpose fulfilled or withdrawn)
      return { executeAt: Date.now(), class: 'IMMEDIATE' };
  }
}
```

### 3.6 Breach Notification Service (with Temporal.io)

```typescript
// breach/workflows.ts
export async function BreachNotificationWorkflow(input: BreachInput): Promise<void> {
  // Log breach event immediately (immutable)
  await logBreachEvent(input);
  
  // CERT-In: 6-hour initial report
  const certInDeadline = addHours(input.detectedAt, 6);
  await submitCertInReport({
    ...input,
    deadline: certInDeadline,
    reportType: 'INITIAL',
  });
  
  // Notify affected Data Principals in their languages
  await notifyAffectedDataPrincipals(input);
  
  // DPBI: 72-hour detailed report
  await sleep(certInDeadline - Date.now()); // Use remaining time productively
  const dpbiDeadline = addHours(input.detectedAt, 72);
  await sleep(dpbiDeadline - Date.now() - 30 * MINUTES); // Start 30min before deadline
  
  const dpbiReport = await generateDpbiReport(input);
  await submitDpbiReport(dpbiReport);
  
  await logComplianceEvent('breach.notification.completed', {
    certInSubmitted: true,
    dpbiSubmitted: true,
    affectedPrincipalsNotified: input.affectedCount,
  });
}
```

---

## 4. Data Flow: Consent Grant

```
DF Widget (React)          API Gateway         Consent Service        Infrastructure
      │                         │                    │                      │
      │ POST /v1/consent        │                    │                      │
      │ {tenantId, dpId,        │                    │                      │
      │  purposeId, action,     │                    │                      │
      │  language, method}      │                    │                      │
      │────────────────────────►│                    │                      │
      │                         │ validateJWT        │                      │
      │                         │ rateLimitCheck     │                      │
      │                         │────────────────────►                      │
      │                         │                    │ validatePurpose       │
      │                         │                    │────────────────────── ►PostgreSQL
      │                         │                    │◄─────────────────────-│purpose found
      │                         │                    │                      │
      │                         │                    │ detectLanguage        │
      │                         │                    │ getNoticeTranslation  │
      │                         │                    │────────────────────── ►Redis (cache hit)
      │                         │                    │◄──────────────────────│
      │                         │                    │                      │
      │                         │                    │ signArtifact          │
      │                         │                    │────────────────────── ►AWS KMS
      │                         │                    │◄──────────────────────│signature
      │                         │                    │                      │
      │                         │                    │ INSERT consent_record │
      │                         │                    │────────────────────── ►PostgreSQL
      │                         │                    │◄──────────────────────│committed
      │                         │                    │                      │
      │                         │                    │ emit consent.granted  │
      │                         │                    │────────────────────── ►Kafka
      │                         │                    │                      │
      │                         │                    │ SET consent:cache key │
      │                         │                    │────────────────────── ►Redis
      │                         │                    │                      │
      │                         │◄───────────────────│                      │
      │◄────────────────────────│                    │                      │
      │ 201 Created             │                    │                      │
      │ {artifactId, signature, │                    │                      │
      │  timestamp, language}   │                    │                      │
```

**End-to-end latency target:** p50 < 200ms, p99 < 800ms

---

## 5. Data Flow: Consent Check (Hot Path)

The consent check endpoint is called by DF backends before every data processing operation. Must be extremely fast.

```
DF Backend                 API Gateway         Cache Layer         DB (fallback)
    │                           │                   │                   │
    │ GET /v1/consent/check     │                   │                   │
    │ ?tenantId=X               │                   │                   │
    │ &dpId=Y                   │                   │                   │
    │ &purposeId=Z              │                   │                   │
    │──────────────────────────►│                   │                   │
    │                           │ validateAPIKey    │                   │
    │                           │──────────────────►│                   │
    │                           │                   │ GET consent:X:Y:Z │
    │                           │                   │──────────────────►│(Redis)
    │                           │                   │◄──────────────────│HIT: {status}
    │                           │◄──────────────────│                   │
    │◄──────────────────────────│                   │                   │
    │ 200 OK {status: ACTIVE}   │                   │                   │
    │ (p50: ~30ms)              │                   │                   │
```

**Cache structure:** `consent:{tenantId}:{dpIdHash}:{purposeId}` → `{status, grantedAt, expiresAt}`  
**Cache TTL:** 5 minutes (balances freshness with performance; withdrawal propagates via Kafka consumer)

---

## 6. Kafka Event Schema

### 6.1 Topic Definitions

| Topic | Retention | Partitions | Purpose |
|-------|-----------|------------|---------|
| `consent.granted` | 7 years | 12 | New consent grants |
| `consent.withdrawn` | 7 years | 12 | Consent withdrawals |
| `erasure.scheduled` | 7 years | 6 | Erasure workflow triggered |
| `erasure.executed` | 7 years | 6 | Deletion complete + certificate |
| `breach.detected` | 7 years | 3 | Breach event received |
| `breach.notification.sent` | 7 years | 3 | Notification sent to DP/DPBI/CERT-In |

### 6.2 Event Schema (consent.granted)

```typescript
interface ConsentGrantedEvent {
  eventId: string;          // UUID
  eventType: 'consent.granted';
  schemaVersion: '1.0';
  tenantId: string;
  dpIdentifierHash: string; // HMAC-SHA256(dpIdentifier, tenantSecret)
  purposeId: string;
  purposeCode: string;      // Human-readable purpose key
  languageCode: string;     // BCP 47
  collectionMethod: 'WIDGET' | 'API' | 'WHATSAPP' | 'VERBAL_AADHAAR';
  artifactId: string;
  signature: string;        // ECDSA P-256 base64
  signingKeyId: string;     // AWS KMS key ARN
  timestamp: string;        // ISO 8601 UTC
  schemaData: {
    dataCategories: string[];
    retentionPeriod: string;
    processingLocations: string[]; // 'IN-MH' (Maharashtra) etc.
  };
}
```

---

## 7. Security Boundary: Dual-Role Isolation

DPDPA Section 2(g) prohibits a Consent Manager from being a Data Processor for the same Data Principal. This is enforced architecturally:

```
┌──────────────────────────────────────────┐
│     AWS Account: TrustStack-Consent      │  VPC: 10.0.0.0/16
│                                           │
│  SahmatOS (Consent Vault, Erasure,       │
│  Breach Notification)                    │
│                                           │
│  ⛔ NO network route to Discovery VPC   │
└──────────────────────────────────────────┘
           ↕ NO VPC peering
┌──────────────────────────────────────────┐
│     AWS Account: TrustStack-Discovery    │  VPC: 10.1.0.0/16
│                                           │
│  Module 2 (AI Discovery Engine,          │
│  PII Scanner, Contract Audit)            │
│                                           │
│  ⛔ Cannot read consent_records table   │
└──────────────────────────────────────────┘
```

If a user (DP) is in the Consent Vault, the Discovery Engine has no path to see their consent records, and vice versa. Enforced by:
1. Separate AWS accounts (IAM cannot cross accounts without explicit trust)
2. No VPC peering between the two accounts
3. No shared databases or message queues
4. Audit rule: any AWS CloudTrail event showing cross-account access triggers immediate alert

---

## 8. Tenant Isolation

Each Data Fiduciary (business customer) is a tenant. Tenant data is isolated:

### 8.1 Database Isolation
- All consent records are partitioned by `tenant_id` (PostgreSQL partition key)
- Row-level security (RLS) policies: every query automatically filtered to authenticated tenant
- Separate KMS keys per tenant (per Rule 4 requirement for independent auditability)

### 8.2 API Isolation
- Tenant identified by API key (hashed) + JWT claims
- API key has tenant scope: cannot access another tenant's data
- Rate limits: per-tenant (DFs cannot starve each other)

### 8.3 KMS Key Management
- Each tenant gets a dedicated KMS symmetric key for data encryption
- Each tenant gets a dedicated KMS asymmetric key pair (ECDSA P-256) for consent signing
- Key rotation: annual, automated via KMS rotation policy
- Key deletion: only on tenant offboarding + 30-day soft-delete window

---

## 9. Caching Strategy

| Cache Key | Data | TTL | Invalidation |
|-----------|------|-----|-------------|
| `consent:{tenant}:{dpHash}:{purpose}` | Consent status | 5 min | Kafka `consent.withdrawn` consumer |
| `purpose:{tenant}:{purposeId}` | Purpose definition + translations | 1 hour | Admin cache invalidation |
| `lang:notice:{purposeId}:{langCode}` | Translated consent notice | 24 hours | Post-review deployment |
| `tenant:{tenantId}` | Tenant config | 10 min | Config update event |
| `font:{langCode}` | Font URL + metadata | 1 week | CDN fingerprinted |

---

## 10. Observability

### 10.1 Structured Logging (PII-Safe)

```typescript
// logging/logger.ts
// NEVER log personal data. Use hashes and IDs only.
logger.info('consent.granted', {
  tenantId: req.tenantId,
  dpIdentifierHash: hashDpIdentifier(req.dpIdentifier), // ✅ safe
  purposeId: req.purposeId,
  languageCode: req.languageCode,
  collectionMethod: req.collectionMethod,
  durationMs: Date.now() - req.startTime,
  // dpIdentifier: req.dpIdentifier,  // ❌ NEVER log raw PII
});
```

### 10.2 Metrics

| Metric | Type | Alert Threshold |
|--------|------|-----------------|
| `consent_grant_duration_ms` | Histogram | p99 > 1000ms |
| `consent_check_duration_ms` | Histogram | p99 > 300ms |
| `consent_grant_errors_total` | Counter | > 5/min |
| `erasure_workflow_failures_total` | Counter | > 0 (zero tolerance) |
| `breach_notification_delay_ms` | Gauge | > 200ms from trigger |
| `kafka_consumer_lag_seconds` | Gauge | > 30s |
| `cache_hit_rate` | Gauge | < 90% (consent check) |
| `kms_signing_duration_ms` | Histogram | p99 > 200ms |

### 10.3 Distributed Tracing

All requests carry `X-Trace-Id` header. Traces captured via OpenTelemetry → AWS X-Ray.

Trace spans:
- API gateway → service routing
- Language detection
- KMS signing call
- DB write
- Kafka emit
- Cache update

---

## 11. Disaster Recovery

| Component | RPO | RTO | Strategy |
|-----------|-----|-----|---------|
| PostgreSQL (Consent Ledger) | 0 (synchronous replica) | 15 min | Multi-AZ RDS, automatic failover |
| Redis (Cache) | 5 min | 5 min | ElastiCache cluster mode |
| Kafka (MSK) | 0 (replication factor 3) | Transparent | MSK multi-AZ |
| Temporal.io | 0 (workflow state replicated) | 5 min | Self-hosted cluster, 2 AZs |
| API Servers | N/A (stateless) | < 1 min | ECS Fargate, auto-replace |
| KMS Keys | N/A (AWS SLA) | AWS SLA | AWS KMS multi-region replicas |

**India-only constraint:** DR site is GCP asia-south1 (also Mumbai). No cross-border replication.

---

## 12. Scalability

### 12.1 Scaling Model

| Component | Scaling Type | Trigger |
|-----------|-------------|---------|
| API servers | Horizontal (ECS) | CPU > 60% or request queue > 100 |
| PostgreSQL | Vertical + read replicas | Read replica for consent checks |
| Redis | Cluster sharding | Memory > 70% |
| Kafka | Partition scaling | Consumer lag > 10s |
| Temporal | Worker scaling | Task queue depth > 1000 |

### 12.2 Peak Load Design

Expected peak: major Indian sales events (Big Billion Day, Great Indian Sale) — 10,000 consent events/second.

- API: 50 ECS tasks × 200 req/s = 10,000 req/s capacity
- DB: Consent writes bypass read replica (append-only, always primary). Consent checks hit Redis (cache) → read replica (miss).
- Kafka: 12 partitions × ~1000 events/s/partition = 12,000 events/s headroom
- KMS: Batch signing with connection pool (KMS allows 100 RPS/key; 100 tenant keys = 10,000 signs/s theoretical)
