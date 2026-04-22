# Consent Lifecycle Diagrams — SahmatOS

## 1. Consent State Machine

```mermaid
stateDiagram-v2
    [*] --> NOT_FOUND : No prior consent record

    NOT_FOUND --> PENDING_GUARDIAN : DP is under 18\n(Section 9)
    NOT_FOUND --> ACTIVE : DP grants consent\n(adult / GRANTED action)

    PENDING_GUARDIAN --> ACTIVE : Guardian grants consent\n(Aadhaar eKYC verified)
    PENDING_GUARDIAN --> NOT_FOUND : Guardian denies\nor timeout (7 days)

    ACTIVE --> WITHDRAWN : DP withdraws consent\nor erasure triggered

    ACTIVE --> EXPIRED : Consent expiry date reached\n(time-bounded purposes only)

    WITHDRAWN --> ACTIVE : DP re-grants consent\n(new artifact, new record)
    EXPIRED --> ACTIVE : DP re-grants consent\n(new artifact, new record)

    WITHDRAWN --> [*] : Data erased\n(erasure workflow completes)

    note right of ACTIVE
        DF may process data
        for this purpose
    end note

    note right of WITHDRAWN
        DF must STOP processing
        immediately (webhook fired)
        Erasure workflow triggered
    end note
```

---

## 2. Consent Grant Flow (Happy Path)

```mermaid
sequenceDiagram
    actor DP as Data Principal<br/>(Hindi speaker)
    participant Widget as Consent Widget<br/>(DF's App)
    participant CDN as Widget CDN<br/>(truststack.in)
    participant API as SahmatOS API
    participant LangEngine as Language Engine
    participant KMS as AWS KMS<br/>(ap-south-1)
    participant DB as PostgreSQL<br/>(Consent Ledger)
    participant Cache as Redis Cache
    participant Kafka as Kafka

    DP->>Widget: Opens app feature<br/>requiring consent

    Widget->>CDN: Load widget JS + fonts
    CDN-->>Widget: widget.js + Noto Sans Devanagari<br/>(Hindi font, bundled)

    Widget->>API: GET /v1/widget/consent<br/>?tenantId=X&purposeId=Y<br/>Accept-Language: hi-IN

    API->>LangEngine: detect(Accept-Language: hi-IN)
    LangEngine-->>API: { code: 'hi', direction: 'ltr',<br/>fontFamily: 'Noto Sans Devanagari' }

    API->>Cache: GET lang:notice:Y:hi
    Cache-->>API: HIT { title: "मार्केटिंग संदेश",<br/>description: "Meesho आपसे..." }

    API-->>Widget: Widget content in Hindi<br/>(title, description, rights, actions)

    Widget-->>DP: Renders consent notice in Hindi<br/>✓ मैं सहमत हूँ  ✗ मैं मना करता हूँ

    DP->>Widget: Taps "मैं सहमत हूँ"<br/>(affirmative action — Section 6)

    Widget->>API: POST /v1/consent<br/>{ action: GRANTED, languageCode: hi,<br/>collectionMethod: WIDGET }

    API->>API: Validate: purpose exists,<br/>DP not a minor, notice was shown

    API->>KMS: Sign(artifactPayload)<br/>ECDSA_SHA_256, tenantKeyArn
    KMS-->>API: signature (base64)

    API->>DB: INSERT INTO consent_records<br/>(append-only, DPDPA Rule 4)
    DB-->>API: id committed

    API->>Cache: SET consent:tenant:dpHash:purpose<br/>= ACTIVE (TTL 5min)
    API->>Kafka: emit('consent.granted', artifact)

    API-->>Widget: 201 Created<br/>{ artifactId, signature, languageCode: 'hi' }

    Widget-->>DP: Confirmation in Hindi:<br/>"आपकी सहमति दर्ज हो गई ✓"
```

---

## 3. Consent Withdrawal Flow

```mermaid
sequenceDiagram
    actor DP as Data Principal
    participant Dashboard as Consumer Dashboard
    participant API as SahmatOS API
    participant DB as PostgreSQL
    participant Cache as Redis
    participant Kafka as Kafka
    participant DF as Data Fiduciary<br/>Webhook

    DP->>Dashboard: Opens "My Consents"<br/>in preferred language

    Dashboard->>API: GET /v1/consent<br/>?dpIdentifier=+919876543210<br/>languageCode=hi

    API-->>Dashboard: List of consents<br/>(names + purposes in Hindi)

    DP->>Dashboard: Taps "वापस लो" (Withdraw)<br/>on Meesho Marketing consent

    Dashboard-->>DP: Confirmation dialog in Hindi:<br/>"क्या आप यह सहमति वापस लेना चाहते हैं?"

    DP->>Dashboard: Confirms withdrawal

    Dashboard->>API: POST /v1/consent<br/>{ action: WITHDRAWN, artifactId: ... }

    API->>DB: INSERT consent_record<br/>action=WITHDRAWN<br/>(new append-only record)
    DB-->>API: committed

    API->>Cache: INVALIDATE consent:tenant:dpHash:purpose
    API->>Kafka: emit('consent.withdrawn', ...)

    API-->>Dashboard: 201 Created (withdrawal artifact)
    Dashboard-->>DP: "सहमति वापस ली गई ✓"<br/>Confirmation in Hindi

    Kafka->>DF: Webhook POST /df-webhook<br/>{ event: 'consent.withdrawn',<br/>purposeId: ..., dpIdentifierHash: ... }

    Note over DF: DF must stop all processing<br/>for this purpose immediately.<br/>(DPDPA Section 6 obligation)

    API->>API: Schedule erasure workflow<br/>(CONSENT_WITHDRAWN trigger)
```

---

## 4. Parental Consent Flow (Section 9)

```mermaid
flowchart TD
    A[DP opens consent flow] --> B{Age check}
    B -->|Adult ≥18| C[Standard consent flow]
    B -->|Minor <18\nor not verified| D[Parental Consent Required\nSection 9 DPDPA]

    D --> E[Show guardian consent request\nin guardian's language]
    E --> F[Guardian receives notification:\nWhatsApp / SMS / App]
    F --> G{Guardian action}

    G -->|Approves| H[Aadhaar eKYC:\nVerify guardian is ≥18]
    G -->|Denies| I[Consent denied\nfor this purpose]
    G -->|No response\n7 days| J[Auto-expire pending\nDeny consent]

    H --> K{Aadhaar verification}
    K -->|Verified ≥18| L[Create guardian consent artifact\nParentalConsent = TRUE]
    K -->|Failed / <18| M[Reject:\nGuardian must be adult]

    L --> N[Link to child's consent record\nparental_consent_id FK]
    N --> O[Consent ACTIVE\nfor minor's purpose]

    I --> P[No data processing\npermitted]
    J --> P
    M --> P

    style D fill:#ffe4b5
    style L fill:#90EE90
    style P fill:#ffcccb
```
