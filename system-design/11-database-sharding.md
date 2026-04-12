# Database Sharding

## Category
Scalability, Performance, Data Management

## Context

Database sharding is a horizontal scaling technique that partitions a large database into smaller, faster, more manageable pieces called **shards**. Each shard is a separate database that holds a subset of the data. A shard key (e.g., user ID, tenant ID, geographic region) determines which shard stores a given record.

Sharding is used when a single database instance can no longer handle the volume of data or query throughput required.

---

## Pros

- **Horizontal scalability**: Distribute data and load across many database servers.
- **Improved performance**: Smaller datasets per shard mean faster queries.
- **Fault isolation**: A failure in one shard does not affect data in other shards.
- **Geographic distribution**: Shards can be placed in different regions for latency optimization.
- **Cost efficiency**: Use many commodity servers instead of one expensive machine.

---

## Cons

- **Increased complexity**: Application must route queries to the correct shard.
- **Cross-shard queries**: Joins or aggregations across shards are difficult and expensive.
- **Rebalancing**: Adding new shards requires redistributing data, which is complex and disruptive.
- **Transactions**: Cross-shard ACID transactions are very hard to implement.
- **Hotspots**: Poor shard key choice can cause uneven load distribution.
- **Schema changes**: Must be applied to all shards consistently.

---

## Design Diagram

```mermaid
graph TD
    App["Application Layer"]
    SR["Shard Router<br/>(Routing Logic)"]

    subgraph "Shard 0 (userId mod 3 = 0)"
        DB0[("Database Shard 0<br/>Users: 0, 3, 6, 9...")]
    end

    subgraph "Shard 1 (userId mod 3 = 1)"
        DB1[("Database Shard 1<br/>Users: 1, 4, 7, 10...")]
    end

    subgraph "Shard 2 (userId mod 3 = 2)"
        DB2[("Database Shard 2<br/>Users: 2, 5, 8, 11...")]
    end

    App --> SR
    SR -->|"userId % 3 = 0"| DB0
    SR -->|"userId % 3 = 1"| DB1
    SR -->|"userId % 3 = 2"| DB2
```

---

## Code Sample

### Shard Router (Node.js)

```typescript
// sharding/shard-router.ts
import { Pool } from 'pg';

// Initialize connection pools for each shard
const shards: Pool[] = [
  new Pool({ connectionString: process.env.SHARD_0_URL }),
  new Pool({ connectionString: process.env.SHARD_1_URL }),
  new Pool({ connectionString: process.env.SHARD_2_URL }),
];

const SHARD_COUNT = shards.length;

function getShardIndex(userId: string): number {
  return parseInt(userId, 10) % SHARD_COUNT;
}

function getShardForUser(userId: string): Pool {
  return shards[getShardIndex(userId)];
}

export async function getUserById(userId: string) {
  const shard = getShardForUser(userId);
  const { rows } = await shard.query('SELECT * FROM users WHERE id = $1', [userId]);
  return rows[0] as { id: string; name: string; email: string } | undefined;
}

export async function createUser(userId: string, name: string, email: string): Promise<void> {
  const shard = getShardForUser(userId);
  await shard.query(
    'INSERT INTO users (id, name, email) VALUES ($1, $2, $3)',
    [userId, name, email],
  );
}
```

### Hash-based Shard Key (consistent hashing for rebalancing)

```typescript
// sharding/consistent-hash.ts
import crypto from 'crypto';

class ConsistentHashRing {
  private ring    = new Map<number, string>();
  private sortedKeys: number[] = [];

  constructor(nodes: string[], virtualNodes = 150) {
    for (const node of nodes) this.addNode(node, virtualNodes);
  }

  addNode(node: string, virtualNodes: number): void {
    for (let i = 0; i < virtualNodes; i++) {
      const hash = this.hash(`${node}:${i}`);
      this.ring.set(hash, node);
      this.sortedKeys.push(hash);
    }
    this.sortedKeys.sort((a, b) => a - b);
  }

  getNode(key: string): string {
    const hash = this.hash(key);
    for (const k of this.sortedKeys) {
      if (hash <= k) return this.ring.get(k)!;
    }
    return this.ring.get(this.sortedKeys[0])!; // Wrap around
  }

  private hash(key: string): number {
    return parseInt(crypto.createHash('md5').update(key).digest('hex').slice(0, 8), 16);
  }
}

const ring = new ConsistentHashRing(['shard-0', 'shard-1', 'shard-2']);
console.log(ring.getNode('user-12345')); // → deterministic shard
```

### Cross-Shard Aggregation (fan-out query)

```typescript
// Fan out a query to all shards and merge results
async function getTotalUserCount(): Promise<number> {
  const counts = await Promise.all(
    shards.map(async (shard) => {
      const { rows } = await shard.query('SELECT COUNT(*) AS cnt FROM users');
      return parseInt(rows[0].cnt as string, 10);
    }),
  );
  return counts.reduce((sum, c) => sum + c, 0);
}
```

## Related Patterns

- [22 — Read Replicas](./22-read-replicas.md) — Scale reads before reaching for sharding
- [31 — Database per Service](./31-database-per-service.md) — Service-level data ownership; sharding is within a service
- [12 — Caching Patterns](./12-caching-patterns.md) — Reduce hot shard load with a caching layer
