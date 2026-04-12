# Go (Golang) Knowledge Base

A practical reference for building production Go services — from project layout and concurrency primitives to observability, security, and performance.

## Index

| # | Topic | Key concepts |
|---|-------|-------------|
| [01](./01-project-structure.md) | Project Structure & Modules | `go mod`, workspace, package layout, build tags |
| [02](./02-concurrency.md) | Concurrency — Goroutines & Channels | goroutines, channels, select, sync, errgroup |
| [03](./03-interfaces-composition.md) | Interfaces & Composition | implicit satisfaction, embedding, functional options |
| [04](./04-error-handling.md) | Error Handling | error wrapping, sentinel errors, custom types, panic/recover |
| [05](./05-testing.md) | Testing & Benchmarking | table-driven tests, mocks, fuzz, benchmarks, testcontainers |
| [06](./06-http-services.md) | HTTP Services | net/http, middleware, chi, context propagation |
| [07](./07-generics.md) | Generics | type parameters, constraints, type sets, generic collections |
| [08](./08-context.md) | Context | propagation, cancellation, deadlines, values |
| [09](./09-database-patterns.md) | Database Patterns | database/sql, pgx, sqlc, migrations, repository pattern |
| [10](./10-observability.md) | Observability | slog, zerolog, OpenTelemetry SDK, Prometheus client |
| [11](./11-grpc-services.md) | gRPC Services | protobuf, grpc-go, interceptors, streaming, reflection |
| [12](./12-dependency-injection.md) | Dependency Injection | wire, fx, manual DI, interface-based design |
| [13](./13-performance-profiling.md) | Performance & Profiling | pprof, escape analysis, zero-alloc, GC tuning |
| [14](./14-cli-tooling.md) | CLI Tooling | cobra, viper, configuration patterns |
| [15](./15-security-patterns.md) | Security Patterns | crypto/tls, JWT, input validation, rate limiting |

---

## Decision Guide

### What are you building?

| Scenario | Recommended reading |
|----------|---------------------|
| New service from scratch | 01 → 06 → 08 → 04 → 10 |
| gRPC microservice | 01 → 11 → 08 → 10 → 04 |
| CLI tool | 01 → 14 → 04 |
| High-throughput data pipeline | 02 → 13 → 09 → 10 |
| Platform / SDK library | 03 → 07 → 04 → 05 |
| Background worker / scheduler | 02 → 08 → 09 → 04 |
| Migrate from Java / Python | 02 → 03 → 04 → 12 |

### Which concurrency tool?

| Problem | Solution |
|---------|----------|
| Run N tasks, wait for all | `errgroup.Group` |
| Pipeline with bounded parallelism | buffered channel + worker pool |
| Cache with concurrent access | `sync.Map` or `sync.RWMutex` |
| Run once at startup | `sync.Once` |
| Fan-out to multiple consumers | multiple goroutines reading the same channel |
| Timeout / cancel a call | `context.WithTimeout` + `select` |

---

## Frequently Combined Patterns

| Goal | Combine |
|------|---------|
| Resilient HTTP service | 06 + 08 + 04 + 10 |
| Testable gRPC service | 11 + 03 + 05 + 12 |
| Safe concurrent worker pool | 02 + 08 + 13 |
| Secure public API | 06 + 15 + 08 + 10 |
| Data-heavy batch job | 09 + 02 + 13 + 10 |