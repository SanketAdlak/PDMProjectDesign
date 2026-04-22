# Product Requirements Document — Regulatory War Room

**Version:** 1.0  
**Date:** 2026-04-22  
**Status:** Draft  

---

## 1. Executive Summary

The Regulatory War Room is TrustStack's CXO-facing compliance intelligence product. While the Consent Vault and Erasure Engine handle the operational mechanics of DPDPA compliance, the War Room is where leadership manages the lifecycle of compliance — understanding where they stand, what actions are overdue, and what their risk exposure is.

The product sits at the intersection of:
- **Operational data** (from Consent Vault + Erasure Engine)
- **Regulatory intelligence** (DPDPA, DPDP Rules, DPBI circulars, enforcement actions)
- **Business data** (which vendors have access, what data categories, what volumes)
- **AI analysis** (gap detection, penalty modelling, regulatory change impact)

---

## 2. Problem Statement

### 2.1 The CXO Visibility Problem

A CEO of a 300-person SaaS company in India:
- Has 15 vendor contracts that may involve data sharing
- Has 3 modules of TrustStack running (consent, erasure, discovery)
- Has no way to answer "are we compliant?" without a 3-hour engineering briefing
- Has a board meeting in 2 weeks where investors will ask about DPDPA readiness
- Has one DPO who is overwhelmed with day-to-day operations

**They need a boardroom-ready compliance status in 60 seconds, not 3 hours.**

### 2.2 The Action Queue Problem

Compliance is not a static state — it's a calendar of obligations. Every month:
- New consent decisions expire
- Erasure timelines approach
- Vendor certifications need renewal
- Regulatory filings are due
- Breach drills should be run

Without a centralised action queue, critical deadlines are missed. After May 2027, each missed deadline can trigger DPBI investigation with penalties up to ₹250Cr.

### 2.3 The Regulatory Change Problem

DPBI regularly issues clarifications, amendments, and circulars that change what compliance looks like. When DPBI issued a clarification on "verifiable parental consent" in Q1 2026, most companies didn't know for 3 months. By then, they were already non-compliant.

The War Room must monitor regulatory changes and automatically assess impact on the company's specific configuration.

---

## 3. User Personas

### 3.1 Aisha, CEO, 250-person D2C Brand (Mumbai)

- **Language:** English / Hindi
- **Time budget:** 15 minutes per week on compliance
- **Need:** One number — "Compliance Score: 87/100 — 2 critical gaps". Board deck auto-generated monthly.
- **Pain:** Every compliance conversation requires 3 engineers + the DPO in a room
- **Moment of truth:** DPBI sends a notice. She needs to know: "Are we exposed? How badly?"

### 3.2 Vikram, DPO, HealthTech Startup (Bangalore)

- **Language:** English / Kannada
- **Time budget:** Full-time — this is his job
- **Need:** Daily action queue, breach drill scheduler, vendor compliance tracker
- **Pain:** Managing compliance across 8 departments. Can't track everything in spreadsheets.
- **Moment of truth:** A vendor reports a data incident. Vikram needs to trigger breach workflow and know which DPs are affected within 30 minutes.

### 3.3 Rajesh, CISO, 500-person Fintech (Hyderabad)

- **Language:** English / Telugu
- **Need:** Security posture as it relates to DPDPA. Breach readiness score. Time-to-detect metrics.
- **Pain:** DPDPA breach notification rules (6hr CERT-In) require pre-built workflows. He needs to test them regularly.
- **Moment of truth:** Penetration testing finds a vulnerability. How quickly can he trigger the breach notification workflow and simulate the 6hr CERT-In deadline?

### 3.4 Meera, CFO, Manufacturing Company (Pune)

- **Language:** English / Marathi
- **Need:** Financial exposure from DPDPA non-compliance. Cost of compliance vs. cost of penalty.
- **Pain:** Can't quantify the risk to present to board or insurers.
- **Moment of truth:** D&O insurance renewal — insurer asks for DPDPA compliance evidence.

---

## 4. Product Modules

The Regulatory War Room comprises five interconnected modules:

### Module A: Compliance Command Centre
Live compliance status. Headline score. Critical gaps. Top 5 actions due this week.

### Module B: Regulatory Lifecycle Tracker
Visual timeline of DPDPA enforcement milestones. Status against each requirement. Days remaining to each deadline.

### Module C: Risk Intelligence Engine
Penalty exposure calculator. Breach probability model. Vendor risk scoring. Data category risk heat map.

