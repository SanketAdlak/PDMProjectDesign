# TrustStack Discovery — Assessment & Data Discovery

> India's first AI-powered PII discovery and vendor contract audit platform for DPDPA 2023 compliance.  
> **Module 2 of TrustStack.** The prerequisite for everything that follows.

---

## What This Is

TrustStack Discovery is **Module 2** of TrustStack — the assessment and discovery layer that tells Indian businesses where their users' personal data actually lives, who their vendors are sharing it with, and whether those vendor contracts are legally sound under DPDPA 2023.

You cannot run erasure without first knowing where data lives. You cannot enforce a breach notification SLA on a vendor whose contract has no such clause. TrustStack Discovery closes both gaps — before enforcement hits in May 2027.

---

## Three Questions It Answers

| # | Question | Sub-product |
|---|----------|-------------|
| 1 | **Where does our users' personal data live — across every vendor, database, S3 bucket, and log system we use?** | AI Data Discovery Engine |
| 2 | **Are our vendor contracts and DPAs compliant with DPDPA 2023 — in Hindi, Tamil, Marathi, or whichever language they were written in?** | NLP Contract Audit Agent |
| 3 | **When we need to erase a user's data, exactly which systems do we touch and in what sequence?** | Data Flow Map → Erasure Snapshot API |

---

## The Two Sub-Products

### Sub-product A: AI Data Discovery Engine

An autonomous, read-only crawler that connects to your vendor systems — Zoho CRM, Razorpay, Tally, WhatsApp Business, AWS S3, GCP Storage, PostgreSQL, MySQL, MongoDB, CloudWatch, ELK — and surfaces every location where personal data lives, including in places your engineering team doesn't know about: application logs, backup buckets, analytics data lakes, and shadow databases.

**What it produces:** A **Data Flow Map** — a graph showing every data category (name, phone, PAN, health, financial) flowing between every system, tagged with the DPDPA purpose it was collected for, the consent status for that purpose, and the applicable retention class under the Third Schedule.

**How it runs:** Temporal.io orchestrates multi-step, resumable scan jobs. Each connector runs in a sandboxed Lambda/ECS task with read-only credentials scoped per vendor. No data is exfiltrated — findings are metadata only (system location, data category, record count, sample field names). PII detection uses a fine-tuned Mistral 7B model on SageMaker, entirely within Indian infrastructure.

### Sub-product B: NLP Contract Audit Agent

A multilingual NLP pipeline that ingests vendor contracts and Data Processing Agreements (DPAs) — including scanned documents in any of 22 Eighth Schedule languages — and flags every clause that violates DPDPA 2023. It then auto-generates a compliant replacement DPA in the vendor's language.

**What it detects:** Unlimited affiliate data sharing, missing breach notification SLAs, cross-border transfer without DPBI pre-authorization, open-ended retention language, absence of data minimisation clauses, profiling of minors without parental consent provisions.

**Language support:** OCR (Tesseract) for scanned Indic documents + fine-tuned LLM for clause extraction and compliance classification in 22 languages. DPA generation outputs in the detected or specified language.

---

## Who Uses Module 2

| Role | Primary Need from TrustStack Discovery |
|------|----------------------------------------|
| **CTO / VP Engineering** | Know exactly which 12 vendor integrations hold user PII before running erasure; get a machine-readable Data Flow Map to feed into automation |
| **Data Protection Officer (DPO)** | Audit all vendor contracts without a lawyer on retainer; generate compliant DPAs in the vendor's language in under 60 seconds |
| **Legal Counsel** | Review contract audit findings by DPDPA section reference; sign off on auto-generated DPAs; maintain evidence of due diligence |
| **CFO / Finance** | Quantify penalty exposure from undiscovered PII (up to ₹250Cr per violation category); use the Data Flow Map to prioritise remediation by financial risk |

---

## Why Discovery Is the Prerequisite for Erasure

Under DPDPA Section 12, Data Principals have the right to erasure. The Erasure Engine (Module 3) is responsible for executing that erasure within class-specific timelines (Third Schedule). But the Erasure Engine cannot know *where* to erase without a prior inventory of all systems containing that Data Principal's data.

**The dependency chain:**

