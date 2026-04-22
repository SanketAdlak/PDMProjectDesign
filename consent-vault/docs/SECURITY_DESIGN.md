# Security Design — SahmatOS Consent Vault

**Version:** 1.0  
**Date:** 2026-04-22  

---

## 1. Security Philosophy

SahmatOS handles the most sensitive regulatory asset a business can hold: evidence that individuals consented to data processing. This evidence must be:
1. **Tamper-evident** — DPBI must be able to detect if a record was modified after consent
2. **Non-repudiable** — A business cannot claim "the user consented" without cryptographic proof
3. **Zero-knowledge** — TrustStack infrastructure should never be able to read the raw personal data in consent records
4. **India-sovereign** — No key material, no data, no processing outside Indian cloud regions

These four principles drive every security design decision.

---

## 2. Cryptographic Architecture

### 2.1 ECDSA P-256 Consent Signing

Every consent action produces a signed artifact. The signature:
- Proves the artifact content was not modified after signing
- Can be independently verified by DPBI without TrustStack's involvement
- Uses per-tenant keys — TrustStack cannot forge a tenant's consents

**Key parameters:**
- Algorithm: ECDSA with SHA-256 (ES256 in JWS terms)
- Curve: P-256 (NIST, widely supported, compact signatures)
- Key management: AWS KMS per-tenant asymmetric key pair

**Signing process:**

```typescript
// signing/kms-signer.ts
import { KMSClient, SignCommand } from '@aws-sdk/client-kms';

export class KMSSigner {
  async sign(artifact: ConsentArtifact, tenantKmsKeyArn: string): Promise<SignedArtifact> {
    // Canonical JSON serialisation (sorted keys, no whitespace)
    const payload = canonicalise(artifact);
    const payloadBuffer = Buffer.from(payload, 'utf-8');
    
    const command = new SignCommand({
      KeyId: tenantKmsKeyArn,
      Message: payloadBuffer,
      MessageType: 'RAW',
      SigningAlgorithm: 'ECDSA_SHA_256',
    });
    
    const response = await this.kms.send(command);
    const signature = Buffer.from(response.Signature!).toString('base64');
    
    return {
      ...artifact,
      signature,
      signingKeyId: tenantKmsKeyArn,
      signedAt: new Date().toISOString(),
    };
  }
}
```

**Verification (independent, by DPBI or auditor):**

```typescript
// Anyone with the tenant's public key can verify without TrustStack
import { KMSClient, VerifyCommand } from '@aws-sdk/client-kms';

async function verifyConsent(
  artifact: ConsentArtifact,
  signature: string,
  keyArn: string
): Promise<boolean> {
  const payload = canonicalise(artifact);
  
  const command = new VerifyCommand({
    KeyId: keyArn,
    Message: Buffer.from(payload),
    MessageType: 'RAW',
    Signature: Buffer.from(signature, 'base64'),
    SigningAlgorithm: 'ECDSA_SHA_256',
  });
  
  const result = await kms.send(command);
  return result.SignatureValid === true;
}
```

### 2.2 Canonical Artifact Format

The artifact signed by KMS must be deterministic — same content always produces same bytes:

```typescript
// Signed payload structure (no signature field in the payload being signed)
interface ConsentArtifactPayload {
  schemaVersion: '1.0';
  artifactId: string;
  tenantId: string;
  dpIdentifierHash: string;      // HMAC-SHA256(identifier, tenantSecret)
  purposeId: string;
  purposeVersion: number;
  noticeText: string;            // Actual notice text shown to user (in their language)
  languageCode: string;
  action: 'GRANTED' | 'WITHDRAWN';
  collectionMethod: string;
  collectionTimestamp: string;   // ISO 8601 UTC, millisecond precision
  dataResidencyRegion: string;
}

function canonicalise(payload: ConsentArtifactPayload): string {
  // Sort keys alphabetically, no whitespace, no trailing commas
  return JSON.stringify(payload, Object.keys(payload).sort());
}
```

### 2.3 Data Encryption (AES-256 at Rest)

All data stored in RDS is encrypted at rest using AWS RDS encryption (AES-256, KMS-managed).

Per-tenant data isolation via separate KMS symmetric keys:

```
AWS KMS (ap-south-1)
├── TrustStack Master Key (platform-level)
│   └── Used for: consumer account identifiers, platform config
├── Tenant A Symmetric Key (AES-256)
│   └── Used for: dp_identifiers.identifier_encrypted for Tenant A
├── Tenant A Signing Key (ECDSA P-256 asymmetric)
│   └── Used for: consent artifact signing for Tenant A
├── Tenant B Symmetric Key (AES-256)
│   └── Used for: dp_identifiers.identifier_encrypted for Tenant B
└── Tenant B Signing Key (ECDSA P-256 asymmetric)
    └── Used for: consent artifact signing for Tenant B
```

