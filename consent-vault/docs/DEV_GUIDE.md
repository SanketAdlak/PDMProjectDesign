# Developer Guide — SahmatOS Consent Vault

**Version:** 1.0  
**Date:** 2026-04-22  

---

## 1. Prerequisites

| Tool | Version | Install |
|------|---------|---------|
| Node.js | 20 LTS | `nvm install 20` |
| npm | 10+ | Included with Node 20 |
| Docker + Docker Compose | 4.x | docker.com |
| AWS CLI | 2.x | `brew install awscli` |
| Temporal CLI | latest | `brew install temporal` |
| PostgreSQL client | 16 | `brew install postgresql@16` |

**AWS Setup (local dev):**
```bash
# Configure a profile for local dev (uses LocalStack — no real AWS charges)
aws configure --profile sahmat-local
# Access Key ID: test
# Secret Access Key: test
# Region: ap-south-1
# Output: json
```

---

## 2. Local Development Setup

### 2.1 Clone and Install

```bash
git clone <repo-url>
cd consent-vault
npm install
```

### 2.2 Start Infrastructure (Docker Compose)

```bash
# Start all local infrastructure
docker-compose up -d

# Verify everything is up
docker-compose ps

# Services started:
# - postgres:16 on :5432
# - redis:7 on :6379
# - kafka:3.6 (+ zookeeper) on :9092
# - temporal:1.24 on :7233 (UI: :8233)
# - localstack on :4566 (AWS KMS, Secrets Manager, SQS)
```

### 2.3 Database Setup

```bash
# Run all migrations
npm run db:migrate

# Verify migrations applied
npm run db:status

# Seed test data (includes test tenant, purposes in all 22 languages)
npm run db:seed

# Reset to clean state (caution: destroys all local data)
npm run db:reset
```

### 2.4 Environment Configuration

```bash
# Copy the example env file
cp .env.example .env.local

# Edit .env.local:
# DATABASE_URL=postgresql://sahmat:password@localhost:5432/sahmat_dev
# REDIS_URL=redis://localhost:6379
# KAFKA_BROKERS=localhost:9092
# TEMPORAL_ADDRESS=localhost:7233
# AWS_ENDPOINT=http://localhost:4566  # LocalStack
# AWS_REGION=ap-south-1
# LOG_LEVEL=debug
# TENANT_KMS_KEY_ARN=arn:aws:kms:ap-south-1:000000000000:key/test-key-id
```

### 2.5 Start the API Server

```bash
# Development (watch mode, hot reload)
npm run dev

# API available at http://localhost:3000
# Swagger UI at http://localhost:3000/docs
# Health check at http://localhost:3000/health
```

### 2.6 Start Temporal Workers

```bash
# In a separate terminal
npm run worker:erasure    # Erasure workflow workers
npm run worker:breach     # Breach notification workers

# Temporal Web UI: http://localhost:8233
```

---

## 3. Project Structure

