# Transactions

## Category

DDIA — Distributed Data (Chapter 7)

## Context

A transaction is a way of grouping several reads and writes together into a logical unit: either all succeed (commit) or all fail (abort+retry). Transactions simplify error handling by removing the need to handle partial failures manually.

ACID is the traditional guarantee:
- **Atomicity**: the transaction commits or aborts completely — no partial state visible
- **Consistency**: the DB is in a valid state before and after (app-defined invariants hold)
- **Isolation**: concurrent transactions appear as if they run serially
- **Durability**: committed data survives crashes

> Note: ACID "consistency" is different from the "C" in CAP theorem. In ACID, consistency is an application-level property; in CAP, consistency refers to linearizability.

### Isolation Levels

Isolation is the most nuanced property. Full isolation (serializability) has a significant performance cost. Most databases default to a weaker isolation level that allows certain anomalies.

| Isolation Level | Dirty Read | Non-repeatable Read | Phantom Read | Write Skew | Performance |
|---|---|---|---|---|---|
| **Read Uncommitted** | ✅ allowed | ✅ allowed | ✅ allowed | ✅ allowed | Fastest |
| **Read Committed** | ❌ prevented | ✅ allowed | ✅ allowed | ✅ allowed | Fast |
| **Repeatable Read** | ❌ prevented | ❌ prevented | ✅ allowed | ✅ allowed | Medium |
| **Snapshot Isolation** | ❌ prevented | ❌ prevented | ❌ prevented | ✅ allowed | Medium |
| **Serializable** | ❌ prevented | ❌ prevented | ❌ prevented | ❌ prevented | Slowest |

**Most databases default to Read Committed** (PostgreSQL, Oracle, SQL Server). MySQL InnoDB defaults to Repeatable Read.

### Anomalies in Detail

| Anomaly | What happens | Example |
|---|---|---|
| **Dirty read** | Reading data from an uncommitted transaction | Read a transferred balance before the transfer completes |
| **Non-repeatable read** | Same row returns different values within one transaction | Read account balance twice; a concurrent update changes it |
| **Phantom read** | A re-executed range query returns different rows | Count rows in a range; a concurrent insert adds a row |
| **Write skew** | Two transactions read the same data, each decide to write based on what they read, creating a conflict | Both doctors check "is someone on call?", both go off-call — now no one is on call |
| **Lost update** | Two concurrent read-modify-write cycles; one update is lost | Two users increment a counter; both read 0; both write 1 |

## Pros

- Transactions dramatically simplify application error handling — no need to track partial state
- Atomicity prevents partial writes from being visible to other transactions
- Isolation allows reasoning about one transaction in isolation, ignoring concurrent effects
- Read Committed (the common default) prevents dirty reads with minimal overhead

## Cons

- Serializable isolation has significant performance cost (reduced concurrency)
- Long-running transactions hold locks, blocking other transactions
- Distributed transactions (spanning multiple nodes or services) are extremely complex
- No transaction system prevents all bugs — it moves the burden from infrastructure to application logic

## Design Diagram

```mermaid
flowchart TD
    subgraph Isolation Anomaly Timeline
        T1[Transaction 1\nReads X=100\nDecides to write X=150]
        T2[Transaction 2\nReads X=100\nDecides to write X=120]
        T1 -- commits X=150 -->DB[(Database)]
        T2 -- commits X=120 -->DB
        DB -- LOST UPDATE: final value X=120\nT1's write is lost -->RESULT[Wrong state]
    end

    subgraph Serializable Snapshot Isolation — SSI
        R1[T1 Reads snapshot\nat start time]
        W1[T1 Writes]
        CHECK{At commit:\nDid any read premise change?}
        R1 --> W1 --> CHECK
        CHECK -- Yes → Abort + retry --> R1
        CHECK -- No → Commit --> OK[Safe commit]
    end
```

## Code Sample

### Isolation levels — PostgreSQL explicit setting

```typescript
import { Pool, PoolClient } from 'pg';
const db = new Pool({ connectionString: process.env.DATABASE_URL });

// Lost update prevention — use SELECT FOR UPDATE (pessimistic locking)
async function incrementCounter(counterId: string): Promise<void> {
  const client = await db.connect();
  try {
    await client.query('BEGIN');
    // Lock the row — concurrent transactions must wait
    const { rows } = await client.query(
      'SELECT value FROM counters WHERE id = $1 FOR UPDATE',
      [counterId]
    );
    const newValue = rows[0].value + 1;
    await client.query('UPDATE counters SET value = $1 WHERE id = $2', [newValue, counterId]);
    await client.query('COMMIT');
  } catch (e) {
    await client.query('ROLLBACK');
    throw e;
  } finally {
    client.release();
  }
}

// Alternatively — atomic update (database handles the increment atomically)
async function atomicIncrement(counterId: string): Promise<number> {
  const { rows } = await db.query(
    'UPDATE counters SET value = value + 1 WHERE id = $1 RETURNING value',
    [counterId]
  );
  return rows[0].value;
}
```

