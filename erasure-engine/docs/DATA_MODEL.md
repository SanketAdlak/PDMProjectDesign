# Data Model — Automatic Erasure Engine

**Version:** 1.0
**Date:** 2026-04-22
**Database:** PostgreSQL 16
**Module:** 3.2 — Erasure Engine (TrustStack)

---

## Overview

The Erasure Engine database stores the **results and artefacts** of erasure workflows. Execution state lives in Temporal.io — PostgreSQL is the system of record for plans, certificates, approvals, and the immutable audit log.

### Core Design Decisions

**No raw PII stored.** Every reference to a Data Principal uses `dp_identifier_hash` (HMAC-SHA-256 of the DP's identifier keyed per tenant). The raw identifier never touches this database. This satisfies DPDPA's data minimisation principle (Section 6(2)) and eliminates the possibility of a breach exposing DP identity from Erasure Engine logs.

**Workflow state in Temporal, results in Postgres.** The Temporal workflow is the authoritative source of what step the operation is on. PostgreSQL holds the durable outputs of each step (plans, safety reviews, certificates) that must survive beyond workflow completion and be queryable for DPBI audits.

**Append-only audit log.** A DDL-level trigger prevents any UPDATE or DELETE on `erasure_audit_log`. DPDPA Rule 4 requires 7-year retention of processing records. The audit log is the forensic record that a deletion happened, who approved it, and what each agent decided. Mutability would destroy its legal value.

**ECDSA-signed certificates.** Deletion certificates are signed with the tenant's ECDSA P-256 key (AWS KMS). The signature covers a SHA-256 of the full certificate JSON so any tampering is detectable. DPBI can verify the signature independently.

**Row-level security.** Every table carrying tenant data has a row-level security (RLS) policy enforcing `tenant_id = current_setting('app.current_tenant_id')`. A compromised API key for Tenant A cannot read Tenant B's erasure records.

---

## Schema Definitions

### Enumerated Types

```sql
-- DPDPA Third Schedule erasure class.
-- Determines whether deletion is immediate (STANDARD) or deferred up to 3yr.
CREATE TYPE erasure_class_enum AS ENUM (
    'ECOMMERCE_GT_2CR',   -- E-commerce DFs with ≥2Cr users: 3yr from last transaction/login
    'GAMING_GT_50L',      -- Online gaming DFs with ≥50L users: 3yr from last login
    'SOCIAL_GT_2CR',      -- Social media DFs with ≥2Cr users: 3yr from last login
    'STANDARD'            -- All other DFs (most SMEs): erase on consent withdrawal or purpose fulfilment
);

-- What caused the erasure to be initiated.
CREATE TYPE trigger_type_enum AS ENUM (
    'CONSENT_WITHDRAWN',     -- Kafka event from Consent Vault (Section 7 DPDPA)
    'DP_REQUEST',            -- Data Principal exercised Section 12 right to erasure
    'PURPOSE_FULFILLED',     -- Consent purpose lifecycle ended; data no longer needed
    'SCHEDULED_RETENTION',   -- Timer fired for a deferred 3yr erasure (large DF class)
    'REGULATORY_AUDIT'       -- DPBI-directed audit erasure
);

-- Lifecycle status of an erasure operation, mirroring Temporal workflow states.
CREATE TYPE operation_status_enum AS ENUM (
    'PENDING_PLAN',       -- Planner Agent running
    'PENDING_SAFETY',     -- Safety Agent running
    'PENDING_APPROVAL',   -- Waiting at Human Approval Gate
    'APPROVED',           -- DPO approved; queued for execution (or in 48hr window)
    'EXECUTING',          -- Executor Agent running deletions
    'VERIFYING',          -- Verifier Agent running post-deletion checks
    'COMPLETED',          -- All systems deleted; certificate issued
    'FAILED',             -- One or more systems failed; partial certificate issued
    'STOPPED',            -- Emergency stop triggered; halted mid-execution
    'REGULATORY_HOLD'     -- Operation paused due to regulatory conflict (court order, DPBI hold)
);

-- Overall safety verdict from the Safety Agent.
CREATE TYPE safety_result_enum AS ENUM (
    'SAFE',                  -- No issues; proceed automatically
    'SAFE_WITH_WARNINGS',    -- Low-risk warnings noted; auto-proceed with warnings logged
    'REQUIRES_HUMAN',        -- Must go through Human Approval Gate before execution
    'BLOCKED'                -- Hard regulatory conflict; operation cannot proceed
);

-- Status of human approval gate.
CREATE TYPE approval_status_enum AS ENUM (
    'PENDING',    -- Awaiting DPO/CTO decision
    'APPROVED',   -- Approved; workflow continues
    'REJECTED',   -- Rejected; workflow cancelled
    'EXPIRED'     -- 72-hour approval window elapsed with no decision
);

-- Status of a single system's erasure within an operation.
CREATE TYPE system_erasure_status_enum AS ENUM (
    'PENDING',                   -- Not yet started
    'IN_PROGRESS',               -- Executor Agent actively deleting
    'COMPLETED',                 -- All targeted records deleted
    'FAILED',                    -- Deletion failed after retries
    'SKIPPED_REGULATORY_HOLD',   -- Skipped because regulatory conflict prevents deletion
    'VERIFIED'                   -- Verifier Agent confirmed deletion (post COMPLETED)
);

-- Connector/vendor type for system erasure records.
CREATE TYPE connector_type_enum AS ENUM (
    'ZOHO_CRM',
    'RAZORPAY',
    'TALLY',
    'WHATSAPP_BUSINESS',
    'AWS_S3',
    'TENANT_POSTGRES',
    'GOOGLE_WORKSPACE',
    'GENERIC_REST'
);

-- Regulatory hold reason: why a record cannot be deleted despite a valid erasure request.
CREATE TYPE regulatory_hold_enum AS ENUM (
    'RBI_5YR_PAYMENT',    -- RBI Payment System Regulations: payment records 5yr
    'GST_6YR_INVOICE',    -- GST Act Section 36: invoice/returns records 6yr
    'SEBI_5YR_TRADE',     -- SEBI regulations: trade records 5yr
    'CERT_IN_LOGS',       -- CERT-In 2022 rules: log retention 180 days to 1yr
    'COURT_ORDER'         -- Active court order prevents destruction
);

-- How a deletion certificate was delivered to the Data Principal.
CREATE TYPE notification_channel_enum AS ENUM (
    'EMAIL',
    'WHATSAPP',
    'IN_APP',
    'SMS',
    'API_CALLBACK'
);

-- Who (or what) performed an action in the audit log.
CREATE TYPE audit_actor_enum AS ENUM (
    'SYSTEM',
    'PLANNER_AGENT',
    'SAFETY_AGENT',
    'EXECUTOR_AGENT',
    'VERIFIER_AGENT',
    'CERTIFIER_AGENT',
    'HUMAN_APPROVER',
    'API_CLIENT',
    'SCHEDULER'
);

-- Plan generator: automated agent or manual DPO override.
CREATE TYPE plan_generator_enum AS ENUM (
    'PLANNER_AGENT',
    'MANUAL_OVERRIDE'
);
```

---

### Table 1: `tenants`

```sql
-- DPDPA context: Each tenant is a Data Fiduciary (DF) registered with TrustStack
-- as their Consent Manager (Section 2(g), DPDPA 2023). The erasure_class drives
-- all Third Schedule timeline calculations. This table is synced read-only from
-- the Consent Vault service; it is not mutated by the Erasure Engine.

CREATE TABLE tenants (
    tenant_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Business identity (mirrors consent-vault.tenants)
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,                  -- URL-safe identifier
    legal_name      TEXT NOT NULL,                         -- Registered company name (CIN)

    -- DPDPA Third Schedule class — determines erasure timeline for this DF.
    -- STANDARD is the default for most SMEs (immediate erasure on consent withdrawal).
    erasure_class   erasure_class_enum NOT NULL DEFAULT 'STANDARD',

    -- TrustStack subscription tier (affects API rate limits and feature access).
    plan_tier       TEXT NOT NULL DEFAULT 'growth' CHECK (plan_tier IN (
                        'awareness_pro', 'growth', 'enterprise', 'platform'
                    )),

    -- KMS key for signing deletion certificates (ECDSA P-256).
    -- One key per tenant so a compromised key cannot forge certs for other tenants.
    kms_signing_key_arn  TEXT NOT NULL,

    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_tenants_slug ON tenants(slug);

-- Row-Level Security: tenants table is readable by all authenticated service roles.
ALTER TABLE tenants ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON tenants
    USING (tenant_id::TEXT = current_setting('app.current_tenant_id', TRUE));
```

---

### Table 2: `erasure_operations`

```sql
-- DPDPA context: Central job record for one erasure lifecycle. Each row represents
-- a single Data Principal's erasure across all connected systems for one tenant.
-- Immutable identity fields (dp_identifier_hash, trigger_type) are set at creation
-- and never updated — status and FK references are the only mutable fields.
--
-- DPDPA Rule 4: operation records retained for 7 years from completion date
-- (processing log retention requirement for Registered Consent Managers).

CREATE TABLE erasure_operations (
    operation_id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    tenant_id                  UUID NOT NULL REFERENCES tenants(tenant_id),

    -- SHA-256 HMAC of the DP's raw identifier (phone/email/user_id), keyed per tenant.
    -- Raw PII is NEVER stored here. The hash is computed by the API layer before insert.
    -- DPDPA data minimisation: Section 6(2) — collect only what is necessary.
    dp_identifier_hash         TEXT NOT NULL,

    trigger_type               trigger_type_enum NOT NULL,

    status                     operation_status_enum NOT NULL DEFAULT 'PENDING_PLAN',

    -- Per-operation erasure class. Normally inherited from tenant; overridable for
    -- edge cases (e.g., tenant recently crossed 2Cr user threshold mid-operation).
    erasure_class              erasure_class_enum NOT NULL,

    -- For STANDARD class: NULL (execute immediately after approval + 48hr window).
    -- For deferred classes: the Temporal workflow sleeps until this timestamp.
    scheduled_at               TIMESTAMPTZ,

    -- Temporal.io workflow ID — the authoritative execution handle.
    -- Used to send signals (emergency stop, approval) to the live workflow.
    temporal_workflow_id       TEXT UNIQUE,

    -- Reference to the Module 2 snapshot JWT used to build the plan.
    -- 15-minute TTL token; this field stores its jti claim for audit purposes.
    -- Enables reconstruction of which discovery snapshot drove each erasure.
    discovery_snapshot_token_id TEXT,

    -- Populated sequentially as each pipeline stage completes.
    plan_id                    UUID REFERENCES erasure_plans(plan_id),
    safety_review_id           UUID REFERENCES safety_reviews(review_id),
    approval_id                UUID REFERENCES approval_requests(approval_id),
    certificate_id             UUID REFERENCES deletion_certificates(certificate_id),

    -- 48-hour pre-deletion notice sent at this time (DPDPA Third Schedule, Item 5).
    -- DP has the window between notification_sent_at and execution to opt out.
    notification_sent_at       TIMESTAMPTZ,

    started_at                 TIMESTAMPTZ,    -- When Executor Agent began
    completed_at               TIMESTAMPTZ,    -- When Certifier Agent finished

    created_at                 TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_erasure_ops_tenant    ON erasure_operations(tenant_id);
CREATE INDEX idx_erasure_ops_status    ON erasure_operations(status);
CREATE INDEX idx_erasure_ops_dp_hash   ON erasure_operations(dp_identifier_hash);
CREATE INDEX idx_erasure_ops_scheduled ON erasure_operations(scheduled_at)
    WHERE scheduled_at IS NOT NULL AND status NOT IN ('COMPLETED', 'FAILED', 'STOPPED');
CREATE INDEX idx_erasure_ops_trigger   ON erasure_operations(trigger_type);
CREATE INDEX idx_erasure_ops_created   ON erasure_operations(created_at DESC);

ALTER TABLE erasure_operations ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON erasure_operations
    USING (tenant_id::TEXT = current_setting('app.current_tenant_id', TRUE));
```

---

### Table 3: `erasure_plans`

```sql
-- DPDPA context: The Planner Agent's output — a structured, human-readable
-- description of which systems hold this DP's data and in what order they
-- should be deleted. The planner_reasoning field satisfies DPDPA's requirement
-- that automated decisions be explainable (relevant under anticipated AI Act
-- convergence). The plan is a proposal; execution only begins after Safety Agent
-- review and (where required) human approval.

CREATE TABLE erasure_plans (
    plan_id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operation_id      UUID NOT NULL REFERENCES erasure_operations(operation_id),
    tenant_id         UUID NOT NULL REFERENCES tenants(tenant_id),

    -- Natural-language explanation from the LLM of why each system is included,
    -- what data categories were found, and why the chosen deletion order is safe.
    -- Stored so DPO can read and verify the plan before approving.
    planner_reasoning TEXT NOT NULL,

    -- Array of target system descriptors. Each element:
    -- {
    --   "systemId": "zoho_crm",
    --   "systemName": "Zoho CRM",
    --   "connector": "ZOHO_CRM",
    --   "dataCategories": ["email", "phone", "purchase_history"],
    --   "executionOrder": 1,
    --   "estimatedRecords": 47,
    --   "riskLevel": "LOW",
    --   "riskReason": null,
    --   "requiresApproval": false,
    --   "deletionMethod": "AUTO_API"
    -- }
    systems_targeted  JSONB NOT NULL,

    -- Ordered array of systemId strings; Executor Agent iterates in this order.
    -- Order respects foreign key dependencies (delete children before parents).
    execution_order   TEXT[] NOT NULL,

    plan_version      INTEGER NOT NULL DEFAULT 1,   -- Incremented on MANUAL_OVERRIDE

    generated_by      plan_generator_enum NOT NULL DEFAULT 'PLANNER_AGENT',

    generated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- Set when DPO approves (or auto-approved after safety pass).
    approved_at       TIMESTAMPTZ
);

CREATE INDEX idx_plans_operation ON erasure_plans(operation_id);
CREATE INDEX idx_plans_tenant    ON erasure_plans(tenant_id);

ALTER TABLE erasure_plans ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON erasure_plans
    USING (tenant_id::TEXT = current_setting('app.current_tenant_id', TRUE));
```

---

### Table 4: `safety_reviews`

```sql
-- DPDPA context: The Safety Agent's independent review of the Planner's output.
-- Regulatory conflicts (RBI, GST, SEBI) are detected here and stored as structured
-- JSON so the Executor Agent can skip blocked systems and the Certificate can explain
-- what was retained and why. Hard rule violations block execution; soft warnings are
-- recorded for transparency but do not stop the workflow.

CREATE TABLE safety_reviews (
    review_id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    plan_id                 UUID NOT NULL REFERENCES erasure_plans(plan_id),
    operation_id            UUID NOT NULL REFERENCES erasure_operations(operation_id),
    tenant_id               UUID NOT NULL REFERENCES tenants(tenant_id),

    overall_result          safety_result_enum NOT NULL,

    -- Violations of hard safety rules that cannot be overridden by any human.
    -- Schema per element:
    -- {
    --   "ruleId": "PAYMENT_RECORDS",
    --   "ruleName": "RBI 5yr Payment Record Retention",
    --   "description": "Payment records for this DP in Razorpay cannot be hard-deleted...",
    --   "blocked_system": "razorpay",
    --   "regulation": "RBI Payment System Regulations 2018 §X"
    -- }
    hard_rule_violations    JSONB NOT NULL DEFAULT '[]',

    -- Warnings from soft rules (e.g., high volume, sensitive category).
    -- These trigger REQUIRES_HUMAN but do not block after human approval.
    -- Schema per element:
    -- {
    --   "ruleId": "HIGH_VOLUME",
    --   "ruleName": "High-Volume Deletion",
    --   "description": "Total records > 1000 across all systems.",
    --   "risk_level": "MEDIUM",
    --   "recommendation": "Review the plan carefully before approving."
    -- }
    soft_rule_warnings      JSONB NOT NULL DEFAULT '[]',

    -- Regulatory conflicts that prevent full deletion for specific data categories.
    -- The Executor will skip these systems; the Certificate will note the hold.
    -- Schema per element:
    -- {
    --   "system": "razorpay",
    --   "conflict": "RBI_5YR_PAYMENT",
    --   "affectedCategories": ["payment_records", "kyc_documents"],
    --   "regulation": "RBI Payment System Regulations 2018",
    --   "expectedReleaseDate": "2031-04-22",
    --   "action": "ANONYMISE_PII"
    -- }
    regulatory_conflicts    JSONB NOT NULL DEFAULT '[]',

    reviewed_at             TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_safety_plan      ON safety_reviews(plan_id);
CREATE INDEX idx_safety_operation ON safety_reviews(operation_id);
CREATE INDEX idx_safety_result    ON safety_reviews(overall_result);

ALTER TABLE safety_reviews ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON safety_reviews
    USING (tenant_id::TEXT = current_setting('app.current_tenant_id', TRUE));
```

---

### Table 5: `approval_requests`

```sql
-- DPDPA context: Human Approval Gate — required when Safety Agent determines that
-- the operation involves >1000 records, financial data, health/biometric data,
-- minor data, or a hard safety escalation. The 72-hour expiry window is intentional:
-- DPBI guidance requires that erasure requests not be indefinitely deferred by
-- inaction from the data fiduciary.

CREATE TABLE approval_requests (
    approval_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operation_id           UUID NOT NULL REFERENCES erasure_operations(operation_id),
    plan_id                UUID NOT NULL REFERENCES erasure_plans(plan_id),
    tenant_id              UUID NOT NULL REFERENCES tenants(tenant_id),

    -- One or more reasons why human approval is required.
    -- Values: 'RECORD_COUNT_EXCEEDS_1000' | 'FINANCIAL_DATA' | 'HEALTH_DATA'
    --       | 'BIOMETRIC_DATA' | 'MINOR_DATA' | 'SAFETY_AGENT_ESCALATION'
    required_because       TEXT[] NOT NULL,

    status                 approval_status_enum NOT NULL DEFAULT 'PENDING',

    -- UUID of the TrustStack user (DPO/CTO) who made the decision.
    -- NULL until a decision is made.
    approved_by            UUID,

    -- Mandatory free-text reason when status = 'REJECTED'.
    rejection_reason       TEXT,

    -- 72 hours from request creation. If no decision by this time, status → EXPIRED
    -- and the Temporal workflow cancels the operation.
    expires_at             TIMESTAMPTZ NOT NULL,

    -- Channels through which the approver was notified (all must succeed or be retried).
    -- Values: 'SLACK' | 'EMAIL' | 'WHATSAPP'
    notification_channels  TEXT[] NOT NULL DEFAULT '{}',

    requested_at           TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    decided_at             TIMESTAMPTZ
);

CREATE INDEX idx_approvals_operation ON approval_requests(operation_id);
CREATE INDEX idx_approvals_status    ON approval_requests(status);
CREATE INDEX idx_approvals_tenant    ON approval_requests(tenant_id);
CREATE INDEX idx_approvals_pending   ON approval_requests(tenant_id, expires_at)
    WHERE status = 'PENDING';

ALTER TABLE approval_requests ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON approval_requests
    USING (tenant_id::TEXT = current_setting('app.current_tenant_id', TRUE));
```

---

### Table 6: `system_erasure_records`

```sql
-- DPDPA context: One row per target system per erasure operation. Provides
-- per-system granularity for the deletion certificate and for the DPO dashboard.
-- Where regulatory_hold_reason is set, the system could not be fully deleted —
-- records_anonymized captures PII anonymisation (e.g., Razorpay customer PII
-- masked while transaction records are retained for RBI compliance).

CREATE TABLE system_erasure_records (
    record_id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operation_id            UUID NOT NULL REFERENCES erasure_operations(operation_id),
    tenant_id               UUID NOT NULL REFERENCES tenants(tenant_id),

    system_name             TEXT NOT NULL,              -- Human-readable (e.g., 'Zoho CRM')
    connector_type          connector_type_enum NOT NULL,

    status                  system_erasure_status_enum NOT NULL DEFAULT 'PENDING',

    -- Position in the Planner's execution_order array.
    execution_order         INTEGER NOT NULL,

    -- Counts as reported by the Module 2 Data Flow Map (estimated before execution).
    records_targeted        INTEGER NOT NULL DEFAULT 0,

    -- Post-execution counts (set by Executor Agent on completion).
    records_deleted         INTEGER NOT NULL DEFAULT 0,

    -- Records where PII fields were anonymised but the row was retained
    -- (e.g., Razorpay transaction rows: name/email/phone nulled, but txn_id kept for RBI).
    records_anonymized      INTEGER NOT NULL DEFAULT 0,

    -- Records that could not be deleted due to regulatory hold.
    records_retained        INTEGER NOT NULL DEFAULT 0,

    -- NULL when no regulatory hold. Otherwise indicates why records_retained > 0.
    regulatory_hold_reason  regulatory_hold_enum,

    -- Structured error information for FAILED status. MUST NOT include PII.
    -- Schema: { "errorCode": "CONNECTOR_TIMEOUT", "message": "...", "attemptCount": 3 }
    error_details           JSONB,

    -- Set by Verifier Agent after Executor marks COMPLETED.
    -- Schema: { "records_found_after_deletion": 0, "verified_at": "...", "method": "SELECT_COUNT" }
    verification_result     JSONB,

    started_at              TIMESTAMPTZ,
    completed_at            TIMESTAMPTZ
);

CREATE INDEX idx_sys_records_operation  ON system_erasure_records(operation_id);
CREATE INDEX idx_sys_records_tenant     ON system_erasure_records(tenant_id);
CREATE INDEX idx_sys_records_status     ON system_erasure_records(status);
CREATE INDEX idx_sys_records_connector  ON system_erasure_records(connector_type);

ALTER TABLE system_erasure_records ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON system_erasure_records
    USING (tenant_id::TEXT = current_setting('app.current_tenant_id', TRUE));
```

---

### Table 7: `deletion_certificates`

```sql
-- DPDPA context: The Certifier Agent's output — a cryptographically signed record
-- that specific data was deleted. This is the artifact the DP receives and the DPO
-- can submit to DPBI as evidence of compliance. Retained for 7 years per Rule 4.
--
-- The ECDSA P-256 signature covers certificate_hash (SHA-256 of the certificate JSON).
-- Any field change after signing invalidates the signature — tamper-evident by design.
-- Signing uses the tenant's own KMS key so TrustStack itself cannot forge certificates.

CREATE TABLE deletion_certificates (
    certificate_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operation_id            UUID NOT NULL REFERENCES erasure_operations(operation_id),
    tenant_id               UUID NOT NULL REFERENCES tenants(tenant_id),

    -- Pseudonymised reference to the Data Principal. Matches dp_identifier_hash
    -- across all tables. Never the raw identifier.
    dp_identifier_hash      TEXT NOT NULL,

    -- Per-system deletion summary included in the certificate.
    -- Schema per element:
    -- {
    --   "systemName": "Zoho CRM",
    --   "connectorType": "ZOHO_CRM",
    --   "status": "DELETED",
    --   "recordsDeleted": 47,
    --   "deletionTimestamp": "2026-04-22T14:32:10Z",
    --   "verificationStatus": "VERIFIED"
    -- }
    systems_deleted         JSONB NOT NULL,

    -- Systems excluded from deletion due to regulatory holds.
    -- Schema per element:
    -- {
    --   "systemName": "Razorpay",
    --   "holdReason": "RBI_5YR_PAYMENT",
    --   "regulation": "RBI Payment System Regulations 2018",
    --   "recordsRetained": 12,
    --   "recordsAnonymized": 3,
    --   "expectedReleaseDate": "2031-04-22"
    -- }
    systems_with_regulatory_hold JSONB NOT NULL DEFAULT '[]',

    -- SHA-256 of the canonical JSON serialisation of the full certificate payload.
    -- Signed by ecdsa_signature. Used to verify integrity without re-fetching the cert.
    certificate_hash        TEXT NOT NULL,

    -- ECDSA P-256 signature (base64url-encoded DER) over certificate_hash.
    ecdsa_signature         TEXT NOT NULL,

    -- AWS KMS key ARN used to produce the signature.
    signing_key_id          TEXT NOT NULL,

    signature_algorithm     TEXT NOT NULL DEFAULT 'ECDSA_SHA_256',

    issued_at               TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- When the DP was notified with the certificate.
    sent_to_dp_at           TIMESTAMPTZ,

    -- Which channel was used to deliver the certificate to the DP.
    sent_channel            notification_channel_enum
);

CREATE INDEX idx_certs_operation ON deletion_certificates(operation_id);
CREATE INDEX idx_certs_tenant    ON deletion_certificates(tenant_id);
CREATE INDEX idx_certs_dp_hash   ON deletion_certificates(dp_identifier_hash);
CREATE INDEX idx_certs_issued    ON deletion_certificates(issued_at DESC);

ALTER TABLE deletion_certificates ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON deletion_certificates
    USING (tenant_id::TEXT = current_setting('app.current_tenant_id', TRUE));
```

---

### Table 8: `pre_deletion_notifications`

```sql
-- DPDPA context: Third Schedule Item 5 requires that DFs notify Data Principals
-- 48 hours before deletion begins. This is a cancellation window — the DP can
-- opt out if they change their mind (e.g., they want to re-download their data
-- first). opt_out_received = true causes the Temporal workflow to cancel execution.
-- The notification is sent in the DP's preferred language (one of 22 Eighth
-- Schedule languages or English) as required by Section 5, Rule 3.

CREATE TABLE pre_deletion_notifications (
    notification_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operation_id             UUID NOT NULL REFERENCES erasure_operations(operation_id),
    tenant_id                UUID NOT NULL REFERENCES tenants(tenant_id),

    -- DP reference — pseudonymised.
    dp_identifier_hash       TEXT NOT NULL,

    -- BCP 47 language code of the notification content (e.g., 'hi', 'ta', 'en').
    -- Must match the DP's registered language preference.
    notification_language    TEXT NOT NULL DEFAULT 'hi',

    notification_channel     notification_channel_enum NOT NULL,

    status                   TEXT NOT NULL DEFAULT 'PENDING' CHECK (status IN (
                                 'PENDING', 'SENT', 'DELIVERED', 'FAILED'
                             )),

    -- 48 hours before the planned execution start time.
    scheduled_for            TIMESTAMPTZ NOT NULL,

    sent_at                  TIMESTAMPTZ,
    delivery_confirmed_at    TIMESTAMPTZ,

    -- True if DP replied with a cancellation before the execution window opened.
    opt_out_received         BOOLEAN NOT NULL DEFAULT FALSE,
    opt_out_at               TIMESTAMPTZ
);

CREATE INDEX idx_notif_operation  ON pre_deletion_notifications(operation_id);
CREATE INDEX idx_notif_tenant     ON pre_deletion_notifications(tenant_id);
CREATE INDEX idx_notif_scheduled  ON pre_deletion_notifications(scheduled_for)
    WHERE status = 'PENDING';

ALTER TABLE pre_deletion_notifications ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON pre_deletion_notifications
    USING (tenant_id::TEXT = current_setting('app.current_tenant_id', TRUE));
```

---

### Table 9: `erasure_audit_log`

```sql
-- DPDPA context: Append-only, immutable audit trail for every state transition,
-- agent action, and human decision in an erasure operation. DPDPA Rule 4 requires
-- Registered Consent Managers to retain processing logs for 7 years. DPBI auditors
-- can request the full log for any operation.
--
-- This table must NEVER be UPDATE-d or DELETE-d. The immutability trigger below
-- enforces this at the DDL level — application bugs and rogue queries cannot alter
-- historical records.

CREATE TABLE erasure_audit_log (
    log_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operation_id    UUID NOT NULL REFERENCES erasure_operations(operation_id),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),

    -- Covers all state transitions, agent outputs, human decisions, and system events.
    -- Examples: 'OPERATION_CREATED', 'PLAN_GENERATED', 'SAFETY_REVIEW_COMPLETED',
    --   'APPROVAL_REQUESTED', 'HUMAN_APPROVED', 'HUMAN_REJECTED', 'EXECUTION_STARTED',
    --   'SYSTEM_DELETED', 'SYSTEM_FAILED', 'SYSTEM_SKIPPED_REGULATORY_HOLD',
    --   'VERIFICATION_PASSED', 'VERIFICATION_FAILED', 'CERTIFICATE_ISSUED',
    --   'DP_NOTIFIED', 'EMERGENCY_STOP', 'OPERATION_COMPLETED', 'OPERATION_FAILED'
    event_type      TEXT NOT NULL,

    actor           audit_actor_enum NOT NULL,

    -- Service name (e.g., 'erasure-engine-api') or user UUID for HUMAN_APPROVER.
    actor_id        TEXT NOT NULL,

    -- Structured event payload. MUST NOT contain raw PII or raw identifiers.
    -- All DP references use dp_identifier_hash. System details may include
    -- counts, status codes, and non-sensitive metadata.
    details         JSONB NOT NULL DEFAULT '{}',

    -- Previous and new operation status for state-transition events.
    previous_status TEXT,
    new_status      TEXT,

    -- Monotonically increasing within an operation; enforced by application layer.
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Clustered index: most queries are for a specific operation's full history.
CREATE INDEX idx_audit_operation ON erasure_audit_log(operation_id, created_at ASC);
CREATE INDEX idx_audit_tenant    ON erasure_audit_log(tenant_id, created_at DESC);
CREATE INDEX idx_audit_actor     ON erasure_audit_log(actor);
CREATE INDEX idx_audit_event     ON erasure_audit_log(event_type);

ALTER TABLE erasure_audit_log ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON erasure_audit_log
    USING (tenant_id::TEXT = current_setting('app.current_tenant_id', TRUE));
```

---

## Immutability Trigger (Audit Log)

```sql
-- Blocks any UPDATE or DELETE on erasure_audit_log at the DDL level.
-- This cannot be disabled without superuser access — application code errors
-- cannot accidentally corrupt the audit record.

CREATE OR REPLACE FUNCTION prevent_audit_log_mutation()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    RAISE EXCEPTION
        'erasure_audit_log is append-only. '
        'UPDATE and DELETE operations are prohibited. '
        'DPDPA Rule 4 requires 7-year immutable processing log retention. '
        'Operation: %, Table: %', TG_OP, TG_TABLE_NAME;
END;
$$;

CREATE TRIGGER audit_log_immutable
    BEFORE UPDATE OR DELETE ON erasure_audit_log
    FOR EACH ROW
    EXECUTE FUNCTION prevent_audit_log_mutation();
```

---

## Forward Reference Resolution

Because `erasure_operations` references `erasure_plans`, `safety_reviews`, `approval_requests`, and `deletion_certificates` via FK, and those tables reference `erasure_operations`, the FKs on `erasure_operations` must be added after all tables are created:

```sql
-- Run after all tables exist.
ALTER TABLE erasure_operations
    ADD CONSTRAINT fk_op_plan
        FOREIGN KEY (plan_id) REFERENCES erasure_plans(plan_id) DEFERRABLE INITIALLY DEFERRED,
    ADD CONSTRAINT fk_op_safety
        FOREIGN KEY (safety_review_id) REFERENCES safety_reviews(review_id) DEFERRABLE INITIALLY DEFERRED,
    ADD CONSTRAINT fk_op_approval
        FOREIGN KEY (approval_id) REFERENCES approval_requests(approval_id) DEFERRABLE INITIALLY DEFERRED,
    ADD CONSTRAINT fk_op_certificate
        FOREIGN KEY (certificate_id) REFERENCES deletion_certificates(certificate_id) DEFERRABLE INITIALLY DEFERRED;
```

---

## Summary

| Table | Rows per erasure | Retention |
|---|---|---|
| `tenants` | 1 (shared) | Indefinite |
| `erasure_operations` | 1 | 7 years |
| `erasure_plans` | 1–2 (v1 + override) | 7 years |
| `safety_reviews` | 1 | 7 years |
| `approval_requests` | 0 or 1 | 7 years |
| `system_erasure_records` | 1 per target system | 7 years |
| `deletion_certificates` | 1 | 7 years |
| `pre_deletion_notifications` | 1 per 48hr notice | 7 years |
| `erasure_audit_log` | 10–50 (lifecycle events) | 7 years |
