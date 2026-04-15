# Message-Based Integration

## Category

System Integration — Asynchronous Messaging

## Context

Message-based integration decouples producers from consumers by placing a **message broker** between them. The producer publishes a message and forgets; the broker stores and routes it; the consumer reads at its own pace. This pattern is the backbone of resilient, independently-deployable distributed systems.

### Core Messaging Concepts

| Concept | Description |
|---------|-------------|
| **Message** | Discrete unit of data with headers (metadata) and body (payload) |
| **Queue** | Point-to-point channel — one producer, one consumer group (competing consumers) |
| **Topic / Exchange** | Fan-out channel — one message delivered to multiple subscribers |
| **Dead-Letter Queue (DLQ)** | Queue for messages that failed processing after N retries |
| **Acknowledgement (ACK)** | Consumer signals successful processing; unACKed messages are requeued |
| **Idempotency** | Consumer must handle duplicate delivery safely |

### Message Delivery Guarantees

| Guarantee | Behaviour | Risk |
|-----------|-----------|------|
| **At-most-once** | Fire and forget; message may be lost | Data loss |
| **At-least-once** | Redelivery on failure; duplicates possible | Duplicate processing |
| **Exactly-once** | Transactional producer + idempotent consumer | Highest complexity and cost |

### RabbitMQ vs Azure Service Bus vs SQS

| Feature | RabbitMQ | Azure Service Bus | AWS SQS |
|---------|---------|-----------------|--------|
| Protocol | AMQP 0-9-1 | AMQP 1.0 | HTTP |
| Routing | Exchange + binding | Subscription filters | SNS fan-out |
| Dead-letter | Native DLX | Native DLQ | Native DLQ |
| Message ordering | Per-queue | Sessions (FIFO) | FIFO queue |
| Max message size | 128 MB | 256 KB (1 MB premium) | 256 KB |
| Message TTL | Per-message or queue | Per-message or queue | Up to 14 days |
| Managed | Self-hosted / CloudAMQP | Fully managed | Fully managed |

## Pros

- Producers and consumers are fully decoupled — release independently
- Broker absorbs traffic spikes; consumers process at their own rate
- Competing consumers (multiple instances reading the same queue) give horizontal throughput
- DLQ captures poison messages for forensic inspection without losing them
- Message persistence survives producer/consumer restarts
- Protocol-agnostic: broker can bridge HTTP → AMQP → TCP

## Cons

- At-least-once semantics require idempotent consumers everywhere
- Eventual consistency — consumer may lag behind producer by seconds or more
- Broker is a new operational dependency (HA, capacity planning, monitoring)
- Debugging cross-service flows requires distributed tracing via correlation IDs
- Schema drift: if producer changes message shape, consumers must be updated or routing breaks
- Message size limits require offloading large payloads to object storage (Claim Check pattern)

## Design Diagram

```mermaid
flowchart LR
    P[Producer<br/>Payment Service] -->|publish| EX{Exchange<br/>Routing Rules}
    EX -->|route: payment.created| Q1[Queue: notifications]
    EX -->|route: payment.created| Q2[Queue: ledger-sync]
    EX -->|route: payment.failed| Q3[Queue: alerts]

    Q1 --> C1[Consumer: Notification<br/>Service × 3]
    Q2 --> C2[Consumer: Ledger<br/>Service × 2]
    Q3 --> C3[Consumer: Alert<br/>Service]

    Q1 -->|after 3 retries| DLQ1[(DLQ: notifications.dlq)]
    Q2 -->|after 3 retries| DLQ2[(DLQ: ledger-sync.dlq)]

    subgraph Broker [RabbitMQ Broker]
        EX
        Q1
        Q2
        Q3
    end
```

## Code Sample

### TypeScript — RabbitMQ producer + consumer with DLQ and idempotency