### Module D: Action Intelligence Queue
Prioritised, owner-assigned actions. Deadlines. DPDPA section references. Auto-generated from operational data + regulatory calendar.

### Module E: Reporting Suite
Board deck generator. DPBI annual audit report. Vendor compliance scorecard. Breach simulation reports. All ECDSA-signed.

---

## 5. User Stories

### 5.1 Compliance Command Centre

| ID | As a... | I want to... | So that... | Priority |
|----|---------|-------------|-----------|----------|
| CC-01 | CEO | See a single compliance score (0-100) on the home screen | I know our overall status at a glance | P0 |
| CC-02 | CEO | See the 3 most critical compliance gaps with business impact | I know exactly what to fix first | P0 |
| CC-03 | DPO | See live counts: consents active, erasures pending, breaches open | I have a real-time operational view | P0 |
| CC-04 | CEO | See a trend line of our compliance score over the past 12 months | I can show the board we're improving | P1 |
| CC-05 | DPO | Set up alerts when compliance score drops below a threshold | I'm notified of regressions immediately | P1 |

### 5.2 Regulatory Lifecycle

| ID | As a... | I want to... | So that... | Priority |
|----|---------|-------------|-----------|----------|
| RL-01 | DPO | See a visual timeline of all DPDPA milestones with our readiness | I know where we are on the journey | P0 |
| RL-02 | DPO | See days remaining to Consent Manager registration (Nov 2026) | I'm not surprised by the deadline | P0 |
| RL-03 | DPO | See our gap vs. Rule 4 CM registration requirements | I have a specific checklist, not a vague goal | P0 |
| RL-04 | CISO | See breach notification readiness (6hr CERT-In / 72hr DPBI) | I know if our workflows are ready | P0 |
| RL-05 | DPO | Track DPBI circular updates and their impact on our operations | Regulatory changes don't surprise me | P1 |

### 5.3 Risk Intelligence

