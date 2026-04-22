# System Design — Regulatory War Room

**Version:** 1.0  
**Date:** 2026-04-22  

---

## 1. Architecture Overview

The Regulatory War Room is a **read-heavy aggregation service**. It does not generate or modify consent records, erasure workflows, or breach events — it only reads, aggregates, and presents data from all other modules.

This read-only design enables three things:
1. No impact on operational systems (War Room queries never slow down consent collection)
2. No PII risk (aggregated metrics only — no DP identifiers)
3. Simple scalability (read path can be cached aggressively)

### 1.1 Architectural Style: CQRS + Event Sourcing

The War Room implements CQRS (Command Query Responsibility Segregation):
- **Commands** happen in Consent Vault, Erasure Engine, Discovery Engine
- **Queries** happen in the War Room, against a purpose-built read model

```
Consent Vault ──► Kafka ──────────┐
Erasure Engine ──► Kafka ──────────┤──► War Room Event Processor
Breach Events ──► Kafka ───────────┘         │
                                              │ Materialise metrics
                                              ▼
                                    War Room Read Model
                                    (TimescaleDB / PostgreSQL)
                                              │
                                    War Room API ──► Dashboard ──► CXO
```

### 1.2 Data Contract: No PII Enters the War Room

Every event or API response that enters the War Room is pre-aggregated:
- Consent events → `{tenantId, purposeId, action, languageCode, timestamp}` (no dpIdentifierHash)
- Erasure events → `{tenantId, purposeId, status, duration, dfClass}` (no dpIdentifierHash)
- Breach events → `{tenantId, affectedCount, categories, notificationStatus}` (no dpIdentifierHash)

Aggregation is done in the source module. The War Room never has access to the DP identifier hash.

---

## 2. Component Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                     Regulatory War Room                               │
│                                                                       │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │                   Data Ingestion Layer                        │    │
│  │                                                               │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │    │
│  │  │ Kafka Events  │  │ Polling APIs │  │  Regulatory Feed  │   │    │
│  │  │ Consumer      │  │ (Module 3.1  │  │  (DPBI, MeitY,   │   │    │
│  │  │               │  │  3.2 metrics)│  │   court orders)  │   │    │
│  │  └──────┬────────┘  └──────┬───────┘  └────────┬─────────┘   │    │
│  │         └──────────────────┴──────────────┘    │              │    │
│  │                            │                   │              │    │
│  └────────────────────────────┼───────────────────┼──────────────┘    │
│                               │                   │                   │
│  ┌────────────────────────────▼───────────────────▼──────────────┐    │
│  │                 Metric Processing Engine                        │    │
│  │                                                                 │    │
│  │  ┌──────────────────┐  ┌─────────────────┐  ┌─────────────┐  │    │
│  │  │ Score Calculator  │  │ Risk Intelligence│  │ Regulatory  │  │    │
│  │  │ (compliance score │  │ Engine (penalty  │  │ AI Monitor  │  │    │
│  │  │  by component)   │  │  exposure, risk) │  │ (DPBI feed) │  │    │
│  │  └──────────────────┘  └─────────────────┘  └─────────────┘  │    │
│  │                                                                 │    │
│  │  ┌──────────────────────────────────────────────────────────┐  │    │
│  │  │  Action Intelligence Engine                               │  │    │
│  │  │  (generates action items from metric gaps + reg calendar) │  │    │
│  │  └──────────────────────────────────────────────────────────┘  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                │                                         │
│  ┌─────────────────────────────▼───────────────────────────────────┐    │
│  │                   Read Model (TimescaleDB)                       │    │
│  │  compliance_metrics  |  risk_scores  |  action_queue             │    │
│  │  regulatory_events   |  vendor_compliance  |  reports            │    │
│  └─────────────────────────────┬───────────────────────────────────┘    │
│                                │                                         │
│  ┌─────────────────────────────▼───────────────────────────────────┐    │
│  │                     API Layer                                     │    │
│  │   Dashboard API  |  Report Generator  |  Alert Service            │    │
│  └──────────────────────────────────────────────────────────────────┘    │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Compliance Score Engine

The compliance score is a weighted composite calculated every 4 hours.

