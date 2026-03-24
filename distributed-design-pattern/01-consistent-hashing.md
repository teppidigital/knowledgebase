# Consistent Hashing

## Category

Distributed Systems, Scalability, Data Partitioning

## Context

Consistent Hashing is a distributed data partitioning strategy that maps both data keys and server nodes onto the same **circular hash ring** (modulo 2³²). When a key needs to be routed, it is hashed and placed on the ring; the request goes to the first node clockwise from that point.

The key advantage over naive modulo hashing (`key % n`) is that adding or removing a node only causes a fraction of keys to be remapped (on average `K/n` where K = number of keys and n = number of nodes), instead of remapping almost all keys.

It is foundational in systems like Amazon DynamoDB, Apache Cassandra, Memcached, and CDN cache routing.

---

## Pros

- **Minimal key remapping**: Adding or removing a node only moves `~K/n` keys, not all of them.
- **Horizontal scalability**: Nodes can be added/removed with predictable and minimal data migration.
- **No central coordinator**: Key-to-node mapping is computed deterministically from the hash alone.
- **Virtual nodes**: Multiple virtual points per real node achieve more uniform key distribution.
- **Load balancing**: With virtual nodes, load distributes evenly even with few physical nodes.

---

## Cons

- **Uneven distribution without virtual nodes**: With few real nodes, hash ring segments are unequal.
- **Hotspots**: If a key range has very high traffic, one node absorbs it (mitigated by virtual nodes).
- **Data migration complexity**: When adding/removing nodes, affected key ranges must be migrated.
- **More complex than modulo hashing**: Requires specialized implementation and testing.
- **Node heterogeneity**: Handling nodes with different capacities requires weight-based virtual node counts.

---

## Design Diagram

```mermaid
graph LR
    subgraph Hash Ring 0 to 2³²
        direction LR
        N1["Node A\n(0-85M)"]
        N2["Node B\n(85M-170M)"]
        N3["Node C\n(170M-255M)"]
        N4["Node D\n(255M-340M)"]

        K1(["Key: user-100\nhash→ 50M → Node A"])
        K2(["Key: order-42\nhash→ 120M → Node B"])
        K3(["Key: product-7\nhash→ 220M → Node C"])
    end

    K1 --> N1
    K2 --> N2
    K3 --> N3
```

---

## Code Sample

### Consistent Hash Ring (TypeScript)

```typescript
// consistent-hashing/ring.ts
import * as crypto from "crypto";

interface VirtualNode {
  hash: number;
  nodeId: string;
}

export class ConsistentHashRing {
  private ring: VirtualNode[] = [];
  private readonly virtualNodesPerNode: number;

  constructor(virtualNodesPerNode = 150) {
    this.virtualNodesPerNode = virtualNodesPerNode;
  }

  addNode(nodeId: string): void {
    for (let i = 0; i < this.virtualNodesPerNode; i++) {
      const hash = this.hash(`${nodeId}-vnode-${i}`);
      this.ring.push({ hash, nodeId });
    }
    this.ring.sort((a, b) => a.hash - b.hash);
  }

  removeNode(nodeId: string): void {
    this.ring = this.ring.filter((vn) => vn.nodeId !== nodeId);
  }

  getNode(key: string): string | null {
    if (this.ring.length === 0) return null;
    const keyHash = this.hash(key);

    // Binary search for the first node clockwise from keyHash
    let lo = 0,
      hi = this.ring.length - 1;
    while (lo < hi) {
      const mid = Math.floor((lo + hi) / 2);
      if (this.ring[mid].hash < keyHash) lo = mid + 1;
      else hi = mid;
    }

    // Wrap around to first node if beyond the last
    const idx = this.ring[lo].hash >= keyHash ? lo : 0;
    return this.ring[idx].nodeId;
  }

  getReplicaNodes(key: string, replicaCount: number): string[] {
    if (this.ring.length === 0) return [];
    const keyHash = this.hash(key);
    const seen = new Set<string>();
    const nodes: string[] = [];

    let startIdx = this.findStartIndex(keyHash);

    for (let i = 0; i < this.ring.length && nodes.length < replicaCount; i++) {
      const idx = (startIdx + i) % this.ring.length;
      const nodeId = this.ring[idx].nodeId;
      if (!seen.has(nodeId)) {
        seen.add(nodeId);
        nodes.push(nodeId);
      }
    }

    return nodes;
  }

  private findStartIndex(keyHash: number): number {
    let lo = 0,
      hi = this.ring.length - 1;
    while (lo < hi) {
      const mid = Math.floor((lo + hi) / 2);
      if (this.ring[mid].hash < keyHash) lo = mid + 1;
      else hi = mid;
    }
    return this.ring[lo].hash >= keyHash ? lo : 0;
  }

  private hash(key: string): number {
    const hex = crypto.createHash("md5").update(key).digest("hex").slice(0, 8);
    return parseInt(hex, 16);
  }

  getDistribution(): Record<string, number> {
    const dist: Record<string, number> = {};
    for (const vn of this.ring) {
      dist[vn.nodeId] = (dist[vn.nodeId] ?? 0) + 1;
    }
    return dist;
  }
}
```

### Usage — Cache Routing

```typescript
// cache/cache-router.ts
import { ConsistentHashRing } from "../consistent-hashing/ring";
import { createClient } from "redis";

const ring = new ConsistentHashRing(150);
const cacheNodes: Record<string, ReturnType<typeof createClient>> = {};

function addCacheNode(nodeId: string, url: string): void {
  ring.addNode(nodeId);
  cacheNodes[nodeId] = createClient({ url });
  console.log(`Added cache node: ${nodeId}`);
}

function removeCacheNode(nodeId: string): void {
  ring.removeNode(nodeId);
  cacheNodes[nodeId]?.quit();
  delete cacheNodes[nodeId];
  console.log(`Removed cache node: ${nodeId}`);
}

async function get(key: string): Promise<string | null> {
  const nodeId = ring.getNode(key);
  if (!nodeId) return null;
  return cacheNodes[nodeId].get(key);
}

async function set(key: string, value: string, ttl = 300): Promise<void> {
  const nodeId = ring.getNode(key);
  if (!nodeId) throw new Error("No cache nodes available");
  await cacheNodes[nodeId].setEx(key, ttl, value);
}

// Setup: 3 cache nodes
addCacheNode("cache-1", "redis://cache-1:6379");
addCacheNode("cache-2", "redis://cache-2:6379");
addCacheNode("cache-3", "redis://cache-3:6379");

// Demonstrate key routing
const testKeys = ["user-1", "user-2", "order-99", "product-42"];
for (const key of testKeys) {
  console.log(`Key "${key}" → ${ring.getNode(key)}`);
}

// Add a 4th node — only ~25% of keys remap
addCacheNode("cache-4", "redis://cache-4:6379");
```

### Weighted Virtual Nodes (heterogeneous capacity)

```typescript
// Add more virtual nodes for higher-capacity nodes
function addNodeWithWeight(nodeId: string, weightMultiplier: number): void {
  const baseVNodes = 150;
  const vnodeCount = Math.floor(baseVNodes * weightMultiplier);
  for (let i = 0; i < vnodeCount; i++) {
    const hash = ring["hash"](`${nodeId}-vnode-${i}`);
    ring["ring"].push({ hash, nodeId });
  }
  ring["ring"].sort((a, b) => a.hash - b.hash);
}

addNodeWithWeight("large-node", 2.0); // Gets ~2x the keys
addNodeWithWeight("small-node", 0.5); // Gets ~0.5x the keys
```
