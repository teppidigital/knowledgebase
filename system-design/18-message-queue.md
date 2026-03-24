# Message Queue Pattern

## Category
Messaging, Decoupling, Reliability, Scalability

## Context

A Message Queue is a **point-to-point** asynchronous communication mechanism where a producer sends messages to a queue and a consumer reads and processes them. Unlike Pub/Sub (fan-out), each message in a queue is processed by **exactly one consumer** (in a consumer group). This enables work distribution, load leveling, and decoupling of producers from consumers.

Common queue systems: RabbitMQ, AWS SQS, Redis Lists/Streams, Azure Service Bus.

---

## Pros

- **Decoupling**: Producer and consumer don't need to run at the same time.
- **Load leveling**: Queue buffers bursts of work; consumers process at their own pace.
- **Reliability**: Messages persist until acknowledged — no work is lost even if a consumer crashes.
- **Work distribution**: Multiple consumers share the queue workload (competing consumers).
- **Retry support**: Failed messages can be re-queued or sent to a Dead Letter Queue (DLQ).
- **Backpressure**: Natural backpressure through queue depth visibility.

---

## Cons

- **Increased latency**: Asynchronous processing adds delay vs. synchronous calls.
- **Ordering challenges**: Parallel consumers may process messages out of order.
- **Message duplication**: At-least-once delivery means consumers must be idempotent.
- **Queue management**: Requires monitoring queue depth, retention, and DLQ handling.
- **Debugging**: Tracing a message through a queue is harder than a direct call.

---

## Design Diagram

```mermaid
graph LR
    P1["Producer 1<br/>(Order Service)"]
    P2["Producer 2<br/>(Upload Service)"]

    Q["Message Queue<br/>(RabbitMQ / SQS)"]
    DLQ["Dead Letter Queue<br/>(DLQ)"]

    C1["Consumer 1<br/>(Worker)"]
    C2["Consumer 2<br/>(Worker)"]
    C3["Consumer 3<br/>(Worker)"]

    P1 -->|"Enqueue job"| Q
    P2 -->|"Enqueue job"| Q

    Q -->|"Deliver (round-robin)"| C1
    Q -->|"Deliver"| C2
    Q -->|"Deliver"| C3

    C1 -->|"NAck after retries"| DLQ
```

---

## Code Sample

### Producer (Node.js / RabbitMQ with amqplib)

```javascript
// producer/job.producer.js
const amqp = require('amqplib');

const QUEUE = 'email-jobs';

async function sendEmailJob(job) {
  const connection = await amqp.connect(process.env.RABBITMQ_URL);
  const channel = await connection.createChannel();

  await channel.assertQueue(QUEUE, {
    durable: true,           // Survive broker restart
    deadLetterExchange: 'dlx', // Route failures to DLQ
    deadLetterRoutingKey: 'email-jobs.dlq',
  });

  channel.sendToQueue(
    QUEUE,
    Buffer.from(JSON.stringify(job)),
    { persistent: true }     // Message persists to disk
  );

  console.log(`[Producer] Queued job: ${JSON.stringify(job)}`);
  await channel.close();
  await connection.close();
}

sendEmailJob({ to: 'user@example.com', template: 'order-confirmation', orderId: 'order-1' });
```

### Consumer (Node.js / RabbitMQ — competing consumers)

```javascript
// consumer/job.consumer.js
const amqp = require('amqplib');

const QUEUE = 'email-jobs';
const MAX_RETRIES = 3;

async function startWorker() {
  const connection = await amqp.connect(process.env.RABBITMQ_URL);
  const channel = await connection.createChannel();

  await channel.assertQueue(QUEUE, { durable: true });
  channel.prefetch(5); // Process max 5 messages concurrently per worker

  console.log(`[Worker] Waiting for jobs on ${QUEUE}`);

  channel.consume(QUEUE, async (msg) => {
    if (!msg) return;

    const job = JSON.parse(msg.content.toString());
    const retries = (msg.properties.headers['x-retry-count'] ?? 0);

    try {
      await processEmailJob(job);
      channel.ack(msg); // Acknowledge — remove from queue
    } catch (err) {
      console.error(`[Worker] Failed to process job: ${err.message}`);
      if (retries < MAX_RETRIES) {
        // Requeue with retry count
        channel.nack(msg, false, false); // Send to DLQ for retry logic
      } else {
        channel.nack(msg, false, false); // Final failure → DLQ
      }
    }
  });
}

async function processEmailJob(job) {
  console.log(`Sending ${job.template} email to ${job.to}`);
  // Call email service
}

startWorker().catch(console.error);
```

### AWS SQS Queue (Node.js / AWS SDK v3)

```javascript
// sqs/sqs-producer.js
const { SQSClient, SendMessageCommand } = require('@aws-sdk/client-sqs');

const sqs = new SQSClient({ region: 'us-east-1' });
const QUEUE_URL = process.env.SQS_QUEUE_URL;

async function enqueueJob(job) {
  await sqs.send(new SendMessageCommand({
    QueueUrl: QUEUE_URL,
    MessageBody: JSON.stringify(job),
    MessageGroupId: job.userId,         // FIFO queue: group by user
    MessageDeduplicationId: job.jobId,  // Prevent duplicates
  }));
}

// sqs/sqs-consumer.js
const { ReceiveMessageCommand, DeleteMessageCommand } = require('@aws-sdk/client-sqs');

async function pollQueue() {
  while (true) {
    const { Messages } = await sqs.send(new ReceiveMessageCommand({
      QueueUrl: QUEUE_URL,
      MaxNumberOfMessages: 10,
      WaitTimeSeconds: 20, // Long polling
    }));

    for (const msg of Messages ?? []) {
      const job = JSON.parse(msg.Body);
      await processJob(job);
      // Delete after successful processing
      await sqs.send(new DeleteMessageCommand({
        QueueUrl: QUEUE_URL,
        ReceiptHandle: msg.ReceiptHandle,
      }));
    }
  }
}

pollQueue().catch(console.error);
```
