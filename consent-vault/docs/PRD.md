# Product Requirements Document — SahmatOS (Consent Vault)

**Version:** 1.0  
**Date:** 2026-04-22  
**Status:** Draft  
**Owner:** TrustStack Product Team  

---

## 1. Executive Summary

SahmatOS is the consent infrastructure layer of TrustStack. It enables Indian businesses to collect, store, manage, and demonstrate lawful consent from their users — in the user's native language — in compliance with the Digital Personal Data Protection Act 2023 (DPDPA) and DPDP Rules 2025.

The product is not built for English-speaking, tech-savvy users. It is built for a 35-year-old vegetable seller in Patna who uses PhonePe, a homemaker in Coimbatore who shops on Meesho, and a gig driver in Hyderabad who uses a food delivery app. These users have rights under DPDPA — SahmatOS ensures they can exercise them.

---

## 2. Problem Statement

### 2.1 The Compliance Gap

- DPDPA 2023 is enforceable from May 2027. Penalties up to ₹250Cr per breach.
- Most Indian SMEs (our target market) have no consent infrastructure.
- Existing solutions (OneTrust, Cookiebot) are designed for GDPR/Western markets — English-only, desktop-first, expensive.
- No Indian product exists that handles Eighth Schedule language consent at scale.

### 2.2 The Language Gap

- 70%+ of India's internet users prefer regional languages (IAMAI 2024).
- DPDPA Section 5 explicitly requires notice in the user's interface language.
- A consent form a user cannot understand is not "free, specific, informed, unconditional" — it is legally invalid.
- Current practice: businesses use English consent checkboxes that 80%+ of Tier 2-3 users cannot meaningfully understand.

### 2.3 The Dual-Role Gap

Under DPDPA Section 2(g), a Registered Consent Manager **cannot** simultaneously act as a Data Processor for the same Data Principal. This prohibits common patterns like "consent tool + analytics platform" bundles. SahmatOS is designed with this constraint as a first-class architectural concern.

---

## 3. Users and Personas

### 3.1 Data Principals (End Users — Primary Beneficiaries)

**Persona 1: Priya, 28, Tier 3 City (Ranchi)**
- Language: Hindi
- Device: ₹8,000 Android, 4G
- Uses: local e-commerce app, UPI payments
- Concern: doesn't know what data is collected; can't read English consent forms
- Need: Understand what she's consenting to; withdraw easily if she changes her mind

**Persona 2: Rajan, 42, Tier 2 City (Madurai)**
- Language: Tamil
- Device: ₹12,000 Android, WiFi at home
- Uses: grocery delivery, Tally-connected SME accounting
- Concern: received SMS from an unknown company that clearly used his data
- Need: Know which businesses have his consent; revoke specific ones

**Persona 3: Fatima, 19, Metro (Lucknow)**
- Language: Urdu
- Device: iPhone SE, 4G
- Uses: social media, online shopping
- Concern: gave consent "by accident" by clicking a pre-checked box
- Need: Easy to understand consent flow; confirmation in her language

**Persona 4: Arjun, 15, Tier 1 City (Bangalore)**
- Language: Kannada / English
- Device: school-issued tablet
- Uses: EdTech platform
- Regulatory concern: Under-18 — requires parental consent under Section 9
- Need: Parent (Lakshmi, 42, Kannada) to approve his data use; parent receives notification

### 3.2 Data Fiduciaries (Business Customers — Paying Users)

**Persona 5: Kiran, CTO, D2C Brand (₹5Cr ARR, Hyderabad)**
- Language: English/Telugu
- Team: 3 engineers
- Need: Embed consent widget in checkout flow; API to check consent before sending marketing SMS; evidence for DPBI audit
- Budget: ₹4,999/mo (Growth plan)

**Persona 6: Sunita, Founder, Healthtech Startup (Mumbai)**
- Language: English/Marathi
- Team: 8 people
- Need: Sensitive health data consent (purpose-level granularity); erasure within 48hr of patient request
- Budget: ₹9,999–14,999/mo (Enterprise plan)

**Persona 7: Amit, IT Manager, 50-person Retail Chain (Jaipur)**
- Language: Hindi/English
- Tech: Tally + WhatsApp for Business
- Need: Consent collection via WhatsApp in Hindi; Tally integration for financial data retention
- Budget: ₹999–4,999/mo

### 3.3 Administrators (TrustStack Internal)

**Persona 8: Compliance Officer**
- Need: DPBI audit trail; real-time breach detection; evidence package generation

---

## 4. User Stories

### 4.1 Consent Collection

