# Product Requirements Document — Automatic Erasure Engine

**Version:** 1.0  
**Date:** 2026-04-22  
**Status:** Draft  
**Owner:** TrustStack Product Team  

---

## 1. Executive Summary

The Automatic Erasure Engine is TrustStack's premium compliance product that automates the most operationally dangerous DPDPA obligation: data deletion. When a Data Principal withdraws consent, fulfils a purpose's lifecycle, or exercises their right to erasure (Section 12), their personal data must be hard-deleted from every system where it exists — with cryptographic proof.

Without automation, this requires engineering effort, is error-prone (easy to miss a data store), and breaks under volume. With automation, it becomes a workflow that runs 24/7, respects timelines, and produces DPBI-submittable evidence.

**It is premium because data deletion is irreversible.** Every safety mechanism, human approval gate, and verification step exists to ensure that when data is deleted, it is the right data, from the right systems, at the right time.

---

## 2. Problem Statement

### 2.1 The Operational Impossibility

A mid-size D2C brand in India has a customer's data in:
- Their own PostgreSQL database (orders, addresses, profiles)
- Zoho CRM (sales notes, call history)
- Razorpay (payment methods, KYC documents)
- WhatsApp Business (conversation history, opt-in records)
- AWS S3 (invoice PDFs, product photos tagged to customer)
- Tally (accounting records tied to customer transactions)
- Their own data warehouse (BigQuery/Redshift analytics)
- Email marketing platform (Mailchimp/Sendgrid)
- Possibly a data broker they forgot about

When a customer exercises their right to erasure (Section 12 DPDPA), this D2C brand has **at most 48 hours** (after the 48hr notification window) to hard-delete from all of these. No engineering team at an SME can reliably do this manually at scale.

### 2.2 The Compliance Gap

- DPDPA has no de minimis threshold — even one missed data store is non-compliant
- Soft delete (archive flag) is explicitly non-compliant
- Deletion certificates must be verifiable by DPBI
- Partial deletion is not meaningful compliance — you either deleted all of it or you didn't
- Class-specific timelines add a scheduling dimension (3yr for large DFs is a long-running workflow)

### 2.3 The Risk Premium

This product carries more risk than consent collection. A wrong deletion:
- Destroys financial transaction records (creates accounting and tax compliance problems)
- Breaks referential integrity in business databases
- May conflict with other regulatory obligations (SEBI, RBI — financial data retention)
- Is irreversible — DPDPA does not allow soft delete as a fallback

The Erasure Engine must navigate these conflicts with intelligence, not just follow a delete instruction blindly.

---

## 3. User Personas

### 3.1 Nisha, DPO/Compliance Head, 200-person HealthTech (Primary)

- **Language:** English/Hindi
- **Role:** Responsible for DPDPA compliance across the organisation
- **Pain:** Gets erasure requests from patients. Has to coordinate with 4 engineering teams to delete from 6 systems. Takes 2 weeks per request.
- **Need:** One-click erasure initiation with audit trail. Human review before any medical record deletion. Certificate she can show DPBI.
- **Risk tolerance:** Very low — patient data deletion has regulatory conflicts with MoHFW records requirements

### 3.2 Karan, IT Manager, 50-person Retail Brand (Primary)

- **Language:** Hindi/English
- **Role:** Manages all technical systems, including Tally, Razorpay, and the company's own app
- **Pain:** Received a Section 12 erasure request. Doesn't know where all the customer's data is.
- **Need:** Let the system discover where data lives (Module 2) and then delete it. Simple queue-based interface. Alert when done.
- **Risk tolerance:** Medium — will approve auto-deletion for CRM data, wants manual confirmation for financial records

### 3.3 Rohit, CTO, 500-person E-Commerce (Secondary)

