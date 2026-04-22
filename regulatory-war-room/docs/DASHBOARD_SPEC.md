# Dashboard Specification — Regulatory War Room

**Version:** 1.0  
**Date:** 2026-04-22  

---

## Design Principles

1. **Answer in 5 seconds**: A CXO opening the app should know their status in 5 seconds without scrolling
2. **Action over information**: Every view leads to a concrete action, not just data
3. **Risk-first language**: Use business risk language, not technical compliance jargon
4. **Vernacular where needed**: Executive summaries available in Marathi, Tamil, Hindi, Telugu — not just English
5. **Mobile-first**: CXOs check this on iPhones between meetings

---

## Screen 1: Command Centre (Home)

```
┌──────────────────────────────────────────────────────────┐
│  [TrustStack Logo]   [Company Name]    [🔔 3] [👤 Aisha] │
├──────────────────────────────────────────────────────────┤
│                                                          │
│   COMPLIANCE STATUS          as of 2 hours ago          │
│                                                          │
│   ┌──────────────────────────────────────────────────┐  │
│   │                                                  │  │
│   │              87/100                              │  │
│   │                                                  │  │
│   │   ████████████████████████░░░░░                  │  │
│   │                                                  │  │
│   │   ↑ +3 from last month     May 2027: 245 days   │  │
│   └──────────────────────────────────────────────────┘  │
│                                                          │
│   CRITICAL GAPS  (2 need immediate action)              │
│                                                          │
│   🔴  CM Registration: 58% complete                     │
│       Action: Submit net worth certification by Jun 26  │
│       [Take Action →]                                   │
│                                                          │
│   🔴  7 consent purposes lack approved Tamil translation │
│       Action: 3 in review, 4 pending submission         │
│       [Take Action →]                                   │
│                                                          │
│   🟡  Breach drill overdue (last: 8 months ago)         │
│       [Schedule drill →]                                │
│                                                          │
├──────────────────────────────────────────────────────────┤
│   TODAY'S METRICS           (live, 15-min refresh)      │
│                                                          │
│   Consents active     Erasures pending    Breaches open  │
│   1,24,847            23                  0              │
│   ↑ 2,341 today       3 overdue ⚠️        ✓ clear        │
│                                                          │
├──────────────────────────────────────────────────────────┤
│   [📊 Reports]  [⚠️ Actions]  [📅 Timeline]  [⚡ Risk]   │
└──────────────────────────────────────────────────────────┘
```

---

## Screen 2: Compliance Score Breakdown

```
┌──────────────────────────────────────────────────────────┐
│  ← Back    COMPLIANCE SCORE DETAIL      [Export Report]  │
├──────────────────────────────────────────────────────────┤
│                                                          │
│   Overall: 87/100  ↑ +3 MoM                             │
│                                                          │
│   COMPONENT SCORES                                       │
│                                                          │
│   Consent Collection (25% weight)          92/100  ✓     │
│   ████████████████████████░░░               [Details →]  │
│                                                          │
│   Erasure Compliance (20% weight)          95/100  ✓     │
│   █████████████████████████░                [Details →]  │
│                                                          │
│   Breach Readiness (20% weight)            68/100  ⚠️    │
│   █████████████████░░░░░░░░░                [Details →]  │
│   ↑ Last drill: 8 months ago. Run a drill!              │
│                                                          │
│   Vendor Compliance (15% weight)           80/100  ⚠️    │
│   ████████████████████░░░░░                 [Details →]  │
│   ↑ 3 vendors have expired DPA agreements              │
│                                                          │
│   CM Registration (10% weight)             58/100  🔴    │
│   ██████████████░░░░░░░░░░░░░               [Details →]  │
│   ↑ 6 of 10 Rule 4 requirements met                    │
│                                                          │
│   Language Coverage (10% weight)           85/100  ✓     │
│   ████████████████████████░                 [Details →]  │
│   ↑ 7/22 languages approved. Phase 1 on track.         │
│                                                          │
├──────────────────────────────────────────────────────────┤
│   TREND (12 months)                                      │
│                                                          │
│   100 ┤                                         ●        │
│    90 ┤                              ●●●●●●●●●●          │
│    80 ┤               ●●●●●●●●●●●●                       │
│    70 ┤    ●●●●●●●●●                                     │
│    60 ┤●●                                                │
│    50 ┤                                                  │
│       └──────────────────────────────────────────        │
│       May    Jul    Sep    Nov    Jan    Mar    Apr       │
│       2025                                    2026       │
└──────────────────────────────────────────────────────────┘
```

