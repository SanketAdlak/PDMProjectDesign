# System Design — Automatic Erasure Engine

**Version:** 1.0  
**Date:** 2026-04-22  

---

## 1. System Context

The Erasure Engine sits at the intersection of two data flows:
1. **Inbound from Module 2 Discovery**: where a DP's data exists across all connected systems
2. **Inbound from Module 3.1 Consent Vault**: consent withdrawal or erasure request triggers

It produces:
1. **Outbound deletion calls** to all connected systems (via Connectors)
2. **Deletion certificates** stored in the audit ledger and delivered to DPO + DP
3. **Status events** consumed by Module 3.3 Regulatory War Room

---

## 2. Architecture Overview

### 2.1 Key Design Constraints

**Constraint 1: Dual-Role Prohibition (DPDPA Section 2(g))**  
The Erasure Engine is part of the Consent Manager's domain. It cannot have live access to Module 2's discovery engine during erasure execution (separate AWS account). Instead, it operates on a **snapshot** of the Data Flow Map produced by Module 2 and shared via a secure, time-limited API token.

**Constraint 2: Durable Multi-Year Workflows**  
For ECOMMERCE_GT_2CR / GAMING / SOCIAL class DFs, erasure may be scheduled 3 years in the future. The workflow must survive: application restarts, infrastructure failures, team changes, cloud region events.

**Constraint 3: Premium Safety**  
Every deletion operation must have explicit safety checks, approval gates, and post-deletion verification. A bug that deletes the wrong DP's data is a catastrophic failure. Safety is a first-class architectural concern.

**Constraint 4: On-Premise AI**  
Agent LLMs must run in AWS ap-south-1. No data sent to external AI APIs.

### 2.2 Architectural Style

Erasure Engine is a **workflow-first service** — Temporal.io is the system of record for operation state, not the database. The database stores results and certificates; Temporal stores execution state.

```
┌──────────────────────────────────────────────────────────────────────┐
│                     Erasure Engine Service                            │
│                                                                       │
│  ┌──────────────┐   ┌─────────────────────────────────────────────┐  │
│  │   REST API   │   │         Temporal.io Workflow Engine         │  │
│  │  (trigger,   │   │                                             │  │
│  │   status,    │   │  ErasureOrchestrator                        │  │
│  │   approve,   │   │  ├── PlannerActivity                        │  │
│  │   cert)      │   │  ├── SafetyActivity                         │  │
│  └──────┬───────┘   │  ├── HumanApprovalSignal (wait signal)      │  │
│         │           │  ├── ExecutorActivity (per system)          │  │
│         ▼           │  ├── VerifierActivity (per system)          │  │
│  ┌──────────────┐   │  └── CertifierActivity                      │  │
│  │  Kafka       │   └──────────────────┬──────────────────────────┘  │
│  │  Consumer    │                      │                              │
│  │  (consent.   │                      ▼                              │
│  │  withdrawn,  │   ┌─────────────────────────────────────────────┐  │
│  │  erasure.*) │   │            Infrastructure Layer              │  │
│  └──────────────┘   │                                             │  │
│                      │  ┌────────────┐  ┌───────┐  ┌──────────┐  │  │
│                      │  │ PostgreSQL  │  │  KMS  │  │SageMaker │  │  │
│                      │  │ (plans,    │  │(cert  │  │(on-prem  │  │  │
│                      │  │  certs,    │  │ sign) │  │ LLM)     │  │  │
│                      │  │  audit)    │  └───────┘  └──────────┘  │  │
│                      │  └────────────┘                            │  │
│                      │  ┌─────────────────────────────────────┐   │  │
│                      │  │  System Connectors (per integration) │   │  │
│                      │  │  Zoho │ Razorpay │ Tally │ S3 │ ...  │   │  │
│                      │  └─────────────────────────────────────┘   │  │
│                      └─────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 3. Data Flow: Erasure Triggered by Consent Withdrawal

```
Consent Vault (Module 3.1)
        │
        │ Kafka event: consent.withdrawn
        │ { tenantId, dpIdentifierHash, purposeId }
        ▼
