# Data Model — TrustStack Module 2: AI Data Discovery & NLP Contract Audit

**Version:** 1.0
**Date:** 2026-04-22
**Database:** PostgreSQL 16
**Region:** AWS ap-south-1 (Mumbai) — all data stays in India (DPDPA Section 16)

---

## Overview

Module 2 discovers where personal data lives across a tenant's vendor ecosystem, maps how it flows, and audits vendor contracts for DPDPA compliance. It has two sub-products:

- **AI Data Discovery Engine**: Crawls connector systems (Zoho, Razorpay, Tally, AWS, etc.), classifies PII findings, and builds a Data Flow Map per tenant.
- **NLP Contract Audit**: Ingests vendor contracts in any of 22 Indian languages, extracts clauses, flags DPDPA violations, and generates a compliant Data Processing Agreement (DPA).

**Critical design rule (Dual-Role Prohibition — Section 2(g) DPDPA):**
TrustStack is a Registered Consent Manager. It cannot simultaneously act as a Data Processor for the same Data Principal. The Discovery Engine runs **sandboxed** — it never reads data from, and never writes to, the Consent Vault (Module 1). The Snapshot API (used by the Erasure Engine) is the only cross-module boundary, and it flows outward from Module 2, not inward.

**No raw PII is ever stored in this schema.** Only SHA-256 hashes of PII values appear in `pii_findings` for deduplication. Data Flow Map nodes store categories and risk scores only. This is consistent with the Zero-Knowledge design documented in the Consent Vault's `SECURITY_DESIGN.md`.

---

## 1. Enum Type Definitions

All custom types are created before tables. Enums are prefixed by domain to avoid naming conflicts across modules in the same PostgreSQL database.