---

## Screen 3: DPDPA Enforcement Timeline

```
┌──────────────────────────────────────────────────────────┐
│  ← Back    REGULATORY LIFECYCLE                          │
├──────────────────────────────────────────────────────────┤
│                                                          │
│   TODAY: 22 Apr 2026        ENFORCEMENT: May 2027       │
│                             245 days remaining           │
│                                                          │
│   MILESTONE TIMELINE                                     │
│                                                          │
│   ✅  Nov 2025                                           │
│   │   DPDP Rules 2025 Notified                          │
│   │   Status: Rules analysed, 23 obligations mapped     │
│   │                                                     │
│   ⏳  ← YOU ARE HERE                                    │
│   │   Apr 2026                                          │
│   │                                                     │
│   🔴  13 Nov 2026  ← 205 days                           │
│   │   Consent Manager Registration Deadline (Rule 4)    │
│   │   Status: 58% complete — 6 gaps remaining           │
│   │   [View Rule 4 Checklist →]                        │
│   │                                                     │
│   ⚪  May 2027                                           │
│       Full Enforcement — No Grace Period                 │
│       Penalties: up to ₹250Cr per violation             │
│       Status: 245 days to prepare                       │
│                                                          │
├──────────────────────────────────────────────────────────┤
│   RULE 4 CHECKLIST (CM Registration)                    │
│                                                          │
│   ✅  India incorporation                               │
│   ✅  ₹2Cr net worth (verified)                        │
│   ✅  AES-256 encryption (implemented)                  │
│   ✅  7-year consent record retention (configured)      │
│   ✅  Machine-readable format (API ready)               │
│   ✅  Indian cloud regions only (ap-south-1)            │
│   🔴  DPBI sandbox testing (not started)                │
│       Action: Apply for sandbox access by Jun 2026      │
│   🔴  Third-party security audit (not started)          │
│       Action: Engage auditor by Aug 2026                │
│   🔴  Board DPA agreement signed                        │
│       Action: Legal team — send template               │
│   ⚪  Formal application submission                      │
│       Due: Oct 2026 (for Nov registration)              │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

## Screen 4: Risk Intelligence

```
┌──────────────────────────────────────────────────────────┐
│  ← Back    RISK INTELLIGENCE           [Download PDF]    │
├──────────────────────────────────────────────────────────┤
│                                                          │
│   PENALTY EXPOSURE (as of today, if audited now)        │
│                                                          │
│   ┌─────────────────────────────────────────────────┐   │
│   │  Maximum Exposure: ₹47.3 Cr                    │   │
│   │  Expected Exposure: ₹8.2 Cr                    │   │
│   │  (Based on current gaps + industry enforcement  │   │
│   │   patterns — not legal advice)                 │   │
│   └─────────────────────────────────────────────────┘   │
│                                                          │
│   EXPOSURE BY RISK CATEGORY                             │
│                                                          │
│   CM Registration gap         ₹25Cr ████████████        │
│   Language compliance gap     ₹12Cr ██████              │
│   Vendor non-compliance       ₹8Cr  ████                │
│   Breach readiness gap        ₹2Cr  █                   │
│                                                          │
├──────────────────────────────────────────────────────────┤
│   VENDOR COMPLIANCE SCORECARD                           │
│                                                          │
│   Vendor               DPA Signed  Audit  Score         │
│   ─────────────────────────────────────────────         │
│   Razorpay             ✅          ✅     94/100 ✓       │
│   Zoho India           ✅          ⚠️     76/100 ⚠️      │
│   AWS (Mumbai region)  ✅          ✅     98/100 ✓       │
│   Tally Solutions      ✅          🔴     52/100 🔴      │
│     ↑ DPA expired 3 months ago — renew immediately     │
│   WhatsApp Business    ✅          ⚠️     71/100 ⚠️      │
│   Mailchimp India      🔴          🔴     31/100 🔴      │
│     ↑ No India DPA — migration or termination needed   │
│                                                          │
├──────────────────────────────────────────────────────────┤
│   DATA RISK HEAT MAP                                    │
│                                                          │
│   Data Category        Volume    Risk Level             │
│   Payment data         1.2L      🔴 HIGH                │
│   Health data          45K       🔴 HIGH                │
│   Contact info         8.4L      🟡 MEDIUM              │
│   Purchase history     8.4L      🟡 MEDIUM              │
│   Device identifiers   12L       🟢 LOW                 │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

