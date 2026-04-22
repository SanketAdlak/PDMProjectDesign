# Agentic AI Design — Automatic Erasure Engine

**Version:** 1.0  
**Date:** 2026-04-22  

---

## 1. Why Agentic AI for Erasure?

Data deletion is not a CRUD operation. It requires:
- **Reasoning** about data relationships across heterogeneous systems
- **Planning** a safe deletion order (delete foreign key dependents before parents)
- **Detecting conflicts** between DPDPA obligations and other regulations
- **Adapting** when a system is unavailable or returns partial results
- **Verifying** that deletion actually happened (not just that the API returned 200)
- **Explaining** its decisions in language a DPO (not an engineer) can understand

A rules-based approach fails because every tenant's data architecture is different. A static deletion script fails because systems change. An agentic approach succeeds because it reasons about the specific situation at the time of execution.

---

## 2. Multi-Agent Architecture

The Erasure Engine uses a **pipeline of specialised agents**, each with a narrow responsibility. Agents communicate via a shared context object (the `ErasurePlan`). No agent can skip its turn — the pipeline enforces ordering.

```
┌─────────────────────────────────────────────────────────────────────┐
│                     ErasureOrchestrator                              │
│           (Temporal.io workflow — durable, multi-year capable)        │
│                                                                       │
│  Input: ErasureRequest (dpId, tenantId, triggeredBy, urgency)        │
│                                                                       │
│  ┌──────────────┐                                                     │
│  │ 1. Planner   │  Reads Module 2 Data Flow Map.                     │
│  │    Agent     │  Creates deletion plan for each discovered system.  │
│  └──────┬───────┘  Output: ErasurePlan (systems, methods, order)     │
│         │                                                             │
│  ┌──────▼───────┐                                                     │
│  │ 2. Safety    │  Reviews plan against conflict library.             │
│  │    Agent     │  Flags high-risk items. Detects regulatory blocks.  │
│  └──────┬───────┘  Output: ErasurePlan + RiskAnnotations             │
│         │                                                             │
│  ┌──────▼───────┐  (Only if Safety Agent flagged high-risk items)    │
│  │ 3. Human     │  Presents plan + risks to DPO for approval.        │
│  │    Approval  │  Blocks execution until approved or timeout.        │
│  │    Gate      │  Output: ApprovalDecision (APPROVED/REJECTED/MODIFIED)│
│  └──────┬───────┘                                                     │
│         │                                                             │
│  ┌──────▼───────┐                                                     │
│  │ 4. Executor  │  Calls deletion API for each approved system.      │
│  │    Agent     │  Handles retries, timeouts, partial failures.       │
│  └──────┬───────┘  Output: ExecutionResults (per-system status)      │
│         │                                                             │
│  ┌──────▼───────┐                                                     │
│  │ 5. Verifier  │  Re-queries each system to confirm deletion.       │
│  │    Agent     │  Uses different API path than deletion.             │
│  └──────┬───────┘  Output: VerificationResults                       │
│         │                                                             │
│  ┌──────▼───────┐                                                     │
│  │ 6. Certifier │  Generates ECDSA-signed deletion certificate.      │
│  │    Agent     │  Notifies DP in their language. Stores 7yr.        │
│  └──────────────┘  Output: DeletionCertificate                       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Agent Specifications

### 3.1 Planner Agent

**Responsibility:** Transform the Module 2 Data Flow Map into an actionable, ordered deletion plan.

**Inputs:**
- `dpIdentifierHash` — the DP to erase
- `tenantDataFlowMap` — Module 2 output: `{system_id, data_categories, estimated_records, last_seen}[]`
- `tenantSystemConfig` — how to connect to each system (API credentials, endpoints)
- `tenantErasurePolicies` — custom rules set by DPO (e.g., "always manual for Tally")

**Outputs:** `ErasurePlan`
```typescript
interface ErasurePlan {
  planId: string;
  dpIdentifierHash: string;
  tenantId: string;
  generatedAt: string;
  
  systems: ErasureSystemPlan[];
  executionOrder: string[];  // system IDs in safe deletion order
  estimatedDuration: number; // seconds
  
  plannerReasoning: string;  // Natural language explanation of decisions
}

interface ErasureSystemPlan {
  systemId: string;
  systemName: string;          // 'zoho_crm', 'razorpay', 'tenant_postgres'
  systemType: SystemType;
  dataCategories: string[];    // What data exists here for this DP
  estimatedRecords: number;
  