- **Language:** English
- **Role:** Responsible for engineering compliance and system reliability
- **Pain:** Has 50,000+ erasure operations scheduled over 3 years (ECOMMERCE_GT_2CR class). Cannot build the scheduling infrastructure himself.
- **Need:** Bulk erasure scheduling, API integration, SLA reporting, integration with existing data pipelines
- **Risk tolerance:** Low — any data loss incident is a P0 incident

### 3.4 Priya, Data Principal (End User — indirect)

- **Language:** Hindi
- **Not a direct user of the product UI**, but the ultimate beneficiary
- Her data gets deleted, she gets notified in Hindi, receives a deletion certificate
- Quality bar: notification must be understandable, not legal jargon

---

## 4. User Stories

### 4.1 Erasure Initiation

| ID | As a... | I want to... | So that... | Priority |
|----|---------|-------------|-----------|----------|
| EI-01 | DPO | Initiate erasure from a single dashboard when I receive a Section 12 request | I don't coordinate across 6 teams | P0 |
| EI-02 | Compliance head | Trigger erasure automatically when a DP withdraws consent | Compliance is automatic, not manual | P0 |
| EI-03 | IT Manager | See exactly which systems will be affected before deletion runs | I can flag conflicts before they happen | P0 |
| EI-04 | CTO | Trigger erasure via API from our own system | We can integrate with our internal tooling | P0 |
| EI-05 | DPO | Run a dry-run of an erasure plan without deleting anything | I can review the impact before committing | P0 |
| EI-06 | DPO | See what data the AI found for this DP across all our systems | I trust the completeness of the plan | P1 |

### 4.2 Approval and Safety

| ID | As a... | I want to... | So that... | Priority |
|----|---------|-------------|-----------|----------|
| AS-01 | DPO | Be shown a human approval gate before any financial data is deleted | We don't accidentally destroy accounting records | P0 |
| AS-02 | DPO | Set custom approval thresholds (e.g., always review deletions >500 records) | I control the risk tolerance | P0 |
| AS-03 | IT Manager | Emergency stop an in-progress erasure | Mistakes can be caught if caught early enough | P0 |
| AS-04 | DPO | See the AI agent's reasoning for each deletion decision | I trust the agent's plan | P1 |
| AS-05 | CTO | Configure which systems require manual override (never auto-delete) | We protect our most critical data stores | P1 |

### 4.3 Execution and Monitoring

| ID | As a... | I want to... | So that... | Priority |
|----|---------|-------------|-----------|----------|
| EM-01 | DPO | Track real-time progress of an erasure across all systems | I know when it's complete | P0 |
| EM-02 | IT Manager | Be alerted if an erasure fails on any system | I can manually intervene | P0 |
| EM-03 | DPO | See which systems succeeded and which failed | Partial success is still partial non-compliance | P0 |
| EM-04 | DPO | Retry failed deletions on specific systems | Not restart the whole operation | P1 |

### 4.4 Certification and Evidence

| ID | As a... | I want to... | So that... | Priority |
|----|---------|-------------|-----------|----------|
| CE-01 | DPO | Receive a signed deletion certificate when erasure is complete | I have DPBI-submittable evidence | P0 |
| CE-02 | Compliance head | Download a deletion certificate in DPBI report format | Annual audit submission is ready | P0 |
| CE-03 | DPO | Send a deletion confirmation to the Data Principal in their language | They have evidence their right was exercised | P0 |
| CE-04 | CTO | Retrieve deletion certificates via API | We integrate into our compliance reporting | P1 |

### 4.5 Scheduling (Class-Specific Timelines)

| ID | As a... | I want to... | So that... | Priority |
|----|---------|-------------|-----------|----------|
| SC-01 | CTO (E-commerce class) | Have erasure automatically scheduled 3yr after last activity | I comply with Third Schedule without building scheduling infra | P0 |
| SC-02 | DPO | See all upcoming scheduled erasures (next 30/90/180 days) | I can plan for operational impact | P1 |
| SC-03 | CTO | Cancel a scheduled erasure before it runs | If the DP re-engages and re-grants consent | P1 |
| SC-04 | Compliance head | Bulk schedule erasures for a cohort of DPs | Annual data minimisation exercise | P2 |

