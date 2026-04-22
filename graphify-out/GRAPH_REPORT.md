# Graph Report - data-discovery  (2026-04-22)

## Corpus Check
- Corpus is ~47,195 words - fits in a single context window. You may not need a graph.

## Summary
- 174 nodes · 238 edges · 13 communities detected
- Extraction: 90% EXTRACTED · 10% INFERRED · 0% AMBIGUOUS · INFERRED: 24 edges (avg confidence: 0.85)
- Token cost: 0 input · 0 output

## Community Hubs (Navigation)
- [[_COMMUNITY_Discovery Pipeline & Data Flow Map API|Discovery Pipeline & Data Flow Map API]]
- [[_COMMUNITY_Temporal Scan Orchestration & Risk Scoring|Temporal Scan Orchestration & Risk Scoring]]
- [[_COMMUNITY_Connector Ecosystem & Regulatory Holds|Connector Ecosystem & Regulatory Holds]]
- [[_COMMUNITY_AI Agent Pipeline (PII Detection & Classification)|AI Agent Pipeline (PII Detection & Classification)]]
- [[_COMMUNITY_Data Model & Compliance Schema|Data Model & Compliance Schema]]
- [[_COMMUNITY_Contract Compliance Checking & DPA Generation|Contract Compliance Checking & DPA Generation]]
- [[_COMMUNITY_Data Flow Map Visualization & Risk Flags|Data Flow Map Visualization & Risk Flags]]
- [[_COMMUNITY_Indic Language OCR & Multi-Lingual NLP|Indic Language OCR & Multi-Lingual NLP]]
- [[_COMMUNITY_PII Findings API & Scan Management|PII Findings API & Scan Management]]
- [[_COMMUNITY_Contract Audit API & Database|Contract Audit API & Database]]
- [[_COMMUNITY_AWS Account Air-Gap Isolation|AWS Account Air-Gap Isolation]]
- [[_COMMUNITY_Purpose-Linked Data Flow Billing|Purpose-Linked Data Flow Billing]]
- [[_COMMUNITY_RFC 9457 Error Format|RFC 9457 Error Format]]

## God Nodes (most connected - your core abstractions)
1. `TrustStack Discovery (Module 2)` - 15 edges
2. `Discovery DB Schema (PostgreSQL 16)` - 15 edges
3. `C4 Container Diagram — Discovery Engine` - 13 edges
4. `Discovery Agents (Python/LangGraph)` - 10 edges
5. `DiscoveryScanWorkflow (Temporal.io)` - 8 edges
6. `Discovery DB (PostgreSQL 16, RDS Multi-AZ)` - 7 edges
7. `Stage 6: ComplianceChecker (DPDPA Taxonomy)` - 7 edges
8. `tenants table` - 7 edges
9. `D2C Brand Data Flow Map (Example Output)` - 7 edges
10. `Snapshot API Container (Node.js/TypeScript)` - 6 edges

## Surprising Connections (you probably didn't know these)
- `Flag: MISSING_BREACH_SLA (DPDPA Rule 7)` --semantically_similar_to--> `Flag: UNINTENTIONAL_LEAK`  [INFERRED] [semantically similar]
  data-discovery/diagrams/contract-audit-flow.md → data-discovery/diagrams/data-flow-map.md
- `TrustStack Discovery (Module 2)` --references--> `C4 Container Diagram — Discovery Engine`  [INFERRED]
  data-discovery/README.md → data-discovery/diagrams/system-architecture.md
- `TrustStack Discovery (Module 2)` --references--> `Discovery DB Schema (PostgreSQL 16)`  [INFERRED]
  data-discovery/README.md → data-discovery/diagrams/data-model.md
- `Data Flow Map Service (Python/NetworkX)` --implements--> `Data Flow Map`  [INFERRED]
  data-discovery/diagrams/system-architecture.md → data-discovery/README.md
- `Snapshot API Container (Node.js/TypeScript)` --implements--> `Snapshot API`  [INFERRED]
  data-discovery/diagrams/system-architecture.md → data-discovery/README.md

