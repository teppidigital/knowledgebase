# Replication

## Category

DDIA — Distributed Data (Chapter 5)

## Context

Replication means keeping copies of the same data on multiple nodes. The reasons are:
1. **Fault tolerance** — if one node dies, another can serve requests
2. **Latency** — serve users from a geographically closer replica
3. **Throughput** — scale read queries across multiple replicas

The hard problem in replication is handling **changes**: how does a write on one node get propagated to others consistently?

### Replication Strategies

| Strategy | How it works | Durability | Consistency | Complexity |
|---|---|---|---|---|
| **Single-leader (leader-follower)** | All writes go to leader; replicated to followers | High (leader acks after WAL write) | Eventual (followers may lag) | Low |
| **Multi-leader** | Multiple nodes accept writes; sync asynchronously | High | Complex (write conflicts possible) | High |
| **Leaderless (Dynamo-style)** | Any node accepts writes; quorum ensures durability | Configurable (w + r > n) | Configurable | High |

## Pros

- **Single-leader**: simple to reason about; no write conflicts; reads can be served from any follower
- **Multi-leader**: writes available in multiple datacenters; no single leader SPOF; useful for offline-capable clients
- **Leaderless**: no leader election required; tunable durability/consistency via quorum; resilient to node failures

## Cons

- **Single-leader**: leader is a write bottleneck; leader failure requires election (downtime window); replication lag creates stale reads
- **Multi-leader**: write conflicts must be resolved (last-write-wins is lossy; merge requires application logic); very complex to implement correctly
- **Leaderless**: read repair and anti-entropy add complexity; no total write ordering without extra protocol; sloppy quorums can return stale data even with w+r > n

## Design Diagram

```mermaid
flowchart LR
    subgraph Single-Leader Replication
        CLIENT1[Client] -- Write --> LEADER[Leader\nAll writes here]
        LEADER -- Replication log --> F1[Follower 1]
        LEADER -- Replication log --> F2[Follower 2]
        CLIENT1 -- Read --> F1
        CLIENT1 -- Read --> F2
    end

    subgraph Leaderless — Quorum Write
        CLIENT2[Client] -- Write w=2 of 3 --> N1[Node 1 ✅]
        CLIENT2 -- Write --> N2[Node 2 ✅]
        CLIENT2 -- Write --> N3[Node 3 ❌ down]
        CLIENT2 -- Read r=2 of 3 --> N1
        CLIENT2 -- Read --> N2
    end
```

## Code Sample

### Replication lag — read-your-own-writes consistency

```typescript
// Problem: user writes to leader; immediately reads from follower; sees stale data
// Solution: after a write, read from leader (or track lag threshold)

import { Pool } from 'pg';

// Two DB connections: leader (writes) + replica (reads)
const leader = new Pool({ connectionString: process.env.DATABASE_LEADER_URL });
const replica = new Pool({ connectionString: process.env.DATABASE_REPLICA_URL });

export class UserRepository {
  // Read-your-own-writes: if user just wrote something, read from leader
  async findById(userId: string, opts: { readAfterWrite?: boolean } = {}): Promise<User | null> {
    const db = opts.readAfterWrite ? leader : replica;
    const { rows } = await db.query('SELECT * FROM users WHERE id = $1', [userId]);
    return rows[0] ?? null;
  }

  async updateProfile(userId: string, updates: Partial<User>): Promise<void> {
    await leader.query(
      'UPDATE users SET name = $1, email = $2, updated_at = NOW() WHERE id = $3',
      [updates.name, updates.email, userId]
    );
    // Signal that subsequent reads for this user should go to leader until lag catches up
    // In practice: set a cookie/session flag; use Redis to track last-write timestamp per user
  }
}

// Monotonic reads: ensure user doesn't see time go backward across replica reads
// Store a replication position in the session; route to leader if replica is behind
export async function getWithMonotonicRead(
  userId: string,
  lastKnownLsn?: string
): Promise<{ user: User | null; currentLsn: string }> {
  const { rows: [{ current_lsn }] } = await replica.query(
    "SELECT pg_last_wal_replay_lsn()::text AS current_lsn"
  );

  if (lastKnownLsn && current_lsn < lastKnownLsn) {
    // Replica is behind what this user last saw — read from leader
    const { rows } = await leader.query('SELECT * FROM users WHERE id = $1', [userId]);
    return { user: rows[0] ?? null, currentLsn: lastKnownLsn };
  }

  const { rows } = await replica.query('SELECT * FROM users WHERE id = $1', [userId]);
  return { user: rows[0] ?? null, currentLsn: current_lsn };
}
```