| ID | As a... | I want to... | So that... | Priority |
|----|---------|-------------|-----------|----------|
| UC-01 | Data Principal | See consent notice in my preferred language | I can understand what I'm agreeing to | P0 |
| UC-02 | Data Principal | Grant or deny consent per-purpose separately | I control which purposes are allowed | P0 |
| UC-03 | Data Principal | Withdraw consent in 1 tap | It is as easy to withdraw as it was to grant | P0 |
| UC-04 | Data Principal | Receive confirmation of my consent action in my language | I have a record of what I agreed to | P0 |
| UC-05 | Data Principal (parent) | Approve consent on behalf of my child under 18 | My child's data is protected | P0 |
| UC-06 | Data Fiduciary | Embed a consent widget in my app with 5 lines of code | Implementation is not a 3-month project | P0 |
| UC-07 | Data Fiduciary | Check via API if a user has valid consent for a purpose | I can gate operations on consent status | P0 |
| UC-08 | Data Fiduciary | Collect consent via WhatsApp | I reach users where they already are | P1 |
| UC-09 | Data Fiduciary | Collect consent via SMS (for feature phones) | I reach users without smartphones | P2 |

### 4.2 Consent Management

| ID | As a... | I want to... | So that... | Priority |
|----|---------|-------------|-----------|----------|
| CM-01 | Data Principal | See all consents I've given across all businesses | I have a complete picture | P0 |
| CM-02 | Data Principal | Withdraw all consents from a specific business at once | I can cut off a business completely | P0 |
| CM-03 | Data Principal | Download a copy of my consent history | I have evidence for disputes | P1 |
| CM-04 | Data Fiduciary | Get webhook on consent withdrawal | I stop processing data immediately | P0 |
| CM-05 | Data Fiduciary | View consent audit log with cryptographic proof | I can demonstrate compliance to DPBI | P0 |
| CM-06 | Data Fiduciary | Set consent expiry per purpose | Consent is time-bounded where required | P1 |
| CM-07 | Compliance Officer | Generate DPBI audit report | Annual compliance submission is ready | P0 |

### 4.3 Erasure

| ID | As a... | I want to... | So that... | Priority |
|----|---------|-------------|-----------|----------|
| ER-01 | Data Principal | Request erasure of all my data from a business | I can exercise my right to be forgotten | P0 |
| ER-02 | Data Principal | Receive confirmation 48hr before deletion | I have time to reconsider | P0 |
| ER-03 | Data Principal | Get a verifiable deletion certificate | I have proof my data was deleted | P1 |
| ER-04 | Data Fiduciary | Trigger erasure via API | I can integrate with my own data deletion logic | P0 |
| ER-05 | Data Fiduciary | See erasure status (scheduled, notified, deleted) | I track compliance in real-time | P0 |
| ER-06 | Data Fiduciary | Receive erasure certificate after deletion | I have legal evidence | P1 |

### 4.4 Breach Notification

| ID | As a... | I want to... | So that... | Priority |
|----|---------|-------------|-----------|----------|
| BR-01 | Data Fiduciary | Trigger breach notification workflow via API | 6hr CERT-In deadline is met automatically | P0 |
| BR-02 | Data Principal | Receive breach notification in my language | I understand what happened to my data | P0 |
| BR-03 | Data Fiduciary | Get 72hr DPBI report auto-generated | 72hr deadline is met automatically | P0 |
| BR-04 | Compliance Officer | Track all breach events and their status | Regulatory evidence is maintained | P0 |

---

## 5. Functional Requirements

### 5.1 Language Requirements (Non-Negotiable)

**FR-LANG-01:** All consent notice text must be displayable in all 22 Eighth Schedule languages:
Assamese, Bengali, Bodo, Dogri, Gujarati, Hindi, Kannada, Kashmiri, Konkani, Maithili, Malayalam, Manipuri, Marathi, Nepali, Odia, Punjabi, Sanskrit, Santali, Sindhi, Tamil, Telugu, Urdu.

**FR-LANG-02:** Language detection must occur before any consent UI is rendered:
1. Check user's app/browser language preference
2. Fall back to device locale
3. Fall back to geographic IP-based language (approximate)
4. Fall back to Hindi (most widely understood across regions)
5. Always allow user to switch language explicitly

**FR-LANG-03:** Right-to-left rendering must be supported for Urdu, Kashmiri (Nastaliq script), Sindhi.

**FR-LANG-04:** Script-appropriate fonts must be loaded for each language (see `INDIC_LANGUAGE_SPEC.md` for font matrix).

**FR-LANG-05:** Translation must be done by our fine-tuned on-premise LLM — consent text must never be sent to external translation APIs.