```sql
-- ─────────────────────────────────────────────────────────────────────────────
-- Connector / Vendor Types
-- Covers all integrations listed in TrustStack product spec.
-- ─────────────────────────────────────────────────────────────────────────────
CREATE TYPE vendor_type AS ENUM (
    'ZOHO_CRM',
    'ZOHO_BOOKS',
    'RAZORPAY',
    'TALLY',
    'WHATSAPP_BUSINESS',
    'AWS_S3',
    'GCP_STORAGE',
    'POSTGRESQL',
    'MYSQL',
    'MONGODB',
    'CLOUDWATCH',
    'ELASTICSEARCH',
    'CUSTOM_REST'
);

CREATE TYPE connector_status AS ENUM (
    'ACTIVE',
    'ERROR',
    'PAUSED',
    'PENDING_AUTH'
);

-- ─────────────────────────────────────────────────────────────────────────────
-- Scan Jobs
-- ─────────────────────────────────────────────────────────────────────────────
CREATE TYPE scan_job_type AS ENUM (
    'FULL_SCAN',          -- Scan all connectors from scratch
    'INCREMENTAL_SCAN',   -- Scan only records changed since last_scan_at
    'CONNECTOR_SCAN',     -- Scan a single specific connector
    'DP_TARGETED_SCAN'    -- Erasure Engine requests scan for one DP hash
);

CREATE TYPE scan_job_status AS ENUM (
    'QUEUED',
    'RUNNING',
    'COMPLETED',
    'FAILED',
    'CANCELLED'
);

CREATE TYPE scan_trigger AS ENUM (
    'SCHEDULED',          -- Cron-based automatic scan
    'MANUAL',             -- DPO or CTO triggered via API/UI
    'ERASURE_REQUEST',    -- Erasure Engine needs current data locations
    'CONSENT_WITHDRAWN'   -- Consent Vault signals consent was withdrawn
);

-- ─────────────────────────────────────────────────────────────────────────────
-- PII Findings
-- Entity types map to DPDPA Section 2(t) personal data categories.
-- MINOR_DATA is flagged separately because Section 9 applies stricter rules.
-- ─────────────────────────────────────────────────────────────────────────────
CREATE TYPE pii_entity_type AS ENUM (
    'FULL_NAME',
    'EMAIL',
    'PHONE_NUMBER',
    'AADHAAR_NUMBER',
    'PAN_NUMBER',
    'PASSPORT_NUMBER',
    'BANK_ACCOUNT',
    'CREDIT_CARD',
    'HEALTH_RECORD_ID',
    'BIOMETRIC_DATA',
    'GPS_LOCATION',
    'IP_ADDRESS',
    'DEVICE_ID',
    'UPI_ID',
    'IFSC_CODE',
    'BEHAVIORAL_PROFILE',
    'MINOR_DATA'
);

-- DPDPA classification groups entity types into regulatory categories
-- used for consent purpose matching and cross-border transfer assessment.
CREATE TYPE dpdpa_category AS ENUM (
    'PERSONAL_IDENTITY',  -- Name, Aadhaar, PAN, Passport
    'FINANCIAL',          -- Bank account, credit card, UPI, IFSC
    'HEALTH',             -- Health records
    'BIOMETRIC',          -- Biometric data
    'CONTACT',            -- Email, phone
    'BEHAVIORAL',         -- Profiles, inferred attributes
    'LOCATION',           -- GPS
    'TECHNICAL',          -- IP address, device ID
    'MINOR'               -- Any data of a person under 18 (Section 9)
);

CREATE TYPE finding_risk_level AS ENUM (
    'CRITICAL',   -- Aadhaar, biometric, health, minor data
    'HIGH',       -- Financial identifiers, passport
    'MEDIUM',     -- Contact data, location
    'LOW'         -- Technical identifiers
);

CREATE TYPE finding_status AS ENUM (
    'OPEN',
    'ACKNOWLEDGED',
    'SUPPRESSED',   -- Retained by other law (see regulatory_conflict field)
    'REMEDIATED'
);

-- When a finding cannot be erased due to conflict with another Indian law,
-- this field documents the legal basis so DPBI can audit the exception.
CREATE TYPE regulatory_conflict_type AS ENUM (
    'RBI_5YR',        -- RBI record-keeping: 5 years
    'GST_6YR',        -- GST Act: 6 years
    'SEBI_5YR',       -- SEBI regulations: 5 years
    'CERT_IN_LOGS'    -- CERT-In log retention mandate
);

-- ─────────────────────────────────────────────────────────────────────────────
-- Data Flow Map
-- ─────────────────────────────────────────────────────────────────────────────
CREATE TYPE data_node_system_type AS ENUM (
    'DATA_PRINCIPAL_SOURCE', -- Where the DP originally submitted data
    'INTERNAL_APP',
    'VENDOR_CRM',
    'VENDOR_PAYMENT',
    'VENDOR_ACCOUNTING',
    'BACKUP_STORE',
    'LOG_SYSTEM',
    'DATA_LAKE',
    'ANALYTICS',
    'EXTERNAL_API'
);

CREATE TYPE data_flow_type AS ENUM (
    'INTENTIONAL',          -- Designed, documented data flow
    'UNINTENTIONAL_LEAK',   -- Discovered unexpected PII in logs, backups, etc.
    'BACKUP',               -- Backup/archive flows
    'ANALYTICS',            -- Data flowing to analytics/BI systems
    'SHARING'               -- Third-party data sharing
);

-- ─────────────────────────────────────────────────────────────────────────────
-- Contract Audit
-- ─────────────────────────────────────────────────────────────────────────────
CREATE TYPE contract_audit_status AS ENUM (
    'PROCESSING',
    'COMPLETED',
    'FAILED',
    'REQUIRES_REVIEW'
);

CREATE TYPE clause_type AS ENUM (
    'PARTIES',
    'PURPOSE',
    'DATA_CATEGORIES',
    'RETENTION',
    'SHARING',
    'CROSS_BORDER',
    'BREACH_NOTIFICATION',
    'DP_RIGHTS',
    'GOVERNING_LAW',
    'TERMINATION',
    'LIABILITY',
    'OTHER'
);

CREATE TYPE clause_compliance_status AS ENUM (
    'COMPLIANT',
    'NON_COMPLIANT',
    'MISSING',       -- Required clause absent from contract
    'PARTIAL',       -- Clause exists but is incomplete
    'NEEDS_REVIEW'   -- AI uncertain; routed to human reviewer
);

-- ─────────────────────────────────────────────────────────────────────────────
-- Discovery Audit Log
-- ─────────────────────────────────────────────────────────────────────────────
CREATE TYPE audit_actor_type AS ENUM (
    'USER',
    'ERASURE_ENGINE',  -- Module 3 service-to-service calls
    'SCHEDULER',       -- Temporal.io scheduled workflows
    'API_CLIENT'       -- External integrations via API key
);

CREATE TYPE audit_action AS ENUM (
    'SCAN_TRIGGERED',
    'CONNECTOR_ADDED',
    'CONNECTOR_REMOVED',
    'SNAPSHOT_REQUESTED',
    'SNAPSHOT_CONSUMED',
    'FINDING_ACKNOWLEDGED',
    'FINDING_SUPPRESSED',
    'CONTRACT_UPLOADED',
    'DPA_DOWNLOADED'
);

-- ─────────────────────────────────────────────────────────────────────────────
-- Tenant Plan
-- Determines which discovery features are available.
-- ─────────────────────────────────────────────────────────────────────────────
CREATE TYPE tenant_plan_tier AS ENUM (
    'GROWTH',       -- ₹4,999/mo — up to 5 connectors
    'ENTERPRISE',   -- ₹14,999/mo — unlimited connectors + contract audit
    'PLATFORM'      -- Custom — multi-tenant management
);
```

---

## 2. Table Definitions

### 2.1 `tenants`

**Role:** Registry of Data Fiduciaries (businesses) using TrustStack's discovery module. One row per customer. Tenant ID is derived from the JWT/API key — it is never accepted from the request body.

This table is intentionally a subset of the Consent Vault's `tenants` table — shared UUID primary key, so cross-module joins work without data duplication. In production, Module 2 reads tenant metadata from Module 1's read replica.

```sql
-- DPDPA context: Each tenant is a Data Fiduciary under Section 2(i).
-- plan_tier determines how many connectors and scans they can run.
-- default_language drives the language used for DPA generation and UI notices.

CREATE TABLE tenants (
    tenant_id        UUID PRIMARY KEY,                   -- Shared PK with consent-vault.tenants
    name             TEXT NOT NULL,
    slug             TEXT NOT NULL UNIQUE,                -- URL-safe identifier
    plan_tier        tenant_plan_tier NOT NULL DEFAULT 'GROWTH',
    -- BCP 47 code from the 22 Eighth Schedule languages (Rule 3 — notice language)
    default_language TEXT NOT NULL DEFAULT 'hi',
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at       TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_tenants_slug ON tenants(slug);

ALTER TABLE tenants ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenants_isolation ON tenants
    USING (tenant_id = current_setting('app.tenant_id', TRUE)::uuid);
```

