# Graph Report - consent-vault  (2026-04-22)

## Corpus Check
- 15 files · ~23,358 words
- Verdict: corpus is large enough that graph structure adds value.

## Summary
- 195 nodes · 279 edges · 21 communities detected
- Extraction: 91% EXTRACTED · 9% INFERRED · 0% AMBIGUOUS · INFERRED: 24 edges (avg confidence: 0.87)
- Token cost: 0 input · 0 output

## Community Hubs (Navigation)
- [[_COMMUNITY_Breach Response & Regulatory Deadlines|Breach Response & Regulatory Deadlines]]
- [[_COMMUNITY_Indic Language Infrastructure & On-Premise Translation|Indic Language Infrastructure & On-Premise Translation]]
- [[_COMMUNITY_Consent Audit Trail & Cryptographic Signing|Consent Audit Trail & Cryptographic Signing]]
- [[_COMMUNITY_Developer Infrastructure, Testing & CICD|Developer Infrastructure, Testing & CI/CD]]
- [[_COMMUNITY_Erasure Engine & Data Principal Rights|Erasure Engine & Data Principal Rights]]
- [[_COMMUNITY_Data Model & Tenant Architecture|Data Model & Tenant Architecture]]
- [[_COMMUNITY_Compliance Workflows (Erasure + Breach APIs)|Compliance Workflows (Erasure + Breach APIs)]]
- [[_COMMUNITY_Core Stack Kafka, PostgreSQL, Redis, KMS|Core Stack: Kafka, PostgreSQL, Redis, KMS]]
- [[_COMMUNITY_Data Principal Rights & Multi-Tenant Security|Data Principal Rights & Multi-Tenant Security]]
- [[_COMMUNITY_Indian Cloud Deployment & Data Residency (Section 16)|Indian Cloud Deployment & Data Residency (Section 16)]]
- [[_COMMUNITY_RTL Script Rendering & Noto Font Pipeline|RTL Script Rendering & Noto Font Pipeline]]
- [[_COMMUNITY_Consent Channels Web Widget & WhatsApp|Consent Channels: Web Widget & WhatsApp]]
- [[_COMMUNITY_Consent Purposes Schema|Consent Purposes Schema]]
- [[_COMMUNITY_ECS Fargate Compute|ECS Fargate Compute]]
- [[_COMMUNITY_TLS 1.3 Transport Security|TLS 1.3 Transport Security]]
- [[_COMMUNITY_API Key Management|API Key Management]]
- [[_COMMUNITY_Erasure Status API|Erasure Status API]]
- [[_COMMUNITY_API Idempotency|API Idempotency]]
- [[_COMMUNITY_RFC 9457 Error Format|RFC 9457 Error Format]]
- [[_COMMUNITY_Meitei Mayek Script (Manipuri)|Meitei Mayek Script (Manipuri)]]
- [[_COMMUNITY_Ol Chiki Script (Santali)|Ol Chiki Script (Santali)]]

## God Nodes (most connected - your core abstractions)
1. `SahmatOS Consent Vault` - 23 edges
2. `SahmatOS PRD` - 11 edges
3. `ConsentService (Domain Service)` - 11 edges
4. `Erasure Workflow (Temporal.io)` - 10 edges
5. `consent_records Table (Append-Only Ledger)` - 10 edges
6. `SahmatOS API Server (Node.js/TypeScript)` - 9 edges
7. `Erasure Engine` - 8 edges
8. `TENANTS Table` - 8 edges
9. `PostgreSQL Append-Only Consent Ledger` - 7 edges
10. `Temporal.io Workflow Engine` - 7 edges

## Surprising Connections (you probably didn't know these)
- `RTL Language Support (Urdu/Kashmiri/Sindhi)` --semantically_similar_to--> `22 Eighth Schedule Language Support`  [INFERRED] [semantically similar]
  consent-vault/diagrams/language-flow.md → consent-vault/README.md
- `Amazon MSK (Kafka 3 Brokers Multi-AZ)` --implements--> `Apache Kafka Event Bus`  [INFERRED]
  consent-vault/diagrams/deployment.md → consent-vault/README.md