  deletionMethod: 'AUTO_API' | 'AUTO_DB' | 'CONNECTOR' | 'MANUAL';
  deletionEndpoint?: string;
  deletionQuery?: string;      // If AUTO_DB
  
  riskLevel: 'LOW' | 'MEDIUM' | 'HIGH' | 'MANUAL_REQUIRED';
  riskReason?: string;         // Why this risk level
  
  dependencies: string[];      // Other system IDs that must be deleted first
  estimatedDuration: number;   // seconds
  
  requiresApproval: boolean;   // Does this item need human sign-off?
}
```

**LLM Prompt Template:**
```
You are a DPDPA compliance deletion planner. Your job is to create a safe, 
complete data deletion plan for a Data Principal.

Data Principal identifier hash: {dpIdentifierHash}
Data found by discovery engine:
{dataFlowMap}

Tenant system configuration:
{systemConfig}

Regulatory conflict library:
{conflictLibrary}

Instructions:
1. For each system where data was found, determine the safest deletion method.
2. Order deletions to respect data dependencies (delete foreign key children first).
3. Flag any data that conflicts with Indian regulations (RBI, GST, SEBI, etc.).
4. Explain your reasoning in plain English for the DPO.
5. Mark any system where deletion should be manual (financial, medical, high-volume).

Output valid JSON matching ErasurePlan schema. Do not delete anything — plan only.
```

### 3.2 Safety Agent

**Responsibility:** Review the Planner's output for risks the planner may have missed. Acts as an independent check.

**Key safety checks:**
1. **Regulatory conflict detection**: Cross-reference each data category against the Conflicts Library
2. **Volume check**: Flag if total records > threshold
3. **Sensitive category check**: Health, biometric, children's data, financial KYC
4. **Business logic check**: Would deleting from system X break system Y (undiscovered dependency)?
5. **Identity verification**: Confirm dpIdentifierHash matches across all system plans
6. **Duplicate deletion guard**: Check if an erasure for this DP+tenant was recently completed

**Outputs:** Risk-annotated `ErasurePlan` + list of `SafetyFlag`
```typescript
interface SafetyFlag {
  systemId: string;
  flagType: 'REGULATORY_CONFLICT' | 'HIGH_VOLUME' | 'SENSITIVE_DATA' 
          | 'BUSINESS_CRITICAL' | 'IDENTITY_MISMATCH' | 'RECENT_DUPLICATE';
  severity: 'WARN' | 'BLOCK' | 'REQUIRE_APPROVAL';
  detail: string;         // Human-readable explanation
  regulation?: string;    // If REGULATORY_CONFLICT: "RBI Payment System Regulations §X"
  recommendation: string; // What the DPO should do about this
}
```

**Safety rules (declarative, not LLM-evaluated):**
```typescript
// Hard rules — LLM cannot override these
const HARD_SAFETY_RULES: SafetyRule[] = [
  {
    id: 'PAYMENT_RECORDS',
    condition: (plan) => plan.dataCategories.includes('payment_records'),
    action: 'BLOCK',
    message: 'Payment records require manual deletion due to RBI 5yr retention rule',
    regulation: 'RBI Payment System Regulations 2018'
  },
  {
    id: 'DUPLICATE_ERASURE',
    condition: async (plan) => await wasRecentlyErased(plan.dpId, plan.tenantId, 30),
    action: 'REQUIRE_APPROVAL',
    message: 'Erasure for this DP was completed within 30 days — verify this is intentional'
  },
  {
    id: 'CHILDREN_DATA',
    condition: (plan) => plan.isMinor,
    action: 'REQUIRE_APPROVAL',
    message: 'Children\'s data deletion requires guardian notification under Section 9'
  },
];

