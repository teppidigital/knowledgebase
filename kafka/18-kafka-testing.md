# Kafka Testing Strategies

## Category

Apache Kafka — Quality Engineering

## Context

Testing Kafka-based systems requires strategies across four levels: **unit** (topology logic without a broker), **integration** (real broker with Testcontainers), **contract** (schema and API compatibility), and **end-to-end** (full pipeline in staging or CI). Each level has different speed, fidelity, and infrastructure requirements.

### Test Strategy Pyramid

| Level | Tool | Speed | Fidelity | Infrastructure |
|-------|------|-------|----------|---------------|
| **Unit — Topology** | Kafka Streams `TopologyTestDriver` | Very fast | Business logic only | None |
| **Unit — Consumer/Producer** | In-memory mocks / Jest | Fast | Schema + serialisation | None |
| **Integration** | Testcontainers + real Kafka | Medium | Real broker | Docker |
| **Contract** | AsyncAPI validator, schema compatibility | Fast | Schema evolution | Schema Registry |
| **End-to-end** | Real cluster (staging) | Slow | Full fidelity | Full infra |

### Testcontainers Kafka

`@testcontainers/kafka` starts a real Kafka broker (Docker) for integration tests. Preferred over embedded broker mocks for validating actual producer/consumer behaviour, TLS config, and ACL enforcement.

### TopologyTestDriver (Kafka Streams)

Allows testing a Streams topology entirely in memory — no broker required. You pipe records in through `TestInputTopic` and read from `TestOutputTopic`. Supports wall-clock time advancement for windowing tests.

## Pros

- `TopologyTestDriver` tests run in milliseconds — include in every commit
- Testcontainers provides identical broker behaviour to production — catches config bugs
- Schema compatibility tests in CI prevent breaking consumer deployments
- Isolated consumer group IDs per test prevent cross-test pollution
- Consumer mock pattern separates business logic from Kafka client lifecycle concerns

## Cons

- `TopologyTestDriver` does not test real serialisation/deserialisation or network behaviour  
- Testcontainers requires Docker in CI — adds ~30–60 s startup per test suite  
- Full end-to-end tests in staging are slow and require maintained test data pipelines  
- Consumer rebalance behaviour cannot be unit-tested without a real broker  
- Schema evolution testing requires a running Schema Registry instance  

## Design Diagram

```mermaid
flowchart TB
    subgraph Unit Tests — No broker
        UT1[TopologyTestDriver<br/>Kafka Streams logic]
        UT2[Jest / Vitest<br/>Producer serialisation<br/>Consumer handler logic]
    end

    subgraph Integration Tests — Docker broker
        TC[Testcontainers<br/>Kafka + Schema Registry]
        IT1[Producer → real topic → Consumer assertions]
        IT2[Kafka Connect SMT validation]
        TC --> IT1
        TC --> IT2
    end

    subgraph Contract / Schema Tests
        SC[Schema Registry<br/>compatibility check]
        AC[AsyncAPI validator<br/>vs canonical spec]
    end

    subgraph CI Pipeline
        UT1 & UT2 -->|fast gate| GATE1{Pass?}
        IT1 & IT2 -->|medium gate| GATE2{Pass?}
        SC & AC -->|schema gate| GATE3{Pass?}
        GATE1 & GATE2 & GATE3 --> DEPLOY[Deploy to staging]
    end
```

## Code Sample

### Java — Kafka Streams topology unit test with `TopologyTestDriver`

```java
import org.apache.kafka.streams.*;
import org.apache.kafka.streams.test.*;
import org.apache.kafka.common.serialization.Serdes;
import org.junit.jupiter.api.*;
import static org.assertj.core.api.Assertions.*;

class PaymentEnrichmentTopologyTest {

    private TopologyTestDriver driver;
    private TestInputTopic<String, String> inputTopic;
    private TestOutputTopic<String, String> outputTopic;

    @BeforeEach
    void setUp() {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "test");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "dummy:9092");
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());

        driver = new TopologyTestDriver(PaymentEnrichmentTopology.build(), props);

        inputTopic = driver.createInputTopic(
            "payments.created",
            Serdes.String().serializer(),
            Serdes.String().serializer()
        );
        outputTopic = driver.createOutputTopic(
            "payments.enriched.large",
            Serdes.String().deserializer(),
            Serdes.String().deserializer()
        );
    }

    @AfterEach
    void tearDown() { driver.close(); }

    @Test
    void highValuePaymentIsEmittedToOutputTopic() {
        inputTopic.pipeInput("account-1", """
            {"paymentId":"pay-1","accountId":"account-1","amount":2500.00,"currency":"GBP"}
        """);

        assertThat(outputTopic.isEmpty()).isFalse();
        var record = outputTopic.readKeyValue();
        assertThat(record.key).isEqualTo("account-1");
        assertThat(record.value).contains("pay-1");
    }

    @Test
    void lowValuePaymentIsFilteredOut() {
        inputTopic.pipeInput("account-2", """
            {"paymentId":"pay-2","accountId":"account-2","amount":50.00,"currency":"GBP"}
        """);

        assertThat(outputTopic.isEmpty()).isTrue();
    }

    @Test
    void windowedAggregationAccumulatesWithinWindow() {
        Instant start = Instant.parse("2026-01-01T10:00:00Z");
        inputTopic.pipeInput(new TestRecord<>("account-3",
            """{"amount":1500.00}""", start));
        inputTopic.pipeInput(new TestRecord<>("account-3",
            """{"amount":2000.00}""", start.plusSeconds(30)));

        var records = outputTopic.readKeyValuesToList();
        assertThat(records).hasSize(2);
        // Last record in window should reflect cumulative sum
    }
}
```