- `Zero-Knowledge Design (TrustStack Cannot Read PII)` --semantically_similar_to--> `Dual-Role Prohibition (Section 2(g))`  [INFERRED] [semantically similar]
  consent-vault/docs/SECURITY_DESIGN.md → consent-vault/docs/PRD.md
- `Deletion Certificate (ECDSA Signed)` --implements--> `ECDSA P-256 Consent Signing (KMSSigner)`  [INFERRED]
  consent-vault/docs/PRD.md → consent-vault/docs/SECURITY_DESIGN.md
- `Language Override Priority Chain (7-step)` --semantically_similar_to--> `Language Detection Pipeline (6-step Priority Chain)`  [INFERRED] [semantically similar]
  consent-vault/docs/API_SPEC.md → consent-vault/docs/INDIC_LANGUAGE_SPEC.md

## Hyperedges (group relationships)
- **Consent Grant: API + KMS Signing + Append-Only Ledger** — consentflow_consent_grant, readme_aws_kms, readme_postgresql_append_only, readme_ecdsa_p256 [EXTRACTED 0.95]
- **Breach Response: CERT-In 6hr + DPBI 72hr + DP Multilingual Notification** — erasure_breach_workflow, erasure_certin_6hr, erasure_dpbi_72hr, readme_22_language_support [EXTRACTED 0.95]
- **Durable Erasure: Temporal.io + Multi-Year Sleep + Hard Deletion Certificate** — erasure_workflow_temporal, erasure_durable_execution, erasure_hard_deletion, erasure_deletion_cert [EXTRACTED 0.90]
- **Consent Grant Data Flow (Widget → API → Signing → Ledger → Kafka → Cache)** — api_post_consent, sd_consent_service, sec_ecdsa_p256_signing, dm_consent_records_table, sd_kafka_event_bus, sd_redis_cache [EXTRACTED 0.95]
- **DPDPA Rule 4 Compliance Stack (Append-Only + ECDSA + 7yr Retention + Indian Residency)** — prd_rule_4_cm_registration, prd_append_only_ledger, sec_ecdsa_p256_signing, prd_indian_cloud_residency, dm_audit_log_table [EXTRACTED 0.95]
- **Breach Notification Compliance (Rule 7: Temporal + CERT-In 6hr + DPBI 72hr + DP Notify)** — prd_rule_7_breach, sd_breach_workflow, prd_temporal_io, sec_breach_response_timeline, dm_breach_events_table [EXTRACTED 0.95]

## Communities

### Community 0 - "Breach Response & Regulatory Deadlines"
Cohesion: 0.11
Nodes (27): Breach Notification Workflow (API Sequence), Parental Consent Flow (Section 9), Consent Lifecycle State Machine, Breach Notification Workflow (Rule 7 Temporal.io), CERT-In 6hr Breach Report, DPBI 72hr Breach Report, Phase 1 Languages (Hindi/Tamil/Telugu/Kannada/Bengali/Marathi/Gujarati), 22 Eighth Schedule Language Support (+19 more)

### Community 1 - "Indic Language Infrastructure & On-Premise Translation"
Cohesion: 0.09
Nodes (26): DPDPA Section Reference Code Comments (Convention), Language Coverage Phases (MVP Nov 2026 to May 2027), consent_purpose_translations Table, DPDPA Legal Glossary (22 Languages), Human QA Checklist Per Language (Pre-Production), On-Premise Fine-Tuned LLM for Translation, Rationale: Language is infrastructure not a feature - 22-language support built in from day one, Rationale: On-premise LLM avoids Section 16 cross-border PII risk and ensures legal precision (+18 more)

### Community 2 - "Consent Audit Trail & Cryptographic Signing"
Cohesion: 0.11
Nodes (25): GET /v1/audit (Audit Log Export), GET /v1/consent/{artifactId} (Audit), POST /v1/consent (Grant/Withdraw), Reversible DB Migration Strategy, No PII Logging Convention, Audit Log Hash Chain (Blockchain-Style), audit_log Table, consent_records Table (Append-Only Ledger) (+17 more)

