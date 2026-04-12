# Encoding and Evolution

## Category

DDIA — Foundations (Chapter 4)

## Context

Data must be **encoded** (serialised) when stored to disk, sent over a network, or passed between processes. The encoding format determines how easily systems can **evolve** — adding fields, removing fields, changing types — without breaking compatibility.

In any large system, different versions of code run simultaneously: rolling deployments mean old and new servers receive the same messages; old clients talk to new services. The encoding format must support this.

### Forward and Backward Compatibility

| Term | Definition | Example |
|---|---|---|
| **Backward compatibility** | New code can read data written by old code | New service reads old data from the DB without crashing |
| **Forward compatibility** | Old code can read data written by new code | Old service reads a message from a new service (ignores unknown fields) |

Both are required for safe rolling deployments and independent service upgrades.

### Encoding Format Comparison

| Format | Type | Schema required | Human-readable | Size | Versioning |
|---|---|---|---|---|---|
| **JSON** | Text | No (schema-on-read) | Yes | Large | Manual (field names in payload) |
| **XML** | Text | Optional (XSD) | Yes | Very large | Manual |
| **CSV** | Text | No | Yes | Medium | Fragile (column order) |
| **MessagePack** | Binary | No | No | Small | Same as JSON issues |
| **Thrift** | Binary | Yes (IDL) | No | Small | Field tags enable evolution |
| **Protocol Buffers (Protobuf)** | Binary | Yes (.proto) | No | Very small | Field numbers enable evolution |
| **Avro** | Binary | Yes (JSON schema) | No | Smallest | Writer + reader schema diff |
| **Parquet / ORC** | Binary columnar | Yes | No | Smallest (compressed) | Schema evolution at file level |

## Pros

- Schema-based binary formats (Protobuf, Avro, Thrift) produce 3–10× smaller payloads than JSON
- Schema serves as documentation, enforced — prevents the "what does this field mean?" problem
- Formal versioning support (field tags in Protobuf, writer/reader schema in Avro) enables safe evolution
- Generated code (from `.proto`, Avro schema) removes hand-coded serialisation bugs

## Cons

- Binary formats are not human-readable — debugging requires tooling
- Schema registry (for Avro/Protobuf at scale) is an additional infrastructure component
- Schema evolution rules must be understood by all teams — easy to accidentally break compatibility
- JSON is a lingua franca: inter-company APIs almost always use JSON even when it's not optimal

## Design Diagram

```mermaid
flowchart TD
    subgraph Schema Evolution — Protobuf Rules
        ADD[Add new field\nOptional with new tag\n✅ Backward + Forward compatible]
        REM[Remove field?\nMust keep tag reserved\nOld code ignores; new code uses default]
        CHANGE[Change field type?\nOnly safe type promotions\ne.g. int32 → int64]
        RENAME[Rename field?\nTag number unchanged = safe\nTag changed = breaking]
    end

    subgraph Avro Schema Resolution
        WRITER[Writer schema\nat write time]
        READER[Reader schema\nat read time]
        COMPAT[Schema registry resolves\nwriter → reader\nField matching by name]
        WRITER --> COMPAT
        READER --> COMPAT
    end
```

## Code Sample

### Protocol Buffers — schema and TypeScript usage

```protobuf
// payment.proto
syntax = "proto3";
package payments;

message PaymentIntent {
  string id = 1;
  int64 amount_cents = 2;
  string currency = 3;
  string customer_id = 4;
  PaymentStatus status = 5;
  // Field 6 was 'legacy_reference' — now RESERVED (never reuse tag 6)
  reserved 6;
  reserved "legacy_reference";
  // Added in v2 — backward compatible (old clients ignore this field)
  string idempotency_key = 7;
  repeated string tags = 8;
}

enum PaymentStatus {
  PAYMENT_STATUS_UNSPECIFIED = 0;  // proto3: always have a zero default
  PENDING = 1;
  SUCCEEDED = 2;
  FAILED = 3;
}
```

```typescript
// Generated types from protoc + ts-proto
import { PaymentIntent, PaymentStatus } from './generated/payment';

// Serialise
const intent: PaymentIntent = {
  id: 'pi_123',
  amountCents: BigInt(1000),
  currency: 'GBP',
  customerId: 'cust_456',
  status: PaymentStatus.PENDING,
  idempotencyKey: 'order_789',
  tags: ['subscription', 'renewal'],
};
const bytes = PaymentIntent.encode(intent).finish();
console.log('Protobuf size:', bytes.length, 'bytes'); // ~50 bytes vs ~200 bytes JSON

// Deserialise
const decoded = PaymentIntent.decode(bytes);
```

### Avro — schema registry pattern