```
consent-vault/
├── src/
│   ├── api/                    # Express routes + OpenAPI spec
│   │   ├── consent/
│   │   │   ├── routes.ts       # POST /v1/consent, GET /v1/consent/check
│   │   │   ├── handlers.ts     # Route handler functions
│   │   │   └── schemas.ts      # Zod validation schemas
│   │   ├── erasure/
│   │   ├── breach/
│   │   ├── audit/
│   │   └── widget/
│   │
│   ├── domain/                 # Business logic (no HTTP concerns)
│   │   ├── consent/
│   │   │   ├── service.ts      # ConsentService
│   │   │   ├── signer.ts       # KMS signing
│   │   │   └── repository.ts   # DB queries
│   │   ├── erasure/
│   │   │   ├── service.ts
│   │   │   └── workflows.ts    # Temporal.io workflow definitions
│   │   ├── breach/
│   │   │   ├── service.ts
│   │   │   └── workflows.ts
│   │   └── language/
│   │       ├── engine.ts       # Language detection + translation
│   │       └── glossary.ts     # DPDPA term translations
│   │
│   ├── infra/                  # Infrastructure adapters
│   │   ├── db/
│   │   │   ├── client.ts       # pg connection pool
│   │   │   └── migrations/     # SQL migration files
│   │   ├── cache/
│   │   │   └── redis.ts        # Redis client
│   │   ├── kafka/
│   │   │   ├── producer.ts
│   │   │   └── consumers/      # Per-topic consumers
│   │   ├── kms/
│   │   │   └── client.ts       # AWS KMS wrapper
│   │   └── temporal/
│   │       └── client.ts       # Temporal client
│   │
│   ├── i18n/                   # Language support
│   │   ├── language-codes.ts   # BCP 47 code map
│   │   ├── font-map.ts         # Language → font mapping
│   │   ├── rtl-languages.ts    # RTL language list
│   │   └── glossary/           # DPDPA term translations per language
│   │       ├── hi.ts           # Hindi
│   │       ├── ta.ts           # Tamil
│   │       └── ...             # All 22 languages
│   │
│   └── shared/
│       ├── logger.ts           # Structured PII-safe logging
│       ├── errors.ts           # Error types + RFC 9457 formatter
│       └── crypto.ts           # HMAC, hash utilities
│
├── tests/
│   ├── unit/                   # Vitest unit tests (mock infra)
│   ├── integration/            # Vitest integration tests (real DB, Redis, Kafka)
│   │   └── language/           # Per-language rendering tests (all 22)
│   └── e2e/                    # Playwright E2E tests
│
├── docs/                       # This directory
├── diagrams/
├── docker-compose.yml
├── vitest.config.ts
└── package.json
```

---

## 4. Coding Conventions

### 4.1 TypeScript and Module System

```typescript
// ✅ ESM imports — always use file extensions
import { ConsentService } from './service.ts';
import type { ConsentArtifact } from '../domain/consent/types.ts';

// ❌ Never use CommonJS
const { ConsentService } = require('./service');
module.exports = { ... };
```

### 4.2 DPDPA Section References

All code that processes personal data **must** include a comment referencing the applicable DPDPA section:

```typescript
// ✅ Required — reference DPDPA section for any PII-adjacent code
// DPDPA Section 5 Rule 3: notice must be in user's interface language
const languageCode = await this.languageEngine.detect(req);

// DPDPA Rule 4: consent records are append-only, 7-year retention
await this.db.query('INSERT INTO consent_records ...', [...]);

// DPDPA Section 9: parental consent required for minors under 18
if (isMinor) {
  return this.parentalConsentService.initiateGuardianFlow(req);
}

// ❌ PII handling without DPDPA reference is a code review blocker
const record = await this.db.query('INSERT INTO consent_records ...');
```

### 4.3 No PII Logging

```typescript
// ✅ Log hashed identifiers and IDs only
logger.info('consent.granted', {
  tenantId: artifact.tenantId,
  dpIdentifierHash: artifact.dpIdentifierHash,  // HMAC hash, not raw
  purposeId: artifact.purposeId,
  languageCode: artifact.languageCode,
});

// ❌ Never log raw personal data — instant code review rejection
logger.info('consent.granted', {
  phone: req.dpIdentifier,           // PII
  email: req.dpIdentifier,           // PII
});
```

### 4.4 Database Queries

```typescript
// ✅ Always parameterised queries via pg
const result = await db.query(
  `SELECT * FROM consent_records
   WHERE tenant_id = $1 AND dp_identifier_hash = $2
   ORDER BY created_at DESC LIMIT $3`,
  [tenantId, dpIdentifierHash, limit]
);

// ❌ Never string interpolation in SQL — SQL injection risk
const result = await db.query(
  `SELECT * FROM consent_records WHERE tenant_id = '${tenantId}'`
);
```

### 4.5 Error Handling