```
[Module 2 — Discovery]
        |
        | produces
        v
[Data Flow Map Snapshot]
        |
        | consumed via one-time signed token (15-min TTL)
        v
[Module 3 — Erasure Engine]
        |
        | executes
        v
[Hard deletion across all discovered systems]
        |
        | produces
        v
[Deletion Certificate (ECDSA signed, DPBI-submittable)]
```

The Discovery Engine runs its full scan **once** (or on schedule). When the Erasure Engine needs to act on a specific Data Principal, it calls the **Snapshot API** with a one-time 15-minute signed token. The snapshot returns the list of systems containing that DP's data, the data categories, and the connection metadata needed for deletion. The Discovery Engine is not called again during erasure execution — the snapshot is authoritative.

**This design enforces the dual-role prohibition** (Section 2(g)): the Discovery account has no write access to any system it scans, and no path to the Consent Vault data (separate AWS account, no VPC peering).

---

## How Discovery Feeds Module 3

| Module 3 Component | What It Receives from Module 2 | When |
|--------------------|--------------------------------|------|
| Erasure Engine | Data Flow Map snapshot: system list, data categories, record identifiers per DP | At erasure trigger, via one-time signed Snapshot API token |
| Regulatory War Room | Vendor contract compliance status, flagged clauses, DPA generation status | On-demand, via Contract Audit API |
| Annual Audit Generator | Discovery scan history, PII inventory at point-in-time, retention class tags | At audit generation time |
| Consent Vault (indirect) | Purpose-to-system mappings inform consent purpose schema design | During tenant onboarding |

---

## Supported Vendor Connectors

| Connector | Data Categories Found | Common Hidden PII Locations | Known Retention Conflicts |
|-----------|----------------------|----------------------------|--------------------------|
| **Zoho CRM / Books / Desk** | Name, email, phone, deal value, support tickets | Custom fields, archived contacts, deleted-but-retained records | Zoho Books retains invoices 6yr (GST) — may conflict with DPDPA erasure for non-threshold SMEs |
| **Razorpay** | Payment details, KYC (PAN, Aadhaar), bank account | Webhook logs, settlement reports, test-mode data in production | RBI requires 5yr payment record retention — document conflict in Data Flow Map |
| **Tally** | Name, PAN, GSTIN, salary, financial transactions | Journal entries, ledger notes, TDS records | GST Act requires 6yr retention — must be flagged, not erased |
| **WhatsApp Business API** | Phone, chat history, consent confirmation messages | Meta-hosted message logs (180-day default), opt-in records | Meta's 180-day log retention; consent confirmations should be mirrored to Consent Vault |
| **AWS S3** | Any data category (unstructured) | Backups, Athena query result dumps, CloudTrail logs with PII, data lake partitions | None statutory — retention per business decision |
| **GCP Cloud Storage** | Any data category (unstructured) | Exported BigQuery tables, Dataflow intermediary outputs, GCS lifecycle-expired but not deleted | None statutory |
| **PostgreSQL / MySQL / MongoDB** | Application-defined | Shadow tables, audit log tables with raw PII, soft-deleted records not hard-deleted | Soft-delete records are non-compliant under DPDPA — flagged as critical finding |
| **CloudWatch / ELK / OpenSearch** | Structured log fields (email, IP, user ID) | Error logs with stack traces containing PII, request logs with query parameters | 1yr minimum log retention required for security (DPDPA Rule 7 audit trail) |
| **Datadog** | APM trace data, log fields | Distributed traces with PII in span attributes, log pipelines that include request bodies | Datadog default 15-day retention; may be deleting security-critical logs too soon |

---

## Critical Isolation: Separate AWS Account

**TrustStack Discovery runs in a dedicated AWS account: `TrustStack-Discovery` (ap-south-1)**

This is not a configuration choice — it is a regulatory requirement.

Under DPDPA Section 2(g), TrustStack operates as a Registered Consent Manager. A Consent Manager **cannot** also act as a Data Processor for the same Data Principal. If the Discovery Engine (which reads vendor data containing user PII) shared infrastructure with the Consent Vault (which holds consent records), TrustStack would be in a dual-role violation.

**Enforcement of isolation:**

