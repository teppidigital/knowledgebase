# Data Architecture Patterns

## Category

DDIA — Derived Data (Chapter 12 synthesis)

## Context

This file synthesises the architectural patterns that emerge from the DDIA book's closing argument: there is no single system that handles every data requirement well. The craft of data architecture is composing specialised systems into a coherent whole, with a reliable integration backbone.

### CQRS — Command Query Responsibility Segregation

CQRS separates the write path (commands) from the read path (queries). The write model enforces invariants and records facts. The read model is a denormalised, query-optimised view.

**Without CQRS:**
```
Application → [single PostgreSQL DB] ← Read queries + Write commands
```
One model must serve all query patterns. Complex reporting queries slow down transactional tables.

**With CQRS:**
```
Commands → Write model (PostgreSQL) → events → Read models (Elasticsearch, Redis, OLAP DB)
                                                     ↑ Read queries
```

### Event Sourcing

Event sourcing stores the full history of state changes as an append-only log of events — rather than storing only the current state. The current state is derived by replaying events.

| Concept | Event Sourcing | Traditional (State-Based) |
|---|---|---|
| **What is stored** | Event: `OrderShipped { orderId, carrier, time }` | Current row: `order SET status='shipped'` |
| **History** | ✅ Complete audit trail | ❌ Only current state (unless triggers) |
| **Rebuilding** | Replay events from offset 0 | Cannot rebuild past states |
| **Concurrency** | Optimistic: check expected version on append | Pessimistic: row-level locks |
| **Complexity** | Higher (projections, schema evolution) | Lower (simple CRUD) |

### Lambda Architecture

Lambda architecture ensures accuracy (batch layer) + low latency (speed layer) by running both in parallel. Queries hit the serving layer which merges both views.

| Layer | Responsibility | Lag | Technology |
|---|---|---|---|
| **Batch layer** | Recompute everything periodically from raw data | Hours | Spark, Hadoop |
| **Speed layer** | Compute real-time incremental updates | Seconds | Flink, Kafka Streams |
| **Serving layer** | Merge batch + speed results; answer queries | — | Druid, HBase, Cassandra |

**Weakness**: same business logic must be implemented twice (batch + stream). They can drift apart.

### Kappa Architecture

Kappa replaces Lambda's batch layer with log replay. Stream processing handles both real-time and historical reprocessing.

```
All events → Kafka (retained, partitioned)
              ├── Current version: stream processor → output table V2
              └── New version:     stream processor → output table V3 (replay from offset 0)
                                                  ↑ migrate readers when ready
```

### Data Mesh

Data Mesh is an organisational and architectural pattern where **domain teams own and publish their own data products**.

| Principle | What it means |
|---|---|
| **Domain ownership** | The team that produces the data is responsible for its quality and SLOs |
| **Data as a product** | Data is published with documentation, schema, SLAs — not just dumped in S3 |
| **Self-serve infrastructure** | Platform team provides tooling; domain teams do not depend on central data team |
| **Federated governance** | Central standards (schemas, security); local ownership of data products |

## Pros (Event Sourcing + CQRS combined)

- Complete audit trail — regulatory compliance; time-travel queries; debugging
- Multiple read models can be created independently for each access pattern
- Read models can be rebuilt from scratch — no schema migration required for read side
- Decoupled read and write scaling — read replicas, caches, search indexes independent of write DB

## Cons

- Eventual consistency between write model and read models — application must tolerate lag
- Event schema evolution is hard — old events must still be processable after schema change
- High initial complexity — not appropriate for simple CRUD applications
- Debugging requires understanding the full event projection chain

## Design Diagram

