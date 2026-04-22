# Product Requirements Document — TrustStack Discovery

**Version:** 1.0  
**Date:** 2026-04-22  
**Status:** Draft  
**Owner:** TrustStack Product Team  
**Module:** 2 — Assessment & Discovery

---

## 1. Executive Summary

TrustStack Discovery is the assessment and data intelligence layer of TrustStack. It answers the question that every Indian SME, SaaS company, and D2C brand must answer before May 2027: *where exactly does our users' personal data live, and are our vendor contracts lawful?*

It has two sub-products: the **AI Data Discovery Engine**, which crawls vendor integrations to surface hidden PII and produces a Data Flow Map; and the **NLP Contract Audit Agent**, which reads vendor contracts and DPAs in 22 Indian languages, flags non-compliant clauses under DPDPA 2023, and auto-generates compliant replacements.

Together they close the assessment gap that makes DPDPA compliance impossible in practice: businesses cannot grant erasure rights they cannot execute, and they cannot enforce vendor obligations they have not verified. TrustStack Discovery makes both executable.

**Target enforcement date:** May 2027. **Maximum penalty:** ₹250Cr per violation category. **Time to act:** now.

---

## 2. Problem Statement

### 2.1 The Hidden PII Problem

Indian businesses have been integrating third-party SaaS tools — Zoho, Razorpay, Tally, WhatsApp, various cloud databases — for years without systematic tracking of where user personal data flows. The result is PII scattered across:

- Application logs that were never meant to hold personal data but do (user IDs, email in error traces, phone numbers in query parameters)
- Backup buckets on S3 or GCP that hold full database dumps including deleted records
- Analytics data lakes with raw event streams containing PII
- Shadow tables in production databases — soft-deleted records that are non-compliant under DPDPA (Section 12 requires hard deletion)
- Test environments seeded with production data

When a Data Principal exercises their erasure right under Section 12, the business has 30 days to comply. Without a prior inventory of all systems holding that DP's data, compliance is guesswork. Guesswork results in incomplete erasure. Incomplete erasure is a violation — penalties up to ₹250Cr.

**The hidden PII problem is not an engineering failure. It is an information gap.** TrustStack Discovery closes it.

### 2.2 The Vendor Contract Blind Spot

Most Indian SMEs have signed vendor contracts — for CRM, payments, support, marketing, cloud infrastructure — without legal review for DPDPA compliance. The average SME has 8–15 active vendor integrations. Common violations found in standard vendor contracts:

| Clause Pattern | DPDPA Violation |
|---------------|----------------|
| "We may share data with our affiliates for any purpose" | Unlimited data sharing without specific consent — violates Sections 5 and 7 |
| No breach notification SLA specified | Vendor cannot commit to 72hr DPBI notification — violates Rule 7 |
| "Data transferred to servers globally for processing" | Cross-border transfer without DPBI pre-authorization — violates Section 16 |
| "Data retained as long as necessary for business purposes" | No specific retention period; violates purpose limitation under Section 5 |
| No data minimisation language | Implicit permission for over-collection — violates Section 6 |
| No parental consent provision for minor users | For any product with under-18 users — violates Section 9 |

These violations are **in signed contracts right now.** Enforcement begins May 2027. A DPO cannot review 15 vendor contracts in three languages without dedicated tools. A lawyer who can read Marathi vendor contracts costs ₹5,000+ per contract. TrustStack Discovery does it in under 60 seconds.

### 2.3 The Erasure Prerequisite Gap

The Erasure Engine (Module 3) requires a complete, machine-readable inventory of every system containing a specific Data Principal's data before it can execute erasure. Without this inventory, the Erasure Engine either:

- Misses systems, leaving the erasure incomplete (violation)
- Requires manual system enumeration for every erasure request (operationally impossible at scale)
- Delays erasure beyond the statutory period (violation)

TrustStack Discovery fills this gap by producing a Data Flow Map snapshot that the Erasure Engine consumes via a one-time, signed, 15-minute TTL token. The Discovery Engine runs its scan independently; the Erasure Engine calls the Snapshot API at erasure time. This decoupling is both an architectural choice and a regulatory requirement — the Discovery Engine must not be re-crawling live systems during time-sensitive erasure execution.

---

## 3. User Personas

### Persona 1: Kiran — CTO, SaaS Company (Pune)

**Background:** CTO at a 200-person B2B SaaS company. The product integrates Zoho CRM (customer data), Razorpay (payment and KYC), an internal PostgreSQL cluster (application data), CloudWatch (logs), and WhatsApp Business (support). Engineering team of 25. DPO is also the Head of Legal — one person wearing two hats.

