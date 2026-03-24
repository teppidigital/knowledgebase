# Quorum-based Replication

## Category
Distributed Systems, Consistency, Availability, Data Replication

## Context

Quorum-based replication ensures **data consistency across multiple replicas** by requiring that a minimum number of nodes (a quorum) agree on a read or write operation before it succeeds. The key invariant is:

> **W + R > N**

Where:
- **N** = total number of replicas
- **W** = write quorum (nodes that must acknowledge a write)
- **R** = read quorum (nodes that must respond to a read)

This guarantees that any successful read will see at least one replica that participated in the last successful write.

Used in: Amazon Dynamo (W=1, R=1, N=3 for availability; W=3, R=2 for consistency), Apache Cassandra, etcd, ZooKeeper, Raft.

---

## Pros

- **Tunable consistency/availability trade-off**: Adjust W and R to balance consistency and availability.
- **Fault tolerance**: With N=3 and W=R=2, the system tolerates 1 node failure at full consistency.
- **No single point of failure**: Any quorum of nodes can serve reads or writes.
- **High write throughput** (at low W): Accept writes when just 1 node acknowledges.
- **Strong consistency** (at W+R>N): Reads always see the latest write.

---

## Cons

- **Availability vs consistency trade-off**: Strict quorum may cause failures when network partitions isolate nodes (CAP theorem).
- **Latency**: Must wait for W or R nodes to respond — slowest node in quorum dominates latency.
- **Complexity**: Conflict resolution is needed when replicas diverge (vector clocks, last-write-wins).
- **Read repair overhead**: When a read finds stale replicas, background repair writes are needed.
- **Sloppy quorum risk**: Accepting writes from non-primary replicas can temporarily hide data.

---

## Design Diagram

```mermaid
graph TD
    Client["Client"]

    subgraph Replication Group N=3
        R1["Replica 1<br/>(Primary)"]
        R2["Replica 2"]
        R3["Replica 3"]
    end

    subgraph Write W=2
        Client -->|"Write: key=Alice"| R1
        R1 -->|"Replicate"| R2
        R1 -->|"Replicate"| R3
        R2 -->|"ACK (W quorum met)"| Client
        Note1["R3 may ACK later<br/>(async)"]
    end

    subgraph Read R=2
        Client2["Client"] -->|"Read: key"| R1
        Client2 --> R2
        R1 -->|"v=5"| Client2
        R2 -->|"v=5"| Client2
        Note2["R3 (stale v=4) not<br/>part of quorum"]
    end
```

---

## Code Sample

### Quorum Write and Read (TypeScript)

```typescript
// quorum/quorum-manager.ts
import axios from 'axios';

interface ReplicaValue {
  value: unknown;
  version: number;  // Timestamp or vector clock
}

interface Replica {
  id: string;
  url: string;
}

export class QuorumManager {
  constructor(
    private readonly replicas: Replica[],
    private readonly writeQuorum: number,   // W
    private readonly readQuorum: number     // R
  ) {
    if (writeQuorum + readQuorum <= replicas.length) {
      throw new Error(`Quorum condition W+R>N not met: ${writeQuorum}+${readQuorum}<=${replicas.length}`);
    }
  }

  // Write: send to all replicas, wait for W acknowledgments
  async write(key: string, value: unknown): Promise<void> {
    const version = Date.now();
    const payload = { key, value, version };

    const results = await this.raceAll(
      this.replicas.map(r => this.writeToReplica(r, payload))
    );

    const acks = results.filter(r => r.success).length;
    if (acks < this.writeQuorum) {
      throw new Error(`Write quorum not met: ${acks}/${this.writeQuorum} replicas ACKed`);
    }

    console.log(`Write quorum met: ${acks}/${this.replicas.length} replicas`);
  }

  // Read: query all replicas, wait for R responses, return highest version
  async read(key: string): Promise<unknown> {
    const results = await this.raceAll(
      this.replicas.map(r => this.readFromReplica(r, key))
    );

    const responses = results
      .filter(r => r.success && r.data !== null)
      .sort((a, b) => b.data!.version - a.data!.version); // Newest version first

    if (responses.length < this.readQuorum) {
      throw new Error(`Read quorum not met: ${responses.length}/${this.readQuorum} replicas responded`);
    }

    const latest = responses[0].data!;

    // Read repair: update stale replicas asynchronously
    const staleReplicas = results.filter(r => r.success && r.data?.version < latest.version);
    this.readRepair(key, latest, staleReplicas.map(r => r.replica));

    return latest.value;
  }

  private async writeToReplica(replica: Replica, payload: object) {
    try {
      await axios.post(`${replica.url}/data`, payload, { timeout: 2000 });
      return { success: true, replica };
    } catch {
      return { success: false, replica };
    }
  }

  private async readFromReplica(replica: Replica, key: string) {
    try {
      const { data } = await axios.get<ReplicaValue>(`${replica.url}/data/${key}`, { timeout: 2000 });
      return { success: true, replica, data };
    } catch {
      return { success: false, replica, data: null };
    }
  }

  private async readRepair(key: string, latest: ReplicaValue, staleReplicas: Replica[]): Promise<void> {
    for (const replica of staleReplicas) {
      console.log(`[ReadRepair] Updating stale replica: ${replica.id}`);
      this.writeToReplica(replica, { key, ...latest }).catch(console.error);
    }
  }

  // Collect first N results regardless of success/failure
  private async raceAll<T>(promises: Promise<T>[]): Promise<T[]> {
    return Promise.all(promises.map(p => p.catch(err => ({ success: false, error: err.message } as any))));
  }
}
```

### Usage — Different Quorum Configurations

```typescript
// N=3 replicas
const replicas: Replica[] = [
  { id: 'replica-1', url: 'http://replica-1:3000' },
  { id: 'replica-2', url: 'http://replica-2:3000' },
  { id: 'replica-3', url: 'http://replica-3:3000' },
];

// Strong consistency: W=2, R=2 — tolerates 1 failure
const strongConsistency = new QuorumManager(replicas, 2, 2);

// High availability: W=1, R=2 — fast writes, consistent reads
const highAvailability = new QuorumManager(replicas, 1, 2);

// High write throughput: W=1, R=3 — eventual consistency
// const eventual = new QuorumManager(replicas, 1, 3); // W+R=4 > N=3

await strongConsistency.write('user:1', { name: 'Alice', age: 30 });
const value = await strongConsistency.read('user:1');
console.log('Read:', value);
```

### Simple Replica Server (Express)

```typescript
// replica/server.ts
import express from 'express';

const app = express();
app.use(express.json());

// In-memory store (use persistent DB in production)
const store = new Map<string, { value: unknown; version: number }>();

app.post('/data', (req, res) => {
  const { key, value, version } = req.body;
  const current = store.get(key);

  if (!current || version >= current.version) {
    store.set(key, { value, version });
    res.status(200).json({ ack: true, version });
  } else {
    res.status(409).json({ ack: false, reason: 'Stale write rejected', currentVersion: current.version });
  }
});

app.get('/data/:key', (req, res) => {
  const entry = store.get(req.params.key);
  if (!entry) return res.status(404).json(null);
  res.json(entry);
});

app.listen(3000, () => console.log(`Replica running on :3000`));
```