### TypeScript — Integration test with Testcontainers (KafkaJS)

```typescript
import { afterAll, beforeAll, describe, expect, it } from 'vitest';
import { KafkaContainer, StartedKafkaContainer } from '@testcontainers/kafka';
import { Kafka } from 'kafkajs';
import crypto from 'node:crypto';

describe('Payment Producer + Consumer integration', () => {
  let container: StartedKafkaContainer;
  let brokers: string[];

  beforeAll(async () => {
    container = await new KafkaContainer('apache/kafka:3.7.0').start();
    brokers = [container.getBootstrapServers()];
  }, 120_000);

  afterAll(async () => {
    await container.stop();
  });

  it('published payment is consumed with correct schema', async () => {
    const kafka = new Kafka({ clientId: 'test', brokers });
    const admin = kafka.admin();
    const producer = kafka.producer({ idempotent: true });
    const consumer = kafka.consumer({ groupId: `test-${crypto.randomUUID()}` });

    await admin.connect();
    await admin.createTopics({
      topics: [{ topic: 'payments.test', numPartitions: 1, replicationFactor: 1 }],
    });
    await admin.disconnect();

    const event = {
      paymentId: crypto.randomUUID(),
      accountId: 'account-123',
      amount: 500,
      currency: 'GBP',
    };

    await producer.connect();
    await producer.send({
      topic: 'payments.test',
      messages: [{ key: event.accountId, value: JSON.stringify(event) }],
    });
    await producer.disconnect();

    const received = await new Promise<Record<string, unknown>>((resolve, reject) => {
      const timeout = setTimeout(() => reject(new Error('Timeout waiting for message')), 15_000);

      consumer.connect().then(() =>
        consumer.subscribe({ topic: 'payments.test', fromBeginning: true }).then(() =>
          consumer.run({
            eachMessage: async ({ message }) => {
              clearTimeout(timeout);
              resolve(JSON.parse(message.value!.toString()));
              await consumer.disconnect();
            },
          }),
        ),
      );
    });

    expect(received.paymentId).toBe(event.paymentId);
    expect(received.currency).toBe('GBP');
  }, 30_000);
});
```

### TypeScript — Unit test for consumer handler business logic (no broker)

```typescript
import { describe, expect, it, vi } from 'vitest';
import { handlePaymentCreated } from './payment-consumer';

describe('handlePaymentCreated', () => {
  it('triggers notification for high-value payment', async () => {
    const mockNotify = vi.fn().mockResolvedValue(undefined);

    await handlePaymentCreated(
      { paymentId: 'p1', accountId: 'a1', amount: 10_000, currency: 'GBP' },
      mockNotify,
    );

    expect(mockNotify).toHaveBeenCalledWith(
      expect.objectContaining({ paymentId: 'p1' }),
    );
  });

  it('does not notify for low-value payment', async () => {
    const mockNotify = vi.fn();

    await handlePaymentCreated(
      { paymentId: 'p2', accountId: 'a2', amount: 50, currency: 'GBP' },
      mockNotify,
    );

    expect(mockNotify).not.toHaveBeenCalled();
  });
});
```

### Shell — Schema compatibility CI check

```bash
#!/usr/bin/env bash
# In CI: check that a candidate schema is BACKWARD_TRANSITIVE compatible

SR="${SCHEMA_REGISTRY_URL}"
SUBJECT="payments.created-value"
CANDIDATE=$(cat ./schemas/PaymentCreated.avsc | jq -Rs .)

RESULT=$(curl -s -o /dev/null -w "%{http_code}" \
  -X POST \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  -u "$SR_API_KEY:$SR_API_SECRET" \
  --data "{\"schema\": $CANDIDATE}" \
  "$SR/compatibility/subjects/$SUBJECT/versions/latest")

if [[ "$RESULT" == "200" ]]; then
  echo "Schema is compatible"
  exit 0
else
  echo "Schema compatibility check FAILED (HTTP $RESULT)"
  exit 1
fi
```

## References

- [Kafka Streams TopologyTestDriver](https://kafka.apache.org/35/documentation/streams/developer-guide/testing.html)
- [Testcontainers Kafka Module (Java)](https://java.testcontainers.org/modules/kafka/)
- [@testcontainers/kafka (Node.js)](https://node.testcontainers.org/modules/kafka/)
- [KafkaJS Testing Guide](https://kafka.js.org/docs/testing)
- [Schema Registry Compatibility API](https://docs.confluent.io/platform/current/schema-registry/develop/api.html#compatibility)