---

### 2.2 `connector_configs`

**Role:** Stores configuration for each vendor system connector. Credentials are **never** stored in the database — only a reference to an AWS Secrets Manager ARN. The scan engine resolves the ARN at runtime. `scan_scope` lets tenants limit which tables, buckets, or log groups are crawled.

```sql
-- DPDPA context: Connector scope definition determines which systems are
-- included in Data Flow Map. Audit trail (discovery_audit_log) records
-- who added/removed connectors and when.

CREATE TABLE connector_configs (
    connector_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id         UUID NOT NULL REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    vendor_type       vendor_type NOT NULL,

    -- AWS Secrets Manager ARN — never raw credentials (Rule 4 security requirement)
    credentials_ref   TEXT NOT NULL,

    -- JSONB defining which tables/buckets/log groups to include or exclude.
    -- Example: {"include": ["users", "orders"], "exclude": ["audit_logs"]}
    scan_scope        JSONB NOT NULL DEFAULT '{}',

    last_scan_at      TIMESTAMPTZ,
    status            connector_status NOT NULL DEFAULT 'PENDING_AUTH',

    -- User who configured this connector (for audit trail)
    created_by        UUID NOT NULL,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_connector_tenant ON connector_configs(tenant_id, status);
CREATE INDEX idx_connector_last_scan ON connector_configs(tenant_id, last_scan_at NULLS FIRST);

ALTER TABLE connector_configs ENABLE ROW LEVEL SECURITY;
CREATE POLICY connector_tenant_isolation ON connector_configs
    USING (tenant_id = current_setting('app.tenant_id', TRUE)::uuid);
```

---

### 2.3 `scan_jobs`

**Role:** Tracks each scan execution — its type, scope, Temporal.io workflow ID, status, and outcome. The Temporal workflow ID allows the UI to poll progress via the workflow API. `error_details` stores structured errors with no PII (stack traces are sanitised before storage).

```sql
-- DPDPA context: Scan jobs are the mechanism by which TrustStack discovers
-- where a Data Principal's personal data is held across vendor systems.
-- DP_TARGETED_SCAN is triggered by the Erasure Engine (Module 3) before
-- initiating an erasure workflow to produce a current Data Flow Map snapshot.

CREATE TABLE scan_jobs (
    job_id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id            UUID NOT NULL REFERENCES tenants(tenant_id),
    job_type             scan_job_type NOT NULL,
    status               scan_job_status NOT NULL DEFAULT 'QUEUED',

    -- Which connectors are in scope for this scan (empty = all active connectors)
    connectors_included  UUID[] NOT NULL DEFAULT '{}',

    -- Temporal.io durable workflow ID — enables reliable long-running scans
    temporal_workflow_id TEXT,

    started_at           TIMESTAMPTZ,
    completed_at         TIMESTAMPTZ,

    -- Aggregate result (detailed results in pii_findings)
    findings_count       INTEGER NOT NULL DEFAULT 0,

    -- Structured error with no PII — stacktrace sanitised before storage
    error_details        JSONB,

    triggered_by         scan_trigger NOT NULL DEFAULT 'MANUAL',
    created_at           TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_scan_tenant_status ON scan_jobs(tenant_id, status, created_at DESC);
CREATE INDEX idx_scan_tenant_type ON scan_jobs(tenant_id, job_type, created_at DESC);
CREATE INDEX idx_scan_temporal ON scan_jobs(temporal_workflow_id) WHERE temporal_workflow_id IS NOT NULL;

ALTER TABLE scan_jobs ENABLE ROW LEVEL SECURITY;
CREATE POLICY scan_tenant_isolation ON scan_jobs
    USING (tenant_id = current_setting('app.tenant_id', TRUE)::uuid);
```

---

### 2.4 `pii_findings`

**Role:** Each row records one instance of detected PII in a vendor system. The actual PII value is **never stored** — only its SHA-256 hash (for deduplication across scans) and its classification. `source_location` stores the structural path (table name, field name, S3 key prefix) but never a value.

When a finding cannot be remediated because another Indian law mandates retention (e.g., RBI 5-year rule), `regulatory_conflict` documents the exception so DPBI auditors can understand why the data persists past the DPDPA erasure window.