Erasure Engine Kafka Consumer
        │
        │ 1. Check if erasure already scheduled/completed for this DP+purpose
        │ 2. Determine df_class (from tenant config)
        │ 3. Fetch Data Flow Map from Module 2 (one-time snapshot, time-limited token)
        ▼
Temporal.io: StartWorkflow(ErasureOrchestrator)
        │
        ├── [IMMEDIATE] For STANDARD class DFs
        │       └── Execute immediately (skip sleep)
        │
        └── [SCHEDULED] For ECOMMERCE_GT_2CR / GAMING / SOCIAL
                └── Sleep until 3yr from last activity - 48hr
                        │
                    48hr before execution:
                        ├── Send DP notification in their language
                        └── Wait 48hr (cancellation window)
                                │
                            Execute ErasurePlan
```

---

## 4. Data Flow: Manual Erasure Request (Section 12)

```
DPO Dashboard / API Call
        │
        │ POST /v1/erasure/plan
        │ { tenantId, dpIdentifier, triggeredBy: 'DP_REQUEST' }
        ▼
Erasure Engine API
        │
        │ 1. Hash dpIdentifier (HMAC-SHA256)
        │ 2. Fetch Data Flow Map from Module 2
        │ 3. StartWorkflow(ErasureOrchestrator, skipSleep: true)
        ▼
        ┌──────────────────────────────────┐
        │  Planner Agent (LLM on SageMaker) │
        │  → Generates ErasurePlan         │
        └──────────────────────────────────┘
                │
                ▼
        ┌──────────────────────────────────┐
        │  Safety Agent (Rules + LLM)      │
        │  → Annotates risks, flags blocks  │
        └──────────────────────────────────┘
                │
                ▼
        ┌──────────────────────────────────┐
        │  DPO sees plan in dashboard      │
        │  "25 systems, 1,847 records"     │
        │  ⚠️ Razorpay: Manual (RBI rule)  │
        │  [Approve] [Reject] [Dry-run]    │
        └──────────────────────────────────┘
                │ DPO approves
                ▼
        ┌──────────────────────────────────┐
        │  Executor Agent                  │
        │  Deletes system by system        │
        │  (real-time progress in UI)      │
        └──────────────────────────────────┘
                │
                ▼
        ┌──────────────────────────────────┐
        │  Verifier Agent                  │
        │  Confirms deletion per system    │
        └──────────────────────────────────┘
                │
                ▼
        ┌──────────────────────────────────┐
        │  Certifier Agent                 │
        │  ECDSA-signed certificate        │
        │  DP notified in their language   │
        └──────────────────────────────────┘
```

---

## 5. Module 2 Integration (Data Flow Map)

The Erasure Engine consumes Module 2's discovery output but does NOT call Module 2 live during erasure execution (architectural isolation).

### 5.1 Data Flow Map Schema (Module 2 output)

```typescript
// Produced by Module 2 Discovery Engine, consumed by Erasure Engine
interface DataFlowMap {
  tenantId: string;
  dpIdentifierHash: string;
  lastDiscoveredAt: string;    // When this map was last generated
  confidence: number;          // 0-1: how complete is this map?
  
  systems: DataFlowSystemEntry[];
}

interface DataFlowSystemEntry {
  systemId: string;
  systemName: string;
  systemType: 'crm' | 'payment' | 'database' | 'storage' | 'messaging' | 'analytics';
  
  dataFound: boolean;
  dataCategories: string[];       // e.g., ['phone', 'email', 'purchase_history']
  estimatedRecords: number;
  lastActivity: string;           // ISO 8601 — used for 3yr timeline calculation
  
  locationDetails: {
    table?: string;               // If database
    bucket?: string;              // If S3
    contactId?: string;           // If CRM
    // System-specific location metadata
  };
}
```

### 5.2 Snapshot Mechanism

```typescript
// Module 2 exposes a time-limited snapshot endpoint (not live scanning)
// Erasure Engine calls this once per erasure operation

