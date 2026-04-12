# Consistency and Consensus

## Category

DDIA — Distributed Data (Chapter 9)

## Context

Chapter 9 is about the theoretical guarantees that distributed systems can provide. It answers: "What does it mean for a distributed system to behave correctly?"

### Linearizability

Linearizability is the strongest single-object consistency guarantee. It makes a distributed system appear as if there is a single copy of the data, and all operations are atomic and instantaneous.

**Key properties:**
- Once a write completes, all future reads must return that value (or a newer one)
- Operations appear to take effect at a single point in time between their invocation and completion
- It is a **recency guarantee** (real-time ordering)

**Linearizability vs Serializability:** These are completely different properties, often confused:

| Property | About | Scope | Example |
|---|---|---|---|
| **Serializability** | Transactions appear to execute serially | Multiple objects, multiple operations | ACID isolation |
| **Linearizability** | Operations appear instantaneous on a single object | Single object, single operation | Register reads/writes |
| **Strict Serializability** | Serializable + Linearizable | Both | Google Spanner, FoundationDB |

### CAP Theorem

CAP theorem states that in the presence of a network **Partition**, you must choose between **Consistency** (linearizability) and **Availability**.

| Property | Meaning | Real-world implication |
|---|---|---|
| **Consistency (C)** | Linearizability — every read returns the most recent write | During partition, unavailable nodes must refuse requests |
| **Availability (A)** | Every request receives a response (not an error) | During partition, nodes respond with potentially stale data |
| **Partition Tolerance (P)** | System continues operating when network splits | Not optional for a distributed system — partitions will occur |

**The real choice is CA vs CP vs AP:**

| Choice | What is sacrificed | Examples |
|---|---|---|
| **CP** | Availability during partitions | ZooKeeper, etcd, HBase, Spanner |
| **AP** | Consistency (may return stale data) | Cassandra, CouchDB, DynamoDB (eventually consistent) |
| **CA** | Partition tolerance — not feasible at scale | Single-node RDBMS (not distributed) |

> "CAP theorem is often misunderstood. It doesn't say you can only have 2 of 3. It says: during a network partition, you must choose between C and A." — DDIA

### Ordering and Causality

Causal consistency (weaker than linearizability) ensures that causally related operations are seen in order: if event A caused event B, then everyone who sees B must have first seen A.

- **Causally consistent** systems can be available during partitions (unlike linearizable systems)
- **Lamport timestamps** provide a total order consistent with causality
- **Vector clocks** capture causal dependencies precisely

### Total Order Broadcast

Total order broadcast ensures all nodes deliver messages in the same order. It is the foundation for:
- **Consensus** (all nodes agree on the same sequence of commands)
- **Replicated state machines** (apply the same commands in the same order → same state)
- **Fencing tokens** (monotonically increasing token = position in the total order)

### Consensus Algorithms: Raft

Raft is a consensus algorithm designed for understandability. It elects a **leader** that handles all writes. ZooKeeper and etcd implement Raft (or a variant).

**Raft phases:**
1. **Leader election**: if no heartbeat for election timeout → start election → first to get majority wins
2. **Log replication**: leader appends entry to its log, replicates to followers, commits when majority acknowledge
3. **Safety**: a leader can only be elected if its log is at least as up-to-date as any majority

## Pros

- Linearizability provides the strongest correctness guarantee — simplifies application reasoning
- Consensus algorithms (Raft) are well understood and implemented reliably in etcd, ZooKeeper, CockroachDB
- Total order broadcast enables building any strongly consistent primitive
- Causal consistency is achievable without the cost of linearizability

## Cons

- Linearizability has significant latency cost — requires round trips to a quorum even for reads
- Consensus requires a quorum — if < quorum nodes available, system is unavailable (CP)
- Leader-based systems have leader as a bottleneck
- In practice, CAP theorem is less impactful than PACELC theorem: even without partitions, there are latency vs consistency trade-offs

## Design Diagram

```mermaid
flowchart TD
    subgraph Raft — Leader Election
        S1[Server 1<br/>Follower]
        S2[Server 2<br/>Candidate → Leader]
        S3[Server 3<br/>Follower]
        S4[Server 4<br/>Follower]
        S5[Server 5<br/>Follower]

        S2 -- RequestVote --> S1
        S2 -- RequestVote --> S3
        S1 -- Vote granted --> S2
        S3 -- Vote granted --> S2
        S2 -- Now leader<br/>AppendEntries heartbeat --> S1
        S2 -- Now leader<br/>AppendEntries heartbeat --> S3
    end

    subgraph Raft — Log Replication
        L[Leader]
        F1[Follower 1]
        F2[Follower 2]
        F3[Follower 3]
        Client2[Client] --> L
        L -- AppendEntries<br/>entry cmd=SET x=5 --> F1
        L -- AppendEntries<br/>entry cmd=SET x=5 --> F2
        F1 -- ACK --> L
        F2 -- ACK --> L
        L -- Commit<br/>majority=2 of 3 --> OK[Command committed<br/>Respond to client]
    end
```

## Code Sample

### Distributed lock via etcd (ZooKeeper-like semantics)

