# GraphQL — Patterns & Practices

A comprehensive catalogue of 15 production-ready patterns for designing, building, securing, and operating GraphQL APIs at scale.

---

## Pattern Index

| # | Pattern | Key Concepts |
|---|---------|-------------|
| 01 | [Schema Design Fundamentals](01-schema-design.md) | Type system, SDL, nullability, directives, interfaces, unions |
| 02 | [Resolvers & Data Fetching](02-resolvers-data-fetching.md) | Resolver chain, context object, field-level vs root resolvers |
| 03 | [DataLoader & Batching](03-dataloader-batching.md) | N+1 problem, per-request cache, batch loading, priming |
| 04 | [Authentication & Authorisation](04-auth-authorisation.md) | Context-based auth, directive auth, field-level permissions, RBAC |
| 05 | [Error Handling](05-error-handling.md) | Error unions, partial success, extensions, nullable fields |
| 06 | [Pagination Patterns](06-pagination-patterns.md) | Relay cursor, offset, keyset, Connections spec |
| 07 | [Subscriptions & Real-time](07-subscriptions-realtime.md) | WebSocket, SSE, subscription filtering, back-pressure |
| 08 | [Federation & Supergraph](08-federation-supergraph.md) | Apollo Federation v2, subgraphs, `@key`, `@external`, Router |
| 09 | [Performance & Caching](09-performance-caching.md) | Persisted queries, response caching, CDN, APQ, cache-control |
| 10 | [Schema Evolution & Versioning](10-schema-evolution.md) | Additive changes, deprecation, breaking change detection |
| 11 | [Testing Strategies](11-testing-strategies.md) | Unit testing resolvers, integration tests, contract testing |
| 12 | [Code Generation](12-code-generation.md) | GraphQL Code Generator, typed operations, server-side typing |
| 13 | [Security](13-security.md) | Query depth, complexity, introspection, CSRF, persisted queries |
| 14 | [File Uploads](14-file-uploads.md) | Multipart request spec, streaming uploads, S3 pre-signed URLs |
| 15 | [Observability](15-observability.md) | Field-level tracing, Apollo Studio, OpenTelemetry, slow query alerts |
| 16 | [Design Patterns & Anti-Patterns](16-design-patterns-antipatterns.md) | 10 patterns, 10 anti-patterns, quick reference card |

---

## Decision Guide

### Choosing a Transport

| Requirement | Recommendation |
|-------------|---------------|
| Request/response API | HTTP POST (standard) |
| Cacheable GET requests | Persisted queries via HTTP GET |
| Real-time push | Subscriptions over WebSocket (graphql-ws) or SSE |
| Gateway fan-out | Apollo Router with Federation |

### Choosing a Federation Strategy

| Scenario | Pattern |
|----------|---------|
| Single team, single service | Monolithic schema — no federation needed |
| Multiple teams, shared types | Apollo Federation v2 subgraphs |
| Multiple orgs / public APIs | Schema stitching or gateway aggregation |
| Incremental migration from monolith | Federation with `@override` on migrated fields |

### Pagination Model Selection

| Data Characteristics | Pattern | Why |
|---------------------|---------|-----|
| Forward-only, append-heavy feeds | Relay cursor connection | Stable, efficient with B-tree indexes |
| Jump-to-page UI | Offset + count | Simple but slow at large offsets |
| High-volume time-series | Keyset (after ID/timestamp) | No offset skip penalty |
| Small stable lists (< 100) | Simple array — no pagination | Avoids Over-engineering |

### Resolver Performance Decision

| Problem | Solution |
|---------|---------|
| N+1 queries on list fields | DataLoader |
| Expensive aggregation used repeatedly | Request-scoped in-memory cache in context |
| Cross-service joins | Federation `@key` entity resolution |
| Heavy computation | Deferred fields with `@defer` |

---

## Categories

- **Schema & Types** — 01, 10, 12
- **Data Fetching** — 02, 03, 06
- **Security & Auth** — 04, 13
- **Error & Edge Cases** — 05, 14
- **Real-time** — 07
- **Architecture** — 08
- **Operations** — 09, 11, 15
