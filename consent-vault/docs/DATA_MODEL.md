# Data Model — SahmatOS Consent Vault

**Version:** 1.0  
**Date:** 2026-04-22  
**Database:** PostgreSQL 16  

---

## Overview

All consent data is stored in **append-only tables** — no UPDATE or DELETE operations permitted by design. This satisfies:
- DPDPA Rule 4: 7-year immutable consent record retention
- ECDSA audit integrity: records match the signed artifact content
- DPBI audit: complete, unaltered history of every consent action

Soft deletes are explicitly **prohibited** — DPDPA requires hard deletion for erasure, and soft deletion would create misleading records.

---

## 1. Schema Definitions

### 1.1 Tenants (Data Fiduciaries)

```sql
-- DPDPA context: Each tenant is a registered Data Fiduciary (business) that
-- collects personal data. TrustStack acts as their Registered Consent Manager.

CREATE TABLE tenants (
    id                   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- Business identity
    name                 TEXT NOT NULL,
    legal_name           TEXT NOT NULL,              -- Registered company name
    cin                  TEXT,                       -- MCA Company Identification Number
    gstin                TEXT,                       -- GST registration
    
    -- DPDPA classification (determines erasure timeline — Third Schedule)
    df_class             TEXT NOT NULL CHECK (df_class IN (
                            'ECOMMERCE_GT_2CR',       -- E-commerce ≥2Cr users: 3yr erasure
                            'GAMING_GT_50L',          -- Online gaming ≥50L users: 3yr erasure
                            'SOCIAL_GT_2CR',          -- Social media ≥2Cr users: 3yr erasure
                            'STANDARD'                -- All others: erase on purpose fulfilment
                         )) DEFAULT 'STANDARD',
    
    -- Configuration
    default_language     TEXT NOT NULL DEFAULT 'hi', -- BCP 47
    webhook_url          TEXT,                        -- Webhook for consent events
    webhook_secret_hash  TEXT,                        -- HMAC secret for webhook signature
    
    -- KMS key references (Rule 4: per-tenant independent keys)
    kms_symmetric_key_arn   TEXT NOT NULL,            -- AES-256 data encryption
    kms_signing_key_arn     TEXT NOT NULL,            -- ECDSA P-256 consent signing
    
    -- Subscription
    plan                 TEXT NOT NULL DEFAULT 'growth' CHECK (plan IN (
                            'awareness_pro', 'growth', 'enterprise', 'platform'
                         )),
    
    -- Metadata
    created_at           TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    activated_at         TIMESTAMPTZ,
    suspended_at         TIMESTAMPTZ,
    
    -- Data residency declaration (Section 16 DPDPA)
    data_residency_region TEXT NOT NULL DEFAULT 'ap-south-1' CHECK (
                            data_residency_region IN ('ap-south-1', 'asia-south1')
                         )
);

CREATE INDEX idx_tenants_cin ON tenants(cin) WHERE cin IS NOT NULL;
```

### 1.2 Consent Purposes

```sql
-- DPDPA context: Section 6 requires separate consent per "purpose".
-- Each purpose must describe exactly what data is collected, why, and for how long.

CREATE TABLE consent_purposes (
    id                   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id            UUID NOT NULL REFERENCES tenants(id),
    
    -- Purpose identity
    code                 TEXT NOT NULL,              -- Tenant-defined key: 'marketing_sms'
    version              INTEGER NOT NULL DEFAULT 1, -- Incremented when purpose text changes
    is_current           BOOLEAN NOT NULL DEFAULT TRUE,
    
    -- DPDPA required fields (Section 5 / Rule 3)
    data_categories      TEXT[] NOT NULL,            -- What data: ['phone', 'purchase_history']
    processing_activities TEXT[] NOT NULL,           -- What done with it: ['send_sms', 'personalise']
    retention_period     TEXT NOT NULL,              -- Human-readable: '2 years from last purchase'
    retention_days       INTEGER,                    -- Calculated value for automation
    
    -- Consent Manager registration metadata
    is_sensitive         BOOLEAN NOT NULL DEFAULT FALSE, -- Rule 4: sensitive data needs stricter notice
    requires_parental_consent BOOLEAN NOT NULL DEFAULT FALSE, -- Section 9
    
    -- Translation status (all 22 languages required before production use)
    translation_status   JSONB NOT NULL DEFAULT '{}', -- {hi: 'approved', ta: 'pending', ...}
    
    -- Third-party data sharing (required in notice)
    shares_with_third_parties BOOLEAN NOT NULL DEFAULT FALSE,
    third_party_names    TEXT[],
    
    created_at           TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    approved_at          TIMESTAMPTZ,               -- Set after legal review
    
    UNIQUE (tenant_id, code, version)
);

CREATE INDEX idx_purposes_tenant ON consent_purposes(tenant_id, code) WHERE is_current = TRUE;
```

