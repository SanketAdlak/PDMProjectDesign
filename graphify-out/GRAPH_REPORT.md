# Graph Report - .  (2026-04-22)

## Corpus Check
- Corpus is ~3,343 words - fits in a single context window. You may not need a graph.

## Summary
- 61 nodes · 71 edges · 11 communities detected
- Extraction: 79% EXTRACTED · 21% INFERRED · 0% AMBIGUOUS · INFERRED: 15 edges (avg confidence: 0.81)
- Token cost: 0 input · 0 output

## Community Hubs (Navigation)
- [[_COMMUNITY_Brand, Design & Conventions|Brand, Design & Conventions]]
- [[_COMMUNITY_Consent Vault & Erasure|Consent Vault & Erasure]]
- [[_COMMUNITY_Graphify & Dev Tooling|Graphify & Dev Tooling]]
- [[_COMMUNITY_Core Consent Regulation|Core Consent Regulation]]
- [[_COMMUNITY_Security & Compliance Infrastructure|Security & Compliance Infrastructure]]
- [[_COMMUNITY_Data Discovery & Dual-Role|Data Discovery & Dual-Role]]
- [[_COMMUNITY_Indian Cloud Data Residency|Indian Cloud Data Residency]]
- [[_COMMUNITY_Vernacular Awareness Module|Vernacular Awareness Module]]
- [[_COMMUNITY_Transport Security|Transport Security]]
- [[_COMMUNITY_Frontend Stack|Frontend Stack]]
- [[_COMMUNITY_Integration Bus|Integration Bus]]

## God Nodes (most connected - your core abstractions)
1. `TrustStack Project Identity` - 19 edges
2. `DPDPA 2023 Regulation` - 11 edges
3. `Consent Vault (SahmatOS, ECDSA-Signed, 22-Language)` - 8 edges
4. `Breach Notification Rule 7 (72hr DPBI + 6hr CERT-In)` - 4 edges
5. `Cross-Border Transfer Section 16 (Indian Cloud Regions)` - 4 edges
6. `Module 1 - Vernacular Awareness (Free Tier)` - 4 edges
7. `Module 2 - Assessment and Discovery (Revenue Driver)` - 4 edges
8. `Discovery Engine Sandbox (Dual-Role Conflict Prevention)` - 4 edges
9. `Module 3 - Implementation and Automation (The Shield)` - 4 edges
10. `Design Principle: Vernacular-First (Hindi/Tamil First)` - 4 edges

## Surprising Connections (you probably didn't know these)
- `Graphify Section (GEMINI.md)` --semantically_similar_to--> `Graphify Section (CLAUDE.md)`  [INFERRED] [semantically similar]
  GEMINI.md → CLAUDE.md
- `Graphify Section (AGENTS.md)` --semantically_similar_to--> `Graphify Section (CLAUDE.md)`  [INFERRED] [semantically similar]
  AGENTS.md → CLAUDE.md
- `God Nodes List (GRAPH_REPORT.md)` --references--> `Graphify Section (CLAUDE.md)`  [EXTRACTED]
  graphify-out/GRAPH_REPORT.md → CLAUDE.md
- `vis-network Graph Visualization (graph.html)` --references--> `Graph Report Summary (11 nodes, 12 edges, 4 communities)`  [INFERRED]
  graphify-out/graph.html → graphify-out/GRAPH_REPORT.md

## Hyperedges (group relationships)
- **DPDPA Compliance Triad: Consent Vault + Erasure Engine + Regulatory War Room implement DPDPA 2023** — claude_consent_vault, claude_erasure_engine, claude_regulatory_war_room, claude_dpdpa_2023 [EXTRACTED 0.95]
- **Indian Cloud Data Residency: AWS Mumbai + GCP Mumbai enforce Section 16 Cross-Border Rule** — claude_aws_mumbai, claude_gcp_mumbai, claude_cross_border_section16 [EXTRACTED 0.95]
- **Consent Engine Tech Stack: Kafka + PostgreSQL + ECDSA P-256 + temporal.io power Consent Vault** — claude_kafka, claude_postgresql, claude_ecdsa_p256, claude_temporal_io, claude_consent_vault [INFERRED 0.85]

## Communities

### Community 0 - "Brand, Design & Conventions"
Cohesion: 0.15
Nodes (14): TrustStack Brand (Build Trust. Ship Faster.), Code Conventions (TypeScript, ESM, OpenAPI 3.1), Design Principle: 3-Tap Compliance, Design Principle: Mobile-First (5.5-inch Screen, 4G), Design Principle: Vernacular-First (Hindi/Tamil First), Razorpay Integration (Payment/KYC Consent Tracking), Tally Integration (Financial PII Discovery), WhatsApp Business Integration (Regional-Language Consent) (+6 more)

### Community 1 - "Consent Vault & Erasure"
Cohesion: 0.22
Nodes (11): SahmatOS Brand (Consent Vault Product), Children Consent Section 9 (Parental Consent Under-18), Consent Vault (SahmatOS, ECDSA-Signed, 22-Language), ECDSA P-256 Signatures, Erasure Engine (Class-Specific Timelines, Deletion Certs), Class-Specific Erasure Rules (Third Schedule), Kafka (Event-Driven Consent Engine), Module 3 - Implementation and Automation (The Shield) (+3 more)

