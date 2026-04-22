# Module 2 — Data Discovery: Pipeline Flows

This document contains two diagrams:

1. **Discovery Agent Pipeline** — the complete flowchart from scan trigger through PII detection, classification, risk scoring, and Data Flow Map assembly, including all regulatory safety gates.
2. **Snapshot API Sequence** — the precise interaction between the Erasure Engine (Module 3) and the Snapshot API when requesting a Data Principal's data location snapshot for a deletion run.

---

## Discovery Agent Pipeline

The pipeline is driven by Temporal.io workflows. The top-level `DiscoveryScanWorkflow` fans out one child workflow per configured connector in parallel, then fans results back in through the `MapperAgent`. This design means a Zoho connector timeout does not block a Razorpay scan — each connector's failure is isolated and retried independently.

All PII values are **hashed before leaving the ScannerAgent** — subsequent stages (ClassifierAgent, RiskScorerAgent, MapperAgent) work only on hashed values and metadata. The LLM Gateway never receives raw personal data.

```mermaid
flowchart TD
    A([DPO / CTO triggers scan\nvia Discovery API]) --> B[API creates scan_job record\nstatus = QUEUED]
    B --> C[API calls Temporal:\nStartWorkflow DiscoveryScanWorkflow\nwith job_id + connector_ids]

    C --> D{Temporal Orchestrator\nDiscoveryScanWorkflow}

    D -->|"Fan-out: parallel child workflows\none per connector"| E1
    D -->|"Fan-out: parallel child workflows\none per connector"| E2
    D -->|"Fan-out: parallel child workflows\none per connector"| E3
    D -->|"Fan-out: parallel child workflows\none per connector"| E4
    D -->|"Fan-out: parallel child workflows\none per connector"| E5

    subgraph zoho_flow["ConnectorWorkflow: Zoho CRM"]
        E1[ConnectorAgent\nZoho OAuth2 auth\nEnumerate: Contacts, Deals, Tickets\nStream batches of 500 records]
        E1 --> F1[ScannerAgent\nChunk records into 50-token windows\nLLM Gateway: PII detection\nHash all PII values SHA-256\nEmit PIIFindings]
        F1 --> G1[ClassifierAgent\nMap entity types to DPDPA categories\nCONTACT / BEHAVIORAL / FINANCIAL\nDetect regulatory conflicts]
        G1 --> H1[RiskScorerAgent\nCompute weighted score]
    end

    subgraph razorpay_flow["ConnectorWorkflow: Razorpay"]
        E2[ConnectorAgent\nAPI key auth read-scope\nEnumerate: Payments, Customers, KYC\nStream batches of 200 records]
        E2 --> F2[ScannerAgent\nChunk + LLM PII detect\nHash values\nEmit PIIFindings]
        F2 --> G2[ClassifierAgent\nMap: FINANCIAL + PERSONAL_IDENTITY\nFlag RBI_5YR regulatory conflict]
        G2 --> H2[RiskScorerAgent\nCompute weighted score]
    end

    subgraph tally_flow["ConnectorWorkflow: Tally"]
        E3[ConnectorAgent\nTally XML Gateway\nEnumerate: Ledgers, Invoices, Parties\nStream batches of 300 records]
        E3 --> F3[ScannerAgent\nChunk + LLM PII detect\nHash values\nEmit PIIFindings]
        F3 --> G3[ClassifierAgent\nMap: FINANCIAL\nFlag GST_6YR regulatory conflict]
        G3 --> H3[RiskScorerAgent\nCompute weighted score]
    end

    subgraph s3_flow["ConnectorWorkflow: AWS S3 / Logs"]
        E4[ConnectorAgent\nCross-account IAM read role\nEnumerate: Buckets, Objects, CloudWatch log groups\nStream object metadata + sampled content]
        E4 --> F4[ScannerAgent\nChunk + LLM PII detect\nHash values\nEmit PIIFindings]
        F4 --> G4[ClassifierAgent\nMap: ALL_CATEGORIES\nFlag UNINTENTIONAL_LEAK if PII in backups/logs]
        G4 --> H4[RiskScorerAgent\nCompute weighted score]
    end

    subgraph wa_flow["ConnectorWorkflow: WhatsApp Business"]
        E5[ConnectorAgent\nWhatsApp Business API auth\nEnumerate: Templates, Message metadata\nStream contact + delivery metadata]
        E5 --> F5[ScannerAgent\nChunk + LLM PII detect\nHash values\nEmit PIIFindings]
        F5 --> G5[ClassifierAgent\nMap: CONTACT\nCheck consent on file]
        G5 --> H5[RiskScorerAgent\nCompute weighted score]
    end

    %% Risk scoring formula
    H1 & H2 & H3 & H4 & H5 --> RISK_NOTE

    RISK_NOTE["Risk Score Formula per finding:\nbase_score\n× 2.0 if cross_border_flag\n× 1.5 if no_consent_on_file\n× 1.8 if regulatory_conflict\n× 1.3 if FINANCIAL category\n× 2.5 if MINOR_DATA flag\nCapped at 10.0"]

    RISK_NOTE --> SAFETY

    subgraph SAFETY["Safety Gate Checks — ClassifierAgent"]
        SG1{PII found in\nlogs or backups?}
        SG1 -->|Yes| SG1Y[Flag: UNINTENTIONAL_LEAK\nSet risk_level = CRITICAL\nAlert DPO immediately]
        SG1 -->|No| SG2

        SG2{Financial data\nin Razorpay?}
        SG2 -->|Yes| SG2Y[Flag: RBI_5YR\nregulatory_conflict = TRUE\nCannot erase until 5yr hold expires]
        SG2 -->|No| SG3

        SG3{Minor / child\ndata detected?}
        SG3 -->|Yes| SG3Y[Flag: SECTION_9_CRITICAL\nrisk_level = CRITICAL\nBlock scan — notify DPO + legal immediately]
        SG3 -->|No| SG4

        SG4{Data stored on\nnon-Indian servers?}
        SG4 -->|Yes| SG4Y[Flag: SECTION_16_CROSS_BORDER\nrisk_level = HIGH\nDPBI access must be confirmed]
        SG4 -->|No| SG5[Continue — standard risk level]
    end

    SG1Y & SG2Y & SG3Y & SG4Y & SG5 --> MAPPER

    MAPPER[MapperAgent\nFan-in all connector results\nBuild or update Data Flow Map graph\nNodes: systems | Edges: data category flows\nAttach risk scores + flags to edges]

    MAPPER --> PERSIST[Persist graph to\nData Flow Map Service\nWrite pii_findings to Discovery DB]

    PERSIST --> REPORT[Report Generator\nCompose visual Data Flow Map\nRisk summary: system × risk × top finding\nRemediation recommendations]

    REPORT --> NOTIF{Max risk score\nabove tenant threshold?}
    NOTIF -->|Yes| NOTIF_Y[Send alert to DPO\nEmail + in-app notification\nInclude top 5 critical findings]
    NOTIF -->|No| NOTIF_N[Update scan_job status = COMPLETE\nNo alert sent]

    NOTIF_Y & NOTIF_N --> SNAP_PRE[Pre-compute Snapshot index\nDP-identifier → [node_ids] mapping\nStored in snapshot_tokens table\nReady for Erasure Engine on demand]

    SNAP_PRE --> DONE([Scan complete\nData Flow Map available\nin Discovery API dashboard])

    %% Styling
    classDef critical fill:#cc2200,color:#fff,stroke:#991100
    classDef high fill:#dd6600,color:#fff,stroke:#aa4400
    classDef info fill:#1a5276,color:#fff,stroke:#154360
    classDef gate fill:#5b2c8b,color:#fff,stroke:#4a235a
    classDef success fill:#1e8449,color:#fff,stroke:#196f3d

    class SG1Y,SG3Y critical
    class SG2Y,SG4Y high
    class SAFETY gate
    class DONE success
    class RISK_NOTE info
```