```mermaid
flowchart TD
    subgraph CQRS + Event Sourcing
        CMD[Command<br/>CreateOrder<br/>ShipOrder]
        AGG[Order Aggregate<br/>Apply events<br/>Enforce invariants]
        ES[(Event Store<br/>Append-only<br/>order-events)]
        CMD --> AGG --> ES
    end

    subgraph Projections — derived read models
        P1[OrderListProjection<br/>→ PostgreSQL read DB]
        P2[OrderSearchProjection<br/>→ Elasticsearch]
        P3[AnalyticsProjection<br/>→ ClickHouse OLAP]
        P4[AuditProjection<br/>→ S3 Parquet]

        ES --> P1
        ES --> P2
        ES --> P3
        ES --> P4
    end

    subgraph Query Side
        Q1[List orders API<br/>PostgreSQL read]
        Q2[Search API<br/>Elasticsearch]
        Q3[Analytics dashboard<br/>SQL on ClickHouse]
    end

    P1 --> Q1
    P2 --> Q2
    P3 --> Q3
```

## Code Sample

### Event Sourcing with aggregate + projection

```typescript
// Event Sourcing: store events, not state

// ——— Domain Events ———
type OrderEvent =
  | { type: 'ORDER_CREATED'; orderId: string; customerId: string; items: string[]; total: number; at: number }
  | { type: 'PAYMENT_RECEIVED'; orderId: string; amount: number; at: number }
  | { type: 'ORDER_SHIPPED'; orderId: string; carrier: string; trackingNumber: string; at: number }
  | { type: 'ORDER_CANCELLED'; orderId: string; reason: string; at: number };

// ——— Aggregate State ———
interface OrderState {
  id: string;
  status: 'pending' | 'paid' | 'shipped' | 'cancelled';
  customerId: string;
  total: number;
  version: number;
}

// ——— Reducer: apply one event to state ———
function applyOrderEvent(state: OrderState | null, event: OrderEvent): OrderState {
  switch (event.type) {
    case 'ORDER_CREATED':
      return { id: event.orderId, status: 'pending', customerId: event.customerId, total: event.total, version: 1 };
    case 'PAYMENT_RECEIVED':
      return { ...state!, status: 'paid', version: state!.version + 1 };
    case 'ORDER_SHIPPED':
      return { ...state!, status: 'shipped', version: state!.version + 1 };
    case 'ORDER_CANCELLED':
      return { ...state!, status: 'cancelled', version: state!.version + 1 };
  }
}

// ——— Event Store ———
class InMemoryEventStore {
  private streams = new Map<string, OrderEvent[]>();

  append(streamId: string, events: OrderEvent[], expectedVersion: number): void {
    const existing = this.streams.get(streamId) ?? [];
    if (existing.length !== expectedVersion) {
      throw new Error(`Optimistic concurrency conflict: expected version ${expectedVersion}, actual ${existing.length}`);
    }
    this.streams.set(streamId, [...existing, ...events]);
  }

  load(streamId: string): OrderEvent[] {
    return this.streams.get(streamId) ?? [];
  }

  // Rebuild state by replaying events
  getState(streamId: string): OrderState | null {
    const events = this.load(streamId);
    return events.reduce<OrderState | null>((state, event) => applyOrderEvent(state, event), null);
  }
}

// ——— Projection (read model) ———
class OrderListProjection {
  private orders = new Map<string, { id: string; status: string; total: number }>();

  apply(event: OrderEvent): void {
    switch (event.type) {
      case 'ORDER_CREATED':
        this.orders.set(event.orderId, { id: event.orderId, status: 'pending', total: event.total });
        break;
      case 'PAYMENT_RECEIVED':
        this.update(event.orderId, { status: 'paid' });
        break;
      case 'ORDER_SHIPPED':
        this.update(event.orderId, { status: 'shipped' });
        break;
      case 'ORDER_CANCELLED':
        this.update(event.orderId, { status: 'cancelled' });
        break;
    }
  }

  private update(id: string, patch: Partial<{ status: string }>): void {
    const existing = this.orders.get(id);
    if (existing) this.orders.set(id, { ...existing, ...patch });
  }

  query(): Array<{ id: string; status: string; total: number }> {
    return [...this.orders.values()];
  }
}

// ——— Usage ———
const store = new InMemoryEventStore();
const projection = new OrderListProjection();

// Write side: command handler appends events
const events: OrderEvent[] = [
  { type: 'ORDER_CREATED', orderId: 'ord-1', customerId: 'cust-1', items: ['prod-a'], total: 150, at: Date.now() }
];
store.append('ord-1', events, 0); // expectedVersion=0 (new stream)
events.forEach(e => projection.apply(e)); // update read model

// Read side: query projection
console.log(projection.query()); // [{ id: 'ord-1', status: 'pending', total: 150 }]

// Rebuild projection from scratch (Kappa-style reprocess)
const freshProjection = new OrderListProjection();
store.load('ord-1').forEach(e => freshProjection.apply(e));
```