## Screen 5: Action Intelligence Queue

```
┌──────────────────────────────────────────────────────────┐
│  ← Back    ACTION QUEUE        [Filter ▾]  [Export]     │
│            23 actions  •  3 critical  •  5 high         │
├──────────────────────────────────────────────────────────┤
│                                                          │
│   🔴 CRITICAL (3 actions)                               │
│                                                          │
│   [!] 3 erasures overdue — legal obligation             │
│       Section 12 + Third Schedule                       │
│       Due: IMMEDIATELY                                   │
│       Owner: Kiran (IT Manager)  [Assign]               │
│       Penalty risk: ₹150Cr per overdue erasure          │
│       [View overdue erasures →]                        │
│                                                          │
│   [!] CM Registration: DPBI sandbox testing needed      │
│       Rule 4 requirement — Nov 2026 deadline            │
│       Due: 15 Jun 2026 (54 days)                        │
│       Owner: Unassigned  [Assign]                       │
│       [Apply for DPBI sandbox →]                       │
│                                                          │
│   [!] Vendor: Mailchimp lacks India DPA agreement       │
│       Processing DP data without valid DPA              │
│       Due: IMMEDIATELY (stop processing or get DPA)     │
│       Owner: Priya (Legal)  [Assign]                    │
│       [Migrate / Get DPA →]                            │
│                                                          │
│   ─────────────────────────────────────────────────     │
│   🟡 HIGH (5 actions)                                    │
│                                                          │
│   [ ] Tamil translations: 4 purposes pending submission │
│       Due: 30 May 2026  Owner: Ravi (Compliance)        │
│       [Submit for review →]                            │
│                                                          │
│   [ ] Breach notification drill overdue                 │
│       Due: 22 May 2026  Owner: Suresh (CISO)           │
│       [Schedule drill →]                               │
│                                                          │
│   [ ] Vendor DPA renewal: Tally Solutions               │
│       Due: 15 May 2026  Owner: Legal team               │
│       [Renew DPA →]                                    │
│                                                          │
│   [+ Show 15 more actions]                              │
└──────────────────────────────────────────────────────────┘
```

---

## Screen 6: Regulatory AI Monitor

