# SahmatOS — Consent Vault

> **सहमत** (Sahmat) = Consent in Hindi  
> India's first Registered Consent Manager infrastructure, built for Bharat — 22 languages, mobile-first, DPDPA 2023 compliant.

---

## What This Is

SahmatOS is **Module 3** of TrustStack — the implementation and automation layer that makes DPDPA compliance real. It is the consent ledger, erasure engine, and breach response system that businesses embed into their products.

Built primarily for Indic-language users across Tier 2–3 cities who interact in Hindi, Tamil, Telugu, Bengali, Marathi, and 18 other Eighth Schedule languages — not English.

---

## Regulatory Basis

| Rule | Requirement | Implementation |
|------|-------------|----------------|
| Section 2(g) DPDPA | Registered Consent Manager | TrustStack acts as CM bridge |
| Section 5, Rule 3 | Notice in user's interface language | 22-language notice engine |
| Rule 4 (Nov 2026) | ₹2Cr net worth, AES-256, 7yr retention | Append-only PostgreSQL + KMS |
| Rule 7 | 72hr DPBI + 6hr CERT-In breach notification | Temporal.io breach workflow |
| Section 9 | Verifiable parental consent for under-18 | Aadhaar eKYC + guardian flow |
| Section 16 | Data stays on Indian cloud regions | AWS Mumbai / GCP Mumbai only |
| Third Schedule | Class-specific erasure timelines | Temporal.io erasure workflow |

---

## Directory Structure

```
consent-vault/
├── README.md                    # This file
├── docs/
│   ├── PRD.md                   # Product Requirements Document
│   ├── SYSTEM_DESIGN.md         # System architecture, components, data flows
│   ├── DATA_MODEL.md            # PostgreSQL schemas, ER diagrams, DPDPA annotations
│   ├── API_SPEC.md              # OpenAPI 3.1 contracts for all endpoints
│   ├── INDIC_LANGUAGE_SPEC.md   # 22-language support: detection, rendering, translation
│   ├── SECURITY_DESIGN.md       # ECDSA signing, KMS, encryption, access control
│   └── DEV_GUIDE.md             # Developer setup, testing, conventions
├── diagrams/
│   ├── system-architecture.md   # High-level component diagram (Mermaid)
│   ├── consent-flow.md          # Consent lifecycle state machine (Mermaid)
│   ├── data-model.md            # Entity-relationship diagram (Mermaid)
│   ├── api-sequence.md          # API interaction sequence diagrams (Mermaid)
│   ├── deployment.md            # AWS/GCP deployment topology (Mermaid)
│   ├── language-flow.md         # Indic language detection + rendering flow (Mermaid)
│   └── erasure-workflow.md      # Temporal.io erasure + breach workflows (Mermaid)
└── src/                         # Source code (to be added)
```

---

## Core Capabilities

### 1. Consent Vault (SahmatOS)
- ECDSA P-256 signed consent artifacts (tamper-evident, verifiable by DPBI)
- Append-only immutable PostgreSQL ledger (7-year retention, Rule 4 compliant)
- Per-purpose granularity — one consent record per purpose per Data Principal
- 22-language widget rendering, language auto-detection
- Parental consent workflow with Aadhaar eKYC (Section 9)
- Consumer dashboard: view, manage, withdraw all consents in 1 tap

### 2. Erasure Engine
- Class-specific erasure timelines (Third Schedule): E-commerce 3yr, Gaming 3yr, Others: on-purpose-fulfilment
- 48-hour pre-deletion notification to Data Principal in their language
- Verifiable deletion certificates (ECDSA signed, DPBI-submittable)
- Propagation to connected data stores (Zoho, Razorpay, Tally, WhatsApp)
- Soft-delete is non-compliant — hard deletion enforced

### 3. Regulatory War Room
- 72-hour DPBI breach notification workflow
- 6-hour CERT-In incident report workflow
- Annual audit report generator
- Regulatory change tracker (DPBI circulars)

---

## Language Philosophy

**Indic languages are not a feature — they are the product.**

Every Indian with a smartphone is a potential Data Principal. 70%+ of India's internet users prefer content in their regional language (IAMAI 2024). Designing in English first and translating later produces consent flows that are inaccessible to the people DPDPA is meant to protect.

Our process:
1. Design consent flows in **Hindi and Tamil first**
2. Verify comprehension with native speakers in Tier 2–3 cities
3. Adapt to English (not translate — adapt)
4. Extend to remaining 20 Eighth Schedule languages

See `docs/INDIC_LANGUAGE_SPEC.md` for the full 22-language implementation spec.

---

## Tech Stack Summary

| Layer | Technology | Why |
|-------|-----------|-----|
| API | TypeScript (ESM), Node.js 20 LTS | Type safety, ESM-first |
| Consent DB | PostgreSQL 16 (append-only) | ACID, immutable, DPBI-auditable |
| Event bus | Apache Kafka | Async consent/erasure events |
| Workflow | temporal.io | Durable multi-year erasure timelines |
| Signing | AWS KMS + ECDSA P-256 | Per-tenant keys, tamper-evident |
| Encryption | AES-256 at rest (RDS), TLS 1.3 in transit | Rule 4 requirement |
| Translation | Fine-tuned LLM (on-premise) | No PII leaves Indian infra |
| Frontend | React Native (mobile-first) | 5.5" screen, 4G Tier 2-3 |
| Cloud | AWS Mumbai (ap-south-1) + GCP Mumbai (asia-south1) | Section 16 data residency |
| Cache | Redis (ElastiCache) | Consent status hot-path |
| Search | OpenSearch | Audit log queries |

---

## Quick Start (Development)

```bash
# Prerequisites: Node.js 20+, Docker, AWS CLI (ap-south-1 profile)
git clone <repo>
cd consent-vault

# Install dependencies
npm install

# Start local stack (PostgreSQL, Kafka, Redis, Temporal)
docker-compose up -d

# Run database migrations
npm run db:migrate

# Seed test data (includes all 22 language test cases)
npm run db:seed

# Start API server
npm run dev

# Run tests
npm test             # unit tests (Vitest)
npm run test:e2e     # E2E tests (Playwright)
npm run test:lang    # Language rendering tests (all 22 scripts)
```

See `docs/DEV_GUIDE.md` for detailed setup instructions.

---

## Key Contacts / Reading Order

For **Product teams**: Start with `docs/PRD.md`, then `docs/INDIC_LANGUAGE_SPEC.md`  
For **Engineering**: Start with `docs/SYSTEM_DESIGN.md`, then `docs/DATA_MODEL.md` → `docs/API_SPEC.md`  
For **Security/Compliance**: Start with `docs/SECURITY_DESIGN.md`  
For **Visual overview**: Open any file in `diagrams/` — all use Mermaid, render in GitHub/Obsidian  

---

## Compliance Status

- [x] DPDPA 2023 core requirements mapped
- [x] DPDP Rules 2025 incorporated
- [ ] Consent Manager registration (Rule 4) — target Nov 2026
- [ ] DPBI sandbox testing — target Q3 2026
- [ ] Third-party security audit — target Q4 2026

---

*TrustStack — Build Trust. Ship Faster.*  
*SahmatOS — India का Consent Infrastructure*