```typescript
// Use domain-specific error types (not generic Error)
export class ConsentNotFoundError extends Error {
  readonly type = 'CONSENT_NOT_FOUND';
  readonly status = 404;
  
  constructor(public readonly tenantId: string, public readonly purposeId: string) {
    super(`No consent found for purpose ${purposeId}`);
  }
}

// API layer converts to RFC 9457 response
// Domain layer throws typed errors, never HTTP status codes
```

### 4.6 Language Code Handling

```typescript
// ✅ Always validate language codes against the allowlist
import { SUPPORTED_LANGUAGE_CODES } from '../i18n/language-codes.ts';

if (!SUPPORTED_LANGUAGE_CODES.includes(languageCode)) {
  throw new ValidationError(`Unsupported language code: ${languageCode}`);
}

// ✅ Use the canonical normalised form
const canonical = LANGUAGE_CODE_MAP[rawCode] ?? 'hi'; // Default to Hindi
```

### 4.7 Commit Message Convention

```
feat(consent): add WhatsApp collection method support
fix(erasure): handle ECOMMERCE_GT_2CR class with null last_activity
docs(api): add language override examples to API spec
refactor(language): extract font loading to dedicated service
test(lang): add Urdu RTL rendering integration tests
```

Format: `type(scope): description`

Types: `feat`, `fix`, `docs`, `refactor`, `test`, `perf`, `chore`

Scopes: `consent`, `erasure`, `breach`, `language`, `api`, `db`, `infra`, `i18n`

### 4.8 Branch Naming

```
feature/consent-whatsapp-collection
feature/erasure-class-specific-timelines
fix/consent-language-detection-fallback
fix/breach-cert-in-deadline-calculation
```

---

## 5. Testing

### 5.1 Test Strategy

| Layer | Tool | What it tests | Infrastructure |
|-------|------|--------------|----------------|
| Unit | Vitest | Business logic in isolation | All mocked |
| Integration | Vitest | Service + real DB/Redis/Kafka | Docker Compose |
| Language | Vitest | All 22 language rendering | Docker Compose |
| E2E | Playwright | Full API flows | Full stack |
| Load | k6 | Performance targets | Staging |

### 5.2 Running Tests

```bash
# Unit tests (fast, ~5s)
npm test

# Integration tests (requires Docker Compose running, ~60s)
npm run test:integration

# Language tests (all 22 languages, ~30s)
npm run test:lang

# E2E tests (requires full stack, ~120s)
npm run test:e2e

# All tests
npm run test:all

# Coverage report
npm run test:coverage
# Minimum coverage thresholds:
# Lines: 80%, Branches: 75%, Functions: 80%
# Domain layer: 90% (higher — this is where DPDPA logic lives)
```

### 5.3 Writing Tests for Consent Features

```typescript
// tests/unit/consent/service.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { ConsentService } from '../../../src/domain/consent/service.ts';

describe('ConsentService.grant', () => {
  let service: ConsentService;
  let mockPurposeRepo: MockPurposeRepo;
  let mockSigner: MockSigner;
  
  beforeEach(() => {
    mockPurposeRepo = createMockPurposeRepo();
    mockSigner = createMockSigner();
    service = new ConsentService(mockPurposeRepo, mockSigner, ...);
  });
  
  it('stores language code in consent artifact', async () => {
    // DPDPA Rule 3: language must be recorded
    const artifact = await service.grant({
      tenantId: TEST_TENANT_ID,
      dpIdentifier: '+919876543210',
      dpIdentifierType: 'phone',
      purposeId: TEST_PURPOSE_ID,
      action: 'GRANTED',
      collectionMethod: 'WIDGET',
      languageCode: 'ta', // Tamil
    });
    
    expect(artifact.languageCode).toBe('ta');
  });
  
  it('routes minor to guardian flow (Section 9)', async () => {
    // DPDPA Section 9: verifiable parental consent for under-18
    mockPurposeRepo.getDpAge.mockResolvedValue(15); // Under 18
    
    await expect(service.grant({ ...baseRequest, dpIdentifier: MINOR_ID }))
      .rejects.toThrow('MINOR_WITHOUT_GUARDIAN');
  });
  
  it('rejects pre-checked consent (no affirmative action)', async () => {
    // DPDPA Section 6: consent must be free and affirmative
    await expect(service.grant({ ...baseRequest, action: 'AUTO_GRANTED' }))
      .rejects.toThrow('INVALID_ACTION');
  });
});
```