### 1.3 Consent Purpose Translations

```sql
-- DPDPA context: Section 5 Rule 3 requires notice in user's interface language.
-- Every purpose must be translated into all 22 Eighth Schedule languages.

CREATE TABLE consent_purpose_translations (
    id                   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    purpose_id           UUID NOT NULL REFERENCES consent_purposes(id),
    
    -- Language (BCP 47 with script subtags for ambiguous cases)
    language_code        TEXT NOT NULL,              -- 'hi', 'ta', 'ur', 'ks-Arab', 'mni-Mtei', etc.
    
    -- Translated content
    title                TEXT NOT NULL,              -- Short purpose name in this language
    description          TEXT NOT NULL,              -- Full notice text (plain language, ≤300 words)
    data_categories_text TEXT NOT NULL,              -- Localised data category descriptions
    your_rights_text     TEXT NOT NULL,              -- Rights explanation in this language
    withdrawal_text      TEXT NOT NULL,              -- How to withdraw (must be as clear as grant)
    
    -- Review chain
    translated_by        TEXT,                       -- 'llm-v2.3', 'human-translator-id'
    reviewed_by          TEXT,                       -- Native speaker reviewer
    legal_reviewed_by    TEXT,                       -- Legal reviewer
    status               TEXT NOT NULL DEFAULT 'pending' CHECK (
                            status IN ('pending', 'reviewed', 'approved', 'rejected')
                         ),
    approved_at          TIMESTAMPTZ,
    
    -- Rendering metadata
    text_direction       TEXT NOT NULL DEFAULT 'ltr' CHECK (text_direction IN ('ltr', 'rtl')),
    script_code          TEXT NOT NULL,              -- 'Deva', 'Tamil', 'Arab', 'Mtei', 'Olck', etc.
    font_family          TEXT NOT NULL,              -- CSS font-family for this script
    
    created_at           TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    
    UNIQUE (purpose_id, language_code)
);

CREATE INDEX idx_translations_purpose_lang ON consent_purpose_translations(purpose_id, language_code)
    WHERE status = 'approved';
```

### 1.4 Consent Records (Core Append-Only Ledger)