```typescript
// score/calculator.ts
export class ComplianceScoreCalculator {
  
  async calculate(tenantId: string): Promise<ComplianceScore> {
    const [
      consentScore,
      erasureScore,
      breachReadinessScore,
      vendorScore,
      cmRegistrationScore,
      languageScore,
    ] = await Promise.all([
      this.calculateConsentScore(tenantId),
      this.calculateErasureScore(tenantId),
      this.calculateBreachReadinessScore(tenantId),
      this.calculateVendorScore(tenantId),
      this.calculateCMRegistrationScore(tenantId),
      this.calculateLanguageScore(tenantId),
    ]);
    
    const weighted = 
      consentScore * 0.25 +
      erasureScore * 0.20 +
      breachReadinessScore * 0.20 +
      vendorScore * 0.15 +
      cmRegistrationScore * 0.10 +
      languageScore * 0.10;
    
    return {
      tenantId,
      score: Math.round(weighted),
      components: {
        consent: { score: consentScore, weight: 0.25 },
        erasure: { score: erasureScore, weight: 0.20 },
        breachReadiness: { score: breachReadinessScore, weight: 0.20 },
        vendor: { score: vendorScore, weight: 0.15 },
        cmRegistration: { score: cmRegistrationScore, weight: 0.10 },
        language: { score: languageScore, weight: 0.10 },
      },
      gaps: this.identifyTopGaps([consentScore, erasureScore, ...]),
      calculatedAt: new Date().toISOString(),
    };
  }
  
  private async calculateConsentScore(tenantId: string): Promise<number> {
    const metrics = await this.consentMetrics.get(tenantId);
    
    // Score factors:
    // - What % of purposes have at least 80% consent rate?
    // - What % of language-required translations are approved?
    // - What % of consent withdrawals triggered erasure correctly?
    // - Are pre-checked boxes used anywhere? (instant 0)
    
    if (metrics.hasPreCheckedBoxes) return 0;
    
    const purposeCompleteness = metrics.purposesWithSufficientConsent / metrics.totalPurposes;
    const translationCompleteness = metrics.approvedTranslations / metrics.requiredTranslations;
    const withdrawalResponseRate = metrics.erasuresTriggeredOnWithdrawal / metrics.totalWithdrawals;
    
    return Math.round(
      purposeCompleteness * 0.5 +
      translationCompleteness * 0.3 +
      withdrawalResponseRate * 0.2
    ) * 100;
  }
  
  private async calculateBreachReadinessScore(tenantId: string): Promise<number> {
    const drill = await this.breachDrillMetrics.get(tenantId);
    
    // Score factors:
    // - Has a breach drill been run in the last 6 months? (-30 if not)
    // - In last drill: was CERT-In 6hr deadline met?
    // - In last drill: was DPBI 72hr deadline met?
    // - Is breach notification workflow configured?
    // - Are affected DP language templates configured?
    
    let score = 0;
    if (drill.workflowConfigured) score += 30;
    if (drill.lastDrillDate && daysSince(drill.lastDrillDate) < 180) score += 30;
    if (drill.lastDrillCertInMet) score += 20;
    if (drill.lastDrillDpbiMet) score += 10;
    if (drill.languageTemplatesConfigured >= 7) score += 10; // Phase 1 languages
    
    return score;
  }
}
```

---

## 4. Regulatory AI Monitor

### 4.1 Data Sources

```typescript
// regulatory/monitor.ts — sources scanned daily

const REGULATORY_SOURCES = [
  {
    id: 'dpbi_official',
    name: 'DPBI Official Website',
    url: 'https://www.dpb.gov.in',
    type: 'SCRAPER',
    sections: ['orders', 'circulars', 'faqs', 'press_releases'],
    updateFrequency: 'DAILY',
  },
  {
    id: 'meity_press',
    name: 'MeitY Press Releases',
    url: 'https://www.meity.gov.in/press-release',
    type: 'RSS_FEED',
    filterKeywords: ['DPDPA', 'DPDP', 'data protection', 'consent manager'],
    updateFrequency: 'DAILY',
  },
  {
    id: 'gazette_india',
    name: 'India Gazette (Official)',
    type: 'SCRAPER',
    filterKeywords: ['Digital Personal Data Protection'],
    updateFrequency: 'WEEKLY',
  },
];
```

### 4.2 Regulatory Change Processing Pipeline

