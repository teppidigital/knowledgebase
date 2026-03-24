# Data Privacy & Compliance Engineering

## Category
Data Solutions, Privacy Engineering, GDPR, CCPA, Right to Erasure, Data Anonymisation, Tokenisation, Differential Privacy, PII

## Context

**Privacy engineering** embeds data protection requirements directly into system design and automated pipelines — preventing privacy violations by construction rather than relying solely on policy enforcement. Regulations like GDPR (EU), CCPA (California), LGPD (Brazil), and PDPA (Singapore) mandate specific data handling obligations with significant penalties for non-compliance.

### Key regulatory requirements

| Requirement | GDPR Article | Engineering implication |
|------------|-------------|------------------------|
| **Right of Access (SAR)** | Art. 15 | Can retrieve all data held about a subject in < 30 days |
| **Right to Erasure** | Art. 17 | Can delete all personal data about a subject across all systems |
| **Data Minimisation** | Art. 5 | Only collect data with a documented purpose |
| **Purpose Limitation** | Art. 5 | Data collected for A cannot be used for B without consent |
| **Storage Limitation** | Art. 5 | Auto-delete data once the retention period expires |
| **Breach Notification** | Art. 33 | Must notify regulator within 72 hours of a breach |
| **Privacy by Design** | Art. 25 | Privacy controls built in, not bolted on |

### Anonymisation vs pseudonymisation

| Technique | Description | Reversible? | GDPR status |
|-----------|-------------|------------|-------------|
| **Pseudonymisation** | Replace identifier with token; mapping table exists | Yes (with mapping) | Still "personal data" |
| **Anonymisation** | Irreversible — individual cannot be re-identified | No | Out of GDPR scope |
| **Tokenisation** | Replace PII with a non-sensitive token; vault stores mapping | Yes (with vault) | Common for payments |
| **Generalisation** | Replace precise value with range (age 34 → 30–40) | No | Anonymisation technique |
| **Data masking** | Replace with realistic fake data (name → "John Smith") | No | Used in non-prod environments |
| **Differential privacy** | Add calibrated noise to aggregate queries | N/A | Statistical guarantee |

### Data classification tiers

| Tier | Examples | Handling |
|------|---------|---------|
| **PUBLIC** | Marketing copy, press releases | No restriction |
| **INTERNAL** | Business metrics, architecture docs | Employee access |
| **CONFIDENTIAL** | Business strategy, trade secrets | Need-to-know |
| **PII / SENSITIVE** | Name, email, national ID, health data | Strict — encrypt, mask, access log |
| **RESTRICTED** | Payment card data (PCI), credentials | Highest controls — tokenise, no logging |

---

## Pros

- **Trust differentiator**: Systems that are demonstrably privacy-safe build customer and regulator trust.
- **Shift-left compliance**: Automated PII tagging in the data catalogue and column masking in the warehouse prevents accidental exposure.
- **GDPR right-to-erasure automation**: Automated deletion jobs prevent manual compliance chaos at scale.
- **Pseudonymisation enables analytics**: Tokenised data can be analysed at scale without exposing raw PII — ML models train on tokenised customer IDs.
- **Data product confidence**: Well-governed data with clear PII tags allows confident sharing across teams and third parties.

---

## Cons

- **Erasure propagation complexity**: A single right-to-erasure request may require deleting data from operational DB, data warehouse, search index, backups, CDN caches, and event streams.
- **Anonymisation at query time costs**: Column-level masking with dynamic data masking (Snowflake, BigQuery) adds query execution overhead.
- **Differential privacy accuracy trade-off**: Adding noise degrades analytical accuracy — must calibrate privacy budget (epsilon) vs analysis uses.
- **Tokenisation vault becomes critical**: If the tokenisation vault is unavailable, systems that need to de-tokenise PII for customer service are blocked.
- **Legacy system integration**: Retrofitting privacy controls to systems not designed for it (e.g., adding PII tagging to a 10-year-old monolith) is expensive.

---

## Design Diagram