### 5.4 Language-Specific Tests

```typescript
// tests/integration/language/all-languages.test.ts

const ALL_LANGUAGES = [
  { code: 'hi', script: 'Devanagari', direction: 'ltr' },
  { code: 'ta', script: 'Tamil', direction: 'ltr' },
  { code: 'ur', script: 'Nastaliq', direction: 'rtl' },
  { code: 'ks-Arab', script: 'Nastaliq', direction: 'rtl' },
  { code: 'mni-Mtei', script: 'MeiteiMayek', direction: 'ltr' },
  { code: 'sat-Olck', script: 'OlChiki', direction: 'ltr' },
  // ... all 22
] satisfies LanguageTestCase[];

describe.each(ALL_LANGUAGES)('Language: $code ($script)', ({ code, direction }) => {
  it('renders consent notice without character corruption', async () => {
    const notice = await languageEngine.getConsentNotice(TEST_PURPOSE_ID, code);
    
    // No Unicode replacement characters
    expect(notice.description).not.toContain('\uFFFD');
    // Non-empty
    expect(notice.description.length).toBeGreaterThan(10);
  });
  
  it('has correct text direction metadata', async () => {
    const ctx = await languageEngine.detect({ languageCode: code });
    expect(ctx.direction).toBe(direction);
  });
  
  it('returns approved font for this script', async () => {
    const ctx = await languageEngine.detect({ languageCode: code });
    expect(FONT_MAP).toHaveProperty(code);
    expect(ctx.fontFamily).toBeTruthy();
  });
});
```

---

## 6. Database Migrations

### 6.1 Creating a Migration

```bash
npm run db:migration:create -- --name add_consent_expiry_support
# Creates: src/infra/db/migrations/0012_add_consent_expiry_support.sql
#          src/infra/db/migrations/0012_add_consent_expiry_support.down.sql
```

### 6.2 Migration Rules

1. **Always reversible** — down migration must cleanly undo the up migration
2. **Additive first** — add new columns as nullable, populate, then add NOT NULL
3. **Never rename columns** — add new column, migrate data, deprecate old (3-sprint cycle)
4. **Never DROP** — without a deprecation cycle documented in the migration comment
5. **Test both up and down** before PR

```sql
-- Example: Adding optional consent expiry
-- Migration: 0012_add_consent_expiry_support.sql

-- Safe: adding nullable column
ALTER TABLE consent_purposes ADD COLUMN default_expiry_days INTEGER;

-- NOT safe (would break existing queries): 
-- ALTER TABLE consent_purposes ADD COLUMN default_expiry_days INTEGER NOT NULL;

-- Down migration: 0012_add_consent_expiry_support.down.sql
ALTER TABLE consent_purposes DROP COLUMN default_expiry_days;
```

---

## 7. Adding a New Language

When adding support for a new Eighth Schedule language:

1. **Add language code to constants:**
   ```typescript
   // src/i18n/language-codes.ts
   export const SUPPORTED_LANGUAGE_CODES = [
     'hi', 'ta', /* ... existing ... */,
     'doi', // ← Add new language code
   ] as const;
   ```

2. **Add font mapping:**
   ```typescript
   // src/i18n/font-map.ts
   FONT_MAP['doi'] = {
     fontFamily: 'Noto Sans Devanagari', // Dogri uses Devanagari script
     url: 'https://fonts.truststack.in/NotoSansDevanagari.woff2',
     script: 'Devanagari',
   };
   ```

