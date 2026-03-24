# API Design & Integration Patterns

A comprehensive reference of 15 API design and integration patterns covering contract design, delivery guarantees, developer experience, resilience, and lifecycle management across REST, GraphQL, gRPC, AsyncAPI, and webhook styles.

---

## Pattern Index

| # | Pattern | Key Concepts |
|---|---------|-------------|
| 01 | [REST API Design](01-rest-api-design.md) | OpenAPI 3.1, RFC 7807, ETags, Idempotency-Key, cursor pagination |
| 02 | [GraphQL Schema Design](02-graphql-schema-design.md) | Schema-first SDL, DataLoader, Relay pagination, error-as-data, graphql-shield |
| 03 | [gRPC & Protobuf](03-grpc-protobuf.md) | Protobuf IDL, `reserved` fields, auth interceptor, deadline + retry, server-streaming |
| 04 | [AsyncAPI & Event-Driven APIs](04-asyncapi-event-driven.md) | AsyncAPI 3.0, KafkaJS, SASL/SCRAM-SHA-512, idempotent consumer, DLQ |
| 05 | [API Versioning Strategies](05-api-versioning.md) | URL versioning, date-based versioning, Deprecation/Sunset headers, expand-only schema |
| 06 | [API Contracts & Consumer-Driven Testing](06-api-contracts-pact.md) | Pact V3, MatchersV3, provider verifier, `can-i-deploy`, PactFlow |
| 07 | [Webhook Design & Reliability](07-webhook-design.md) | HMAC-SHA256 signatures, replay protection, exponential backoff, DLQ, idempotency |
| 08 | [API Gateway Patterns](08-api-gateway.md) | Kong declarative, AWS API Gateway Terraform, JWT auth plugin, custom JS plugin |
| 09 | [SDK Generation](09-sdk-generation.md) | openapi-generator-cli, TypeScript SDK wrapper, retry/auth, CI auto-publish pipeline |
| 10 | [API Rate Limiting](10-api-rate-limiting.md) | Sliding window counter, token bucket (Lua), 429 + Retry-After, per-tier limits |
| 11 | [API Monetisation](11-api-monetisation.md) | Secure API key gen/hash, usage plans, quota enforcement, Stripe metered billing |
| 12 | [Hypermedia & HATEOAS](12-hypermedia-hateoas.md) | HAL builder, `_links`, JSON:API, lifecycle-driven links, cursor pagination |
| 13 | [API Deprecation Lifecycle](13-api-deprecation.md) | RFC 8594 Sunset, Deprecation header, 410 Gone, sunset alerts, OpenAPI `x-sunset` |
| 14 | [API Mocking & Service Virtualisation](14-api-mocking.md) | MSW handlers, WireMock stateful scenarios, Prism spec-driven, Docker Compose mocks |
| 15 | [GraphQL Federation](15-graphql-federation.md) | Apollo Federation v2, `@key` / `@external` / `@requires`, Apollo Router, supergraph config |

---

## Decision Guide

### Which API style should I use?

```
Is your consumer a browser or mobile app needing flexible queries?
  ├── Yes → GraphQL (pattern 02) or REST (pattern 01)
  └── No → Is this service-to-service?
            ├── Synchronous, low-latency → gRPC (pattern 03)
            ├── Event-driven / async → AsyncAPI + Kafka (pattern 04)
            └── Public API / B2B → REST with OpenAPI (pattern 01)
```

### Which GraphQL deployment should I use?

```
One team, one service → Monolith GraphQL (pattern 02)
Multiple teams, independent services → Apollo Federation (pattern 15)
Legacy with existing GraphQL services → Schema stitching (evaluate migration to Federation)
```

### How should I handle versioning?

```
Breaking change (field removal, type change) → New URL path version (pattern 05)
Non-breaking addition → Expand-only schema (pattern 05)
Internal service → Date-based header versioning (pattern 05)
Consumer-specific compatibility → Pact contracts (pattern 06)
```

### How do I protect a public API?

```
Identity → API key (hashed, never stored in plaintext) (pattern 11)
Rate limiting → Sliding window counter in Redis (pattern 10)
Quota → Monthly call cap per tier (patterns 10, 11)
Gateway auth → JWT validation at API Gateway (pattern 08)
```

### When should I use webhooks vs polling vs SSE?

```
B2B outbound notification to consumer endpoint → Webhook (pattern 07)
Real-time UI update, browser → SSE or WebSocket
Consumer cannot expose a public endpoint → Long polling or SSE
```

---

## Tool Ecosystem

### API Specification

| Tool | Purpose |
|------|---------|
| **OpenAPI 3.1** | REST API contract (YAML/JSON) |
| **AsyncAPI 3.0** | Event-driven API contract |
| **Protobuf + buf** | gRPC schema and code generation |
| **GraphQL SDL** | GraphQL schema-first definition |

### Code Generation

| Tool | From | To |
|------|------|----|
| `openapi-generator-cli` | OpenAPI 3.x | TypeScript, Java, Go, Python, … |
| `@graphql-codegen/cli` | GraphQL SDL + operations | TypeScript hooks, resolvers, SDL |
| `buf generate` | Protobuf | TypeScript, Go, Java |

### Testing

| Tool | Layer |
|------|-------|
| Pact V3 | Consumer-driven contract tests |
| MSW | In-process mock for browser + Node |
| WireMock | Stateful HTTP mock server |
| Prism | Spec-driven mock + request validation |

### Infrastructure

| Tool | Role |
|------|------|
| Kong | Self-hosted API Gateway (declarative YAML) |
| AWS API Gateway | Managed gateway (Terraform) |
| Apollo Router | GraphQL Federation supergraph layer |
| Redis | Rate limiting, quota counters, API key cache |

### Observability

| Signal | Tool |
|--------|------|
| API metrics | Prometheus + Grafana |
| Distributed tracing | OpenTelemetry → Jaeger / Datadog |
| Access logs | Structured JSON (ECS format) → ELK |
| API health | Uptime checks per endpoint |

---

## Key Conventions

- **TypeScript**: strict mode, no `any`, `Record<string, unknown>` for dynamic objects
- **Errors**: RFC 7807 Problem Details (`type`, `title`, `status`, `detail`)
- **Security**: HMAC-SHA256 signatures (timing-safe compare), hashed API keys, Bearer JWT
- **Pagination**: cursor-based (not offset) — `cursor` + `limit`, return `nextCursor`
- **Rate limits**: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, `Retry-After`
- **Deprecation**: `Deprecation` (IETF draft) + `Sunset` (RFC 8594) + `Link` rel=successor-version
- **API Keys**: `crypto.randomBytes(32)` → `base64url` → `SHA-256` hash for storage
- **Credentials**: Environment variables only — never hardcoded, never committed