### 2.4 Transport Security

- TLS 1.3 for all external connections (API, widget CDN, webhooks)
- TLS 1.2 minimum for internal service-to-service (KMS, RDS, Kafka MSK, Redis ElastiCache)
- Certificate pinning in native mobile SDK
- HSTS with max-age=31536000 for API domain

---

## 3. Zero-Knowledge Design

TrustStack infrastructure must not be able to read personal data content. This is enforced:

### 3.1 DP Identifier Pseudonymisation

Raw identifiers (phone numbers, emails) are:
1. Never stored in `consent_records` — only the HMAC-SHA256 hash
2. Stored in `dp_identifiers` table — encrypted with tenant's KMS key
3. TrustStack cannot decrypt without the tenant's KMS key (which tenant controls)

```typescript
// The hash stored in consent_records
function hashDpIdentifier(identifier: string, tenantSecret: string): string {
  return createHmac('sha256', tenantSecret)
    .update(identifier)
    .digest('hex');
}
// Prefix: 'sha256:' + hex — consistent format for DPBI

// The encrypted version stored in dp_identifiers (only for erasure/consumer use)
async function encryptDpIdentifier(
  identifier: string,
  tenantKmsKeyArn: string
): Promise<string> {
  const encrypted = await kms.encrypt({
    KeyId: tenantKmsKeyArn,
    Plaintext: Buffer.from(identifier),
  });
  return Buffer.from(encrypted.CiphertextBlob!).toString('base64');
}
```

### 3.2 Access Controls

The `dp_identifiers` table has stricter access than `consent_records`:

| Operation | consent_records | dp_identifiers |
|-----------|----------------|----------------|
| Consent service (grant/withdraw) | Read + Write | Write only |
| Consent check service | Read (via hash) | No access |
| Erasure service | Read | Read (decrypt to send notification) |
| Consumer dashboard | Read (own consents) | Read (own identifier) |
| Audit log export | Read (hashes only) | No access |
| TrustStack admin | Read (metadata) | No access without tenant KMS key |

---

## 4. Key Management Lifecycle

### 4.1 Key Creation (Tenant Onboarding)

```typescript
// tenant/onboarding.ts
async function provisionTenantKeys(tenantId: string): Promise<TenantKeys> {
  // Symmetric key for data encryption
  const symmetricKey = await kms.createKey({
    KeySpec: 'SYMMETRIC_DEFAULT',    // AES-256
    KeyUsage: 'ENCRYPT_DECRYPT',
    Description: `SahmatOS Tenant ${tenantId} - Data Encryption`,
    Tags: [
      { TagKey: 'tenantId', TagValue: tenantId },
      { TagKey: 'purpose', TagValue: 'data-encryption' },
      { TagKey: 'service', TagValue: 'sahmat-os' },
    ],
    MultiRegion: false,              // ap-south-1 only (Section 16 compliance)
  });
  
  // Asymmetric key pair for consent signing
  const signingKey = await kms.createKey({
    KeySpec: 'ECC_NIST_P256',
    KeyUsage: 'SIGN_VERIFY',
    Description: `SahmatOS Tenant ${tenantId} - Consent Signing`,
    Tags: [
      { TagKey: 'tenantId', TagValue: tenantId },
      { TagKey: 'purpose', TagValue: 'consent-signing' },
    ],
    MultiRegion: false,
  });
  
  // Enable automatic annual key rotation (symmetric only — KMS handles rotation)
  await kms.enableKeyRotation({ KeyId: symmetricKey.KeyMetadata!.KeyId! });
  
  return {
    symmetricKeyArn: symmetricKey.KeyMetadata!.Arn!,
    signingKeyArn: signingKey.KeyMetadata!.Arn!,
  };
}
```

### 4.2 Key Rotation

| Key Type | Rotation | Mechanism |
|----------|----------|-----------|
| Symmetric (AES-256) | Annual | AWS KMS automatic rotation (key material rotates, ARN stays same) |
| Asymmetric signing (ECDSA P-256) | Annual | Manual rotation (new key ARN, old key retained for verification) |

**Signing key rotation — why manual:**
Old artifacts were signed with the old key. DPBI may audit artifacts from years ago. Old signing keys must be retained indefinitely (marked "inactive" but not deleted) so historical artifacts remain verifiable.

```typescript
// Key rotation procedure for signing keys (run annually)
async function rotateSigningKey(tenantId: string): Promise<void> {
  const tenant = await getTenant(tenantId);
  
  // Create new signing key
  const newKey = await provisionSigningKey(tenantId);
  
  // Mark old key as inactive (NOT deleted)
  await updateTenant(tenantId, {
    kmsSigningKeyArn: newKey.arn,
    previousSigningKeyArns: [...tenant.previousSigningKeyArns, tenant.kmsSigningKeyArn],
  });
  
  // Disable old KMS key for new operations (still allows Verify operations)
  await kms.disableKey({ KeyId: extractKeyId(tenant.kmsSigningKeyArn) });
  
  // Audit log
  await audit.log('tenant.signing_key_rotated', {
    tenantId,
    newKeyArn: newKey.arn,
    oldKeyArn: tenant.kmsSigningKeyArn,
  });
}
```

