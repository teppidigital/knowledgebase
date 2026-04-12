# Apache Kafka — Patterns & Practices

A comprehensive catalogue of 18 production-ready patterns for building reliable, scalable, and observable event-streaming systems with Apache Kafka.

---

## Pattern Index

| # | Pattern | Key Concepts |
|---|---------|-------------|
| 01 | [Kafka Architecture & Core Concepts](01-kafka-architecture.md) | Brokers, ZooKeeper/KRaft, ISR, leader election, log segments |
| 02 | [Topics, Partitions & Offsets](02-topics-partitions-offsets.md) | Partition strategy, key-based routing, retention, compaction |
| 03 | [Producers — Configuration & Reliability](03-producers-reliability.md) | `acks`, `retries`, idempotent producer, batching, compression |
| 04 | [Consumer Groups & Offset Management](04-consumer-groups.md) | Group rebalance, cooperative sticky, manual commit, lag monitoring |
| 05 | [Kafka Streams — Stream Processing](05-kafka-streams.md) | KStream, KTable, GlobalKTable, windowing, interactive queries |
| 06 | [Kafka Connect — Data Integration](06-kafka-connect.md) | Source/sink connectors, SMTs, Debezium CDC, connector config |
| 07 | [Schema Registry & Avro / Protobuf](07-schema-registry.md) | Confluent Schema Registry, compatibility modes, schema evolution |
| 08 | [Exactly-Once Semantics (EOS)](08-exactly-once-semantics.md) | Idempotent producer, transactions, `read_committed` isolation |
| 09 | [Kafka Security](09-kafka-security.md) | TLS encryption, SASL/SCRAM, ACLs, mTLS, secrets rotation |
| 10 | [Kafka Observability & Monitoring](10-kafka-observability.md) | JMX metrics, Prometheus JMX Exporter, Grafana dashboards, consumer lag |
| 11 | [Performance Tuning](11-performance-tuning.md) | Throughput vs latency, batch size, linger.ms, compression, page cache |
| 12 | [Kafka on Kubernetes — Strimzi](12-kafka-kubernetes-strimzi.md) | Strimzi operator, KafkaTopic CRDs, rolling updates, PodDisruptionBudget |
| 13 | [Event-Driven Architecture with Kafka](13-event-driven-architecture.md) | Domain events, choreography, saga pattern, event storming |
| 14 | [CQRS & Event Sourcing](14-cqrs-event-sourcing.md) | Command/query split, event store, projections, outbox pattern |
| 15 | [Multi-Cluster & MirrorMaker 2](15-multi-cluster-mirrormaker.md) | Active-active, active-passive, MM2 config, offset translation |
| 16 | [Dead Letter Queue (DLQ) Patterns](16-dead-letter-queue.md) | Poison-pill handling, retry topics, DLQ routing, alerting |
| 17 | [Compacted Topics & State Stores](17-compacted-topics.md) | Log compaction, changelog topics, KTable materialization |
| 18 | [Kafka Testing Strategies](18-kafka-testing.md) | EmbeddedKafka, Testcontainers, TopologyTestDriver, contract tests |

---

## Decision Guide

### Choosing a Delivery Guarantee

| Requirement | Setting | Trade-off |
|-------------|---------|-----------|
| Best-effort / max throughput | `acks=0` | Data loss possible |
| Leader ack only | `acks=1` | Loss on leader failure |
| All replicas ack | `acks=all` + `min.insync.replicas=2` | Higher latency |
| No duplicates + no loss | EOS (`transactional.id`) | Lower throughput |

### Choosing a Processing Model

| Model | Tool | Best For |
|-------|------|----------|
| Simple consume + process | KafkaJS / kafka-python consumer | Stateless event handlers |
| Stateful stream processing | Kafka Streams / Flink | Aggregations, joins, windowing |
| Data pipeline / CDC | Kafka Connect + Debezium | DB-to-Kafka, Kafka-to-DB |
| Complex multi-step workflows | Saga + choreography | Distributed transactions |

### Partition Count Guidelines

| Throughput Target | Recommended Partitions | Notes |
|-------------------|----------------------|-------|
| < 10 MB/s per topic | 3–6 | Dev / low-volume |
| 10–100 MB/s | 12–24 | Standard production |
| > 100 MB/s | 48–96+ | High-throughput; benchmark first |

---

## Categories

### Foundations
- **Architecture** — Understand brokers, KRaft, ISR, and log storage
- **Topics & Partitions** — Design your partition strategy and retention policy

### Reliability & Correctness
- **Producers** — Tune `acks`, `retries`, and idempotency for your SLA
- **EOS** — Exactly-once end-to-end with Kafka transactions

### Consumers
- **Consumer Groups** — Cooperative rebalance, lag monitoring, manual offsets

### Stream Processing
- **Kafka Streams** — Embedded JVM library for stateful processing
- **Compacted Topics** — Materialise current state from event log

### Data Integration
- **Kafka Connect** — Plug-and-play connectors for 200+ systems
- **Schema Registry** — Schema evolution without breaking consumers

### Operations
- **Security** — Encrypt, authenticate, and authorise every connection
- **Observability** — Metrics, dashboards, and lag alerts
- **Performance Tuning** — Squeeze throughput or minimise latency
- **Kubernetes / Strimzi** — Cloud-native Kafka operations

### Architecture Patterns
- **Event-Driven Architecture** — Domain events, choreography, and sagas
- **CQRS & Event Sourcing** — Command/query split and event store
- **Multi-Cluster / MirrorMaker 2** — Geo-replication and disaster recovery
- **Dead Letter Queues** — Resilient error handling for poison pills

### Quality Engineering
- **Testing** — Unit, integration, and contract testing for Kafka consumers and streams