```sql
-- DPDPA context: Rule 4 requires 7-year retention of all consent records.
-- This is the legally binding audit trail. APPEND ONLY — no modifications ever.
-- ECDSA-signed artifacts ensure tamper-evidence detectable by DPBI.

CREATE TABLE consent_records (
    id                   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- Tenant and purpose linkage
    tenant_id            UUID NOT NULL REFERENCES tenants(id),
    purpose_id           UUID NOT NULL REFERENCES consent_purposes(id),
    
    -- Data Principal identification (NEVER store raw PII — DPDPA Zero-Knowledge design)
    dp_identifier_hash   TEXT NOT NULL,              -- HMAC-SHA256(raw_id, tenant_secret)
    dp_identifier_enc    TEXT,                       -- AES-256 encrypted raw id (KMS), separate table
    
    -- Consent action
    action               TEXT NOT NULL CHECK (action IN ('GRANTED', 'WITHDRAWN', 'EXPIRED')),
    
    -- Collection evidence
    collection_method    TEXT NOT NULL CHECK (collection_method IN (
                            'WIDGET',          -- Web/mobile widget
                            'API',             -- Server-side API call
                            'WHATSAPP',        -- WhatsApp Business conversation
                            'VERBAL_AADHAAR'   -- Aadhaar eKYC + verbal consent (for parental)
                         )),
    
    -- Language context (required by Rule 3 — notice must be in user's language)
    language_code        TEXT NOT NULL,              -- BCP 47: 'hi', 'ta', 'ur', 'ks-Arab', etc.
    notice_version       INTEGER NOT NULL,           -- purpose translation version at consent time
    
    -- Signed consent artifact (the legal document)
    consent_artifact     JSONB NOT NULL,             -- Full artifact content (signed payload)
    signature            TEXT NOT NULL,              -- ECDSA P-256 base64 signature
    signature_key_id     TEXT NOT NULL,              -- AWS KMS key ARN used for signing
    
    -- Evidence metadata (for breach/audit — hashed/tokenised, never raw PII)
    ip_address_hash      TEXT,                       -- SHA-256 of IP (no raw IP stored)
    user_agent_hash      TEXT,                       -- SHA-256 of User-Agent
    session_id_hash      TEXT,                       -- SHA-256 of session ID
    
    -- Parental consent (Section 9 — Data Principals under 18)
    is_minor             BOOLEAN NOT NULL DEFAULT FALSE,
    parental_consent_id  UUID,                       -- References parent's consent_records row
    guardian_id_hash     TEXT,                       -- Aadhaar-based guardian identifier (hashed)
    
    -- Expiry (optional — some purposes have time-bounded consent)
    expires_at           TIMESTAMPTZ,
    
    -- Channel-specific evidence
    channel_evidence     JSONB,                      -- WhatsApp: {waba_id, message_id, read_receipt}
    
    -- Immutability enforcement
    created_at           TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    
    -- Rule 4: data residency declaration on every record
    data_residency_region TEXT NOT NULL DEFAULT 'ap-south-1'
);

-- Partitioned by tenant for query performance at scale
-- PostgreSQL 16 declarative partitioning by tenant_id (hash, 16 partitions)
-- CREATE TABLE consent_records PARTITION BY HASH (tenant_id);

-- Indexes for common query patterns
CREATE INDEX idx_consent_records_dp_hash ON consent_records (tenant_id, dp_identifier_hash, purpose_id, created_at DESC);
CREATE INDEX idx_consent_records_purpose ON consent_records (tenant_id, purpose_id, action, created_at DESC);
CREATE INDEX idx_consent_records_created ON consent_records (tenant_id, created_at DESC);

-- CRITICAL: Append-only enforcement via DB trigger
-- No UPDATE or DELETE permitted — violation is a Rule 4 breach
CREATE OR REPLACE FUNCTION enforce_consent_immutability()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'UPDATE' THEN
        RAISE EXCEPTION 'consent_records is append-only (DPDPA Rule 4). UPDATE not permitted. Record ID: %', OLD.id;
    END IF;
    IF TG_OP = 'DELETE' THEN
        RAISE EXCEPTION 'consent_records is append-only (DPDPA Rule 4). DELETE not permitted. Record ID: %', OLD.id;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_consent_immutability
BEFORE UPDATE OR DELETE ON consent_records
FOR EACH ROW EXECUTE FUNCTION enforce_consent_immutability();
```

### 1.5 Data Principal Identifiers (PII Table)

```sql
-- Separate table with much stricter access controls.
-- KMS-encrypted raw identifiers, referenced by hash from consent_records.
-- Only accessible to: erasure workflow, parental consent verification, consumer dashboard.

CREATE TABLE dp_identifiers (
    id                   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id            UUID NOT NULL REFERENCES tenants(id),
    
    -- The hash must match dp_identifier_hash in consent_records
    identifier_hash      TEXT NOT NULL,              -- HMAC-SHA256 — must match consent_records
    
    -- Encrypted raw identifier (e.g., phone number, email, user ID)
    identifier_encrypted TEXT NOT NULL,              -- AES-256 via tenant's KMS key
    identifier_type      TEXT NOT NULL CHECK (identifier_type IN (
                            'phone', 'email', 'user_id', 'aadhaar_virtual_id'
                         )),
    
    -- Registered language preference (for notifications in DP's language)
    preferred_language   TEXT NOT NULL DEFAULT 'hi', -- BCP 47
    
    -- Age classification for Section 9
    age_verified         BOOLEAN NOT NULL DEFAULT FALSE,
    is_minor             BOOLEAN,                    -- NULL = not verified
    guardian_identifier_hash TEXT,                   -- If minor, links to guardian
    
    -- Consent Manager registration (Section 2(g)) — DP can have account
    has_consumer_account BOOLEAN NOT NULL DEFAULT FALSE,
    consumer_account_id  UUID,
    
    created_at           TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    
    UNIQUE (tenant_id, identifier_hash)
);

-- Row-level security: tenant can only see their own DPs
ALTER TABLE dp_identifiers ENABLE ROW LEVEL SECURITY;
CREATE POLICY dp_identifiers_tenant_isolation ON dp_identifiers
    USING (tenant_id = current_setting('app.tenant_id')::uuid);
```

