# Automatic Erasure Engine — TrustStack Module 3.2

> **Premium Feature.** Automated, agentic data erasure across all connected systems — built for DPDPA compliance without destroying business operations.

---

## What This Is

The Automatic Erasure Engine is TrustStack's premium data deletion product. It uses **multi-agent AI** to autonomously find, plan, execute, and verify data deletion across every system where a Data Principal's personal data lives — then generates a cryptographically signed deletion certificate as legal evidence.

**It is the only product that connects Module 2's Discovery results to Module 3's compliance obligations.** Without discovery, you don't know where data lives. Without automated erasure, deletion is error-prone and incomplete.

---

## Why It's Premium

Manual data deletion across Zoho CRM + Tally + Razorpay + WhatsApp + your own DB + your data warehouse + your backups is a 3-day engineering project per request. That's untenable at scale.

But automated deletion without guardrails is catastrophic — delete the wrong record in a payment system and you lose transaction history, trigger accounting audits, or break regulatory requirements outside DPDPA.

The Erasure Engine is premium because it requires:
1. Agentic AI that understands data relationships and deletion safety
2. Human-in-the-loop approval gates for sensitive operations
3. Per-business customisation of deletion policies
4. Integration depth with each connected system

**Pricing: ₹2,499–9,999/mo by Data Principal volume (see `docs/PRD.md`)**

---

## Relationship to Module 2 (Discovery Engine)

```
Module 2: Discovery Engine          Module 3.2: Erasure Engine
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
"Where does this person's         "Delete it from all of those
 data exist across all our         places, in the right order,
 systems?"                         with proof."

Outputs: Data Flow Map JSON         Consumes: Data Flow Map JSON
         PII location index         Produces: Deletion certificates
         System inventory                     Audit trail
```

**Critical boundary:** The Erasure Engine reads discovery output from Module 2. It does NOT have direct access to the Discovery Engine's live scanning capabilities during deletion (dual-role prohibition: Module 2 runs in a separate AWS account and cannot access consent data — see `docs/SYSTEM_DESIGN.md`).

---

## User Segment

**Primary:** DPO (Data Protection Officer), IT Manager, Compliance Officer at SME/Mid-market Data Fiduciaries  
**Secondary:** CTO / Engineering lead reviewing erasure operation logs  
**Watch-only:** Regulatory War Room (Module 3.3) consumes erasure status for CXO dashboard  

---

## Directory Structure

```
erasure-engine/
├── README.md                       # This file
├── docs/
│   ├── PRD.md                      # Product Requirements Document
│   ├── SYSTEM_DESIGN.md            # Agentic AI architecture, safety mechanisms
│   ├── DATA_MODEL.md               # Erasure plan, execution log, certificate schemas
│   ├── API_SPEC.md                 # REST API contracts
│   └── AGENTIC_AI_DESIGN.md        # Multi-agent design, agent responsibilities, safety
├── diagrams/
│   ├── system-architecture.md      # Component diagram + network isolation (Mermaid)
│   ├── agentic-erasure-flow.md     # Multi-agent execution flow + safety gates (Mermaid)
│   └── data-model.md               # ER diagram (Mermaid)
└── src/                            # Source code (to be added)
```

---

## Core Capabilities

### 1. Discovery-Driven Deletion Planning
- Consumes Module 2's Data Flow Map to know exactly where data exists
- Plans deletion order respecting data dependencies (foreign keys, audit logs)
- Identifies systems that cannot be auto-deleted (require manual process)

### 2. Multi-Agent Execution
- **Planner Agent**: Creates deletion plan from Data Flow Map
- **Safety Agent**: Reviews plan for risk (flags destructive operations)
- **Executor Agent**: Calls deletion APIs per connected system
- **Verifier Agent**: Confirms deletion post-execution
- **Certificate Agent**: Signs and issues ECDSA-signed deletion certificate

### 3. Human-in-the-Loop Approval Gates
- Auto-approve: ≤100 records, non-sensitive categories, STANDARD class DF
- Human approval required: >1,000 records, sensitive data, financial records
- Emergency stop: override at any point before execution completes

### 4. Deletion Certificates (DPBI-Submittable)
- ECDSA P-256 signed by tenant's KMS key
- Contains: per-system deletion evidence, timestamps, verifier agent output
- Stored for 7 years (Rule 4 retention of processing logs)

### 5. Third Schedule Class-Specific Timelines
- ECOMMERCE_GT_2CR / GAMING_GT_50L / SOCIAL_GT_2CR: 3yr from last activity
- STANDARD (most SMEs): immediate on consent withdrawal / purpose fulfilment
- Durable scheduling via Temporal.io (survives restarts across multi-year timelines)

---

## Integration Points

| System | Integration Type | Deletion Method |
|--------|-----------------|-----------------|
| Tenant's own database | REST API (tenant-provided) | DF-supplied deletion endpoint |
| Zoho CRM | Zoho API | Record deletion + data purge |
| Razorpay | Razorpay API | Customer record erasure |
| Tally | Tally Connector | Financial record handling (with accountant flag) |
| WhatsApp Business | Meta API | Contact + chat history deletion |
| AWS S3 (tenant) | AWS SDK | Object deletion + bucket lifecycle |
| Google Workspace | Admin SDK | User data export + deletion |
| Module 2 Discovery Data | Internal API | Remove DP from PII index |

---

## Safety Guarantees

1. **Dry-run mode**: Full plan generation and simulation without any actual deletion
2. **Staged execution**: Delete from lowest-risk systems first, escalate to approval for high-risk
3. **48hr cancellation window**: Built into Temporal workflow (DPDPA-required notification window)
4. **No rollback post-execution**: DPDPA requires hard deletion — we don't soften this
5. **Independent verification**: Verifier Agent re-queries each system after deletion
6. **Partial failure handling**: Certificate explicitly marks which systems succeeded/failed

---

*TrustStack — Build Trust. Ship Faster.*  
*Erasure Engine — Delete with confidence. Prove it in court.*