### Write skew prevention — serializable isolation

```typescript
// Classic write skew: on-call doctors
// Both doctors check "is at least one doctor on call?"
// Both see yes; both go off call; now no one is on call

// Solution 1: Serializable isolation level
async function goOffCall(doctorId: string, db: Pool): Promise<void> {
  const client = await db.connect();
  try {
    await client.query('SET TRANSACTION ISOLATION LEVEL SERIALIZABLE');
    await client.query('BEGIN');

    const { rows } = await client.query(
      "SELECT COUNT(*) AS on_call FROM doctors WHERE on_call = true AND id != $1",
      [doctorId]
    );
    if (parseInt(rows[0].on_call) === 0) {
      await client.query('ROLLBACK');
      throw new Error('At least one doctor must be on call');
    }

    await client.query('UPDATE doctors SET on_call = false WHERE id = $1', [doctorId]);
    await client.query('COMMIT');
    // With SERIALIZABLE: if two transactions run concurrently, one will abort and retry
  } catch (e) {
    await client.query('ROLLBACK');
    throw e;
  } finally {
    client.release();
  }
}

// Solution 2: Materialise the conflict — create a lock row that both transactions must touch
async function goOffCallWithMaterialisedConflict(doctorId: string, db: Pool): Promise<void> {
  const client = await db.connect();
  try {
    await client.query('BEGIN');
    // Both transactions take a lock on the "on_call_lock" row — only one can proceed
    await client.query("SELECT pg_advisory_xact_lock(1234)"); // explicit advisory lock
    const { rows } = await client.query(
      "SELECT COUNT(*) AS on_call FROM doctors WHERE on_call = true"
    );
    if (parseInt(rows[0].on_call) <= 1) {
      throw new Error('At least one doctor must remain on call');
    }
    await client.query('UPDATE doctors SET on_call = false WHERE id = $1', [doctorId]);
    await client.query('COMMIT');
  } catch (e) {
    await client.query('ROLLBACK');
    throw e;
  } finally {
    client.release();
  }
}
```

### Idempotent retry with transaction

```typescript
// Retry on serialisation failure (error code 40001 in PostgreSQL)
async function withRetry<T>(
  fn: (client: PoolClient) => Promise<T>,
  maxRetries = 3
): Promise<T> {
  const client = await db.connect();
  let attempt = 0;
  while (true) {
    try {
      await client.query('BEGIN');
      const result = await fn(client);
      await client.query('COMMIT');
      return result;
    } catch (e: any) {
      await client.query('ROLLBACK');
      // 40001 = serialization_failure; 40P01 = deadlock_detected
      if ((e.code === '40001' || e.code === '40P01') && attempt < maxRetries) {
        attempt++;
        const delay = Math.min(100 * 2 ** attempt, 2000); // exponential backoff
        await new Promise(r => setTimeout(r, delay));
        continue;
      }
      throw e;
    } finally {
      if (attempt >= maxRetries) client.release();
    }
  }
}
```

## Key Patterns

### Locking Strategies

| Strategy | Mechanism | Best for |
|---|---|---|
| **Two-Phase Locking (2PL)** | Acquire all locks before any release; release only at commit | Strong isolation; high contention → deadlocks possible |
| **Optimistic concurrency** | No locks; check for conflicts at commit; abort if detected | Low contention; read-heavy workloads; SSI uses this |
| **Pessimistic locking** | `SELECT FOR UPDATE`; lock row immediately | High contention; long read-modify-write cycles |
| **Advisory locks** | Application-level locks via `pg_advisory_lock` | Non-row-level locks; custom lock granularity |

### When to Use Each Isolation Level

| Use case | Recommended level |
|---|---|
| General read-heavy: product catalog, user profiles | Read Committed (default) |
| Reporting queries that must see a consistent snapshot | Snapshot Isolation (Repeatable Read in PG) |
| Financial transactions: balances, inventory, on-call schedules | Serializable |
| Events and audit logs (append-only) | Read Committed (no concurrent updates) |

### Transaction Checklist

```
[ ] Transactions are as short as possible (hold locks minimally)
[ ] No external HTTP calls inside a transaction (long-running + can't roll back)
[ ] Retry logic handles serialisation failures (40001) and deadlocks (40P01)
[ ] SELECT FOR UPDATE used where lost update prevention is needed
[ ] ISOLATION LEVEL SERIALIZABLE used for write skew scenarios
[ ] Outbox pattern used for transactional messaging (DB write + message in one transaction)
```

## Related Patterns

- [14 — Distributed Transactions](./14-distributed-transactions.md) — Transactions across multiple nodes/services
- [07 — Storage and Retrieval](./03-storage-retrieval.md) — How transactions interact with storage engines
- [09 — Consistency and Consensus](./09-consistency-consensus.md) — Serializability and linearizability relationship
- [12 — Derived Data Systems](./12-derived-data-systems.md) — Outbox pattern for transactional messaging