**Language:** English  
**Primary channel:** Web dashboard, API

**Core problem:**
> "I know we have user data in Zoho, Razorpay, and our own DB. But I genuinely don't know if it's also in our CloudWatch logs or in the S3 backups from 2022. Before we can build erasure automation, I need a system inventory I can trust. I can't build on guesswork."

**What success looks like:** A Data Flow Map that shows every data category in every system, with record counts. An API he can call from the Erasure Engine to get the per-DP system list at erasure time. Confidence that the inventory is current — with scheduled re-scans.

**Willingness to pay:** ₹999/Purpose-Linked Data Flow discovered. Expects 30–50 flows across 12 integrations. Budget: ₹30,000–₹50,000 for the initial scan.

---

### Persona 2: Praveena — DPO, HealthTech Startup (Chennai)

**Background:** Data Protection Officer at a 60-person health technology company. The product connects to lab systems, pharmacy APIs, and uses Zoho Desk for patient support. Vendor contracts are a mix of English and Tamil. The company has 15 active vendor agreements, none reviewed for DPDPA compliance. Praveena speaks Tamil natively; her legal background is in Tamil Nadu law.

**Language:** Tamil and English  
**Primary channel:** Web dashboard

**Core problem:**
> "I have 15 vendor contracts. Some are in Tamil, some in English, one is in a mix. I need to know which ones are non-compliant and I need to generate replacement DPAs — but I can't afford a lawyer for every single contract. And the health data context makes this urgent. We handle medical records."

**What success looks like:** Upload 15 contracts (PDF, some scanned), receive a compliance report for each in under 5 minutes, with DPDPA section references. For the flagged ones, auto-generate a compliant DPA in the same language as the original contract. Audit trail she can present to the DPBI if audited.

**Willingness to pay:** ₹499/contract audit. 15 contracts = ₹7,485 initial spend. Ongoing: new contracts as signed.

---

### Persona 3: Meera — CFO, D2C Brand (Mumbai)

**Background:** CFO at a D2C lifestyle brand doing ₹40Cr ARR. 18 vendor integrations including Razorpay, Shiprocket, a custom WMS, and several ad-tech platforms. The brand collects purchase history, delivery addresses, phone numbers, and (unknowingly) has PII in its Meta Ads pixel event data.

**Language:** English  
**Primary channel:** Executive summary dashboard, PDF reports

**Core problem:**
> "I've read the headlines about ₹250Cr penalties. I need to know: what is our actual exposure? Not theoretically — what data do we have that we shouldn't, where is it, and what does a penalty look like for each category? I need a number I can take to the board."

**What success looks like:** A Data Flow Map with a financial risk overlay — penalty exposure per undiscovered PII category based on DPDPA penalty schedule. A PDF executive summary she can present to the board. A prioritised remediation plan sorted by financial risk, not technical complexity.

**Willingness to pay:** ₹14,999/mo (Enterprise Growth plan). Values business outcome (risk reduction) over feature count.

---

### Persona 4: Arjun — Legal Counsel, Manufacturing SME (Pune)

**Background:** In-house Legal Counsel at a manufacturing company with 500 employees. The company uses Tally for accounting (salary data, vendor PAN, GST data), a custom ERP, and has recently started using WhatsApp Business for vendor communication. Vendor contracts are primarily in Marathi. Arjun studied law in Pune; most of his contract work is in Marathi and English.

**Language:** Marathi and English  
**Primary channel:** Web dashboard, downloadable reports

**Core problem:**
> "Our standard vendor terms are in Marathi. When I upload them to any compliance tool, it either fails to parse the Marathi or gives me a generic English output. I need the audit to actually work in Marathi — flag the non-compliant clauses in Marathi, and generate the replacement DPA in Marathi so I can negotiate with vendors in their language."

**What success looks like:** Upload a Marathi-language vendor contract. Receive a compliance report in Marathi with flagged clauses highlighted and DPDPA section references in Marathi. Download a compliant replacement DPA in Marathi, ready to send to the vendor. No English-only fallback.

**Willingness to pay:** ₹499/contract. Expects to audit 20+ contracts in Year 1.

---

## 4. Product Modules

### Module A: AI Data Discovery Engine

Autonomous, read-only, multi-connector PII discovery system. Connects to vendor integrations and internal infrastructure via API keys, OAuth tokens, or database connection strings (read-only, scoped). Detects personal data using a fine-tuned Mistral 7B model running on AWS SageMaker (ap-south-1). Produces structured findings: system location, data category (per DPDPA classification), record count, sample field names, applicable consent purpose, Third Schedule retention class.