async function fetchDataFlowMapSnapshot(
  tenantId: string,
  dpIdentifierHash: string
): Promise<DataFlowMap> {
  // One-time token (15-minute TTL) from Module 2's auth service
  // This is the only cross-account communication allowed
  const token = await module2Auth.getErasureToken(tenantId);
  
  const response = await module2ApiClient.get('/v1/data-flow-map/snapshot', {
    headers: { 'X-Erasure-Token': token },
    params: { tenantId, dpIdentifierHash },
  });
  
  return response.data;
}
```

---

## 6. Connector Architecture

### 6.1 Connector Interface

```typescript
export abstract class BaseConnector implements SystemConnector {
  abstract systemId: string;
  abstract systemName: string;
  
  abstract delete(request: DeletionRequest): Promise<DeletionResult>;
  abstract verify(dpHash: string, categories: string[]): Promise<VerificationResult>;
  
  async ping(): Promise<boolean> {
    try {
      await this.healthCheck();
      return true;
    } catch {
      return false;
    }
  }
  
  // Built-in: retry with exponential backoff
  protected async withRetry<T>(
    fn: () => Promise<T>,
    maxAttempts = 3
  ): Promise<T> {
    for (let i = 0; i < maxAttempts; i++) {
      try {
        return await fn();
      } catch (err) {
        if (i === maxAttempts - 1) throw err;
        await sleep(Math.pow(2, i + 1) * 1000);
      }
    }
    throw new Error('unreachable');
  }
}
```

### 6.2 Built-in Connector: Razorpay

```typescript
export class RazorpayConnector extends BaseConnector {
  systemId = 'razorpay';
  systemName = 'Razorpay';
  
  async delete(request: DeletionRequest): Promise<DeletionResult> {
    // Step 1: Find customer by DP hash (using phone/email index)
    const customerId = await this.findCustomerId(request.dpIdentifierHash);
    if (!customerId) {
      return { status: 'NOT_FOUND', recordsDeleted: 0 };
    }
    
    // Step 2: Safety check — do not delete if active subscriptions or pending payments
    const hasActiveOrders = await this.checkActiveOrders(customerId);
    if (hasActiveOrders) {
      return {
        status: 'REQUIRES_MANUAL',
        reason: 'Active subscriptions or pending payments exist — contact finance team'
      };
    }
    
    // Step 3: Delete customer record
    // Note: Razorpay retains transaction records for RBI compliance (5yr)
    // We can only delete PII fields, not the transaction records themselves
    await this.withRetry(() =>
      this.razorpayClient.customers.delete(customerId)
    );
    
    return {
      status: 'DELETED',
      recordsDeleted: 1,
      note: 'Transaction history retained per RBI regulations (5yr). Customer PII anonymised.'
    };
  }
  
  async verify(dpHash: string): Promise<VerificationResult> {
    const customerId = await this.findCustomerId(dpHash);
    return {
      verified: customerId === null,
      residualData: customerId ? ['customer_record'] : [],
    };
  }
}
```

### 6.3 Generic REST Connector (Tenant-Defined)

For tenants' own systems that don't have a built-in connector:

```typescript
// Tenant provides this config in their dashboard
interface GenericConnectorConfig {
  systemId: string;
  systemName: string;
  deletionEndpoint: string;  // e.g., 'https://api.myapp.com/users/{dp_hash}/erase'
  authType: 'BEARER' | 'API_KEY' | 'BASIC';
  authCredentialRef: string; // AWS Secrets Manager secret name
  httpMethod: 'DELETE' | 'POST';
  requestBody?: object;      // Template: { "userId": "{dp_hash}", "hardDelete": true }
  verificationEndpoint: string;
  verificationSuccessCondition: 'HTTP_404' | 'FIELD_NULL';
}
```

---

## 7. SageMaker LLM Deployment

```yaml
# SageMaker endpoint config (Terraform)
resource: aws_sagemaker_endpoint
endpoint_name: erasure-engine-llm
region: ap-south-1

model:
  image: 123456789.dkr.ecr.ap-south-1.amazonaws.com/llm-inference:v1
  model_data: s3://truststack-models/erasure-planner-mistral-7b-4bit/
  