### 1.6 Erasure Records

```sql
-- DPDPA context: Third Schedule class-specific erasure timelines.
-- 48hr pre-notification required. Hard deletion only. Certificate generated.
-- Temporal.io workflows read/update this table.

CREATE TABLE erasure_records (
    id                   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id            UUID NOT NULL REFERENCES tenants(id),
    
    -- DP being erased
    dp_identifier_hash   TEXT NOT NULL,
    purpose_id           UUID REFERENCES consent_purposes(id), -- NULL = all purposes
    
    -- Trigger
    triggered_by         TEXT NOT NULL CHECK (triggered_by IN (
                            'CONSENT_WITHDRAWN',    -- Automatic on withdrawal
                            'PURPOSE_FULFILLED',    -- DF signals purpose complete
                            'DP_REQUEST',           -- Direct DP erasure request
                            'CLASS_TIMELINE'        -- Third Schedule 3yr expiry
                         )),
    
    -- Third Schedule class applied
    df_class_at_trigger  TEXT NOT NULL,             -- Snapshot of tenant.df_class at trigger time
    
    -- Timeline
    triggered_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    notification_due_at  TIMESTAMPTZ NOT NULL,       -- 48hr before execution
    notification_sent_at TIMESTAMPTZ,
    execute_at           TIMESTAMPTZ NOT NULL,       -- Scheduled deletion time
    executed_at          TIMESTAMPTZ,
    
    -- Status
    status               TEXT NOT NULL DEFAULT 'SCHEDULED' CHECK (status IN (
                            'SCHEDULED', 'NOTIFIED', 'IN_PROGRESS', 'COMPLETED', 'FAILED', 'CANCELLED'
                         )),
    
    -- Language for notifications
    notification_language TEXT NOT NULL DEFAULT 'hi', -- DP's registered language
    
    -- Temporal.io workflow tracking
    temporal_workflow_id TEXT UNIQUE,
    temporal_run_id      TEXT,
    
    -- Deletion evidence
    deletion_certificate JSONB,                      -- Generated after hard deletion
    deletion_signature   TEXT,                       -- ECDSA P-256 signed certificate
    
    -- Where data was deleted from (propagation log)
    deletion_targets     JSONB,                      -- [{system: 'crm', status: 'deleted'}, ...]
    
    created_at           TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at           TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_erasure_execute_at ON erasure_records (tenant_id, execute_at) WHERE status = 'SCHEDULED';
CREATE INDEX idx_erasure_dp_hash ON erasure_records (tenant_id, dp_identifier_hash);
```

### 1.7 Breach Events

```sql
-- DPDPA context: Rule 7 — ALL breaches must be notified (no materiality threshold).
-- CERT-In: 6hr initial. DPBI: 72hr detailed. APs: immediate in their language.

CREATE TABLE breach_events (
    id                   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id            UUID NOT NULL REFERENCES tenants(id),
    
    -- Breach details
    detected_at          TIMESTAMPTZ NOT NULL,
    detected_by          TEXT NOT NULL CHECK (detected_by IN (
                            'SYSTEM', 'TENANT', 'THIRD_PARTY', 'DPBI'
                         )),
    breach_type          TEXT NOT NULL CHECK (breach_type IN (
                            'UNAUTHORISED_ACCESS', 'DATA_LOSS', 'RANSOMWARE',
                            'ACCIDENTAL_DISCLOSURE', 'INSIDER_THREAT', 'OTHER'
                         )),
    breach_description   TEXT NOT NULL,              -- Internal description
    affected_data_categories TEXT[] NOT NULL,
    estimated_affected_count INTEGER,
    
    -- Notification deadlines (Rule 7)
    cert_in_deadline     TIMESTAMPTZ NOT NULL,       -- detected_at + 6hr
    dpbi_deadline        TIMESTAMPTZ NOT NULL,        -- detected_at + 72hr
    
    -- Notification status
    cert_in_submitted_at TIMESTAMPTZ,
    cert_in_report_id    TEXT,
    dpbi_submitted_at    TIMESTAMPTZ,
    dpbi_report_id       TEXT,
    dp_notification_sent_at TIMESTAMPTZ,
    dp_notification_count INTEGER,
    
    -- Temporal.io workflow
    temporal_workflow_id TEXT UNIQUE,
    
    -- Status
    status               TEXT NOT NULL DEFAULT 'DETECTED' CHECK (status IN (
                            'DETECTED', 'NOTIFYING', 'CERT_IN_SUBMITTED', 'DPBI_SUBMITTED', 'CLOSED'
                         )),
    
    created_at           TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    
    -- Append-only: breaches cannot be modified or deleted
    CONSTRAINT breach_events_no_soft_delete CHECK (TRUE) -- enforced by trigger (same as consent_records)
);

CREATE INDEX idx_breach_events_tenant ON breach_events (tenant_id, detected_at DESC);
```