### Module B: NLP Contract Audit Agent

Multilingual NLP pipeline for vendor contract compliance analysis. Accepts PDF, DOCX, or image uploads. Tesseract OCR for scanned Indic documents. Language detection (22 Eighth Schedule languages + English). Fine-tuned LLM for clause extraction and DPDPA compliance classification. Non-compliant clause taxonomy mapped to DPDPA sections. Auto-generates compliant DPA in the detected or user-specified language.

### Module C: Data Flow Map

Graph-based visualisation of all discovered data flows. Nodes are systems (vendor SaaS, databases, log systems, storage buckets). Edges are data flows (category, volume, purpose, consent status). Risk overlay: penalty exposure per data category. Filterable by: vendor, data category, consent status, retention class, risk level. Exportable as JSON (for Erasure Engine), SVG (for audit reports), PDF (for board presentations).

### Module D: Snapshot API (Erasure Engine Integration)

Stateless API that issues one-time, signed, 15-minute TTL tokens for the Erasure Engine to consume the Data Flow Map for a specific Data Principal. Returns: list of systems containing the DP's data, data categories per system, connection metadata for deletion, record identifiers where available. Does not re-crawl live systems at request time — serves the most recent scan snapshot. Signed with ECDSA P-256; Erasure Engine verifies signature before acting.

---

## 5. User Stories

### Discovery Engine — Connector Management

| ID | Story | Acceptance Criteria |
|----|-------|---------------------|
| US-001 | As a CTO, I want to connect my Zoho CRM instance using OAuth so that TrustStack Discovery can scan it without me sharing admin credentials | OAuth flow completes, read-only scopes granted, connection status shown in dashboard |
| US-002 | As a CTO, I want to connect my PostgreSQL database using a read-only connection string so that Discovery can scan it without write access | Connection tested and validated as read-only before scan begins; write attempt returns error |
| US-003 | As a CTO, I want to see all connected vendor systems in a single dashboard with their last scan time and status | Dashboard shows: connector name, auth status, last successful scan timestamp, finding count, next scheduled scan |
| US-004 | As a CTO, I want to revoke a connector's access without deleting its historical scan findings so that I retain audit history | Connector marked inactive; credentials deleted from Secrets Manager; findings retained with "connector inactive" label |
| US-005 | As a DPO, I want to receive an alert when a connector's API key expires or OAuth token is revoked so that scans don't silently fail | Alert sent via email + webhook 7 days before expiry and immediately on revocation |

### Discovery Engine — Scan Triggering and Scheduling

| ID | Story | Acceptance Criteria |
|----|-------|---------------------|
| US-006 | As a CTO, I want to trigger an on-demand scan of a specific connector so that I can get updated findings after a system change | Scan initiates within 60 seconds; progress visible in dashboard; completion notification sent |
| US-007 | As a CTO, I want to schedule weekly automated scans for all connectors so that the Data Flow Map stays current without manual intervention | Cron-based scan schedule configurable per connector; last scan timestamp updated after each run |
| US-008 | As a DPO, I want scan jobs to be resumable if they fail mid-way so that I don't have to restart a 3-hour scan from scratch | Temporal.io workflow checkpoints at each connector step; resume from last checkpoint on failure |
| US-009 | As a CTO, I want to receive a scan completion report summarising new findings, resolved findings, and changes from the previous scan | Diff report: new PII locations found, locations that no longer contain the data, record count changes |

### Discovery Engine — PII Findings Review

| ID | Story | Acceptance Criteria |
|----|-------|---------------------|
| US-010 | As a DPO, I want to see all PII findings categorised by data class (financial, health, biometric, contact) so that I can prioritise remediation by sensitivity | Findings grouped by DPDPA data category with count, system, and field names |
| US-011 | As a CTO, I want to suppress a finding with a written justification so that known-legitimate PII locations don't clutter my active findings | Suppression recorded with: user, timestamp, justification text; suppressed findings visible in audit log but excluded from active dashboard |
| US-012 | As a DPO, I want to see which findings represent non-compliant soft-deleted records so that I can prioritise hard deletion | Findings tagged "SOFT_DELETE_NON_COMPLIANT" for records in soft-delete state per DPDPA Section 12 |
| US-013 | As a CFO, I want a financial risk score per finding that estimates penalty exposure based on the DPDPA penalty schedule so that I can prioritise by business impact | Risk score = data category severity × record count × penalty tier; displayed in INR |

### Data Flow Map

