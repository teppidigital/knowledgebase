# Modularity and Coupling

## Category

Continuous Architecture — Design

## Context

Modularity is the degree to which a system's components can be separated and recombined. It is the primary structural lever for changeability (Principle 4: the power of small). Coupling is the measure of interdependence between modules — the enemy of modularity.

Most systems start well-modularised and degrade over time. Coupling tends to increase with every sprint because it is always easier to add a dependency than to avoid one. Modularity requires active, continuous management.

### Coupling Taxonomy

| Type | Description | Example | How to fix |
|---|---|---|---|
| **Content coupling** | Module A directly accesses internal data of module B | Service A reads directly from service B's database table | Define a service API; no cross-DB reads |
| **Common coupling** | Multiple modules share mutable global state | Shared singleton config object mutated by multiple services | Pass config explicitly; make shared state immutable |
| **External coupling** | Multiple modules depend on the same external format/protocol | 5 services tightly coupled to a vendor API format | Anti-corruption layer / adapter per consumer |
| **Control coupling** | Module A passes a flag to module B to control its behaviour | `processPayment(type: 'chase' | 'stripe')` | Replace with polymorphism or strategy pattern |
| **Stamp coupling** | Module A passes a large data structure of which B uses 2 fields | Passing full `Order` object to `EmailService` | Pass only the fields needed |
| **Data coupling** | Module A passes only the data needed to module B | `sendEmail(to: string, subject: string, body: string)` | Ideal — maintain this |
| **Message coupling** | Modules communicate through events/messages only | Kafka events between services | Ideal for microservices |

Coupling increases from top to bottom (content = worst; message = best for distributed systems).

### Cohesion

Cohesion is the degree to which elements within a module belong together. High cohesion and low coupling are the twin goals of modular design.

| Cohesion type | Description | Example |
|---|---|---|
| **Coincidental** (worst) | Elements grouped arbitrarily | `UtilsService` with logging, formatting, payment retry |
| **Logical** | Elements do similar things but are unrelated | `InputHandler` for mouse, keyboard, touch, voice |
| **Temporal** | Elements execute at the same time | Startup initialisation routines grouped together |
| **Procedural** | Elements follow a sequence | `validateOrder → reserveInventory → chargePayment` |
| **Communicational** | Elements operate on the same data | All functions that operate on the `Order` entity |
| **Sequential** | Output of one is input of next | `parseCSV → validateRow → transformRow → insertRow` |
| **Functional** (best) | Elements all contribute to a single, well-defined function | `PaymentProcessor` — only processes payments |

## Pros

- Low-coupled, high-cohesion modules can be changed, tested, and deployed independently
- Blast radius of failures is bounded to the module boundary
- Teams can own modules autonomously — reduces coordination overhead
- Modules can be extracted into services when scale justifies it, without rewriting
- Dependency analysis provides an objective measure of architectural health

## Cons

- Refactoring toward low coupling takes time and is often deprioritised
- Premature modularisation creates overhead without benefit (too many modules too early)
- Module boundaries that don't align with team boundaries create friction (Conway's Law)
- Some coupling is intentional and correct — eliminating all coupling is neither possible nor desirable

## Design Diagram

```mermaid
flowchart LR
    subgraph Domain Layer — High Cohesion
        ORDER[Order Domain\nOrderAggregate\nOrderRepository port]
        PAYMENT[Payment Domain\nPaymentAggregate\nPaymentRepository port]
        INVENTORY[Inventory Domain\nInventoryAggregate\nInventoryRepository port]
    end

    subgraph Infrastructure Layer — Adapters
        DB_O[(Orders DB\nPostgres)]
        DB_P[(Payments DB\nPostgres)]
        MQ[Message Broker\nKafka]
    end

    ORDER -- events --> MQ
    PAYMENT -- events --> MQ
    MQ -- consumes --> INVENTORY
    ORDER --> DB_O
    PAYMENT --> DB_P

    note1["❌ NEVER: ORDER reads from PAYMENT DB directly"]
    note2["✅ ALWAYS: communicate via events or API"]
```

## Code Sample

### Measuring coupling with dependency-cruiser

```bash
# Generate coupling metrics for all modules
npx depcruise src --output-type metrics | jq '.modules[] | {
  name: .source,
  afferent: .dependents | length,       # number of modules that depend on this
  efferent: .dependencies | length,     # number of modules this depends on
  instability: (.dependencies | length) / ((.dependencies | length) + (.dependents | length))
}' | sort-by '.instability'
```