**FR-LANG-06:** Human review of consent templates by native speakers required before production deployment.

### 5.2 Consent Collection Requirements

**FR-CON-01:** Consent must be collected per purpose — omnibus consent ("agree to everything") is prohibited.

**FR-CON-02:** No pre-checked boxes. Affirmative action required (tap/click).

**FR-CON-03:** Each consent action must produce a signed consent artifact containing:
- Tenant ID (Data Fiduciary)
- Data Principal identifier (pseudonymised)
- Purpose ID and human-readable purpose description (in DP's language)
- Action (GRANTED / WITHDRAWN / EXPIRED)
- Timestamp (UTC)
- Collection method (WIDGET / API / WHATSAPP / VERBAL_AADHAAR)
- Language code
- ECDSA P-256 signature
- Signing key ID (AWS KMS)

**FR-CON-04:** Withdrawal must be accessible from the same UI as grant — identical number of taps (DPBI requirement).

**FR-CON-05:** Parental consent flow: detect age (via declared DOB at registration) → if under 18, route to guardian consent flow → Aadhaar eKYC for age verification of guardian → store parent's consent artifact linked to child's record.

**FR-CON-06:** Consent widget must be embeddable via:
- JavaScript SDK (web)
- React Native component (mobile)
- REST API call (server-side, for non-interactive flows)
- WhatsApp Business API (conversational)

### 5.3 Consent Storage Requirements

**FR-STORE-01:** Consent records are append-only. No UPDATE or DELETE operations on `consent_records` table. DB-level triggers enforce this.

**FR-STORE-02:** Retention: 7 years minimum (Rule 4 requirement), after which records may be archived but not deleted without DPBI approval.

**FR-STORE-03:** All consent records must be stored in Indian cloud regions (AWS ap-south-1 or GCP asia-south1). Cross-region replication is permitted only within India.

**FR-STORE-04:** Data at rest: AES-256 encryption (AWS RDS encryption, KMS-managed keys).

**FR-STORE-05:** Personal data in consent records (DP identifier) must be stored as a HMAC-SHA256 hash. Raw identifier stored encrypted in separate table with stricter access controls.

### 5.4 Erasure Requirements

**FR-ERASE-01:** Erasure timelines follow Third Schedule class rules:
- Data Fiduciaries with ≥2Cr e-commerce users: 3yr from last transaction/login
- Data Fiduciaries with ≥50L gaming users: 3yr from last login
- Data Fiduciaries with ≥2Cr social media users: 3yr from last login
- All other DFs (most SMEs — primary target): Erase when purpose fulfilled or consent withdrawn

**FR-ERASE-02:** 48hr pre-deletion notification to Data Principal in their registered language before any erasure.

**FR-ERASE-03:** Erasure must be hard deletion — no soft delete, no archive flags. Physical deletion from all data stores in the DF's system (propagated via Erasure API).

**FR-ERASE-04:** Deletion certificate must be generated and signed after successful deletion, stored for audit.

**FR-ERASE-05:** Erasure workflows must survive server restarts, deployments, and infrastructure failures (Temporal.io durable execution).

### 5.5 Breach Notification Requirements

**FR-BREACH-01:** Breach workflow triggers within 15 minutes of breach detection event.

**FR-BREACH-02:** CERT-In initial report auto-generated and submitted within 6 hours.

**FR-BREACH-03:** DPBI detailed report auto-generated within 72 hours.

**FR-BREACH-04:** Affected Data Principals notified in their registered language.

**FR-BREACH-05:** All breach events immutably logged with timestamps.

---

## 6. Non-Functional Requirements

| Category | Requirement | Target | Notes |
|----------|-------------|--------|-------|
| Performance | Consent grant API (p50) | < 200ms | End-to-end including signing |
| Performance | Consent grant API (p99) | < 800ms | Under load |
| Performance | Consent check API (p50) | < 50ms | Redis cache hit |
| Performance | Consent check API (p99) | < 200ms | Cache miss + DB |
| Performance | Widget first render (4G) | < 2s | Tier 3 connectivity |
| Availability | API uptime | 99.9% | 8.7hr downtime/yr max |
| Availability | Consent check (read path) | 99.95% | Higher SLA, cached |
| Scalability | Consent events/second | 10,000 peak | During major sales events |
| Scalability | Concurrent widget renders | 50,000 | |
| Data retention | Consent records | 7 years | Rule 4 minimum |
| Language | Supported scripts | 22 Eighth Schedule | Non-negotiable |
| Language | Widget render time (Hindi) | < 100ms additional | Font + translation |
| Language | RTL rendering | Urdu, Kashmiri, Sindhi | Nastaliq script |
| Security | Signing algorithm | ECDSA P-256 | Per-tenant KMS keys |
| Security | Encryption at rest | AES-256 | RDS + KMS |
| Security | Transport | TLS 1.3 | |
| Compliance | Data residency | India only | ap-south-1 / asia-south1 |
| Compliance | Audit log retention | 7 years | Rule 4 |

---

## 7. Out of Scope (SahmatOS v1.0)

- PII discovery/scanning (Module 2 — separate service, separate VPC)
- Analytics on consent patterns (dual-role prohibition — cannot process what we consent-manage)
- Integration with non-Indian cloud providers for primary storage
- Voice-based consent collection (roadmap for v1.5)
- Biometric consent (roadmap for v2.0)
- Cross-border consent flows

---

## 8. Success Metrics

### Launch Metrics (Month 1–6)

| Metric | Target | Why |
|--------|--------|-----|
| Consent widgets embedded (businesses) | 500 | Top-of-funnel adoption |
| Consent events processed (monthly) | 10M | Scale validation |
| Languages served (unique) | 15+ | Language breadth validation |
| Widget render time (median, 4G) | < 2s | Tier 3 accessibility |
| Consent withdrawal success rate | > 99.5% | DPBI would flag failures |
| Erasure workflow success rate | > 99.9% | Legal obligation |

### Compliance Metrics

| Metric | Target |
|--------|--------|
| Breach notification compliance rate | 100% (no tolerance) |
| CERT-In 6hr deadline met | 100% |
| DPBI 72hr deadline met | 100% |
| DPBI audit requests fulfilled within SLA | 100% |

### Business Metrics (Year 1)

| Metric | Target |
|--------|--------|
| Module 3 (SahmatOS) ARR | ₹80L |
| Enterprise customers | 20+ |
| Growth plan customers | 100+ |
| Net Revenue Retention | > 110% |

---

## 9. Assumptions and Dependencies

### Assumptions
1. DPDPA enforcement begins May 2027 as notified — no further extensions expected.
2. DPBI will accept ECDSA P-256 signed artifacts as tamper evidence.
3. WhatsApp Business API provides read receipts sufficient for consent acknowledgement.
4. Aadhaar eKYC API will be available for parental consent verification.

### Dependencies
1. **AWS KMS (ap-south-1)** — per-tenant signing keys. No fallback to software signing.
2. **Temporal.io** — erasure and breach workflow durability. Self-hosted on Indian infra.
3. **Apache Kafka** — event bus. Self-managed on AWS MSK Mumbai.
4. **WhatsApp Business API** — consent via messaging. Meta's availability SLA applies.
5. **UIDAI Aadhaar eKYC API** — parental age verification. Government API availability.
6. **DPBI Sandbox** — testing breach notification workflows (expected Q2 2026).

---

## 10. Risks

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|-----------|
| DPDPA enforcement delayed again | Medium | Low (more time to build) | Build anyway — compliance is table stakes |
| DPBI rejects ECDSA artifacts | Low | Critical | Engage DPBI sandbox early; use standard X.509 format |
| Aadhaar eKYC API down for parental consent | Medium | High | Alternative: OTP + self-declaration with audit trail |
| Kafka MSK latency > 50ms | Low | Medium | Test at scale; circuit breaker to synchronous fallback |
| Font loading fails on low-end Android | Medium | High | Bundle critical fonts (Hindi/Tamil) in app; lazy-load rest |
| Machine translation quality insufficient for consent | Medium | Critical | Human review mandatory; legal team sign-off per language |

---

## Appendix A: DPDPA Compliance Mapping

| DPDPA Clause | Product Mechanism |
|--------------|------------------|
| Section 5 (Notice) | Language-matched notice renderer |
| Section 6 (Consent) | Per-purpose consent widget, no pre-checks |
| Section 6(4) (Withdrawal ease) | 1-tap withdrawal = 1-tap grant |
| Section 7 (Deemed consent) | Not applicable — CM role only |
| Section 9 (Children) | Aadhaar eKYC guardian flow |
| Section 12 (Rights) | Consumer dashboard: access, correct, erase |
| Section 13 (Grievance) | In-app grievance submission |
| Section 16 (Cross-border) | Indian cloud regions, Section 16 metadata tag |
| Rule 4 (CM Registration) | TrustStack registration + infrastructure requirements |
| Rule 7 (Breach) | Temporal.io breach workflow |
| Third Schedule (Erasure) | Class-aware Temporal.io erasure workflow |