| ID | Story | Acceptance Criteria |
|----|-------|---------------------|
| US-014 | As a CTO, I want to view an interactive Data Flow Map showing all data flows between systems so that I can understand our data topology at a glance | Graph renders in under 3 seconds; nodes = systems, edges = data flows; hover shows data category and purpose |
| US-015 | As a DPO, I want to filter the Data Flow Map by data category so that I can see all flows containing health data | Filter applies to both nodes and edges; non-matching flows dimmed; count of matching flows shown |
| US-016 | As a CFO, I want to export the Data Flow Map as a PDF for board presentation so that I can communicate compliance status without technical jargon | PDF export includes executive summary, map image, top 5 risk findings by financial exposure |
| US-017 | As a CTO, I want to export the Data Flow Map as JSON so that I can use it in my own automation and erasure tooling | JSON schema documented in API_SPEC.md; includes all system metadata, data categories, and consent status |

### Contract Audit — Upload and Processing

| ID | Story | Acceptance Criteria |
|----|-------|---------------------|
| US-018 | As a DPO, I want to upload a vendor contract as a PDF (including scanned Indic language documents) so that the system can audit it for DPDPA compliance | Upload accepts PDF/DOCX/JPG/PNG up to 50MB; Tesseract OCR applied for scanned documents; processing begins within 5 seconds of upload |
| US-019 | As a Legal Counsel, I want the system to automatically detect the language of the uploaded contract so that I don't have to specify it manually | Language detection accuracy ≥95% for all 22 Eighth Schedule languages; detected language shown to user with confidence score |
| US-020 | As a Legal Counsel, I want to override the detected language if it is incorrect so that clause extraction uses the right model | Language override dropdown available before and after processing; re-processing triggered on override |

### Contract Audit — Clause Flagging

| ID | Story | Acceptance Criteria |
|----|-------|---------------------|
| US-021 | As a DPO, I want each flagged clause to include the exact contract text, the DPDPA section it violates, a plain-language explanation, and a suggested compliant replacement, so that I can act without re-reading the entire contract | Each finding card shows: quoted text, violation type, DPDPA reference (section + rule number), explanation in the same language as the contract, suggested replacement clause |
| US-022 | As a Legal Counsel, I want to see a summary of all findings grouped by DPDPA violation category so that I can assess the severity of the contract's non-compliance overall | Summary table: violation category, clause count, severity (critical/major/minor) per DPDPA section |
| US-023 | As a DPO, I want to mark a flagged clause as "accepted risk with justification" so that the audit trail shows deliberate legal decisions | Acceptance recorded with user, timestamp, justification; accepted findings visible in audit log; excluded from open findings count |

### DPA Generation

| ID | Story | Acceptance Criteria |
|----|-------|---------------------|
| US-024 | As a DPO, I want to auto-generate a compliant Data Processing Agreement in the same language as the original vendor contract so that I can negotiate with the vendor without translation costs | Generated DPA: in detected or specified language; includes all DPDPA mandatory clauses (Section 8(2)); downloadable as DOCX and PDF |
| US-025 | As a Legal Counsel, I want to edit the auto-generated DPA in a rich-text editor before downloading so that I can add company-specific terms | Inline WYSIWYG editor with Indic language support; changes tracked; original auto-generated version preserved |

### Snapshot API — Erasure Engine Integration

| ID | Story | Acceptance Criteria |
|----|-------|---------------------|
| US-026 | As the Erasure Engine (Module 3), I want to request a Data Flow Map snapshot for a specific Data Principal so that I know exactly which systems to erase from | Snapshot API returns: list of systems, data categories per system, connection metadata, record identifiers; signed ECDSA P-256 |
| US-027 | As the Erasure Engine, I want the snapshot token to expire after 15 minutes so that credentials are not long-lived | Token TTL enforced server-side; expired token returns 401 with error code TOKEN_EXPIRED; Erasure Engine must request new token |
| US-028 | As a Security Engineer, I want all Snapshot API calls to be logged with the requesting service identity, DP identifier hash, and timestamp so that I have a complete audit trail | Every Snapshot API call logged in CloudTrail and internal audit log; DP identifier stored as HMAC-SHA256 hash, never raw |

---

## 6. Functional Requirements

### FR-DISC: Discovery Engine

#### FR-DISC-01: Connector Management