### 1.8 Audit Log

```sql
-- Comprehensive audit trail for DPBI compliance.
-- Every consent-relevant action logged here, regardless of which table it touches.

CREATE TABLE audit_log (
    id                   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id            UUID REFERENCES tenants(id),
    
    -- Actor
    actor_type           TEXT NOT NULL CHECK (actor_type IN (
                            'SYSTEM', 'TENANT_API', 'CONSUMER', 'ADMIN', 'TEMPORAL_WORKFLOW'
                         )),
    actor_id             TEXT,                       -- API key hash, user ID, workflow ID
    
    -- Action
    event_type           TEXT NOT NULL,              -- 'consent.granted', 'erasure.executed', etc.
    resource_type        TEXT NOT NULL,              -- 'consent_record', 'erasure', 'breach', etc.
    resource_id          UUID,
    
    -- Context (PII-safe)
    dp_identifier_hash   TEXT,
    purpose_id           UUID,
    
    -- Payload (sanitised — no raw PII)
    event_data           JSONB NOT NULL DEFAULT '{}',
    
    -- Immutability
    created_at           TIMESTAMPTZ NOT NULL DEFAULT NOW()
    
    -- No indexes on dp_identifier_hash in audit_log — range scans by tenant + time only
);

CREATE INDEX idx_audit_log_tenant_time ON audit_log (tenant_id, created_at DESC);
CREATE INDEX idx_audit_log_resource ON audit_log (tenant_id, resource_type, resource_id);
```

### 1.9 Consumer Accounts (Data Principal Dashboard)

```sql
-- Data Principals can create an account to manage all their consents across tenants.
-- Section 12 DPDPA: right to access, correct, and erase personal data.

CREATE TABLE consumer_accounts (
    id                   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- Identifier (verified phone or email — stored encrypted)
    identifier_encrypted TEXT NOT NULL,              -- AES-256 via TrustStack master KMS key
    identifier_hash      TEXT NOT NULL UNIQUE,       -- SHA-256 for lookup
    identifier_type      TEXT NOT NULL CHECK (identifier_type IN ('phone', 'email')),
    
    -- Preferred language for consumer dashboard
    preferred_language   TEXT NOT NULL DEFAULT 'hi', -- BCP 47
    
    -- Consent for TrustStack itself to hold this account
    truststack_consent   BOOLEAN NOT NULL DEFAULT FALSE,
    truststack_consent_at TIMESTAMPTZ,
    
    -- Session
    last_active_at       TIMESTAMPTZ,
    
    created_at           TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## 2. Entity Relationships

```
tenants (1)
    │
    ├──────── consent_purposes (N) [one tenant has many purposes]
    │              │
    │              └──── consent_purpose_translations (22) [one per language]
    │
    ├──────── consent_records (N) [append-only, partitioned by tenant_id]
    │              │
    │              ├─── tenant_id → tenants.id
    │              ├─── purpose_id → consent_purposes.id
    │              └─── dp_identifier_hash ──► dp_identifiers.identifier_hash
    │
    ├──────── dp_identifiers (N) [PII table, strict access]
    │              │
    │              └─── guardian_identifier_hash ──► dp_identifiers.identifier_hash (self-ref)
    │
    ├──────── erasure_records (N)
    │              │
    │              └─── dp_identifier_hash ──► dp_identifiers.identifier_hash
    │
    ├──────── breach_events (N)
    │
    └──────── audit_log (N)

consumer_accounts (1)
    │
    └──────── (cross-tenant view of consent_records via identifier_hash lookup)
```

---

## 3. Row-Level Security Policies

```sql
-- All data-bearing tables have tenant isolation enforced at the DB level.
-- Application layer sets app.tenant_id per request.

