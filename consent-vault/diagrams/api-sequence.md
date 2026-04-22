# API Sequence Diagrams — SahmatOS

## 1. Data Fiduciary Integration Flow (Day 1 Onboarding)

```mermaid
sequenceDiagram
    actor df as Data Fiduciary<br/>(CTO / Developer)
    participant ts as TrustStack<br/>Dashboard
    participant api as SahmatOS API
    participant kms as AWS KMS
    participant db as PostgreSQL

    df->>ts: Register as Data Fiduciary
    df->>ts: Declare df_class (STANDARD / ECOMMERCE_GT_2CR / etc.)
    df->>ts: Set default language (e.g. 'hi')
    df->>ts: Provide webhook URL

    ts->>api: POST /internal/tenants (provision tenant)
    api->>kms: CreateKey (AES-256 symmetric)
    kms-->>api: symmetricKeyArn
    api->>kms: CreateKey (ECDSA P-256 asymmetric)
    kms-->>api: signingKeyArn
    api->>db: INSERT tenant with KMS key ARNs
    api-->>ts: tenantId + API keys

    ts-->>df: API Key (ts_live_...) + tenantId<br/>KMS public key (for independent verification)

    df->>api: POST /v1/purposes<br/>{ code: 'marketing_sms',<br/>  dataCategories: ['phone'],<br/>  retentionPeriod: '2 years' }
    api->>db: INSERT consent_purposes
    api-->>df: purposeId

    Note over df,api: Purpose translation required before production use

    df->>ts: Submit purpose for translation review
    ts->>ts: LLM translates to 22 languages
    ts->>ts: Human review queue (native speakers)
    ts->>db: UPDATE translations status='approved'
    ts-->>df: "Purpose approved in 15 languages (7 pending review)"
```

---

## 2. Consent Collection via Widget (Web)

```mermaid
sequenceDiagram
    actor user as User (Tamil speaker)
    participant app as Data Fiduciary's<br/>Web App
    participant cdn as SahmatOS CDN
    participant api as SahmatOS API
    participant redis as Redis Cache
    participant db as PostgreSQL
    participant kms as AWS KMS

    user->>app: Opens checkout page<br/>(Tamil browser locale: ta-IN)

    app->>cdn: Load widget.js<br/>X-API-Key: ts_live_...<br/>data-purpose-id: Y
    cdn-->>app: widget.js + Noto Sans Tamil font

    app->>api: GET /v1/widget/consent<br/>?purposeId=Y&tenantId=X<br/>Accept-Language: ta-IN
    api->>redis: GET lang:notice:Y:ta
    redis-->>api: HIT: Tamil consent notice
    api-->>app: { title: "சம்மதம்", description: "...", direction: "ltr" }

    app-->>user: Renders consent widget in Tamil<br/>✓ நான் சம்மதிக்கிறேன்  ✗ நான் மறுக்கிறேன்

    user->>app: Taps "நான் சம்மதிக்கிறேன்"

    app->>api: POST /v1/consent<br/>{ action: GRANTED, languageCode: ta,<br/>  collectionMethod: WIDGET }
    api->>kms: Sign(artifact) ECDSA_SHA_256
    kms-->>api: signature
    api->>db: INSERT consent_records (append-only)
    api->>redis: SET consent:X:dpHash:Y = ACTIVE
    api-->>app: 201 { artifactId, signature, languageCode: 'ta' }

    app-->>user: "ஒப்புதல் பதிவு செய்யப்பட்டது ✓"<br/>(Confirmation in Tamil)
```

---

## 3. Real-Time Consent Check (High-Frequency API)

```mermaid
sequenceDiagram
    participant df as DF Backend<br/>(Before every SMS/email send)
    participant api as SahmatOS API
    participant redis as Redis Cache
    participant db as PostgreSQL<br/>Read Replica

    df->>api: GET /v1/consent/check<br/>?tenantId=X&dpIdentifier=+91...&purposeId=Y
    api->>redis: GET consent:X:hmac(+91...):Y

    alt Cache HIT (p50: ~30ms)
        redis-->>api: { status: ACTIVE, grantedAt: ... }
        api-->>df: 200 { status: ACTIVE }
        df->>df: ✅ Proceed with SMS send
    else Cache MISS (p99: ~200ms)
        redis-->>api: null
        api->>db: SELECT latest consent record<br/>WHERE dp_hash = $1 AND purpose = $2<br/>ORDER BY created_at DESC LIMIT 1
        db-->>api: { action: WITHDRAWN }
        api->>redis: SET consent:X:hmac:Y = WITHDRAWN (TTL 5min)
        api-->>df: 200 { status: WITHDRAWN }
        df->>df: ❌ Do NOT send SMS — consent withdrawn
    end
```