// Soft rules — LLM provides context but human decides
const SOFT_SAFETY_RULES: SafetyRule[] = [
  {
    id: 'HIGH_VOLUME',
    condition: (plan) => plan.totalRecords > 1000,
    action: 'REQUIRE_APPROVAL',
    message: 'High-volume deletion (>1000 records) requires human approval'
  },
];
```

### 3.3 Executor Agent

**Responsibility:** Execute approved deletions in the planned order. Handle failures gracefully.

**Execution loop per system:**
```typescript
async function executeSystemDeletion(
  system: ErasureSystemPlan,
  plan: ErasurePlan,
  connector: SystemConnector
): Promise<SystemExecutionResult> {
  const startTime = Date.now();
  
  let attempt = 0;
  const maxAttempts = 3;
  
  while (attempt < maxAttempts) {
    try {
      // Call the system's deletion endpoint
      const result = await connector.delete({
        systemId: system.systemId,
        dpIdentifierHash: plan.dpIdentifierHash,
        dataCategories: system.dataCategories,
        deletionQuery: system.deletionQuery,
      });
      
      return {
        systemId: system.systemId,
        status: 'DELETED',
        recordsDeleted: result.recordsDeleted,
        deletionTimestamp: new Date().toISOString(),
        durationMs: Date.now() - startTime,
        attemptNumber: attempt + 1,
      };
      
    } catch (error) {
      attempt++;
      if (attempt >= maxAttempts) {
        return {
          systemId: system.systemId,
          status: 'FAILED',
          error: error.message,
          durationMs: Date.now() - startTime,
          attemptNumber: attempt,
        };
      }
      // Exponential backoff: 2s, 4s, 8s
      await sleep(Math.pow(2, attempt) * 1000);
    }
  }
}
```

**Emergency stop handling:**
```typescript
// Temporal.io signal — sent by DPO via UI or API
export async function handleEmergencyStop(signal: EmergencyStopSignal): Promise<void> {
  console.log(`Emergency stop received: ${signal.reason}`);
  emergencyStopFlag = true;  // Checked before each system deletion
  
  await updateErasureStatus('PAUSED', signal.reason);
  await notifyDPO('Erasure paused — emergency stop triggered');
}

// In execution loop — check flag before each system
if (emergencyStopFlag) {
  break; // Stop processing remaining systems
}
```

### 3.4 Verifier Agent

**Responsibility:** Independently confirm that data was actually deleted from each system. Does NOT trust the Executor's 200 OK response.

**Verification approach per system type:**

| System Type | Verification Method |
|-------------|-------------------|
| Relational DB | `SELECT COUNT(*) WHERE dp_identifier = ?` → expect 0 |
| Zoho CRM | `GET /contacts/{id}` → expect 404 |
| Razorpay | `GET /customers/{id}` → expect 404 or masked fields |
| WhatsApp | Contact lookup → expect not found |
| S3 | `ListObjectsV2 --prefix dp/{hash}` → expect 0 objects |
| Tally | Connector query → expect no matching records |

**Verification agent prompt:**
```
You are a deletion verification agent. For each system below, you have:
1. The deletion that was executed
2. The verification query to run
3. The verification result

Determine if each deletion was successful. Be strict — any data that still 
exists for this DP identifier is a failure, even if it seems minor.

For each system, output: VERIFIED, FAILED, or PARTIAL (explain what remains).
```

### 3.5 Certifier Agent

**Responsibility:** Assemble all agent outputs into a signed, DPBI-submittable deletion certificate. Send notification to Data Principal.

**Certificate assembly:**
```typescript
interface DeletionCertificate {
  version: '2.0';
  certificateId: string;
  erasureOperationId: string;
  
  // Identity
  tenantId: string;
  tenantLegalName: string;
  dpIdentifierHash: string;
  
  // Summary
  totalSystemsTargeted: number;
  systemsDeleted: number;
  systemsFailed: number;
  systemsExcluded: number;     // Regulatory conflicts
  
  // Per-system evidence
  systems: Array<{
    systemName: string;
    status: 'DELETED' | 'FAILED' | 'EXCLUDED' | 'MANUAL_REQUIRED';
    recordsDeleted?: number;
    deletionTimestamp?: string;
    verificationStatus: 'VERIFIED' | 'FAILED' | 'PARTIAL';
    exclusionReason?: string;   // Why excluded (regulatory conflict)
    exclusionRegulation?: string;
  }>;
  
  // AI agent chain of reasoning (full transparency)
  agentTrace: {
    plannerOutput: string;      // Planner's reasoning
    safetyFlags: SafetyFlag[];  // What safety agent flagged
    approvalDecision?: string;  // What DPO approved/modified
    executorLog: string;        // Step-by-step execution log
    verifierFindings: string;   // Verifier's findings per system
  };
  
  // Compliance metadata
  dpdpaSection: 'Section 12 (Right to Erasure)';
  thirdScheduleClass: string;
  initiatedAt: string;
  completedAt: string;
  
  // Signature (tamper-evident)
  signature: string;            // ECDSA P-256 (tenant's KMS key)
  signingKeyId: string;
  issuedAt: string;
  