| ID | As a... | I want to... | So that... | Priority |
|----|---------|-------------|-----------|----------|
| RI-01 | CFO | See our maximum penalty exposure (in ₹Cr) if audited today | I can present financial risk to board | P0 |
| RI-02 | CISO | See breach probability score based on current security posture | I prioritise security investments | P1 |
| RI-03 | DPO | See a vendor risk table (each vendor's compliance status) | I know which vendors are compliance liabilities | P0 |
| RI-04 | DPO | See which data categories are highest risk if breached | I focus consent and security efforts correctly | P1 |
| RI-05 | CEO | See a 30/60/90-day risk forecast | I plan compliance budget and resource allocation | P2 |

### 5.4 Action Queue

| ID | As a... | I want to... | So that... | Priority |
|----|---------|-------------|-----------|----------|
| AQ-01 | DPO | See all overdue compliance actions sorted by risk | I prioritise correctly | P0 |
| AQ-02 | DPO | Assign actions to specific team members | Accountability is clear | P0 |
| AQ-03 | CTO | See only actions assigned to engineering team | I filter noise | P1 |
| AQ-04 | DPO | Mark actions as completed with evidence attached | Audit trail of what we did and when | P0 |
| AQ-05 | DPO | Run a breach notification drill from the action queue | We test our workflows before we need them | P1 |
| AQ-06 | DPO | Get WhatsApp/Slack/email alerts for critical overdue actions | I'm notified in my preferred channel | P1 |

### 5.5 Reporting

| ID | As a... | I want to... | So that... | Priority |
|----|---------|-------------|-----------|----------|
| RE-01 | CEO | Generate a board-ready compliance deck in one click | Board meeting prep is 15 minutes, not 3 hours | P0 |
| RE-02 | DPO | Generate the DPBI annual audit report in DPBI's format | Annual filing is done in one click | P0 |
| RE-03 | DPO | Generate a vendor compliance scorecard | I can show vendors their status and demand improvements | P1 |
| RE-04 | DPO | Generate a consent health report (by purpose, language, channel) | I understand consent collection quality | P1 |
| RE-05 | CFO | Export compliance cost vs. risk analysis | I build the ROI case for compliance investment | P2 |

---

## 6. Functional Requirements

### 6.1 Compliance Score Calculation

**FR-SCORE-01:** Compliance score (0-100) is calculated from weighted components:

| Component | Weight | Source |
|-----------|--------|--------|
| Consent collection completeness | 25% | Module 3.1: What % of purposes have active consents? |
| Erasure compliance rate | 20% | Module 3.2: What % of erasures completed on time? |
| Breach notification readiness | 20% | Module 3.3: Has the 6hr/72hr workflow been tested? |
| Vendor compliance coverage | 15% | Module 3.3: What % of vendors have compliance certs? |
| CM registration progress | 10% | Module 3.3: What % of Rule 4 requirements met? |
| Language coverage | 10% | Module 3.1: What % of purposes have approved translations? |

**FR-SCORE-02:** Score is refreshed every 4 hours (near-real-time for most CXO needs).

**FR-SCORE-03:** Score breakdown must show per-component scores so DPO knows what to fix.

### 6.2 Regulatory Change Monitoring

**FR-REG-01:** The AI Regulatory Monitor scans for DPDPA-related updates daily:
- DPBI official website (orders, circulars, FAQs)
- Ministry of Electronics and IT (MeitY) press releases
- High Court / Supreme Court judgments involving DPDPA
- Industry body guidance (IAMAI, NASSCOM)

**FR-REG-02:** For each new regulatory document, the AI Monitor must:
1. Classify: Rule change / Guidance / Enforcement action / Clarification
2. Assess impact on this specific tenant: High / Medium / Low / Not applicable
3. Generate plain-English summary of what changed and what action is needed
4. Create an action item in the Action Queue if impact is High/Medium

**FR-REG-03:** AI Monitor must not hallucinate regulatory changes. Every alert must link to the source document.

### 6.3 Penalty Exposure Calculator

**FR-PENALTY-01:** Calculate estimated penalty exposure using:
```
Maximum Exposure = Σ (DataCategory_i × Records_i × PenaltyRate_i)

Where PenaltyRate is determined by:
- Tier 1 (no consent for advertising/analytics): Up to ₹250Cr per instance
- Tier 2 (breach not notified within Rule 7 timelines): Up to ₹200Cr
- Tier 3 (CM registration not obtained by Nov 2026): Up to ₹150Cr
- Tier 4 (inadequate security measures): Up to ₹250Cr
```

**FR-PENALTY-02:** Penalty calculator must show current gaps that drive exposure (not hypothetical scenarios).

**FR-PENALTY-03:** Calculator output is for internal risk management — include disclaimer that it is not legal advice.

### 6.4 Reporting Engine

**FR-REPORT-01:** Board deck template:
- Compliance score with trend
- Top 3 risks and mitigation status
- Consent metrics (by purpose, language, channel)
- Erasure compliance metrics
- Upcoming key dates
- One-paragraph narrative (AI-generated, editable)

**FR-REPORT-02:** DPBI audit report must match DPBI's prescribed format (to be updated as DPBI publishes format).

**FR-REPORT-03:** All reports are ECDSA-signed by TrustStack's master signing key (attestation that data came from TrustStack systems).

**FR-REPORT-04:** Reports generated in English by default; executive summary available in tenant's default language.

---

## 7. Non-Functional Requirements

| Category | Requirement | Target |
|----------|-------------|--------|
| Latency | Dashboard initial load | < 3s |
| Latency | Compliance score refresh | < 500ms (cached) |
| Latency | Report generation | < 30s for standard templates |
| Freshness | Operational data (consent/erasure) | 4-hour lag max |
| Freshness | Regulatory change alerts | 24-hour max from publication |
| Availability | War Room API | 99.9% |
| Security | No PII in War Room | All metrics aggregated — zero raw DP data |
| Security | Report signing | ECDSA P-256 on all generated reports |
| Language | Executive summaries | Tenant's default language |
| Mobile | Mobile-friendly | CXOs check this on phones |

---

## 8. Pricing

The Regulatory War Room is bundled with higher TrustStack tiers:

| Plan | Price | Includes |
|------|-------|---------|
| Growth (₹4,999/mo) | — | Basic compliance score, action queue |
| Enterprise (₹14,999/mo) | — | Full War Room: score + lifecycle + risk + reports |
| Platform (Custom) | — | White-label War Room, custom reporting, API access |
| Add-on: AI Regulatory Monitor | ₹2,999/mo | Regulatory change alerts + impact assessment |
| Add-on: Penalty Exposure Calculator | ₹1,999/mo | Financial risk modelling |

---

## 9. Integration Requirements

The War Room consumes data from all three TrustStack modules via:
- **Kafka events**: real-time consent, erasure, breach events
- **Aggregation APIs**: Module 3.1 and 3.2 expose read APIs for metric aggregation
- **Static data**: Tenant config, vendor list, regulatory calendar (maintained by TrustStack compliance team)

**No raw DP data enters the War Room.** All aggregation happens at the module level. The War Room only receives counts, rates, and timestamps — never identifiers or content.