### Data architecture decision framework

```typescript
// Decision tree: which architecture to use?
function recommendArchitecture(requirements: {
  queryPatterns: ('point-lookup' | 'full-text-search' | 'analytics' | 'graph')[];
  latencyRequirement: 'realtime' | 'near-realtime' | 'batch';
  auditRequired: boolean;
  multipleReadModels: boolean;
  crossServiceTransactions: boolean;
}): string[] {
  const recommendations: string[] = [];

  if (requirements.auditRequired || requirements.multipleReadModels) {
    recommendations.push('Event Sourcing: store events as source of truth');
    recommendations.push('CQRS: separate write model from read models');
  }

  if (requirements.latencyRequirement === 'realtime') {
    recommendations.push('Kappa Architecture: single stream processor for both real-time and historical reprocessing');
  } else if (requirements.latencyRequirement === 'batch') {
    recommendations.push('Lambda Architecture: batch layer for accuracy + speed layer for real-time');
  }

  if (requirements.queryPatterns.includes('full-text-search')) {
    recommendations.push('Elasticsearch / OpenSearch projection: CDC or event-driven sync');
  }
  if (requirements.queryPatterns.includes('analytics')) {
    recommendations.push('ClickHouse / BigQuery projection: batch or stream insert');
  }
  if (requirements.queryPatterns.includes('graph')) {
    recommendations.push('Neo4j or Neptune for graph queries: event-driven sync');
  }

  if (requirements.crossServiceTransactions) {
    recommendations.push('Saga Pattern (orchestration): compensating transactions across services');
    recommendations.push('Outbox Pattern: atomic DB + event publication in one local transaction');
  }

  return recommendations;
}
```

## Key Patterns

### Architecture Pattern Comparison

| Pattern | Consistency | Complexity | Best for |
|---|---|---|---|
| **Single RDBMS** | Strong | Low | Monoliths; small scale; all-in-one |
| **CQRS** | Eventual (read side) | Medium | Multiple query patterns from one write model |
| **Event Sourcing + CQRS** | Eventual (projections) | High | Audit trail; complex domain; multi-model reads |
| **Lambda Architecture** | Strong (batch layer) | High | Accurate historical analytics + real-time dashboard |
| **Kappa Architecture** | Eventual | Medium | Event-driven; Kafka-based; simpler than Lambda |
| **Data Mesh** | Per-domain SLO | High (org change) | Large orgs with many independent data producers |

### DDIA Mental Model Summary

```
📦 System of Record (single source of truth)
    │
    ▼ append-only event log (Kafka / WAL)
    │
    ├── Stream processor → real-time projections (Redis, Elasticsearch)
    ├── Batch processor  → historical projections (OLAP, ML features)
    └── CDC sync         → derived databases (PostgreSQL read replica, ClickHouse)

Rules:
1. Never mutate the event log — only append
2. Every derived store can be deleted + rebuilt from the log
3. The log defines causal order; projections are deterministic functions of the log
4. Embrace eventual consistency in derived stores — design queries and UX accordingly
```

## Related Patterns

- [12 — Derived Data Systems](./12-derived-data-systems.md) — Lambda/Kappa and the log as integration backbone
- [11 — Stream Processing](./11-stream-processing.md) — Kappa architecture stream processor
- [10 — Batch Processing](./10-batch-processing.md) — Lambda architecture batch layer
- [14 — Distributed Transactions](./14-distributed-transactions.md) — Saga and Outbox patterns for cross-service consistency
- [07 — Transactions](./07-transactions.md) — Optimistic concurrency in event sourcing (aggregate version check)