```typescript
import { SchemaRegistry } from '@kafkajs/confluent-schema-registry';
import { Kafka } from 'kafkajs';

const registry = new SchemaRegistry({ host: process.env.SCHEMA_REGISTRY_URL! });
const kafka = new Kafka({ brokers: [process.env.KAFKA_BROKERS!] });

// Avro schema (JSON format)
const paymentSchema = {
  type: 'record',
  name: 'PaymentEvent',
  namespace: 'com.company.payments',
  fields: [
    { name: 'id', type: 'string' },
    { name: 'amountCents', type: 'long' },
    { name: 'currency', type: 'string' },
    // New field — safe with default value (forward + backward compatible)
    { name: 'idempotencyKey', type: ['null', 'string'], default: null },
  ],
};

// Register schema (idempotent)
const { id: schemaId } = await registry.register({ type: 'AVRO', schema: JSON.stringify(paymentSchema) });

// Produce — encoded with schema ID as prefix (Confluent wire format)
const producer = kafka.producer();
await producer.connect();
await producer.send({
  topic: 'payments.events',
  messages: [{
    key: 'pi_123',
    value: await registry.encode(schemaId, {
      id: 'pi_123', amountCents: 1000, currency: 'GBP', idempotencyKey: 'order_789',
    }),
  }],
});

// Consume — schema ID in message prefix → registry fetches writer schema
const consumer = kafka.consumer({ groupId: 'payment-processor' });
await consumer.connect();
await consumer.subscribe({ topic: 'payments.events' });
await consumer.run({
  eachMessage: async ({ message }) => {
    const event = await registry.decode(message.value!);
    // Old consumer reading new message: idempotencyKey field present, unknown → ignored by old schema
    console.log('Decoded:', event);
  },
});
```

### JSON — safe evolution patterns

```typescript
// Schema-on-read: handle missing fields explicitly; never trust field presence

interface PaymentEventV1 { id: string; amount: number; currency: string }
interface PaymentEventV2 extends PaymentEventV1 { idempotencyKey?: string; tags?: string[] }

// Parser that handles both old and new versions
function parsePaymentEvent(raw: unknown): PaymentEventV2 {
  if (typeof raw !== 'object' || raw === null) throw new Error('Not an object');
  const obj = raw as Record<string, unknown>;

  // Required fields — throw if missing (both versions have them)
  if (typeof obj.id !== 'string') throw new Error('Missing id');
  if (typeof obj.amount !== 'number') throw new Error('Missing amount');
  if (typeof obj.currency !== 'string') throw new Error('Missing currency');

  return {
    id: obj.id,
    amount: obj.amount,
    currency: obj.currency,
    // Optional fields from v2 — old messages simply won't have them
    idempotencyKey: typeof obj.idempotencyKey === 'string' ? obj.idempotencyKey : undefined,
    tags: Array.isArray(obj.tags) ? obj.tags.filter(t => typeof t === 'string') : undefined,
  };
}
```

## Key Patterns

### Protobuf Evolution Rules

| Change | Backward compatible? | Forward compatible? |
|---|---|---|
| Add optional field (new tag number) | ✅ (old code ignores) | ✅ (new field absent = default) |
| Remove optional field (reserve tag) | ✅ (reserved = ignored) | ✅ |
| Add required field | ❌ (old code can't satisfy) | ❌ |
| Change field type (compatible, e.g. int32→int64) | ✅ | ✅ |
| Change field type (incompatible, e.g. string→int) | ❌ | ❌ |
| Rename field (keep same tag) | ✅ | ✅ |
| Reuse a removed tag number | ❌ NEVER — data corruption | ❌ |

### Dataflow Modes

How data flows between systems determines what compatibility guarantees are needed:

| Mode | Example | Compatibility required |
|---|---|---|
| **DB read/write** | App writes, different app version reads | Backward (new reads old data) |
| **REST API** | Client calls server | Backward (new server reads old request) |
| **Message queue** | Producer sends; consumer reads later | Both: producer may be newer or older than consumer |
| **RPC** | Direct service call | Both: rolling deployments create mixed versions |

### Versioning Strategy

| Strategy | How | When |
|---|---|---|
| **Field tags (Protobuf, Thrift)** | Schema file tracks tag → field mapping; old code ignores unknown tags | Preferred for high-performance internal services |
| **Schema registry (Avro/Confluent)** | Writer schema stored in registry; reader schema resolved at decode | Preferred for Kafka event buses; enforces compatibility |
| **URL versioning (REST)** | `/api/v1/payments` vs `/api/v2/payments` | External-facing APIs; breaking changes between versions |
| **Accept header versioning** | `Accept: application/vnd.company.v2+json` | REST APIs; cleaner URLs |
| **Sunset headers** | `Sunset: Sat, 01 Jan 2027 00:00:00 GMT` | Communicating deprecation timelines |

## Related Patterns

- [05 — Replication](./05-replication.md) — Replication logs encode data; format matters for compatibility
- [11 — Stream Processing](./11-stream-processing.md) — Kafka + Schema Registry is canonical Avro usage
- [12 — Derived Data Systems](./12-derived-data-systems.md) — Event log as source of truth depends on durable encoding