### Community 3 - "Developer Infrastructure, Testing & CI/CD"
Cohesion: 0.1
Nodes (25): GET /v1/consent/check (Consent Status), GET /v1/widget/consent (Widget Render), Language Override Priority Chain (7-step), Webhook Events (consent.withdrawn, erasure.completed, etc.), Docker Compose Local Infrastructure, GitHub Actions CI/CD Pipeline, LocalStack (AWS KMS + Secrets Manager Emulator), Playwright E2E Tests (+17 more)

### Community 4 - "Erasure Engine & Data Principal Rights"
Cohesion: 0.16
Nodes (16): Erasure Request Flow (API Sequence), Consent Withdrawal Flow, Consumer Dashboard (1-tap withdrawal), BREACH_EVENTS Table, 48-Hour Pre-Deletion Notification, Data Fiduciary Class (Third Schedule), Temporal.io Durable Execution Model, Hard Deletion (Non-Compliant Soft-Delete Rejected) (+8 more)

### Community 5 - "Data Model & Tenant Architecture"
Cohesion: 0.17
Nodes (16): Data Fiduciary Onboarding Flow, AUDIT_LOG Table (Chain Integrity), CONSENT_PURPOSE_TRANSLATIONS Table, CONSENT_PURPOSES Table, CONSENT_RECORDS Table (Append-Only), CONSUMER_ACCOUNTS Table, DP Identifier Hash (HMAC-SHA256, Never Raw PII), DP_IDENTIFIERS Table (+8 more)

### Community 6 - "Compliance Workflows (Erasure + Breach APIs)"
Cohesion: 0.24
Nodes (13): POST /v1/breach (Trigger Breach Workflow), POST /v1/erasure (Schedule Erasure), breach_events Table, erasure_records Table, Breach Notification Workflow, Deletion Certificate (ECDSA Signed), Erasure Engine, DPDPA Rule 7 - Breach Notification (+5 more)

### Community 7 - "Core Stack: Kafka, PostgreSQL, Redis, KMS"
Cohesion: 0.35
Nodes (11): Real-Time Consent Check API (Redis Hot-Path), Consent Grant Flow (Happy Path), Language Detection Pipeline, AWS KMS (ap-south-1), Apache Kafka Event Bus, PostgreSQL Append-Only Consent Ledger, Redis Cache (ElastiCache), Application Load Balancer (TLS 1.3) (+3 more)

### Community 8 - "Data Principal Rights & Multi-Tenant Security"
Cohesion: 0.22
Nodes (9): GET /v1/consent (Consumer Dashboard List), consumer_accounts Table, DF Class Erasure Timeline (Third Schedule), dp_identifiers Table (PII Table), Row-Level Security (Tenant Isolation), tenants Table (Data Fiduciaries), Consumer Dashboard (Data Principal), DPDPA Section 12 - Data Principal Rights (+1 more)

### Community 9 - "Indian Cloud Deployment & Data Residency (Section 16)"
Cohesion: 0.29
Nodes (8): AWS ap-south-1 Mumbai (Primary), CI/CD Pipeline (GitHub Actions + Blue/Green), ECS Fargate Cluster (Auto-scaling), GCP asia-south1 Mumbai (DR), Amazon MSK (Kafka 3 Brokers Multi-AZ), RDS Multi-AZ PostgreSQL 16, AWS WAF (Rate Limiting + Bot Protection), Section 16 Data Residency (Indian Cloud)

### Community 10 - "RTL Script Rendering & Noto Font Pipeline"
Cohesion: 0.4
Nodes (5): HarfBuzz Text Shaping, Noto Font Family (Devanagari/Tamil/Nastaliq), RTL Language Support (Urdu/Kashmiri/Sindhi), Consent Widget Rendering Pipeline, Widget CDN (CloudFront ap-south-1)