  // DP notification
  dpNotifiedAt: string;
  dpNotificationLanguage: string;
}
```

---

## 4. LLM Selection and Hosting

**Requirement:** Agents must run on-premise in Indian cloud (AWS ap-south-1). No data sent to external LLM APIs.

The erasure plan contains:
- Data category names (potentially sensitive: "health_records", "biometric_data")
- System architecture details (internal API endpoints, database schemas)
- DP's pseudonymised identifier

Sending this to OpenAI/Anthropic external APIs is not acceptable.

**Architecture:**
```
Model: Fine-tuned Mistral 7B (or similar) — instruction-tuned for legal/compliance
Hosting: AWS SageMaker (ap-south-1) with ml.g5.2xlarge instances
Context window: 32K tokens (sufficient for 20-system data flow maps)
Inference latency target: < 20s per agent call
Quantisation: 4-bit GGUF for cost efficiency (quality acceptable for structured output)
```

**Agent framework:** LangGraph (graph-based agent orchestration)
- Nodes = agents
- Edges = conditional routing (e.g., skip Human Approval if Safety Agent finds no flags)
- State = ErasurePlan (shared across all agents)
- Persistence = Temporal.io (durable state across restarts)

---

## 5. System Connectors

Each connected system requires a **Connector** — an adapter that translates generic deletion/verification commands into system-specific API calls.

```typescript
// connector/base.ts
export interface SystemConnector {
  systemId: string;
  systemName: string;
  
  // Discover what data exists for a DP (called by Module 2 — read only)
  discover(dpHash: string): Promise<DiscoveryResult>;
  
  // Delete DP data from this system
  delete(request: DeletionRequest): Promise<DeletionResult>;
  
  // Verify deletion was successful
  verify(dpHash: string, categories: string[]): Promise<VerificationResult>;
  
  // Health check
  ping(): Promise<boolean>;
}

// Built-in connectors (v1.0)
export const CONNECTORS = {
  'zoho_crm': ZohoCRMConnector,
  'razorpay': RazorpayConnector,
  'tally': TallyConnector,
  'whatsapp_business': WhatsAppConnector,
  'aws_s3': S3Connector,
  'tenant_postgres': TenantPostgresConnector, // Generic REST adapter
  'google_workspace': GoogleWorkspaceConnector,
};

// Tenant-defined connectors (custom integrations)
// Tenants provide: endpoint URL, auth method, request format, response format
// Erasure Engine calls them via standardised HTTP adapter
```

---

## 6. Safety Guarantee Matrix

| Scenario | Behaviour |
|----------|-----------|
| Planner Agent returns no systems | Abort — cannot generate certificate for empty plan |
| Safety Agent flags REGULATORY_CONFLICT | System moves to EXCLUDED status; never attempted |
| Safety Agent flags REQUIRE_APPROVAL | Pause; wait for DPO approval (24hr timeout) |
| DPO rejects plan | Erasure cancelled; DP notified of delay; audit event logged |
| Executor call times out after 3 retries | System marked FAILED; continue with others |
| Emergency stop signal received | Halt remaining systems; generate partial certificate |
| Verifier finds data still present | Status = FAILED; DPO alerted; partial certificate |
| Wrong DP identifier detected | Hard abort — zero tolerance for identity errors |
| Cloud outage during 3yr timeline | Temporal.io resumes from last checkpoint |
| Tenant revokes connector credentials | Connector fails; Executor marks FAILED; DPO alerted |

---

## 7. Audit Trail (Per Agent Call)

Every agent invocation is logged with:
```typescript
interface AgentAuditEntry {
  agentName: 'PLANNER' | 'SAFETY' | 'EXECUTOR' | 'VERIFIER' | 'CERTIFIER';
  erasureOperationId: string;
  tenantId: string;
  dpIdentifierHash: string;
  
  invocationId: string;
  startedAt: string;
  completedAt: string;
  durationMs: number;
  
  inputSummary: string;   // Non-sensitive description of what was passed in
  outputSummary: string;  // Non-sensitive description of what came out
  
  llmModel?: string;      // Which model was used (if LLM agent)
  llmTokensInput?: number;
  llmTokensOutput?: number;
  
  success: boolean;
  errorMessage?: string;
}
```

This audit trail is:
- Included in the deletion certificate's `agentTrace`
- Stored in the audit log for 7 years
- Accessible to DPO via dashboard
- Queryable for DPBI during audit