```sql
-- DPDPA context: Findings feed the Data Flow Map and drive the Erasure Engine's
-- deletion targets. Only SHA-256 hashes of values are stored — the Discovery
-- Engine has no access to raw personal data after classification (Zero-Knowledge
-- design, consistent with dual-role prohibition under Section 2(g)).
--
-- MINOR_DATA findings (entity_type = 'MINOR_DATA') trigger an automatic
-- escalation flag because Section 9 prohibits targeted advertising and profiling
-- of persons under 18.

CREATE TABLE pii_findings (
    finding_id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    job_id               UUID NOT NULL REFERENCES scan_jobs(job_id),
    tenant_id            UUID NOT NULL REFERENCES tenants(tenant_id),
    connector_id         UUID NOT NULL REFERENCES connector_configs(connector_id),

    entity_type          pii_entity_type NOT NULL,
    dpdpa_category       dpdpa_category NOT NULL,

    -- SHA-256 of the actual value — used for deduplication only.
    -- The raw value is NEVER stored anywhere in this schema.
    value_hash           TEXT NOT NULL,

    -- Human-readable connector name (e.g., "Zoho CRM — Production")
    source_system        TEXT NOT NULL,

    -- Structural path to where the value was found. Contains no PII values.
    -- Example: {"table": "contacts", "field": "mobile_number", "sample_row_count": 1420}
    -- Example (S3): {"bucket": "acme-crm-backup", "prefix": "2025/contacts/", "file_count": 12}
    source_location      JSONB NOT NULL,

    -- Model confidence in the classification (0.00 = uncertain, 1.00 = certain)
    confidence           DECIMAL(4,3) NOT NULL CHECK (confidence BETWEEN 0 AND 1),
    risk_level           finding_risk_level NOT NULL,
    status               finding_status NOT NULL DEFAULT 'OPEN',

    -- Who acknowledged this finding (user UUID from tenant's IAM)
    acknowledged_by      UUID,
    acknowledged_at      TIMESTAMPTZ,

    -- Required when status = 'SUPPRESSED': explains why erasure is blocked
    suppression_reason   TEXT,

    -- If null: no conflict — DPDPA erasure timelines apply normally.
    -- If set: erasure is blocked by this other law; DPBI exception documented here.
    regulatory_conflict  regulatory_conflict_type,

    found_at             TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_at           TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Support fast filtered queries by the DPO dashboard and Erasure Engine
CREATE INDEX idx_findings_tenant_risk ON pii_findings(tenant_id, risk_level, status, found_at DESC);
CREATE INDEX idx_findings_connector ON pii_findings(tenant_id, connector_id, status);
CREATE INDEX idx_findings_category ON pii_findings(tenant_id, dpdpa_category, status);
CREATE INDEX idx_findings_job ON pii_findings(job_id);
-- Hash-based deduplication: same value in same location should not create duplicate findings
CREATE UNIQUE INDEX idx_findings_dedup ON pii_findings(tenant_id, connector_id, value_hash, source_location)
    WHERE status != 'REMEDIATED';

ALTER TABLE pii_findings ENABLE ROW LEVEL SECURITY;
CREATE POLICY findings_tenant_isolation ON pii_findings
    USING (tenant_id = current_setting('app.tenant_id', TRUE)::uuid);
```

---

### 2.5 `data_flow_nodes`

**Role:** Each node represents one system in the tenant's data ecosystem — an internal app, a vendor SaaS, a backup store, etc. Nodes are the vertices of the Data Flow Map graph. `metadata` records cross-border transfer flags so that Section 16 (data residency) compliance can be assessed per node.

```sql
-- DPDPA context: The Data Flow Map gives tenants a real-time view of where
-- personal data flows across their systems. Nodes with cross_border_flag=true
-- are flagged for Section 16 assessment — DPBI must be able to access consent
-- logs even for data on foreign infrastructure.
--
-- connector_id is nullable: internal systems (e.g., the tenant's own app) appear
-- as nodes in the map even if TrustStack does not crawl them directly.

CREATE TABLE data_flow_nodes (
    node_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id        UUID NOT NULL REFERENCES tenants(tenant_id),

    system_name      TEXT NOT NULL,  -- e.g., "Zoho CRM — Production"
    system_type      data_node_system_type NOT NULL,

    -- NULL for internal/manually-added nodes
    connector_id     UUID REFERENCES connector_configs(connector_id) ON DELETE SET NULL,

    -- {region: "ap-south-1", cloud_provider: "AWS", cross_border_flag: false}
    -- cross_border_flag=true triggers Section 16 assessment requirement
    metadata         JSONB NOT NULL DEFAULT '{}',

    -- Composite risk score (0.00-1.00) computed from findings on this node
    risk_score       DECIMAL(4,3) NOT NULL DEFAULT 0 CHECK (risk_score BETWEEN 0 AND 1),

    last_updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_nodes_tenant ON data_flow_nodes(tenant_id, system_type);
CREATE INDEX idx_nodes_risk ON data_flow_nodes(tenant_id, risk_score DESC);

ALTER TABLE data_flow_nodes ENABLE ROW LEVEL SECURITY;
CREATE POLICY nodes_tenant_isolation ON data_flow_nodes
    USING (tenant_id = current_setting('app.tenant_id', TRUE)::uuid);
```

---

### 2.6 `data_flow_edges`

**Role:** Directed edges between nodes representing observed or inferred data flows. `data_categories` is an array of `dpdpa_category` values — it records *what* flows, not the actual data. `cross_border` triggers a Section 16 flag when true. `flow_type = 'UNINTENTIONAL_LEAK'` is set when a scan finds PII in a system that has no declared purpose for that data category.