```typescript
// messaging/rabbitmq-client.ts
import amqp, { Channel, Connection } from 'amqplib';
import { createHash } from 'crypto';

// ── Types ─────────────────────────────────────────────────────────────────────
export interface MessageEnvelope<T = unknown> {
  messageId: string;         // idempotency key — producer sets this
  correlationId: string;     // trace the flow across services
  type: string;              // e.g. "payment.created"
  payload: T;
  timestamp: string;
  retryCount?: number;
}

// ── Exchange / Queue topology ─────────────────────────────────────────────────
const EXCHANGE     = 'payments';
const QUEUE        = 'payments.notifications';
const DLQ_EXCHANGE = 'payments.dlx';
const DLQ_QUEUE    = 'payments.notifications.dlq';

// ── Publisher ─────────────────────────────────────────────────────────────────
export class MessagePublisher {
  private channel!: Channel;

  async connect(url: string): Promise<void> {
    const conn: Connection = await amqp.connect(url);
    this.channel = await conn.createChannel();

    // Declare exchange (topic routing)
    await this.channel.assertExchange(EXCHANGE, 'topic', { durable: true });
  }

  async publish<T>(type: string, payload: T, correlationId: string): Promise<void> {
    const envelope: MessageEnvelope<T> = {
      messageId: createHash('sha256')
        .update(`${type}:${JSON.stringify(payload)}`)
        .digest('hex')
        .slice(0, 16),
      correlationId,
      type,
      payload,
      timestamp: new Date().toISOString(),
    };

    const ok = this.channel.publish(
      EXCHANGE,
      type,                                // routing key = event type
      Buffer.from(JSON.stringify(envelope)),
      {
        persistent: true,                  // survive broker restart
        messageId: envelope.messageId,
        correlationId,
        contentType: 'application/json',
        headers: { 'x-source-service': 'payment-service' },
      },
    );

    if (!ok) throw new Error('Channel write buffer full — apply backpressure');
    console.log(`[publisher] sent ${type} [${envelope.messageId}]`);
  }
}

// ── Consumer with at-least-once + idempotency ─────────────────────────────────
export class MessageConsumer {
  private channel!: Channel;
  private processed = new Set<string>();  // in production: Redis with TTL

  async connect(url: string): Promise<void> {
    const conn: Connection = await amqp.connect(url);
    this.channel = await conn.createChannel();
    await this.channel.prefetch(10);       // max 10 unACKed messages in-flight

    // DLQ infrastructure
    await this.channel.assertExchange(DLQ_EXCHANGE, 'direct', { durable: true });
    await this.channel.assertQueue(DLQ_QUEUE, { durable: true });
    await this.channel.bindQueue(DLQ_QUEUE, DLQ_EXCHANGE, QUEUE);

    // Main queue — messages rejected after x-death go to DLQ
    await this.channel.assertExchange(EXCHANGE, 'topic', { durable: true });
    await this.channel.assertQueue(QUEUE, {
      durable: true,
      arguments: {
        'x-dead-letter-exchange': DLQ_EXCHANGE,
        'x-dead-letter-routing-key': QUEUE,
        'x-message-ttl': 60_000,           // 60s — unprocessed messages → DLQ
      },
    });
    await this.channel.bindQueue(QUEUE, EXCHANGE, 'payment.*');
  }

  async consume(handler: (envelope: MessageEnvelope) => Promise<void>): Promise<void> {
    await this.channel.consume(QUEUE, async (msg) => {
      if (!msg) return;

      const envelope: MessageEnvelope = JSON.parse(msg.content.toString());

      // Idempotency guard — skip if already processed
      if (this.processed.has(envelope.messageId)) {
        this.channel.ack(msg);
        console.log(`[consumer] duplicate skipped: ${envelope.messageId}`);
        return;
      }

      try {
        await handler(envelope);
        this.processed.add(envelope.messageId);
        this.channel.ack(msg);             // ✅ success — remove from queue
      } catch (err) {
        const retries = (msg.properties.headers?.['x-death']?.[0]?.count ?? 0) as number;
        if (retries >= 3) {
          // Send to DLQ after 3 failures
          this.channel.nack(msg, false, false);
          console.error(`[consumer] DLQ after ${retries} retries: ${envelope.messageId}`, err);
        } else {
          // Requeue for retry
          this.channel.nack(msg, false, true);
          console.warn(`[consumer] retry ${retries + 1}: ${envelope.messageId}`);
        }
      }
    });
  }
}

// ── Wire-up example ───────────────────────────────────────────────────────────
const AMQP_URL = process.env.AMQP_URL ?? 'amqp://guest:guest@localhost:5672';

const publisher = new MessagePublisher();
await publisher.connect(AMQP_URL);

const consumer = new MessageConsumer();
await consumer.connect(AMQP_URL);

await consumer.consume(async (envelope) => {
  console.log('[handler] processing:', envelope.type, envelope.payload);
  // Business logic here
});

await publisher.publish(
  'payment.created',
  { id: 'pay_001', amount: 99.99, currency: 'GBP' },
  'corr-abc-123',
);
```

### YAML — RabbitMQ topology via `rabbitmq-definitions.json` (infrastructure as code)

```json
{
  "exchanges": [
    { "name": "payments",     "type": "topic",  "durable": true, "auto_delete": false },
    { "name": "payments.dlx", "type": "direct", "durable": true, "auto_delete": false }
  ],
  "queues": [
    {
      "name": "payments.notifications",
      "durable": true,
      "arguments": {
        "x-dead-letter-exchange": "payments.dlx",
        "x-dead-letter-routing-key": "payments.notifications",
        "x-message-ttl": 60000
      }
    },
    { "name": "payments.notifications.dlq", "durable": true, "arguments": {} }
  ],
  "bindings": [
    {
      "source": "payments",
      "destination": "payments.notifications",
      "destination_type": "queue",
      "routing_key": "payment.*"
    },
    {
      "source": "payments.dlx",
      "destination": "payments.notifications.dlq",
      "destination_type": "queue",
      "routing_key": "payments.notifications"
    }
  ]
}
```

## References

- [RabbitMQ — AMQP Concepts](https://www.rabbitmq.com/amqp-concepts.html)
- [Enterprise Integration Patterns — Message Channel](https://www.enterpriseintegrationpatterns.com/MessageChannel.html)
- [AWS SQS Dead-Letter Queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)
- [Azure Service Bus — Dead-lettering](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-dead-letter-message-queues)
- [Claim Check Pattern](https://www.enterpriseintegrationpatterns.com/StoreInLibrary.html)