```typescript
// Simplified etcd-backed distributed lock using the Lease API
// In production: use etcd3 npm package

interface DistributedLock {
  acquire(key: string, ttlSeconds: number): Promise<{ token: string; granted: boolean }>;
  release(key: string, token: string): Promise<void>;
  keepAlive(token: string): Promise<void>;
}

// Simplified implementation showing the pattern
class EtcdDistributedLock implements DistributedLock {
  private readonly etcdUrl: string;

  constructor(etcdUrl: string) {
    this.etcdUrl = etcdUrl;
  }

  async acquire(key: string, ttlSeconds: number): Promise<{ token: string; granted: boolean }> {
    // 1. Create a lease (TTL-bound resource)
    const leaseResponse = await fetch(`${this.etcdUrl}/v3/lease/grant`, {
      method: 'POST',
      body: JSON.stringify({ TTL: ttlSeconds }),
      headers: { 'Content-Type': 'application/json' }
    }).then(r => r.json());

    const leaseId = leaseResponse.ID;

    // 2. Attempt to create the key only if it doesn't exist (compare-and-swap)
    const txnResponse = await fetch(`${this.etcdUrl}/v3/kv/txn`, {
      method: 'POST',
      body: JSON.stringify({
        compare: [{ key: btoa(key), target: 'VERSION', version: 0 }], // key must not exist
        success: [{ requestPut: { key: btoa(key), value: btoa(leaseId), lease: leaseId } }],
        failure: []
      }),
      headers: { 'Content-Type': 'application/json' }
    }).then(r => r.json());

    return {
      token: leaseId,
      granted: txnResponse.succeeded
    };
  }

  async release(key: string, token: string): Promise<void> {
    // Revoke the lease — key is automatically deleted
    await fetch(`${this.etcdUrl}/v3/lease/revoke`, {
      method: 'POST',
      body: JSON.stringify({ ID: token }),
      headers: { 'Content-Type': 'application/json' }
    });
  }

  async keepAlive(token: string): Promise<void> {
    await fetch(`${this.etcdUrl}/v3/lease/keepalive`, {
      method: 'POST',
      body: JSON.stringify({ ID: token }),
      headers: { 'Content-Type': 'application/json' }
    });
  }
}

// Usage pattern: acquire → do work → release; with fencing
async function withDistributedLock<T>(
  lock: DistributedLock,
  key: string,
  fn: (token: string) => Promise<T>
): Promise<T> {
  const { granted, token } = await lock.acquire(key, 30);
  if (!granted) throw new Error(`Could not acquire lock: ${key}`);

  // Keep alive in the background (every 10s, TTL is 30s)
  const keepAliveInterval = setInterval(() => lock.keepAlive(token), 10_000);

  try {
    return await fn(token); // pass fencing token to storage writes
  } finally {
    clearInterval(keepAliveInterval);
    await lock.release(key, token);
  }
}
```

### Lamport timestamps for causal ordering

```typescript
// Lamport clock: track causal order across nodes without coordination
// Rule: on send, max(local, remote) + 1; on receive, update local

class LamportClock {
  private time = 0;

  tick(): number {
    return ++this.time;
  }

  // Call on receive: update local clock to be > max of local and received
  update(received: number): number {
    this.time = Math.max(this.time, received) + 1;
    return this.time;
  }

  now(): number {
    return this.time;
  }
}

interface Event {
  id: string;
  timestamp: number;
  nodeId: string;
  payload: unknown;
}

// Total ordering: if timestamps equal, break tie by nodeId
function compareEvents(a: Event, b: Event): number {
  if (a.timestamp !== b.timestamp) return a.timestamp - b.timestamp;
  return a.nodeId.localeCompare(b.nodeId); // tie-break: deterministic
}

// Example: two nodes create events
const node1Clock = new LamportClock();
const node2Clock = new LamportClock();

const e1: Event = { id: '1', timestamp: node1Clock.tick(), nodeId: 'node-1', payload: 'Create user' };
const e2: Event = { id: '2', timestamp: node2Clock.tick(), nodeId: 'node-2', payload: 'Create order' };

// Node 2 receives a message from Node 1 (e1's timestamp is 1)
node2Clock.update(e1.timestamp);
const e3: Event = { id: '3', timestamp: node2Clock.tick(), nodeId: 'node-2', payload: 'Post-create hook' };
// e3.timestamp is guaranteed > e1.timestamp — causal order preserved
```

## Key Patterns

### Choosing Consistency Level

| Use case | Consistency model | Reason |
|---|---|---|
| Leader election, distributed locks | Linearizable (CP) | Must agree on single leader; stale read → split brain |
| User profile reads, product catalog | Causal / eventual | Staleness acceptable; higher availability more valuable |
| Financial ledgers, bank balances | Serializable (strict) | Prevents write skew; all anomalies must be ruled out |
| Order of messages in a chat | Causal consistency | Messages from one user must appear in order |
| Shopping cart (availability > consistency) | Eventual | Prefer cart added over cart error; merge on conflict |

### ZooKeeper / etcd Use Cases

```
✅ Distributed locks (leader election, single-writer guarantees)
✅ Service discovery (who is the current primary?)
✅ Configuration management (watch for changes, CP guarantee)
✅ Fencing token generation (monotonically increasing ID)
❌ General-purpose data storage (not designed for high throughput)
❌ Large values (designed for small metadata, not payloads)
```

### PACELC Trade-offs (Beyond CAP)

CAP only addresses partition behaviour. PACELC adds: even without a partition (P), you must choose between Latency (L) and Consistency (C).

| System | Under partition | Without partition |
|---|---|---|
| DynamoDB | AP | EL (low latency, eventual) |
| Cassandra | AP | EL (tunable) |
| CockroachDB | CP | EC (consistent but higher latency) |
| Spanner | CP | EC (TrueTime; consistent reads have latency) |
| MySQL (single-node) | CA | EC (consistent; acceptable latency) |

## Related Patterns

- [08 — Distributed Systems Trouble](./08-distributed-systems-trouble.md) — Network failures that consensus must handle
- [05 — Replication](./05-replication.md) — How replication relates to consistency
- [07 — Transactions](./07-transactions.md) — Serializability vs linearizability
- [14 — Distributed Transactions](./14-distributed-transactions.md) — 2PC and consensus in cross-service transactions