---

## 5. Functional Requirements

### 5.1 Erasure Plan Generation

**FR-PLAN-01:** Given a Data Principal identifier, generate a complete erasure plan showing:
- All systems where the DP's data was found (from Module 2 Data Flow Map)
- Estimated records per system
- Deletion method per system (API, DB, manual)
- Deletion risk level per system (Low / Medium / High / Requires Manual)
- Regulatory conflicts identified (e.g., RBI mandates 5yr retention for payment records)
- Recommended execution order

**FR-PLAN-02:** Plans must be generated in < 30 seconds for ≤ 20 connected systems.

**FR-PLAN-03:** Dry-run mode must execute all plan steps except actual deletion calls.

**FR-PLAN-04:** The AI Planner Agent must explain its reasoning for each deletion decision in plain English (and optionally in the DPO's preferred language).

**FR-PLAN-05:** Plans must identify data that CANNOT be deleted due to conflicting regulatory obligations, with specific regulation cited.

### 5.2 Human Approval Gates

**FR-GATE-01:** Auto-approve deletion (no human needed) when ALL of:
- Total records ≤ 100
- No sensitive data categories (health, finance, biometric, children's data)
- STANDARD class DF (not ECOMMERCE_GT_2CR / GAMING / SOCIAL)

**FR-GATE-02:** Mandatory human approval when ANY of:
- Total records > 1,000
- Financial data categories involved (payment methods, transaction history, KYC)
- Health or biometric data categories
- System marked as "protected" in tenant config
- Conflicting regulatory obligations detected

**FR-GATE-03:** Approval must be via authenticated action (not just email link). Multi-person approval supported (require N of M approvers for high-risk deletions).

**FR-GATE-04:** Approval request expires after 24 hours. DPO notified. Erasure paused.

### 5.3 Execution

**FR-EXEC-01:** Execution must respect deletion order defined in the plan (dependencies first).

**FR-EXEC-02:** Each deletion call must have a timeout (configurable, default 30s). Retry 3 times with exponential backoff before marking as failed.

**FR-EXEC-03:** Partial failure must not stop execution. Continue with remaining systems. Certificate explicitly marks failed systems.

**FR-EXEC-04:** Emergency stop must halt execution within 60 seconds of trigger. Systems not yet deleted are marked "pending" and the workflow pauses.

**FR-EXEC-05:** All deletion operations are idempotent (safe to retry if the workflow is interrupted mid-execution).

### 5.4 Verification

**FR-VERIFY-01:** After deletion, the Verifier Agent must query each system to confirm the DP's data no longer exists.

**FR-VERIFY-02:** Verification must use a different API call/query than the deletion call (not just a success response from the deletion).

**FR-VERIFY-03:** Verification failure must be flagged in the deletion certificate and reported to DPO.

**FR-VERIFY-04:** Verification must be logged in the audit trail with timestamp and query used.

### 5.5 Deletion Certificates

**FR-CERT-01:** Certificate contains:
- Erasure operation ID, tenant ID, DP identifier hash
- List of all targeted systems with: deletion status (DELETED / FAILED / EXCLUDED), records deleted, deletion timestamp, verification status
- AI agent execution log (planner output, safety checks, verifier output)
- Certificate issuance timestamp
- ECDSA P-256 signature (tenant's KMS key)

**FR-CERT-02:** Certificate must be stored for 7 years (Rule 4 — processing logs).

**FR-CERT-03:** Certificate available for download in PDF (human-readable) and JSON (machine-readable / DPBI format).

**FR-CERT-04:** DP notification of deletion must be sent in their registered language with a reference to the certificate.

### 5.6 Scheduling

**FR-SCHED-01:** Temporal.io workflows must durably track erasure timelines for up to 3 years.

**FR-SCHED-02:** Workflow must survive: application restarts, infrastructure failures, cloud region availability events.

**FR-SCHED-03:** 48hr pre-deletion notification to DP in their language is mandatory before execution (Third Schedule + DPDPA Rule 7 principle of prior notice).

**FR-SCHED-04:** DP can cancel during the 48hr window. Cancellation recorded in audit trail. Workflow paused.

---

## 6. Non-Functional Requirements

| Category | Requirement | Target |
|----------|-------------|--------|
| Latency | Erasure plan generation | < 30s for ≤20 systems |
| Latency | First deletion call after approval | < 60s |
| Latency | Full erasure (STANDARD class, ≤5 systems) | < 5 minutes |
| Reliability | Erasure workflow completion rate | > 99.9% |
| Reliability | Temporal.io workflow durability | Survives 3-year timeline |
| Safety | Zero false deletions (wrong DP) | 100% — non-negotiable |
| Safety | Regulatory conflict detection rate | > 95% of known conflict patterns |
| Security | Dry-run isolation | Zero actual deletion calls in dry-run |
| Audit | Deletion log completeness | 100% of deletion attempts logged |
| Availability | Erasure Engine API uptime | 99.9% |
| Scalability | Concurrent erasure workflows | 10,000 active |

---

## 7. Premium Pricing Model

| Tier | Volume | Price | Included |
|------|--------|-------|---------|
| Starter | ≤1,000 DPs/mo | ₹2,499/mo | Auto-erasure, basic systems, standard cert |
| Growth | ≤10,000 DPs/mo | ₹4,999/mo | + Custom approval gates, retry logic |
| Enterprise | ≤100,000 DPs/mo | ₹9,999/mo | + Advanced AI reasoning, multi-approver, DPBI XML export |
| Platform | Unlimited | Custom | + White-label, dedicated infra |

**Add-ons:**
- Per-system connector (non-standard integrations): ₹999/system/mo
- Bulk erasure scheduling (cohort operations): ₹1,999/mo
- Agentic reasoning audit logs (full AI decision trail): included in Enterprise+

---

## 8. Risks and Mitigations

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|-----------|
| AI agent deletes wrong DP's data | Very Low | Critical | DP identifier verification at every step; human approval for >100 records |
| Conflicting regulation not detected (RBI, SEBI) | Medium | High | Pre-built conflict rule library; legal team review of conflict patterns quarterly |
| External system API unavailable at deletion time | Medium | Medium | Retry with backoff; partial certificate; DPO alert; retry queue |
| Temporal.io workflow lost during cloud outage | Very Low | High | Multi-AZ Temporal cluster; GCP DR for workflow state |
| DP claims data not deleted (certificate challenged) | Low | High | ECDSA signature + verifier agent output in certificate; reproducible verification |
| Tenant misconfigures "protected" systems list | Medium | Medium | System-type default policies (financial = protected by default) |

---

## Appendix: Regulatory Conflicts Library

Systems that may require manual deletion or conflict flagging:

| Data Category | Conflicting Regulation | Retention Requirement | Resolution |
|--------------|----------------------|----------------------|-----------|
| Payment records | RBI Payment System Regulations | 5 years from transaction | Mark EXCLUDED in certificate; flag for manual review |
| GST-related invoices | GST Act | 6 years | Mark EXCLUDED; inform DPO |
| PF/Employee records | EPF Act | 5 years post-separation | Applicable only to HR systems |
| Securities transactions | SEBI | 5 years | Mark EXCLUDED |
| Insurance documents | IRDAI | Policy term + 3 years | Mark EXCLUDED |
| Healthcare records (hospital) | Clinical Establishments Act | Varies by state (3-10yr) | Flag for DPO + legal review |
| Children's educational records | RTE Act | Until age 18 or graduation | Flag for legal review |
