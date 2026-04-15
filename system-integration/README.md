# System Integration Patterns

A comprehensive reference of 15 system integration patterns covering integration styles, messaging, event streaming, CDC, distributed transactions, B2B protocols, testing, observability, and security — for connecting heterogeneous systems reliably in modern enterprise and cloud-native environments.

---

## Pattern Index

| # | Pattern | Key Concepts |
|---|---------|-------------|
| 01 | [Integration Styles](01-integration-styles.md) | P2P, Hub-and-Spoke, Message Bus, ESB, iPaaS, event mesh — trade-off comparison |
| 02 | [Message-Based Integration](02-message-based-integration.md) | AMQP, RabbitMQ, competing consumers, dead-letter, routing, message envelope |
| 03 | [Event-Driven Integration](03-event-driven-integration.md) | Kafka, CloudEvents, event mesh, event sourcing as integration, pub/sub vs queue |
| 04 | [Synchronous HTTP Integration](04-sync-http-integration.md) | Circuit breaker, retry + backoff, bulkhead, timeout chain, resilience4j |
| 05 | [File & Batch Integration](05-file-batch-integration.md) | SFTP polling, AS2 transport, file routing, idempotent file processing, Spring Batch |
| 06 | [Change Data Capture (CDC)](06-change-data-capture.md) | Debezium, WAL-based CDC, Outbox pattern, dual-write hazard, exactly-once semantics |
| 07 | [Anti-Corruption Layer (ACL)](07-anti-corruption-layer.md) | Domain translation, adapter, façade, model mapping between bounded contexts |
| 08 | [Canonical Data Model](08-canonical-data-model.md) | Universal schema, transformer chain, JSON Schema registry, versioned model |
| 09 | [Saga Pattern](09-saga-pattern.md) | Choreography vs orchestration, compensating transactions, rollback chain |
| 10 | [Strangler Fig Integration](10-strangler-fig-integration.md) | Façade routing, feature parity tracking, dark launch, traffic cutover |
| 11 | [B2B Integration](11-b2b-integration.md) | EDI/EDIFACT, ISO 20022, SWIFT, AS2/AS4, PEPPOL, message acknowledgement |
| 12 | [iPaaS & Integration Platforms](12-ipaas-integration-platform.md) | Apache Camel, MuleSoft, Temporal, Azure Logic Apps, low-code vs code-first |
| 13 | [Integration Testing](13-integration-testing.md) | Consumer-driven contracts (Pact), service virtualisation (WireMock), CDC testing |
| 14 | [Integration Observability](14-integration-observability.md) | Correlation IDs, W3C Trace Context, dead-letter monitoring, integration health SLOs |
| 15 | [Integration Security](15-integration-security.md) | mTLS, OAuth2 client credentials (M2M), API key rotation, IP allowlisting, payload signing |

---

## Decision Guide

### Which integration style should I use?

```
Is the target system under your control?
  ├── No (third-party / SaaS) → API-led or file/B2B (patterns 04, 05, 11)
  └── Yes → Is the interaction synchronous (caller blocks for response)?
              ├── Yes → Synchronous HTTP (pattern 04)
              └── No → Is ordering / replay required?
                        ├── Yes → Event streaming / Kafka (pattern 03)
                        └── No → Message queue / RabbitMQ (pattern 02)

Do many services need to connect to many others?
  ├── N-to-N mesh → Message Bus or ESB (pattern 01)
  └── Clear producers and consumers → Pub/Sub (pattern 03)

Is the integration crossing organisational boundaries?
  └── Yes → B2B protocols (EDI, AS2, ISO 20022) (pattern 11)
```

### How do I handle cross-service data consistency?

```
Immediate strong consistency required?
  └── Avoid distributed transactions if possible. Consider event sourcing + projection.

Single business process spans multiple services?
  ├── < 3 steps, simple rollback → Choreography Saga (pattern 09)
  └── Complex branching or long-running → Orchestration Saga / Temporal (patterns 09, 12)

Sync DB to another system in near-real-time?
  └── Change Data Capture + Outbox (pattern 06)
```