### Community 2 - "Graphify & Dev Tooling"
Cohesion: 0.29
Nodes (7): Graphify Section (AGENTS.md), Graphify Section (CLAUDE.md), Graphify Section (GEMINI.md), vis-network Graph Visualization (graph.html), Communities: Agent Instruction Files, Graph Report, Wiki, Update Command, God Nodes List (GRAPH_REPORT.md), Graph Report Summary (11 nodes, 12 edges, 4 communities)

### Community 3 - "Core Consent Regulation"
Cohesion: 0.33
Nodes (6): Consent as Only Basis for Advertising/Analytics, Consent Requirements (Free, Specific, Informed, Unconditional), DPDPA 2023 Regulation, Enforcement Timeline (Nov 2025 to May 2027), Notice Requirements (Section 5, Rule 3), Registered Consent Manager (Section 2(g) DPDPA)

### Community 4 - "Security & Compliance Infrastructure"
Cohesion: 0.33
Nodes (6): AES-256 Encryption at Rest, Breach Notification Rule 7 (72hr DPBI + 6hr CERT-In), CERT-In (Indian Computer Emergency Response Team), Consent Manager Registration Rule 4 (Nov 2026), DPBI (Data Protection Board of India), DPDP Rules 2025

### Community 5 - "Data Discovery & Dual-Role"
Cohesion: 0.33
Nodes (6): AI Data Discovery Engine (PII Crawl, Data Flow Map), Discovery Engine Sandbox (Dual-Role Conflict Prevention), Dual-Role Prohibition (Consent Manager Cannot Be Data Processor), Module 2 - Assessment and Discovery (Revenue Driver), Multilingual Contract Audit (NLP, 22 Languages), Rationale: Discovery Engine Sandboxed to Prevent Dual-Role Conflict

### Community 6 - "Indian Cloud Data Residency"
Cohesion: 0.5
Nodes (4): AWS Mumbai Cloud Region, Cross-Border Transfer Section 16 (Indian Cloud Regions), GCP Mumbai Cloud Region, Rationale: Use Indian Cloud Regions for Consent Records

### Community 7 - "Vernacular Awareness Module"
Cohesion: 0.5
Nodes (4): PrivacyDost Brand (Tier 3 Vernacular Sub-Brand), Jargon Slayer (AI DPDPA Translation, 22 Languages), Module 1 - Vernacular Awareness (Free Tier), 3-Min Trust Check (Free PII Scanner)

### Community 8 - "Transport Security"
Cohesion: 1.0
Nodes (1): TLS 1.3 In Transit

### Community 9 - "Frontend Stack"
Cohesion: 1.0
Nodes (1): React and React Native Frontend (Mobile-First)

### Community 10 - "Integration Bus"
Cohesion: 1.0
Nodes (1): GraphQL Integration Bus

## Knowledge Gaps
- **31 isolated node(s):** `Graphify Section (GEMINI.md)`, `Graphify Section (AGENTS.md)`, `Consent as Only Basis for Advertising/Analytics`, `Notice Requirements (Section 5, Rule 3)`, `Enforcement Timeline (Nov 2025 to May 2027)` (+26 more)
  These have ≤1 connection - possible missing edges or undocumented components.
- **Thin community `Transport Security`** (1 nodes): `TLS 1.3 In Transit`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `Frontend Stack`** (1 nodes): `React and React Native Frontend (Mobile-First)`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `Integration Bus`** (1 nodes): `GraphQL Integration Bus`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.

## Suggested Questions
_Questions this graph is uniquely positioned to answer:_

- **Why does `TrustStack Project Identity` connect `Brand, Design & Conventions` to `Consent Vault & Erasure`, `Graphify & Dev Tooling`, `Core Consent Regulation`, `Data Discovery & Dual-Role`, `Indian Cloud Data Residency`, `Vernacular Awareness Module`?**
  _High betweenness centrality (0.640) - this node is a cross-community bridge._
- **Why does `DPDPA 2023 Regulation` connect `Core Consent Regulation` to `Brand, Design & Conventions`, `Consent Vault & Erasure`, `Security & Compliance Infrastructure`, `Data Discovery & Dual-Role`, `Indian Cloud Data Residency`?**
  _High betweenness centrality (0.311) - this node is a cross-community bridge._
- **Why does `Graphify Section (CLAUDE.md)` connect `Graphify & Dev Tooling` to `Brand, Design & Conventions`?**
  _High betweenness centrality (0.178) - this node is a cross-community bridge._
- **Are the 4 inferred relationships involving `Consent Vault (SahmatOS, ECDSA-Signed, 22-Language)` (e.g. with `Kafka (Event-Driven Consent Engine)` and `PostgreSQL (Append-Only Immutable Audit Tables)`) actually correct?**
  _`Consent Vault (SahmatOS, ECDSA-Signed, 22-Language)` has 4 INFERRED edges - model-reasoned connections that need verification._
- **What connects `Graphify Section (GEMINI.md)`, `Graphify Section (AGENTS.md)`, `Consent as Only Basis for Advertising/Analytics` to the rest of the system?**
  _31 weakly-connected nodes found - possible documentation gaps or missing edges._