---

## Snapshot API — Erasure Engine Integration Sequence

When the Erasure Engine (Module 3, running in `TrustStack-Consent` account) needs to delete a Data Principal's data, it first needs to know which downstream vendor systems hold that person's data. It cannot query the Discovery DB directly — the accounts are air-gapped. Instead, it calls the Snapshot API with a one-time signed token.

The token is valid for **15 minutes** and is **single-use**. Once consumed, any replay attempt returns HTTP 410 Gone. This prevents the Erasure Engine from building a persistent cache of data locations — every erasure run requires a fresh authorization.

```mermaid
sequenceDiagram
    actor ErasureMgr as Erasure Engine<br/>(TrustStack-Consent)
    participant DiscovAPI as Discovery API<br/>(TrustStack-Discovery)
    participant SnapAPI as Snapshot API<br/>(public ALB, TrustStack-Discovery)
    participant DB as Discovery DB<br/>(PostgreSQL)
    participant MapSvc as Data Flow Map Service

    Note over ErasureMgr,MapSvc: Pre-condition: Erasure Engine holds a long-lived API key<br/>issued during tenant onboarding. NOT a personal data key.

    ErasureMgr->>DiscovAPI: POST /v1/snapshot/request<br/>{ dp_identifier_hash: "sha256:abc...", tenant_id: "t_123" }<br/>Authorization: Bearer <long-lived-api-key>

    DiscovAPI->>DB: Validate tenant API key + check rate limit
    DB-->>DiscovAPI: OK — tenant t_123, plan: Enterprise

    DiscovAPI->>DB: INSERT snapshot_tokens<br/>{ token_id, tenant_id, dp_identifier_hash,<br/>  expires_at: NOW() + 15min, consumed: false }
    DB-->>DiscovAPI: token_id = "snap_xyz789"

    DiscovAPI->>DiscovAPI: Sign JWT<br/>{ sub: dp_identifier_hash, jti: snap_xyz789,<br/>  exp: NOW()+15min, iss: discovery-api }<br/>Using tenant KMS key (ECDSA P-256)

    DiscovAPI-->>ErasureMgr: 201 Created<br/>{ snapshot_token: "eyJ...<signed-jwt>...",<br/>  expires_at: "2026-04-22T10:15:00Z",<br/>  fetch_url: "https://snapshot.truststack.in/v1/fetch" }

    Note over ErasureMgr,SnapAPI: Erasure Engine now calls Snapshot API directly

    ErasureMgr->>SnapAPI: GET /v1/fetch<br/>Authorization: Bearer <signed-jwt>

    SnapAPI->>SnapAPI: Verify JWT signature (ECDSA P-256)<br/>Check exp — not expired<br/>Extract jti = snap_xyz789

    SnapAPI->>DB: SELECT consumed, expires_at FROM snapshot_tokens<br/>WHERE token_id = snap_xyz789

    DB-->>SnapAPI: { consumed: false, expires_at: "2026-04-22T10:15:00Z" }

    SnapAPI->>DB: UPDATE snapshot_tokens SET consumed = true,<br/>consumed_at = NOW()<br/>WHERE token_id = snap_xyz789 AND consumed = false<br/>(atomic CAS update)

    DB-->>SnapAPI: 1 row updated (token successfully claimed)

    SnapAPI->>MapSvc: GET /internal/graph/dp-snapshot<br/>{ dp_identifier_hash: "sha256:abc...", tenant_id: "t_123" }

    MapSvc->>DB: SELECT nodes, edges WHERE dp_identifier_hash matches<br/>traverse graph to find all reachable systems

    DB-->>MapSvc: [ node: Zoho CRM (CONTACT, BEHAVIORAL),<br/>  node: Razorpay (FINANCIAL — RBI_5YR hold),<br/>  node: AWS S3 Backup (ALL_CATEGORIES — UNINTENTIONAL_LEAK),<br/>  edges with data_categories + regulatory_conflicts ]

    MapSvc-->>SnapAPI: DataFlowSnapshot { nodes: [...], edges: [...],<br/>  generated_at: "2026-04-22T10:01:00Z" }

    SnapAPI-->>ErasureMgr: 200 OK<br/>{ snapshot: { nodes, edges, generated_at },<br/>  token_consumed: true,<br/>  note: "This token is now invalid. Request a new token for retry." }

    Note over ErasureMgr,MapSvc: Erasure Engine uses snapshot to target deletion commands<br/>at each vendor system. RBI_5YR hold nodes are flagged — Erasure<br/>Engine will skip deletion but record the regulatory hold.

    ErasureMgr->>SnapAPI: GET /v1/fetch (replay attempt with same token)
    SnapAPI->>DB: SELECT consumed FROM snapshot_tokens WHERE token_id = snap_xyz789
    DB-->>SnapAPI: { consumed: true }
    SnapAPI-->>ErasureMgr: 410 Gone<br/>{ error: "SNAPSHOT_TOKEN_CONSUMED",<br/>  message: "Token already used. Request a new snapshot token." }
```

---

## Pipeline Design Decisions

| Decision | Rationale |
|---|---|
| Temporal.io for orchestration | Long-running scans (hours for large Zoho CRM tenants) need durable execution with automatic retries; Temporal persists workflow state across worker restarts |
| Parallel fan-out per connector | A slow or failing connector (e.g., Tally XML Gateway timeout) does not block other connectors; each child workflow has independent retry budget |
| Hash PII before leaving ScannerAgent | Discovery DB contains zero personal data values; satisfies Zero-Knowledge principle and limits blast radius if DB is compromised |
| Section 9 (minor data) blocks scan | Child data is the most sensitive category under DPDPA; automatic block-and-escalate prevents the platform from silently continuing a scan where unlawful processing may be occurring |
| One-time snapshot tokens | Prevents Erasure Engine from caching stale data locations; forces a fresh snapshot per erasure run, ensuring the deletion target list reflects the current Data Flow Map |
| 15-minute token expiry | Short enough to limit window for token interception; long enough for Erasure Engine to receive the token, make the fetch call, and begin issuing deletion commands |