## Hyperedges (group relationships)
- **Zero-Knowledge PII Protection Pattern (Hash + No Raw Values + Air-Gap)** — sysarch_scanner_agent, datamodel_pii_value_hash, sysarch_discovery_db, readme_truststack_discovery_aws_account, sysarch_truststack_consent_account [INFERRED 0.88]
- **Snapshot API One-Time Token Lifecycle (Issue → Fetch → Consume → 410)** — sysarch_snapshot_api_container, datamodel_snapshot_tokens_table, discoverypipeline_15min_token_expiry, datamodel_snapshot_atomic_claim, readme_erasure_engine [EXTRACTED 1.00]
- **9-Stage NLP Contract Audit Pipeline (FileProcessor → ECDSA Signer)** — contractaudit_file_processor, contractaudit_ocr_agent, contractaudit_language_detector, contractaudit_translation_agent, contractaudit_clause_extractor, contractaudit_compliance_checker, contractaudit_gap_analyzer, contractaudit_dpa_generator, contractaudit_ecdsa_signer [EXTRACTED 1.00]
- **5-Agent Discovery Pipeline: Temporal.io orchestration + LangGraph sub-graphs + SageMaker Mistral 7B** — agentic_connector_agent, agentic_scanner_agent, agentic_classifier_agent, agentic_risk_scorer_agent, agentic_mapper_agent, sysdesign_scan_orchestrator [EXTRACTED 1.00]
- **Snapshot API cross-module boundary: Discovery → Erasure Engine via one-time signed JWT token** — prd_snapshot_api, api_snapshot_endpoints, dm_snapshot_tokens, dm_snapshot_one_time_use_trigger [EXTRACTED 1.00]
- **Dual-Role Prohibition enforcement: separate AWS account + sandbox IAM + read-only connectors** — sysdesign_aws_account_isolation_rationale, sysdesign_readonly_connector_rationale, dm_dual_role_prohibition_rationale, api_snapshot_dual_role_rationale [EXTRACTED 0.95]

## Communities

### Community 0 - "Discovery Pipeline & Data Flow Map API"
Cohesion: 0.1
Nodes (26): MapperAgent (aggregates to Data Flow Map graph), UNINTENTIONAL_LEAK edge semantics (MapperAgent), Connector Management API endpoints, Data Flow Map API endpoints, Snapshot API dual-role prohibition rationale, Snapshot API endpoints (POST /snapshot/request, GET /snapshot/{token}), enforce_discovery_audit_immutability() trigger, connector_configs table (+18 more)

### Community 1 - "Temporal Scan Orchestration & Risk Scoring"
Cohesion: 0.15
Nodes (25): PII Value Hashing (SHA-256 + tenant_salt), DiscoveryScanWorkflow (Temporal.io), Rationale: Hash PII Before Leaving ScannerAgent, Rationale: Parallel Fan-Out Per Connector, Rationale: Temporal.io for Durable Long-Running Scans, Risk Score Formula (weighted multipliers), Artifact Store (AWS S3, AES-256-SSE), C4 Container Diagram — Discovery Engine (+17 more)

### Community 2 - "Connector Ecosystem & Regulatory Holds"
Cohesion: 0.1
Nodes (22): Flag: GST_6YR Regulatory Hold, AI Data Discovery Engine, AWS S3 Connector, CloudWatch / ELK / OpenSearch Connector, Tally Connector, WhatsApp Business API Connector, Zoho CRM/Books/Desk Connector, Consent Vault (Module 3) (+14 more)

### Community 3 - "AI Agent Pipeline (PII Detection & Classification)"
Cohesion: 0.1
Nodes (22): ClassifierAgent (DPDPA category mapping, LangGraph), ConnectorAgent (Temporal activity, I/O only), DPDPA_CATEGORY_MAP (Classifier Agent), ENTITY_TO_DPDPA_CATEGORY static mapping, India-Specific PII Detection Patterns (regex), On-Premise LLM Rationale — Section 16 cross-border prohibition (Agentic), PII_DETECTION_PROMPT (Scanner Agent, SageMaker Mistral 7B), Risk Score Formula (category_weights × multipliers) (+14 more)

### Community 4 - "Data Model & Compliance Schema"
Cohesion: 0.15
Nodes (21): 7-Year Retention Policy (DPDPA Rule 4), connector_configs table, consent_purpose_ref — Opaque Cross-Account Reference, contract_audits table, contract_clauses table, data_flow_edges table, data_flow_nodes table, discovery_audit_log table (append-only) (+13 more)

### Community 5 - "Contract Compliance Checking & DPA Generation"
Cohesion: 0.15
Nodes (13): Compliance Audit Report, Stage 6: ComplianceChecker (DPDPA Taxonomy), Compliant DPA (ECDSA-signed output), Stage 8: DPAGenerator (Mistral 7B), Stage 9: ECDSA Signer (P-256), Flag: CROSS_BORDER_UNAUTHORIZED (DPDPA Section 16), Flag: DATA_SHARING_UNLIMITED (DPDPA Section 6(2)), Flag: MISSING_BREACH_SLA (DPDPA Rule 7) (+5 more)

### Community 6 - "Data Flow Map Visualization & Risk Flags"
Cohesion: 0.18
Nodes (12): D2C Brand Data Flow Map (Example Output), Flag: CROSS_BORDER, Flag: CROSS_MODULE_EXPANSION, Flag: RBI_5YR Regulatory Hold, Flag: REGULATORY_HOLD, Flag: UNINTENTIONAL_LEAK, Safety Gate: SECTION_16_CROSS_BORDER, Safety Gate: SECTION_9_CRITICAL (Minor Data) (+4 more)