```typescript
// regulatory/pipeline.ts

export async function processRegulatoryDocument(
  document: RegulatoryDocument
): Promise<RegulatoryAlert[]> {
  
  // Step 1: Classify document type (LLM — on-premise)
  const classification = await llm.classify(`
    Classify this regulatory document:
    Title: ${document.title}
    Summary: ${document.excerpt}
    
    Categories: RULE_CHANGE | GUIDANCE | ENFORCEMENT_ACTION | CLARIFICATION | DRAFT
    Return JSON: { type, confidence }
  `);
  
  // Step 2: Extract actionable obligations (LLM)
  const obligations = await llm.extract(`
    Extract all actionable obligations for a Registered Consent Manager from:
    ${document.fullText}
    
    For each obligation, extract:
    - requirement: what must be done
    - deadline: by when (if specified)
    - dpdpa_section: which section/rule
    - penalty: what penalty for non-compliance
    
    Return JSON array of obligations.
    Ground every obligation in the document text — do not hallucinate.
    Cite the paragraph you extracted each obligation from.
  `);
  
  // Step 3: Assess tenant-specific impact
  const alerts = [];
  for (const obligation of obligations) {
    const impact = await this.assessTenantImpact(obligation);
    if (impact.severity !== 'NOT_APPLICABLE') {
      alerts.push({
        source: document.url,   // Always linked to source
        obligation,
        impact,
        actionRequired: impact.severity !== 'LOW',
      });
    }
  }
  
  return alerts;
}
```

### 4.3 Hallucination Prevention

The AI Monitor is the highest-risk feature for errors (regulatory misinformation is worse than no information):

1. **Source citation required**: Every obligation must cite the paragraph it was extracted from
2. **Confidence threshold**: Obligations with < 0.75 confidence are flagged for human review, not auto-published
3. **Human review queue**: All HIGH impact alerts are reviewed by TrustStack's compliance team before tenant notification
4. **Version tracking**: Every published regulatory alert references the source document URL and version
5. **No extrapolation**: The LLM is prompted to not infer obligations not explicitly stated

---

## 5. Action Intelligence Engine

The Action Queue is not manually maintained — it is generated from gaps between current state and required state.

```typescript
// actions/generator.ts

const ACTION_RULES: ActionRule[] = [
  {
    id: 'MISSING_LANGUAGE_TRANSLATION',
    condition: (metrics) =>
      metrics.approvedTranslations < metrics.requiredTranslations,
    generate: (metrics) => ({
      title: `Approve translations for ${metrics.pendingLanguages.join(', ')}`,
      dueDate: nextMilestone('CM_REGISTRATION'), // Nov 2026
      priority: 'HIGH',
      owner: 'COMPLIANCE',
      dpdpaSection: 'Section 5, Rule 3',
      penaltyRisk: 'Consent without approved translation is invalid — Section 6 violation',
    }),
  },
  {
    id: 'ERASURE_OVERDUE',
    condition: (metrics) => metrics.overdueErasures > 0,
    generate: (metrics) => ({
      title: `${metrics.overdueErasures} erasure(s) overdue — immediate action required`,
      dueDate: 'IMMEDIATE',
      priority: 'CRITICAL',
      owner: 'DPO + ENGINEERING',
      dpdpaSection: 'Third Schedule + Section 12',
      penaltyRisk: `Up to ₹150Cr for each missed erasure obligation`,
    }),
  },
  {
    id: 'BREACH_DRILL_OVERDUE',
    condition: (metrics) => daysSince(metrics.lastBreachDrill) > 180,
    generate: (metrics) => ({
      title: 'Breach notification drill overdue (>6 months since last test)',
      dueDate: addDays(new Date(), 30),
      priority: 'HIGH',
      owner: 'CISO + DPO',
      dpdpaSection: 'Rule 7',
      penaltyRisk: 'Untested workflows risk missing 6hr CERT-In deadline in real breach',
    }),
  },
  {
    id: 'CM_REGISTRATION_GAP',
    condition: (metrics) => metrics.cmRegistrationProgress < 100 && daysUntil('2026-11-13') < 180,
    generate: (metrics) => ({
      title: `CM Registration ${metrics.cmRegistrationProgress}% complete — ${daysUntil('2026-11-13')} days remaining`,
      dueDate: '2026-11-13',
      priority: metrics.cmRegistrationProgress < 50 ? 'CRITICAL' : 'HIGH',
      owner: 'CEO + LEGAL',
      dpdpaSection: 'Rule 4',
      penaltyRisk: 'Cannot legally operate as Consent Manager without registration',
    }),
  },
];
```