### 4.3 Key Deletion (Tenant Offboarding)

```
1. Tenant requests offboarding
2. 30-day notice period begins
3. All Temporal.io workflows complete or are cancelled
4. All erasure records executed
5. Deletion certificates archived to S3 (Glacier, 7-year)
6. KMS keys scheduled for deletion (minimum 7-day pending deletion period)
7. Tenant record soft-deleted from tenants table
8. After 7 days: KMS keys permanently deleted
```

---

## 5. Identity and Access Management

### 5.1 API Key Management

```typescript
// API keys: ts_live_{32-hex-chars} (production), ts_test_{32-hex-chars} (sandbox)
function generateApiKey(environment: 'live' | 'test'): ApiKey {
  const rawKey = randomBytes(32).toString('hex');
  const keyHash = createHash('sha256').update(rawKey).digest('hex');
  
  return {
    displayKey: `ts_${environment}_${rawKey}`,  // Shown once to tenant, never again
    keyHash,                                      // Stored in DB (never the raw key)
  };
}

// API key lookup is by hash
async function validateApiKey(apiKey: string): Promise<Tenant | null> {
  const keyHash = createHash('sha256').update(apiKey).digest('hex');
  return await tenantRepo.findByApiKeyHash(keyHash);
}
```

### 5.2 Role-Based Access Control (RBAC)

| Role | Permissions |
|------|-------------|
| `TENANT_OWNER` | All operations for own tenant |
| `TENANT_DEVELOPER` | Read/write consent operations; no billing/key management |
| `TENANT_READONLY` | Read consent records and audit logs |
| `CONSUMER` | Read and manage own consents only (consumer dashboard) |
| `TS_ADMIN` | Cross-tenant audit; no access to encrypted PII |
| `TS_COMPLIANCE` | Breach reports, DPBI audit exports |
| `TEMPORAL_WORKER` | Internal: erasure and breach workflow operations only |

### 5.3 Service-to-Service Authentication