### Multi-leader conflict resolution — last-write-wins with version vectors

```typescript
// Version vector (vector clock) tracks causal history per replica
type VersionVector = Record<string, number>; // nodeId → sequence number

interface VersionedValue<T> {
  value: T;
  vector: VersionVector;
  nodeId: string;
  writtenAt: Date;
}

function happensBefore(a: VersionVector, b: VersionVector): boolean {
  // a happened before b if every component of a <= b AND at least one is <
  const allLessOrEqual = Object.entries(a).every(([node, seq]) => (b[node] ?? 0) >= seq);
  const atLeastOneLess = Object.entries(a).some(([node, seq]) => (b[node] ?? 0) > seq);
  return allLessOrEqual && atLeastOneLess;
}

function resolveConflict<T>(current: VersionedValue<T>, incoming: VersionedValue<T>): VersionedValue<T> {
  if (happensBefore(current.vector, incoming.vector)) return incoming;
  if (happensBefore(incoming.vector, current.vector)) return current;
  // Concurrent writes — neither happened before the other
  // Last-Write-Wins as fallback (lossy but simple)
  console.warn('Concurrent write conflict — applying LWW');
  return current.writtenAt > incoming.writtenAt ? current : incoming;
}
```

### Quorum reads and writes — leaderless (Dynamo/Cassandra style)

```typescript
// n=3 nodes; w=2 (write quorum); r=2 (read quorum)
// Guarantee: at least one node in the read set overlaps with the write set
// → Read always returns at least one up-to-date value (then resolve by version)

const N = 3, W = 2, R = 2;

interface QuorumWrite { key: string; value: string; version: number }

async function quorumWrite(key: string, value: string, nodes: StorageNode[]): Promise<boolean> {
  const version = Date.now();
  const results = await Promise.allSettled(
    nodes.map(node => node.write({ key, value, version }))
  );
  const successes = results.filter(r => r.status === 'fulfilled').length;
  return successes >= W; // durable if at least W nodes acknowledged
}

async function quorumRead(key: string, nodes: StorageNode[]): Promise<string | null> {
  const results = await Promise.allSettled(nodes.map(node => node.read(key)));
  const values = results
    .filter(r => r.status === 'fulfilled')
    .map(r => (r as PromiseFulfilledResult<{ value: string; version: number } | null>).value)
    .filter(Boolean) as { value: string; version: number }[];

  if (values.length < R) return null; // not enough responses for quorum
  // Return highest version (read repair: also write highest version back to lagging nodes)
  return values.sort((a, b) => b.version - a.version)[0].value;
}
```

## Key Patterns

### Replication Lag Anomalies

| Anomaly | What happens | Solution |
|---|---|---|
| **Stale reads** | Read follower; see data before latest write | Read-your-own-writes: route user's own data reads to leader |
| **Moving backward in time** | Two reads return data; second is older than first | Monotonic reads: always read from the same replica; advance the "known position" |
| **Prefix causality violation** | User sees answer before the question | Consistent prefix reads: within a partition, reads see writes in order |

### Leader Election

When a leader fails:
1. Followers detect timeout (e.g., lease expiry)
2. Election begins — typically the most up-to-date follower wins
3. New leader is announced; old clients redirect

**Split-brain risk**: two nodes both believe they are the leader. Solutions:
- STONITH (Shoot The Other Node In The Head) — fencing
- Majority quorum required for leader — a minority can't elect a new leader
- Lease-based leadership — leader must renew; if renewal fails, leadership expires

### Statement-Based vs Row-Based Replication

| Type | How | Pros | Cons |
|---|---|---|---|
| **Statement-based** | Replicate SQL statement | Compact log | Non-deterministic functions (NOW(), RAND()) produce different results on replica |
| **Row-based (WAL shipping)** | Replicate actual row changes | Deterministic; works for all queries | Larger log; less human-readable |
| **Logical replication** | Row changes in a structured format (Postgres logical replication) | Selectable by table; cross-version; useful for CDC | More setup |

## Related Patterns

- [09 — Consistency and Consensus](./09-consistency-consensus.md) — Linearizability requires single-node semantics
- [05 — Replication (CA)](../continuous-architecture/05-architecture-agile.md) — Replication lag is an architecture concern
- [14 — Distributed Transactions](./14-distributed-transactions.md) — Transactions across replica sets
- [12 — Derived Data Systems](./12-derived-data-systems.md) — Replication log as event stream (CDC)