```sql
-- DPDPA context: Edges where flow_type = 'UNINTENTIONAL_LEAK' represent
-- compliance violations — PII exists in a system with no declared consent
-- purpose. These require immediate attention from the DPO.
--
-- cross_border = true on any edge triggers a Section 16 assessment: DPBI
-- must retain the ability to access consent logs even for cross-border flows.

CREATE TABLE data_flow_edges (
    edge_id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id                UUID NOT NULL REFERENCES tenants(tenant_id),

    source_node_id           UUID NOT NULL REFERENCES data_flow_nodes(node_id),
    destination_node_id      UUID NOT NULL REFERENCES data_flow_nodes(node_id),

    -- Array of DPDPA categories flowing on this edge (categories, not values)
    data_categories          dpdpa_category[] NOT NULL,

    flow_type                data_flow_type NOT NULL DEFAULT 'INTENTIONAL',

    -- Stated purposes for this data flow (free text from connector metadata)
    purposes                 TEXT[] NOT NULL DEFAULT '{}',

    estimated_records_per_day INTEGER,

    -- Section 16: flag cross-border flows for DPBI access assessment
    cross_border             BOOLEAN NOT NULL DEFAULT FALSE,

    -- Composite risk score for this flow
    risk_score               DECIMAL(4,3) NOT NULL DEFAULT 0 CHECK (risk_score BETWEEN 0 AND 1),

    last_data_seen_at        TIMESTAMPTZ,
    discovered_in_job_id     UUID NOT NULL REFERENCES scan_jobs(job_id),

    created_at               TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at               TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_edges_tenant ON data_flow_edges(tenant_id, flow_type);
CREATE INDEX idx_edges_source ON data_flow_edges(tenant_id, source_node_id);
CREATE INDEX idx_edges_destination ON data_flow_edges(tenant_id, destination_node_id);
CREATE INDEX idx_edges_cross_border ON data_flow_edges(tenant_id) WHERE cross_border = TRUE;

ALTER TABLE data_flow_edges ENABLE ROW LEVEL SECURITY;
CREATE POLICY edges_tenant_isolation ON data_flow_edges
    USING (tenant_id = current_setting('app.tenant_id', TRUE)::uuid);
```

---

### 2.7 `snapshot_tokens`

**Role:** One-time-use signed tokens issued exclusively to the Erasure Engine (Module 3). When the Erasure Engine needs to run erasure for a specific Data Principal, it first calls the Snapshot API to get that DP's current data locations. The token expires in 15 minutes and can only be consumed once — this prevents replay attacks and ensures the Erasure Engine always acts on a current, authorised snapshot.

The `dp_identifier_hash` is a SHA-256 hash of the DP's identifier — the raw identifier is never present in this table or the API response.

```sql
-- DPDPA context: The Snapshot API satisfies the Erasure Engine's need to know
-- which vendor systems hold a specific DP's data before triggering deletion.
-- One-time-use tokens (Rule 4 audit integrity): each erasure event is linked
-- to exactly one authorised snapshot, preventing stale or replayed data maps
-- from causing incomplete erasures.
--
-- Token signing: JWT signed with ECDSA P-256 (same signing infrastructure as
-- consent artifacts) — tamper-evident and verifiable by the Erasure Engine
-- without a round-trip to the Discovery API.

CREATE TABLE snapshot_tokens (
    token_id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id            UUID NOT NULL REFERENCES tenants(tenant_id),

    -- SHA-256 of the DP identifier — never the raw identifier
    dp_identifier_hash   TEXT NOT NULL,

    -- Signed JWT (ECDSA P-256). One-time use — invalidated after consumption.
    token_value          TEXT NOT NULL UNIQUE,

    -- 15-minute TTL from creation (short window limits exposure if intercepted)
    expires_at           TIMESTAMPTZ NOT NULL,

    -- Consumption tracking (append-only semantics enforced by trigger below)
    consumed             BOOLEAN NOT NULL DEFAULT FALSE,
    consumed_at          TIMESTAMPTZ,

    -- Identity of the Erasure Engine instance that consumed the token
    consumed_by          TEXT,  -- e.g., "erasure-engine/v1/worker-7"

    -- IAM principal that requested the token (for audit)
    requested_by         TEXT NOT NULL,

    -- Which data_flow_nodes are included in this snapshot
    node_ids_included    UUID[] NOT NULL DEFAULT '{}',

    created_at           TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_snapshot_token_value ON snapshot_tokens(token_value);
CREATE INDEX idx_snapshot_tenant_dp ON snapshot_tokens(tenant_id, dp_identifier_hash, created_at DESC);
CREATE INDEX idx_snapshot_unexpired ON snapshot_tokens(expires_at) WHERE consumed = FALSE;

-- Once consumed = true, it cannot be reset to false.
-- This enforces one-time-use semantics at the database level.
CREATE OR REPLACE FUNCTION enforce_snapshot_token_one_time_use()
RETURNS TRIGGER AS $$
BEGIN
    IF OLD.consumed = TRUE AND NEW.consumed = FALSE THEN
        RAISE EXCEPTION
            'snapshot_tokens: consumed token (id=%) cannot be un-consumed. ' ||
            'One-time-use semantics are required for Erasure Engine integrity.',
            OLD.token_id;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_snapshot_one_time_use
BEFORE UPDATE ON snapshot_tokens
FOR EACH ROW EXECUTE FUNCTION enforce_snapshot_token_one_time_use();

ALTER TABLE snapshot_tokens ENABLE ROW LEVEL SECURITY;
-- Erasure Engine uses a service API key scoped to ERASURE_ENGINE role;
-- it can see all tenants' snapshot tokens it requested.
-- DPO users can only see their own tenant's tokens.
CREATE POLICY snapshot_tenant_isolation ON snapshot_tokens
    USING (
        tenant_id = current_setting('app.tenant_id', TRUE)::uuid
        OR current_setting('app.role', TRUE) = 'ERASURE_ENGINE'
    );
```

---

### 2.8 `contract_audits`