-- Enable RLS
ALTER TABLE consent_records ENABLE ROW LEVEL SECURITY;
ALTER TABLE consent_purposes ENABLE ROW LEVEL SECURITY;
ALTER TABLE erasure_records ENABLE ROW LEVEL SECURITY;
ALTER TABLE dp_identifiers ENABLE ROW LEVEL SECURITY;
ALTER TABLE audit_log ENABLE ROW LEVEL SECURITY;

-- Tenant isolation policies
CREATE POLICY tenant_isolation ON consent_records
    USING (tenant_id = current_setting('app.tenant_id', TRUE)::uuid);

CREATE POLICY tenant_isolation ON consent_purposes
    USING (tenant_id = current_setting('app.tenant_id', TRUE)::uuid);

CREATE POLICY tenant_isolation ON erasure_records
    USING (tenant_id = current_setting('app.tenant_id', TRUE)::uuid);

CREATE POLICY tenant_isolation ON dp_identifiers
    USING (tenant_id = current_setting('app.tenant_id', TRUE)::uuid);

CREATE POLICY tenant_isolation ON audit_log
    USING (tenant_id = current_setting('app.tenant_id', TRUE)::uuid
           OR current_setting('app.tenant_id', TRUE) IS NULL); -- Admin bypass

-- Breach table: tenant can see own + TrustStack admin can see all
CREATE POLICY tenant_isolation ON breach_events
    USING (tenant_id = current_setting('app.tenant_id', TRUE)::uuid);
```

---

## 4. Database Migration Strategy

All migrations must be:
1. **Reversible** — every migration has an `up` and `down`
2. **Non-destructive** — never DROP TABLE or DROP COLUMN without a deprecation cycle
3. **Zero-downtime** — new columns added with DEFAULT first, then populated, then NOT NULL applied

```bash
# Migration naming convention
migrations/
  0001_create_tenants.sql
  0002_create_consent_purposes.sql
  0003_create_consent_records.sql
  0003_create_consent_records.down.sql  # Rollback
  0004_create_dp_identifiers.sql
  ...
```

---

## 5. Data Retention Policy

| Table | Retention | Mechanism |
|-------|-----------|-----------|
| `consent_records` | 7 years (Rule 4 minimum) | No automatic deletion; archived after 7yr to cold storage |
| `consent_purpose_translations` | Indefinite | Historical versions kept |
| `erasure_records` | 7 years | Deletion certificates retained even after data erased |
| `breach_events` | 7 years | Required for DPBI |
| `audit_log` | 7 years | Required for DPBI |
| `dp_identifiers` | Until erasure requested + 7yr processing logs | Hard-deleted on erasure |
| `consumer_accounts` | Until account deleted | Separate GDPR-style right to erasure |

---

## 6. Backup and Point-in-Time Recovery

```sql
-- PostgreSQL WAL archiving configuration (AWS RDS)
-- Backup retention: 35 days (maximum RDS allows)
-- Point-in-time recovery: any second within 35 days
-- Automated snapshots: daily
-- Cross-AZ replication: synchronous (Multi-AZ RDS)
-- DR (GCP Mumbai): asynchronous logical replication for consent_records, audit_log
```

**Key constraint:** Backup restoration must preserve the immutability triggers. After restoration, immediately verify trigger presence:

```sql
SELECT trigger_name, event_manipulation, event_object_table
FROM information_schema.triggers
WHERE trigger_name = 'trg_consent_immutability';
-- Must return rows for consent_records
```

---

## 7. DPDPA Compliance Annotations

| Table | Section | Requirement Met |
|-------|---------|-----------------|
| `consent_records` | Rule 4 | Append-only, 7-year retention |
| `consent_records` | Section 5/Rule 3 | `language_code`, `notice_version` stored per record |
| `consent_records` | Section 6 | `action IN ('GRANTED','WITHDRAWN')` enforced |
| `consent_records` | Section 9 | `is_minor`, `parental_consent_id` fields |
| `consent_records` | Section 16 | `data_residency_region` stored on every record |
| `dp_identifiers` | Section 12 | Enables access/correction/erasure per DP rights |
| `erasure_records` | Third Schedule | `df_class_at_trigger` records class at time of scheduling |
| `breach_events` | Rule 7 | Timestamps for all notifications; 6hr/72hr deadlines tracked |
| `audit_log` | Rule 4 | Complete immutable event trail for DPBI |