### How do I integrate a legacy system without rewriting it?

```
Legacy has a stable API surface → Wrap with Anti-Corruption Layer (pattern 07)
Legacy owns valuable data → CDC to stream data out (pattern 06)
Replacing legacy incrementally → Strangler Fig (pattern 10)
Legacy speaks EDI / SWIFT → B2B adapter (pattern 11)
```

### How do I test integrations?

```
Test a service's impact on its consumers → Consumer-Driven Contracts / Pact (pattern 13)
Remove external dependency in CI → Service Virtualisation / WireMock (pattern 13)
Observe end-to-end message flow → Correlation ID tracing (pattern 14)
```

---

## Integration Style Comparison

| Style | Coupling | Latency | Ordering | Best For |
|-------|----------|---------|---------|---------|
| Point-to-Point | High | Low | N/A | Simple, stable A→B |
| Hub-and-Spoke (ESB) | Medium | Medium | Optional | Enterprise routing with transformation |
| Message Bus | Low | Low | Partition-ordered | Decoupled async microservices |
| Event Streaming | Low | Low–Medium | Strict (partition) | Audit log, replay, high-throughput |
| iPaaS | Low–Medium | Medium | Depends | B2B, SaaS integrations, low-code flows |
| File / Batch | Low | High | Sequence-by-filename | Legacy, bulk, scheduled |
| Synchronous HTTP | Medium | Lowest | N/A | Real-time query, request-reply |

---

## Tool Ecosystem

### Messaging & Streaming
| Tool | Protocol | Strengths |
|------|---------|----------|
| RabbitMQ | AMQP 0-9-1 | Complex routing, DLQ, low-latency |
| Apache Kafka | Custom TCP | Durable log, replay, high throughput |
| AWS SQS + SNS | HTTP | Managed, at-least-once, fan-out |
| Azure Service Bus | AMQP 1.0 | Sessions, transactions, dead-letter |
| NATS JetStream | NATS | Ultra-low latency, built-in ack |

### Integration Platforms
| Tool | Style | Strengths |
|------|-------|----------|
| Apache Camel | Code-first | 300+ connectors, EIP patterns, Java/Quarkus |
| MuleSoft Anypoint | Visual + code | Enterprise routing, DataWeave transform |
| Temporal | Workflow code | Durable execution, long-running sagas |
| Azure Logic Apps | Low-code | SaaS connectors, event-grid triggered |
| AWS Step Functions | Visual | Lambda orchestration, state machines |
| n8n / Zapier | No-code | Rapid SaaS automation |

### CDC & Data Sync
| Tool | Mechanism | Strengths |
|------|----------|----------|
| Debezium | WAL (log-based) | Low-latency, no polling overhead |
| AWS DMS | Log + query | Managed, multi-source migration |
| Airbyte | Query-based | 300+ connectors, schema evolution |
| Fivetran | Managed SaaS | Zero-maintenance, CI/CD connectors |

### B2B Protocols
| Protocol | Domain | Standard Body |
|---------|-------|--------------|
| EDIFACT / X12 | Supply chain, invoicing | UN/CEFACT, ASC X12 |
| ISO 20022 | Payments (SWIFT, SEPA) | ISO |
| AS2 / AS4 | File transport (secure) | IETF / OASIS |
| PEPPOL | Public procurement (EU) | OpenPEPPOL |
| SWIFT MT/MX | Interbank payments | SWIFT |

---

## Cross-Reference

| If you need… | See… |
|-------------|------|
| API contract design | [API Design](../api-design/README.md) |
| Kafka patterns in depth | [Kafka](../kafka/README.md) |
| Distributed transaction primitives | [Distributed Design Patterns](../distributed-design-pattern/README.md) |
| Legacy data migration (streaming) | [Data Solutions — CDC](../data-solutions/02-change-data-capture.md) |
| Authentication for M2M calls | [CIAM/IAM — OAuth2](../ciam-iam/01-oauth2-oidc.md) |
| Secrets rotation for integration credentials | [Security — Secrets Management](../security/README.md) |