**Role:** Tracks each uploaded vendor contract through its audit lifecycle. Contracts are stored in S3 (encrypted at rest); this table stores the S3 key, metadata, and aggregate compliance scores. `contract_hash` (SHA-256 of file content) ensures deduplication — uploading the same contract twice is detected and the second audit is linked to the first.

`ocr_used` and `ocr_confidence` record whether the document required OCR (scanned PDFs), which affects the reliability of clause extraction and compliance scoring.

```sql
-- DPDPA context: Vendor contracts (with processors, sub-processors, cloud vendors)
-- must include mandatory clauses under DPDPA — data categories, purpose, retention,
-- breach notification timelines, DP rights, governing law. Missing or non-compliant
-- clauses are flagged for remediation or for DPA generation.
--
-- Multilingual support: NLP pipeline handles all 22 Eighth Schedule languages.
-- detected_language stores the ISO code of the contract's primary language.

CREATE TABLE contract_audits (
    audit_id                 UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id                UUID NOT NULL REFERENCES tenants(tenant_id),

    vendor_name              TEXT NOT NULL,
    contract_filename        TEXT NOT NULL,

    -- SHA-256 of uploaded file content — used for deduplication
    contract_hash            TEXT NOT NULL,

    -- ISO 639-1 or ISO 639-3 code detected by the NLP pipeline
    -- (e.g., 'hi', 'ta', 'mr', 'mai', 'sat', 'mni')
    detected_language        TEXT,

    file_size_bytes          INTEGER NOT NULL,
    page_count               INTEGER,

    status                   contract_audit_status NOT NULL DEFAULT 'PROCESSING',

    -- 0 = fully non-compliant, 100 = fully compliant
    overall_compliance_score DECIMAL(5,2) CHECK (overall_compliance_score BETWEEN 0 AND 100),

    -- Counts of flagged clauses by severity (for dashboard summary)
    critical_flags_count     INTEGER NOT NULL DEFAULT 0,
    high_flags_count         INTEGER NOT NULL DEFAULT 0,

    -- Whether a DPA has been generated from this audit
    dpa_generated            BOOLEAN NOT NULL DEFAULT FALSE,
    dpa_generation_id        UUID,  -- FK added below after dpa_generations table exists

    -- OCR metadata (relevant for scanned contracts)
    ocr_used                 BOOLEAN NOT NULL DEFAULT FALSE,
    ocr_confidence           DECIMAL(4,3) CHECK (ocr_confidence BETWEEN 0 AND 1),

    started_at               TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at             TIMESTAMPTZ,

    created_by               UUID NOT NULL  -- User who uploaded the contract
);

CREATE INDEX idx_contract_tenant ON contract_audits(tenant_id, status, started_at DESC);
CREATE INDEX idx_contract_hash ON contract_audits(tenant_id, contract_hash);
CREATE INDEX idx_contract_vendor ON contract_audits(tenant_id, vendor_name);

ALTER TABLE contract_audits ENABLE ROW LEVEL SECURITY;
CREATE POLICY contract_tenant_isolation ON contract_audits
    USING (tenant_id = current_setting('app.tenant_id', TRUE)::uuid);
```

---

### 2.9 `contract_clauses`

**Role:** Each row is one extracted clause from a contract. `original_text` is the raw text in the detected language; `translated_text` is the English translation (generated only if the contract is non-English). `compliance_flags` is a JSONB array of structured flag objects — each flag maps to a specific DPDPA section.

```sql
-- DPDPA context: Clause-level granularity lets the DPO see exactly which
-- part of a contract is non-compliant and which DPDPA section it violates.
-- compliance_flags[].dpdpa_section references the specific rule (e.g.,
-- "Section 5, Rule 3" for inadequate notice, "Rule 7" for missing breach
-- notification clause).
--
-- MISSING clauses (compliance_status = 'MISSING') represent clauses that
-- are required by DPDPA but entirely absent from the contract.

CREATE TABLE contract_clauses (
    clause_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    audit_id          UUID NOT NULL REFERENCES contract_audits(audit_id) ON DELETE CASCADE,
    tenant_id         UUID NOT NULL REFERENCES tenants(tenant_id),

    clause_type       clause_type NOT NULL,

    -- Original clause text in the contract's detected language
    original_text     TEXT NOT NULL,

    -- English translation (NULL if contract is already in English)
    translated_text   TEXT,

    page_number       INTEGER,

    compliance_status clause_compliance_status NOT NULL,

    -- Array of flag objects. Each flag:
    -- {
    --   "flag_id": "uuid",
    --   "severity": "CRITICAL|HIGH|MEDIUM|LOW",
    --   "dpdpa_section": "Section 5, Rule 3",
    --   "description": "Notice does not specify retention period",
    --   "remediation": "Add explicit retention duration tied to purpose"
    -- }
    compliance_flags  JSONB NOT NULL DEFAULT '[]',

    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_clauses_audit ON contract_clauses(audit_id, clause_type);
CREATE INDEX idx_clauses_compliance ON contract_clauses(tenant_id, compliance_status);

ALTER TABLE contract_clauses ENABLE ROW LEVEL SECURITY;
CREATE POLICY clauses_tenant_isolation ON contract_clauses
    USING (tenant_id = current_setting('app.tenant_id', TRUE)::uuid);
```

---

### 2.10 `dpa_generations`