instance_type: ml.g5.2xlarge    # 24GB GPU VRAM, sufficient for 7B 4-bit model
initial_instance_count: 2        # HA — 2 instances
scaling:
  min_capacity: 1
  max_capacity: 5
  target_invocations_per_minute: 30

# Security: VPC-only endpoint (no public internet)
vpc_config:
  subnet_ids: [private-subnet-a, private-subnet-b]
  security_group_ids: [sagemaker-sg]
```

**Model serving:**
- Input: structured JSON prompt (planner/safety/verifier templates)
- Output: structured JSON (ErasurePlan, SafetyFlags, VerificationResult)
- Guided generation via LMQL or Outlines (forces JSON schema compliance)
- Fallback: if LLM output fails schema validation 3 times → escalate to human review

---

## 8. Temporal.io Workflow Details

```typescript
// Full workflow definition
export async function ErasureOrchestrator(input: ErasureInput): Promise<DeletionCertificate> {
  
  // Step 1: Fetch Data Flow Map from Module 2
  const dataFlowMap = await activities.fetchDataFlowMap(input);
  
  // Step 2: Generate deletion plan (Planner Agent)
  const plan = await activities.generateErasurePlan(dataFlowMap, input);
  
  // Step 3: Safety review (Safety Agent)
  const { annotatedPlan, safetyFlags } = await activities.runSafetyReview(plan);
  
  // Step 4: Human approval (if required)
  if (safetyFlags.some(f => f.severity === 'REQUIRE_APPROVAL')) {
    await activities.notifyDPOForApproval(annotatedPlan, safetyFlags);
    
    // Wait for approval signal (Temporal.io signal)
    const approval = await workflow.condition(
      () => approvalReceived !== null,
      { timeout: '24 hours' }
    );
    
    if (!approval || approval.decision === 'REJECTED') {
      await activities.cancelErasure(plan, approval?.reason);
      throw new ApplicationFailure('Erasure rejected by DPO');
    }
  }
  
  // Step 5: 48hr notification (DPDPA required)
  await activities.sendPreDeletionNotification(input.dpIdentifier, input.language);
  await sleep('48 hours');  // Temporal timer — durable, survives restarts
  
  // Check if DP cancelled during the window
  if (cancellationReceived) {
    await activities.cancelErasure(plan, 'DP cancelled during 48hr window');
    return;
  }
  
  // Step 6: Execute deletions (per system, in order)
  const executionResults: SystemExecutionResult[] = [];
  for (const system of annotatedPlan.executionOrder) {
    if (emergencyStopFlag) break;  // Honour emergency stop
    
    const result = await activities.deleteFromSystem(system, plan);
    executionResults.push(result);
  }
  
  // Step 7: Verify deletions (Verifier Agent)
  const verificationResults = await activities.verifyDeletions(
    executionResults.filter(r => r.status === 'DELETED'), 
    plan.dpIdentifierHash
  );
  
  // Step 8: Generate and store certificate (Certifier Agent)
  const certificate = await activities.generateCertificate({
    plan, executionResults, verificationResults, input,
  });
  
  await activities.storeCertificate(certificate);
  await activities.notifyDP(input.dpIdentifier, input.language, certificate);
  await activities.emitKafkaEvent('erasure.executed', certificate);
  
  return certificate;
}
```

---

## 9. Observability

### 9.1 Key Metrics

| Metric | Alert |
|--------|-------|
| `erasure_plan_generation_duration_ms` | p99 > 30s |
| `erasure_executor_system_success_rate` | < 95% |
| `erasure_verifier_failed_total` | > 0 (every unverified deletion is a finding) |
| `erasure_approval_timeout_total` | > 5/day (DPO not reviewing on time) |
| `llm_inference_duration_ms` | p99 > 25s |
| `erasure_workflow_failures_total` | > 0 (zero tolerance) |
| `connector_timeout_total` | Per connector — track worst connectors |

### 9.2 DPO Dashboard Metrics

- Erasures completed this month / this year
- Average time from trigger to completion
- Systems with highest failure rate
- Upcoming scheduled erasures (30/90/180 days)
- Regulatory conflicts detected (by regulation)
- Agent reasoning accuracy (DPO acceptance rate of plans)
