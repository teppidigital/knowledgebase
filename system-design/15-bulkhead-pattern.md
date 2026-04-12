# Bulkhead Pattern

## Category

Resilience, Fault Isolation, Stability

## Context

The Bulkhead pattern isolates elements of an application into pools so that if one fails, the others continue to function. Named after the watertight compartments in a ship's hull — if one compartment floods, the ship doesn't sink.

In software, bulkheads prevent one slow or failing component from consuming all available resources (threads, connections, memory) and degrading the entire system. Resources are partitioned into separate pools per service, client, or use case.

---

## Pros

- **Fault isolation**: A failure or slowdown in one pool does not exhaust resources for other pools.
- **Predictable capacity**: Each consumer or service has guaranteed resources.
- **Prevents cascading failures**: Resource exhaustion in one area stays contained.
- **Better SLAs**: Critical paths get dedicated pools and can meet SLAs even under stress.
- **Pairs well with Circuit Breaker**: Combined, they offer strong resilience.

---

## Cons

- **Resource underutilization**: Pools sit idle when not fully utilized — resources can't be shared.
- **Configuration overhead**: Sizing each pool correctly requires load testing and tuning.
- **More complex thread/connection management**: Especially in polyglot or async environments.
- **Diminishing returns for small systems**: The complexity may not be worth it at low scale.

---

## Design Diagram

```mermaid
graph TD
    App["Application"]

    subgraph "Payment Pool (max 10 connections)"
        P1["Thread 1"]
        P2["Thread 2"]
        P3["Thread 3"]
        Pdots["... up to 10"]
    end

    subgraph "Inventory Pool (max 10 connections)"
        I1["Thread 1"]
        I2["Thread 2"]
        I3["Thread 3"]
        Idots["... up to 10"]
    end

    subgraph "Notification Pool (max 5 connections)"
        N1["Thread 1"]
        N2["Thread 2"]
        Ndots["... up to 5"]
    end

    PaymentSvc["Payment Service"]
    InventorySvc["Inventory Service"]
    NotifSvc["Notification Service"]

    App -->|"Payment calls"| P1
    App -->|"Payment calls"| P2
    P1 --> PaymentSvc
    P2 --> PaymentSvc

    App -->|"Inventory calls"| I1
    I1 --> InventorySvc

    App -->|"Notification calls"| N1
    N1 --> NotifSvc
```

---

## Code Sample

### Thread Pool Bulkhead (Node.js — using worker threads)

```typescript
// bulkhead/pool.ts
interface QueueEntry { resolve: () => void; reject: (err: Error) => void; }

export class BulkheadPool {
  private activeCount = 0;
  private queue: QueueEntry[] = [];

  constructor(
    private readonly maxConcurrent: number,
    private readonly queueSize: number,
  ) {}

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.activeCount >= this.maxConcurrent) {
      if (this.queue.length >= this.queueSize) {
        throw new Error('Bulkhead queue full — request rejected');
      }
      await new Promise<void>((resolve, reject) => {
        this.queue.push({ resolve, reject });
      });
    }

    this.activeCount++;
    try {
      return await fn();
    } finally {
      this.activeCount--;
      const next = this.queue.shift();
      next?.resolve();
    }
  }
}

// Separate pools per downstream service
export const paymentPool      = new BulkheadPool(10, 20);
export const inventoryPool    = new BulkheadPool(10, 20);
export const notificationPool = new BulkheadPool(5,  10);
```

### Usage with HTTP calls

```typescript
// services/payment.service.ts
import { paymentPool, inventoryPool } from '../bulkhead/pool';
import axios from 'axios';

export async function processPayment(orderId: string, amount: number) {
  return paymentPool.execute(async () => {
    const response = await axios.post('http://payment-service/pay', { orderId, amount });
    return response.data;
  });
}

export async function callInventory(orderId: string, items: unknown[]) {
  return inventoryPool.execute(async () => {
    const response = await axios.post('http://inventory-service/reserve', { orderId, items });
    return response.data;
  });
}
```

### Connection Pool Bulkhead (PostgreSQL per service)

```typescript
// db/pools.ts
import { Pool } from 'pg';

// Dedicated connection pools per domain — not one shared pool
export const orderPool = new Pool({
  connectionString: process.env.ORDER_DB_URL,
  max: 20,               // Dedicated 20 connections for order operations
  idleTimeoutMillis: 30_000,
});

export const reportPool = new Pool({
  connectionString: process.env.REPORT_DB_URL,
  max: 5,                // Reports get fewer connections — lower priority
  idleTimeoutMillis: 60_000,
  connectionTimeoutMillis: 3_000,
});
```

### Combining Bulkhead + Circuit Breaker

```typescript
// resilience/resilient-call.ts
import { CircuitBreaker } from "./circuit-breaker";
import { BulkheadPool } from "./bulkhead";

const pool = new BulkheadPool(10, 20);
const breaker = new CircuitBreaker({
  failureThreshold: 5,
  recoveryTimeout: 30_000,
  successThreshold: 2,
});

export async function resilientCall<T>(
  fn: () => Promise<T>,
  fallback?: () => T,
): Promise<T> {
  return pool.execute(() => breaker.execute(fn, fallback));
}
```

## Related Patterns

- [13 — Circuit Breaker](./13-circuit-breaker.md) — Fail-fast complement to bulkhead's concurrency isolation
- [24 — Retry Pattern](./24-retry-pattern.md) — Combine all three (retry + CB + bulkhead) for defence-in-depth resilience
- [10 — Load Balancing](./10-load-balancing.md) — Load balancer distributes across instances; bulkhead isolates within an instance