```mermaid
flowchart LR
    subgraph Collection["Data Collection"]
        UI[Web/mobile<br/>consent capture]
        CONSENT[Consent store<br/>(purpose + timestamp)]
        API4[API layer<br/>PII validation at boundary]
    end

    subgraph Storage2["Storage (Privacy Controls)"]
        PG3[(PostgreSQL<br/>column encryption<br/>for PII)]
        VAULT2[Tokenisation Vault<br/>PII → token mapping]
        LAKE4[Data Lake<br/>pseudonymised<br/>(tokens only)]
    end

    subgraph Analytics["Analytics (Anonymised)"]
        MASK[Column masking<br/>Snowflake / BigQuery<br/>by role]
        AGG[Aggregated exports<br/>differential privacy noise]
    end

    subgraph Compliance["Compliance Automation"]
        SAR_SVC[SAR Service<br/>collect data across systems]
        ERASURE[Erasure Service<br/>coordinated delete]
        AUDIT3[Audit log<br/>every PII access event]
    end

    UI -->|consent recorded| CONSENT
    API4 -->|raw PII| VAULT2
    VAULT2 -->|token| LAKE4 & PG3
    LAKE4 --> MASK --> AGG
    ERASURE -->|delete token mapping| VAULT2
    ERASURE -->|delete by token| LAKE4 & PG3
    SAR_SVC -->|query by token| VAULT2 & LAKE4 & PG3
    PG3 -->|access event| AUDIT3
```

---

## Code Sample

### TypeScript — Tokenisation service: PII vault

```typescript
// src/privacy/tokenisation.ts
// Tokenise PII at ingestion; de-tokenise only for authorised operational use

import { createCipheriv, createDecipheriv, randomBytes } from 'crypto';
import { Pool }                                           from 'pg';

const pool = new Pool({ connectionString: process.env.PII_VAULT_DB_URL!, ssl: { rejectUnauthorized: true } });

// AES-256-GCM for encryption of the PII stored in the vault
const ALGORITHM    = 'aes-256-gcm';
const KEY          = Buffer.from(process.env.PII_VAULT_KEY!, 'hex');   // 32-byte hex key — from KMS

function encryptPII(plaintext: string): { ciphertext: string; iv: string; tag: string } {
  const iv     = randomBytes(12);   // 96-bit IV for GCM
  const cipher = createCipheriv(ALGORITHM, KEY, iv);
  const encrypted = Buffer.concat([cipher.update(plaintext, 'utf8'), cipher.final()]);
  return {
    ciphertext: encrypted.toString('base64'),
    iv:         iv.toString('base64'),
    tag:        cipher.getAuthTag().toString('base64'),
  };
}

function decryptPII(ciphertext: string, iv: string, tag: string): string {
  const decipher = createDecipheriv(ALGORITHM, KEY, Buffer.from(iv, 'base64'));
  decipher.setAuthTag(Buffer.from(tag, 'base64'));
  return Buffer.concat([
    decipher.update(Buffer.from(ciphertext, 'base64')),
    decipher.final(),
  ]).toString('utf8');
}

/**
 * Returns an opaque token for a PII value.
 * The same PII value always returns the same token (deterministic) — enables joins.
 */
export async function tokenise(piiType: string, piiValue: string): Promise<string> {
  const client = await pool.connect();
  try {
    // Check if already tokenised
    const existing = await client.query(
      'SELECT token FROM pii_vault WHERE pii_type = $1 AND pii_hash = digest($2, \'sha256\')',
      [piiType, piiValue],
    );
    if (existing.rows.length > 0) return existing.rows[0].token as string;

    // New PII: generate token + encrypt value
    const token             = randomBytes(16).toString('hex');
    const { ciphertext, iv, tag } = encryptPII(piiValue);

    await client.query(`
      INSERT INTO pii_vault (token, pii_type, pii_hash, ciphertext, iv, tag, created_at)
      VALUES ($1, $2, digest($3, 'sha256'), $4, $5, $6, NOW())
    `, [token, piiType, piiValue, ciphertext, iv, tag]);

    return token;
  } finally {
    client.release();
  }
}

/** Retrieve original PII from token — requires audit log entry */
export async function deTokenise(token: string, requestedBy: string, purpose: string): Promise<string | null> {
  const client = await pool.connect();
  try {
    const result = await client.query(
      'SELECT ciphertext, iv, tag FROM pii_vault WHERE token = $1',
      [token],
    );
    if (result.rows.length === 0) return null;

    // Mandatory audit log: every de-tokenisation is recorded
    await client.query(`
      INSERT INTO pii_access_audit (token, requested_by, purpose, accessed_at)
      VALUES ($1, $2, $3, NOW())
    `, [token, requestedBy, purpose]);

    const { ciphertext, iv, tag } = result.rows[0];
    return decryptPII(ciphertext, iv, tag);
  } finally {
    client.release();
  }
}
```