### Package principle: Stable Dependencies Principle

```typescript
// ✅ CORRECT: Stable modules (low instability) depended on by unstable modules
//    Domain (stable) ← Application (unstable) ← Infrastructure (unstable)

// domain/payment.ts — zero external dependencies (most stable)
export interface Payment {
  id: string;
  amount: Money;
  status: PaymentStatus;
}

export interface PaymentRepository {
  findById(id: string): Promise<Payment | null>;
  save(payment: Payment): Promise<void>;
}

// application/charge-payment.ts — depends on domain only
import { PaymentRepository } from '../domain/payment';

export class ChargePaymentUseCase {
  constructor(private payments: PaymentRepository) {} // depends on stable port
  async execute(amount: Money, method: string): Promise<Payment> { /* ... */ }
}

// infrastructure/postgres-payment-repo.ts — depends on domain + external libs
import { PaymentRepository, Payment } from '../../domain/payment';
import { Pool } from 'pg'; // external dependency — isolated here only

export class PostgresPaymentRepository implements PaymentRepository {
  constructor(private db: Pool) {}
  async findById(id: string) { /* ... */ }
  async save(payment: Payment) { /* ... */ }
}
```

### Architecture fitness function — coupling threshold

```typescript
// test/architecture/coupling.test.ts
import { join } from 'path';
import cruise from 'dependency-cruiser';

describe('Architecture fitness: coupling thresholds', () => {
  it('domain modules have zero external dependencies', () => {
    const result = cruise(['src/domain'], {
      exclude: { path: 'node_modules' },
      outputType: 'json',
    });
    const domainModules = result.output.modules.filter(m =>
      m.source.startsWith('src/domain')
    );
    for (const mod of domainModules) {
      const externalDeps = mod.dependencies.filter(d => d.dependencyTypes.includes('npm'));
      expect(externalDeps).toHaveLength(0); // domain must not depend on npm packages
    }
  });

  it('no circular dependencies in any module', () => {
    const result = cruise(['src'], { outputType: 'json' });
    const circular = result.output.modules.filter(m =>
      m.dependencies.some(d => d.circular)
    );
    expect(circular).toHaveLength(0);
  });
});
```

## Key Patterns

### The Instability Metric

```
Instability (I) = Ce / (Ca + Ce)

Where:
  Ca = Afferent coupling (modules that depend ON this module)
  Ce = Efferent coupling (modules that this module depends ON)

I = 0 → Maximally stable (nothing this module depends on can change without affecting it)
I = 1 → Maximally unstable (this module can change freely; nothing depends on it)
```

**Stable Dependencies Principle**: A module should only depend on modules that are more stable than itself. Put concretely: infrastructure (I≈1) depends on application (I≈0.5) which depends on domain (I≈0).

### Modularisation Strategies

| Strategy | When to use | Mechanism |
|---|---|---|
| **Modular monolith** | Pre-scale; team <20 engineers; domain not yet clear | Strict package boundaries enforced by FF; no cross-package direct access |
| **Service extraction** | Seam is stable; team needs independent deployability; scale justifies it | Extract after the boundary is well-proven in the monolith |
| **Anti-corruption layer** | Integrating with external systems or legacy | Adapter converts external model to internal domain model |
| **Strangler fig** | Incrementally replacing a legacy system | New system intercepts traffic; gradually replaces old system piece by piece |

### Common Refactoring Patterns for Coupling

| Smell | Coupling type | Refactoring |
|---|---|---|
| Service A fetches data from Service B's DB | Content | Define a Service B API; Service B publishes events |
| Passing large objects where only 2 fields are used | Stamp | Extract a DTO with only the fields needed |
| `if (type === 'A') {...} else if (type === 'B') {...}` spread everywhere | Control | Strategy pattern; polymorphism; registry pattern |
| Shared mutable singleton cache | Common | Pass explicit config; localise the cache; event-driven invalidation |
| 12 services all import the same `utils/helpers.ts` | Afferent coupling on a low-cohesion module | Split `helpers` into focused domain-specific modules |

## Related Patterns

- [01 — Six Principles](./01-six-principles.md) — Principle 4: Power of Small
- [06 — Fitness Functions](./06-fitness-functions.md) — Automating coupling threshold gates
- [03 — Technical Debt](./03-technical-debt.md) — Coupling metrics as debt indicators
- [09 — Team Topologies](./09-team-topologies.md) — Coupling in the org (Conway's Law)