| Requirement | Detail |
|-------------|--------|
| FR-DISC-01.1 | Support OAuth 2.0 authorisation flow for: Zoho CRM, Zoho Books, Zoho Desk, Razorpay, WhatsApp Business API |
| FR-DISC-01.2 | Support API key authentication for: Datadog, CloudWatch (via AWS IAM read-only role), ELK/OpenSearch |
| FR-DISC-01.3 | Support PostgreSQL, MySQL, MongoDB connection strings with read-only credential validation before first scan |
| FR-DISC-01.4 | Support S3 and GCP Cloud Storage via read-only IAM role / service account with cross-account assume-role |
| FR-DISC-01.5 | All connector credentials stored in AWS Secrets Manager (`TrustStack-Discovery` account); never in application DB or logs |
| FR-DISC-01.6 | Connector credentials must be rotatable without losing historical scan data |
| FR-DISC-01.7 | Connector health check runs every 6 hours; alerts sent on auth failure before next scheduled scan |

#### FR-DISC-02: Scan Scheduling and Orchestration

| Requirement | Detail |
|-------------|--------|
| FR-DISC-02.1 | Scan jobs orchestrated by Temporal.io; each connector runs as a separate Activity with independent retry logic |
| FR-DISC-02.2 | Default scan schedule: weekly. Configurable per connector: daily, weekly, monthly, or on-demand only |
| FR-DISC-02.3 | Scan jobs must be resumable from the last successful checkpoint on failure or timeout |
| FR-DISC-02.4 | Maximum scan duration: 4 hours per full tenant scan (all connectors). Timeout triggers graceful shutdown with partial results |
| FR-DISC-02.5 | Concurrent scans across connectors within a tenant; sequential scans across tenants (no cross-tenant parallelism) |
| FR-DISC-02.6 | Scan audit log written to PostgreSQL on every state transition: started, connector_complete, failed, completed |
| FR-DISC-02.7 | On-demand scans triggered via API (`POST /v1/scans`) or dashboard; queued behind scheduled scans with configurable priority |

#### FR-DISC-03: PII Detection and Classification

| Requirement | Detail |
|-------------|--------|
| FR-DISC-03.1 | PII detection model: fine-tuned Mistral 7B on AWS SageMaker (ap-south-1). No data sent to external APIs (Section 16) |
| FR-DISC-03.2 | Detection must classify findings into DPDPA data categories: identity, contact, financial, health, biometric, children's data, location, behavioural |
| FR-DISC-03.3 | Model must detect PII in: structured DB columns, semi-structured JSON/logs, unstructured text (PDF, CSV, free-text fields) |
| FR-DISC-03.4 | Findings store: system identifier, field path / column name, data category, record count, sample field names (NOT sample values), scan ID, timestamp |
| FR-DISC-03.5 | Raw PII values must never be stored in the findings DB. Violation of this requirement is a critical security defect |
| FR-DISC-03.6 | Detection confidence score attached to each finding; findings below 80% confidence flagged for human review |
| FR-DISC-03.7 | Soft-deleted records (records with `deleted_at` column set but not physically removed) flagged as FR-DISC critical finding: "SOFT_DELETE_NON_COMPLIANT" |

#### FR-DISC-04: Retention Conflict Detection

| Requirement | Detail |
|-------------|--------|
| FR-DISC-04.1 | For Razorpay connector: automatically flag payment records as "RBI 5-year retention conflict" — erasure not available; must document |
| FR-DISC-04.2 | For Tally connector: automatically flag records tagged with GST / TDS purpose as "GST Act 6-year retention conflict" |
| FR-DISC-04.3 | For WhatsApp connector: flag Meta-hosted message logs as "external retention — mirror consent confirmations to SahmatOS" |
| FR-DISC-04.4 | All retention conflicts must be documented in the Data Flow Map with the conflicting statute cited and resolution guidance provided |

---

### FR-MAP: Data Flow Map

| Requirement | Detail |
|-------------|--------|
| FR-MAP-01 | Data Flow Map is a directed graph: nodes = systems, edges = data flows. Built from aggregated scan findings after each scan cycle |
| FR-MAP-02 | Each edge annotated with: data categories flowing, declared consent purpose, consent status (active/withdrawn/none), Third Schedule retention class |
| FR-MAP-03 | Map renders in browser within 3 seconds for graphs up to 50 nodes / 200 edges |
| FR-MAP-04 | Risk score per node: calculated as `Σ(data_category_severity × record_count)` for all findings in that system |
| FR-MAP-05 | Risk overlay: financial penalty exposure in INR per node, per data category, based on DPDPA penalty schedule (up to ₹250Cr) |
| FR-MAP-06 | Map filterable by: data category, consent status, retention class, vendor type, risk level (critical / high / medium / low) |
| FR-MAP-07 | Export formats: interactive HTML (self-contained), JSON (Erasure Engine schema), SVG, PDF (with executive summary) |
| FR-MAP-08 | Point-in-time snapshots: each scan cycle produces a versioned snapshot. Historical snapshots browsable and exportable |
| FR-MAP-09 | Map diff view: compare any two snapshots to show added/removed data flows between scan cycles |