| Control | Implementation |
|---------|----------------|
| Separate AWS account | `TrustStack-Discovery` account — no shared IAM, no shared VPCs |
| No VPC peering with Consent Vault | Consent Vault runs in `TrustStack-ConsentVault` account; no peering, no transit gateway between accounts |
| Read-only connector credentials | All vendor API keys/connection strings scoped read-only; stored in `TrustStack-Discovery` Secrets Manager only |
| Metadata-only findings | Discovery stores field names, record counts, data categories — NOT raw PII values |
| Snapshot API boundary | The only cross-module communication path is the Snapshot API, which issues one-time 15-minute signed tokens and returns metadata, never raw data |
| Audit trail | All cross-account API calls logged in CloudTrail in both accounts |

---

## Regulatory Context: What DPDPA Obligations Module 2 Helps Fulfil

| DPDPA Obligation | Section / Rule | How TrustStack Discovery Helps |
|------------------|----------------|-------------------------------|
| Data minimisation | Section 6 | Data Flow Map reveals PII collected beyond the stated purpose — findings include over-collection alerts |
| Purpose limitation | Section 5, Rule 3 | Maps each data category to its declared purpose; flags data used for undisclosed purposes |
| Erasure on purpose-fulfilment | Section 12, Third Schedule | Provides the system inventory the Erasure Engine needs to execute complete erasure |
| Vendor / processor accountability | Section 8(2) | Contract Audit Agent verifies vendor DPAs include mandatory DPDPA obligations |
| Breach notification readiness | Rule 7 | Data Flow Map shows which vendor systems a breach would affect; used in breach impact assessment |
| Cross-border transfer restriction | Section 16 | Flags any connector endpoint resolving outside Indian IP ranges; alerts on data sent to non-DPBI-whitelisted jurisdictions |
| Data Principal rights fulfilment | Section 12–13 | Complete system inventory is prerequisite for erasure, correction, and portability requests |
| Consent basis verification | Section 7 | Identifies data flows lacking a mapped consent purpose — highlights unlawful processing |

---

## Directory Structure

```
data-discovery/
├── README.md                          # This file
├── docs/
│   ├── PRD.md                         # Product Requirements Document
│   ├── SYSTEM_DESIGN.md               # Architecture: Temporal, SageMaker, connectors
│   ├── DATA_MODEL.md                  # PostgreSQL schemas for findings, scans, audits
│   ├── API_SPEC.md                    # OpenAPI 3.1: Discovery API + Snapshot API
│   ├── CONNECTOR_GUIDE.md             # Per-vendor connector auth, scopes, limitations
│   ├── NLP_CONTRACT_AUDIT.md          # NLP pipeline: language detection, clause taxonomy, DPA gen
│   └── RETENTION_CONFLICT_MATRIX.md   # Statutory retention vs DPDPA erasure — per data class
├── diagrams/
│   ├── system-architecture.md         # Discovery account isolation diagram (Mermaid)
│   ├── scan-workflow.md               # Temporal.io scan orchestration flow (Mermaid)
│   ├── data-flow-map.md               # Data Flow Map graph schema (Mermaid)
│   ├── snapshot-api-sequence.md       # Snapshot API → Erasure Engine sequence (Mermaid)
│   ├── contract-audit-pipeline.md     # NLP pipeline: OCR → extraction → classification (Mermaid)
│   └── connector-auth-flow.md         # OAuth / API key connector auth flows (Mermaid)
└── src/                               # Source code (to be added)
```

---

## Compliance Status

- [x] Dual-role prohibition architecture validated (separate AWS account)
- [x] Section 16 data residency: all compute in ap-south-1
- [x] PII detection model on-premise (SageMaker, no external API)
- [ ] Connector certification (Zoho, Razorpay, Tally) — target Q3 2026
- [ ] Snapshot API security audit — target Q4 2026
- [ ] DPBI sandbox testing for Discovery-Erasure flow — target Q1 2027

---

## Reading Order

For **Product teams**: Start with `docs/PRD.md`  
For **Engineering**: `docs/SYSTEM_DESIGN.md` → `docs/DATA_MODEL.md` → `docs/API_SPEC.md` → `docs/CONNECTOR_GUIDE.md`  
For **Compliance / Legal**: `docs/RETENTION_CONFLICT_MATRIX.md` → `docs/NLP_CONTRACT_AUDIT.md`  
For **Security**: `diagrams/system-architecture.md` (isolation model) → `docs/API_SPEC.md` (Snapshot API signing)

---

*TrustStack — Build Trust. Ship Faster.*  
*TrustStack Discovery — Know your data before the regulator does.*
