## graphify

This project has a graphify knowledge graph at graphify-out/.

Rules:
- Before answering architecture or codebase questions, read graphify-out/GRAPH_REPORT.md for god nodes and community structure
- If graphify-out/wiki/index.md exists, navigate it instead of reading raw files
- After modifying code files in this session, run `graphify update .` to keep the graph current (AST-only, no API cost)

---

## TrustStack — DPDPA Compliance Suite

### Project Identity
TrustStack is India's first AI-powered DPDPA compliance platform for SMEs and startups (Tier 1–3 cities). We operate as a **Registered Consent Manager** under Section 2(g) of DPDPA 2023 — the bridge between Data Principals (individuals) and Data Fiduciaries (businesses). Modelled on RBI's Account Aggregator framework.

### Regulatory Context (CRITICAL — read before any feature work)

**DPDPA 2023 + DPDP Rules 2025 — Core Rules:**
- Consent is the ONLY basis for advertising/analytics/marketing. No "legitimate interests" (unlike GDPR).
- Consent Manager registration: Rule 4, effective 13 Nov 2026. Requires: India incorporation, ₹2Cr net worth, AES-256 encryption, 7-year consent record retention, machine-readable format.
- Notice (Section 5, Rule 3): Standalone notice in English + any of 22 Eighth Schedule languages, matching the user's interface language.
- Consent: Free, specific, informed, unconditional, unambiguous. Separate per purpose. No pre-checked boxes. Withdrawal must be as easy as granting (DPBI rejects cumbersome flows).
- Breach (Rule 7): Immediate notify Data Principals + DPBI. 72hr detailed report. No materiality threshold (ALL breaches). Also CERT-In within 6hrs.
- Children (Section 9): Verifiable parental consent for under-18. No targeted ads or profiling of minors.
- Cross-border (Section 16): DPBI must be able to access consent logs even on foreign infra. Use Indian cloud regions.
- Dual-role prohibition: Consent Manager CANNOT also be Data Processor for the same Data Principal.

**Erasure Rules (CLASS-SPECIFIC — do NOT generalise as "3 years"):**
- E-commerce ≥2Cr users: 3yr from last transaction/login
- Online gaming ≥50L users: 3yr from last login
- Social media ≥2Cr users: 3yr from last login
- All other DFs (most SMEs — our target): Erase when purpose fulfilled or consent withdrawn
- 48hr pre-deletion notification required. Soft-delete is non-compliant.
- 1yr minimum retention of processing logs for security.

**Enforcement:** Nov 2025 rules notified → Nov 2026 CM registration → May 2027 full enforcement (no grace period, penalties up to ₹250Cr).

### Product Architecture (3 Modules)

**Module 1 — Vernacular Awareness (Hook / Free tier)**
- Jargon Slayer: AI translating DPDPA into 22 languages in "Founder-speak"
- 3-Min Trust Check: Free scanner → missing consent banners, ghost trackers, unencrypted PII, language gaps
- Target: 100K scans in 6mo for top-of-funnel

**Module 2 — Assessment & Discovery (Revenue Driver)**
- Unit-Based Billing per "Purpose-Linked Data Flow" (not per-seat)
- AI Data Discovery: Crawls AWS/GCP/Zoho/Razorpay/WhatsApp for hidden PII → Data Flow Map
- Multilingual Contract Audit: NLP reads vendor contracts in 22 languages, flags non-compliant clauses
- CRITICAL: Discovery Engine runs SANDBOXED — no access to Consent Vault data (prevents dual-role conflict)

**Module 3 — Implementation & Automation (The Shield)**
- Consent Vault: API-first, ECDSA-signed artifacts, purpose-level granularity, 22-language widgets, parental consent workflow (Section 9), consumer dashboard
- Erasure Engine: Class-specific timelines per Third Schedule, 48hr alerts, verifiable deletion certificates
- Regulatory War Room: 72hr DPBI + 6hr CERT-In breach workflow, annual audit generator, regulatory change tracker

### Tech Stack
- Cloud: AWS Mumbai / GCP Mumbai (consent records MUST stay on Indian servers)
- Consent Engine: Kafka (event-driven), PostgreSQL (append-only immutable audit tables), ECDSA P-256 signatures
- AI: Fine-tuned LLMs for translation/PII detection/doc analysis, hosted on-premise
- Integration: GraphQL bus, OAuth 2.0, read-only discovery + permissioned write for erasure
- Security: AES-256 at rest, TLS 1.3 in transit, Zero-Knowledge (data unreadable to TrustStack)
- Workflow: temporal.io for consent/erasure/breach orchestration
- Frontend: React / React Native, mobile-first (5.5" screen on 4G for Tier 2–3)

### Integrations
- Tally (7M+ biz): Financial PII discovery, consent-linked retention
- Zoho (CRM/Books/Desk): Embedded consent widgets, cross-suite erasure
- WhatsApp Business (500M+ users): Regional-language consent, 1-tap withdrawal
- Razorpay: Payment/KYC consent tracking

### Monetisation
Free → ₹499/mo (Awareness Pro) → ₹999/flow (Assessment) → ₹4,999/mo (Growth) → ₹14,999/mo (Enterprise) → ₹2,499–9,999/mo (Consent Mgmt by DP volume) → Custom (Platform). Y1 target: ₹2–3Cr ARR, 500+ customers.

### Branding
- **TrustStack**: Primary brand. "Build Trust. Ship Faster."
- **SahmatOS**: Consent Vault product name (सहमत = consent)
- **PrivacyDost**: Tier 3 vernacular sub-brand (दोस्त = friend)

### Design Principles
- Vernacular-first: Design in Hindi/Tamil FIRST, adapt to English
- 3-tap compliance: Any core action in ≤3 taps
- Progressive disclosure: Simple action first, legal detail on demand
- Withdrawal = 1-tap, identical UX to granting (DPBI requirement)
- Mobile-first: Full workflow on 5.5" screen, 4G connection

### Code Conventions
- TypeScript, ESM modules (no CommonJS)
- All APIs documented with OpenAPI 3.1
- Tests: Vitest (unit), Playwright (E2E)
- Commits: conventional (feat:, fix:, docs:, refactor:)
- Branches: feature/MODULE-description, fix/MODULE-description
- All PII-handling code MUST have comments referencing the DPDPA section it addresses
- Never log personal data. Structured logging with PII-safe fields only.
- All DB migrations must be reversible
