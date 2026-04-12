# Microservices Architecture

## Category

Architectural, Scalability, Maintainability

## Context

Microservices is an architectural style that structures an application as a collection of small, independently deployable services. Each service is self-contained, focuses on a single business capability, owns its data, and communicates with other services via well-defined APIs (typically REST, gRPC, or messaging queues).

This pattern emerged as an alternative to monolithic architectures to address problems of large-scale systems that need to be maintained and deployed frequently by independent teams.

---

## Pros

- **Independent deployability**: Each service can be deployed, updated, scaled, and restarted independently without affecting other services.
- **Technology heterogeneity**: Different services can use different programming languages, frameworks, and databases best suited for their needs.
- **Fault isolation**: A failure in one service does not cascade to bring down the entire system.
- **Team autonomy**: Small, cross-functional teams can own and operate individual services independently.
- **Scalability**: Individual services can be scaled horizontally based on demand.
- **Easier maintenance**: Smaller codebases are easier to understand, test, and modify.

---

## Cons

- **Distributed system complexity**: Network latency, partial failures, and data consistency across services add significant complexity.
- **Operational overhead**: Requires robust infrastructure—container orchestration (Kubernetes), service discovery, distributed tracing, centralized logging.
- **Inter-service communication**: Services must handle network failures, retries, and timeouts.
- **Data management**: Each service owns its own database, making joins and cross-service transactions harder.
- **Testing complexity**: Integration and end-to-end testing across multiple services is more difficult.
- **Latency**: Network calls between services are slower than in-process calls.

---

## Design Diagram

```mermaid
graph TD
    Client["Client (Web / Mobile)"]
    GW["API Gateway"]
    US["User Service"]
    OS["Order Service"]
    PS["Payment Service"]
    NS["Notification Service"]
    UserDB[("User DB")]
    OrderDB[("Order DB")]
    PaymentDB[("Payment DB")]
    MQ["Message Broker (Kafka / RabbitMQ)"]

    Client --> GW
    GW --> US
    GW --> OS
    GW --> PS
    US --> UserDB
    OS --> OrderDB
    PS --> PaymentDB
    OS -->|"order.created event"| MQ
    MQ --> NS
    MQ --> PS
```

---

## Code Sample

### Service A — Order Service (Node.js / Express)

```typescript
// order-service/src/index.ts
import express from 'express';
import { Kafka } from 'kafkajs';

const app = express();
app.use(express.json());

const kafka = new Kafka({ brokers: ['kafka:9092'] });
const producer = kafka.producer();

interface OrderBody { items: unknown[]; customerId: string; }

app.post('/orders', async (req, res) => {
  const order = { id: Date.now(), ...(req.body as OrderBody), status: 'PENDING' };
  // Persist to Order DB (omitted for brevity)
  await producer.send({
    topic: 'order.created',
    messages: [{ value: JSON.stringify(order) }],
  });
  res.status(201).json(order);
});

(async () => {
  await producer.connect();
  app.listen(3001, () => console.log('Order Service running on port 3001'));
})();
```

### Service B — Notification Service (Node.js)

```typescript
// notification-service/src/index.ts
import { Kafka } from 'kafkajs';

const kafka = new Kafka({ brokers: ['kafka:9092'] });
const consumer = kafka.consumer({ groupId: 'notification-group' });

(async () => {
  await consumer.connect();
  await consumer.subscribe({ topic: 'order.created', fromBeginning: false });
  await consumer.run({
    eachMessage: async ({ message }) => {
      const order = JSON.parse(message.value!.toString()) as { id: number };
      console.log(`Sending notification for order: ${order.id}`);
      // Send email / push notification
    },
  });
})();
```

### Docker Compose Skeleton

```yaml
version: "3.8"
services:
  order-service:
    build: ./order-service
    ports: ["3001:3001"]
    environment:
      - KAFKA_BROKER=kafka:9092

  notification-service:
    build: ./notification-service
    environment:
      - KAFKA_BROKER=kafka:9092

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    ports: ["9092:9092"]
```

## Related Patterns

- [02 — Monolithic Architecture](./02-monolithic.md) — Consider monolith-first; only migrate when team and scale justify it
- [07 — API Gateway](./07-api-gateway.md) — Single entry point for all clients in a microservices system
- [26 — Service Mesh](./26-service-mesh.md) — Handles mTLS, retries, and observability between services
- [31 — Database per Service](./31-database-per-service.md) — Each service must own its own data store
- [03 — Event-Driven Architecture](./03-event-driven-architecture.md) — Preferred communication style for loose coupling
- [09 — Backends for Frontends](./09-backends-for-frontends.md) — Dedicated API layer per client type
- [06 — Saga Pattern](./06-saga-pattern.md) — Manage distributed transactions across services