### Community 11 - "Consent Channels: Web Widget & WhatsApp"
Cohesion: 0.4
Nodes (5): SahmatOS Widget JavaScript SDK, Consent Widget Mobile-First Layout (5.5" 360x800), WhatsApp Conversational Consent Flow, Consent Widget (JS SDK + React Native), WhatsApp Consent Collection

### Community 12 - "Consent Purposes Schema"
Cohesion: 1.0
Nodes (1): consent_purposes Table

### Community 13 - "ECS Fargate Compute"
Cohesion: 1.0
Nodes (1): AWS ECS Fargate (API Servers)

### Community 14 - "TLS 1.3 Transport Security"
Cohesion: 1.0
Nodes (1): TLS 1.3 Transport Security

### Community 15 - "API Key Management"
Cohesion: 1.0
Nodes (1): API Key Management (Hashed, Never Stored Raw)

### Community 16 - "Erasure Status API"
Cohesion: 1.0
Nodes (1): GET /v1/erasure/{id} (Erasure Status)

### Community 17 - "API Idempotency"
Cohesion: 1.0
Nodes (1): API Idempotency via X-Request-Id

### Community 18 - "RFC 9457 Error Format"
Cohesion: 1.0
Nodes (1): RFC 9457 Problem Details Error Format

### Community 19 - "Meitei Mayek Script (Manipuri)"
Cohesion: 1.0
Nodes (1): Meitei Mayek Script (Manipuri, U+ABC0)

### Community 20 - "Ol Chiki Script (Santali)"
Cohesion: 1.0
Nodes (1): Ol Chiki Script (Santali, U+1C50)

## Knowledge Gaps
- **68 isolated node(s):** `TrustStack Platform`, `DPDP Rules 2025`, `React Native Frontend`, `Data Principal`, `Data Fiduciary` (+63 more)
  These have ≤1 connection - possible missing edges or undocumented components.
- **Thin community `Consent Purposes Schema`** (1 nodes): `consent_purposes Table`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `ECS Fargate Compute`** (1 nodes): `AWS ECS Fargate (API Servers)`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `TLS 1.3 Transport Security`** (1 nodes): `TLS 1.3 Transport Security`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `API Key Management`** (1 nodes): `API Key Management (Hashed, Never Stored Raw)`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `Erasure Status API`** (1 nodes): `GET /v1/erasure/{id} (Erasure Status)`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `API Idempotency`** (1 nodes): `API Idempotency via X-Request-Id`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `RFC 9457 Error Format`** (1 nodes): `RFC 9457 Problem Details Error Format`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `Meitei Mayek Script (Manipuri)`** (1 nodes): `Meitei Mayek Script (Manipuri, U+ABC0)`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `Ol Chiki Script (Santali)`** (1 nodes): `Ol Chiki Script (Santali, U+1C50)`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.

## Suggested Questions
_Questions this graph is uniquely positioned to answer:_

- **Why does `SahmatOS Consent Vault` connect `Breach Response & Regulatory Deadlines` to `Indian Cloud Deployment & Data Residency (Section 16)`, `Erasure Engine & Data Principal Rights`, `Data Model & Tenant Architecture`, `Core Stack: Kafka, PostgreSQL, Redis, KMS`?**
  _High betweenness centrality (0.098) - this node is a cross-community bridge._
- **Why does `SahmatOS PRD` connect `Indic Language Infrastructure & On-Premise Translation` to `Data Principal Rights & Multi-Tenant Security`, `Consent Audit Trail & Cryptographic Signing`, `Compliance Workflows (Erasure + Breach APIs)`?**
  _High betweenness centrality (0.098) - this node is a cross-community bridge._
- **Why does `ConsentService (Domain Service)` connect `Developer Infrastructure, Testing & CI/CD` to `Indic Language Infrastructure & On-Premise Translation`, `Consent Audit Trail & Cryptographic Signing`?**
  _High betweenness centrality (0.077) - this node is a cross-community bridge._
- **Are the 2 inferred relationships involving `Erasure Workflow (Temporal.io)` (e.g. with `Erasure Engine` and `Erasure Request Flow (API Sequence)`) actually correct?**
  _`Erasure Workflow (Temporal.io)` has 2 INFERRED edges - model-reasoned connections that need verification._
- **What connects `TrustStack Platform`, `DPDP Rules 2025`, `React Native Frontend` to the rest of the system?**
  _68 weakly-connected nodes found - possible documentation gaps or missing edges._
- **Should `Breach Response & Regulatory Deadlines` be split into smaller, more focused modules?**
  _Cohesion score 0.11 - nodes in this community are weakly interconnected._
- **Should `Indic Language Infrastructure & On-Premise Translation` be split into smaller, more focused modules?**
  _Cohesion score 0.09 - nodes in this community are weakly interconnected._