### Community 7 - "Indic Language OCR & Multi-Lingual NLP"
Cohesion: 0.24
Nodes (10): 22 Eighth Schedule Languages Support, Stage 5: ClauseExtractor (BERT-multilingual), Stage 1: FileProcessor, Stage 3: LanguageDetector, NLP Contract Audit Pipeline Flow, Stage 2: OCRAgent (Tesseract 5 + Indic Packs), Rationale: BERT-multilingual for Indian Legal Clause Segmentation, Rationale: OCR Confidence Gate at 0.75 (+2 more)

### Community 8 - "PII Findings API & Scan Management"
Cohesion: 0.22
Nodes (10): PII Findings API endpoints, Scan Management API endpoints, Webhook Events (discovery module), dpdpa_category enum, pii_entity_type enum, pii_findings table, regulatory_conflict_type enum, scan_jobs table (+2 more)

### Community 9 - "Contract Audit API & Database"
Cohesion: 0.33
Nodes (9): NLP Contract Audit Pipeline (9-stage), Contract Audit API endpoints, contract_audits table, contract_clauses table, dpa_generations table, DPDPA Compliance Violation Taxonomy (PRD), DPA Generation (PRD), NLP Contract Audit Agent (PRD) (+1 more)

### Community 10 - "AWS Account Air-Gap Isolation"
Cohesion: 1.0
Nodes (2): Network Isolation Diagram (AWS Account Air-Gap), TrustStack-Consent AWS Account

### Community 11 - "Purpose-Linked Data Flow Billing"
Cohesion: 1.0
Nodes (1): Purpose-Linked Data Flow (billing unit)

### Community 12 - "RFC 9457 Error Format"
Cohesion: 1.0
Nodes (1): RFC 9457 Problem Details error format

## Knowledge Gaps
- **68 isolated node(s):** `NLP Contract Audit Agent`, `Consent Vault (Module 3)`, `TrustStack-Discovery AWS Account (ap-south-1)`, `Temporal.io Scan Orchestration`, `Mistral 7B (PII Detection, SageMaker)` (+63 more)
  These have ≤1 connection - possible missing edges or undocumented components.
- **Thin community `AWS Account Air-Gap Isolation`** (2 nodes): `Network Isolation Diagram (AWS Account Air-Gap)`, `TrustStack-Consent AWS Account`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `Purpose-Linked Data Flow Billing`** (1 nodes): `Purpose-Linked Data Flow (billing unit)`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `RFC 9457 Error Format`** (1 nodes): `RFC 9457 Problem Details error format`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.

## Suggested Questions
_Questions this graph is uniquely positioned to answer:_

- **Why does `Discovery DB Schema (PostgreSQL 16)` connect `Data Model & Compliance Schema` to `Temporal Scan Orchestration & Risk Scoring`, `Connector Ecosystem & Regulatory Holds`?**
  _High betweenness centrality (0.119) - this node is a cross-community bridge._
- **Why does `TrustStack Discovery (Module 2)` connect `Connector Ecosystem & Regulatory Holds` to `Temporal Scan Orchestration & Risk Scoring`, `Data Model & Compliance Schema`, `Data Flow Map Visualization & Risk Flags`?**
  _High betweenness centrality (0.103) - this node is a cross-community bridge._
- **Why does `Flag: UNINTENTIONAL_LEAK` connect `Data Flow Map Visualization & Risk Flags` to `Data Model & Compliance Schema`, `Contract Compliance Checking & DPA Generation`?**
  _High betweenness centrality (0.097) - this node is a cross-community bridge._
- **Are the 2 inferred relationships involving `TrustStack Discovery (Module 2)` (e.g. with `C4 Container Diagram — Discovery Engine` and `Discovery DB Schema (PostgreSQL 16)`) actually correct?**
  _`TrustStack Discovery (Module 2)` has 2 INFERRED edges - model-reasoned connections that need verification._
- **Are the 2 inferred relationships involving `Discovery DB Schema (PostgreSQL 16)` (e.g. with `TrustStack Discovery (Module 2)` and `Discovery DB (PostgreSQL 16, RDS Multi-AZ)`) actually correct?**
  _`Discovery DB Schema (PostgreSQL 16)` has 2 INFERRED edges - model-reasoned connections that need verification._
- **What connects `NLP Contract Audit Agent`, `Consent Vault (Module 3)`, `TrustStack-Discovery AWS Account (ap-south-1)` to the rest of the system?**
  _68 weakly-connected nodes found - possible documentation gaps or missing edges._
- **Should `Discovery Pipeline & Data Flow Map API` be split into smaller, more focused modules?**
  _Cohesion score 0.1 - nodes in this community are weakly interconnected._