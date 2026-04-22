# System Architecture Diagram — SahmatOS

```mermaid
C4Context
    title SahmatOS — System Context (DPDPA Consent Manager)

    Person(dp, "Data Principal", "Indian user (Hindi/Tamil/etc.). Grants or withdraws consent.")
    Person(df_dev, "Data Fiduciary", "Business embedding SahmatOS (Meesho, Tata, PharmEasy...)")
    Person(dpbi, "DPBI", "Data Protection Board of India. Receives breach reports, audits.")
    Person(certin, "CERT-In", "Receives 6hr breach incident reports.")

    System(sahmat, "SahmatOS (Consent Vault)", "Registered Consent Manager. Collects, stores, and proves consent in 22 Indian languages.")

    System_Ext(whatsapp, "WhatsApp Business API", "Consent via chat for Tier 2-3 users")
    System_Ext(aadhaar, "UIDAI Aadhaar eKYC", "Age verification for parental consent")
    System_Ext(df_system, "Data Fiduciary's App/Backend", "Embeds consent widget, checks consent status")
    System_Ext(kms, "AWS KMS (ap-south-1)", "ECDSA signing keys, data encryption keys")

    Rel(dp, sahmat, "Grants/withdraws consent", "Widget / WhatsApp / Consumer Dashboard")
    Rel(df_dev, sahmat, "Collects consent, checks status, triggers erasure", "REST API / SDK")
    Rel(sahmat, dpbi, "Sends breach notification reports", "DPBI API (72hr)")
    Rel(sahmat, certin, "Sends incident reports", "CERT-In portal (6hr)")
    Rel(sahmat, whatsapp, "Sends consent messages in regional languages")
    Rel(sahmat, aadhaar, "Verifies guardian age for parental consent (Section 9)")
    Rel(sahmat, kms, "Signs consent artifacts, encrypts PII")
    Rel(df_system, sahmat, "Embeds widget, calls consent check API")
```

---

```mermaid
C4Container
    title SahmatOS — Container Architecture

    Person(dp, "Data Principal")
    Person(df, "Data Fiduciary")

    System_Boundary(sahmat_boundary, "SahmatOS (AWS ap-south-1)") {

        Container(alb, "Application Load Balancer", "AWS ALB", "TLS 1.3 termination. Routes to API servers.")
        Container(api, "API Server", "Node.js 20 / TypeScript ESM", "Express + OpenAPI 3.1. Consent, Erasure, Breach, Widget, Audit endpoints.")
        Container(widget_cdn, "Widget CDN", "CloudFront (ap-south-1)", "JavaScript SDK, fonts for 22 scripts, consent widget assets.")
        Container(lang_engine, "Language Engine", "TypeScript module", "Language detection, translation retrieval, font manifest, RTL detection.")
        Container(temporal_workers, "Temporal Workers", "Node.js", "Erasure workflows (up to 3yr). Breach notification workflows (6hr/72hr deadlines).")

        ContainerDb(postgres, "Consent Ledger", "PostgreSQL 16 (RDS Multi-AZ)", "Append-only consent_records. 7-year retention. ECDSA-signed. AES-256 encrypted.")
        ContainerDb(redis, "Cache", "Redis (ElastiCache Cluster)", "Consent status cache (5min TTL). Purpose/translation cache (24hr TTL).")
        ContainerDb(kafka, "Event Bus", "Apache Kafka (MSK)", "consent.granted, consent.withdrawn, erasure.scheduled, erasure.executed, breach.*")
        ContainerDb(temporal_db, "Workflow State", "Temporal.io Server (self-hosted)", "Durable workflow state for erasure and breach orchestration.")
        ContainerDb(opensearch, "Audit Search", "Amazon OpenSearch", "Searchable audit log for DPBI compliance queries.")
    }

    System_Ext(kms_ext, "AWS KMS", "Per-tenant ECDSA P-256 signing + AES-256 encryption keys")

    Rel(dp, alb, "HTTPS", "Widget embed / Consumer dashboard")
    Rel(df, alb, "HTTPS + X-API-Key", "Backend API calls")
    Rel(alb, api, "HTTP internal")
    Rel(api, lang_engine, "In-process call", "Language detection + translation")
    Rel(api, postgres, "TCP 5432", "Consent record writes + reads")
    Rel(api, redis, "TCP 6379", "Consent status cache reads/writes")
    Rel(api, kafka, "TCP 9092", "Emit consent events")
    Rel(api, kms_ext, "HTTPS", "Sign consent artifacts")
    Rel(kafka, temporal_workers, "Consume events")
    Rel(temporal_workers, temporal_db, "Workflow state persistence")
    Rel(temporal_workers, postgres, "Read erasure records, write deletion certs")
    Rel(api, opensearch, "HTTPS", "Write audit events")
    Rel(dp, widget_cdn, "HTTPS", "Load widget JS + fonts")
```

---

```mermaid
graph TB
    subgraph dual_role["Dual-Role Isolation (DPDPA Section 2(g))"]
        subgraph consent_account["AWS Account: TrustStack-Consent"]
            sahmat["SahmatOS\n(Consent Vault)"]
            consent_db[("Consent Ledger\nPostgreSQL")]
            sahmat --> consent_db
        end

        subgraph discovery_account["AWS Account: TrustStack-Discovery"]
            discovery["Module 2\n(AI Discovery Engine)"]
            discovery_db[("Discovery Data\nSeparate DB)"]
            discovery --> discovery_db
        end

        consent_account -. "NO VPC PEERING\nNO SHARED RESOURCES\nNO IAM CROSS-ACCOUNT" .- discovery_account
    end

    note["DPDPA: Consent Manager\nCANNOT be Data Processor\nfor same Data Principal"]
    style note fill:#ff9999,stroke:#cc0000
```