```
┌──────────────────────────────────────────────────────────┐
│  ← Back    REGULATORY MONITOR           [Mark all read]  │
│            7 new updates since last login               │
├──────────────────────────────────────────────────────────┤
│                                                          │
│   🔴 HIGH IMPACT  •  Yesterday                          │
│                                                          │
│   DPBI Circular 14/2026: Clarification on "Verifiable   │
│   Parental Consent" under Section 9                     │
│                                                          │
│   What changed: DPBI clarified that Aadhaar eKYC alone  │
│   is insufficient — must be combined with OTP to parent's│
│   registered mobile number for final verification.      │
│                                                          │
│   Impact on you: Your current parental consent flow     │
│   uses Aadhaar eKYC only. Update required.              │
│                                                          │
│   Source: [DPBI Circular 14/2026 ↗]                    │
│   Reviewed by: TrustStack Compliance Team ✓             │
│   [Add to Action Queue]  [Dismiss]                     │
│                                                          │
│   ─────────────────────────────────────────────────     │
│   🟡 MEDIUM IMPACT  •  3 days ago                       │
│                                                          │
│   MeitY Press Release: Draft DPDP Amendment for         │
│   Sensitive Personal Data definition                    │
│                                                          │
│   What changed: Draft amendment proposes expanding      │
│   "sensitive personal data" to include financial        │
│   transaction history (currently not classified).       │
│                                                          │
│   Impact on you: DRAFT ONLY — not yet law. Monitor.    │
│   If enacted: your e-commerce purpose notices will     │
│   need to be updated to reflect sensitive data notice. │
│                                                          │
│   Source: [MeitY Press Release ↗]                      │
│   Status: DRAFT — under public consultation            │
│   [Monitor]  [Dismiss]                                 │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

## Screen 7: Report Generation

```
┌──────────────────────────────────────────────────────────┐
│  ← Back    REPORTS                                       │
├──────────────────────────────────────────────────────────┤
│                                                          │
│   GENERATE REPORT                                        │
│                                                          │
│   ┌─────────────────────────────────────────────────┐   │
│   │  📊 Board Compliance Deck               [Generate]│  │
│   │  Monthly summary for board meetings               │  │
│   │  Format: PDF + PowerPoint  •  ~30 seconds        │  │
│   │  Last generated: 15 Apr 2026                     │  │
│   └─────────────────────────────────────────────────┘   │
│                                                          │
│   ┌─────────────────────────────────────────────────┐   │
│   │  📋 DPBI Annual Audit Report           [Generate]│  │
│   │  Formal compliance submission to DPBI            │  │
│   │  Format: DPBI XML + PDF  •  ~60 seconds         │  │
│   │  Due: 31 Mar 2027  (344 days)                   │  │
│   └─────────────────────────────────────────────────┘   │
│                                                          │
│   ┌─────────────────────────────────────────────────┐   │
│   │  🏢 Vendor Compliance Scorecard        [Generate]│  │
│   │  Share with vendors to demand improvements       │  │
│   │  Format: PDF  •  ~20 seconds                    │  │
│   └─────────────────────────────────────────────────┘   │
│                                                          │
│   ┌─────────────────────────────────────────────────┐   │
│   │  ⚡ Breach Readiness Assessment        [Generate]│  │
│   │  Post-drill compliance evidence                  │  │
│   │  Format: PDF (ECDSA signed)  •  ~15 seconds     │  │
│   │  Last drill: 8 months ago (⚠️ Run drill first)  │  │
│   └─────────────────────────────────────────────────┘   │
│                                                          │
├──────────────────────────────────────────────────────────┤
│   RECENT REPORTS                                         │
│                                                          │
│   📊 Board Deck — Apr 2026          [Download] [Share]  │
│      Generated: 15 Apr 2026 • ECDSA signed ✓           │
│                                                          │
│   📋 Vendor Scorecard — Mar 2026    [Download] [Share]  │
│      Generated: 31 Mar 2026 • ECDSA signed ✓           │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

## Mobile View (CXO on iPhone)

```
┌─────────────────────────┐
│ ≡  TrustStack       🔔3 │
├─────────────────────────┤
│                         │
│   COMPLIANCE SCORE      │
│                         │
│      87/100             │
│   ██████████░░          │
│   ↑ +3 MoM              │
│                         │
│ ─────────────────────── │
│                         │
│ 🔴 3 critical actions   │
│   [View →]             │
│                         │
│ ─────────────────────── │
│                         │
│ TODAY                   │
│ 1,24,847  consents      │
│ 23        erasures      │
│ 0         breaches      │
│                         │
│ ─────────────────────── │
│                         │
│ [Score] [Actions] [Risk]│
│ [Timeline]  [Reports]   │
└─────────────────────────┘
```
