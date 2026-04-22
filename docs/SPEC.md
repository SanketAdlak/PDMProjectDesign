# TrustStack — System Architecture & Design Specification

**Version:** 1.0  
**Date:** April 2026  
**Classification:** Confidential — Internal & Investor Use  
**Status:** Draft for Review

---

## Table of Contents

1. [Product Understanding](#1-product-understanding)
2. [System Overview — High-Level Design](#2-system-overview--high-level-design)
3. [Architecture Design](#3-architecture-design)
4. [Consent Management System Design](#4-consent-management-system-design)
5. [Data Discovery System Design](#5-data-discovery-system-design)
6. [Low-Level Design — Key Components](#6-low-level-design--key-components)
7. [Architecture Decision Records](#7-architecture-decision-records)
8. [Non-Functional Requirements](#8-non-functional-requirements)
9. [Risks and Open Questions](#9-risks-and-open-questions)
10. [Visualisation Plan](#10-visualisation-plan)

---

## 1. Product Understanding

### 1.1 What TrustStack Does

TrustStack is India's first AI-powered DPDPA compliance platform built for SMEs and startups. It operates as a **Registered Consent Manager** under Section 2(g) of the Digital Personal Data Protection Act (DPDPA), 2023 — the legal intermediary between Data Principals (individuals) and Data Fiduciaries (businesses).

The platform mirrors the proven RBI Account Aggregator model: a regulated, consent-centric intermediary that cannot itself access the personal data it manages consent for. TrustStack solves compliance end-to-end across three lifecycle stages: awareness, assessment, and automated implementation.

The core value proposition is threefold:
- **Regulatory completeness** — Built ground-up for DPDPA, not adapted from GDPR. Handles the "no legitimate interests" clause that invalidates most Western compliance tools in India.
- **Vernacular-first** — All consent flows, notices, and audit materials in 22 Eighth Schedule languages. Designed in Hindi and Tamil first.
- **Indian stack native** — Deep integrations with Tally, Zoho, Razorpay, and WhatsApp Business — the actual tools India's 63M+ SMEs use.

### 1.2 Target Users and Personas

| Persona | Description | Primary Need | Entry Module |
|---|---|---|---|
| **Founder / Founding Team** | Pre-seed to Series A startup, 2–15 person team, no dedicated privacy officer | Fast, cheap DPDPA compliance without legal overhead | Module 1 (Free Trust Check) |
| **SME Owner (Tier 2–3)** | Traditional business digitising operations (Tally, Zoho), Hindi or regional-language primary | Understand obligations in local language, avoid penalties | Module 1 via PrivacyDost |
| **CTO / Engineering Lead** | Needs to implement consent collection and erasure in existing product | Drop-in SDK, clear API contracts, audit trail | Module 3 (Consent Vault SDK) |
| **Compliance / Legal** | Manages vendor DPAs, breach response, annual audit reports | Automated workflows, verifiable records | Module 3 (Regulatory War Room) |
| **Data Principal (End User)** | Individual whose data is being processed | Transparent consent, easy withdrawal, data portability | Consumer Dashboard (SahmatOS) |

### 1.3 Key Problems Solved

1. **The Awareness Gap** — 94% of India's top websites have no consent banner. SME founders lack basic awareness that their product is non-compliant.
2. **The Language Barrier** — DPDPA requires notices in the user's interface language. No existing tool supports 22 scheduled Indian languages natively.
3. **The Discovery Problem** — Most SMEs don't know where their PII lives. It's scattered across Tally, Zoho CRM, Razorpay, WhatsApp, and S3 buckets.
4. **The Implementation Cost** — OneTrust and Drata cost $7,500–$50,000/year. India's SMEs have ₹2–5 lakh total IT budgets.
5. **The Dual-Role Trap** — A consent tool that also processes the data creates an illegal dual role under DPDPA Rule 4. Existing SaaS tools don't understand this constraint.
6. **The Erasure Complexity** — Class-specific timelines, 48hr pre-deletion alerts, verifiable deletion certificates — no off-the-shelf tool handles this correctly.

### 1.4 Core Feature Set

- **Jargon Slayer** — AI translation of DPDPA obligations into Founder-speak, 22 languages
- **3-Minute Trust Check** — Free scanner: missing banners, ghost trackers, unencrypted PII in URLs, language gaps
- **AI Data Discovery Engine** — Sandboxed PII crawler across AWS/GCP/Zoho/Razorpay/WhatsApp → Data Flow Map
- **Multilingual Contract Audit** — NLP on vendor contracts, non-compliant clause flagging, DPA generation
- **Consent Vault (SahmatOS)** — API-first, ECDSA-signed, purpose-level consent with 22-language widgets and parental consent workflow
- **Erasure Engine** — Class-specific timeline management, 48hr alerts, cryptographic deletion certificates
- **Regulatory War Room** — 72hr DPBI breach workflow, 6hr CERT-In parallel reporting, annual audit generator

---

## 2. System Overview — High-Level Design

### 2.1 Architecture Overview

TrustStack is structured as three independently deployable product modules sharing a common platform layer. The modules correspond to the SME compliance journey and can be licensed individually or as a suite.

```
┌──────────────────────────────────────────────────────────────────────┐
│                         TRUSTSTACK PLATFORM                          │
│                                                                      │
│  ┌─────────────────┐  ┌──────────────────┐  ┌───────────────────┐  │
│  │   MODULE 1      │  │    MODULE 2      │  │    MODULE 3       │  │
│  │  Vernacular     │  │  Assessment &    │  │  Implementation   │  │
│  │  Awareness      │  │  Discovery       │  │  & Automation     │  │
│  │  (Free Tier)    │  │  (Revenue)       │  │  (The Shield)     │  │
│  │                 │  │                  │  │                   │  │
│  │ • Jargon Slayer │  │ • Discovery Eng  │  │ • Consent Vault   │  │
│  │ • Trust Check   │  │ • Contract Audit │  │ • Erasure Engine  │  │
│  └─────────────────┘  └──────────────────┘  │ • Regulatory WR   │  │
│                                              └───────────────────┘  │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    PLATFORM LAYER                             │   │
│  │  GraphQL API Bus │ Auth (OAuth 2.0) │ Tenant Management      │   │
│  │  Event Bus (Kafka) │ Audit Log │ Notification Service       │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                  DATA & INFRA LAYER                           │   │
│  │  PostgreSQL (append-only) │ Kafka │ Redis │ S3               │   │
│  │  AWS Mumbai + GCP Mumbai (consent data must be Indian)       │   │
│  └──────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘
```

**Critical architectural constraint:** Module 2 (Discovery) runs in a **fully sandboxed environment** with zero access to Module 3 (Consent Vault) data. This is a hard regulatory requirement under the dual-role prohibition (DPDPA Rule 4, First Schedule Part B).

### 2.2 Major Components

| Component | Responsibility | Module |
|---|---|---|
| **Trust Check Scanner** | External-facing, stateless scan of a URL against compliance criteria | M1 |
| **Jargon Slayer Engine** | LLM inference + translation of DPDPA clauses to 22 languages | M1 |
| **Discovery Sandbox** | Isolated PII crawl agent with read-only connector access | M2 |
| **Contract Audit Agent** | NLP-based analysis of vendor contracts, DPA generation | M2 |
| **Consent Vault API** | Core consent lifecycle management, ECDSA signing, SDK surface | M3 |
| **Erasure Orchestrator** | Temporal.io workflow managing class-specific erasure timelines | M3 |
| **Regulatory War Room** | Breach response workflow automation, audit report generation | M3 |
| **Consumer Dashboard** | Data Principal self-service portal: view, withdraw, download consents | M3 |
| **GraphQL Bus** | Internal API gateway, schema federation across modules | Platform |
| **Notification Service** | 48hr erasure alerts, breach notifications to DPs and DPBI | Platform |
| **Tenant Registry** | Multi-tenant configuration, plan limits, feature flags | Platform |

### 2.3 Data Flow Across Components

**Happy-path consent flow:**

```
Data Principal (User)
        │
        ▼
   Consent Widget (22-lang embed)
        │  HTTP POST /consent/grant
        ▼
   Consent Vault API
        │
        ├─► ECDSA Sign (P-256 key per tenant)
        │
        ├─► Kafka publish [consent.granted] event
        │
        ├─► PostgreSQL INSERT (append-only, immutable)
        │
        └─► Notification Service (confirmation to DP)

[Async]
Kafka Consumer
        │
        ├─► Erasure Orchestrator (register purpose + expiry)
        ├─► Audit Trail Service (compliance log)
        └─► Tenant webhook (inform Data Fiduciary's system)
```

**Discovery pipeline flow (sandboxed):**

```
Connector (read-only OAuth)
        │
        ▼
   Discovery Sandbox (isolated compute)
        │
        ├─► PII Detector (on-premise LLM, no data leaves sandbox)
        │
        ├─► Metadata Extractor (field names, locations, volumes)
        │
        └─► Data Flow Map (output: metadata only, not raw data)
                │
                ▼
        Module 2 Dashboard (tenant-facing)
                │
                ▼ (manual action by tenant)
        Consent Vault: register new data flows
```

### 2.4 External Integrations

| Integration | Protocol | Direction | Purpose | DPDPA Relevance |
|---|---|---|---|---|
| **Tally** | Tally XML + REST | Read (discovery) | Financial PII discovery, GL entries | Purpose-linked retention |
| **Zoho CRM/Books** | Zoho REST API, OAuth 2.0 | Read + Write | PII discovery, consent widget embed, cross-suite erasure | Vendor DPA, erasure propagation |
| **WhatsApp Business** | Meta Cloud API | Write | Consent request via WhatsApp, 1-tap withdrawal | Accessible consent channel, Rule 4 |
| **Razorpay** | Razorpay Webhooks + REST | Read | Payment/KYC consent tracking | Financial data processing consent |
| **DPBI Portal** | REST (Government API, TBD) | Write | Breach notification submission | Rule 7 obligation |
| **CERT-In** | REST / Email gateway | Write | Security incident reporting | Parallel 6hr reporting obligation |
| **Aadhaar eKYC** | UIDAI API (licensed) | Read | Parental age verification | Section 9 parental consent |

### 2.5 Multi-Tenancy

TrustStack is a multi-tenant SaaS platform with **logical isolation** at the database level and **cryptographic isolation** for consent artifacts.

- Each Data Fiduciary (tenant) has a unique **Tenant ID** and **ECDSA P-256 key pair**
- Consent records are partitioned by `tenant_id` with row-level security policies in PostgreSQL
- Tenant configuration (plan, feature flags, connected integrations, branding) is stored in the **Tenant Registry**
- At Enterprise tier, **dedicated schema** isolation is offered (separate PostgreSQL schema per tenant)
- **Cross-tenant data access is architecturally impossible by design** — no shared data stores between tenants

### 2.6 Scalability and Availability

- **Stateless API services** horizontally scale behind a load balancer (AWS ALB)
- **Kafka** decouples consent event ingestion from downstream processing — ingestion never blocks on erasure or audit
- **PostgreSQL read replicas** serve the Consumer Dashboard and audit report queries without touching the primary write path
- **Temporal.io** handles long-running erasure and breach workflows durably — restarts after failures without data loss
- **Regional failover**: Primary on AWS Mumbai, warm standby on GCP Mumbai. Consent records **never leave India** (Section 16 requirement)
- **Target uptime**: 99.9% for Consent Vault API (contractual SLA at Growth tier+)

---

## 3. Architecture Design

### 3.1 Architectural Style

TrustStack uses a **modular monolith with event-driven async boundaries**.

- Within each module, code is structured as a modular monolith (TypeScript ESM modules, co-deployed) for development velocity
- Between modules, communication is exclusively via **Kafka events** and **GraphQL** — no direct function calls or shared databases
- This gives the team startup-speed development while preserving the ability to extract services independently as scale demands it

**Rationale for not going microservices immediately:** The regulatory domain requires tight internal consistency (especially around consent state and erasure workflows). Microservices introduce distributed transaction complexity that would create compliance risk before the team has regulatory precedent to lean on. Re-evaluate at 50+ engineers or when module teams operate independently.

### 3.2 Service Boundaries and Responsibilities

```
Module 1: Awareness Services
  ├── trust-check-service      Stateless scanner, no data persistence
  └── jargon-slayer-service    LLM inference, translation cache

Module 2: Discovery Services (SANDBOXED — separate deployment boundary)
  ├── discovery-orchestrator   Job scheduling, connector management
  ├── pii-detector-service     On-premise LLM inference, processes raw data
  └── contract-audit-service   NLP pipeline for vendor contracts

Module 3: Compliance Services
  ├── consent-vault-api        Core consent CRUD, ECDSA signing
  ├── erasure-orchestrator     Temporal.io workflows, timeline management
  └── war-room-service         Breach workflows, audit report generation

Platform Services
  ├── graphql-gateway          Schema federation, rate limiting, auth
  ├── tenant-registry          Tenant config, plan management, feature flags
  ├── notification-service     Multi-channel (WhatsApp, email, SMS) alerts
  ├── auth-service             OAuth 2.0, JWT issuance, API key management
  └── audit-trail-service      Immutable event log, compliance query API
```

**Discovery Sandbox boundary** is a hard deployment isolation: separate VPC, no network routes to Consent Vault subnets, no shared IAM roles. This is not just a logical separation — it is enforced at the infrastructure level.

### 3.3 API Design Principles

- **OpenAPI 3.1** for all external-facing REST APIs (required by DPDPA for machine-readable consent records)
- **GraphQL** for internal service-to-service communication and the developer-facing Data Fiduciary API
- **Versioning**: URI versioning (`/v1/`, `/v2/`) for external APIs; schema evolution for GraphQL via additive changes + deprecation
- **Consent API is idempotent**: every consent operation is keyed on `(tenant_id, dp_identifier, purpose_id)` — duplicate requests are safely deduplicated
- **All mutation endpoints require HMAC request signing** at Enterprise tier (defends against replay attacks on consent records)
- **PII-safe error messages**: error responses never echo back personal data fields — only identifiers and error codes

### 3.4 Data Storage Strategy

| Store | Technology | Use Case | DPDPA Requirement Met |
|---|---|---|---|
| **Consent Ledger** | PostgreSQL append-only tables | Immutable consent events, 7-year retention | Rule 4 Part B: machine-readable, 7yr retention |
| **Audit Trail** | PostgreSQL (separate schema, insert-only trigger) | All platform events for compliance queries | Rule 4 Part B: audit-ready logging |
| **Discovery Metadata** | PostgreSQL (sandboxed schema) | Data flow maps, PII locations (no raw data) | Dual-role isolation |
| **Session/Cache** | Redis | API rate limiting, consent widget state, session tokens | — |
| **Blob Storage** | AWS S3 (Mumbai) | Signed consent artifacts (PDF), deletion certificates, audit reports | Section 16: Indian servers |
| **LLM Cache** | Redis / S3 | Translation cache for Jargon Slayer (same input → same output) | — |
| **Tenant Config** | PostgreSQL | Plan configuration, feature flags, integration credentials (encrypted) | — |

**Append-only enforcement strategy for consent ledger:**
- PostgreSQL tables use a `CHECK` trigger that prevents `UPDATE` and `DELETE` at the DB level
- Application layer has no `UPDATE`/`DELETE` permissions on consent tables (role-based)
- Consent state is derived by replaying the event log, not by mutating rows
- This mirrors the ECDSA audit trail model used in UPI and Account Aggregator

### 3.5 Messaging / Eventing

Kafka is the backbone of all cross-module communication.

**Key topics:**

| Topic | Producer | Consumers | Payload |
|---|---|---|---|
| `consent.granted` | Consent Vault API | Erasure Orchestrator, Audit Trail, Tenant Webhook | `{tenant_id, dp_id, purpose_id, timestamp, signature}` |
| `consent.withdrawn` | Consent Vault API | Erasure Orchestrator, Audit Trail, Tenant Webhook | `{tenant_id, dp_id, purpose_id, timestamp, signature}` |
| `erasure.scheduled` | Erasure Orchestrator | Notification Service | `{tenant_id, dp_id, scheduled_at, reason}` |
| `erasure.executed` | Erasure Orchestrator | Audit Trail, Cert Generator | `{tenant_id, dp_id, certificate_hash, timestamp}` |
| `breach.detected` | War Room Service | Notification Service, DPBI Reporter | `{tenant_id, breach_id, affected_count, severity}` |
| `discovery.completed` | Discovery Orchestrator | Module 2 Dashboard | `{tenant_id, job_id, flow_count, pii_locations_count}` |

**Retention**: Kafka topics retain events for 30 days; downstream PostgreSQL provides permanent storage.

### 3.6 Security Architecture

| Layer | Control | Specification |
|---|---|---|
| **Data at rest** | AES-256-GCM encryption | All PostgreSQL volumes, S3 buckets (SSE-S3 + SSE-KMS) |
| **Data in transit** | TLS 1.3 minimum | All inter-service and external communication |
| **Consent artifact integrity** | ECDSA P-256 signatures | Per-tenant signing keys, each consent record signed at creation |
| **Key management** | AWS KMS (Mumbai region) | Tenant signing keys never leave KMS; signing via KMS API |
| **Zero-knowledge design** | Encrypted consent payloads | TrustStack infrastructure cannot decrypt the purpose payload — only tenant can |
| **Authentication** | OAuth 2.0 + JWT | Data Fiduciary API access; short-lived tokens (15min), refresh token rotation |
| **API keys** | HMAC-SHA256 | Embedded SDK and webhook authentication |
| **Network isolation** | VPC segmentation | Discovery Sandbox in isolated VPC with no routes to Consent Vault subnets |
| **PII logging policy** | Structured logging, PII-free fields | Logs reference `dp_identifier_hash` only, never raw personal data |
| **Dependency security** | Automated CVE scanning | GitHub Dependabot + SAST in CI pipeline |

---

## 4. Consent Management System Design

### 4.1 Consent Lifecycle

```
               ┌─────────────────────────────────┐
               │          CONSENT LIFECYCLE       │
               └─────────────────────────────────┘

  [Notice Served]
       │  Section 5 + Rule 3: standalone notice, user's language
       ▼
  [Consent Requested]
       │  Purpose-specific, no pre-checked boxes
       ▼
  [Consent Granted] ──────────────────────────────────────────────┐
       │  ECDSA signed, timestamped, stored in ledger             │
       │  Kafka: consent.granted                                  │
       ▼                                                          │
  [Processing Authorised]                                        │
       │  Tenant webhook notified                                 │
       ▼                                                          │
  [Purpose Monitoring]                                           │
       │  Erasure Orchestrator tracks purpose lifecycle           │
       ▼                                                          │
  [Purpose Fulfilled / Consent Withdrawn] ◄───────────────────────┘
       │  Event: consent.withdrawn OR erasure timer triggered
       │  Withdrawal UX must be identical to granting (DPBI req)
       ▼
  [48hr Pre-Deletion Alert]
       │  Notification Service → DP via WhatsApp/email/in-app
       ▼
  [Data Erasure Executed]
       │  Propagated to all connected systems via tenant webhooks
       │  Cryptographic deletion certificate issued
       ▼
  [Erasure Certified]
       │  Certificate stored in S3 (Mumbai), hash in audit trail
       ▼
  [Audit Record Retained]
       └── Processing logs retained 1yr minimum (security req)
```

### 4.2 Data Model for Consent

**`consent_records` table** (append-only, no UPDATE/DELETE)

```sql
-- [DPDPA Section 5, Rule 4 Part B: 7-year retention, machine-readable]
CREATE TABLE consent_records (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id         UUID NOT NULL REFERENCES tenants(id),
  dp_identifier     TEXT NOT NULL,           -- hashed: sha256(email+tenant_salt)
  dp_identifier_raw TEXT,                    -- encrypted with tenant key via KMS
  purpose_id        UUID NOT NULL REFERENCES consent_purposes(id),
  action            TEXT NOT NULL            -- CHECK: 'GRANTED' | 'WITHDRAWN' | 'EXPIRED'
                    CHECK (action IN ('GRANTED', 'WITHDRAWN', 'EXPIRED')),
  consent_artifact  JSONB NOT NULL,          -- full notice text + purpose at time of consent
  language_code     TEXT NOT NULL,           -- ISO 639-1 or BCP-47; one of 22 scheduled langs
  collection_method TEXT NOT NULL            -- 'WIDGET' | 'API' | 'WHATSAPP' | 'VERBAL_AADHAAR'
                    CHECK (collection_method IN ('WIDGET', 'API', 'WHATSAPP', 'VERBAL_AADHAAR')),
  signature         TEXT NOT NULL,           -- ECDSA P-256 base64 signature
  signature_key_id  TEXT NOT NULL,           -- KMS key version for signature verification
  ip_address_hash   TEXT,                    -- hashed, not raw (PII-safe logging)
  user_agent_hash   TEXT,
  parental_consent  BOOLEAN DEFAULT FALSE,   -- [Section 9: Children's consent flag]
  parent_id_hash    TEXT,                    -- hash of parent Aadhaar eKYC token if applicable
  created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  -- NO updated_at, NO deleted_at — this table is append-only
  CONSTRAINT no_update_trigger CHECK (TRUE) -- enforced via DB-level trigger
);

-- Partition by tenant_id for query performance at scale
-- RLS policy: tenant can only SELECT own rows
```

**`consent_purposes` table**

```sql
CREATE TABLE consent_purposes (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id      UUID NOT NULL REFERENCES tenants(id),
  name           TEXT NOT NULL,              -- e.g. "Marketing Emails"
  description    JSONB NOT NULL,             -- {en: "...", hi: "...", ta: "..."} per language
  data_categories TEXT[] NOT NULL,           -- e.g. ['email', 'phone', 'purchase_history']
  legal_basis    TEXT NOT NULL DEFAULT 'CONSENT'  -- DPDPA has no other basis for this use case
                 CHECK (legal_basis = 'CONSENT'),
  erasure_class  TEXT NOT NULL,              -- 'SME_GENERAL' | 'ECOMMERCE_2CR' | 'GAMING_50L' | 'SOCIAL_2CR'
  retention_days INTEGER,                    -- NULL = erase on purpose fulfilment
  is_children_purpose BOOLEAN DEFAULT FALSE, -- [Section 9 flag]
  created_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deprecated_at  TIMESTAMPTZ                 -- soft-deprecate; never delete
);
```

**`erasure_schedules` table**

```sql
CREATE TABLE erasure_schedules (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id       UUID NOT NULL,
  dp_identifier   TEXT NOT NULL,
  purpose_id      UUID NOT NULL,
  trigger_type    TEXT NOT NULL              -- 'PURPOSE_FULFILLED' | 'CONSENT_WITHDRAWN' | 'TIMELINE'
                  CHECK (trigger_type IN ('PURPOSE_FULFILLED', 'CONSENT_WITHDRAWN', 'TIMELINE')),
  scheduled_at    TIMESTAMPTZ NOT NULL,
  notified_at     TIMESTAMPTZ,              -- 48hr pre-deletion notification sent timestamp
  executed_at     TIMESTAMPTZ,             -- NULL until executed
  certificate_s3_key TEXT,                 -- S3 key of deletion certificate
  status          TEXT NOT NULL DEFAULT 'PENDING'
                  CHECK (status IN ('PENDING', 'NOTIFIED', 'EXECUTED', 'FAILED'))
);
```

### 4.3 Policy Enforcement Mechanism

Consent policy is enforced at three layers:

1. **SDK / Widget Layer**: Pre-built consent widget enforces no pre-checked boxes, renders notice in the Data Principal's browser language, and blocks form submission until explicit affirmative action is recorded.

2. **API Layer**: Every API call from a Data Fiduciary that involves personal data processing is gated by a `ConsentMiddleware` that queries the Consent Vault for an active consent record. If none exists for the `(tenant_id, dp_identifier, purpose_id)` tuple, the API returns `403 Consent Required`.

3. **Erasure Orchestrator Layer**: Temporal.io workflows enforce timeline compliance regardless of the application layer — they execute regardless of tenant system state.

### 4.4 Integration Points

| Integration Type | API | SDK | Widget | Webhook |
|---|---|---|---|---|
| Data Fiduciary (tenant) backend | ✅ REST + GraphQL | ✅ JS, Python | — | ✅ Consent events |
| Consumer-facing product | — | ✅ React, React Native | ✅ Embed | — |
| Zoho CRM | ✅ | — | ✅ Embedded | ✅ |
| WhatsApp Business | ✅ | — | — (WhatsApp template) | ✅ |
| Tally | ✅ Tally XML | — | — | ✅ |

### 4.5 Compliance Considerations

- **Notice language**: Widget detects `Accept-Language` header and serves notice in the closest supported scheduled language. Falls back to English.
- **Withdrawal parity**: The withdrawal flow uses identical UI components to the grant flow — same number of taps, same visual weight. This is validated by DPBI as a registration requirement.
- **Children's data (Section 9)**: Age-gate at widget level. If user declares age < 18, flow switches to parental consent mode. Parent identity verified via Aadhaar eKYC (UIDAI API). Consent artifact records `parental_consent: true` and stores parent ID hash.
- **Seven-year retention**: Consent records are retained for 7 years after the consent event, enforced by a retention policy on the PostgreSQL table and S3 lifecycle rules. Deletion of consent records before 7 years is blocked at the DB level.
- **Machine-readable format**: Consent records are stored in structured JSON (consent_artifact column) and exposed via an export API that produces ISO 27560-compatible consent receipts.

### 4.6 Auditability and Traceability

Every consent event generates:
1. An **immutable database record** with full artifact (notice text, purpose description, language, timestamp, IP hash)
2. An **ECDSA P-256 signature** — verifiable against the tenant's public key without contacting TrustStack
3. A **Kafka event** consumed by the audit trail service
4. A **consent receipt** (JSON or PDF) delivered to the Data Principal via their preferred channel

For breach investigations or DPBI audits, the Regulatory War Room provides a query interface that reconstructs the full consent history for any `dp_identifier` across all purposes, with cryptographic proof of each event.

---

## 5. Data Discovery System Design

### 5.1 Overview and Sandbox Architecture

The Discovery module is the most architecturally sensitive component due to the **dual-role prohibition**. TrustStack, as a Consent Manager, cannot access the personal data it manages consent for. The Discovery Engine must therefore:

- Operate in a **fully isolated compute environment** (separate VPC)
- Connect to tenant data sources via **read-only OAuth tokens**
- **Never retain raw personal data** — only output metadata (field names, record counts, PII classification labels, data flow topology)
- Have **no network route** to the Consent Vault's data stores

This is enforced at the infrastructure level: separate AWS account, separate VPC, security groups explicitly deny inbound from Consent Vault subnets.

### 5.2 Metadata Ingestion Strategy

Each supported connector follows a three-phase ingestion:

1. **Authentication**: Tenant provides read-only OAuth 2.0 credentials for their cloud account / SaaS tool. Credentials are stored encrypted in Tenant Registry; never passed to Discovery Sandbox directly — sandbox assumes an IAM role scoped to the specific tenant's resources.

2. **Schema Crawl**: Connector enumerates available data sources (S3 bucket names, RDS table names, Zoho CRM module fields, Razorpay webhook logs). Produces a **schema graph**: tables/collections → fields → data types.

3. **Sample-Based PII Classification**: For each field, the connector samples a statistically representative subset (configurable, default 500 records). The sample is passed to the on-premise PII Detector LLM. **Samples are never persisted** — they exist only in memory during inference. Output is a PII classification label (`EMAIL | PHONE | PAN | AADHAAR | NAME | ADDRESS | FINANCIAL | NONE`) and a confidence score.

### 5.3 Data Catalog Architecture

```
Discovery Sandbox (Isolated VPC)
┌──────────────────────────────────────────────────────┐
│                                                      │
│  Connector Layer          PII Detection Layer        │
│  ┌──────────────┐        ┌───────────────────────┐   │
│  │ AWS S3       │──────► │ PII Detector LLM      │   │
│  │ Connector    │        │ (On-premise, isolated) │   │
│  ├──────────────┤        │                       │   │
│  │ Zoho CRM     │──────► │ Input: field samples  │   │
│  │ Connector    │        │ Output: PII labels     │   │
│  ├──────────────┤        │         + confidence  │   │
│  │ Razorpay     │──────► └──────────┬────────────┘   │
│  │ Connector    │                   │                │
│  ├──────────────┤                   ▼                │
│  │ Tally        │        ┌──────────────────────┐    │
│  │ Connector    │──────► │  Metadata Assembler  │    │
│  └──────────────┘        │  (no raw data)       │    │
│                           └──────────┬───────────┘    │
└──────────────────────────────────────┼────────────────┘
                                       │
                        Metadata only (no personal data)
                                       │
                                       ▼
                          ┌────────────────────────┐
                          │  Discovery Catalog DB   │
                          │  (Sandboxed PostgreSQL) │
                          │                        │
                          │  • data_sources        │
                          │  • field_catalog       │
                          │  • pii_classifications │
                          │  • data_flow_graph     │
                          └────────────────────────┘
```

### 5.4 Indexing and Search

The Discovery Catalog is searchable by:
- **PII type** — "Show me all fields that contain AADHAAR numbers"
- **Data source** — "What PII exists in our Zoho CRM?"
- **Processing purpose** — "Which fields are used for marketing but lack a consent record?"
- **Risk score** — Fields ranked by sensitivity (AADHAAR > PAN > EMAIL > NAME)

Search is implemented via PostgreSQL full-text search (`tsvector`) on field names, data source names, and PII labels. At scale, this migrates to Elasticsearch (Indian hosted).

### 5.5 Classification and Tagging

**Automated classification** (PII Detector LLM):
- Trained on Indian-specific PII patterns: Aadhaar (12-digit), PAN (alphanumeric), Indian mobile numbers (+91), Indian postal codes
- Confidence threshold: `< 0.7` → flagged for manual review; `≥ 0.7` → auto-classified
- Classification is deterministic per field name + sample — cached after first run

**Manual tagging**:
- Tenant compliance team can override, confirm, or reject auto-classifications
- Overrides are tracked with `classified_by: 'HUMAN'` flag for audit

**Purpose tagging**:
- After PII classification, the tenant maps each data flow to a consent purpose
- This mapping populates the Data Flow Map and feeds directly into Consent Vault as the set of purposes requiring consent widgets

### 5.6 Lineage Tracking

Data lineage captures how personal data moves between systems:

```
[Source: Razorpay] ──payment.customer_email──► [Destination: Zoho CRM] ──► [Destination: Marketing Tool]
       │                                               │
       ▼                                               ▼
  Consent Purpose: Payment Processing         Consent Purpose: Marketing Emails
  Status: ✅ Active consent                   Status: ❌ No consent record found
```

Lineage is represented as a **directed graph** in the catalog:
- Nodes: data sources (systems)
- Edges: data fields that flow between them
- Edge metadata: PII types, consent status, purpose ID

This graph is the **Data Flow Map** — the primary output of Module 2 and the primary input to Consent Vault onboarding.

### 5.7 Integration with Consent System

The Discovery → Consent handoff is the key value-creation moment:

1. Discovery Catalog produces a Data Flow Map with all `(source, destination, field, pii_type)` tuples
2. For each unique processing purpose inferred from the flow, Module 2 generates a **draft consent purpose** record
3. Tenant reviews and approves in the Module 2 dashboard
4. Approved purposes are published to the Consent Vault via the internal GraphQL API
5. The Consent Vault activates consent collection for those purposes

**This is the only cross-module data transfer, and it carries zero personal data — only metadata.**

---

## 6. Low-Level Design — Key Components

### 6.1 Consent Vault API

**Service**: `consent-vault-api`  
**Language**: TypeScript ESM  
**Framework**: Fastify (chosen for performance and schema-first JSON validation)

**Key modules:**
- `ConsentGrantHandler` — validates request, resolves purpose, calls KMS signer, writes to ledger, emits Kafka event
- `ConsentWithdrawHandler` — validates active consent exists, writes withdrawal record, emits Kafka event
- `ConsentQueryHandler` — reads from append-only ledger, rebuilds current state from event log
- `KMSSigner` — wraps AWS KMS API, caches public keys, handles key rotation
- `LanguageResolver` — maps `Accept-Language` to nearest supported scheduled language
- `ParentalConsentHandler` — orchestrates age verification gate and Aadhaar eKYC flow

**API Contract — Grant Consent:**

```http
POST /v1/consent/grant
Authorization: Bearer <tenant_jwt>
Content-Type: application/json

{
  "dp_identifier": "user@example.com",     // or hashed form
  "purpose_id": "uuid-of-purpose",
  "language_code": "hi",                    // Hindi
  "collection_method": "WIDGET",
  "consent_artifact": {
    "notice_text": "हम आपके ईमेल का उपयोग...",
    "notice_version": "v2.1",
    "data_categories": ["email"],
    "widget_session_id": "abc123"
  },
  "parental_consent": false
}

→ 201 Created
{
  "consent_id": "uuid",
  "signature": "base64-ecdsa-signature",
  "signature_key_id": "kms-key-version",
  "receipt_url": "https://sahmat.truststack.in/receipt/uuid",
  "created_at": "2026-04-22T10:00:00Z"
}
```

**API Contract — Check Consent (middleware use):**

```http
GET /v1/consent/check?dp_identifier=hash&purpose_id=uuid
Authorization: Bearer <tenant_jwt>

→ 200 OK
{
  "has_active_consent": true,
  "consent_id": "uuid",
  "granted_at": "2026-04-22T10:00:00Z",
  "language_code": "hi"
}

→ 404 (no consent found — caller should gate processing)
{
  "has_active_consent": false,
  "reason": "NO_CONSENT_RECORD"
}
```

### 6.2 Erasure Orchestrator

**Service**: `erasure-orchestrator`  
**Technology**: Temporal.io (TypeScript SDK)

Temporal workflows provide durable execution: if the service restarts mid-erasure, the workflow resumes from the last checkpoint without data loss.

**`ErasureWorkflow` (Temporal):**

```typescript
// DPDPA Rule 8 + Third Schedule — class-specific erasure
export async function ErasureWorkflow(input: ErasureInput): Promise<ErasureCertificate> {
  
  // Step 1: Determine erasure timeline based on DF class
  const schedule = await determineErasureSchedule(input.tenantId, input.purposeId);
  
  // Step 2: Sleep until T-48hrs
  await sleep(schedule.executeAt - 48 * HOURS);
  
  // Step 3: Send 48hr pre-deletion notification [DPDPA Rule 8 requirement]
  await sendErasureNotification(input.dpIdentifier, schedule.executeAt, input.language);
  
  // Step 4: Sleep until execution time
  await sleep(schedule.executeAt - NOW());
  
  // Step 5: Issue deletion commands to all connected systems
  const deletionResults = await propagateDeletion({
    tenantId: input.tenantId,
    dpIdentifier: input.dpIdentifier,
    purposeId: input.purposeId,
    connectedSystems: await getConnectedSystems(input.tenantId),
  });
  
  // Step 6: Generate cryptographic deletion certificate
  const certificate = await generateDeletionCertificate({
    results: deletionResults,
    tenantId: input.tenantId,
    dpIdentifier: input.dpIdentifier,
    executedAt: NOW(),
  });
  
  // Step 7: Store certificate on S3 (Mumbai) and record in audit trail
  await storeCertificate(certificate);
  await emitAuditEvent('erasure.executed', certificate);
  
  return certificate;
}
```

**`determineErasureSchedule` logic** (implements Third Schedule):

```typescript
function determineErasureSchedule(tenantId, purposeId): ErasureSchedule {
  const tenant = await getTenant(tenantId);
  const purpose = await getPurpose(purposeId);
  
  // Third Schedule class-specific rules
  if (tenant.category === 'ECOMMERCE' && tenant.registeredUsers >= 20_000_000) {
    return { type: 'TIMELINE', daysFromLastInteraction: 3 * 365 };
  }
  if (tenant.category === 'GAMING' && tenant.registeredUsers >= 5_000_000) {
    return { type: 'TIMELINE', daysFromLastInteraction: 3 * 365 };
  }
  if (tenant.category === 'SOCIAL_MEDIA' && tenant.registeredUsers >= 20_000_000) {
    return { type: 'TIMELINE', daysFromLastInteraction: 3 * 365 };
  }
  
  // All other DFs (most SMEs) — erase on purpose fulfilment or withdrawal
  return { type: 'ON_TRIGGER', trigger: 'PURPOSE_FULFILLED_OR_WITHDRAWN' };
}
```

### 6.3 Discovery Connector: Zoho CRM

```typescript
// Read-only connector — never writes to Zoho
export class ZohoCRMConnector implements DiscoveryConnector {
  
  async crawlSchema(credentials: ZohoOAuthToken): Promise<FieldCatalog> {
    const modules = await zohoAPI.getModules(credentials);
    
    const catalog: FieldCatalog = { sources: [] };
    
    for (const module of modules) {
      const fields = await zohoAPI.getFields(credentials, module.api_name);
      catalog.sources.push({
        name: `zoho_crm.${module.api_name}`,
        fields: fields.map(f => ({ name: f.api_name, type: f.data_type }))
      });
    }
    
    return catalog;
  }
  
  async sampleField(
    credentials: ZohoOAuthToken,
    module: string,
    field: string,
    sampleSize = 500
  ): Promise<string[]> {
    // Fetch sample records — data NEVER persisted, only held in memory for LLM inference
    const records = await zohoAPI.searchRecords(credentials, module, {
      fields: [field],
      per_page: sampleSize
    });
    
    const samples = records.map(r => String(r[field] ?? '')).filter(Boolean);
    
    // Pass to PII detector, discard samples immediately after
    const classification = await piiDetector.classify(field, samples);
    
    // samples go out of scope here — GC eligible
    return classification;
  }
}
```

### 6.4 Breach Notification Workflow

**`BreachResponseWorkflow` (Temporal):**

Dual obligation: Data Principals notified immediately, DPBI within 72hrs, CERT-In within 6hrs (parallel).

```typescript
export async function BreachResponseWorkflow(breach: BreachReport): Promise<void> {
  
  // Immediate: notify affected Data Principals
  await notifyAffectedDataPrincipals(breach); // async, non-blocking
  
  // CERT-In: 6hr deadline [separate directive]
  await withDeadline(6 * HOURS, async () => {
    await submitCERTInReport(breach);
  });
  
  // DPBI: 72hr comprehensive report [DPDPA Rule 7]
  await withDeadline(72 * HOURS, async () => {
    const report = await generateDPBIReport(breach);
    await submitDPBIReport(report);
  });
  
  await recordBreachClosure(breach.id);
}
```

---

## 7. Architecture Decision Records

### ADR-001: Modular Monolith Over Microservices

**Context:** At founding-team stage, a microservices architecture creates significant operational overhead (service discovery, distributed tracing, inter-service contracts) and increases the blast radius of failures.

**Decision:** Build as a modular monolith within each product module. Module boundaries enforced via Kafka and GraphQL, not deployment boundaries.

**Alternatives considered:**
- Full microservices: Each capability (consent-grant, erasure, audit) as a separate service. Rejected — too much operational complexity for a team of < 15 engineers.
- Serverless (Lambda): Appealing for Trust Check (stateless scan). Rejected for Consent Vault — cold starts on consent grant requests create latency that degrades user trust.

**Trade-offs:** Harder to scale individual components independently later. Mitigated by clean internal module boundaries — extraction to microservices is achievable without data model changes.

---

### ADR-002: PostgreSQL Append-Only Tables for Consent Ledger

**Context:** DPDPA Rule 4 requires consent records to be immutable, machine-readable, and retained for 7 years. The consent ledger is the foundation of TrustStack's regulatory standing.

**Decision:** Use PostgreSQL with append-only enforcement via DB-level triggers (`BEFORE UPDATE OR DELETE` triggers that raise exceptions). No application-level updates or deletes are permitted on consent tables.

**Alternatives considered:**
- Blockchain / distributed ledger: Provides tamper-evidence but adds operational complexity, latency, and cost with no regulatory requirement for decentralisation.
- Event store (EventStoreDB): Native append-only, but introduces another infrastructure dependency. PostgreSQL is more operable for a small team.
- Audit log in S3: Simple but harder to query for real-time consent checks.

**Trade-offs:** PostgreSQL is not natively immutable — the append-only guarantee is enforced by the DB trigger and IAM role restrictions. A compromised DBA could bypass this. Mitigated by KMS-based ECDSA signatures: even if a record is modified, the signature verification would fail.

---

### ADR-003: ECDSA P-256 Over RSA for Consent Signing

**Context:** Each consent artifact must be cryptographically signed to provide tamper-evidence. Key choice matters for performance (consent is issued at scale), key size (stored in DB), and regulatory acceptability.

**Decision:** ECDSA P-256 (secp256r1). Signing via AWS KMS; keys never leave KMS.

**Alternatives considered:**
- RSA-2048: Widely understood but larger signatures (256 bytes vs 64 bytes for P-256), slower signing.
- Ed25519: Faster and smaller than P-256, but less broadly supported in India's regulatory infrastructure (Aadhaar signing uses P-256; alignment is advantageous).
- HMAC-SHA256: Fast but symmetric — requires sharing secret key with Data Principals for verification, which breaks the verifiability model.

**Trade-offs:** KMS signing introduces latency (~5ms per sign) and cost ($0.03/10K API calls). At 1M consents/day, KMS cost is ~$3/day — acceptable. Latency is mitigated by async signing: consent is acknowledged to the DP immediately; signature is applied and stored asynchronously within 100ms.

---

### ADR-004: Kafka for Cross-Module Eventing

**Context:** Consent events must trigger multiple downstream actions (erasure scheduling, audit trail, tenant webhooks, notifications) without the consent grant API blocking on all of them.

**Decision:** Kafka as the event bus for all cross-module communication.

**Alternatives considered:**
- Direct database polling: Simple but creates tight coupling and polling overhead.
- AWS SNS/SQS: Managed, lower operational overhead. Rejected because it ties the architecture to a single cloud provider, and cross-cloud deployment (AWS Mumbai + GCP Mumbai) would require bridging.
- RabbitMQ: Good for complex routing, but operationally heavier and lacks the retention model needed for event replay.

**Trade-offs:** Kafka requires operational expertise (partition management, consumer group lag monitoring). Mitigated by using AWS MSK (managed Kafka) on Mumbai region, reducing ops burden to configuration rather than cluster management.

---

### ADR-005: Temporal.io for Erasure and Breach Workflows

**Context:** Erasure workflows have durations measured in years (3yr timeline for large e-commerce). Breach workflows have hard deadlines (6hr, 72hr). Both must survive service restarts, infrastructure failures, and code deployments.

**Decision:** Temporal.io for all long-running compliance workflows.

**Alternatives considered:**
- Cron jobs + database state machine: Simple but fragile — a crashed cron job mid-erasure leaves the system in an inconsistent state with no recovery path.
- AWS Step Functions: Managed, well-integrated with AWS. Rejected for same cloud vendor lock-in reason as SNS/SQS. Also less expressive for long-duration workflows with complex branching.
- Custom workflow engine: Too much engineering effort to build reliably.

**Trade-offs:** Temporal introduces a new infrastructure dependency and requires engineers to learn its execution model. Investment is justified given the severity of failure modes — a missed erasure deadline is a ₹200Cr liability.

---

### ADR-006: Sandboxed Deployment for Discovery Engine

**Context:** DPDPA Rule 4 First Schedule Part B: Consent Manager cannot simultaneously act as Data Processor. If the Discovery Engine (which crawls raw personal data) shares infrastructure with the Consent Vault, TrustStack faces a dual-role regulatory violation.

**Decision:** Discovery Engine runs in a completely isolated AWS account with a separate VPC. No network routes to Consent Vault subnets. Separate IAM roles, separate databases, separate KMS keys.

**Alternatives considered:**
- Logical separation (same account, network ACLs): Does not satisfy the regulatory intent. DPBI could argue that technical access remains possible for TrustStack operators.
- Separate legal entity: Discovery business operated by a subsidiary. Architecturally cleanest but adds corporate overhead. Held as a contingency if DPBI interpretation becomes stricter.

**Trade-offs:** Separate AWS account adds cost and operational complexity. Accepted as non-negotiable for regulatory compliance.

---

### ADR-007: On-Premise LLMs for PII Detection and Translation

**Context:** The PII Detector (Discovery module) processes raw personal data samples. The Jargon Slayer processes regulatory text. Sending either to a third-party LLM API would either create a dual-role violation (PII samples) or a data processor relationship requiring a DPA (regulatory text).

**Decision:** Fine-tuned open-weight LLMs hosted on-premise (AWS Mumbai EC2 GPU instances) for both tasks.

**Alternatives considered:**
- OpenAI / Anthropic APIs: Superior model quality for Jargon Slayer. Rejected for PII Detector (creates data processing relationship). Acceptable for Jargon Slayer (no PII), but on-premise provides consistency and avoids dependency on external API availability.
- Managed inference (AWS Bedrock with Titan): Keeps data on Indian infra but still creates a data processor relationship with AWS for the PII data.

**Trade-offs:** On-premise LLM inference requires GPU infrastructure ($800–1500/mo per A10G instance), model fine-tuning expertise, and model update workflows. Accepted as a strategic capability — the PII classification models become a defensible moat.

---

### ADR-008: WhatsApp Business as Primary Consent Channel for Tier 2–3

**Context:** In Tier 2–3 cities, WhatsApp is the primary digital interface. Asking these users to navigate a web consent widget creates a drop-off that harms both compliance (no consent = no processing) and DP experience.

**Decision:** WhatsApp Business API integration as a first-class consent channel, including consent grant, review, and 1-tap withdrawal.

**Alternatives considered:**
- SMS OTP: No rich content support, no thread history for consent review.
- Email: Low open rates in Tier 2–3; requires smartphone internet, not just WhatsApp.

**Trade-offs:** Meta (WhatsApp) controls the API and message template approval process. Dependency on a third party for a compliance-critical function. Mitigated by ensuring every consent granted via WhatsApp is immediately replicated to the Consent Vault — WhatsApp is the channel, not the record.

---

### ADR-009: Unit-Based Billing by Purpose-Linked Data Flow

**Context:** Per-seat SaaS pricing does not reflect the compliance complexity of Indian SMEs (a 3-person startup can have 20 data flows; a 100-person company might have only 8). Per-seat pricing undercharges high-complexity customers and overcharges simple ones, creating churn.

**Decision:** Billing unit is the "Purpose-Linked Data Flow" — a unique `(source, destination, purpose)` tuple discovered and registered in the platform.

**Alternatives considered:**
- Per-seat: Standard SaaS. Doesn't correlate with compliance complexity.
- Per-API-call: Creates unpredictability for SME budgets.
- Annual flat fee: Simpler but caps revenue growth as customers add complexity.

**Trade-offs:** Requires customers to understand what a "data flow" is. Mitigated by the Data Flow Map visualisation in Module 2 — customers discover flows, not count them manually.

---

### ADR-010: React Native for Consumer Dashboard (Mobile-First)

**Context:** Data Principals (end users) in India are overwhelmingly mobile-first. The Consumer Dashboard (where users view and withdraw consent) must be usable on a 5.5" screen on a 4G connection. A web-only dashboard effectively disenfranchises a significant portion of the user base.

**Decision:** React Native for the Consumer Dashboard mobile app; React (web) for the Data Fiduciary dashboard.

**Alternatives considered:**
- Native iOS + Android: Better performance but 2× development cost for a startup.
- Flutter: Strong alternative, but TypeScript is the team's primary language — React Native reduces context switching.
- Progressive Web App: Good performance on modern Android but poor iOS support for push notifications (needed for consent change alerts).

**Trade-offs:** React Native's bridge architecture can introduce latency for complex animations. Accepted — the Consumer Dashboard is a functional tool, not a showcase app. Performance targets are p95 < 1s for consent grant/withdrawal on 4G.

---

## 8. Non-Functional Requirements

### 8.1 Scalability

| Component | Current Target | Scale Target (Y2) | Strategy |
|---|---|---|---|
| Consent Vault API | 1K TPS | 50K TPS | Horizontal pod autoscaling, read replicas |
| Trust Check Scanner | 100 scans/min | 10K scans/min | Stateless + Lambda for burst |
| Discovery Jobs | 10 concurrent tenants | 500 concurrent | Job queue + worker pool scaling |
| Kafka | 10K events/sec | 1M events/sec | MSK partition scaling |
| PostgreSQL (Consent Ledger) | 500 writes/sec | 50K writes/sec | Citus for horizontal sharding at scale |

### 8.2 Performance

| Operation | P50 Target | P95 Target | P99 Target |
|---|---|---|---|
| Consent grant (API) | 50ms | 150ms | 300ms |
| Consent check (middleware) | 10ms | 30ms | 80ms |
| Trust Check scan | 15s | 45s | 90s |
| Consumer Dashboard load | 800ms | 2s | 3s |
| Consent withdrawal | 200ms | 500ms | 1s |

### 8.3 Reliability

- **Consent Vault API SLA**: 99.9% uptime (< 8.7hr downtime/year) — contractual at Growth tier
- **Erasure workflows**: Temporal.io provides at-least-once execution guarantees. Idempotency keys prevent double-erasure
- **Breach notification**: PagerDuty alerting for War Room service failures; manual override procedure for SLA compliance during outage
- **RTO / RPO**:
  - Consent Vault: RTO 1hr, RPO 1min (synchronous Kafka + async PostgreSQL replication)
  - Discovery Sandbox: RTO 4hr, RPO 1hr (lower criticality, no real-time user impact)

### 8.4 Security

- Annual penetration testing by CERT-In empanelled auditor
- VAPT before Consent Manager registration submission (Nov 2026 deadline)
- Bug bounty programme post-launch
- Secrets management via AWS Secrets Manager (never environment variables in code)
- Dependency CVE scanning in CI (Snyk or GitHub Advanced Security)
- All PII-handling code must reference the DPDPA section it addresses in a comment (enforced via PR checklist)
- Zero-knowledge design: TrustStack platform cannot decrypt the content of personal data — only tenant KMS-derived keys can

### 8.5 Observability

| Signal | Tool | Retention | Use Case |
|---|---|---|---|
| **Structured logs** | AWS CloudWatch Logs (Mumbai) | 90 days hot / S3 archive 7yr | Compliance audit trail; no PII in logs |
| **Metrics** | Prometheus + Grafana | 30 days | SLA monitoring, capacity planning |
| **Distributed tracing** | AWS X-Ray | 30 days | Latency debugging across services |
| **Alerting** | PagerDuty | — | On-call rotation for consent API + breach workflows |
| **Synthetic monitoring** | Checkly | — | Consent grant flow tested every 5min from Mumbai |
| **Kafka consumer lag** | Grafana + MSK metrics | 30 days | Erasure/audit pipeline health |

**PII-safe logging rule**: Log lines may contain `tenant_id`, `purpose_id`, `dp_identifier_hash` (SHA-256 of `dp_identifier + tenant_salt`), error codes, and latency metrics. Raw email addresses, names, phone numbers, or any personal data are never logged.

### 8.6 Cost Considerations

| Component | Monthly Estimate (Launch) | Monthly Estimate (Scale, Y2) |
|---|---|---|
| AWS Mumbai (EC2, RDS, MSK, KMS, S3) | ~₹2.5L | ~₹15L |
| GCP Mumbai (warm standby) | ~₹50K | ~₹3L |
| GPU instances (on-premise LLM) | ~₹1.2L | ~₹5L (multiple A10G) |
| Temporal.io cloud | ~₹40K | ~₹2L |
| WhatsApp Business API | Variable (per message) | ~₹1L at 1M messages |
| **Total infra** | **~₹4.5L/mo** | **~₹26L/mo** |

At Y1 ARR target of ₹2–3Cr, infra cost is 15–18% of revenue at launch — acceptable for a regulated infrastructure business. Target < 10% at Y2 scale.

---

## 9. Risks and Open Questions

### 9.1 Regulatory Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| DPBI delays Consent Manager registration beyond Nov 2026 | Medium | High | Modules 1–2 are revenue-generating without CM registration. Consent Vault can operate as SaaS pre-registration. |
| DPBI interprets dual-role prohibition more strictly (e.g., bans Discovery + CM in same corporate entity) | Low | High | Structural separation plan ready: Discovery as wholly-owned subsidiary with separate board. |
| ₹2Cr net worth requirement creates seed round pressure | High | Medium | Structure seed round to satisfy statutory capital. May need strategic angel with balance sheet. |
| DPDPA interpretation diverges from GDPR (creating new compliance patterns we haven't designed for) | Medium | Medium | Maintain regulatory tracker in War Room; design modular compliance engine for new rules. |

### 9.2 Technical Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| On-premise LLM performance degrades on edge cases (regional dialects, code-mixed Hindi-English) | High | Medium | Human-in-the-loop review queue for < 0.7 confidence classifications; quarterly model fine-tuning |
| Tally/Zoho API changes break connectors | Medium | High | Versioned connectors with automated regression testing; vendor relationship management |
| Temporal.io cluster failure causes erasure workflow delays | Low | High | Multi-AZ Temporal cluster; manual override workflow documented for regulatory emergency |
| PostgreSQL append-only trigger bypass by compromised operator | Low | High | ECDSA signature chain provides detection even if records tampered; quarterly external audit of ledger integrity |

### 9.3 Open Questions

1. **DPBI API availability**: Will the DPBI provide a REST API for breach notification submission (Rule 7), or will initial compliance require manual portal submission? Design should support both.

2. **Aadhaar eKYC for parental consent (Section 9)**: UIDAI API licensing requires IRDA/RBI/UIDAI approval for non-financial entities. What is the timeline and eligibility? Alternative: Video KYC as fallback.

3. **Machine-readable consent receipt format**: DPDPA says "machine-readable" but does not specify a format. ISO 27560 (Consent Record Information Structure) is the likely standard. Confirm with DPBI before finalising receipt schema.

4. **Cross-border SaaS usage by tenants**: Many SMEs use Salesforce, HubSpot, or foreign-hosted tools. When TrustStack propagates an erasure command to these tools via webhook, does TrustStack incur any cross-border transfer obligation? Legal opinion needed.

5. **WhatsApp consent validity**: Is a consent granted via a WhatsApp message (button tap) legally equivalent to one granted via a web widget? DPBI has not issued guidance. Conservative implementation: WhatsApp initiates flow, web confirmation required for full legal validity.

6. **Discovery Sandbox compute isolation**: At what level must isolation be enforced to satisfy DPBI's dual-role interpretation? Network isolation (our current design) or legal entity isolation? Awaiting regulatory clarification before finalising corporate structure.

---

## 10. Visualisation Plan

The following diagrams must be created for investor decks, technical documentation, and engineering onboarding. No images are generated here — this section specifies what each diagram should contain and the recommended tool.

---

### Diagram 1: System Architecture Overview

**Purpose:** Investor and engineering overview of the three-module structure and platform layer.

**Contents:**
- Three module boxes (Vernacular Awareness, Assessment & Discovery, Implementation & Automation)
- Platform layer below (GraphQL Bus, Kafka, Auth, Tenant Registry)
- Data/Infra layer (PostgreSQL, Redis, S3, AWS/GCP Mumbai)
- Discovery Sandbox shown as isolated box with explicit "no data path" barrier to Consent Vault
- External integrations on right side (Tally, Zoho, WhatsApp, Razorpay, DPBI, CERT-In)

**Tool:** Excalidraw (hand-drawn style for investor decks) or Figma (polished version for investor deck PDF)

---

### Diagram 2: Consent Lifecycle Flow

**Purpose:** Show regulators, legal, and engineers the complete state machine for consent from notice to erasure.

**Contents:**
- Linear flow: Notice Served → Consent Requested → Consent Granted → Processing Authorised → Purpose Monitoring → [Purpose Fulfilled / Withdrawal] → 48hr Alert → Erasure Executed → Certificate Issued → Audit Record Retained
- Decision branches: children's flow (age gate → parental consent), withdrawal path
- Regulatory annotations at each step (e.g., "Section 5 Rule 3" at Notice, "Rule 8 Third Schedule" at Erasure)
- Colour coding: DP actions (blue), system actions (green), regulatory obligations (amber)

**Tool:** Mermaid (state diagram) for docs; Figma for investor deck

---

### Diagram 3: Data Flow Diagram — Consent Event

**Purpose:** Technical documentation of how a consent grant propagates through the system.

**Contents:**
- Actor: Data Principal (browser/mobile/WhatsApp)
- Consent Widget → Consent Vault API → KMS (signing) → PostgreSQL (append) → Kafka (event)
- Kafka consumers: Erasure Orchestrator, Audit Trail Service, Tenant Webhook, Notification Service
- Annotate each arrow with protocol (HTTP, Kafka, KMS API) and key data fields

**Tool:** Draw.io (good for data flow arrows with labels)

---

### Diagram 4: Service Interaction Map

**Purpose:** Engineering team reference for inter-service dependencies and communication patterns.

**Contents:**
- All 12 services as boxes
- Synchronous calls as solid arrows (HTTP/GraphQL)
- Asynchronous events as dashed arrows (Kafka topics labelled)
- Discovery Sandbox shown with red border = isolated deployment
- Temporal.io shown as orchestrator with bidirectional arrows to Erasure Orchestrator and War Room

**Tool:** Draw.io or C4 model (Context/Container/Component diagrams)

---

### Diagram 5: Data Discovery Pipeline

**Purpose:** Explain to enterprise customers and investors how the sandboxed discovery works without compromising the dual-role prohibition.

**Contents:**
- Left side: Tenant data sources (Zoho, Razorpay, S3, Tally) — labelled "Fiduciary's Systems"
- Centre: Discovery Sandbox (isolated VPC boundary shown as dotted box) with: Connectors → Schema Crawler → Sample Extractor → PII Detector LLM → Metadata Assembler
- Critical label: "Raw personal data processed in memory only — never persisted"
- Right side: Discovery Catalog (metadata only) → Data Flow Map → Tenant Dashboard
- Bottom right: Arrow from Tenant Dashboard to Consent Vault with label "Metadata handoff only (zero personal data)"

**Tool:** Figma (for the visual clarity needed to explain dual-role isolation)

---

### Diagram 6: Deployment Architecture — Cloud

**Purpose:** Regulatory (DPBI accessibility), technical (availability), and investor (cost) reference.

**Contents:**
- **AWS Mumbai** (primary): Consent Vault API pods, PostgreSQL primary, MSK Kafka cluster, KMS, S3 (consent artifacts), Temporal.io cluster
- **GCP Mumbai** (warm standby): Consent Vault replica, PostgreSQL read replica, Cloud Storage backup
- **Discovery Sandbox** (separate AWS account, Mumbai): EC2 GPU instances, Discovery Catalog DB, Connector workers
- **CDN** (optional, configurable): Consumer Dashboard static assets — CloudFront with Mumbai origin
- Labels: "Consent records never leave India (Section 16)", "Separate account = dual-role isolation"
- Network flow arrows showing: tenant → ALB → Consent Vault API; no arrow between Discovery VPC and Consent Vault VPC

**Tool:** AWS Architecture Icons in Lucidchart or Draw.io

---

### Diagram 7: Erasure Workflow (Temporal.io)

**Purpose:** Engineering documentation for the Temporal workflow steps; regulatory evidence of automated compliance.

**Contents:**
- Temporal workflow timeline from consent_granted event
- Decision node: "Which erasure class?" branching to SME (immediate on trigger) vs Large DF (3yr countdown)
- Timer checkpoint: T-48hrs → notification
- Timer checkpoint: T-0 → deletion propagation to connected systems
- Certificate generation and S3 storage
- Audit trail update
- Annotate with DPDPA references at each step

**Tool:** Mermaid (sequence diagram) for docs, Lucidchart for formal compliance documentation

---

*End of TrustStack System Architecture & Design Specification v1.0*

*Next steps: (1) Legal review of DPBI/Aadhaar open questions, (2) Engineering kickoff using LLD section, (3) Cloud account setup with dual-account Discovery isolation, (4) VAPT scope definition for Nov 2026 CM registration.*