3. **Add DPDPA glossary:**
   ```typescript
   // src/i18n/glossary/doi.ts
   export const DPDPA_GLOSSARY_DOI = {
     consent: 'सहमति',  // Dogri — Devanagari script
     withdrawal: 'वापसी',
     // ... (get native speaker to verify)
   };
   ```

4. **Add test case in language test suite** (see 5.4 above)

5. **Submit for human review:** Create a GitHub issue with `lang-review` label. Native speaker + legal review required before language reaches production.

---

## 8. Local Temporal.io Development

```bash
# Start Temporal dev server (included in docker-compose)
# Web UI: http://localhost:8233

# Run erasure workflow locally (test with 30-second delay instead of 3 years)
npm run worker:erasure -- --mode dev

# Trigger a test erasure
curl -X POST http://localhost:3000/v1/erasure \
  -H "X-API-Key: ts_test_abc123" \
  -H "X-Tenant-Id: $(cat .env.local | grep TEST_TENANT_ID | cut -d= -f2)" \
  -H "Content-Type: application/json" \
  -d '{
    "tenantId": "...",
    "dpIdentifier": "+919876543210",
    "dpIdentifierType": "phone",
    "purposeId": "...",
    "triggeredBy": "DP_REQUEST",
    "notificationLanguage": "hi"
  }'

# Watch workflow progress in Temporal UI or CLI
temporal workflow show --workflow-id "erasure-<id>"
```

---

## 9. Environment Promotion

```
local → dev → staging → production
```

| Stage | Infra | Data | AWS Region |
|-------|-------|------|-----------|
| local | Docker Compose (LocalStack) | Seeded test data | Simulated |
| dev | AWS ECS (t3.small) | Anonymised subset | ap-south-1 |
| staging | AWS ECS (t3.medium) | Anonymised prod-like | ap-south-1 |
| production | AWS ECS (t3.large + auto-scale) | Real data | ap-south-1 |

**Never run migrations on production manually** — all migrations go through CI/CD.

---

## 10. CI/CD Pipeline

```yaml
# GitHub Actions: .github/workflows/ci.yml (outline)

on: [push, pull_request]

jobs:
  lint:
    - npm run lint (ESLint + TypeScript strict)
    - npm run typecheck

  test:
    - npm test (unit)
    - npm run test:integration (with Docker services)
    - npm run test:lang (all 22 languages)

  security:
    - npm audit --audit-level=moderate
    - npx semgrep --config=p/typescript

  build:
    - npm run build
    - docker build (verify Dockerfile)

  deploy-dev:          # Auto on main branch
    - docker push
    - ECS deploy (dev)
    - npm run db:migrate (dev)

  deploy-staging:      # Manual trigger
    - Same as dev, staging infra

  deploy-prod:         # Manual trigger with approval gate
    - Same, production infra
    - Post-deploy: smoke tests on /health + consent check endpoint
```

---

## 11. Common Issues and Solutions

### "Missing font for language X"
```bash
# Check font map
cat src/i18n/font-map.ts | grep -A3 '"X"'

# If missing, see Section 7 (Adding a New Language)
```

### "Consent immutability trigger firing unexpectedly"
```bash
# This is working as designed — consent_records is append-only
# If you need to correct a record, the correct approach is:
# 1. Insert a new WITHDRAWN record
# 2. Then insert a new GRANTED record with the corrected data
# Never attempt to UPDATE consent_records
```

### "Temporal workflow stuck in pending"
```bash
# Check Temporal Web UI: http://localhost:8233
# Common cause: worker not running
npm run worker:erasure

# Or: check worker logs
docker-compose logs temporal-worker
```

### "KMS signing fails in local dev"
```bash
# LocalStack KMS must be running
docker-compose ps localstack

# Check LocalStack KMS has the test key
aws --endpoint-url=http://localhost:4566 kms list-keys
# If empty, run the LocalStack seeder:
npm run localstack:seed
```