Internal services authenticate via:
- AWS IAM roles (ECS task roles with least-privilege policies)
- KMS key policies (only specific IAM roles can use each tenant's key)
- Temporal.io: mutual TLS between server and workers

---

## 6. Network Security

### 6.1 VPC Architecture

```
AWS ap-south-1 — TrustStack-Consent Account
├── VPC: 10.0.0.0/16
│   ├── Public Subnets (10.0.1.0/24, 10.0.2.0/24)
│   │   └── Application Load Balancer (TLS termination)
│   ├── Private Subnets (10.0.10.0/24, 10.0.11.0/24)
│   │   ├── ECS Fargate (API servers)
│   │   ├── Temporal.io workers
│   │   └── Kafka consumers
│   └── Isolated Subnets (10.0.20.0/24, 10.0.21.0/24)
│       ├── RDS Multi-AZ (PostgreSQL)
│       ├── ElastiCache (Redis)
│       └── MSK (Kafka)
│
├── Security Groups
│   ├── ALB: inbound 443 from internet; outbound to API servers
│   ├── API servers: inbound from ALB only; outbound to RDS/Redis/Kafka/KMS
│   ├── RDS: inbound from API servers + Temporal workers only; no internet
│   ├── Redis: inbound from API servers only
│   └── Kafka: inbound from API servers + Kafka consumers only
│
└── NO VPC peering to TrustStack-Discovery account (dual-role prohibition)
```

### 6.2 Secrets Management

All secrets (DB passwords, Kafka credentials, Temporal certs) stored in AWS Secrets Manager.

**No secrets in environment variables, no secrets in code.**

```typescript
// secrets/manager.ts
async function getSecret(secretName: string): Promise<string> {
  const client = new SecretsManagerClient({ region: 'ap-south-1' });
  const response = await client.send(new GetSecretValueCommand({ SecretId: secretName }));
  return response.SecretString!;
}

// Usage: DB connection string fetched at startup, cached in memory, rotated on scheduled trigger
```

---

## 7. Input Validation and Injection Prevention

```typescript
// All API inputs validated before processing

// DP Identifier validation
const DP_IDENTIFIER_SCHEMAS = {
  phone: z.string().regex(/^\+91[6-9]\d{9}$/),          // Indian mobile numbers
  email: z.string().email().max(254),
  user_id: z.string().min(1).max(256).regex(/^[a-zA-Z0-9_\-\.]+$/),
  aadhaar_virtual_id: z.string().regex(/^\d{16}$/),       // 16-digit VID
};

// Purpose ID: UUID only — prevents injection in SQL queries
const PURPOSE_ID_SCHEMA = z.string().uuid();

// Language code: allowlist — prevents arbitrary code injection  
const LANGUAGE_CODE_SCHEMA = z.enum([
  'as', 'bn', 'brx', 'doi', 'gu', 'hi', 'kn',
  'ks-Arab', 'ks-Deva', 'kok', 'mai', 'ml', 'mni-Mtei',
  'mr', 'ne', 'or', 'pa', 'sa', 'sat-Olck', 'sd-Arab', 'sd-Deva',
  'ta', 'te', 'ur',
]);

// All DB queries use parameterised statements via pg (node-postgres)
// No raw SQL string interpolation
const result = await db.query(
  'SELECT * FROM consent_records WHERE tenant_id = $1 AND dp_identifier_hash = $2',
  [tenantId, dpIdentifierHash]  // Parameters, never string concatenated
);
```

---

## 8. Audit Trail Integrity

The audit log is itself append-only (same trigger as consent_records). Additionally:

1. Every audit log entry has a monotonically increasing sequence number per tenant
2. Each entry includes SHA-256 hash of the previous entry (blockchain-style chain)
3. Chain integrity can be verified by DPBI auditors independently

```sql
-- Audit log hash chain (verified during DPBI audit)
ALTER TABLE audit_log ADD COLUMN sequence_number BIGINT;
ALTER TABLE audit_log ADD COLUMN prev_entry_hash TEXT;

CREATE SEQUENCE audit_log_seq_<tenant_id>;  -- Per-tenant sequence

-- On insert, compute prev_hash and sequence
CREATE OR REPLACE FUNCTION set_audit_chain()
RETURNS TRIGGER AS $$
DECLARE
  prev_hash TEXT;
  seq BIGINT;
BEGIN
  seq := nextval('audit_log_seq_' || NEW.tenant_id::text);
  SELECT sha256(id::text || created_at::text || event_data::text)
  INTO prev_hash
  FROM audit_log
  WHERE tenant_id = NEW.tenant_id
  ORDER BY sequence_number DESC
  LIMIT 1;
  
  NEW.sequence_number := seq;
  NEW.prev_entry_hash := COALESCE(prev_hash, 'GENESIS');
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

---

## 9. Vulnerability Management

### 9.1 Dependency Scanning

```bash
# Run on every PR via CI
npm audit --audit-level=moderate  # Fail build on moderate+ vulnerabilities
npx better-npm-audit audit --level moderate

# Weekly full scan
npm audit --json | jq '.vulnerabilities | to_entries[] | select(.value.severity == "critical")'
```

### 9.2 SAST (Static Analysis)

```bash
# In CI pipeline
npx semgrep --config=p/typescript --config=p/nodejs-security --error
```

### 9.3 Penetration Testing Schedule

| Type | Frequency | Scope |
|------|-----------|-------|
| Automated DAST | Every deployment | API endpoints, widget |
| External penetration test | Annual | Full scope |
| DPBI sandbox compliance test | Before CM registration (Nov 2026) | Consent artifacts, audit log |
| Third-party security audit | Q4 2026 | Full code + infrastructure |

---

## 10. Incident Response

### 10.1 Security Incident Classification

| Level | Description | Response Time | Escalation |
|-------|-------------|---------------|-----------|
| P0 | Active breach, data exfiltration | 15 minutes | CEO + Legal + DPBI (Rule 7) |
| P1 | Suspected unauthorised access | 1 hour | CTO + Security team |
| P2 | Vulnerability discovered (no active exploit) | 4 hours | Engineering team |
| P3 | Configuration/compliance issue | 24 hours | Engineering team |

### 10.2 Breach Response Sequence

```
T+0:    Breach detected (automated or manual)
T+15m:  POST /v1/breach triggers Temporal.io workflow
T+30m:  Internal incident declared, team assembled
T+1h:   Initial scope assessment complete
T+4h:   Affected Data Principals identified
T+6h:   ⚡ CERT-In initial report SUBMITTED (Rule 7 deadline)
T+8h:   DP notifications sent in their languages
T+24h:  Full forensic analysis in progress
T+72h:  ⚡ DPBI detailed report SUBMITTED (Rule 7 deadline)
T+7d:   Post-incident review and DPBI supplementary report
```

---

## 11. Compliance Certifications (Target)

| Certification | Target Date | Scope |
|---------------|-------------|-------|
| SOC 2 Type II | Q2 2027 | Security, Availability, Confidentiality |
| ISO 27001 | Q4 2027 | Full ISMS |
| DPDPA Consent Manager Registration (Rule 4) | Nov 2026 | Mandatory for operation |
| MeitY Cloud empanelment | Q1 2027 | Government sector customers |
