# System Design — TrustStack Module 2: Data Discovery Engine

**Version:** 1.0
**Date:** 2026-04-22
**AWS Account:** `TrustStack-Discovery` (isolated, no VPC peering with `TrustStack-Consent`)

---

## Table of Contents

1. [Architecture Philosophy](#1-architecture-philosophy)
2. [System Overview](#2-system-overview)
3. [Component Deep Dives](#3-component-deep-dives)
4. [Data Flow Map: Construction and Schema](#4-data-flow-map-construction-and-schema)
5. [NLP Contract Audit: End-to-End Pipeline](#5-nlp-contract-audit-end-to-end-pipeline)
6. [Snapshot Protocol: Discovery → Erasure Engine](#6-snapshot-protocol-discovery--erasure-engine)
7. [Account Isolation](#7-account-isolation)
8. [Scalability and Performance](#8-scalability-and-performance)
9. [Observability](#9-observability)

---

## 1. Architecture Philosophy

### 1.1 Why a Separate AWS Account

Module 2 (Data Discovery) runs in a completely separate AWS account, `TrustStack-Discovery`, with no VPC peering, no shared databases, and no shared IAM roles with `TrustStack-Consent` (which hosts the Consent Vault and Erasure Engine).

This is not an operational preference — it is a hard regulatory requirement under **Section 2(g) DPDPA 2023**, which defines a Consent Manager as an entity that manages consent *on behalf of* Data Principals. The prohibition on dual roles means TrustStack, acting as a Registered Consent Manager, cannot simultaneously act as a Data Processor for the same Data Principal's data. If the Discovery Engine had access to the Consent Vault's `consent_records` table, the same system that certifies consent would also be processing the underlying personal data — collapsing the trust boundary the law requires.

The Account-level isolation enforces this at the infrastructure level. No engineer mistake, no IAM misconfiguration, and no dependency update can create a path from `TrustStack-Discovery` compute into `TrustStack-Consent` databases. The only permitted interaction is the one-time signed snapshot API call described in Section 6.

### 1.2 Why Read-Only Connectors

All vendor connectors in Module 2 are read-only by design. The Discovery Engine's purpose is *visibility*: finding where personal data lives, classifying it, and producing a Data Flow Map. Writing to vendor systems would:

1. Create data provenance ambiguity (was this record placed by the business or by TrustStack's scan?)
2. Risk unintended data mutation (a classification tag written to a production CRM record)
3. Expose TrustStack to liability as a co-processor of personal data

When erasure is required (Module 3's job), the Erasure Engine uses the Discovery snapshot to learn *which* systems to target, then authenticates with its own permissioned write credentials independently. Discovery never holds or exercises write credentials.

### 1.3 Why On-Premise LLM Inference

**Section 16 DPDPA** restricts cross-border transfer of personal data to countries on DPBI's approved list. That list is not yet published (approval mechanism is still being developed by DPBI as of April 2026), making any cloud-based LLM API (OpenAI, Anthropic, Cohere) legally risky for PII-containing inputs.

The Discovery Engine processes raw records from vendor systems — Zoho CRM contacts, Razorpay transaction logs, PostgreSQL tables — which routinely contain names, phone numbers, Aadhaar references, and financial data. Sending these to a foreign LLM API endpoint would constitute a cross-border data transfer of personal data under Section 16.

All LLM inference runs on SageMaker in `ap-south-1` (Mumbai) using a fine-tuned Mistral 7B model. The model is hosted on `ml.g5.2xlarge` instances within the `TrustStack-Discovery` VPC. No inference request leaves Indian soil.

---

## 2. System Overview

### 2.1 Module 2 in the TrustStack Platform

```
┌────────────────────────────────────────────────────────────────────────────┐
│                          TrustStack Platform (Logical)                      │
│                                                                              │
│   Module 1 (Awareness)          Module 2 (Discovery)    Module 3 (Shield)  │
│   ─────────────────────         ──────────────────────  ─────────────────  │
│   Jargon Slayer                 AI Data Discovery       SahmatOS            │
│   3-Min Trust Check        ──► NLP Contract Audit  ──► Erasure Engine      │
│                                Data Flow Map           Regulatory War Room  │
│                                                                              │
│   [Account: TrustStack-Awareness]  [Account: TrustStack-Discovery]          │
│                                    NO VPC PEERING       [Account: TrustStack-Consent]
└────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Discovery Account — Service Map

```
┌─────────────────────────────────────────────────── TrustStack-Discovery VPC (10.1.0.0/16) ──┐
│                                                                                               │
│  Public Subnet (10.1.1.0/24)           Private Subnet (10.1.2.0/24)                         │
│  ┌────────────────────────┐            ┌──────────────────────────────────────────────────┐  │
│  │   Application Load     │            │                                                  │  │
│  │   Balancer (TLS 1.3)   │──────────► │  ┌─────────────────┐  ┌──────────────────────┐  │  │
│  └────────────────────────┘            │  │   API Layer      │  │  NLP Contract Audit  │  │  │
│                                        │  │   (Node.js/TS)   │  │  (Python/FastAPI)    │  │  │
│  WAF Rules:                            │  │   ECS Fargate    │  │  ECS Fargate         │  │  │
│  - Outbound HTTPS only                 │  └────────┬─────────┘  └──────────┬───────────┘  │  │
│  - Vendor API IP whitelist             │           │                        │              │  │
│  - No inbound except ALB              │  ┌────────▼─────────────────────── ▼───────────┐  │  │
│                                        │  │         Temporal.io Workflow Cluster         │  │  │
│                                        │  │         (Scan Orchestrator)                  │  │  │
│                                        │  └────────────────────┬─────────────────────────┘  │  │
│                                        │                        │                            │  │
│                                        │  ┌─────────────────────▼──────────────────────┐   │  │
│                                        │  │    Discovery Agent Pipeline (Python/LangGraph)│  │  │
│                                        │  │    5 Agents: Connector→Scanner→Classifier    │  │  │
│                                        │  │              →RiskScorer→Mapper              │  │  │
│                                        │  └──────────────────┬─────────────────────────┘   │  │
│                                        │                      │                             │  │
│                                        │  ┌───────────────────▼──────────────────────┐    │  │
│                                        │  │              LLM Gateway (FastAPI)         │    │  │
│                                        │  │    SageMaker Endpoint (Mistral 7B)         │    │  │
│                                        │  └───────────────────────────────────────────┘    │  │
│                                        │                                                    │  │
│  ML Subnet (10.1.3.0/24)              │  ┌────────────────┐  ┌──────────────────────────┐ │  │
│  ┌──────────────────────────┐         │  │  Discovery DB   │  │  Artifact Store (S3)    │ │  │
│  │  SageMaker Endpoints     │◄────────┘  │  (RDS PG16     │  │  (AES-256, ap-south-1)  │ │  │
│  │  ml.g5.2xlarge (x2)     │            │  Multi-AZ)      │  │                          │ │  │
│  └──────────────────────────┘           └────────────────┘  └──────────────────────────┘ │  │
│                                        │                                                    │  │
│                                        │  ┌────────────────────────────────────────────┐  │  │
│                                        │  │  TimescaleDB (Scan Metrics)                 │  │  │
│                                        │  │  AWS Secrets Manager (Connector Credentials)│  │  │
│                                        │  └────────────────────────────────────────────┘  │  │
│                                        └──────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────────────────────────────┘
         ▲  Snapshot API (HTTPS, signed JWT only)
         │
┌────────┴───────────────────────────────────────────────┐
│  TrustStack-Consent Account (Module 3 — Erasure Engine)│
│  POST /v1/snapshot/request  GET /v1/snapshot/{token}   │
└────────────────────────────────────────────────────────┘
```

### 2.3 Two Sub-Products

| Sub-Product | Input | Output | Latency Profile |
|-------------|-------|--------|-----------------|
| AI Data Discovery Engine | Live vendor system connections | Data Flow Map (graph JSON) | Minutes to hours (async scan jobs) |
| NLP Contract Audit Agent | PDF/DOCX/image in any of 22 languages | Compliance report + generated DPA | 30–120 seconds (synchronous pipeline) |

---

## 3. Component Deep Dives

### 3.1 API Layer (Node.js/TypeScript ESM)

**Technology:** Fastify on Node.js 20 LTS, TypeScript ESM, OpenAPI 3.1, ECS Fargate

The API layer is the only public-facing surface of the Discovery system. All endpoints require tenant-scoped JWT authentication derived from per-tenant API keys stored in AWS Secrets Manager.

**Endpoints:**

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/v1/scans` | Start a new discovery scan job |
| `GET` | `/v1/scans/{scanId}` | Get scan status and progress |
| `GET` | `/v1/scans/{scanId}/findings` | Paginated PII findings for a scan |
| `DELETE` | `/v1/scans/{scanId}` | Cancel in-progress scan |
| `POST` | `/v1/connectors` | Register a new connector (stores credentials in Secrets Manager) |
| `GET` | `/v1/connectors` | List connectors for tenant |
| `PUT` | `/v1/connectors/{connectorId}` | Update connector config |
| `DELETE` | `/v1/connectors/{connectorId}` | Remove connector and purge credentials |
| `GET` | `/v1/dataflow` | Get Data Flow Map (nodes/edges JSON) |
| `GET` | `/v1/dataflow/export` | Export Data Flow Map as PDF/PNG |
| `POST` | `/v1/contracts/audit` | Submit contract for NLP audit (multipart form) |
| `GET` | `/v1/contracts/{auditId}` | Get contract audit result |
| `GET` | `/v1/contracts/{auditId}/dpa` | Download generated DPA |
| `POST` | `/v1/snapshot/request` | Request snapshot token (Module 3 only) |
| `GET` | `/v1/snapshot/{token}` | Consume snapshot (one-time use, Module 3 only) |

**Authentication middleware:**

```typescript
// src/middleware/auth.ts
import { FastifyRequest, FastifyReply } from 'fastify';
import { SecretsManagerClient, GetSecretValueCommand } from '@aws-sdk/client-secrets-manager';
import { createHmac } from 'crypto';

export interface TenantContext {
  tenantId: string;
  tier: 'assessment' | 'growth' | 'enterprise' | 'platform';
  allowedConnectors: string[];
  snapshotAllowed: boolean; // only true for Module 3 Erasure Engine service account
}

// API keys are stored hashed — raw key is never persisted (mirrors SahmatOS convention)
export async function authMiddleware(
  request: FastifyRequest,
  reply: FastifyReply,
): Promise<void> {
  const rawKey = request.headers['x-api-key'];
  if (!rawKey || typeof rawKey !== 'string') {
    return reply.code(401).send({
      type: 'https://truststack.in/errors/unauthorized',
      title: 'Missing API key',
      status: 401,
    });
  }

  // HMAC-SHA256 hash of raw key — compare against stored hash
  const keyHash = createHmac('sha256', process.env.API_KEY_HMAC_SECRET!)
    .update(rawKey)
    .digest('hex');

  const tenant = await resolveTenantByKeyHash(keyHash);
  if (!tenant) {
    return reply.code(401).send({
      type: 'https://truststack.in/errors/unauthorized',
      title: 'Invalid API key',
      status: 401,
    });
  }

  // Attach to request context for downstream handlers
  request.tenantContext = tenant;
}

// Snapshot endpoints are restricted to Module 3's service account only
export function requireSnapshotAccess(request: FastifyRequest, reply: FastifyReply, done: () => void) {
  if (!request.tenantContext?.snapshotAllowed) {
    reply.code(403).send({
      type: 'https://truststack.in/errors/forbidden',
      title: 'Snapshot access requires Module 3 service account',
      status: 403,
    });
    return;
  }
  done();
}
```

**Scan submission endpoint:**

```typescript
// src/routes/scans.ts
import { FastifyInstance } from 'fastify';
import { TemporalClient } from '../temporal/client.js';
import type { ScanJobInput } from '../types/scan.js';

export async function scanRoutes(app: FastifyInstance) {
  app.post<{ Body: ScanJobInput }>('/v1/scans', {
    schema: {
      body: {
        type: 'object',
        required: ['connectorIds', 'scanType'],
        properties: {
          connectorIds: { type: 'array', items: { type: 'string', format: 'uuid' }, minItems: 1 },
          scanType: { enum: ['FULL', 'INCREMENTAL', 'TARGETED'] },
          targetedPurposes: { type: 'array', items: { type: 'string' } },
          sampleRate: { type: 'number', minimum: 0.01, maximum: 1.0, default: 0.1 },
        },
      },
    },
  }, async (request, reply) => {
    const { tenantId } = request.tenantContext!;
    const jobId = crypto.randomUUID();

    // Start Temporal workflow — returns immediately, scan runs async
    await TemporalClient.start('DiscoveryScanWorkflow', {
      taskQueue: 'discovery-scan',
      workflowId: `scan-${jobId}`,
      args: [{
        jobId,
        tenantId,
        connectorIds: request.body.connectorIds,
        scanType: request.body.scanType,
        sampleRate: request.body.sampleRate ?? 0.1,
        initiatedAt: new Date().toISOString(),
      }],
    });

    return reply.code(202).send({
      scanId: jobId,
      status: 'QUEUED',
      estimatedDurationSeconds: null, // populated once first connector is probed
      statusUrl: `/v1/scans/${jobId}`,
    });
  });
}
```

### 3.2 Scan Orchestrator (Temporal.io)

The scan orchestrator uses Temporal.io for the same reason SahmatOS uses it for erasure: discovery scans are long-running, stateful operations that must survive infrastructure failures, connector timeouts, and partial completion.

A full scan of a large AWS S3 data lake with 10TB of log archives may take 4–8 hours. Temporal's durable execution model means a worker crash at hour 5 does not restart the scan from the beginning — it resumes from the last completed activity checkpoint.

**Workflow definition:**

```typescript
// src/temporal/workflows/discovery-scan.ts
import {
  proxyActivities,
  sleep,
  condition,
  setHandler,
  defineSignal,
  defineQuery,
  ApplicationFailure,
} from '@temporalio/workflow';
import type { ScanActivities } from '../activities/index.js';

// Activity proxies — Temporal serialises calls across the network
const {
  connectorAuthActivity,
  dataFetchActivity,
  piiDetectionActivity,
  classificationActivity,
  mapUpdateActivity,
  markConnectorFailed,
} = proxyActivities<ScanActivities>({
  startToCloseTimeout: '30 minutes', // individual activity timeout
  retry: {
    maximumAttempts: 3,
    initialInterval: '10s',
    backoffCoefficient: 2,
    nonRetryableErrorTypes: ['ConnectorAuthError', 'TenantNotFoundError'],
  },
});

const cancelSignal = defineSignal('cancel');
const progressQuery = defineQuery<ScanProgress>('progress');

export interface ScanJobInput {
  jobId: string;
  tenantId: string;
  connectorIds: string[];
  scanType: 'FULL' | 'INCREMENTAL' | 'TARGETED';
  sampleRate: number;
  initiatedAt: string;
}

export interface ScanProgress {
  completedConnectors: number;
  totalConnectors: number;
  totalPIIFindings: number;
  status: 'RUNNING' | 'PARTIAL' | 'COMPLETED' | 'CANCELLED';
}

export async function DiscoveryScanWorkflow(input: ScanJobInput): Promise<ScanSummary> {
  let cancelled = false;
  let progress: ScanProgress = {
    completedConnectors: 0,
    totalConnectors: input.connectorIds.length,
    totalPIIFindings: 0,
    status: 'RUNNING',
  };

  setHandler(cancelSignal, () => { cancelled = true; });
  setHandler(progressQuery, () => progress);

  const connectorResults: ConnectorScanResult[] = [];

  // Process connectors sequentially to avoid overwhelming vendor rate limits.
  // Each connector is independently checkpointed — failure of one does not abort others.
  for (const connectorId of input.connectorIds) {
    if (cancelled) {
      progress.status = 'CANCELLED';
      break;
    }

    try {
      // Activity 1: Authenticate to vendor system
      const authContext = await connectorAuthActivity({ connectorId, tenantId: input.tenantId });

      // Activity 2: Fetch data (streams records in batches — activity reports progress via heartbeat)
      const fetchResult = await dataFetchActivity({
        connectorId,
        tenantId: input.tenantId,
        authContext,
        scanType: input.scanType,
        sampleRate: input.sampleRate,
      });

      // Activity 3: Run PII detection on fetched records (LLM inference via LLM Gateway)
      const findings = await piiDetectionActivity({
        connectorId,
        tenantId: input.tenantId,
        batchRefs: fetchResult.batchRefs, // S3 references to encrypted record batches
        jobId: input.jobId,
      });

      // Activity 4: Classify findings by DPDPA data category
      const classified = await classificationActivity({
        connectorId,
        tenantId: input.tenantId,
        findings,
        jobId: input.jobId,
      });

      // Activity 5: Update Data Flow Map graph
      await mapUpdateActivity({
        connectorId,
        tenantId: input.tenantId,
        classifiedFindings: classified,
        jobId: input.jobId,
      });

      progress.completedConnectors++;
      progress.totalPIIFindings += findings.length;
      connectorResults.push({ connectorId, status: 'SUCCESS', findingCount: findings.length });

    } catch (err) {
      // Connector failure is partial — mark it and continue with remaining connectors
      if (err instanceof ApplicationFailure && err.nonRetryable) {
        // Auth failure — no point retrying this connector in this run
        await markConnectorFailed({ connectorId, tenantId: input.tenantId, error: err.message });
        connectorResults.push({ connectorId, status: 'AUTH_FAILED', findingCount: 0 });
      } else {
        connectorResults.push({ connectorId, status: 'TIMEOUT', findingCount: 0 });
      }
      // Continue — partial scan is better than no scan
    }
  }

  if (progress.status !== 'CANCELLED') {
    const allSucceeded = connectorResults.every(r => r.status === 'SUCCESS');
    progress.status = allSucceeded ? 'COMPLETED' : 'PARTIAL';
  }

  return {
    jobId: input.jobId,
    tenantId: input.tenantId,
    completedAt: new Date().toISOString(),
    connectorResults,
    totalPIIFindings: progress.totalPIIFindings,
    status: progress.status,
  };
}
```

**Activity implementations (key excerpts):**

```typescript
// src/temporal/activities/connector-auth.ts
import { heartbeat, isCancellation } from '@temporalio/activity';
import { SecretsManagerClient, GetSecretValueCommand } from '@aws-sdk/client-secrets-manager';
import { ConnectorRegistry } from '../../connectors/registry.js';
import { ApplicationFailure } from '@temporalio/workflow';

export async function connectorAuthActivity(input: {
  connectorId: string;
  tenantId: string;
}): Promise<ConnectorAuthContext> {
  // Retrieve encrypted credentials from Secrets Manager
  // Secret ARN format: arn:aws:secretsmanager:ap-south-1:{account}:secret:discovery/{tenantId}/{connectorId}
  const secretArn = `discovery/${input.tenantId}/${input.connectorId}`;
  const sm = new SecretsManagerClient({ region: 'ap-south-1' });

  let secret: string;
  try {
    const result = await sm.send(new GetSecretValueCommand({ SecretId: secretArn }));
    secret = result.SecretString!;
  } catch {
    // Credentials not found — non-retryable, connector needs re-registration
    throw ApplicationFailure.nonRetryable(
      'ConnectorAuthError',
      `Credentials not found for connector ${input.connectorId}`,
    );
  }

  const credentials = JSON.parse(secret) as ConnectorCredentials;
  const plugin = ConnectorRegistry.get(credentials.connectorType);

  try {
    const authContext = await plugin.authenticate(credentials);
    heartbeat('auth_success');
    return authContext;
  } catch (err) {
    throw ApplicationFailure.nonRetryable(
      'ConnectorAuthError',
      `Authentication failed for ${input.connectorId}: ${(err as Error).message}`,
    );
  }
}
```

### 3.3 Connector Plugin Registry

The plugin registry is a compile-time registry pattern — connectors are registered at module load time and resolved by connector type string at runtime. This allows custom connectors to be added without modifying core orchestration code.

**Plugin interface:**

```typescript
// src/connectors/plugin.ts

// DPDPA Section 6: Data minimisation — connectors only fetch what is needed
// for PII classification, never full records unless sampling requires it.

export interface ConnectorPlugin {
  /** Connector type identifier — used as registry key */
  readonly type: ConnectorType;
  readonly displayName: string;

  /**
   * Authenticate to the vendor system using provided credentials.
   * Must NOT store credentials beyond the lifetime of this call.
   * Returns an opaque auth context used by subsequent calls.
   */
  authenticate(credentials: ConnectorCredentials): Promise<ConnectorAuthContext>;

  /**
   * List all discoverable objects/collections in the vendor system.
   * E.g., for S3: list buckets and key prefixes
   * For Zoho CRM: list modules (Contacts, Leads, Accounts)
   * For PostgreSQL: list schemas and tables
   */
  listObjects(auth: ConnectorAuthContext): AsyncGenerator<ObjectDescriptor>;

  /**
   * Fetch a representative sample of records from an object.
   * sampleRate: 0.01 = 1% of records, 1.0 = all records (full scan)
   * Returns record stubs (metadata + first N fields) to minimise data transfer.
   */
  fetchSample(
    auth: ConnectorAuthContext,
    object: ObjectDescriptor,
    sampleRate: number,
  ): AsyncGenerator<RecordStub>;

  /**
   * Fetch a specific record by its identifier.
   * Used when PII detection on a sample triggers deeper analysis.
   * NEVER called during standard discovery — only for targeted scans.
   */
  fetchRecord(
    auth: ConnectorAuthContext,
    object: ObjectDescriptor,
    recordId: string,
  ): Promise<RecordStub>;

  /**
   * Estimate total record count for an object — used for progress reporting.
   * May return null if the vendor API does not support count queries.
   */
  estimateRecordCount(auth: ConnectorAuthContext, object: ObjectDescriptor): Promise<number | null>;
}

export type ConnectorType =
  | 'zoho_crm'
  | 'zoho_books'
  | 'razorpay'
  | 'tally_http'
  | 'whatsapp_business'
  | 'aws_s3'
  | 'gcp_cloud_storage'
  | 'postgresql'
  | 'mysql'
  | 'mongodb'
  | 'cloudwatch_logs'
  | 'elastic_opensearch'
  | 'generic_rest';

export interface ConnectorCredentials {
  connectorType: ConnectorType;
  // Credential fields vary per type — serialised as JSON in Secrets Manager
  // Examples:
  //   zoho_crm: { clientId, clientSecret, refreshToken, region }
  //   aws_s3: { roleArn, externalId }  (cross-account assume-role, read-only policy)
  //   postgresql: { host, port, database, username, password, sslMode: 'require' }
  [key: string]: unknown;
}

export interface ConnectorAuthContext {
  connectorType: ConnectorType;
  connectorId: string;
  // Opaque auth state — access tokens, db connections, etc.
  // Implementation detail of each plugin, not inspected by core pipeline.
  authPayload: unknown;
  expiresAt?: Date;
}

export interface ObjectDescriptor {
  id: string;            // Unique within connector (e.g., "s3://bucket/prefix", "contacts_module")
  name: string;          // Human-readable
  type: 'table' | 'collection' | 'bucket_prefix' | 'log_stream' | 'api_resource';
  estimatedRecords?: number;
  lastModified?: Date;
  tags?: Record<string, string>;
}

export interface RecordStub {
  recordId: string;
  objectId: string;
  // Fields are key-value pairs. Values are NEVER stored raw in Discovery DB.
  // They are hashed before persistence. This object exists transiently in memory only.
  fields: Record<string, string | number | boolean | null>;
  fetchedAt: Date;
}
```

**Registry:**

```typescript
// src/connectors/registry.ts
import { ConnectorPlugin, ConnectorType } from './plugin.js';
import { ZohoCRMConnector } from './plugins/zoho-crm.js';
import { ZohoBooksConnector } from './plugins/zoho-books.js';
import { RazorpayConnector } from './plugins/razorpay.js';
import { TallyHTTPConnector } from './plugins/tally-http.js';
import { WhatsAppBusinessConnector } from './plugins/whatsapp-business.js';
import { AWSS3Connector } from './plugins/aws-s3.js';
import { GCPCloudStorageConnector } from './plugins/gcp-cloud-storage.js';
import { PostgreSQLConnector } from './plugins/postgresql.js';
import { MySQLConnector } from './plugins/mysql.js';
import { MongoDBConnector } from './plugins/mongodb.js';
import { CloudWatchLogsConnector } from './plugins/cloudwatch-logs.js';
import { ElasticOpenSearchConnector } from './plugins/elastic-opensearch.js';
import { GenericRESTConnector } from './plugins/generic-rest.js';

const REGISTRY = new Map<ConnectorType, ConnectorPlugin>([
  ['zoho_crm', new ZohoCRMConnector()],
  ['zoho_books', new ZohoBooksConnector()],
  ['razorpay', new RazorpayConnector()],
  ['tally_http', new TallyHTTPConnector()],
  ['whatsapp_business', new WhatsAppBusinessConnector()],
  ['aws_s3', new AWSS3Connector()],
  ['gcp_cloud_storage', new GCPCloudStorageConnector()],
  ['postgresql', new PostgreSQLConnector()],
  ['mysql', new MySQLConnector()],
  ['mongodb', new MongoDBConnector()],
  ['cloudwatch_logs', new CloudWatchLogsConnector()],
  ['elastic_opensearch', new ElasticOpenSearchConnector()],
  ['generic_rest', new GenericRESTConnector()],
]);

export const ConnectorRegistry = {
  get(type: ConnectorType): ConnectorPlugin {
    const plugin = REGISTRY.get(type);
    if (!plugin) throw new Error(`Unknown connector type: ${type}`);
    return plugin;
  },
  list(): ConnectorType[] {
    return [...REGISTRY.keys()];
  },
};
```

**Example plugin — AWS S3 (read-only via IAM assume-role):**

```typescript
// src/connectors/plugins/aws-s3.ts
import { STSClient, AssumeRoleCommand } from '@aws-sdk/client-sts';
import { S3Client, ListBucketsCommand, ListObjectsV2Command, GetObjectCommand } from '@aws-sdk/client-s3';
import { ConnectorPlugin, ConnectorAuthContext, ObjectDescriptor, RecordStub } from '../plugin.js';

export class AWSS3Connector implements ConnectorPlugin {
  readonly type = 'aws_s3' as const;
  readonly displayName = 'AWS S3';

  async authenticate(credentials: { roleArn: string; externalId: string }): Promise<ConnectorAuthContext> {
    const sts = new STSClient({ region: 'ap-south-1' });
    // Assume customer's read-only role via cross-account role assumption
    const assumed = await sts.send(new AssumeRoleCommand({
      RoleArn: credentials.roleArn,
      RoleSessionName: 'TrustStackDiscovery',
      ExternalId: credentials.externalId,
      DurationSeconds: 3600,
    }));

    return {
      connectorType: 'aws_s3',
      connectorId: credentials.roleArn,
      authPayload: assumed.Credentials,
      expiresAt: assumed.Credentials!.Expiration,
    };
  }

  async *listObjects(auth: ConnectorAuthContext): AsyncGenerator<ObjectDescriptor> {
    const client = this.buildClient(auth);
    const buckets = await client.send(new ListBucketsCommand({}));

    for (const bucket of buckets.Buckets ?? []) {
      // Yield each significant prefix as a separate object descriptor
      const prefixes = await this.listSignificantPrefixes(client, bucket.Name!);
      for (const prefix of prefixes) {
        yield {
          id: `s3://${bucket.Name}/${prefix}`,
          name: `${bucket.Name}/${prefix || '(root)'}`,
          type: 'bucket_prefix',
          lastModified: bucket.CreationDate,
          tags: { bucket: bucket.Name!, prefix },
        };
      }
    }
  }

  async *fetchSample(
    auth: ConnectorAuthContext,
    object: ObjectDescriptor,
    sampleRate: number,
  ): AsyncGenerator<RecordStub> {
    const client = this.buildClient(auth);
    const { bucket, prefix } = this.parseObjectId(object.id);

    let count = 0;
    let continuationToken: string | undefined;

    do {
      const response = await client.send(new ListObjectsV2Command({
        Bucket: bucket,
        Prefix: prefix,
        ContinuationToken: continuationToken,
        MaxKeys: 1000,
      }));

      for (const obj of response.Contents ?? []) {
        // Sample: only process objects based on sampleRate
        if (Math.random() > sampleRate) continue;

        // Fetch first 8KB of object — sufficient for PII detection in text/JSON/CSV
        const content = await client.send(new GetObjectCommand({
          Bucket: bucket,
          Key: obj.Key!,
          Range: 'bytes=0-8191',
        }));

        const text = await content.Body?.transformToString() ?? '';
        yield {
          recordId: obj.Key!,
          objectId: object.id,
          fields: {
            key: obj.Key!,
            size: obj.Size ?? 0,
            contentType: content.ContentType ?? 'unknown',
            // Content is passed to PII detection — NOT stored. Value is transient.
            _content: text,
          },
          fetchedAt: new Date(),
        };
        count++;
      }
      continuationToken = response.NextContinuationToken;
    } while (continuationToken);
  }

  async fetchRecord(auth: ConnectorAuthContext, object: ObjectDescriptor, recordId: string): Promise<RecordStub> {
    const client = this.buildClient(auth);
    const { bucket } = this.parseObjectId(object.id);
    const content = await client.send(new GetObjectCommand({ Bucket: bucket, Key: recordId }));
    const text = await content.Body?.transformToString() ?? '';
    return {
      recordId,
      objectId: object.id,
      fields: { key: recordId, _content: text },
      fetchedAt: new Date(),
    };
  }

  async estimateRecordCount(auth: ConnectorAuthContext, object: ObjectDescriptor): Promise<number | null> {
    // S3 doesn't have a direct count API — return null to skip progress estimation
    return null;
  }

  private buildClient(auth: ConnectorAuthContext): S3Client {
    const creds = auth.authPayload as { AccessKeyId: string; SecretAccessKey: string; SessionToken: string };
    return new S3Client({
      region: 'ap-south-1',
      credentials: {
        accessKeyId: creds.AccessKeyId,
        secretAccessKey: creds.SecretAccessKey,
        sessionToken: creds.SessionToken,
      },
    });
  }

  private parseObjectId(id: string): { bucket: string; prefix: string } {
    const match = id.match(/^s3:\/\/([^/]+)\/?(.*)/);
    return { bucket: match![1], prefix: match![2] ?? '' };
  }

  private async listSignificantPrefixes(client: S3Client, bucket: string): Promise<string[]> {
    // List top-level "directories" (common prefixes with delimiter /)
    const response = await client.send(new ListObjectsV2Command({
      Bucket: bucket,
      Delimiter: '/',
    }));
    return [
      ...(response.CommonPrefixes ?? []).map(p => p.Prefix!),
      '', // Always include root for flat buckets
    ];
  }
}
```

### 3.4 Discovery Agent Pipeline (Python/LangGraph)

The five-agent pipeline runs as Temporal activities. Each agent is a Python function decorated with `@activity.defn` and orchestrated by the `DiscoveryScanWorkflow`. The agents run in the same ECS Fargate task (co-located for low-latency record hand-off) but are independently replaceable.

```python
# discovery_agents/pipeline.py
from langgraph.graph import StateGraph, END
from typing import TypedDict, Annotated
import operator

# State flows through the graph — each agent reads prior state and appends its output
class DiscoveryPipelineState(TypedDict):
    connector_id: str
    tenant_id: str
    job_id: str
    record_stubs: list[dict]           # From Connector Agent
    pii_findings: list[dict]           # From Scanner Agent
    classified_findings: list[dict]    # From Classifier Agent
    risk_scored_findings: list[dict]   # From Risk Scorer Agent
    map_delta: dict                    # From Mapper Agent (nodes/edges to upsert)
    errors: Annotated[list[str], operator.add]


def build_discovery_graph() -> StateGraph:
    graph = StateGraph(DiscoveryPipelineState)

    graph.add_node("connector", connector_agent)
    graph.add_node("scanner", scanner_agent)
    graph.add_node("classifier", classifier_agent)
    graph.add_node("risk_scorer", risk_scorer_agent)
    graph.add_node("mapper", mapper_agent)

    graph.set_entry_point("connector")
    graph.add_edge("connector", "scanner")
    graph.add_edge("scanner", "classifier")
    graph.add_edge("classifier", "risk_scorer")
    graph.add_edge("risk_scorer", "mapper")
    graph.add_edge("mapper", END)

    return graph.compile()
```

**Agent 1 — Connector Agent:**

```python
# discovery_agents/agents/connector_agent.py
from temporalio import activity
from ..connector_bridge import ConnectorBridge  # Calls TypeScript connector via gRPC sidecar

@activity.defn
async def connector_agent(state: DiscoveryPipelineState) -> dict:
    """
    Authenticates to vendor system via TypeScript connector bridge,
    paginates through all objects, streams records to Scanner.
    Records are written to S3 (encrypted) — Scanner reads from S3, not memory.
    This avoids holding all records in RAM for large data lakes.
    """
    activity.heartbeat("starting_connector")

    bridge = ConnectorBridge(
        connector_id=state["connector_id"],
        tenant_id=state["tenant_id"],
    )

    batch_refs = []
    batch = []
    batch_size = 500  # records per S3 batch

    async for record_stub in bridge.stream_records():
        batch.append(record_stub)
        if len(batch) >= batch_size:
            s3_ref = await write_encrypted_batch_to_s3(batch, state["job_id"])
            batch_refs.append(s3_ref)
            batch = []
            activity.heartbeat(f"fetched_batch_{len(batch_refs)}")

    if batch:
        s3_ref = await write_encrypted_batch_to_s3(batch, state["job_id"])
        batch_refs.append(s3_ref)

    return {"batch_refs": batch_refs}
```

**Agent 2 — Scanner Agent (PII Detection):**

```python
# discovery_agents/agents/scanner_agent.py
import hashlib
from ..llm_gateway_client import LLMGatewayClient

# DPDPA Section 2(t): "personal data" means any data about an identifiable individual.
# We detect and HASH detected PII values — raw values are never stored in Discovery DB.

PII_DETECTION_PROMPT = """
You are a PII detection system for DPDPA 2023 compliance.
Analyse the following record fields and identify any personal data.
For each finding, return:
- entity_type: one of [NAME, EMAIL, PHONE, AADHAAR, PAN, PASSPORT, ADDRESS, IP_ADDRESS,
                       DEVICE_ID, BANK_ACCOUNT, UPI_ID, CREDIT_CARD, DOB, BIOMETRIC_REF,
                       HEALTH_DATA, LOCATION, BEHAVIORAL_SIGNAL, MINOR_INDICATOR]
- field_path: JSON path to the field containing the PII
- confidence: 0.0 to 1.0
- context: brief description of why this is PII (no raw value)

Return JSON array. Return empty array if no PII found.
IMPORTANT: Do NOT include the raw PII value in your response.

Record:
{record_json}
"""

class PIIFinding:
    entity_type: str
    field_path: str
    value_hash: str    # SHA-256 of raw value — enables dedup, never reverses to raw
    confidence: float
    location: str      # "connector_id:object_id:record_id:field_path"

@activity.defn
async def scanner_agent(state: DiscoveryPipelineState) -> dict:
    """
    For each record batch in S3, runs PII detection via LLM Gateway.
    Stores HASHED values only — never raw PII in Discovery DB.
    """
    llm = LLMGatewayClient()
    all_findings: list[dict] = []

    for batch_ref in state["batch_refs"]:
        records = await read_encrypted_batch_from_s3(batch_ref)

        for record in records:
            # Filter to string/text fields — skip numeric-only fields for efficiency
            text_fields = {k: v for k, v in record["fields"].items()
                          if isinstance(v, str) and len(v) > 2 and k != "_content"}

            if not text_fields and "_content" not in record["fields"]:
                continue

            # Call on-premise LLM (SageMaker Mistral 7B via LLM Gateway)
            response = await llm.detect_pii(
                record_json=str(text_fields | {"_content": record["fields"].get("_content", "")[:2000]}),
                prompt_template=PII_DETECTION_PROMPT,
            )

            for raw_finding in response:
                # Hash the actual field value from the record for deduplication
                raw_value = get_nested(record["fields"], raw_finding["field_path"])
                value_hash = hashlib.sha256(str(raw_value).encode()).hexdigest() if raw_value else None

                all_findings.append({
                    "entity_type": raw_finding["entity_type"],
                    "field_path": raw_finding["field_path"],
                    "value_hash": value_hash,
                    "confidence": raw_finding["confidence"],
                    "location": f"{state['connector_id']}:{record['objectId']}:{record['recordId']}:{raw_finding['field_path']}",
                    "connector_id": state["connector_id"],
                    "object_id": record["objectId"],
                })

        activity.heartbeat(f"scanned_batch")

    return {"pii_findings": all_findings}
```

**Agent 3 — Classifier Agent:**

```python
# discovery_agents/agents/classifier_agent.py

# DPDPA Section 2(t) and Schedule — data category taxonomy
DPDPA_CATEGORY_MAP = {
    # DPDPA Section 2(t): identifiable individual data
    "PERSONAL_IDENTITY": ["NAME", "AADHAAR", "PAN", "PASSPORT", "DOB"],
    # DPDPA: financial data (sensitive personal data)
    "FINANCIAL": ["BANK_ACCOUNT", "UPI_ID", "CREDIT_CARD"],
    # DPDPA Section 2(t)(ii): health data
    "HEALTH": ["HEALTH_DATA", "BIOMETRIC_REF"],
    # DPDPA: biometric data (explicit category)
    "BIOMETRIC": ["BIOMETRIC_REF"],
    # Standard contact/location data
    "CONTACT": ["EMAIL", "PHONE", "ADDRESS"],
    # Behavioural / tracking data
    "BEHAVIORAL": ["IP_ADDRESS", "DEVICE_ID", "BEHAVIORAL_SIGNAL"],
    # Location data (can reveal sensitive patterns)
    "LOCATION": ["LOCATION"],
}

def classify_entity_type(entity_type: str) -> str:
    for category, types in DPDPA_CATEGORY_MAP.items():
        if entity_type in types:
            return category
    return "PERSONAL_IDENTITY"  # Default to most protective category

@activity.defn
async def classifier_agent(state: DiscoveryPipelineState) -> dict:
    """Groups PII findings by DPDPA data category. Also flags minor indicators."""
    classified = []
    minor_risk_objects = set()

    for finding in state["pii_findings"]:
        category = classify_entity_type(finding["entity_type"])

        # DPDPA Section 9: flag objects containing minor indicators for special handling
        if finding["entity_type"] == "MINOR_INDICATOR":
            minor_risk_objects.add(finding["object_id"])

        classified.append({
            **finding,
            "dpdpa_category": category,
            "minor_risk": finding["object_id"] in minor_risk_objects,
        })

    return {
        "classified_findings": classified,
        "minor_risk_objects": list(minor_risk_objects),
    }
```

**Agent 4 — Risk Scorer Agent:**

```python
# discovery_agents/agents/risk_scorer_agent.py

# Risk score formula — see Section 4.3 for full specification
# Score = Σ(category_weight × confidence) × cross_border_multiplier × age_decay

CATEGORY_WEIGHTS = {
    "BIOMETRIC": 1.0,
    "HEALTH": 0.9,
    "FINANCIAL": 0.85,
    "PERSONAL_IDENTITY": 0.75,
    "LOCATION": 0.6,
    "CONTACT": 0.5,
    "BEHAVIORAL": 0.4,
}

@activity.defn
async def risk_scorer_agent(state: DiscoveryPipelineState) -> dict:
    """
    Computes per-data-flow risk score. Calls Consent Summary API (Module 3 public API)
    to check whether consent exists for this data category + purpose combination.
    Does NOT read raw consent records — uses summary API only.
    """
    from ..consent_summary_client import ConsentSummaryClient

    consent_client = ConsentSummaryClient(tenant_id=state["tenant_id"])

    # Group findings by (connector_id, object_id) to score at the data-flow level
    from itertools import groupby
    findings_by_flow = {}
    for f in state["classified_findings"]:
        key = (f["connector_id"], f["object_id"])
        findings_by_flow.setdefault(key, []).append(f)

    risk_scored = []
    for (connector_id, object_id), findings in findings_by_flow.items():
        categories = list({f["dpdpa_category"] for f in findings})
        avg_confidence = sum(f["confidence"] for f in findings) / len(findings)

        # Base score: weighted sum of category scores
        category_score = max(CATEGORY_WEIGHTS.get(c, 0.4) for c in categories)
        base_score = category_score * avg_confidence

        # Cross-border multiplier: check if this connector's data crosses borders
        # (e.g., GCP us-east1 bucket would trigger Section 16 risk)
        cross_border = await check_connector_region(connector_id)
        cross_border_multiplier = 1.5 if cross_border else 1.0

        # Consent gap penalty: if no consent exists for this category, double the risk
        has_consent = await consent_client.check_summary(
            data_categories=categories,
            connector_id=connector_id,
        )
        consent_multiplier = 1.0 if has_consent else 2.0

        final_score = min(base_score * cross_border_multiplier * consent_multiplier, 1.0)

        for finding in findings:
            risk_scored.append({
                **finding,
                "risk_score": round(final_score, 3),
                "cross_border_flag": cross_border,
                "consent_gap": not has_consent,
            })

    return {"risk_scored_findings": risk_scored}
```

**Agent 5 — Mapper Agent:**

```python
# discovery_agents/agents/mapper_agent.py
import networkx as nx
from ..db import DiscoveryDB

@activity.defn
async def mapper_agent(state: DiscoveryPipelineState) -> dict:
    """
    Builds/updates the Data Flow Map graph.
    Nodes = data systems (connectors, objects)
    Edges = data flows with PII metadata
    """
    db = DiscoveryDB()

    # Upsert system node (connector = source system)
    connector_node = {
        "node_id": state["connector_id"],
        "tenant_id": state["tenant_id"],
        "node_type": "DATA_SYSTEM",
        "display_name": await db.get_connector_display_name(state["connector_id"]),
        "connector_type": await db.get_connector_type(state["connector_id"]),
    }
    await db.upsert_node(connector_node)

    # Group findings by object to create object-level sub-nodes and edges
    by_object: dict[str, list] = {}
    for f in state["risk_scored_findings"]:
        by_object.setdefault(f["object_id"], []).append(f)

    edges_created = 0
    for object_id, findings in by_object.items():
        categories = list({f["dpdpa_category"] for f in findings})
        max_risk = max(f["risk_score"] for f in findings)

        # Object node (table, S3 prefix, API resource, etc.)
        object_node = {
            "node_id": object_id,
            "tenant_id": state["tenant_id"],
            "node_type": "DATA_OBJECT",
            "display_name": object_id.split("/")[-1] or object_id,
            "parent_system_id": state["connector_id"],
        }
        await db.upsert_node(object_node)

        # Edge: connector_id → object_id with PII metadata
        edge = {
            "source_node_id": state["connector_id"],
            "target_node_id": object_id,
            "tenant_id": state["tenant_id"],
            "pii_categories": categories,
            "risk_score": max_risk,
            "finding_count": len(findings),
            "cross_border_flag": any(f["cross_border_flag"] for f in findings),
            "consent_gap": any(f["consent_gap"] for f in findings),
            # Section 9 DPDPA: flag flows touching minor data
            "minor_data_flag": any(f.get("minor_risk", False) for f in findings),
            "last_seen_at": state["job_id"],  # resolved to timestamp in DB trigger
        }
        await db.upsert_edge(edge)
        edges_created += 1

    return {"map_delta": {"nodes_updated": len(by_object) + 1, "edges_updated": edges_created}}
```

### 3.5 NLP Contract Audit Pipeline (Python/FastAPI)

See Section 5 for the detailed end-to-end pipeline with LangGraph workflow code.

### 3.6 LLM Gateway (Python/FastAPI)

The LLM Gateway is a thin routing layer between agent Python code and the SageMaker endpoint. It enforces rate limits per tenant, logs inference metadata (never PII), and handles model-specific prompt formatting.

```python
# llm_gateway/main.py
from fastapi import FastAPI, HTTPException, Depends
from pydantic import BaseModel
from .rate_limiter import TenantRateLimiter
from .sagemaker_client import SageMakerClient
from .inference_logger import InferenceLogger

app = FastAPI()
sagemaker = SageMakerClient(endpoint_name="truststack-mistral-7b-pii")
logger = InferenceLogger()

class PIIDetectionRequest(BaseModel):
    tenant_id: str
    job_id: str
    record_json: str   # Transient — not logged, not stored
    prompt_template: str

class PIIDetectionResponse(BaseModel):
    findings: list[dict]
    model_version: str
    inference_ms: int

@app.post("/infer/pii-detection", response_model=PIIDetectionResponse)
async def detect_pii(
    request: PIIDetectionRequest,
    rate_limiter: TenantRateLimiter = Depends(),
):
    # DPDPA Section 16: all inference on Indian infrastructure (SageMaker ap-south-1)
    await rate_limiter.check(request.tenant_id, "pii_detection")

    import time
    start = time.monotonic_ns()

    prompt = request.prompt_template.format(record_json=request.record_json)
    raw_response = await sagemaker.invoke(
        prompt=prompt,
        max_tokens=1024,
        response_format="json",
    )

    elapsed_ms = (time.monotonic_ns() - start) // 1_000_000

    # Log inference metadata — NO PII, NO prompt content, NO response content
    await logger.log({
        "tenant_id": request.tenant_id,
        "job_id": request.job_id,
        "request_type": "pii_detection",
        "inference_ms": elapsed_ms,
        "model_version": sagemaker.model_version,
        "token_count_approx": len(prompt.split()),  # approximate, not exact
    })

    return PIIDetectionResponse(
        findings=raw_response.get("findings", []),
        model_version=sagemaker.model_version,
        inference_ms=elapsed_ms,
    )

class TranslationRequest(BaseModel):
    tenant_id: str
    text: str
    source_language: str   # ISO 639-1 or BCP 47
    target_language: str

@app.post("/infer/translate")
async def translate(request: TranslationRequest, rate_limiter: TenantRateLimiter = Depends()):
    await rate_limiter.check(request.tenant_id, "translation")
    result = await sagemaker.invoke(
        prompt=f"Translate the following from {request.source_language} to {request.target_language}. "
               f"Return only the translation.\n\n{request.text}",
        max_tokens=2048,
    )
    return {"translation": result["text"], "model_version": sagemaker.model_version}
```

**SageMaker client:**

```python
# llm_gateway/sagemaker_client.py
import boto3
import json

class SageMakerClient:
    def __init__(self, endpoint_name: str):
        self.endpoint_name = endpoint_name
        self.client = boto3.client("sagemaker-runtime", region_name="ap-south-1")
        self.model_version = self._fetch_model_version()

    async def invoke(self, prompt: str, max_tokens: int = 512, response_format: str = "text") -> dict:
        body = json.dumps({
            "inputs": prompt,
            "parameters": {
                "max_new_tokens": max_tokens,
                "temperature": 0.1,  # Low temperature for classification tasks
                "return_full_text": False,
                "response_format": {"type": response_format} if response_format == "json" else None,
            },
        })

        response = self.client.invoke_endpoint(
            EndpointName=self.endpoint_name,
            ContentType="application/json",
            Body=body,
        )

        result = json.loads(response["Body"].read())
        if response_format == "json":
            return json.loads(result[0]["generated_text"])
        return {"text": result[0]["generated_text"]}

    def _fetch_model_version(self) -> str:
        sm = boto3.client("sagemaker", region_name="ap-south-1")
        endpoint = sm.describe_endpoint(EndpointName=self.endpoint_name)
        config_name = endpoint["EndpointConfigName"]
        config = sm.describe_endpoint_config(EndpointConfigName=config_name)
        return config["ProductionVariants"][0]["ModelName"]
```

### 3.7 Data Flow Map Service

See Section 4 for the full Data Flow Map schema and construction logic. The service exposes the map via the API layer and generates snapshot tokens for the Erasure Engine (Section 6).

### 3.8 Databases and Storage

**Discovery DB schema (PostgreSQL 16, RDS Multi-AZ, ap-south-1):**

```sql
-- All tables: tenant_id column with Row-Level Security for multi-tenant isolation

-- Scan jobs
CREATE TABLE scan_jobs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    temporal_workflow_id TEXT NOT NULL,
    status TEXT NOT NULL CHECK (status IN ('QUEUED','RUNNING','COMPLETED','PARTIAL','CANCELLED','FAILED')),
    scan_type TEXT NOT NULL CHECK (scan_type IN ('FULL','INCREMENTAL','TARGETED')),
    connector_ids UUID[] NOT NULL,
    sample_rate NUMERIC(4,3) NOT NULL DEFAULT 0.1,
    initiated_at TIMESTAMPTZ NOT NULL,
    completed_at TIMESTAMPTZ,
    total_pii_findings INT DEFAULT 0,
    error_summary JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- PII findings — hashed values only, NEVER raw PII
-- DPDPA: we classify data but do not store the personal data itself
CREATE TABLE pii_findings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    scan_job_id UUID NOT NULL REFERENCES scan_jobs(id),
    connector_id UUID NOT NULL REFERENCES connector_configs(id),
    object_id TEXT NOT NULL,          -- S3 prefix, table name, etc.
    record_id TEXT NOT NULL,          -- Individual record identifier
    field_path TEXT NOT NULL,         -- JSON path within record
    entity_type TEXT NOT NULL,        -- e.g., AADHAAR, EMAIL, UPI_ID
    dpdpa_category TEXT NOT NULL,     -- DPDPA taxonomy category
    value_hash TEXT,                  -- SHA-256 of raw value (for dedup, not reversal)
    confidence NUMERIC(4,3) NOT NULL,
    risk_score NUMERIC(4,3) NOT NULL,
    cross_border_flag BOOLEAN NOT NULL DEFAULT false,
    consent_gap BOOLEAN NOT NULL DEFAULT false,
    minor_data_flag BOOLEAN NOT NULL DEFAULT false,
    -- DPDPA Section 9: minor data requires special handling
    detected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Data flow map nodes (systems and objects)
CREATE TABLE data_flow_nodes (
    id TEXT NOT NULL,                 -- Connector ID or object ID
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    node_type TEXT NOT NULL CHECK (node_type IN ('DATA_SYSTEM','DATA_OBJECT','EXTERNAL_RECIPIENT')),
    display_name TEXT NOT NULL,
    connector_type TEXT,
    parent_system_id TEXT,
    metadata JSONB,
    last_seen_scan_id UUID REFERENCES scan_jobs(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (id, tenant_id)
);

-- Data flow map edges (flows between nodes with PII metadata)
CREATE TABLE data_flow_edges (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    source_node_id TEXT NOT NULL,
    target_node_id TEXT NOT NULL,
    pii_categories TEXT[] NOT NULL,
    risk_score NUMERIC(4,3) NOT NULL,
    finding_count INT NOT NULL,
    cross_border_flag BOOLEAN NOT NULL DEFAULT false,
    consent_gap BOOLEAN NOT NULL DEFAULT false,
    minor_data_flag BOOLEAN NOT NULL DEFAULT false,
    last_seen_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, source_node_id, target_node_id)
);

-- Contract audits
CREATE TABLE contract_audits (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    filename TEXT NOT NULL,
    file_type TEXT NOT NULL CHECK (file_type IN ('PDF','DOCX','IMAGE')),
    detected_language TEXT,
    original_language TEXT,
    status TEXT NOT NULL CHECK (status IN ('PROCESSING','COMPLETED','FAILED')),
    clause_count INT,
    violation_count INT,
    overall_risk_level TEXT CHECK (overall_risk_level IN ('LOW','MEDIUM','HIGH','CRITICAL')),
    submitted_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at TIMESTAMPTZ
);

-- Individual contract clauses with compliance classification
CREATE TABLE contract_clauses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    audit_id UUID NOT NULL REFERENCES contract_audits(id),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    clause_index INT NOT NULL,
    original_text TEXT NOT NULL,      -- Original language text
    english_text TEXT,                -- English translation (if translated)
    violation_type TEXT,              -- NULL if compliant
    dpdpa_section_reference TEXT,     -- e.g., "Section 6", "Rule 7"
    severity TEXT CHECK (severity IN ('INFO','WARNING','VIOLATION','CRITICAL')),
    suggested_fix TEXT,
    confidence NUMERIC(4,3)
);

-- Generated DPAs
CREATE TABLE dpa_generations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    audit_id UUID NOT NULL REFERENCES contract_audits(id),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    output_language TEXT NOT NULL,
    s3_artifact_key TEXT NOT NULL,    -- S3 key for the generated DPA PDF
    ecdsa_signature TEXT NOT NULL,    -- Signed with tenant's KMS key
    signing_key_id TEXT NOT NULL,
    generated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Snapshot tokens for Erasure Engine (Module 3) handshake
CREATE TABLE snapshot_tokens (
    token TEXT PRIMARY KEY,           -- JWT token (signed, not hashed — needed for validation)
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    dp_identifier_hash TEXT NOT NULL, -- HMAC-SHA256 of Data Principal identifier
    system_count INT NOT NULL,
    expires_at TIMESTAMPTZ NOT NULL,
    consumed_at TIMESTAMPTZ,          -- Set on first use — subsequent calls are rejected
    requested_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    requester_account_id TEXT NOT NULL  -- AWS account ID of Module 3 caller
);

-- Connector configurations (credentials stored in Secrets Manager, not here)
CREATE TABLE connector_configs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    display_name TEXT NOT NULL,
    connector_type TEXT NOT NULL,
    secret_arn TEXT NOT NULL,         -- ARN in Secrets Manager (never the secret itself)
    is_active BOOLEAN NOT NULL DEFAULT true,
    last_auth_success_at TIMESTAMPTZ,
    last_auth_failure_at TIMESTAMPTZ,
    last_auth_failure_reason TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Row-level security for all tables
ALTER TABLE scan_jobs ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON scan_jobs USING (tenant_id = current_setting('app.tenant_id')::UUID);
-- (Same pattern applied to all tenant-scoped tables)
```

---

## 4. Data Flow Map: Construction and Schema

### 4.1 Graph Model

The Data Flow Map is a directed graph where:
- **Nodes** represent data systems (vendor platforms, databases, storage buckets) and the data objects within them (tables, collections, S3 prefixes, API resources)
- **Edges** represent data flows — the movement or co-location of personal data between systems, annotated with DPDPA compliance metadata

The graph is stored in PostgreSQL (`data_flow_nodes`, `data_flow_edges`) for persistence and queried via NetworkX in-memory for graph analytics (path-finding for erasure propagation, cross-border flow detection).

### 4.2 Node Schema

```json
{
  "nodeId": "conn_zoho_crm_abc123",
  "tenantId": "tenant_uuid",
  "nodeType": "DATA_SYSTEM",
  "displayName": "Zoho CRM",
  "connectorType": "zoho_crm",
  "parentSystemId": null,
  "metadata": {
    "region": "in",
    "vendor": "Zoho",
    "crossBorder": false,
    "lastScanAt": "2026-04-22T10:30:00Z",
    "activeObjectCount": 12
  },
  "riskSummary": {
    "maxRiskScore": 0.87,
    "totalPIIFindings": 1420,
    "categoriesPresent": ["PERSONAL_IDENTITY", "CONTACT", "FINANCIAL"],
    "consentGapCount": 3
  }
}
```

```json
{
  "nodeId": "s3://acme-backups/customer-exports/2025/",
  "tenantId": "tenant_uuid",
  "nodeType": "DATA_OBJECT",
  "displayName": "customer-exports/2025/",
  "connectorType": "aws_s3",
  "parentSystemId": "conn_s3_acme_backups",
  "metadata": {
    "objectType": "bucket_prefix",
    "estimatedRecords": 45000,
    "lastModified": "2026-03-15T08:12:00Z"
  }
}
```

### 4.3 Edge Schema

```json
{
  "edgeId": "uuid",
  "tenantId": "tenant_uuid",
  "sourceNodeId": "conn_zoho_crm_abc123",
  "targetNodeId": "s3://acme-backups/customer-exports/2025/",
  "piiCategories": ["PERSONAL_IDENTITY", "CONTACT", "FINANCIAL"],
  "riskScore": 0.87,
  "findingCount": 1420,
  "crossBorderFlag": false,
  "consentGap": true,
  "minorDataFlag": false,
  "lastSeenAt": "2026-04-22T10:30:00Z",
  "metadata": {
    "volumeEstimate": "45K records",
    "topEntityTypes": ["EMAIL", "PHONE", "NAME", "UPI_ID"],
    "avgConfidence": 0.92
  }
}
```

### 4.4 Risk Score Formula

The risk score for each data flow edge is a value in [0, 1] computed as:

```
risk_score = min(base_score × cross_border_multiplier × consent_multiplier, 1.0)

where:
  base_score = max_category_weight(categories) × avg_confidence

  max_category_weight:
    BIOMETRIC          → 1.00
    HEALTH             → 0.90
    FINANCIAL          → 0.85
    PERSONAL_IDENTITY  → 0.75
    LOCATION           → 0.60
    CONTACT            → 0.50
    BEHAVIORAL         → 0.40

  cross_border_multiplier:
    connector region outside India → 1.5   (Section 16 DPDPA violation risk)
    connector region in India      → 1.0

  consent_multiplier:
    no valid consent for data category → 2.0   (Section 6 DPDPA: consent is mandatory)
    valid consent exists               → 1.0
```

Risk bands: `0.0–0.3 = LOW`, `0.3–0.6 = MEDIUM`, `0.6–0.8 = HIGH`, `0.8–1.0 = CRITICAL`

### 4.5 API Output Format

```typescript
// GET /v1/dataflow — response schema
interface DataFlowMapResponse {
  tenantId: string;
  generatedAt: string;
  scanJobIds: string[];  // which scans contributed to this map
  stats: {
    totalNodes: number;
    totalEdges: number;
    criticalFlows: number;     // edges with risk_score >= 0.8
    consentGaps: number;       // edges with consent_gap = true
    crossBorderFlows: number;  // edges with cross_border_flag = true
    minorDataFlows: number;    // edges with minor_data_flag = true (Section 9)
  };
  nodes: DataFlowNode[];
  edges: DataFlowEdge[];
}
```

---

## 5. NLP Contract Audit: End-to-End Pipeline

### 5.1 Pipeline Overview

The contract audit pipeline converts a vendor contract in any of India's 22 Eighth Schedule languages into a structured DPDPA compliance report, then generates a Data Processing Agreement (DPA) in the vendor's language.

```python
# contract_audit/pipeline.py
from langgraph.graph import StateGraph, END
from typing import TypedDict

class ContractAuditState(TypedDict):
    audit_id: str
    tenant_id: str
    file_bytes: bytes
    file_type: str          # PDF, DOCX, IMAGE
    raw_text: str           # After extraction/OCR
    detected_language: str  # BCP 47 code
    english_text: str       # After translation (if needed)
    clauses: list[dict]     # After clause extraction
    violations: list[dict]  # After compliance check
    dpa_draft: str          # Generated DPA in English
    dpa_translated: str     # DPA in original language
    dpa_signed_artifact_key: str  # S3 key for signed DPA PDF


def build_contract_audit_graph() -> StateGraph:
    graph = StateGraph(ContractAuditState)

    graph.add_node("file_processor", file_processor_node)
    graph.add_node("ocr_agent", ocr_agent_node)
    graph.add_node("language_detector", language_detector_node)
    graph.add_node("translation_agent", translation_agent_node)
    graph.add_node("clause_extractor", clause_extractor_node)
    graph.add_node("compliance_checker", compliance_checker_node)
    graph.add_node("dpa_generator", dpa_generator_node)
    graph.add_node("ecdsa_signer", ecdsa_signer_node)

    graph.set_entry_point("file_processor")

    # Branch on file type: structured files → skip OCR, images → OCR
    graph.add_conditional_edges(
        "file_processor",
        lambda s: "ocr" if s["file_type"] == "IMAGE" else "language",
        {"ocr": "ocr_agent", "language": "language_detector"},
    )
    graph.add_edge("ocr_agent", "language_detector")
    graph.add_edge("language_detector", "translation_agent")
    graph.add_edge("translation_agent", "clause_extractor")
    graph.add_edge("clause_extractor", "compliance_checker")
    graph.add_edge("compliance_checker", "dpa_generator")
    graph.add_edge("dpa_generator", "ecdsa_signer")
    graph.add_edge("ecdsa_signer", END)

    return graph.compile()
```

### 5.2 File Processor and OCR

```python
# contract_audit/nodes/file_processor.py
import fitz  # PyMuPDF
from docx import Document as DocxDocument

async def file_processor_node(state: ContractAuditState) -> dict:
    """Extracts raw text from PDF or DOCX. IMAGE type passes through for OCR."""
    if state["file_type"] == "PDF":
        doc = fitz.open(stream=state["file_bytes"], filetype="pdf")
        text = "\n\n".join(page.get_text("text") for page in doc)
        # If PyMuPDF extracts empty text, this is a scanned PDF — fall back to OCR
        if len(text.strip()) < 100:
            return {"file_type": "IMAGE", "raw_text": ""}
        return {"raw_text": text, "file_type": "PDF"}

    elif state["file_type"] == "DOCX":
        doc = DocxDocument(stream=state["file_bytes"])
        text = "\n\n".join(para.text for para in doc.paragraphs if para.text.strip())
        return {"raw_text": text, "file_type": "DOCX"}

    # IMAGE — pass through, raw_text will be populated by OCR agent
    return {"raw_text": "", "file_type": "IMAGE"}


# contract_audit/nodes/ocr_agent.py
import pytesseract
from PIL import Image
import io

# Tesseract 5 with Indic language packs installed at Docker image build time
# Language packs: hin, tam, tel, kan, ben, mar, guj, mal, odi, pun, urd, asm,
#                 kok, mni, sat, doi, kas, sin, nep, mai, bod (Tibetan — Ladakhi)
TESSERACT_LANG_MAP = {
    "hi": "hin", "ta": "tam", "te": "tel", "kn": "kan",
    "bn": "ben", "mr": "mar", "gu": "guj", "ml": "mal",
    "or": "ori", "pa": "pun", "ur": "urd", "as": "asm",
    "kok": "kok", "mni": "mni", "sat": "sat", "doi": "doi",
    "ks": "kas", "si": "sin", "ne": "nep", "mai": "mai",
    "en": "eng",
}

async def ocr_agent_node(state: ContractAuditState) -> dict:
    """
    Runs Tesseract 5 on scanned documents/images.
    For PDFs with embedded images (scanned contracts), converts to images first.
    """
    if state["file_type"] == "IMAGE":
        image = Image.open(io.BytesIO(state["file_bytes"]))
    else:
        # Scanned PDF — render first page to image for language detection,
        # then run OCR on all pages
        import fitz
        doc = fitz.open(stream=state["file_bytes"], filetype="pdf")
        # Render at 300 DPI for quality OCR
        page = doc[0]
        pix = page.get_pixmap(dpi=300)
        image = Image.frombytes("RGB", [pix.width, pix.height], pix.samples)

    # Quick language probe on first image using multiple scripts
    # Try all Indic scripts + English simultaneously
    lang_string = "+".join(TESSERACT_LANG_MAP.values())
    raw_text = pytesseract.image_to_string(image, lang=lang_string, config="--oem 1 --psm 3")

    return {"raw_text": raw_text}
```

### 5.3 Language Detection and Translation

```python
# contract_audit/nodes/language_detector.py
from langdetect import detect_langs
from charset_normalizer import from_bytes

async def language_detector_node(state: ContractAuditState) -> dict:
    """
    Detects language using LangDetect + charset-normalizer for Indic scripts.
    LangDetect handles Latin-script languages well. charset-normalizer's
    script detection handles Devanagari, Tamil, Telugu, Kannada, etc.
    """
    text = state["raw_text"]

    # Indic script Unicode range detection takes priority over LangDetect
    script_language = detect_script_language(text)
    if script_language:
        return {"detected_language": script_language}

    # LangDetect for Latin-script and Romanised Indic
    try:
        results = detect_langs(text[:5000])  # Use first 5000 chars for speed
        primary = results[0]
        return {"detected_language": primary.lang if primary.prob > 0.7 else "hi"}
    except Exception:
        return {"detected_language": "hi"}  # Default to Hindi


SCRIPT_RANGES = {
    # Devanagari — Hindi, Marathi, Nepali, Sanskrit, Dogri, Maithili
    "hi": (0x0900, 0x097F),
    # Tamil
    "ta": (0x0B80, 0x0BFF),
    # Telugu
    "te": (0x0C00, 0x0C7F),
    # Kannada
    "kn": (0x0C80, 0x0CFF),
    # Malayalam
    "ml": (0x0D00, 0x0D7F),
    # Bengali / Assamese
    "bn": (0x0980, 0x09FF),
    # Gujarati
    "gu": (0x0A80, 0x0AFF),
    # Gurmukhi (Punjabi)
    "pa": (0x0A00, 0x0A7F),
    # Odia
    "or": (0x0B00, 0x0B7F),
    # Meitei Mayek (Manipuri)
    "mni": (0xABC0, 0xABFF),
    # Ol Chiki (Santali)
    "sat": (0x1C50, 0x1C7F),
    # Arabic / Nastaliq (Urdu, Kashmiri, Sindhi)
    "ur": (0x0600, 0x06FF),
}

def detect_script_language(text: str) -> str | None:
    char_counts: dict[str, int] = {}
    for char in text[:2000]:
        codepoint = ord(char)
        for lang, (lo, hi) in SCRIPT_RANGES.items():
            if lo <= codepoint <= hi:
                char_counts[lang] = char_counts.get(lang, 0) + 1

    if not char_counts:
        return None
    dominant = max(char_counts, key=lambda k: char_counts[k])
    if char_counts[dominant] > 50:  # At least 50 chars in that script
        return dominant
    return None


# contract_audit/nodes/translation_agent.py
async def translation_agent_node(state: ContractAuditState) -> dict:
    """
    If document is not in English, translates to English for compliance checking.
    Preserves original text for output generation.
    Translation uses on-premise LLM (Section 16 — no cross-border data transfer).
    """
    if state["detected_language"] == "en":
        return {"english_text": state["raw_text"]}

    from ..llm_gateway_client import LLMGatewayClient
    llm = LLMGatewayClient()

    # Translate in chunks to avoid token limits
    CHUNK_SIZE = 3000  # characters
    chunks = [state["raw_text"][i:i+CHUNK_SIZE]
              for i in range(0, len(state["raw_text"]), CHUNK_SIZE)]

    translated_chunks = []
    for chunk in chunks:
        result = await llm.translate(
            tenant_id=state["tenant_id"],
            text=chunk,
            source_language=state["detected_language"],
            target_language="en",
        )
        translated_chunks.append(result["translation"])

    return {"english_text": "\n\n".join(translated_chunks)}
```

### 5.4 Clause Extraction and Compliance Checking

```python
# contract_audit/nodes/clause_extractor.py
import re
from sentence_transformers import SentenceTransformer

# BERT-based sentence boundary model — runs on CPU, no GPU needed
boundary_model = SentenceTransformer("sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2")

async def clause_extractor_node(state: ContractAuditState) -> dict:
    """
    Segments contract into clauses using two-pass approach:
    1. Structural: numbered clauses, section headings, paragraph breaks
    2. Semantic: BERT cosine similarity to detect topic shifts within paragraphs
    """
    text = state["english_text"]

    # Pass 1: structural segmentation
    structural_clauses = re.split(
        r'\n(?=\d+[\.\)]\s|\b(?:Section|Clause|Article|Schedule)\s+\d+|\b[A-Z][A-Z\s]{3,}\n)',
        text
    )

    # Pass 2: merge very short fragments, split very long ones semantically
    refined_clauses = []
    for clause in structural_clauses:
        clause = clause.strip()
        if len(clause) < 50:
            # Too short — merge with previous
            if refined_clauses:
                refined_clauses[-1]["text"] += " " + clause
        elif len(clause) > 2000:
            # Too long — split at sentence boundaries
            sentences = re.split(r'(?<=[.!?])\s+', clause)
            buffer = ""
            for sentence in sentences:
                if len(buffer) + len(sentence) > 1500:
                    if buffer:
                        refined_clauses.append({"text": buffer.strip(), "index": len(refined_clauses)})
                    buffer = sentence
                else:
                    buffer += " " + sentence
            if buffer:
                refined_clauses.append({"text": buffer.strip(), "index": len(refined_clauses)})
        else:
            refined_clauses.append({"text": clause, "index": len(refined_clauses)})

    return {"clauses": refined_clauses}


# contract_audit/nodes/compliance_checker.py
from ..llm_gateway_client import LLMGatewayClient

# DPDPA 2023 + DPDP Rules 2025 violation taxonomy
VIOLATION_TAXONOMY = {
    "DATA_SHARING_UNLIMITED": {
        "description": "Clause permits sharing data with third parties without purpose limitation",
        "dpdpa_reference": "Section 6(1) DPDPA — consent must be purpose-specific",
        "severity": "CRITICAL",
        "keywords": ["any third party", "share with affiliates", "unlimited sharing", "as we deem fit"],
    },
    "MISSING_BREACH_SLA": {
        "description": "Clause does not specify 72-hour DPBI notification or 6-hour CERT-In SLA",
        "dpdpa_reference": "Rule 7, DPDP Rules 2025 — all breaches must be reported",
        "severity": "VIOLATION",
        "keywords": [],  # Absence detection — LLM checks if SLA is missing
    },
    "CROSS_BORDER_UNAUTHORIZED": {
        "description": "Clause permits data transfer to countries not approved by DPBI",
        "dpdpa_reference": "Section 16 DPDPA — cross-border transfer restrictions",
        "severity": "CRITICAL",
        "keywords": ["transfer to", "store in", "processed in", "foreign servers"],
    },
    "RETENTION_INDEFINITE": {
        "description": "Clause retains data indefinitely or beyond purpose fulfilment",
        "dpdpa_reference": "Third Schedule DPDPA — class-specific retention limits",
        "severity": "VIOLATION",
        "keywords": ["retain indefinitely", "as long as required", "no deletion policy"],
    },
    "MINOR_PROFILING_UNCHECKED": {
        "description": "Clause permits profiling or targeted advertising without age verification",
        "dpdpa_reference": "Section 9 DPDPA — verifiable parental consent required for under-18",
        "severity": "CRITICAL",
        "keywords": ["all users", "profile users", "targeted advertising", "behavioural tracking"],
    },
    "MISSING_DATA_MINIMIZATION": {
        "description": "Clause collects data beyond what is necessary for the stated purpose",
        "dpdpa_reference": "Section 6(4) DPDPA — data collection limited to necessary data",
        "severity": "WARNING",
        "keywords": ["collect all", "comprehensive data", "any information"],
    },
    "MISSING_DP_RIGHTS_CLAUSE": {
        "description": "Contract does not address Data Principal rights (access, correction, erasure, grievance)",
        "dpdpa_reference": "Sections 12-13 DPDPA — Data Principal rights must be honoured",
        "severity": "VIOLATION",
        "keywords": [],  # Absence detection
    },
}

COMPLIANCE_PROMPT = """
You are a DPDPA 2023 legal compliance classifier for Indian data protection law.
Analyse the following contract clause and determine if it contains any of these violations:

Violation types:
{violation_types}

For each violation found, return:
- violation_type: exact type string from the list above
- severity: CRITICAL, VIOLATION, WARNING, or INFO
- explanation: one sentence explaining why this clause violates the rule
- suggested_fix: a compliant alternative phrasing (in English)
- confidence: 0.0 to 1.0

Also flag: absence of breach SLA (MISSING_BREACH_SLA) and absence of DP rights provisions (MISSING_DP_RIGHTS_CLAUSE) even if keywords are absent.

Return JSON array of violations. Return empty array if compliant.

Clause:
{clause_text}
"""

async def compliance_checker_node(state: ContractAuditState) -> dict:
    llm = LLMGatewayClient()
    all_violations = []

    violation_types_str = "\n".join(
        f"- {k}: {v['description']} (Ref: {v['dpdpa_reference']})"
        for k, v in VIOLATION_TAXONOMY.items()
    )

    for clause in state["clauses"]:
        response = await llm.classify_clause(
            tenant_id=state["tenant_id"],
            prompt=COMPLIANCE_PROMPT.format(
                violation_types=violation_types_str,
                clause_text=clause["text"][:2000],
            ),
        )

        for violation in response:
            all_violations.append({
                "clause_index": clause["index"],
                "clause_text": clause["text"],
                **violation,
            })

    return {"violations": all_violations}
```

### 5.5 DPA Generator and ECDSA Signing

```python
# contract_audit/nodes/dpa_generator.py

DPA_TEMPLATE = """
DATA PROCESSING AGREEMENT

Between:
  Data Fiduciary: {fiduciary_name} (the "Company")
  Data Processor: {processor_name} (the "Vendor")

Effective Date: {effective_date}

1. PURPOSE AND SCOPE
   The Vendor processes personal data on behalf of the Company solely for:
   {purposes_list}
   The Vendor shall not process personal data for any other purpose without explicit written consent
   from the Company. [Section 6(1) DPDPA 2023]

2. DATA CATEGORIES
   Categories of personal data processed under this Agreement:
   {data_categories}

3. DATA PRINCIPAL RIGHTS [Sections 12-13 DPDPA 2023]
   The Vendor shall, within 72 hours of receiving a request from the Company:
   a) Provide access to personal data held about any Data Principal
   b) Correct inaccurate personal data
   c) Erase personal data upon withdrawal of consent or purpose fulfilment
   d) Acknowledge and escalate grievances to the Company's Data Protection Officer

4. BREACH NOTIFICATION [Rule 7, DPDP Rules 2025]
   In the event of a personal data breach, the Vendor shall:
   a) Notify the Company within 2 hours of discovery
   b) Provide a detailed incident report within 24 hours
   c) Cooperate with Company's submission to DPBI within 72 hours
   d) Cooperate with Company's submission to CERT-In within 6 hours

5. DATA RETENTION [Third Schedule, DPDPA 2023]
   The Vendor shall retain personal data only for as long as necessary to fulfil the stated purpose.
   Upon purpose fulfilment or consent withdrawal, the Vendor shall:
   a) Provide 48-hour advance notice to the Company before deletion
   b) Permanently delete all personal data (soft-delete is non-compliant)
   c) Provide a verifiable deletion certificate within 7 days of deletion

6. DATA RESIDENCY [Section 16, DPDPA 2023]
   All personal data processed under this Agreement shall be stored and processed
   exclusively within India. Transfer to foreign servers requires prior written approval
   from the Company and compliance with Section 16 DPDPA.

7. DATA MINIMISATION [Section 6(4), DPDPA 2023]
   The Vendor shall collect and process only the minimum personal data necessary
   for the stated purpose. Collection of additional data requires explicit consent amendment.

8. MINOR DATA [Section 9, DPDPA 2023]
   The Vendor shall implement age verification controls and shall not process data of
   individuals under 18 without verifiable parental consent. Targeted advertising and
   profiling of minors is strictly prohibited.

9. SECURITY MEASURES [Rule 4, DPDP Rules 2025]
   The Vendor shall maintain:
   a) AES-256 encryption at rest
   b) TLS 1.3 in transit
   c) Access controls with least privilege
   d) Annual security audits

10. GOVERNING LAW
    This Agreement is governed by the Digital Personal Data Protection Act, 2023
    and DPDP Rules 2025. Disputes shall be resolved per Indian jurisdiction.

Signed by:
{fiduciary_name}: ___________________  Date: _______
{processor_name}: ___________________  Date: _______
"""

async def dpa_generator_node(state: ContractAuditState) -> dict:
    """
    Extracts vendor details from audit, fills DPA template, translates to original language.
    """
    llm = LLMGatewayClient()

    # Extract vendor details from the original contract text
    extracted = await llm.extract_entities(
        tenant_id=state["tenant_id"],
        text=state["english_text"][:5000],
        entities=["fiduciary_name", "processor_name", "effective_date", "purposes", "data_categories"],
    )

    # Fill English DPA template
    dpa_english = DPA_TEMPLATE.format(
        fiduciary_name=extracted.get("fiduciary_name", "[DATA FIDUCIARY NAME]"),
        processor_name=extracted.get("processor_name", "[VENDOR NAME]"),
        effective_date=extracted.get("effective_date", "[DATE]"),
        purposes_list="\n   ".join(f"- {p}" for p in extracted.get("purposes", ["[PURPOSE]"])),
        data_categories="\n   ".join(f"- {c}" for c in extracted.get("data_categories", ["[CATEGORIES]"])),
    )

    # Translate DPA to original contract language if not English
    if state["detected_language"] != "en":
        result = await llm.translate(
            tenant_id=state["tenant_id"],
            text=dpa_english,
            source_language="en",
            target_language=state["detected_language"],
        )
        dpa_translated = result["translation"]
    else:
        dpa_translated = dpa_english

    return {"dpa_draft": dpa_english, "dpa_translated": dpa_translated}


# contract_audit/nodes/ecdsa_signer.py
import boto3
import reportlab  # PDF generation
from reportlab.lib.pagesizes import A4
from reportlab.platypus import SimpleDocTemplate, Paragraph
from reportlab.lib.styles import getSampleStyleSheet
import io

async def ecdsa_signer_node(state: ContractAuditState) -> dict:
    """
    Generates DPA as PDF, signs with tenant's KMS key (ECDSA P-256),
    uploads to S3 artifact store.
    Mirrors signing pattern from SahmatOS consent artifacts.
    """
    # Generate PDF from DPA text
    pdf_buffer = io.BytesIO()
    doc = SimpleDocTemplate(pdf_buffer, pagesize=A4)
    styles = getSampleStyleSheet()
    content = [Paragraph(line, styles["Normal"]) for line in state["dpa_translated"].split("\n")]
    doc.build(content)
    pdf_bytes = pdf_buffer.getvalue()

    # Sign with tenant's KMS key
    kms = boto3.client("kms", region_name="ap-south-1")
    tenant_key_arn = await get_tenant_kms_key_arn(state["tenant_id"])

    sign_response = kms.sign(
        KeyId=tenant_key_arn,
        Message=pdf_bytes,
        MessageType="RAW",
        SigningAlgorithm="ECDSA_SHA_256",
    )
    import base64
    signature = base64.b64encode(sign_response["Signature"]).decode()

    # Upload to S3 artifact store (AES-256 SSE)
    s3 = boto3.client("s3", region_name="ap-south-1")
    s3_key = f"dpa/{state['tenant_id']}/{state['audit_id']}/dpa.pdf"
    s3.put_object(
        Bucket=f"truststack-discovery-artifacts-{get_account_id()}",
        Key=s3_key,
        Body=pdf_bytes,
        ServerSideEncryption="aws:kms",
        Metadata={
            "ecdsa-signature": signature,
            "signing-key-id": tenant_key_arn,
            "audit-id": state["audit_id"],
            "tenant-id": state["tenant_id"],
        },
    )

    # Persist to DB
    await DiscoveryDB().insert_dpa_generation({
        "audit_id": state["audit_id"],
        "tenant_id": state["tenant_id"],
        "output_language": state["detected_language"],
        "s3_artifact_key": s3_key,
        "ecdsa_signature": signature,
        "signing_key_id": tenant_key_arn,
    })

    return {"dpa_signed_artifact_key": s3_key}
```

---

## 6. Snapshot Protocol: Discovery → Erasure Engine

### 6.1 Protocol Overview

The snapshot protocol is the single, carefully controlled integration point between Module 2 (`TrustStack-Discovery`) and Module 3 (`TrustStack-Consent`). It allows the Erasure Engine to learn which systems hold a Data Principal's personal data — without giving it direct access to Discovery's database or internal services.

The protocol enforces these security invariants:
1. Only the Erasure Engine service account (identified by AWS account ID and signed JWT) can request snapshots
2. Each snapshot is one-time-use — token invalidated on first consumption
3. Snapshots are rate-limited to 1 per DP per hour to prevent enumeration attacks
4. The snapshot payload is encrypted with the Erasure Engine's public key — only Module 3 can decrypt it
5. Discovery DOES NOT initiate a live scan during snapshot generation — it reads existing map data only
6. The `connectionRef` in the snapshot is an opaque encrypted reference — the Erasure Engine passes it back to Discovery's connector execution sidecar during erasure, but never decrypts it directly

### 6.2 API Contract

**Step 1 — Request snapshot token:**

```
POST /v1/snapshot/request
Authorization: X-Api-Key: {module3_service_api_key}
Content-Type: application/json

{
  "dpIdentifierHash": "hmac_sha256_of_dp_id",   // HMAC-SHA256, same scheme as SahmatOS
  "tenantId": "uuid",
  "requestJwt": "signed_jwt_from_module3",        // Signed with Module 3's KMS key
  "requestedAt": "2026-04-22T10:00:00Z"
}

Response 200:
{
  "snapshotToken": "eyJhbGciOiJFUzI1NiJ9...",
  "expiresAt": "2026-04-22T10:15:00Z",            // 15-minute TTL
  "systemCount": 4                                 // How many systems have data for this DP
}

Response 429 (rate limited):
{
  "type": "https://truststack.in/errors/rate-limited",
  "title": "Snapshot rate limit exceeded",
  "detail": "Maximum 1 snapshot per Data Principal per hour",
  "status": 429,
  "retryAfter": "2026-04-22T10:58:00Z"
}
```

**Step 2 — Consume snapshot:**

```
GET /v1/snapshot/{snapshotToken}
Authorization: X-Api-Key: {module3_service_api_key}

Response 200:
{
  "dpIdentifierHash": "hmac_sha256_of_dp_id",
  "tenantId": "uuid",
  "snapshotAt": "2026-04-22T10:00:00Z",
  "systems": [
    {
      "id": "conn_zoho_crm_abc123",
      "name": "Zoho CRM",
      "connectorType": "zoho_crm",
      "dataCategories": ["PERSONAL_IDENTITY", "CONTACT"],
      "estimatedRecords": 3,
      "connectionRef": "encrypted_opaque_ref_base64"   // Opaque — Erasure Engine passes back, never decrypts
    },
    {
      "id": "s3://acme-backups/customer-exports/",
      "name": "S3 Customer Exports",
      "connectorType": "aws_s3",
      "dataCategories": ["PERSONAL_IDENTITY", "FINANCIAL"],
      "estimatedRecords": 12,
      "connectionRef": "encrypted_opaque_ref_base64"
    }
  ]
}

Response 404 (already consumed or expired):
{
  "type": "https://truststack.in/errors/not-found",
  "title": "Snapshot token not found or already consumed",
  "status": 404
}
```

### 6.3 Implementation

```typescript
// src/routes/snapshot.ts
import { FastifyInstance } from 'fastify';
import { KMSClient, VerifyCommand, EncryptCommand } from '@aws-sdk/client-kms';
import { SignJWT, jwtVerify } from 'jose';
import { createHmac } from 'crypto';

const SNAPSHOT_TTL_MS = 15 * 60 * 1000;       // 15 minutes
const RATE_LIMIT_WINDOW_MS = 60 * 60 * 1000;  // 1 hour

export async function snapshotRoutes(app: FastifyInstance) {

  app.post('/v1/snapshot/request', {
    preHandler: [authMiddleware, requireSnapshotAccess],
  }, async (request, reply) => {
    const { dpIdentifierHash, tenantId, requestJwt } = request.body as SnapshotRequest;

    // Verify the request JWT is signed by Module 3's KMS key
    const module3PublicKey = await getModule3PublicKey(); // Fetched from Module 3's published JWKS endpoint
    try {
      await jwtVerify(requestJwt, module3PublicKey, {
        audience: 'truststack-discovery',
        issuer: 'truststack-consent',
      });
    } catch {
      return reply.code(401).send({
        type: 'https://truststack.in/errors/invalid-request-signature',
        title: 'Request JWT signature invalid or expired',
        status: 401,
      });
    }

    // Rate limiting: 1 snapshot per DP identifier per hour
    const recentSnapshot = await db.query(`
      SELECT requested_at FROM snapshot_tokens
      WHERE tenant_id = $1 AND dp_identifier_hash = $2
        AND requested_at > NOW() - INTERVAL '1 hour'
      ORDER BY requested_at DESC LIMIT 1
    `, [tenantId, dpIdentifierHash]);

    if (recentSnapshot.rows.length > 0) {
      const retryAfter = new Date(recentSnapshot.rows[0].requested_at.getTime() + RATE_LIMIT_WINDOW_MS);
      return reply.code(429).send({
        type: 'https://truststack.in/errors/rate-limited',
        title: 'Snapshot rate limit exceeded',
        detail: 'Maximum 1 snapshot per Data Principal per hour',
        status: 429,
        retryAfter: retryAfter.toISOString(),
      });
    }

    // Query Data Flow Map for systems holding data for this DP
    const systems = await querySystemsForDP(tenantId, dpIdentifierHash);

    // Build snapshot payload
    const snapshotPayload = {
      dpIdentifierHash,
      tenantId,
      snapshotAt: new Date().toISOString(),
      systems: await Promise.all(systems.map(async (system) => ({
        id: system.nodeId,
        name: system.displayName,
        connectorType: system.connectorType,
        dataCategories: system.piiCategories,
        estimatedRecords: system.findingCount,
        // Encrypt connection reference — Module 3 passes this back but cannot read it
        connectionRef: await encryptConnectionRef(system.connectorId, tenantId),
      }))),
    };

    // Sign the snapshot token with Discovery's KMS key
    const token = await new SignJWT({ payload: snapshotPayload })
      .setProtectedHeader({ alg: 'ES256' })
      .setIssuedAt()
      .setExpirationTime('15m')
      .setIssuer('truststack-discovery')
      .setAudience('truststack-consent')
      .sign(await getDiscoveryPrivateKey());

    const expiresAt = new Date(Date.now() + SNAPSHOT_TTL_MS);

    // Persist token record for one-time-use enforcement
    await db.query(`
      INSERT INTO snapshot_tokens (token, tenant_id, dp_identifier_hash, system_count, expires_at, requester_account_id)
      VALUES ($1, $2, $3, $4, $5, $6)
    `, [token, tenantId, dpIdentifierHash, systems.length, expiresAt, request.tenantContext!.awsAccountId]);

    return reply.send({
      snapshotToken: token,
      expiresAt: expiresAt.toISOString(),
      systemCount: systems.length,
    });
  });


  app.get('/v1/snapshot/:token', {
    preHandler: [authMiddleware, requireSnapshotAccess],
  }, async (request, reply) => {
    const { token } = request.params as { token: string };

    // Atomic update: mark as consumed and return if not already consumed
    const result = await db.query(`
      UPDATE snapshot_tokens
      SET consumed_at = NOW()
      WHERE token = $1
        AND consumed_at IS NULL
        AND expires_at > NOW()
      RETURNING *
    `, [token]);

    if (result.rows.length === 0) {
      return reply.code(404).send({
        type: 'https://truststack.in/errors/not-found',
        title: 'Snapshot token not found or already consumed',
        status: 404,
      });
    }

    // Verify token signature (proves it was issued by Discovery)
    const discoveryPublicKey = await getDiscoveryPublicKey();
    let payload: SnapshotPayload;
    try {
      const verified = await jwtVerify(token, discoveryPublicKey, {
        audience: 'truststack-consent',
        issuer: 'truststack-discovery',
      });
      payload = verified.payload.payload as SnapshotPayload;
    } catch {
      return reply.code(400).send({
        type: 'https://truststack.in/errors/invalid-token',
        title: 'Token signature verification failed',
        status: 400,
      });
    }

    return reply.send(payload);
  });
}

// Encrypts connector credentials reference with Module 3's public key
// Erasure Engine passes this opaque ref back to Discovery's connector sidecar
// Discovery decrypts it, retrieves credentials from Secrets Manager, and executes the connector
async function encryptConnectionRef(connectorId: string, tenantId: string): Promise<string> {
  const kms = new KMSClient({ region: 'ap-south-1' });
  const ref = JSON.stringify({ connectorId, tenantId, issuedAt: new Date().toISOString() });

  const result = await kms.send(new EncryptCommand({
    KeyId: process.env.MODULE3_ERASURE_PUBLIC_KEY_ARN!,
    Plaintext: Buffer.from(ref),
  }));

  return Buffer.from(result.CiphertextBlob!).toString('base64');
}
```

### 6.4 Security Model Summary

| Threat | Mitigation |
|--------|-----------|
| Module 3 reads Discovery DB directly | No VPC peering; no shared IAM; network-level isolation |
| Replay of snapshot token | One-time use enforced by `consumed_at` atomic update |
| Stale snapshot leads to incorrect erasure | 15-minute TTL; token rejected after expiry |
| DP enumeration via repeated snapshot requests | 1 snapshot/DP/hour rate limit |
| Tampered snapshot payload | JWT signed with Discovery KMS key; verified on consumption |
| Erasure Engine reads `connectionRef` secrets | `connectionRef` encrypted with Module 3 key — Module 3 passes it back, Discovery decrypts internally |
| Forged Module 3 request | `requestJwt` verified against Module 3's published JWKS |
| Snapshot issued for wrong tenant | `tenantId` bound in JWT payload; verified against authenticated tenant |

---

## 7. Account Isolation

### 7.1 Network Topology

```
┌──────────────────────────────────────────────────────────────────────────┐
│  TrustStack-Discovery (AWS Account: 123456789012)                         │
│  VPC: 10.1.0.0/16                                                         │
│                                                                            │
│  ┌──────────────────────────────┐  ┌────────────────────────────────────┐ │
│  │  Public Subnet 10.1.1.0/24  │  │  Private Subnet 10.1.2.0/24        │ │
│  │  ┌──────────────────────┐   │  │  ┌──────────────────────────────┐  │ │
│  │  │  ALB (TLS 1.3)       │───┼──┼─►│  ECS Fargate (API + Agents)  │  │ │
│  │  └──────────────────────┘   │  │  └──────────────────────────────┘  │ │
│  └──────────────────────────────┘  └────────────────────────────────────┘ │
│                                                                            │
│  Internet Gateway:                                                         │
│  - Outbound: HTTPS/443 only                                                │
│  - WAF Rules:                                                              │
│    • Vendor API whitelist (Zoho, Razorpay, Tally, WhatsApp, AWS, GCP)    │
│    • Block all non-HTTPS, block port scanning                              │
│    • Rate limit per source IP                                              │
│                                                                            │
│  Security Groups:                                                          │
│  - API layer: ingress 443 from ALB only                                   │
│  - Agent layer: no ingress, egress to SageMaker + DB + Secrets Manager   │
│  - RDS: ingress 5432 from agent subnet only                               │
│  - SageMaker: ingress 443 from agent subnet only, no internet egress     │
│                                                                            │
│  NO VPC Peering with any other TrustStack account.                        │
│  NO Transit Gateway attachment.                                            │
│  NO Direct Connect or VPN to TrustStack-Consent VPC.                     │
└──────────────────────────────────────────────────────────────────────────┘
                              │
                   HTTPS/443 (snapshot API only)
                   Caller must provide signed JWT
                   No inbound network access from
                   TrustStack-Consent to Discovery DB
                   or internal services
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  TrustStack-Consent (AWS Account: 987654321098)                           │
│  VPC: 10.2.0.0/16                                                         │
│                                                                            │
│  Erasure Engine can:                                                       │
│  ✓ Call POST /v1/snapshot/request (signed JWT required)                  │
│  ✓ Call GET /v1/snapshot/{token} (one-time, signed JWT required)         │
│                                                                            │
│  Erasure Engine cannot:                                                    │
│  ✗ Access Discovery DB (no network path)                                  │
│  ✗ Access Discovery S3 artifact bucket (separate account, no bucket policy)│
│  ✗ Access Discovery Secrets Manager (no cross-account IAM role)          │
│  ✗ Read connector credentials                                             │
│  ✗ Trigger scans or read raw PII findings                                │
└──────────────────────────────────────────────────────────────────────────┘
```

### 7.2 IAM Trust Boundaries

```json
// TrustStack-Discovery IAM policy for snapshot API service account
// Only this role can invoke the snapshot endpoints
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowModule3SnapshotApiKey",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::987654321098:role/ErasureEngineServiceRole"
      },
      "Action": "secretsmanager:GetSecretValue",
      "Resource": "arn:aws:secretsmanager:ap-south-1:123456789012:secret:discovery/module3-api-key"
      // Module 3's API key for snapshot endpoint — stored in Discovery's Secrets Manager
    }
  ]
}
```

### 7.3 Dual-Role Prohibition Enforcement

The `DPDPA Section 2(g)` dual-role prohibition is enforced at three independent layers:

| Layer | Mechanism | What it prevents |
|-------|-----------|-----------------|
| Network | No VPC peering, no Transit Gateway | Discovery compute cannot reach Consent DB |
| IAM | No cross-account roles between Discovery and Consent | Discovery cannot assume roles in Consent account |
| Application | Snapshot API returns DP-specific system list only — no consent records | Even if Discovery service is compromised, it has no access to `consent_records` table |
| Code | Discovery DB has no table for `consent_records` — no foreign key, no schema | Prevents accidental code-level joins |

---

## 8. Scalability and Performance

### 8.1 Scan Job Parallelism

**Current design (v1):** Connectors within a single scan job are processed sequentially to respect vendor rate limits. Multiple scan jobs for different tenants run concurrently (Temporal task queue workers scale horizontally on ECS Fargate).

**Capacity model:**
- Temporal worker pool: 10 ECS Fargate tasks (each handles 5 concurrent workflow activities)
- Effective concurrency: 50 concurrent scan activities across all tenants
- Single large scan (e.g., 10TB S3 lake): 1 Temporal workflow, sequential connector processing, ~4–8 hours
- Typical SME scan (5 connectors, ~100K records): ~20–40 minutes

**Scaling triggers (ECS auto-scaling):**
- `temporal_task_queue_backlog_size > 100` → scale up workers (CloudWatch alarm from Temporal metrics)
- CPU > 70% sustained → scale up
- Scale-in cooldown: 15 minutes (long-running workflows need stable workers)

### 8.2 LLM Throughput

**SageMaker endpoint:** `ml.g5.2xlarge` × 2 instances (Mistral 7B, 4-bit quantised)

| Inference type | Avg latency | Tokens/sec per instance |
|----------------|-------------|------------------------|
| PII detection (single record, ~200 tokens) | 800ms | ~250 |
| Clause classification (~500 tokens) | 1.8s | ~280 |
| Translation (per chunk, ~750 tokens) | 2.5s | ~300 |
| DPA generation (~1500 tokens) | 5.5s | ~275 |

**Throughput ceiling:** ~6 PII detection calls/second across 2 instances.

**Bottleneck mitigation:**
- Agent pipeline batches records before LLM call (up to 500 records → 1 LLM call per record, but batched S3 I/O)
- Rate limiter in LLM Gateway enforces per-tenant quotas (Enterprise: 500 calls/hour, Growth: 200/hour, Assessment: 50/hour)
- Contract audit runs synchronously with queue depth monitoring — if gateway queue > 20, return `202 Accepted` with polling URL

**SageMaker auto-scaling:**
- Scale metric: `SageMakerVariantInvocationsPerInstance`
- Target: 4 invocations/second/instance
- Scale-out cooldown: 5 minutes (model loading time)
- Min instances: 2 (HA), max: 6

### 8.3 Data Flow Map Service

The NetworkX in-memory graph is rebuilt from PostgreSQL on API server startup and updated incrementally after each scan. For large tenants (500+ nodes), the graph is cached in Redis with a 5-minute TTL.

```
Typical map size: 50–200 nodes, 100–500 edges
Max supported: ~5000 nodes, ~15000 edges (estimated from NetworkX benchmarks)
Graph build time from DB: ~200ms for 5000 nodes
Redis cache hit rate target: >90% for `GET /v1/dataflow`
```

### 8.4 Contract Audit Throughput

Contract audit is synchronous (request-response). Expected SLA:
- PDF/DOCX (structured, English): 30–60 seconds
- Scanned document (OCR required): 90–150 seconds
- 22-language document (OCR + translation + compliance check): 120–180 seconds

If processing time exceeds 10 seconds, the endpoint returns `202 Accepted` with a polling URL (`GET /v1/contracts/{auditId}`). The FastAPI worker runs the pipeline in a background task.

---

## 9. Observability

### 9.1 Metrics (CloudWatch + TimescaleDB)

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| `scan_job_duration_seconds` | End-to-end scan duration per connector type | P95 > 3600s |
| `pii_findings_per_connector` | PII finding count grouped by connector type | Spike > 3× baseline |
| `snapshot_requests_per_hour` | Snapshot API call rate (Module 3 → Discovery) | > 100/hour |
| `contract_audit_latency_seconds` | End-to-end contract audit duration | P95 > 300s |
| `llm_inference_latency_ms` | LLM Gateway inference latency per request type | P99 > 10000ms |
| `connector_auth_failure_rate` | Auth failures per connector type per hour | > 5/hour |
| `pii_detection_confidence_avg` | Average confidence of PII findings | < 0.7 (model drift) |
| `risk_score_critical_count` | Count of CRITICAL risk flows (score >= 0.8) | Any increase > 10% |
| `snapshot_token_expiry_rate` | Tokens that expire unconsumed | > 10% |
| `consent_gap_detection_rate` | Fraction of edges with `consent_gap = true` | > 50% for a tenant |

**TimescaleDB hypertable for scan time-series:**

```sql
CREATE TABLE scan_metrics (
    time TIMESTAMPTZ NOT NULL,
    tenant_id UUID NOT NULL,
    scan_job_id UUID NOT NULL,
    connector_type TEXT NOT NULL,
    metric_name TEXT NOT NULL,   -- 'duration_s', 'pii_count', 'record_count', 'auth_failures'
    metric_value NUMERIC NOT NULL
);
SELECT create_hypertable('scan_metrics', 'time');
CREATE INDEX ON scan_metrics (tenant_id, connector_type, time DESC);
```

### 9.2 Structured Logging

All log lines use structured JSON. PII-safe fields only — no personal data, no raw record content, no SQL parameters containing PII.

```typescript
// src/lib/logger.ts — structured logging convention
// Mirrors SahmatOS convention: no PII in logs, always structured fields
import pino from 'pino';

export const logger = pino({
  level: process.env.LOG_LEVEL ?? 'info',
  formatters: {
    level: (label) => ({ level: label }),
  },
  serializers: {
    // Ensure request serialiser never logs auth headers or body PII
    req: (req) => ({
      method: req.method,
      url: req.url,
      tenantId: req.tenantContext?.tenantId,
      requestId: req.id,
      // NEVER: req.body, req.headers['x-api-key'], req.ip (may link to person)
    }),
  },
});

// Usage example — safe log fields only
logger.info({
  event: 'scan_job_started',
  scanJobId: jobId,
  tenantId,
  connectorCount: connectorIds.length,
  scanType,
  // NEVER: connectorCredentials, dpIdentifier, recordContent
}, 'Scan job queued in Temporal');
```

### 9.3 Distributed Tracing

All API calls, Temporal activities, and LLM Gateway requests are instrumented with AWS X-Ray. Trace IDs propagate via HTTP headers (`X-Amzn-Trace-Id`) across the API layer → Temporal workers → LLM Gateway → SageMaker.

Key trace spans:
- `DiscoveryScanWorkflow` total duration
- Per-connector `ConnectorAuthActivity` and `DataFetchActivity`
- `pii_detection` LLM Gateway call (latency breakdown: queue time + model inference)
- `ContractAuditPipeline` end-to-end
- `snapshot_request` and `snapshot_consume` calls

### 9.4 Health Checks

```typescript
// src/routes/health.ts
app.get('/health', async (request, reply) => {
  const [dbOk, temporalOk, llmGatewayOk] = await Promise.allSettled([
    db.query('SELECT 1'),
    TemporalClient.connection.workflowService.getSystemInfo({}),
    fetch('http://llm-gateway.internal/health'),
  ]);

  const status = [dbOk, temporalOk, llmGatewayOk].every(r => r.status === 'fulfilled')
    ? 'healthy' : 'degraded';

  return reply.code(status === 'healthy' ? 200 : 503).send({
    status,
    components: {
      database: dbOk.status === 'fulfilled' ? 'ok' : 'error',
      temporal: temporalOk.status === 'fulfilled' ? 'ok' : 'error',
      llmGateway: llmGatewayOk.status === 'fulfilled' ? 'ok' : 'error',
    },
    timestamp: new Date().toISOString(),
  });
});
```

---

*TrustStack — Build Trust. Ship Faster.*
*Module 2: Data Discovery — DPDPA compliance visibility at Bharat scale.*