**Role:** Tracks generated Data Processing Agreements. A DPA is generated from an audit and is produced in the same language as the original contract (DPDPA Section 5 / Rule 3: notice must be in the user's language). Both DOCX and signed PDF versions are stored in S3. The PDF is ECDSA-signed using AWS KMS — the same signing infrastructure used by the Consent Vault.

```sql
-- DPDPA context: The generated DPA satisfies the mandatory contractual
-- requirements between Data Fiduciary and Data Processor (Section 28-style
-- obligations). ECDSA signature ensures the generated document is tamper-evident
-- and the signing key ID is retained so DPBI can verify authenticity.
--
-- The DPA is generated in the contract's detected language (detected_language
-- from contract_audits) — this satisfies Rule 3 multilingual notice requirements
-- for the contractual agreement itself.

CREATE TABLE dpa_generations (
    dpa_id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    audit_id             UUID NOT NULL REFERENCES contract_audits(audit_id),
    tenant_id            UUID NOT NULL REFERENCES tenants(tenant_id),

    vendor_name          TEXT NOT NULL,

    -- ISO language code — must match contract_audits.detected_language
    generated_language   TEXT NOT NULL,

    -- S3 object keys (files stored encrypted in AWS ap-south-1)
    dpa_docx_s3_key      TEXT NOT NULL,
    dpa_pdf_s3_key       TEXT NOT NULL,

    -- ECDSA P-256 signature over the PDF content hash
    ecdsa_signature      TEXT NOT NULL,
    signing_key_id       TEXT NOT NULL,  -- AWS KMS key ARN
    signature_algorithm  TEXT NOT NULL DEFAULT 'ECDSA_SHA_256',

    -- Version of the DPA template used (for reproducibility and future audits)
    template_version     TEXT NOT NULL,

    created_at           TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Add the deferred FK from contract_audits back to dpa_generations
ALTER TABLE contract_audits
    ADD CONSTRAINT fk_contract_dpa_generation
    FOREIGN KEY (dpa_generation_id) REFERENCES dpa_generations(dpa_id);

CREATE INDEX idx_dpa_audit ON dpa_generations(audit_id);
CREATE INDEX idx_dpa_tenant ON dpa_generations(tenant_id, created_at DESC);

ALTER TABLE dpa_generations ENABLE ROW LEVEL SECURITY;
CREATE POLICY dpa_tenant_isolation ON dpa_generations
    USING (tenant_id = current_setting('app.tenant_id', TRUE)::uuid);
```

---

### 2.11 `discovery_audit_log`

**Role:** Append-only, immutable audit trail of every significant action in the Discovery module. This is the DPBI-facing audit record. No PII appears in this table — IP addresses are stored as SHA-256 hashes. The table is enforced append-only by trigger (mirroring the Consent Vault's `audit_log` pattern).

DPDPA Rule 4 requires a 7-year retention of processing records. DPDPA Rules 2025 also require a minimum 1-year retention of security-relevant processing logs (CERT-In overlap). This table satisfies both.

```sql
-- DPDPA context: Rule 4 requires immutable processing logs retained for 7 years.
-- This table is the Module 2 equivalent of consent-vault's audit_log.
-- The ERASURE_ENGINE actor_type records cross-module snapshot consumption events,
-- giving a complete audit trail of every erasure-driven data map access.
--
-- No raw IP addresses, no raw identifiers — all hashed before storage.
-- Structured logging convention: no personal data in metadata JSONB.

CREATE TABLE discovery_audit_log (
    log_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID REFERENCES tenants(tenant_id),

    actor_id        UUID NOT NULL,  -- User UUID or service principal UUID
    actor_type      audit_actor_type NOT NULL,

    action          audit_action NOT NULL,

    -- What kind of resource was acted on (e.g., 'connector', 'scan_job', 'finding')
    resource_type   TEXT NOT NULL,
    resource_id     UUID,

    -- Structured metadata — MUST NOT contain PII (enforced by code convention)
    -- Example: {"connector_type": "ZOHO_CRM", "scan_job_type": "INCREMENTAL_SCAN"}
    metadata        JSONB NOT NULL DEFAULT '{}',

    -- SHA-256 hash of the client IP — never raw IP stored (No PII Logging convention)
    ip_address_hash TEXT,

    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_audit_log_tenant_time ON discovery_audit_log(tenant_id, created_at DESC);
CREATE INDEX idx_audit_log_action ON discovery_audit_log(tenant_id, action, created_at DESC);
CREATE INDEX idx_audit_log_resource ON discovery_audit_log(resource_type, resource_id);

-- Append-only enforcement: no UPDATE or DELETE permitted.
-- Mirrors consent-vault's enforce_consent_immutability() pattern.
CREATE OR REPLACE FUNCTION enforce_discovery_audit_immutability()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'UPDATE' THEN
        RAISE EXCEPTION
            'discovery_audit_log is append-only (DPDPA Rule 4). UPDATE not permitted. Log ID: %',
            OLD.log_id;
    END IF;
    IF TG_OP = 'DELETE' THEN
        RAISE EXCEPTION
            'discovery_audit_log is append-only (DPDPA Rule 4). DELETE not permitted. Log ID: %',
            OLD.log_id;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_discovery_audit_immutability
BEFORE UPDATE OR DELETE ON discovery_audit_log
FOR EACH ROW EXECUTE FUNCTION enforce_discovery_audit_immutability();

-- RLS: tenants see their own logs; TrustStack admin sees all
ALTER TABLE discovery_audit_log ENABLE ROW LEVEL SECURITY;
CREATE POLICY audit_log_tenant_isolation ON discovery_audit_log
    USING (
        tenant_id = current_setting('app.tenant_id', TRUE)::uuid
        OR current_setting('app.role', TRUE) = 'TRUSTSTACK_ADMIN'
    );
```

---

## 3. Entity Relationship Diagram

```
tenants (1)
    │
    ├──── connector_configs (N)
    │          │
    │          ├──── scan_jobs (N) [via connectors_included UUID[]]
    │          │         │
    │          │         └──── pii_findings (N)
    │          │
    │          └──── data_flow_nodes (N) [connector_id FK — nullable]
    │
    ├──── scan_jobs (N)
    │         │
    │         └──── data_flow_edges (N) [discovered_in_job_id]
    │
    ├──── data_flow_nodes (N)
    │         │
    │         └──── data_flow_edges (N) [source_node_id / destination_node_id]
    │
    ├──── snapshot_tokens (N)
    │         │
    │         └──── node_ids_included UUID[] → data_flow_nodes
    │
    ├──── contract_audits (N)
    │         │
    │         ├──── contract_clauses (N)
    │         │
    │         └──── dpa_generations (1, nullable)
    │
    └──── discovery_audit_log (N) [append-only]
```

---

## 4. Cross-Module Boundaries

| Module | Direction | Mechanism | What is shared |
|--------|-----------|-----------|----------------|
| Consent Vault (Module 1) → Discovery | Inbound trigger | Kafka event `consent.withdrawn` | `dp_identifier_hash`, `tenant_id` — triggers `DP_TARGETED_SCAN` |
| Discovery → Erasure Engine (Module 3) | Outbound API | Snapshot API (`POST /snapshot/request`, `GET /snapshot/{token}`) | Data Flow Map nodes for a specific DP hash |
| Discovery ← Erasure Engine (Module 3) | Inbound API | `POST /snapshot/request` with `X-API-Key` (ERASURE_ENGINE scope) | Justification type, DP identifier hash |

The Discovery Engine has **no read or write access** to consent-vault's `consent_records`, `dp_identifiers`, or any Consent Vault table. The Erasure Engine receives a Data Flow Map snapshot (node metadata + data categories) but never raw PII. This satisfies the dual-role prohibition under Section 2(g).

---

## 5. Row-Level Security Summary

| Table | Policy | Bypass |
|-------|--------|--------|
| `tenants` | `tenant_id = app.tenant_id` | None |
| `connector_configs` | `tenant_id = app.tenant_id` | None |
| `scan_jobs` | `tenant_id = app.tenant_id` | None |
| `pii_findings` | `tenant_id = app.tenant_id` | None |
| `data_flow_nodes` | `tenant_id = app.tenant_id` | None |
| `data_flow_edges` | `tenant_id = app.tenant_id` | None |
| `snapshot_tokens` | `tenant_id = app.tenant_id` | `app.role = ERASURE_ENGINE` |
| `contract_audits` | `tenant_id = app.tenant_id` | None |
| `contract_clauses` | `tenant_id = app.tenant_id` | None |
| `dpa_generations` | `tenant_id = app.tenant_id` | None |
| `discovery_audit_log` | `tenant_id = app.tenant_id` | `app.role = TRUSTSTACK_ADMIN` |

---

## 6. Data Retention Policy

| Table | Retention | Mechanism |
|-------|-----------|-----------|
| `pii_findings` | Until remediated + 1yr processing log | Hard delete after `REMEDIATED` + 1yr; status tracked in `discovery_audit_log` |
| `scan_jobs` | 7 years (Rule 4 processing records) | Archive to S3 cold storage after 1yr |
| `snapshot_tokens` | 7 years (Rule 4 audit integrity) | Immutable; rows retained but token_value is invalidated post-expiry |
| `discovery_audit_log` | 7 years (Rule 4) | Append-only; no deletion path |
| `contract_audits` | 7 years (contractual compliance evidence) | Archive to cold storage after 2yr |
| `dpa_generations` | 7 years (Rule 4 — DPA is a compliance artifact) | S3 objects versioned; table row retained |

---

## 7. DPDPA Compliance Annotations

| Table | Section | Requirement Met |
|-------|---------|-----------------|
| `pii_findings` | Section 2(t) | PII categories mapped to DPDPA classification |
| `pii_findings` | Section 9 | `entity_type = 'MINOR_DATA'` triggers Section 9 escalation |
| `pii_findings` | Zero-Knowledge | Only SHA-256 hash of value stored — raw PII never persisted |
| `data_flow_edges` | Section 16 | `cross_border` flag drives residency compliance assessment |
| `snapshot_tokens` | Section 2(g) | One-time-use + 15min TTL prevents dual-role data accumulation |
| `snapshot_tokens` | Rule 4 | Token consumption events logged in `discovery_audit_log` |
| `contract_clauses` | Section 5 / Rule 3 | `original_text` in detected language; `translated_text` for English review |
| `dpa_generations` | Section 5 / Rule 3 | DPA generated in contract's detected language |
| `dpa_generations` | Rule 4 | ECDSA-signed — tamper-evident for DPBI audit |
| `discovery_audit_log` | Rule 4 | Append-only, 7-year retention, no PII |
| `pii_findings.regulatory_conflict` | Rule 7 / Other laws | Documents RBI/GST/SEBI/CERT-In exceptions to DPDPA erasure |