---

## 6. Reporting Engine

### 6.1 Board Deck Generation

```typescript
// reporting/board-deck.ts

export async function generateBoardDeck(
  tenantId: string,
  options: ReportOptions
): Promise<Report> {
  
  const [score, metrics, risks, actions] = await Promise.all([
    complianceScore.get(tenantId),
    aggregateMetrics.get(tenantId, { period: '3m' }),
    riskEngine.get(tenantId),
    actionQueue.getCritical(tenantId),
  ]);
  
  // AI generates executive narrative (on-premise LLM)
  const narrative = await llm.generate(`
    Write a 2-paragraph executive summary for a CEO board presentation 
    about our DPDPA compliance status.
    
    Compliance score: ${score.score}/100
    Trend: ${score.trend}
    Top risks: ${risks.top3.map(r => r.description).join(', ')}
    Actions overdue: ${actions.overdue.length}
    
    Tone: confident but factual. Acknowledge gaps, emphasise progress.
    Language: ${options.language || 'en'}
  `);
  
  const slides = [
    buildComplianceScoreSlide(score),
    buildConsentMetricsSlide(metrics.consent),
    buildErasureMetricsSlide(metrics.erasure),
    buildRiskExposureSlide(risks),
    buildActionQueueSlide(actions),
    buildTimelineSlide(),
    buildNarrativeSlide(narrative),
  ];
  
  const report = {
    reportId: generateId(),
    type: 'BOARD_DECK',
    tenantId,
    generatedAt: new Date().toISOString(),
    slides,
    dataAsOf: metrics.calculatedAt,
    generatedBy: 'TrustStack Regulatory War Room v1.0',
  };
  
  // ECDSA sign the report (TrustStack master signing key)
  const signature = await kms.sign(canonicalise(report));
  
  return { ...report, signature, signingKeyId: TRUSTSTACK_MASTER_KEY_ARN };
}
```

### 6.2 Report Types

| Report | Format | Signed | Frequency | Use |
|--------|--------|--------|-----------|-----|
| Board Compliance Deck | PDF + PPTX | Yes | Monthly or on-demand | Board meetings |
| DPBI Annual Audit Report | DPBI XML + PDF | Yes | Annual | Regulatory submission |
| Consent Health Report | PDF + CSV | No | Weekly | DPO operational |
| Vendor Compliance Scorecard | PDF | Yes | Quarterly | Vendor management |
| Breach Readiness Assessment | PDF | Yes | Post-drill | CISO + DPBI |
| Penalty Exposure Report | PDF | No | On-demand | CFO + Board |

---

## 7. Technology Stack

| Component | Technology | Notes |
|-----------|-----------|-------|
| API | TypeScript ESM / Express | Same pattern as other modules |
| Read Model | TimescaleDB (PostgreSQL extension) | Time-series metrics, hypertables |
| Metric cache | Redis (ElastiCache) | 4hr TTL for score, 1hr for components |
| Event processing | Apache Kafka consumers | Multi-module event streams |
| Regulatory scraper | Python (Playwright + BeautifulSoup) | DPBI + MeitY scraping |
| AI Monitor | On-premise LLM (SageMaker, ap-south-1) | Shared with Erasure Engine |
| Report generation | Puppeteer (PDF) + python-pptx | Board deck generation |
| Report signing | AWS KMS (master key) | TrustStack-level attestation |
| Dashboard | React (web) + React Native (mobile) | CXOs check on phones |
| Alerts | WhatsApp Business + Email + Slack webhook | DPO preferred channel |

---

## 8. Data Freshness Strategy

| Metric | Freshness | Strategy |
|--------|-----------|---------|
| Compliance score | 4 hours | Kafka-triggered recalculation |
| Consent counts | 15 minutes | Kafka consumer aggregation |
| Erasure status | 15 minutes | Kafka consumer aggregation |
| Breach events | Real-time | Direct Kafka event |
| Vendor compliance | Weekly | Manual input + API polling |
| Regulatory changes | 24 hours | Scheduled scraper |
| Action queue | 4 hours | Regenerated with score |
| Report data | At generation time | Snapshot at click time |
