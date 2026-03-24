# Event-Driven Architecture (EDA)

## Category
Architectural, Scalability, Decoupling, Asynchronous

## Context

Event-Driven Architecture (EDA) is a design paradigm in which components of a system communicate by producing and consuming **events**. An event represents a significant state change (e.g., "OrderPlaced", "PaymentProcessed"). Producers emit events without knowing who will consume them; consumers subscribe to event streams and react independently.

EDA enables high decoupling, elasticity, and real-time responsiveness. It is commonly implemented using message brokers like Apache Kafka, RabbitMQ, or AWS EventBridge.

---

## Pros

- **Loose coupling**: Producers and consumers are decoupled — neither knows about each other.
- **Scalability**: Consumers can scale independently based on event backlog.
- **Resilience**: If a consumer is unavailable, events are buffered and processed when it recovers.
- **Extensibility**: New consumers can subscribe to existing events without changing producers.
- **Real-time processing**: Enables reactive, real-time data pipelines.
- **Auditability**: Event logs provide a natural audit trail.

---

## Cons

- **Eventual consistency**: The system is eventually consistent — not immediately consistent.
- **Complexity**: Harder to trace the flow of a business process across multiple events and services.
- **Debugging difficulty**: Distributed, asynchronous systems are harder to debug and observe.
- **Event schema evolution**: Changing event schemas without breaking consumers is challenging.
- **Message ordering**: Maintaining strict event ordering at scale requires careful design.
- **Duplicate events**: Consumers must be idempotent to handle event redelivery.

---

## Design Diagram

```mermaid
sequenceDiagram
    participant Client
    participant OrderService as Order Service
    participant Broker as Message Broker (Kafka)
    participant PaymentService as Payment Service
    participant NotificationService as Notification Service
    participant InventoryService as Inventory Service

    Client->>OrderService: POST /orders
    OrderService->>Broker: Publish "OrderPlaced" event
    Broker-->>PaymentService: Consume "OrderPlaced"
    Broker-->>InventoryService: Consume "OrderPlaced"
    Broker-->>NotificationService: Consume "OrderPlaced"
    PaymentService->>Broker: Publish "PaymentProcessed" event
    Broker-->>OrderService: Consume "PaymentProcessed"
    Broker-->>NotificationService: Consume "PaymentProcessed"
```

---

## Code Sample

### Producer — Order Service (Node.js / KafkaJS)

```javascript
// order-service/producer.js
const { Kafka } = require('kafkajs');

const kafka = new Kafka({ clientId: 'order-service', brokers: ['kafka:9092'] });
const producer = kafka.producer();

async function publishOrderPlaced(order) {
  await producer.connect();
  await producer.send({
    topic: 'order.placed',
    messages: [
      {
        key: String(order.id),
        value: JSON.stringify({
          eventType: 'OrderPlaced',
          timestamp: new Date().toISOString(),
          payload: order,
        }),
      },
    ],
  });
}

module.exports = { publishOrderPlaced };
```

### Consumer — Payment Service (Node.js / KafkaJS)

```javascript
// payment-service/consumer.js
const { Kafka } = require('kafkajs');

const kafka = new Kafka({ clientId: 'payment-service', brokers: ['kafka:9092'] });
const consumer = kafka.consumer({ groupId: 'payment-service-group' });

async function startConsumer() {
  await consumer.connect();
  await consumer.subscribe({ topic: 'order.placed', fromBeginning: false });

  await consumer.run({
    eachMessage: async ({ topic, partition, message }) => {
      const event = JSON.parse(message.value.toString());
      if (event.eventType === 'OrderPlaced') {
        console.log(`Processing payment for order: ${event.payload.id}`);
        // Process payment logic here
        await publishPaymentProcessed(event.payload.id);
      }
    },
  });
}

startConsumer().catch(console.error);
```

### Idempotent Consumer (preventing duplicate processing)

```javascript
const processedEvents = new Set(); // Use Redis in production

async function eachMessage({ message }) {
  const eventId = message.key.toString();
  if (processedEvents.has(eventId)) {
    console.log(`Skipping duplicate event: ${eventId}`);
    return;
  }
  processedEvents.add(eventId);
  // Process the event
}
```
