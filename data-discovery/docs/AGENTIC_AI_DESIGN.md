# TrustStack Module 2 — Agentic AI Design Specification

**Owner:** ML Platform Team  
**Status:** Draft v0.1  
**Date:** 2026-04-22  
**DPDPA Reference:** Section 2(g), Section 5, Section 6, Section 9, Section 16, Rule 3, Rule 7

---

## Table of Contents

1. [Overview](#overview)
2. [Discovery Agent Pipeline](#discovery-agent-pipeline)
   - [Architecture Overview](#architecture-overview)
   - [ConnectorAgent](#connectoragent)
   - [ScannerAgent](#scanneragent)
   - [ClassifierAgent](#classifieragent)
   - [RiskScorerAgent](#riskscoreragent)
   - [MapperAgent](#mapperagent)
   - [India-Specific PII Detection Patterns](#india-specific-pii-detection-patterns)
   - [Temporal Activity Wrapper Pattern](#temporal-activity-wrapper-pattern)
   - [Safety Constraints](#safety-constraints)
3. [NLP Contract Audit Pipeline](#nlp-contract-audit-pipeline)
   - [Pipeline Overview](#pipeline-overview)
   - [FileProcessor](#fileprocessor)
   - [OCRAgent](#ocragent)
   - [LanguageDetector](#languagedetector)
   - [TranslationAgent](#translationagent)
   - [ClauseExtractor](#clauseextractor)
   - [ComplianceChecker](#compliancechecker)
   - [GapAnalyzer](#gapanalyzer)
   - [DPAGenerator](#dpagenerator)
   - [Signer](#signer)
   - [Multi-Language Handling Strategy](#multi-language-handling-strategy)
4. [Model Specifications](#model-specifications)
5. [Evaluation and Quality](#evaluation-and-quality)

---

## Overview

Module 2 contains two AI agent systems. Both run entirely on-premise in `ap-south-1` (AWS Mumbai). No data — PII, contract text, or inferred findings — leaves India during inference.

**Why on-premise LLM (Section 16 rationale):**  
DPDPA Section 16 requires that DPBI can access consent logs even when processed on foreign infrastructure, and the Rules currently prohibit cross-border transfer of personal data without explicit DPBI pre-authorization. Cloud-hosted LLM APIs (OpenAI, Anthropic) route requests through US/EU data centres. Because the Discovery Engine reads raw vendor data that may contain personal data, and the Contract Audit Pipeline reads vendor contracts that often embed PII (party names, employee data, customer references), routing inference calls to foreign APIs would constitute unauthorized cross-border personal data transfer. All LLM inference therefore runs on AWS SageMaker endpoints in `ap-south-1`.

**System 1 — Discovery Agent Pipeline:**  
A 5-agent sequential pipeline that crawls connected vendor systems (AWS S3, Zoho CRM, Razorpay, Tally, WhatsApp Business, generic databases), finds hidden or unintentional PII, scores compliance risk, and builds a Data Flow Map graph. Each agent is a Temporal.io activity. LangGraph StateGraph is used internally by the Scanner and Classifier agents for multi-step LLM reasoning.

**System 2 — NLP Contract Audit Pipeline:**  
A 9-stage pipeline that ingests vendor contracts in any of the 22 Eighth Schedule languages, extracts clauses, checks them against the DPDPA compliance taxonomy, generates a gap report, and produces a compliant Data Processing Agreement (DPA) in the contract's source language, cryptographically signed with the tenant's KMS key.

**Dual-role prohibition (Section 2(g)):**  
The Discovery Engine operates in a strict **read-only, sandboxed** mode. It has zero access to the Consent Vault (SahmatOS). The scan result database is append-only. No finding is used to process personal data on behalf of the Data Principal — only to alert the Data Fiduciary. This architectural separation is enforced at the IAM level: the scan IAM role has no policy attachment to Consent Vault resources.

---

## Discovery Agent Pipeline

### Architecture Overview

```
ConnectorAgent → ScannerAgent → ClassifierAgent → RiskScorerAgent → MapperAgent
```

Each agent is a Temporal activity. The Temporal workflow sequences them, passing typed outputs as inputs to the next activity. A scan job for one connector (e.g., a Zoho CRM instance) runs as a single Temporal workflow instance. Multiple connectors for a tenant run as child workflows in parallel, with the MapperAgent as a terminal step that aggregates all child results into a single Data Flow Map update.

```
Temporal Workflow: DiscoveryScanWorkflow
├── Child: ScanConnectorWorkflow(zoho_crm)
│   ├── Activity: ConnectorAgent
│   ├── Activity: ScannerAgent        ← uses LangGraph internally
│   ├── Activity: ClassifierAgent     ← uses LangGraph internally
│   └── Activity: RiskScorerAgent
├── Child: ScanConnectorWorkflow(razorpay)
│   └── ... same chain
├── Child: ScanConnectorWorkflow(aws_s3_logs)
│   └── ... same chain
└── Activity: MapperAgent             ← runs after all children complete
```

**Typed data flow:**

```python
from dataclasses import dataclass, field
from enum import Enum
from typing import Optional, AsyncIterator
import datetime

class PIIEntityType(str, Enum):
    FULL_NAME = "FULL_NAME"
    EMAIL = "EMAIL"
    PHONE_NUMBER = "PHONE_NUMBER"
    AADHAAR_NUMBER = "AADHAAR_NUMBER"
    PAN_NUMBER = "PAN_NUMBER"
    PASSPORT_NUMBER = "PASSPORT_NUMBER"
    BANK_ACCOUNT = "BANK_ACCOUNT"
    CREDIT_CARD = "CREDIT_CARD"
    HEALTH_RECORD_ID = "HEALTH_RECORD_ID"
    BIOMETRIC_DATA = "BIOMETRIC_DATA"
    GPS_LOCATION = "GPS_LOCATION"
    IP_ADDRESS = "IP_ADDRESS"
    DEVICE_ID = "DEVICE_ID"
    BEHAVIORAL_PROFILE = "BEHAVIORAL_PROFILE"
    MINOR_DATA = "MINOR_DATA"
    IFSC_CODE = "IFSC_CODE"
    UPI_ID = "UPI_ID"
    INDIAN_POSTAL_CODE = "INDIAN_POSTAL_CODE"

class DPDPACategory(str, Enum):
    PERSONAL_IDENTITY = "PERSONAL_IDENTITY"   # Section 3 sensitive
    FINANCIAL = "FINANCIAL"                   # RBI compliance risk
    HEALTH = "HEALTH"                         # Section 3 sensitive
    BIOMETRIC = "BIOMETRIC"                   # Section 3 sensitive — highest risk
    LOCATION = "LOCATION"
    BEHAVIORAL = "BEHAVIORAL"
    MINOR = "MINOR"                           # Section 9 — parental consent
    CONTACT = "CONTACT"
    TECHNICAL = "TECHNICAL"

class RiskLevel(str, Enum):
    LOW = "LOW"
    MEDIUM = "MEDIUM"
    HIGH = "HIGH"

@dataclass
class ConnectorConfig:
    tenant_id: str
    connector_id: str
    vendor_type: str  # "zoho_crm" | "razorpay" | "aws_s3" | "tally" | "whatsapp_business" | "postgres" | "mysql"
    credentials_secret_arn: str  # AWS Secrets Manager ARN — never the credential itself
    scan_scope: "ScanScope"
    cross_border: bool = False
    requested_by: str = ""  # user_id of DF admin who triggered scan

@dataclass
class ScanScope:
    tables: list[str] = field(default_factory=list)      # empty = all
    s3_prefixes: list[str] = field(default_factory=list)
    log_lookback_days: int = 90
    max_records_full_scan: int = 1_000_000  # above this, switch to sampling
    sampling_fraction: float = 0.05

@dataclass
class ObjectRef:
    source_system: str
    object_type: str  # "table_row" | "s3_object" | "log_entry" | "api_record"
    object_id: str
    estimated_size_bytes: int
    is_log_or_backup: bool = False  # triggers × 1.8 risk multiplier

@dataclass
class DataBatch:
    connector_id: str
    batch_id: str
    records: list[dict]              # raw records — never persisted after scan activity completes
    object_refs: list[ObjectRef]
    source_metadata: dict

@dataclass
class PIIFinding:
    batch_id: str
    object_ref: ObjectRef
    entity_type: PIIEntityType
    value_hash: str                  # SHA-256 of raw value — NEVER the raw value
    context_field: str               # e.g. "column: customer_phone"
    confidence: float
    found_in_log_or_backup: bool

@dataclass
class ClassifiedFinding:
    finding: PIIFinding
    dpdpa_category: DPDPACategory
    regulatory_conflicts: list[str]  # e.g. ["RBI_RETENTION_CONFLICT", "HIPAA_EQUIVALENT"]

@dataclass
class DataFlowContext:
    connector_id: str
    vendor_type: str
    cross_border: bool
    consent_status: str              # "CONSENTED" | "NO_CONSENT_ON_RECORD" | "PARTIAL"
    is_processor: bool               # DF is processor, not controller

@dataclass
class RiskScore:
    connector_id: str
    vendor_type: str
    base_risk: float
    final_risk: float
    risk_level: RiskLevel
    penalty_exposure_cr: float       # ₹ crore
    multipliers_applied: list[str]
    findings_summary: dict[DPDPACategory, int]  # category → count
    scanned_at: datetime.datetime
```

---

### ConnectorAgent

**Purpose:** Authenticate to a vendor system, enumerate all objects/records in scope, and stream batches to the ScannerAgent. This is a pure I/O agent — it has no LLM component.

**Temporal activity signature:**

```python
# DPDPA Section 16: ConnectorAgent never calls any external API outside ap-south-1
@activity.defn(name="connector_agent")
async def connector_agent_activity(config: ConnectorConfig) -> list[str]:
    """Returns a list of batch_ids written to the scan job's ephemeral batch store.
    Batches are written to encrypted S3 (ap-south-1) with 24hr TTL.
    Raw records are NEVER written to the scan findings database."""
    plugin = ConnectorPluginRegistry.get(config.vendor_type)
    session = await plugin.authenticate(config)
    batch_ids: list[str] = []
    try:
        refs = [ref async for ref in plugin.list_objects(session, config.scan_scope)]
        for chunk in _chunk(refs, size=500):
            batch = await plugin.fetch_batch(session, chunk)
            batch_id = await _write_ephemeral_batch(batch, ttl_hours=24)
            batch_ids.append(batch_id)
            activity.heartbeat(f"batches_written={len(batch_ids)}")
    finally:
        await plugin.close(session)
    return batch_ids
```

**ConnectorPlugin interface (TypeScript — wraps Python agent via gRPC):**

```typescript
// DPDPA Section 16: All connector calls route through ap-south-1 VPC only.
// Credentials are retrieved from AWS Secrets Manager; they are NEVER logged.
interface ConnectorConfig {
  tenantId: string;
  connectorId: string;
  vendorType:
    | "zoho_crm"
    | "razorpay"
    | "aws_s3"
    | "tally"
    | "whatsapp_business"
    | "postgres"
    | "mysql";
  credentialsSecretArn: string; // Secrets Manager ARN only — never the credential value
  scanScope: ScanScope;
  crossBorder: boolean;
}

interface ConnectorSession {
  sessionId: string;
  vendorType: string;
  expiresAt: Date;
  // Opaque handle — never serialized to logs or DB
}

interface ScanScope {
  tables?: string[];          // empty = all tables
  s3Prefixes?: string[];
  logLookbackDays?: number;   // default: 90
  maxRecordsFullScan?: number; // default: 1_000_000; above → sampling
  samplingFraction?: number;  // default: 0.05
}

interface ObjectRef {
  sourceSystem: string;
  objectType: "table_row" | "s3_object" | "log_entry" | "api_record";
  objectId: string;
  estimatedSizeBytes: number;
  isLogOrBackup: boolean;     // triggers risk × 1.8 multiplier
}

interface DataBatch {
  connectorId: string;
  batchId: string;
  records: Record<string, unknown>[];  // raw records — ephemeral only
  objectRefs: ObjectRef[];
  sourceMetadata: Record<string, string>;
}

interface ConnectorPlugin {
  /**
   * Retrieve credentials from Secrets Manager and establish a session.
   * Never log or store credentials. Session handle is opaque.
   */
  authenticate(config: ConnectorConfig): Promise<ConnectorSession>;

  /**
   * Enumerate all objects in scope. Cursor-based for APIs, offset for DBs.
   * Implements rate limiting per vendor API limits.
   */
  listObjects(
    session: ConnectorSession,
    scope: ScanScope
  ): AsyncIterable<ObjectRef>;

  /**
   * Fetch a batch of records. Max 500 refs per call.
   * For datasets > maxRecordsFullScan: uniform random sampling at samplingFraction.
   */
  fetchBatch(
    session: ConnectorSession,
    refs: ObjectRef[]
  ): Promise<DataBatch>;

  /**
   * Release session resources and revoke ephemeral tokens.
   */
  close(session: ConnectorSession): Promise<void>;
}
```

**Key behaviors:**

| Behavior | Detail |
|---|---|
| Pagination | Cursor-based for REST APIs (Zoho, Razorpay); offset-based for SQL; multipart listing for S3 |
| Rate limiting | Respects `X-RateLimit-Remaining` headers; exponential backoff with jitter; configurable per vendor |
| Sampling | Full scan for datasets under `max_records_full_scan` (default 1M). Uniform random sample at `sampling_fraction` (default 5%) for larger datasets. Sample seed is stored in scan job record for reproducibility. |
| Log scope | Last 90 days by default. Configurable per scope. Logs older than 90 days scanned only if explicitly requested by DF admin. |
| Credentials | Retrieved from AWS Secrets Manager at activity start. Never passed through Temporal workflow history (use `activity.defn` input only for the ARN). Never logged. Session revoked in the `finally` block. |
| Heartbeating | Activity sends heartbeats every 30 seconds with batch count. Temporal uses this to detect stuck activities and apply retry policy. |

---

### ScannerAgent

**Purpose:** Detect PII entities in each data batch using a fine-tuned Mistral 7B model. Returns findings with SHA-256 hashes only — raw PII values are never stored.

**Temporal activity signature:**

```python
@activity.defn(name="scanner_agent")
async def scanner_agent_activity(batch_id: str) -> list[PIIFinding]:
    batch = await _read_ephemeral_batch(batch_id)
    graph = build_scanner_graph()
    result = await graph.ainvoke({"batch": batch, "findings": []})
    return result["findings"]
```

**LangGraph StateGraph:**

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, Annotated
import operator
import hashlib
import re

class ScannerState(TypedDict):
    batch: DataBatch
    chunks: list[dict]           # text chunks with field context
    raw_llm_output: list[dict]   # structured JSON from LLM
    findings: Annotated[list[PIIFinding], operator.add]

def preprocess(state: ScannerState) -> ScannerState:
    """Normalize records into flat text chunks with field-level context.
    Strips binary fields, base64 blobs, and non-text columns before LLM call."""
    chunks = []
    for record in state["batch"].records:
        for field_name, value in record.items():
            if isinstance(value, str) and len(value) > 0:
                chunks.append({
                    "field": field_name,
                    "text": value[:2000],  # truncate very long fields
                    "object_ref": record.get("_object_ref"),
                })
    return {**state, "chunks": chunks}

def chunk_text(state: ScannerState) -> ScannerState:
    """Group chunks into LLM-sized batches (max 20 fields per call, ~4K tokens)."""
    grouped = []
    for i in range(0, len(state["chunks"]), 20):
        grouped.append(state["chunks"][i:i + 20])
    return {**state, "chunks": grouped}

SCANNER_SYSTEM_PROMPT = """You are a PII detection engine for an Indian data compliance platform.

Analyze the provided fields and identify all personally identifiable information (PII) present.
Focus especially on India-specific PII: Aadhaar numbers (XXXX-XXXX-XXXX format), PAN numbers
(format: AAAAA0000A), UPI IDs (format: user@bank), IFSC codes (format: AAAA0000000),
Indian mobile numbers (10-digit starting with 6-9), and Indian postal codes (6-digit).

CRITICAL RULES:
1. Do NOT reproduce the actual PII value in your output. Use the literal string "REDACTED".
2. Provide a SHA-256 hash placeholder — the system will compute it separately.
3. Report confidence as a float between 0.0 and 1.0.
4. Only report entities with confidence >= 0.80.
5. Report the field name in the "context" field.

Respond ONLY with valid JSON matching this exact schema:
{
  "entities": [
    {
      "type": "<ENTITY_TYPE>",
      "value": "REDACTED",
      "context": "field: <field_name>",
      "confidence": <float>
    }
  ]
}

Valid entity types: FULL_NAME, EMAIL, PHONE_NUMBER, AADHAAR_NUMBER, PAN_NUMBER,
PASSPORT_NUMBER, BANK_ACCOUNT, CREDIT_CARD, HEALTH_RECORD_ID, BIOMETRIC_DATA,
GPS_LOCATION, IP_ADDRESS, DEVICE_ID, BEHAVIORAL_PROFILE, MINOR_DATA,
IFSC_CODE, UPI_ID, INDIAN_POSTAL_CODE.

If no PII is found, return: {"entities": []}"""

async def pii_detect(state: ScannerState) -> ScannerState:
    """Call Mistral 7B SageMaker endpoint for each chunk group."""
    raw_outputs = []
    for chunk_group in state["chunks"]:
        user_content = "\n".join(
            f"[field: {c['field']}] {c['text']}" for c in chunk_group
        )
        response = await sagemaker_invoke(
            endpoint_name="truststack-mistral7b-pii-v2",
            system=SCANNER_SYSTEM_PROMPT,
            user=user_content,
            max_tokens=1024,
            temperature=0.0,   # deterministic output for compliance audit trail
        )
        raw_outputs.append({
            "llm_response": response,
            "chunk_group": chunk_group,
        })
    return {**state, "raw_llm_output": raw_outputs}

def extract_entities(state: ScannerState) -> ScannerState:
    """Parse LLM JSON output. Reject malformed responses. Apply regex validation."""
    findings: list[PIIFinding] = []
    for output in state["raw_llm_output"]:
        try:
            parsed = _safe_parse_json(output["llm_response"])
            entities = parsed.get("entities", [])
        except ValueError:
            # Log parse failure (no PII logged) and continue — fail open for safety
            _log_parse_failure(output["llm_response"][:100])
            continue

        for entity in entities:
            if entity.get("confidence", 0) < 0.80:
                continue
            entity_type = entity.get("type")
            if entity_type not in PIIEntityType.__members__:
                continue

            # Find the matching chunk to get the original value for hashing
            field_name = entity.get("context", "").replace("field: ", "")
            original_value = _find_original_value(
                output["chunk_group"], field_name
            )
            if original_value is None:
                continue

            # Apply regex validation before accepting the finding
            if not _validate_entity_with_regex(entity_type, field_name, original_value):
                continue

            # CRITICAL: Hash the value — NEVER store raw PII
            # DPDPA Section 5: Purpose limitation — raw values serve no compliance purpose
            value_hash = "sha256:" + hashlib.sha256(
                original_value.encode("utf-8")
            ).hexdigest()

            # Determine if this came from a log or backup object_ref
            object_ref = _find_object_ref(output["chunk_group"], field_name)
            findings.append(PIIFinding(
                batch_id=state["batch"].batch_id,
                object_ref=object_ref,
                entity_type=PIIEntityType(entity_type),
                value_hash=value_hash,
                context_field=field_name,
                confidence=entity["confidence"],
                found_in_log_or_backup=object_ref.is_log_or_backup if object_ref else False,
            ))
    return {**state, "findings": findings}

def hash_values(state: ScannerState) -> ScannerState:
    """Deduplicate findings by value_hash within this batch.
    Same hash appearing in multiple fields is still one entity instance."""
    seen_hashes: set[str] = set()
    deduped: list[PIIFinding] = []
    for finding in state["findings"]:
        key = f"{finding.value_hash}:{finding.entity_type}"
        if key not in seen_hashes:
            seen_hashes.add(key)
            deduped.append(finding)
    return {**state, "findings": deduped}

def emit_findings(state: ScannerState) -> ScannerState:
    """Final node — findings list is returned to Temporal activity."""
    return state

def build_scanner_graph() -> StateGraph:
    graph = StateGraph(ScannerState)
    graph.add_node("preprocess", preprocess)
    graph.add_node("chunk_text", chunk_text)
    graph.add_node("pii_detect", pii_detect)
    graph.add_node("extract_entities", extract_entities)
    graph.add_node("hash_values", hash_values)
    graph.add_node("emit_findings", emit_findings)

    graph.set_entry_point("preprocess")
    graph.add_edge("preprocess", "chunk_text")
    graph.add_edge("chunk_text", "pii_detect")
    graph.add_edge("pii_detect", "extract_entities")
    graph.add_edge("extract_entities", "hash_values")
    graph.add_edge("hash_values", "emit_findings")
    graph.add_edge("emit_findings", END)
    return graph.compile()
```

**Structured LLM output format:**

```json
{
  "entities": [
    {
      "type": "PHONE_NUMBER",
      "value": "REDACTED",
      "context": "field: customer_phone",
      "confidence": 0.97
    },
    {
      "type": "AADHAAR_NUMBER",
      "value": "REDACTED",
      "context": "field: kyc_id",
      "confidence": 0.99
    },
    {
      "type": "UPI_ID",
      "value": "REDACTED",
      "context": "field: payment_ref",
      "confidence": 0.94
    }
  ]
}
```

---

### ClassifierAgent

**Purpose:** Map PII entity types to DPDPA data categories, detect regulatory conflicts, and apply Section 3 sensitive data flags.

**Temporal activity signature:**

```python
@activity.defn(name="classifier_agent")
async def classifier_agent_activity(findings: list[PIIFinding]) -> list[ClassifiedFinding]:
    graph = build_classifier_graph()
    result = await graph.ainvoke({"findings": findings, "classified": []})
    return result["classified"]
```

**LangGraph StateGraph:**

```python
class ClassifierState(TypedDict):
    findings: list[PIIFinding]
    classified: Annotated[list[ClassifiedFinding], operator.add]

# Static classification map — no LLM required for primary mapping
ENTITY_TO_DPDPA_CATEGORY: dict[PIIEntityType, DPDPACategory] = {
    PIIEntityType.AADHAAR_NUMBER:    DPDPACategory.PERSONAL_IDENTITY,
    PIIEntityType.PAN_NUMBER:        DPDPACategory.PERSONAL_IDENTITY,
    PIIEntityType.PASSPORT_NUMBER:   DPDPACategory.PERSONAL_IDENTITY,
    PIIEntityType.BANK_ACCOUNT:      DPDPACategory.FINANCIAL,
    PIIEntityType.CREDIT_CARD:       DPDPACategory.FINANCIAL,
    PIIEntityType.IFSC_CODE:         DPDPACategory.FINANCIAL,
    PIIEntityType.HEALTH_RECORD_ID:  DPDPACategory.HEALTH,
    PIIEntityType.BIOMETRIC_DATA:    DPDPACategory.BIOMETRIC,
    PIIEntityType.GPS_LOCATION:      DPDPACategory.LOCATION,
    PIIEntityType.BEHAVIORAL_PROFILE:DPDPACategory.BEHAVIORAL,
    PIIEntityType.MINOR_DATA:        DPDPACategory.MINOR,
    PIIEntityType.EMAIL:             DPDPACategory.CONTACT,
    PIIEntityType.PHONE_NUMBER:      DPDPACategory.CONTACT,
    PIIEntityType.FULL_NAME:         DPDPACategory.CONTACT,
    PIIEntityType.DEVICE_ID:         DPDPACategory.TECHNICAL,
    PIIEntityType.IP_ADDRESS:        DPDPACategory.TECHNICAL,
    PIIEntityType.UPI_ID:            DPDPACategory.FINANCIAL,
    PIIEntityType.INDIAN_POSTAL_CODE:DPDPACategory.LOCATION,
}

SECTION_3_SENSITIVE = {
    DPDPACategory.PERSONAL_IDENTITY,
    DPDPACategory.HEALTH,
    DPDPACategory.BIOMETRIC,
    DPDPACategory.FINANCIAL,  # RBI Act + DPDPA overlap
    DPDPACategory.MINOR,      # Section 9 — special treatment
}

def classify_findings(state: ClassifierState) -> ClassifierState:
    """Map each PIIFinding to its DPDPA category using the static map."""
    classified: list[ClassifiedFinding] = []
    for finding in state["findings"]:
        category = ENTITY_TO_DPDPA_CATEGORY.get(
            finding.entity_type, DPDPACategory.CONTACT
        )
        conflicts = _detect_regulatory_conflicts(finding.entity_type, category)
        classified.append(ClassifiedFinding(
            finding=finding,
            dpdpa_category=category,
            regulatory_conflicts=conflicts,
        ))
    return {**state, "classified": classified}

def _detect_regulatory_conflicts(
    entity_type: PIIEntityType,
    category: DPDPACategory,
) -> list[str]:
    """Detect conflicts between DPDPA rules and other Indian regulatory regimes."""
    conflicts: list[str] = []

    # FINANCIAL data: RBI retention rules may override DPDPA erasure timelines
    # Reference: RBI Master Direction on KYC (2016), PMLA 2002 — 10yr retention
    if category == DPDPACategory.FINANCIAL:
        conflicts.append("RBI_RETENTION_CONFLICT")

    # HEALTH data: No HIPAA in India, but draft DISHA 2022 overlaps with DPDPA
    # If DISHA is enacted, health data may require additional consent provisions
    if category == DPDPACategory.HEALTH:
        conflicts.append("DISHA_OVERLAP_WATCH")

    # AADHAAR data: Aadhaar Act 2016 imposes additional restrictions on storage
    # DPDPA cannot override Aadhaar Act — stricter rule applies
    if entity_type == PIIEntityType.AADHAAR_NUMBER:
        conflicts.append("AADHAAR_ACT_SUPERSEDES")

    # MINOR data: Section 9 requires verifiable parental consent
    # AND prohibits targeted advertising and profiling of minors
    if category == DPDPACategory.MINOR:
        conflicts.append("SECTION_9_PARENTAL_CONSENT_REQUIRED")
        conflicts.append("MINOR_PROFILING_PROHIBITED")

    return conflicts

def flag_section3(state: ClassifierState) -> ClassifierState:
    """LLM-assisted step: for BEHAVIORAL and ambiguous types, ask the model
    whether the data qualifies as 'sensitive personal data' under Section 3.
    Static categories above are already correctly mapped; this step handles
    edge cases like a 'notes' field that contains health information."""
    edge_cases = [
        cf for cf in state["classified"]
        if cf.finding.entity_type in (PIIEntityType.BEHAVIORAL_PROFILE,)
        and cf.finding.confidence < 0.92
    ]
    if not edge_cases:
        return state

    # Only invoke LLM for ambiguous cases to minimize inference cost
    prompt = _build_section3_classification_prompt(edge_cases)
    llm_verdicts = _invoke_classifier_llm(prompt)
    updated = _apply_llm_verdicts(state["classified"], llm_verdicts)
    return {**state, "classified": updated}

def build_classifier_graph() -> StateGraph:
    graph = StateGraph(ClassifierState)
    graph.add_node("classify_findings", classify_findings)
    graph.add_node("flag_section3", flag_section3)

    graph.set_entry_point("classify_findings")
    graph.add_edge("classify_findings", "flag_section3")
    graph.add_edge("flag_section3", END)
    return graph.compile()
```

**Section 3 sensitive data classification (DPDPA):**

| DPDPA Category | Section 3 Sensitive? | Additional Regulatory Overlay |
|---|---|---|
| BIOMETRIC | Yes — highest risk | Biometric data cannot be shared without explicit consent per Section 3 |
| HEALTH | Yes | Potential overlap with draft DISHA 2022 |
| PERSONAL_IDENTITY | Yes (Aadhaar, PAN) | Aadhaar Act 2016 additionally restricts storage |
| FINANCIAL | Yes | RBI KYC/PMLA retention may override DPDPA erasure |
| MINOR | Yes — Section 9 | Profiling and targeted advertising absolutely prohibited |
| LOCATION | No | But GPS precision < 100m may qualify as sensitive in draft Rules |
| BEHAVIORAL | No (usually) | May qualify if inferred from health/financial data |
| CONTACT | No | — |
| TECHNICAL | No | — |

---

### RiskScorerAgent

**Purpose:** Score each connector/data flow for DPDPA compliance risk. No LLM — deterministic formula applied to ClassifiedFinding counts and DataFlowContext flags.

**Temporal activity signature:**

```python
@activity.defn(name="risk_scorer_agent")
async def risk_scorer_agent_activity(
    classified_findings: list[ClassifiedFinding],
    context: DataFlowContext,
) -> RiskScore:
    return _compute_risk_score(classified_findings, context)
```

**Risk formula:**

```python
CATEGORY_WEIGHTS: dict[DPDPACategory, float] = {
    DPDPACategory.BIOMETRIC:          10.0,
    DPDPACategory.MINOR:               9.0,
    DPDPACategory.HEALTH:              8.0,
    DPDPACategory.FINANCIAL:           7.0,
    DPDPACategory.PERSONAL_IDENTITY:   6.0,
    DPDPACategory.BEHAVIORAL:          5.0,
    DPDPACategory.LOCATION:            4.0,
    DPDPACategory.CONTACT:             3.0,
    DPDPACategory.TECHNICAL:           2.0,
}

def _compute_risk_score(
    findings: list[ClassifiedFinding],
    ctx: DataFlowContext,
) -> RiskScore:
    # Count findings per category
    counts: dict[DPDPACategory, int] = {}
    found_in_log_or_backup = False
    for cf in findings:
        cat = cf.dpdpa_category
        counts[cat] = counts.get(cat, 0) + 1
        if cf.finding.found_in_log_or_backup:
            found_in_log_or_backup = True

    # Base risk: weighted sum
    base_risk = sum(
        CATEGORY_WEIGHTS[cat] * count
        for cat, count in counts.items()
    )

    multipliers_applied: list[str] = []

    # Cross-border multiplier (DPDPA Section 16)
    # Only applies when cross_border is True AND there is no DPBI authorization on record
    # Authorization status is checked via Consent Vault read-only API (policy lookup only)
    m = 1.0
    if ctx.cross_border and not _has_dpbi_authorization(ctx.connector_id):
        m *= 2.0
        multipliers_applied.append("CROSS_BORDER_NO_DPBI_AUTH × 2.0")

    # No consent on record (DPDPA Section 6 — consent is the only lawful basis
    # for advertising/analytics/marketing for most SMEs)
    if ctx.consent_status == "NO_CONSENT_ON_RECORD":
        m *= 1.5
        multipliers_applied.append("NO_CONSENT_ON_RECORD × 1.5")

    # Unintentional storage in logs or backups
    if found_in_log_or_backup:
        m *= 1.8
        multipliers_applied.append("FOUND_IN_LOGS_OR_BACKUPS × 1.8")

    # Regulatory retention exemption: RBI/GST/SEBI mandate may legitimately require
    # retention even after purpose fulfillment — reduce risk for these cases
    if "RBI_RETENTION_CONFLICT" in _get_all_conflicts(findings):
        m *= 0.5
        multipliers_applied.append("REGULATORY_RETENTION_APPLIES × 0.5")

    final_risk = base_risk * m

    # Penalty exposure estimate
    # DPDPA penalty schedule: up to ₹250Cr for breach of consent obligations
    # Heuristic: HIGH risk → up to ₹50Cr exposure; MEDIUM → ₹10Cr; LOW → ₹1Cr
    # These are estimates for DF awareness only — not legal advice
    risk_level = (
        RiskLevel.HIGH if final_risk > 50
        else RiskLevel.MEDIUM if final_risk > 20
        else RiskLevel.LOW
    )
    penalty_exposure_cr = {
        RiskLevel.HIGH: min(50.0, final_risk * 0.8),
        RiskLevel.MEDIUM: min(10.0, final_risk * 0.4),
        RiskLevel.LOW: min(1.0, final_risk * 0.1),
    }[risk_level]

    return RiskScore(
        connector_id=ctx.connector_id,
        vendor_type=ctx.vendor_type,
        base_risk=base_risk,
        final_risk=final_risk,
        risk_level=risk_level,
        penalty_exposure_cr=penalty_exposure_cr,
        multipliers_applied=multipliers_applied,
        findings_summary=counts,
        scanned_at=datetime.datetime.utcnow(),
    )
```

**Worked example:**

A Zoho CRM connector for an e-commerce DF with no cross-border data and partial consent:

| Category | Count | Weight | Subtotal |
|---|---|---|---|
| PERSONAL_IDENTITY (Aadhaar, PAN in KYC table) | 3 | 6.0 | 18.0 |
| FINANCIAL (bank account in orders) | 12 | 7.0 | 84.0 |
| CONTACT (email, phone) | 150 | 3.0 | 450.0 |
| TECHNICAL (device IDs) | 40 | 2.0 | 80.0 |

`BaseRisk = 18 + 84 + 450 + 80 = 632`

Multipliers: NO_CONSENT_ON_RECORD × 1.5; FOUND_IN_LOGS_OR_BACKUPS × 1.8 (phone numbers found in CRM activity log)

`FinalRisk = 632 × 1.5 × 1.8 = 1,706.4` → `HIGH`

`PenaltyExposureCr = min(50, 1706.4 × 0.8) = ₹50Cr` (capped at schedule maximum)

---

### MapperAgent

**Purpose:** Aggregate risk scores from all connector scans into a single Data Flow Map graph. Persists to PostgreSQL. Runs after all child ScanConnectorWorkflows complete.

**Temporal activity signature:**

```python
@activity.defn(name="mapper_agent")
async def mapper_agent_activity(
    risk_scores: list[RiskScore],
    scan_job_id: str,
    tenant_id: str,
) -> str:
    """Returns the updated data_flow_map_id."""
    return await _upsert_data_flow_map(risk_scores, scan_job_id, tenant_id)
```

**Graph schema (PostgreSQL):**

```sql
-- DPDPA Section 5: Data flow map is internal compliance tooling only.
-- No personal data values are stored in the graph — only structural metadata.

CREATE TABLE dfm_nodes (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id    UUID NOT NULL,
    system_name  TEXT NOT NULL,
    system_type  TEXT NOT NULL CHECK (system_type IN (
                     'DATA_PRINCIPAL_SOURCE', 'INTERNAL_SYSTEM',
                     'VENDOR_SYSTEM', 'BACKUP_STORE',
                     'LOG_SYSTEM', 'EXTERNAL'
                 )),
    vendor_type  TEXT,
    cross_border BOOLEAN NOT NULL DEFAULT FALSE,
    first_seen   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_seen    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE dfm_edges (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id         UUID NOT NULL,
    source_node_id    UUID NOT NULL REFERENCES dfm_nodes(id),
    dest_node_id      UUID NOT NULL REFERENCES dfm_nodes(id),
    data_categories   TEXT[] NOT NULL,        -- DPDPACategory enum values
    risk_level        TEXT NOT NULL,          -- RiskLevel enum
    risk_score        FLOAT NOT NULL,
    penalty_exposure_cr FLOAT NOT NULL,
    volume_estimate   INTEGER,               -- rough record count
    is_unintentional_leak BOOLEAN NOT NULL DEFAULT FALSE,
    -- ^ CRITICAL: set to TRUE when PII found in logs/backups without declared purpose
    scan_job_id       UUID NOT NULL,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_dfm_edges_tenant ON dfm_edges(tenant_id);
CREATE INDEX idx_dfm_edges_unintentional ON dfm_edges(tenant_id, is_unintentional_leak)
    WHERE is_unintentional_leak = TRUE;
-- The unintentional leak index is used by the DF dashboard to surface critical flags
```

**UNINTENTIONAL_LEAK edge semantics:**

When a ScannerAgent finds PII entity types (e.g., `AADHAAR_NUMBER`) in an `ObjectRef` where `is_log_or_backup = true`, and the data category is `PERSONAL_IDENTITY` or `BIOMETRIC`, the MapperAgent creates an edge with `is_unintentional_leak = TRUE`. The DF dashboard surfaces these edges as critical compliance alerts. This pattern most commonly occurs when:

- Application logs contain customer Aadhaar numbers in error stack traces
- Database backups contain KYC documents copied without purpose limitation
- WhatsApp webhook logs cache message bodies containing phone numbers and names

---

### India-Specific PII Detection Patterns

The ScannerAgent's regex validation layer (`_validate_entity_with_regex`) uses the following patterns to confirm LLM entity detections before emitting findings. These patterns run after the LLM output is parsed and reduce false positives.

```python
import re

INDIA_PII_PATTERNS: dict[PIIEntityType, list[re.Pattern]] = {
    PIIEntityType.AADHAAR_NUMBER: [
        # Aadhaar: 12 digits, optionally formatted as XXXX-XXXX-XXXX or XXXX XXXX XXXX
        # First digit cannot be 0 or 1 (UIDAI rule)
        re.compile(r'\b[2-9]\d{3}[\s-]?\d{4}[\s-]?\d{4}\b'),
    ],
    PIIEntityType.PAN_NUMBER: [
        # PAN: 5 alpha + 4 numeric + 1 alpha, always uppercase
        # 4th character indicates entity type: P=individual, C=company, etc.
        re.compile(r'\b[A-Z]{5}[0-9]{4}[A-Z]\b'),
    ],
    PIIEntityType.PASSPORT_NUMBER: [
        # Indian passport: 1 alpha + 7 digits
        re.compile(r'\b[A-Z][0-9]{7}\b'),
    ],
    PIIEntityType.IFSC_CODE: [
        # IFSC: 4 alpha (bank code) + 0 + 6 alphanumeric (branch code)
        re.compile(r'\b[A-Z]{4}0[A-Z0-9]{6}\b'),
    ],
    PIIEntityType.UPI_ID: [
        # UPI VPA: local-part @ PSP handle
        # PSP handles: okaxis, oksbi, okicici, okhdfcbank, ybl, ibl, axl, etc.
        re.compile(
            r'\b[\w.\-+]+@(okaxis|oksbi|okicici|okhdfcbank|ybl|ibl|axl|'
            r'upi|paytm|apl|razorypay|icici|hdfcbank|sbi|axisbank|'
            r'kotak|indus|federal|jsb|rbl|dbs|sc|citi)\b',
            re.IGNORECASE
        ),
    ],
    PIIEntityType.PHONE_NUMBER: [
        # Indian mobile: 10 digits starting with 6, 7, 8, or 9
        # Optionally prefixed with +91, 0, or 91
        re.compile(r'\b(?:\+91|91|0)?[6-9]\d{9}\b'),
    ],
    PIIEntityType.BANK_ACCOUNT: [
        # Indian bank account: 9 to 18 digits (no fixed format)
        # Context-dependent — requires field name heuristic (field contains "account", "acc", "acct")
        re.compile(r'\b\d{9,18}\b'),
    ],
    PIIEntityType.INDIAN_POSTAL_CODE: [
        # PIN code: exactly 6 digits, first digit 1-8 (no 0 or 9 in series)
        re.compile(r'\b[1-8][0-9]{5}\b'),
    ],
    PIIEntityType.EMAIL: [
        re.compile(r'\b[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}\b'),
    ],
    PIIEntityType.IP_ADDRESS: [
        re.compile(r'\b(?:\d{1,3}\.){3}\d{1,3}\b'),
        re.compile(r'\b(?:[0-9a-fA-F]{1,4}:){7}[0-9a-fA-F]{1,4}\b'),  # IPv6
    ],
    PIIEntityType.CREDIT_CARD: [
        # Luhn-valid 13–19 digit sequences, major card networks
        re.compile(r'\b(?:4[0-9]{12}(?:[0-9]{3})?'     # Visa
                   r'|5[1-5][0-9]{14}'                  # Mastercard
                   r'|3[47][0-9]{13}'                   # Amex
                   r'|6(?:011|5[0-9]{2})[0-9]{12})\b'), # Discover/RuPay prefix
    ],
}

# Field name heuristics for context-dependent types
FIELD_NAME_HEURISTICS: dict[PIIEntityType, list[str]] = {
    PIIEntityType.BANK_ACCOUNT: [
        "account", "acc", "acct", "bank_acc", "settlement", "beneficiary"
    ],
    PIIEntityType.BIOMETRIC_DATA: [
        "fingerprint", "face_id", "retina", "biometric", "iris", "thumb"
    ],
    PIIEntityType.MINOR_DATA: [
        "dob", "birth_date", "age", "guardian", "parent", "minor", "child"
    ],
    PIIEntityType.HEALTH_RECORD_ID: [
        "health", "medical", "diagnosis", "prescription", "abha", "uhid", "mrn"
    ],
}

def _validate_entity_with_regex(
    entity_type: str,
    field_name: str,
    original_value: str,
) -> bool:
    """Returns True if the entity passes regex validation.
    For types without regex patterns, falls back to field name heuristic."""
    pii_type = PIIEntityType(entity_type)
    patterns = INDIA_PII_PATTERNS.get(pii_type)

    if patterns:
        return any(p.search(original_value) for p in patterns)

    # Context-dependent types: use field name heuristic
    heuristics = FIELD_NAME_HEURISTICS.get(pii_type, [])
    if heuristics:
        return any(h in field_name.lower() for h in heuristics)

    # No validation possible — accept LLM verdict (confidence already >= 0.80)
    return True
```

---

### Temporal Activity Wrapper Pattern

All five agents follow this Temporal activity wrapper pattern. The wrapper provides:
- Durable execution (automatic retry on transient failures)
- Heartbeating (detect stuck activities)
- Structured logging with PII-safe fields only
- Timeout enforcement

```typescript
// TypeScript Temporal worker — dispatches to Python activities via gRPC
import { proxyActivities, sleep, log } from "@temporalio/workflow";
import type * as activities from "./discovery-activities";

const {
  connectorAgentActivity,
  scannerAgentActivity,
  classifierAgentActivity,
  riskScorerAgentActivity,
  mapperAgentActivity,
} = proxyActivities<typeof activities>({
  startToCloseTimeout: "30 minutes",
  heartbeatTimeout: "90 seconds",
  retry: {
    maximumAttempts: 3,
    initialInterval: "10 seconds",
    backoffCoefficient: 2,
    maximumInterval: "5 minutes",
    nonRetryableErrorTypes: [
      "ConnectorAuthenticationError",   // bad credentials — don't retry, alert DF admin
      "ScanScopeViolationError",        // attempted access outside granted scope
      "DualRoleViolationError",         // attempted access to Consent Vault data
    ],
  },
});

// DPDPA Section 16: DiscoveryScanWorkflow runs entirely in ap-south-1.
// No Temporal activity in this workflow may call endpoints outside the VPC.
export async function scanConnectorWorkflow(
  config: ConnectorConfig
): Promise<RiskScore> {
  log.info("scan_connector_start", {
    connector_id: config.connectorId,
    vendor_type: config.vendorType,
    tenant_id: config.tenantId,
    // NOTE: never log credentials_secret_arn or any credential value
  });

  // Step 1: ConnectorAgent — enumerate and batch records
  const batchIds = await connectorAgentActivity(config);

  log.info("connector_agent_complete", {
    connector_id: config.connectorId,
    batch_count: batchIds.length,
  });

  // Steps 2 & 3: Scanner + Classifier — run per batch, aggregate results
  const allClassifiedFindings: ClassifiedFinding[] = [];
  for (const batchId of batchIds) {
    const piiFindings = await scannerAgentActivity(batchId);
    const classified = await classifierAgentActivity(piiFindings);
    allClassifiedFindings.push(...classified);
  }

  log.info("scan_classifier_complete", {
    connector_id: config.connectorId,
    finding_count: allClassifiedFindings.length,
    // NOTE: never log finding values or hashes — only counts
  });

  // Step 4: RiskScorer
  const flowContext: DataFlowContext = {
    connectorId: config.connectorId,
    vendorType: config.vendorType,
    crossBorder: config.crossBorder,
    consentStatus: await fetchConsentSummary(config.tenantId, config.connectorId),
    isProcessor: false,
  };
  const riskScore = await riskScorerAgentActivity(allClassifiedFindings, flowContext);

  log.info("risk_score_complete", {
    connector_id: config.connectorId,
    risk_level: riskScore.riskLevel,
    final_risk: riskScore.finalRisk,
    // NOTE: penalty_exposure_cr logged for compliance dashboard, not PII
    penalty_exposure_cr: riskScore.penaltyExposureCr,
  });

  return riskScore;
}

export async function discoveryScanWorkflow(
  tenantId: string,
  scanJobId: string,
  connectorConfigs: ConnectorConfig[]
): Promise<string> {
  // Run all connector scans in parallel as child workflows
  const childHandles = await Promise.all(
    connectorConfigs.map((config) =>
      startChild(scanConnectorWorkflow, {
        args: [config],
        workflowId: `scan-${scanJobId}-${config.connectorId}`,
      })
    )
  );

  const riskScores = await Promise.all(
    childHandles.map((h) => h.result())
  );

  // MapperAgent runs once after all children complete
  const dataFlowMapId = await mapperAgentActivity(riskScores, scanJobId, tenantId);

  log.info("discovery_scan_complete", {
    scan_job_id: scanJobId,
    tenant_id: tenantId,
    connectors_scanned: connectorConfigs.length,
    data_flow_map_id: dataFlowMapId,
  });

  return dataFlowMapId;
}
```

---

### Safety Constraints

These constraints are enforced at multiple layers (IAM, code, network) — not just by convention.

| Constraint | Enforcement |
|---|---|
| **No raw PII stored** | ScannerAgent only writes SHA-256 hashes to the findings DB. Ephemeral batch store (S3) has 24hr TTL and is encrypted with a per-scan-job KMS key that is deleted after TTL. |
| **Read-only connectors** | ConnectorPlugin IAM role has only `s3:GetObject`, `s3:ListBucket`, `zoho:read:*`, `rds:Select`. Write permissions are absent from the IAM policy. This is verified in the CI pipeline via `aws iam simulate-principal-policy`. |
| **No cross-account calls during scan** | ScannerAgent runs in a dedicated VPC with no internet gateway. Outbound traffic is restricted to the vendor API endpoints listed in the DF's connector config. Any outbound call not matching the allowlist is blocked by Security Group egress rules. |
| **No Consent Vault access** | The scan IAM role has an explicit `Deny` on all `sahmat:*` API actions and all Consent Vault S3 bucket ARNs. This enforces the dual-role prohibition (Section 2(g)). |
| **Credentials never in Temporal history** | `ConnectorConfig` contains only the `credentials_secret_arn`. The actual credential is fetched inside the Temporal activity using the activity's execution role. It is never returned or logged. |
| **Scan scope validation** | `ScanScopeViolationError` (non-retryable) is raised if the ConnectorAgent attempts to access an object outside the declared `ScanScope`. Scope is validated before any API call. |
| **PII-safe logging** | All structured log fields are on an allowlist. Any log call that includes a key matching `INDIA_PII_PATTERNS` is rejected at the logger middleware level. |
| **7-year finding retention** | Scan findings (hashes + categories, not values) are retained for 7 years per Rule 4 CM registration requirement. The findings DB table is append-only with no `DELETE` or `UPDATE` permissions on the application role. |

---

## NLP Contract Audit Pipeline

### Pipeline Overview

```
FileProcessor → OCRAgent (if needed) → LanguageDetector → TranslationAgent
    → ClauseExtractor → ComplianceChecker → GapAnalyzer → DPAGenerator → Signer
```

The pipeline processes one contract document per Temporal workflow instance. Stages 1–4 (ingestion) and stages 5–7 (analysis) each run as a single Temporal activity. DPAGenerator and Signer are separate activities. All LLM inference is on-premise Mistral 7B in `ap-south-1`.

**Typed data structures:**

```python
from dataclasses import dataclass, field
from enum import Enum
from typing import Optional

class ClauseType(str, Enum):
    PARTIES = "PARTIES"
    PURPOSE = "PURPOSE"
    DATA_CATEGORIES = "DATA_CATEGORIES"
    RETENTION = "RETENTION"
    SHARING = "SHARING"
    CROSS_BORDER = "CROSS_BORDER"
    BREACH_NOTIFICATION = "BREACH_NOTIFICATION"
    DP_RIGHTS = "DP_RIGHTS"
    GOVERNING_LAW = "GOVERNING_LAW"
    TERMINATION = "TERMINATION"
    LIABILITY = "LIABILITY"

class ComplianceSeverity(str, Enum):
    CRITICAL = "CRITICAL"
    HIGH = "HIGH"
    MEDIUM = "MEDIUM"
    LOW = "LOW"

@dataclass
class DetectedLanguage:
    language_code: str   # ISO 639-1 + BCP 47: "hi", "ta", "mr-Deva", "ur-Arab"
    script: str          # "Devanagari", "Tamil", "Latin", "Arabic" (for Urdu)
    confidence: float
    text_span: tuple[int, int]   # character offsets in the document

@dataclass
class Clause:
    clause_type: ClauseType
    text: str                    # English text (translated if needed)
    source_text: str             # Original language text
    confidence: float
    page_number: int
    char_offset: int

@dataclass
class ComplianceFlag:
    rule_id: str                 # e.g., "DPDPA-S6-001"
    clause_type: ClauseType
    flag: str                    # e.g., "DATA_SHARING_UNLIMITED"
    severity: ComplianceSeverity
    section_reference: str       # e.g., "Section 6 — purpose must be specific"
    affected_text: str           # The clause text that triggered the flag
    remediation: str             # Specific fix instruction
    is_absence_violation: bool   # True when the clause is missing entirely
    penalty_exposure_cr: float   # Estimated from DPDPA penalty schedule

@dataclass
class AuditReport:
    contract_id: str
    tenant_id: str
    vendor_name: str
    detected_languages: list[DetectedLanguage]
    clauses_found: list[Clause]
    compliance_flags: list[ComplianceFlag]
    gap_matrix: dict             # clause_type → "PRESENT" | "ABSENT" | "PARTIAL" | "NON_COMPLIANT"
    total_penalty_exposure_cr: float
    remediation_priority_list: list[str]   # rule_ids ordered by severity
    generated_at: str

@dataclass
class SignedArtifact:
    artifact_type: str           # "DPA" | "AUDIT_REPORT"
    content_docx_s3_uri: str
    content_pdf_s3_uri: str
    signature: str               # ECDSA P-256 signature (hex)
    signer_key_id: str           # KMS key ARN used
    audit_report_hash: str       # SHA-256 of AuditReport JSON
    timestamp: str               # ISO 8601
    verification_url: str        # https://truststack.in/verify/{artifact_id}
```

---

### FileProcessor

**Purpose:** Ingest the uploaded file and extract raw text with metadata. Acts as router — determines whether OCR is needed.

```python
@activity.defn(name="file_processor")
async def file_processor_activity(
    file_s3_uri: str,
    contract_id: str,
) -> tuple[str, dict, bool]:
    """Returns (raw_text, metadata, needs_ocr)."""
    file_bytes, content_type = await _download_from_s3(file_s3_uri)
    metadata = {"content_type": content_type, "size_bytes": len(file_bytes)}

    if content_type in ("application/pdf",):
        text, confidence = _extract_pdf_text(file_bytes)
        if confidence < 0.80:
            # Scanned PDF — delegate to OCR
            return "", metadata, True
        return text, metadata, False

    elif content_type in (
        "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
    ):
        text = _extract_docx_text(file_bytes)
        return text, metadata, False

    elif content_type.startswith("image/"):
        return "", metadata, True

    elif content_type == "text/plain":
        return file_bytes.decode("utf-8"), metadata, False

    raise UnsupportedFileFormatError(f"Unsupported format: {content_type}")
```

**Chunking strategy:** Documents are split into page-level chunks (PDF) or heading-delimited sections (DOCX) for parallel OCR and translation. The chunk index is preserved for page number tracking in ClauseExtractor output.

---

### OCRAgent

**Purpose:** Extract text from scanned documents and images using Tesseract 5 with Indic language packs.

```python
TESSERACT_LANG_PACK_MAP: dict[str, str] = {
    # ISO 639-1 → Tesseract language pack
    "hi": "hin",   # Hindi (Devanagari)
    "ta": "tam",   # Tamil
    "te": "tel",   # Telugu
    "kn": "kan",   # Kannada
    "ml": "mal",   # Malayalam
    "mr": "mar",   # Marathi (Devanagari)
    "gu": "guj",   # Gujarati
    "bn": "ben",   # Bengali
    "pa": "pan",   # Punjabi (Gurmukhi)
    "ur": "urd",   # Urdu (Arabic script, RTL)
    "en": "eng",
    # Additional 12 Eighth Schedule languages use "script" packs
    "or": "ori",   # Odia
    "as": "asm",   # Assamese
    "ne": "nep",   # Nepali
    "sa": "san",   # Sanskrit
    "sd": "snd",   # Sindhi
    "ks": "kas",   # Kashmiri
    "mni": "mni",  # Manipuri (Meitei Mayek)
    "sat": "sat",  # Santali (Ol Chiki)
    "kok": "kok",  # Konkani
    "mai": "mai",  # Maithili
    "doi": "doi",  # Dogri
    "bho": "bho",  # Bodo
}

@activity.defn(name="ocr_agent")
async def ocr_agent_activity(
    file_s3_uri: str,
    hint_languages: list[str],    # ISO 639-1 codes to try first
) -> tuple[str, float]:
    """Returns (extracted_text, mean_confidence)."""
    file_bytes = await _download_raw(file_s3_uri)
    images = _pdf_to_images(file_bytes) if file_bytes[:4] == b'%PDF' else [file_bytes]

    # Build Tesseract lang string — try hinted languages first, fallback to eng+hin
    lang_packs = [TESSERACT_LANG_PACK_MAP[l] for l in hint_languages
                  if l in TESSERACT_LANG_PACK_MAP]
    lang_packs = lang_packs or ["hin", "eng"]
    lang_string = "+".join(lang_packs)

    page_texts: list[str] = []
    confidences: list[float] = []

    for image_bytes in images:
        result = tesseract.image_to_data(
            image_bytes,
            lang=lang_string,
            config="--psm 3 --oem 1",  # PSM 3: auto page seg; OEM 1: LSTM engine
        )
        page_texts.append(result.text)
        confidences.append(result.mean_confidence / 100.0)

    mean_confidence = sum(confidences) / len(confidences)

    if mean_confidence < 0.75:
        # Warn but continue — do not block the pipeline
        activity.logger.warning(
            "ocr_low_confidence",
            {"confidence": mean_confidence, "file_s3_uri": file_s3_uri}
            # NOTE: file_s3_uri logged only — never the extracted text (may contain PII)
        )

    return "\n\n".join(page_texts), mean_confidence
```

**Indic OCR post-processing:** Common character confusions in Tesseract's Devanagari model (e.g., ण/ण, ष/ष) and Tamil model (ற/ர confusion) are corrected using a rule-based substitution table derived from the `aksharamukha` transliteration library. This table is applied after OCR, before language detection.

---

### LanguageDetector

**Purpose:** Identify all languages present in the document. Indian contracts commonly code-switch between a regional language and English legal terms within the same clause.

```python
import langdetect
import fasttext

EIGHTH_SCHEDULE_LANG_CODES = {
    "as", "bn", "bho", "doi", "gu", "hi", "kn", "ks", "kok", "mai",
    "ml", "mni", "mr", "ne", "or", "pa", "sa", "sat", "sd", "ta", "te", "ur"
}

@activity.defn(name="language_detector")
async def language_detector_activity(text: str) -> list[DetectedLanguage]:
    """Detect all languages in a document. Handles code-switching."""
    segments = _segment_by_script(text)  # split at script boundaries (Unicode block changes)

    detected: list[DetectedLanguage] = []
    for segment_text, char_start, char_end in segments:
        if len(segment_text.strip()) < 20:
            continue  # too short for reliable detection

        # fastText for Indic scripts (better than langdetect for South Asian languages)
        ft_lang, ft_conf = ft_model.predict(segment_text.replace("\n", " "), k=1)
        ft_lang_code = ft_lang[0].replace("__label__", "")

        # langdetect as secondary verifier for Latin-script segments
        if _is_latin_script(segment_text):
            try:
                ld_langs = langdetect.detect_langs(segment_text)
                ld_lang = ld_langs[0].lang if ld_langs else "en"
            except:
                ld_lang = "en"
            lang_code = ld_lang
            confidence = float(ld_langs[0].prob) if ld_langs else ft_conf[0]
        else:
            lang_code = ft_lang_code
            confidence = float(ft_conf[0])

        script = _get_unicode_script(segment_text)
        detected.append(DetectedLanguage(
            language_code=lang_code,
            script=script,
            confidence=confidence,
            text_span=(char_start, char_end),
        ))

    return detected
```

---

### TranslationAgent

**Purpose:** Translate contract text to English for clause analysis. Preserves original text for DPA generation in source language.

**Temporal activity:**

```python
TRANSLATION_SYSTEM_PROMPT = """You are a legal translation engine specializing in Indian contracts and regulatory documents.

Translate the following text to English. Preserve all legal terminology precisely.
Do not add, remove, or interpret meaning. Do not add disclaimers or explanations.
If you encounter Indian legal terms with no direct English equivalent
(e.g., "Tehsildar", "Gram Sabha", "Taluk"), retain the original term in parentheses
after the English approximation.

Preserve the paragraph and section structure of the original text.
Output ONLY the translated text — no preamble, no explanation."""

LEGAL_GLOSSARY: dict[str, str] = {
    # Hindi legal terms
    "अनुबंध": "contract",
    "समझौता": "agreement",
    "डेटा प्रिंसिपल": "Data Principal",
    "डेटा फिड्युशरी": "Data Fiduciary",
    "सहमति": "consent",
    "व्यक्तिगत डेटा": "personal data",
    "उल्लंघन": "breach",
    "प्रतिधारण": "retention",
    "मिटाना": "erasure",
    # Tamil legal terms
    "தரவு முதன்மை": "Data Principal",
    "ஒப்பந்தம்": "agreement",
    "தனியுரிமை": "privacy",
    "சம்மதம்": "consent",
    # Marathi legal terms
    "करार": "agreement",
    "संमती": "consent",
    "वैयक्तिक माहिती": "personal data",
    # Gujarati legal terms
    "કરાર": "agreement",
    "સંમતિ": "consent",
    "ગોપનીયતા": "privacy",
}

@activity.defn(name="translation_agent")
async def translation_agent_activity(
    text: str,
    source_language: str,
) -> tuple[str, str]:
    """Returns (english_translation, source_text_preserved)."""
    if source_language == "en":
        return text, text

    # Inject legal glossary as few-shot context
    glossary_context = "\n".join(
        f"  {src} → {tgt}" for src, tgt in LEGAL_GLOSSARY.items()
        if any(c in text for c in src)  # only inject relevant entries
    )
    user_prompt = f"Legal glossary for reference:\n{glossary_context}\n\n---\n\n{text}"

    translated = await sagemaker_invoke(
        endpoint_name="truststack-mistral7b-legal-translate-v1",
        system=TRANSLATION_SYSTEM_PROMPT,
        user=user_prompt,
        max_tokens=4096,
        temperature=0.1,  # low but not zero — allows natural translation fluency
    )
    return translated, text
```

---

### ClauseExtractor

**Purpose:** Segment the translated contract into typed clauses using a BERT-based segmentation model fine-tuned on Indian commercial contracts.

```python
@activity.defn(name="clause_extractor")
async def clause_extractor_activity(
    english_text: str,
    source_text: str,
    page_boundaries: list[int],   # character offsets of page breaks
) -> list[Clause]:
    """Segment contract text into DPDPA-relevant clause types."""
    # BERT inference — batch inference on SageMaker
    segments = await _bert_segment(english_text)
    # Each segment: {"type": ClauseType, "text": str, "start": int, "end": int, "confidence": float}

    clauses: list[Clause] = []
    for seg in segments:
        if seg["confidence"] < 0.70:
            continue  # below threshold — skip rather than misclassify

        page_num = _char_offset_to_page(seg["start"], page_boundaries)
        source_slice = _align_source_text(seg["start"], seg["end"], english_text, source_text)

        clauses.append(Clause(
            clause_type=ClauseType(seg["type"]),
            text=seg["text"],
            source_text=source_slice,
            confidence=seg["confidence"],
            page_number=page_num,
            char_offset=seg["start"],
        ))
    return clauses
```

**BERT model specifics:**

- Base: `ai4bharat/indic-bert` (pre-trained on IndicCorp — 12 Indic languages)
- Fine-tuning: Supervised on 2,400 annotated Indian commercial contracts (NDA, MSA, DPA, vendor agreements) with clause-level labels. Dataset sourced from open Indian legal repositories and manually annotated by legal SMEs.
- Input: Contract segments of max 512 tokens (BERT limit). Overlapping windows of 50 tokens for long clauses.
- Output: Per-segment classification logits for the 11 `ClauseType` classes.
- Inference: SageMaker `ml.m5.xlarge` (CPU inference — BERT is much smaller than Mistral 7B).

---

### ComplianceChecker

**Purpose:** Check each extracted clause against the DPDPA compliance taxonomy. Uses both rule-based pattern matching and LLM-based semantic analysis for ambiguous cases.

**Compliance rule taxonomy (all rules):**

| Rule ID | Clause Type | Trigger | Flag | Severity | DPDPA Reference |
|---|---|---|---|---|---|
| DPDPA-S6-001 | PURPOSE | "any purpose", "all purposes", "any lawful purpose", "as determined by" | DATA_SHARING_UNLIMITED | CRITICAL | Section 6 — purpose must be specific and informed |
| DPDPA-S6-002 | PURPOSE | Clause is absent entirely | MISSING_PURPOSE_CLAUSE | CRITICAL | Section 6 — notice must specify purpose |
| DPDPA-S6-003 | PURPOSE | "legitimate interest", "business interest", "operational need" (without enumeration) | LEGITIMATE_INTEREST_INVALID | CRITICAL | Section 6 — no legitimate interest basis under DPDPA |
| DPDPA-R7-001 | BREACH_NOTIFICATION | Clause absent | MISSING_BREACH_NOTIFICATION_SLA | CRITICAL | Rule 7 — vendor must notify DF within 24hr; DF notifies DPBI within 72hr |
| DPDPA-R7-002 | BREACH_NOTIFICATION | Notification SLA > 72hr or unspecified | BREACH_SLA_NON_COMPLIANT | CRITICAL | Rule 7 — 72hr maximum to DPBI |
| DPDPA-S16-001 | CROSS_BORDER | "transfer outside India", "international", "cross-border" without DPBI authorization | CROSS_BORDER_UNAUTHORIZED | HIGH | Section 16 — requires DPBI pre-authorization |
| DPDPA-S9-001 | DATA_CATEGORIES | "children", "minors", "under 18", "under-18" | MINOR_DATA_NO_PARENTAL_CONSENT | CRITICAL | Section 9 — verifiable parental consent required |
| DPDPA-S9-002 | PURPOSE | "targeted advertising", "profiling" combined with presence of MINOR data category | MINOR_PROFILING_PROHIBITED | CRITICAL | Section 9 — targeting and profiling of minors absolutely prohibited |
| DPDPA-S11-001 | DP_RIGHTS | Clause absent | MISSING_DATA_PRINCIPAL_RIGHTS | HIGH | Section 11 — right to access, correction, erasure must be stated |
| DPDPA-S11-002 | DP_RIGHTS | "rights may be limited", "at our discretion", no specific mechanism for rights requests | DP_RIGHTS_RESTRICTED | HIGH | Section 11 — rights cannot be contractually limited |
| DPDPA-S12-001 | RETENTION | Clause absent | MISSING_RETENTION_CLAUSE | HIGH | Section 12 — data must be erased when purpose is fulfilled |
| DPDPA-S12-002 | RETENTION | Retention period is indefinite ("until further notice", "for the duration of the relationship") | INDEFINITE_RETENTION | CRITICAL | Section 12 — indefinite retention is non-compliant |
| DPDPA-S12-003 | RETENTION | Retention > purpose fulfillment without regulatory mandate | EXCESSIVE_RETENTION | HIGH | Section 12 — must erase when purpose fulfilled |
| DPDPA-S17-001 | SHARING | "share with affiliates", "share with group companies", "share with partners" without consent | UNAUTHORIZED_SHARING | HIGH | Section 17 — sharing requires specific consent per recipient |
| DPDPA-S17-002 | SHARING | "sell", "monetize", "license data to third parties" | DATA_MONETIZATION_PROHIBITED | CRITICAL | Section 17 — selling personal data to third parties is prohibited |
| DPDPA-R3-001 | PARTIES | Notice not in Data Principal's interface language | NOTICE_LANGUAGE_NON_COMPLIANT | HIGH | Rule 3 — notice must be in the language of the interface |
| DPDPA-CM-001 | (any) | Vendor is also processing data for same Data Principals as client | DUAL_ROLE_CONFLICT | CRITICAL | Section 2(g) — Consent Manager cannot also be Data Processor |

**LangGraph StateGraph for compliance checking:**

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, Annotated
import operator

class ComplianceState(TypedDict):
    clauses: list[Clause]
    flags: Annotated[list[ComplianceFlag], operator.add]
    absence_check_done: bool

COMPLIANCE_RULES: list[dict] = [
    {
        "id": "DPDPA-S6-001",
        "clause_type": ClauseType.PURPOSE,
        "patterns": [
            r"any purpose", r"all purposes", r"any lawful purpose",
            r"as determined by", r"at our discretion",
        ],
        "absence_flag": False,
        "flag": "DATA_SHARING_UNLIMITED",
        "severity": ComplianceSeverity.CRITICAL,
        "section": "Section 6 — purpose must be specific and informed",
        "remediation": (
            "Replace vague purpose language with an enumerated list of specific purposes "
            "that are directly linked to the data categories collected. "
            "Each purpose must be described in plain language."
        ),
        "penalty_exposure_cr": 50.0,
    },
    {
        "id": "DPDPA-R7-001",
        "clause_type": ClauseType.BREACH_NOTIFICATION,
        "patterns": [],
        "absence_flag": True,   # triggers when clause TYPE is entirely absent
        "flag": "MISSING_BREACH_NOTIFICATION_SLA",
        "severity": ComplianceSeverity.CRITICAL,
        "section": "Rule 7 — vendor must notify within 72hrs of becoming aware",
        "remediation": (
            "Add: 'Vendor shall notify Data Fiduciary within 24 hours of becoming aware "
            "of any personal data breach. Data Fiduciary shall notify the Data Protection "
            "Board of India within 72 hours. Vendor shall cooperate with DPBI investigation.'"
        ),
        "penalty_exposure_cr": 50.0,
    },
    {
        "id": "DPDPA-S16-001",
        "clause_type": ClauseType.CROSS_BORDER,
        "patterns": [
            r"transfer.*outside India", r"international.*transfer",
            r"cross.border.*transfer", r"store.*outside India",
            r"process.*outside India",
        ],
        "absence_flag": False,
        "flag": "CROSS_BORDER_UNAUTHORIZED",
        "severity": ComplianceSeverity.HIGH,
        "section": "Section 16 — cross-border transfer requires DPBI pre-authorization",
        "remediation": (
            "Either: (a) add a clause restricting all processing to India and confirm "
            "vendor infrastructure is in Indian data centres; or (b) add a DPBI "
            "pre-authorization clause specifying the authorized jurisdictions and "
            "the authorization reference number."
        ),
        "penalty_exposure_cr": 20.0,
    },
    {
        "id": "DPDPA-S6-003",
        "clause_type": ClauseType.PURPOSE,
        "patterns": [
            r"legitimate interest", r"business interest",
            r"operational need(?! for the following specific purposes)",
        ],
        "absence_flag": False,
        "flag": "LEGITIMATE_INTEREST_INVALID",
        "severity": ComplianceSeverity.CRITICAL,
        "section": "Section 6 — DPDPA has no legitimate interest basis unlike GDPR",
        "remediation": (
            "Remove all references to 'legitimate interest' as a processing basis. "
            "Replace with explicit consent or a specific statutory obligation. "
            "Enumerate the specific purposes with corresponding data categories."
        ),
        "penalty_exposure_cr": 50.0,
    },
    {
        "id": "DPDPA-S12-002",
        "clause_type": ClauseType.RETENTION,
        "patterns": [
            r"until further notice", r"for the duration of the relationship",
            r"indefinitely", r"as long as necessary.*without.*specific.*period",
        ],
        "absence_flag": False,
        "flag": "INDEFINITE_RETENTION",
        "severity": ComplianceSeverity.CRITICAL,
        "section": "Section 12 — data must be erased when purpose is fulfilled",
        "remediation": (
            "Replace with a specific retention period tied to each data category and purpose. "
            "Reference the DPDPA Third Schedule class-specific timelines where applicable. "
            "Add a 48-hour pre-deletion notification clause."
        ),
        "penalty_exposure_cr": 30.0,
    },
    {
        "id": "DPDPA-S17-002",
        "clause_type": ClauseType.SHARING,
        "patterns": [
            r"sell.*data", r"monetize.*data", r"license.*data.*third part",
            r"data.*for advertising.*third", r"provide.*data.*for.*consideration",
        ],
        "absence_flag": False,
        "flag": "DATA_MONETIZATION_PROHIBITED",
        "severity": ComplianceSeverity.CRITICAL,
        "section": "Section 17 — selling personal data to third parties is prohibited",
        "remediation": (
            "Remove any clause permitting the sale, licensing, or monetization of "
            "personal data to third parties. Replace with permitted sharing clauses "
            "that require explicit per-recipient consent."
        ),
        "penalty_exposure_cr": 50.0,
    },
    # Additional rules: DPDPA-S6-002, DPDPA-S9-001, DPDPA-S9-002, DPDPA-S11-001,
    # DPDPA-S11-002, DPDPA-S12-001, DPDPA-S12-003, DPDPA-S17-001,
    # DPDPA-R3-001, DPDPA-CM-001 — follow same schema
]

def pattern_check(state: ComplianceState) -> ComplianceState:
    """Rule-based pattern matching against all clauses."""
    import re
    flags: list[ComplianceFlag] = []
    found_clause_types = {c.clause_type for c in state["clauses"]}

    for rule in COMPLIANCE_RULES:
        if rule["absence_flag"]:
            # Check if the required clause type is entirely absent
            if ClauseType(rule["clause_type"]) not in found_clause_types:
                flags.append(ComplianceFlag(
                    rule_id=rule["id"],
                    clause_type=rule["clause_type"],
                    flag=rule["flag"],
                    severity=rule["severity"],
                    section_reference=rule["section"],
                    affected_text="[CLAUSE ABSENT]",
                    remediation=rule["remediation"],
                    is_absence_violation=True,
                    penalty_exposure_cr=rule["penalty_exposure_cr"],
                ))
        else:
            # Pattern match against relevant clause type
            for clause in state["clauses"]:
                if clause.clause_type != rule["clause_type"]:
                    continue
                for pattern in rule["patterns"]:
                    if re.search(pattern, clause.text, re.IGNORECASE):
                        flags.append(ComplianceFlag(
                            rule_id=rule["id"],
                            clause_type=rule["clause_type"],
                            flag=rule["flag"],
                            severity=rule["severity"],
                            section_reference=rule["section"],
                            affected_text=clause.text[:500],
                            remediation=rule["remediation"],
                            is_absence_violation=False,
                            penalty_exposure_cr=rule["penalty_exposure_cr"],
                        ))
                        break  # one flag per rule per clause

    return {**state, "flags": flags, "absence_check_done": True}

COMPLIANCE_LLM_SYSTEM_PROMPT = """You are a DPDPA 2023 compliance expert analyzing an Indian vendor contract.

You have been given a contract clause that has passed initial regex checks but may still
have semantic compliance issues. Analyze whether this clause violates any of the following
DPDPA provisions:

1. Section 6: Purpose must be specific, informed, and not overly broad. No "legitimate interest" basis.
2. Section 9: No targeted advertising or profiling of minors. Parental consent required for under-18.
3. Section 11: Data Principal rights (access, correction, erasure, grievance) must be explicitly preserved.
4. Section 12: Retention must be purpose-linked. Indefinite retention is non-compliant.
5. Section 16: Cross-border transfers require DPBI pre-authorization.
6. Section 17: Data cannot be sold or monetized. Sharing requires per-recipient consent.
7. Rule 7: Breach notification must be within 72hrs to DPBI; vendor must notify DF within 24hrs.

Respond ONLY with a JSON array of compliance issues found:
[
  {
    "rule_id": "DPDPA-<SECTION>-<ID>",
    "flag": "<FLAG_NAME>",
    "severity": "CRITICAL" | "HIGH" | "MEDIUM",
    "explanation": "<one sentence>",
    "affected_phrase": "<exact phrase from the clause that caused the issue>",
    "remediation": "<specific fix instruction>"
  }
]

If no compliance issues are found, respond with: []
Do not invent issues. Only flag clear violations or strong risks."""

async def llm_semantic_check(state: ComplianceState) -> ComplianceState:
    """LLM-based semantic analysis for clauses that passed regex but may still be non-compliant."""
    # Only run for clauses of high-risk types that weren't already flagged
    already_flagged_types = {f.clause_type for f in state["flags"]}
    high_risk_unflagged = [
        c for c in state["clauses"]
        if c.clause_type in (
            ClauseType.PURPOSE, ClauseType.SHARING,
            ClauseType.RETENTION, ClauseType.CROSS_BORDER,
            ClauseType.BREACH_NOTIFICATION,
        )
        and c.clause_type not in already_flagged_types
    ]

    if not high_risk_unflagged:
        return state

    llm_flags: list[ComplianceFlag] = []
    for clause in high_risk_unflagged:
        user_prompt = (
            f"Clause type: {clause.clause_type.value}\n\n"
            f"Clause text:\n{clause.text[:2000]}"
        )
        response = await sagemaker_invoke(
            endpoint_name="truststack-mistral7b-compliance-v1",
            system=COMPLIANCE_LLM_SYSTEM_PROMPT,
            user=user_prompt,
            max_tokens=512,
            temperature=0.0,
        )
        issues = _safe_parse_json(response)
        for issue in issues:
            llm_flags.append(ComplianceFlag(
                rule_id=issue.get("rule_id", "LLM-INFERRED"),
                clause_type=clause.clause_type,
                flag=issue.get("flag", "SEMANTIC_VIOLATION"),
                severity=ComplianceSeverity(issue.get("severity", "MEDIUM")),
                section_reference=issue.get("explanation", ""),
                affected_text=issue.get("affected_phrase", clause.text[:200]),
                remediation=issue.get("remediation", ""),
                is_absence_violation=False,
                penalty_exposure_cr=0.0,   # LLM-inferred flags get 0 — reviewed by human
            ))

    return {**state, "flags": llm_flags}

def build_compliance_graph() -> StateGraph:
    graph = StateGraph(ComplianceState)
    graph.add_node("pattern_check", pattern_check)
    graph.add_node("llm_semantic_check", llm_semantic_check)

    graph.set_entry_point("pattern_check")
    graph.add_edge("pattern_check", "llm_semantic_check")
    graph.add_edge("llm_semantic_check", END)
    return graph.compile()
```

---

### GapAnalyzer

**Purpose:** Produce a structured audit report with a compliance gap matrix and prioritized remediation checklist.

```python
# Required DPDPA provisions: every compliant vendor contract MUST address these
REQUIRED_DPDPA_PROVISIONS: list[ClauseType] = [
    ClauseType.PARTIES,
    ClauseType.PURPOSE,
    ClauseType.DATA_CATEGORIES,
    ClauseType.RETENTION,
    ClauseType.SHARING,
    ClauseType.BREACH_NOTIFICATION,
    ClauseType.DP_RIGHTS,
    ClauseType.CROSS_BORDER,     # required IF any cross-border flag was raised
    ClauseType.TERMINATION,
    ClauseType.LIABILITY,
]

GapStatus = str  # "PRESENT_COMPLIANT" | "PRESENT_NON_COMPLIANT" | "PRESENT_PARTIAL" | "ABSENT"

@activity.defn(name="gap_analyzer")
async def gap_analyzer_activity(
    clauses: list[Clause],
    flags: list[ComplianceFlag],
    contract_id: str,
    tenant_id: str,
    vendor_name: str,
    detected_languages: list[DetectedLanguage],
) -> AuditReport:
    clause_types_found = {c.clause_type for c in clauses}
    flagged_types = {f.clause_type for f in flags if not f.is_absence_violation}
    absent_types = {f.clause_type for f in flags if f.is_absence_violation}

    gap_matrix: dict[str, GapStatus] = {}
    for provision in REQUIRED_DPDPA_PROVISIONS:
        if provision in absent_types:
            gap_matrix[provision.value] = "ABSENT"
        elif provision in flagged_types:
            critical_flags = [
                f for f in flags
                if f.clause_type == provision
                and f.severity == ComplianceSeverity.CRITICAL
            ]
            gap_matrix[provision.value] = (
                "PRESENT_NON_COMPLIANT" if critical_flags
                else "PRESENT_PARTIAL"
            )
        elif provision in clause_types_found:
            gap_matrix[provision.value] = "PRESENT_COMPLIANT"
        else:
            gap_matrix[provision.value] = "ABSENT"

    total_penalty = sum(f.penalty_exposure_cr for f in flags)

    # Sort remediation by severity then by penalty exposure
    severity_order = {
        ComplianceSeverity.CRITICAL: 0,
        ComplianceSeverity.HIGH: 1,
        ComplianceSeverity.MEDIUM: 2,
        ComplianceSeverity.LOW: 3,
    }
    sorted_flags = sorted(
        flags,
        key=lambda f: (severity_order[f.severity], -f.penalty_exposure_cr)
    )
    remediation_priority = [f.rule_id for f in sorted_flags]

    return AuditReport(
        contract_id=contract_id,
        tenant_id=tenant_id,
        vendor_name=vendor_name,
        detected_languages=detected_languages,
        clauses_found=clauses,
        compliance_flags=flags,
        gap_matrix=gap_matrix,
        total_penalty_exposure_cr=total_penalty,
        remediation_priority_list=remediation_priority,
        generated_at=datetime.datetime.utcnow().isoformat(),
    )
```

**Gap matrix example output (Zoho CRM vendor contract):**

| Provision | Status | Flags |
|---|---|---|
| PARTIES | PRESENT_COMPLIANT | — |
| PURPOSE | PRESENT_NON_COMPLIANT | DPDPA-S6-001: DATA_SHARING_UNLIMITED |
| DATA_CATEGORIES | PRESENT_PARTIAL | categories listed but too broad |
| RETENTION | ABSENT | DPDPA-S12-001: MISSING_RETENTION_CLAUSE |
| SHARING | PRESENT_NON_COMPLIANT | DPDPA-S17-001: UNAUTHORIZED_SHARING |
| BREACH_NOTIFICATION | ABSENT | DPDPA-R7-001: MISSING_BREACH_NOTIFICATION_SLA |
| DP_RIGHTS | PRESENT_PARTIAL | DPDPA-S11-002: DP_RIGHTS_RESTRICTED |
| CROSS_BORDER | PRESENT_COMPLIANT | — |
| TERMINATION | PRESENT_COMPLIANT | — |
| LIABILITY | PRESENT_PARTIAL | — |

**Total penalty exposure: ₹180Cr**

---

### DPAGenerator

**Purpose:** Generate a DPDPA-compliant Data Processing Agreement, filling in vendor-specific details extracted from the contract. Output is in the contract's source language.

**DPA template structure (12 sections):**

| Section | Title | What Goes In |
|---|---|---|
| 1 | Parties and Definitions | Data Fiduciary name, Vendor (Data Processor) name, effective date, DPDPA definitions (Data Principal, personal data, consent, purpose) as per Section 2 |
| 2 | Scope and Purpose | Enumerated list of specific processing purposes (extracted from contract + gap-filled by LLM). Each purpose tied to specific data categories. No "any purpose" language. |
| 3 | Data Categories | Exhaustive list of personal data categories to be processed. Separate section for sensitive personal data (Section 3). |
| 4 | Processing Instructions | Vendor processes data ONLY on written instructions from DF. No independent processing. Obligations on vendor's sub-processors (Section 6). |
| 5 | Data Principal Rights | Procedures for handling access, correction, erasure, nomination, and grievance requests (Section 11). SLA: respond within 30 days. |
| 6 | Retention and Erasure | Class-specific retention periods per DPDPA Third Schedule. 48hr pre-deletion notification. Hard deletion (not soft delete). Deletion certificates (ECDSA signed). |
| 7 | Security Measures | AES-256 at rest, TLS 1.3 in transit, access logging, penetration testing schedule, incident response procedure. |
| 8 | Breach Notification | Vendor notifies DF within 24hrs of becoming aware of breach. DF notifies DPBI within 72hrs (Rule 7). DF notifies affected Data Principals. Vendor cooperates with DPBI investigation. |
| 9 | Cross-Border Transfers | Either: (a) all processing restricted to India; or (b) DPBI authorization reference, approved jurisdictions, and standard contractual clauses (Section 16). |
| 10 | Sub-Processors | List of approved sub-processors. Prior written consent required for additions. Sub-processors bound by terms equivalent to this DPA. |
| 11 | Audit Rights | DF or DF's auditor may audit vendor's processing on 30-day notice. Vendor provides records on request. Annual compliance attestation. |
| 12 | Governing Law and Dispute Resolution | Indian law. Jurisdiction: courts of the DF's registered place. Dispute escalation to DPBI before litigation. |

**DPA generation prompt:**

```python
DPA_GENERATION_SYSTEM_PROMPT = """You are a DPDPA 2023 legal drafting engine.

Generate Section {section_number}: {section_title} of a Data Processing Agreement (DPA)
between the following parties:

Data Fiduciary: {df_name}
Data Processor (Vendor): {vendor_name}
Effective Date: {effective_date}

Use the following information extracted from the existing contract:
{extracted_context}

The DPA must comply with India's Digital Personal Data Protection Act 2023 and DPDP Rules 2025.
All provisions must be specific and enforceable. Do not use vague language such as
"reasonable efforts", "as necessary", or "appropriate measures" without defining them.

If writing in {target_language}, use legally precise {target_language} with English
legal terms in parentheses on first use (e.g., 'सहमति (Consent)').

Output ONLY the section text — no headings above the section, no preamble, no explanation."""
```

**Translation for DPA:** When the source contract was in a non-English language, the generated DPA is produced in English first, then translated using the same `translation_agent` pipeline (direction reversed: English → source language). The English version is always preserved as the authoritative text for legal purposes.

---

### Signer

**Purpose:** Cryptographically sign the generated DPA and audit report using the tenant's ECDSA P-256 KMS key. Same signing mechanism used by the SahmatOS Consent Vault for interoperability.

```python
@activity.defn(name="signer")
async def signer_activity(
    dpa_docx_s3_uri: str,
    dpa_pdf_s3_uri: str,
    audit_report: AuditReport,
    tenant_id: str,
) -> SignedArtifact:
    import hashlib, json, boto3
    from datetime import datetime, timezone

    kms_client = boto3.client("kms", region_name="ap-south-1")

    # Retrieve tenant's signing key ID from tenant config
    key_id = await _get_tenant_signing_key(tenant_id)

    # Compute content hash
    audit_report_json = json.dumps(
        _serialize_audit_report(audit_report), sort_keys=True
    ).encode("utf-8")
    audit_hash = hashlib.sha256(audit_report_json).hexdigest()

    dpa_pdf_bytes = await _download_raw(dpa_pdf_s3_uri)
    dpa_pdf_hash = hashlib.sha256(dpa_pdf_bytes).hexdigest()

    timestamp = datetime.now(timezone.utc).isoformat()

    # Payload to sign: SHA-256(dpa_pdf) + SHA-256(audit_report) + timestamp
    signing_payload = f"{dpa_pdf_hash}:{audit_hash}:{timestamp}".encode("utf-8")
    payload_hash = hashlib.sha256(signing_payload).hexdigest()

    # Sign using AWS KMS ECDSA P-256
    # DPDPA Rule 4: Consent Manager must use cryptographic signing for all artifacts
    sign_response = kms_client.sign(
        KeyId=key_id,
        Message=bytes.fromhex(payload_hash),
        MessageType="DIGEST",
        SigningAlgorithm="ECDSA_SHA_256",
    )
    signature_hex = sign_response["Signature"].hex()

    artifact_id = _generate_artifact_id(tenant_id, audit_report.contract_id)

    return SignedArtifact(
        artifact_type="DPA",
        content_docx_s3_uri=dpa_docx_s3_uri,
        content_pdf_s3_uri=dpa_pdf_s3_uri,
        signature=signature_hex,
        signer_key_id=key_id,
        audit_report_hash=audit_hash,
        timestamp=timestamp,
        verification_url=f"https://truststack.in/verify/{artifact_id}",
    )
```

**Verification:** Any party with TrustStack's public key can verify a DPA signature:

```bash
# Verification endpoint (public)
GET https://truststack.in/verify/{artifact_id}

# Response includes: signer_key_id, timestamp, audit_report_hash, signature
# Verifier reconstructs payload and validates ECDSA signature against TrustStack public key
```

---

### Multi-Language Handling Strategy

**Four-layer approach for full 22-language support:**

1. **Script detection first:** Unicode block analysis determines script before language identification. This avoids langdetect misidentifying Devanagari (used by Hindi, Marathi, Nepali, Sanskrit, Dogri) as a single language.

2. **fastText primary, langdetect secondary:** fastText's `lid.176.bin` model trained on 176 languages (including all 22 Eighth Schedule languages) is used as primary identifier. For Latin-script segments (common in English legal terms within Indic-language contracts), langdetect provides higher accuracy.

3. **Code-switching segmentation:** Indian contracts frequently embed English legal terms within Indic-language sentences (e.g., "डेटा fiduciary को 72 hours के भीतर DPBI को notify करना होगा"). The `_segment_by_script` function splits at script boundaries, allowing each segment to be processed with the correct language model.

4. **On-premise Mistral 7B for translation:** The fine-tuned Mistral 7B handles all 22 languages including rarer ones (Dogri, Bodo, Santali) where commercial translation APIs have limited coverage. The model was fine-tuned on a legal domain corpus to preserve terminology. For languages with very limited training data (Dogri, Maithili, Santali), the model first transliterates to the Latin script before translating to English — reducing hallucination risk.

**Language-specific DPA generation:**

| Language Group | DPA Strategy |
|---|---|
| Hindi, Marathi, Gujarati, Punjabi, Urdu | Full DPA generation in target language. English terms in parentheses on first use. |
| Tamil, Telugu, Kannada, Malayalam | Full DPA generation. English legal terms transliterated to script on first use. |
| Bengali, Odia, Assamese | Full DPA generation. |
| Kashmiri, Sindhi (Arabic script) | DPA generated in English + translation appended. RTL layout in DOCX output. |
| Santali (Ol Chiki), Manipuri (Meitei Mayek) | DPA generated in English. Translation advisory flagged for human review. |
| All others (Dogri, Bodo, Maithili, Konkani, Nepali, Sanskrit) | DPA generated in English. Hindi translation provided as proxy if available. |

**RTL document handling:** Kashmiri and Sindhi contracts use Arabic script (RTL). The DOCX template uses `python-docx`'s `OxmlElement("w:bidi")` directive to set RTL paragraph direction for these sections. The PDF renderer uses `reportlab`'s bidirectional algorithm (Unicode Bidi Algorithm, UAX #9).

---

## Model Specifications

### Mistral 7B Deployments

Two separate SageMaker endpoints are deployed in `ap-south-1`:

**Endpoint 1: `truststack-mistral7b-pii-v2`** (Discovery — PII detection)

| Parameter | Value |
|---|---|
| Base model | Mistral-7B-Instruct-v0.3 |
| Fine-tuning method | QLoRA (4-bit quantized LoRA), rank=64, alpha=128 |
| Fine-tuning dataset | 15,000 synthetic records containing India-specific PII patterns (Aadhaar, PAN, UPI, IFSC, Indian mobile, Indian postal codes) with ground-truth entity labels. Synthetic generation using Faker-IN library + manual review. |
| Quantization | GPTQ 4-bit for inference (HuggingFace AutoGPTQ) |
| Instance type | `ml.g5.2xlarge` (1× NVIDIA A10G, 24GB VRAM) |
| Region | `ap-south-1` only |
| Max tokens (input) | 4,096 |
| Max tokens (output) | 1,024 |
| Temperature | 0.0 (deterministic) |
| Throughput target | 8 requests/second (batch inference) |
| P50 latency | < 800ms per request |
| P99 latency | < 2,500ms per request |
| Auto-scaling | Min 1 / Max 4 instances. Scale-out on `InvocationsPerInstance > 6/sec`. |

**Endpoint 2: `truststack-mistral7b-legal-translate-v1`** (Contract Audit — translation + compliance)

| Parameter | Value |
|---|---|
| Base model | Mistral-7B-Instruct-v0.3 |
| Fine-tuning method | QLoRA rank=64, separate LoRA adapter per language family (Indic-Devanagari, Indic-Dravidian, Indic-Eastern) |
| Fine-tuning dataset | (a) Translation: 50,000 sentence pairs from Indian legal documents (NDA, vendor agreements, privacy policies) in 12 Indic languages aligned to English. Sources: NLSIU open corpus, IndicCorp, manual annotation. (b) Compliance: 2,000 contract clauses annotated with DPDPA compliance flags by legal SMEs. |
| Quantization | GPTQ 4-bit |
| Instance type | `ml.g5.2xlarge` (1× NVIDIA A10G, 24GB VRAM) |
| Region | `ap-south-1` only |
| Max tokens (input) | 4,096 |
| Max tokens (output) | 4,096 (DPA generation requires longer output) |
| Temperature | 0.1 (translation); 0.0 (compliance checking) |
| Throughput target | 4 requests/second |
| P50 latency | < 1,500ms per request |
| P99 latency | < 5,000ms per request |
| Auto-scaling | Min 1 / Max 2 instances. Contract audit is batch-workload — burst scaling not needed. |

**BERT clause segmentation model**

| Parameter | Value |
|---|---|
| Base model | `ai4bharat/indic-bert` (multilingual BERT pretrained on IndicCorp) |
| Fine-tuning method | Full fine-tuning, 3 epochs, learning rate 2e-5, batch size 16 |
| Fine-tuning dataset | 2,400 Indian commercial contracts annotated at clause level with 11 `ClauseType` labels |
| Instance type | `ml.m5.xlarge` (CPU inference — BERT is 110M params, runs fast on CPU) |
| P50 latency | < 100ms per 512-token window |
| P99 latency | < 250ms |

**Tesseract 5 deployment**

Deployed as a sidecar container in the Temporal worker ECS task. Not a SageMaker endpoint — runs on the worker host.

| Parameter | Value |
|---|---|
| Version | Tesseract 5.3.x (LSTM engine) |
| Language packs | `hin`, `tam`, `tel`, `kan`, `mal`, `mar`, `guj`, `ben`, `pan`, `urd`, `ori`, `asm`, `eng` + script packs for remaining Eighth Schedule languages |
| GPU | None (Tesseract is CPU-only) |
| P50 latency | < 3s per A4 page |
| P99 latency | < 8s per A4 page (degraded scan quality) |

---

## Evaluation and Quality

### PII Detection (ScannerAgent)

**Targets per entity type:**

| Entity Type | Precision Target | Recall Target | Current Baseline | Notes |
|---|---|---|---|---|
| AADHAAR_NUMBER | ≥ 0.99 | ≥ 0.98 | 0.982 / 0.961 | Regex validator provides high precision; LLM handles obfuscated formats |
| PAN_NUMBER | ≥ 0.99 | ≥ 0.97 | 0.994 / 0.972 | Fixed format makes this near-deterministic |
| PHONE_NUMBER | ≥ 0.95 | ≥ 0.92 | 0.941 / 0.908 | False positives from OTP codes; regex handles most cases |
| UPI_ID | ≥ 0.97 | ≥ 0.95 | 0.963 / 0.944 | PSP suffix list reduces false positives |
| IFSC_CODE | ≥ 0.99 | ≥ 0.96 | 0.991 / 0.955 | Fixed format |
| BANK_ACCOUNT | ≥ 0.85 | ≥ 0.80 | 0.821 / 0.774 | High false positive rate from numeric sequences; field heuristic required |
| FULL_NAME | ≥ 0.88 | ≥ 0.85 | 0.871 / 0.841 | Hardest entity type — context-dependent; Indic names vary widely |
| EMAIL | ≥ 0.99 | ≥ 0.99 | 0.994 / 0.991 | Near-deterministic via regex |
| HEALTH_RECORD_ID | ≥ 0.90 | ≥ 0.85 | 0.883 / 0.831 | ABHA ID (14-digit) gaining prevalence — update patterns quarterly |
| BIOMETRIC_DATA | ≥ 0.88 | ≥ 0.82 | 0.867 / 0.804 | Field name heuristic critical; raw biometric bytes rarely in DB text fields |
| MINOR_DATA | ≥ 0.92 | ≥ 0.88 | — | No baseline yet — Section 9 enforcement begins Nov 2026 |

**Evaluation methodology:** Monthly eval run against a held-out test set of 5,000 synthetic records per entity type (never used in fine-tuning). Test set is refreshed quarterly to include new PII obfuscation patterns discovered in production scans. All eval runs are logged to MLflow on-premise instance.

**False positive handling:** Any entity finding with confidence < 0.85 that does not pass regex validation is suppressed and logged as a `LOW_CONFIDENCE_SUPPRESSED` event. The DF dashboard shows suppression counts to help tune thresholds per connector type.

### Contract Audit (ClauseExtractor + ComplianceChecker)

**Clause detection accuracy:**

| Clause Type | Accuracy Target | Current Baseline | Notes |
|---|---|---|---|
| PURPOSE | ≥ 0.92 | 0.894 | Common but often implicit; BERT model sometimes classifies as LIABILITY |
| BREACH_NOTIFICATION | ≥ 0.95 | 0.961 | Distinctive vocabulary; high accuracy |
| RETENTION | ≥ 0.90 | 0.871 | Confused with TERMINATION when both mention "end of contract" |
| CROSS_BORDER | ≥ 0.93 | 0.912 | Often embedded in DATA_CATEGORIES clause — split detection needed |
| DP_RIGHTS | ≥ 0.88 | 0.841 | Highly variable phrasing in Indian contracts |
| SHARING | ≥ 0.90 | 0.878 | "Sub-contractors" vs "third parties" distinction needs improvement |
| DATA_CATEGORIES | ≥ 0.90 | 0.901 | Good performance |

**Compliance flag precision:**

| Rule ID | Precision Target | Current Baseline |
|---|---|---|
| DPDPA-S6-001 (DATA_SHARING_UNLIMITED) | ≥ 0.90 | 0.882 |
| DPDPA-R7-001 (MISSING_BREACH_NOTIFICATION) | ≥ 0.98 | 0.971 (absence check) |
| DPDPA-S16-001 (CROSS_BORDER_UNAUTHORIZED) | ≥ 0.88 | 0.851 |
| DPDPA-S12-002 (INDEFINITE_RETENTION) | ≥ 0.92 | 0.903 |
| DPDPA-S17-002 (DATA_MONETIZATION_PROHIBITED) | ≥ 0.95 | 0.944 |

**Evaluation methodology:** Legal SME ground-truth annotation on 200 Indian vendor contracts (NDA, MSA, DPA, vendor agreements). Contracts span Hindi, Tamil, Marathi, and English. Evaluated monthly. Disagreements between SME annotators (κ > 0.75 required) are resolved by a third annotator.

### DPA Quality — Human Review Workflow

For each new tenant, the first 5 generated DPAs are flagged for human legal review before delivery. The review workflow:

1. **Automated pre-check:** Signer activity generates the DPA. A separate QA activity runs `gap_analyzer` on the DPA itself to verify all 12 required sections are present and internally consistent.
2. **Review queue:** DPA is pushed to the `dpa_review_queue` table with status `PENDING_REVIEW`. Legal reviewer is notified via Temporal signal.
3. **Review UI:** Legal reviewer reads DPA inline, approves or annotates sections with corrections. Corrections are stored as structured feedback.
4. **Feedback loop:** Corrections are used to update the DPA generation prompts for that tenant's language/sector combination. After 5 approved DPAs with no CRITICAL corrections, the tenant is moved to the automated tier (reviews every 30th DPA).
5. **Flagging criteria for manual review:**
   - Any DPA generated for a MINOR_DATA context (Section 9 risk)
   - Any DPA where `total_penalty_exposure_cr > 50` in the input audit report
   - Any DPA in a language where the translation model confidence < 0.80
   - First 5 DPAs for any new tenant regardless of above

**Quality SLA:**
- DPA generation (automated): < 10 minutes end-to-end
- Human review turnaround: < 2 business days (SLA for Growth/Enterprise tier)
- Post-review DPA delivery: < 1 hour after review approval