### TypeScript — Right-to-erasure (GDPR Art. 17) orchestration

```typescript
// src/privacy/erasure.ts
// Orchestrates GDPR right-to-erasure: deletes PII across all systems

interface ErasureTarget {
  system:  string;
  deleteF: () => Promise<void>;
}

export async function executeErasureRequest(customerId: string, requestId: string): Promise<void> {
  // First resolve customer's tokens from the vault — needed to delete from downstream
  const vaultClient    = await pool.connect();
  const tokenRows      = await vaultClient.query(
    "SELECT token FROM pii_vault WHERE pii_type = 'customer_id' AND pii_hash = digest($1, 'sha256')",
    [customerId],
  );
  vaultClient.release();
  const customerToken: string | null = tokenRows.rows[0]?.token ?? null;

  const targets: ErasureTarget[] = [
    {
      system:  'pii_vault',
      deleteF: async () => {
        // Delete encrypted PII — token becomes a dead reference
        await pool.query("DELETE FROM pii_vault WHERE pii_type LIKE 'customer_%' AND pii_hash = digest($1, 'sha256')", [customerId]);
      },
    },
    {
      system:  'operational_db',
      deleteF: async () => {
        // Pseudonymise rather than hard delete in operational DB — preserve referential integrity
        await pool.query(`
          UPDATE customers
          SET email = 'deleted-' || id || '@erased.local',
              full_name = 'Deleted User',
              phone = NULL,
              address = NULL,
              erased_at = NOW(),
              erasure_request_id = $2
          WHERE id = $1
        `, [customerId, requestId]);
      },
    },
    {
      system:  'search_index',
      deleteF: async () => {
        // Delete customer document from Elasticsearch
        await fetch(`${process.env.ES_URL}/customers/_doc/${customerToken ?? customerId}`, {
          method:  'DELETE',
          headers: { Authorization: `ApiKey ${process.env.ES_API_KEY}` },
        });
      },
    },
    {
      system:  'data_lake',
      deleteF: async () => {
        // Iceberg row-level delete (requires DELETE predicate support — Iceberg v2)
        // Executed via Spark job in production — simplified here
        console.log(`[erasure] Queueing Iceberg delete for token: ${customerToken}`);
      },
    },
  ];

  const results: { system: string; status: 'success' | 'failed'; error?: string }[] = [];

  for (const target of targets) {
    try {
      await target.deleteF();
      results.push({ system: target.system, status: 'success' });
    } catch (err) {
      results.push({ system: target.system, status: 'failed', error: String(err) });
    }
  }

  // Record erasure completion in audit log
  await pool.query(`
    INSERT INTO erasure_audit (request_id, customer_id, results, completed_at)
    VALUES ($1, $2, $3, NOW())
  `, [requestId, customerId, JSON.stringify(results)]);

  const failed = results.filter(r => r.status === 'failed');
  if (failed.length > 0) {
    throw new Error(`Erasure partially failed for systems: ${failed.map(f => f.system).join(', ')}`);
  }

  console.log(`Erasure complete for request ${requestId}`);
}
```