---

## 4. Erasure Request Flow

```mermaid
sequenceDiagram
    actor dp as Data Principal
    participant dashboard as Consumer Dashboard
    participant api as SahmatOS API
    participant temporal as Temporal.io
    participant db as PostgreSQL
    participant notif as Notification Service<br/>(WhatsApp/SMS)
    participant df_system as Data Fiduciary's System
    participant kafka as Kafka

    dp->>dashboard: "मेरा सारा डेटा मिटाएं"<br/>(Delete all my data — Hindi)

    dashboard->>api: POST /v1/erasure<br/>{ triggeredBy: DP_REQUEST,\n  notificationLanguage: hi }
    api->>db: INSERT erasure_records\n(status: SCHEDULED)
    api->>temporal: StartWorkflow(ErasureWorkflow, ...)
    temporal-->>api: workflowId
    api-->>dashboard: 202 { erasureId, executeAt }
    dashboard-->>dp: "मिटाने का अनुरोध दर्ज हुआ।\n48 घंटे में आपको सूचित किया जाएगा।"

    Note over temporal: For STANDARD class DFs (most SMEs):\nexecuteAt = NOW (immediate).\nFor ECOMMERCE_GT_2CR etc.: up to 3 years.

    temporal->>db: Update notification_due_at reached
    temporal->>notif: Send 48hr pre-deletion notification
    notif->>dp: WhatsApp in Hindi:\n"आपका डेटा 48 घंटे में मिटाया जाएगा।\nरोकने के लिए यहाँ टैप करें।"

    dp-->>temporal: (Does not cancel — proceeds)

    temporal->>temporal: sleep(48hr)
    temporal->>df_system: POST /erasure-webhook\n{ dpIdentifierHash, purposes }
    df_system-->>temporal: { deleted: true }
    temporal->>db: UPDATE erasure_records status=IN_PROGRESS
    temporal->>temporal: Hard delete from all connected systems
    temporal->>temporal: Generate ECDSA-signed deletion certificate
    temporal->>db: UPDATE erasure_records\nstatus=COMPLETED, deletion_certificate=...
    temporal->>kafka: emit('erasure.executed', certificate)

    kafka->>notif: Notify DP of completion
    notif->>dp: WhatsApp:\n"आपका डेटा मिटा दिया गया है। प्रमाण पत्र: <link>"
```

---

## 5. Breach Notification Workflow (Rule 7)

```mermaid
sequenceDiagram
    participant df as Data Fiduciary
    participant api as SahmatOS API
    participant temporal as Temporal.io<br/>(Breach Workflow)
    participant certin as CERT-In Portal
    participant dpbi as DPBI Portal
    participant notif as Notification Service
    actor dps as Affected Data Principals<br/>(multiple languages)

    Note over df,temporal: T+0: Breach detected

    df->>api: POST /v1/breach\n{ detectedAt, breachType,\n  affectedCount: 15000,\n  categories: ['phone','email'] }
    api->>temporal: StartWorkflow(BreachNotificationWorkflow)
    api-->>df: 202 { breachId, certInDeadline: T+6hr, dpbiDeadline: T+72hr }

    Note over temporal: T+0 to T+6hr: CERT-In window

    temporal->>temporal: Generate CERT-In initial report
    temporal->>certin: Submit initial incident report
    certin-->>temporal: reportId: CERT-IN-2026-04-...

    temporal->>db: Record cert_in_submitted_at

    Note over temporal: Parallel: Notify Data Principals

    temporal->>notif: Notify affected DPs in their languages
    notif->>dps: Hindi: "आपका डेटा सुरक्षा उल्लंघन हुआ है..."
    notif->>dps: Tamil: "உங்கள் தரவு மீறல் ஏற்பட்டது..."
    notif->>dps: Bengali: "আপনার ডেটা লঙ্ঘন হয়েছে..."
    Note over notif,dps: Notification in each DP's registered language

    Note over temporal: T+6hr to T+72hr: DPBI window

    temporal->>temporal: Generate comprehensive DPBI report\n(forensic analysis, affected systems,\nremediation steps)
    temporal->>dpbi: Submit 72hr detailed report
    dpbi-->>temporal: reportId: DPBI-2026-04-...

    temporal->>db: Record dpbi_submitted_at, status=DPBI_SUBMITTED

    Note over temporal: Timeline compliance check

    temporal->>temporal: Assert: cert_in submitted within 6hr ✓
    temporal->>temporal: Assert: dpbi submitted within 72hr ✓
    temporal->>temporal: Emit compliance event to audit log
```