---

### FR-AUDIT: Contract Audit Agent

#### FR-AUDIT-01: Document Ingestion

| Requirement | Detail |
|-------------|--------|
| FR-AUDIT-01.1 | Accepted formats: PDF, DOCX, JPG, PNG (scanned documents). Maximum file size: 50MB |
| FR-AUDIT-01.2 | Tesseract OCR applied to image-based PDFs and image uploads; OCR accuracy target ≥90% for all 22 Eighth Schedule scripts |
| FR-AUDIT-01.3 | Language detection applied post-OCR using fastText language ID model; supports all 22 Eighth Schedule languages + English |
| FR-AUDIT-01.4 | Multi-page documents processed as a single contract unit; clause boundaries detected across page breaks |
| FR-AUDIT-01.5 | Upload stored in S3 (`TrustStack-Discovery`, AES-256 SSE-KMS); deleted after 30 days unless user explicitly retains |
| FR-AUDIT-01.6 | Processing SLA: 20-page document fully audited within 60 seconds of upload completion |

#### FR-AUDIT-02: Compliance Taxonomy

The following non-compliant clause patterns must be detected, tagged, and cross-referenced to the relevant DPDPA provision:

| Violation ID | Pattern | DPDPA Reference | Severity |
|-------------|---------|----------------|----------|
| AUDIT-V01 | Unlimited affiliate data sharing ("share with affiliates for any purpose") | Section 5 (purpose limitation), Section 7 (consent basis) | Critical |
| AUDIT-V02 | Missing breach notification SLA or SLA longer than 72 hours | Rule 7 (breach notification — 72hr DPBI, 6hr CERT-In) | Critical |
| AUDIT-V03 | Cross-border data transfer without DPBI pre-authorization | Section 16 (cross-border restriction) | Critical |
| AUDIT-V04 | Open-ended retention ("as long as necessary", "until no longer needed") | Section 5, Rule 3 (purpose limitation, specific retention) | Major |
| AUDIT-V05 | No data minimisation language | Section 6 (data minimisation) | Major |
| AUDIT-V06 | Profiling or targeting of minors without parental consent provision | Section 9 (children's data) | Critical |
| AUDIT-V07 | Vendor claims right to use data for own product development / model training | Section 5, 7 (purpose and consent basis) | Critical |
| AUDIT-V08 | No Data Principal rights fulfilment obligation on vendor | Section 12–13 (DP rights — erasure, correction, portability) | Major |
| AUDIT-V09 | Limitation of liability clause that caps vendor breach liability below penalty exposure | Section 8(2) (processor accountability) | Major |
| AUDIT-V10 | Sub-processor engagement without Data Fiduciary notification | Section 8(2) (processor obligations) | Minor |

#### FR-AUDIT-03: DPA Generation

| Requirement | Detail |
|-------------|--------|
| FR-AUDIT-03.1 | Auto-generated DPA must include all mandatory elements required by Section 8(2) DPDPA 2023: purpose specification, data categories, retention period, deletion obligations, breach notification SLA (72hr/6hr), sub-processor notification, DP rights facilitation |
| FR-AUDIT-03.2 | DPA generated in the same language as the detected source contract by default; overridable to any of 22 languages |
| FR-AUDIT-03.3 | Legal glossary maintained for all 22 languages: DPDPA-specific terms (Data Fiduciary, Data Principal, Consent Manager) translated consistently |
| FR-AUDIT-03.4 | Generated DPA is a DOCX with tracked-changes markup showing the non-compliant clauses it replaces from the original |
| FR-AUDIT-03.5 | Generated DPA is not legal advice. Disclaimer in the output language: "Generated by AI. Review with qualified legal counsel before execution." |
| FR-AUDIT-03.6 | All generated DPAs stored in audit log with version history; deletable by tenant but not editable retroactively |

---

### FR-SNAP: Snapshot API

| Requirement | Detail |
|-------------|--------|
| FR-SNAP-01 | Snapshot API endpoint: `POST /v1/snapshots/token` — issues a one-time 15-minute TTL signed token for a given DP identifier |
| FR-SNAP-02 | Token signed with ECDSA P-256 using the `TrustStack-Discovery` KMS key; Erasure Engine verifies signature using the published public key |
| FR-SNAP-03 | Token payload: tenant ID, DP identifier (HMAC-SHA256 hash), scan snapshot version, issued-at, expires-at |
| FR-SNAP-04 | `GET /v1/snapshots/{token}` — Erasure Engine presents signed token; returns snapshot payload |
| FR-SNAP-05 | Snapshot payload: array of system objects, each containing: system type, connector ID, data categories, record count, deletion endpoint metadata |
| FR-SNAP-06 | Token is single-use: invalidated on first successful `GET /v1/snapshots/{token}` call |
| FR-SNAP-07 | Every Snapshot API call (token issuance and consumption) logged in CloudTrail and internal audit log: requesting service, DP hash, scan snapshot version, timestamp |
| FR-SNAP-08 | Snapshot API is only accessible from within the `TrustStack-ErasureEngine` VPC endpoint; not exposed on public internet |
| FR-SNAP-09 | If no scan snapshot exists for a tenant (no scan ever completed), `POST /v1/snapshots/token` returns 409 CONFLICT with code NO_SNAPSHOT_AVAILABLE |
| FR-SNAP-10 | Snapshot reflects the most recent completed scan. Staleness indicated in response: `snapshot_age_hours`. Erasure Engine must warn operator if `snapshot_age_hours > 168` (1 week) |

---

## 7. Non-Functional Requirements

### 7.1 Performance

| Metric | Target | Notes |
|--------|--------|-------|
| Full tenant scan (all connectors) | < 4 hours | Measured p95 across connector set |
| Single connector scan (Zoho CRM, up to 100K records) | < 30 minutes | Measured p95 |
| Data Flow Map render (browser) | < 3 seconds | For graphs ≤50 nodes / 200 edges |
| Contract audit (20-page PDF) | < 60 seconds | End-to-end: OCR + language detection + clause extraction + classification |
| Snapshot API token issuance | < 500ms | p99 |
| Snapshot API data retrieval | < 1 second | p99 |
| Dashboard initial load | < 2 seconds | On 4G connection |

### 7.2 Data Residency (Section 16)

| Requirement | Implementation |
|-------------|----------------|
| All compute must run in Indian cloud regions | AWS ap-south-1 (Mumbai) exclusively for `TrustStack-Discovery` account |
| PII detection model must not call external APIs | Mistral 7B on SageMaker (ap-south-1); no internet egress for inference |
| Contract documents must not leave Indian infrastructure | S3 (ap-south-1, AES-256 SSE-KMS); no third-party OCR or NLP APIs |
| Findings database must reside in India | RDS PostgreSQL 16 in ap-south-1, Multi-AZ within ap-south-1 |
| DPBI must be able to access audit logs | Audit logs in ap-south-1; DPBI access via dedicated read-only IAM role |

### 7.3 Security and Isolation

| Requirement | Implementation |
|-------------|----------------|
| Dual-role prohibition (Section 2(g)) | `TrustStack-Discovery` AWS account; no VPC peering with `TrustStack-ConsentVault` |
| Connector credentials | AWS Secrets Manager; read-only scope; never logged |
| Findings storage | Metadata only — no raw PII values in findings DB |
| Snapshot API access | VPC endpoint only; not publicly routable |
| Audit trail | CloudTrail + internal PostgreSQL audit log; 7-year retention |
| Encryption at rest | AES-256 SSE-KMS for all S3, RDS storage |
| Encryption in transit | TLS 1.3 for all API endpoints |

### 7.4 Language Support

| Requirement | Detail |
|-------------|--------|
| Contract Audit must support all 22 Eighth Schedule languages | Assamese, Bengali, Bodo, Dogri, Gujarati, Hindi, Kashmiri, Kannada, Konkani, Maithili, Malayalam, Manipuri (Meitei), Marathi, Nepali, Odia, Punjabi, Sanskrit, Santali, Sindhi, Tamil, Telugu, Urdu |
| DPA generation in all 22 languages | Legal glossary maintained per language; human QA per language before production release |
| Dashboard UI | English and Hindi at launch; 22-language UI by May 2027 |
| OCR script support | Devanagari, Tamil, Telugu, Kannada, Malayalam, Bengali, Gujarati, Gurmukhi, Odia, Meitei Mayek, Ol Chiki, Nastaliq (Urdu), HarfBuzz for RTL rendering |

### 7.5 Reliability

| Metric | Target |
|--------|--------|
| API availability | 99.9% monthly |
| Scan job success rate | ≥98% (Temporal.io retry handles transient failures) |
| Data loss | RPO = 0 for findings DB (Multi-AZ RDS); RTO = 1 hour |
| Snapshot API availability | 99.95% (Erasure Engine depends on it for time-sensitive operations) |

### 7.6 Compliance Audit Trail

| Requirement | Detail |
|-------------|--------|
| All scan events logged | scan_id, tenant_id, connector_id, status, timestamps — 7-year retention |
| All contract audit events logged | contract_id, tenant_id, language, findings count, DPA generated — 7-year retention |
| All Snapshot API calls logged | token_id, requesting_service, dp_identifier_hash, snapshot_version, timestamp — 7-year retention |
| Logs are append-only | No UPDATE or DELETE on audit log tables; enforced by PostgreSQL row-level security |

---

## 8. Pricing

| Offering | Price | Unit |
|----------|-------|------|
| AI Data Discovery — per Purpose-Linked Data Flow discovered | ₹999 | Per unique Purpose-Linked Data Flow |
| NLP Contract Audit | ₹499 | Per contract upload |
| Discovery + Awareness Pro bundle | ₹1,999/mo | Unlimited scans up to 5 connectors |
| Growth Plan (includes Discovery) | ₹14,999/mo | Unlimited connectors, weekly scans, priority support |
| Enterprise | Custom | Dedicated SageMaker instance, SLA, custom connectors |

**Billing logic for Discovery:**
A "Purpose-Linked Data Flow" is a unique tuple of (source system, data category, declared consent purpose). Example: (Zoho CRM, `contact.phone`, `marketing_sms`) is one flow. (Zoho CRM, `contact.phone`, `order_fulfilment`) is a separate flow. Billing is one-time per discovered flow; re-scans that find the same flow are not re-billed. New flows discovered in subsequent scans are billed at the point of discovery.

**Billing logic for Contract Audit:**
Charged at upload. Re-processing the same document (e.g., after language override) does not incur additional charge within 7 days of original upload.

---

## 9. Out of Scope (v1.0)

| Feature | Rationale |
|---------|-----------|
| Automated erasure execution from Discovery findings | Erasure is Module 3 (SahmatOS Erasure Engine). Discovery provides the inventory; execution is Module 3's responsibility. |
| Real-time continuous scanning (stream-based) | v1.0 uses scheduled and on-demand batch scans via Temporal.io. Streaming connectors (Kafka, Kinesis) in roadmap for v2.0. |
| Consent purpose auto-mapping to legal basis | Requires integration with SahmatOS Consent Vault — cross-module feature, scoped to v2.0 |
| PII masking / redaction in vendor systems | Write operations on vendor systems outside Discovery's read-only scope; requires explicit erasure module integration |
| DPA e-signature workflow | Third-party e-sign integration (Leegality / DigiSign) in roadmap; not in v1.0 scope |
| Discovery for on-premise servers (not cloud/SaaS) | Requires agent installation on customer infra; security and operational complexity pushed to v2.0 |

---

## 10. Dependencies and Assumptions

| Item | Detail |
|------|--------|
| Module 3 (Erasure Engine) is a separate consumer | Erasure Engine calls the Snapshot API; it does not share the Discovery DB or compute |
| Vendor API stability | Zoho, Razorpay, WhatsApp connector implementations depend on vendor API stability. Breaking changes tracked via Connector Health Monitor |
| SageMaker model availability | Mistral 7B fine-tuned model must be trained and evaluated before connector launch. Model versioning with rollback capability required |
| Tenant must provide read-only credentials | Discovery cannot scan systems for which it has not been granted access. Onboarding flow must guide tenants through least-privilege credential setup |
| Legal review of DPA templates | Auto-generated DPAs must be reviewed by a qualified legal counsel (empanelled by TrustStack) before each language variant is released to production |

---

## 11. Milestones

| Milestone | Target Date | Key Deliverables |
|-----------|-------------|-----------------|
| M1: Connector MVP (Zoho + PostgreSQL) | Q2 2026 | 2 connectors, PII detection, basic findings dashboard |
| M2: Data Flow Map v1 | Q3 2026 | Interactive graph, JSON export, Snapshot API v1 |
| M3: Contract Audit v1 (English + Hindi) | Q3 2026 | Upload, clause detection, DPA generation in 2 languages |
| M4: Full connector suite (9 connectors) | Q4 2026 | All connectors in connector table; Razorpay retention conflict detection |
| M5: 22-language Contract Audit | Q1 2027 | All Eighth Schedule languages for audit + DPA generation |
| M6: Enterprise GA | Q2 2027 | SLA, dedicated compute, custom connectors, pre-enforcement launch |

---

*TrustStack — Build Trust. Ship Faster.*  
*TrustStack Discovery — Know your data before the regulator does.*
