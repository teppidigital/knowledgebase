# Rust Knowledge Base

A practical reference for building production Rust systems — from ownership and the type system to async, FFI, and performance tuning.

## Index

| # | Topic | Key concepts |
|---|-------|-------------|
| [01](./01-ownership-borrowing.md) | Ownership & Borrowing | move semantics, borrow checker, lifetimes, `Clone` vs `Copy` |
| [02](./02-type-system.md) | Type System | structs, enums, `Option`, `Result`, pattern matching, traits |
| [03](./03-error-handling.md) | Error Handling | `Result`, `?` operator, `thiserror`, `anyhow`, custom errors |
| [04](./04-traits-generics.md) | Traits & Generics | trait bounds, associated types, blanket impls, monomorphisation |
| [05](./05-async-tokio.md) | Async & Tokio | `async/await`, `Future`, Tokio runtime, tasks, channels |
| [06](./06-http-services.md) | HTTP Services | Axum, middleware, extractors, graceful shutdown |
| [07](./07-concurrency.md) | Concurrency | `Arc`, `Mutex`, channels (`mpsc`, `broadcast`), `rayon` |
| [08](./08-memory-management.md) | Memory Management | heap vs stack, `Box`, `Rc`, `Arc`, interior mutability, `Cell`/`RefCell` |
| [09](./09-testing.md) | Testing | unit tests, integration tests, mocks (`mockall`), property testing (`proptest`) |
| [10](./10-database-patterns.md) | Database Patterns | `sqlx`, `SeaORM`, connection pooling, migrations (sqlx migrate) |
| [11](./11-cli-tooling.md) | CLI Tooling | `clap`, `indicatif`, `dialoguer`, configuration |
| [12](./12-ffi-unsafe.md) | FFI & Unsafe | `unsafe`, C interop, `cbindgen`, `bindgen`, raw pointers |
| [13](./13-macros.md) | Macros | declarative (`macro_rules!`), procedural (derive, attribute, function-like) |
| [14](./14-performance-profiling.md) | Performance & Profiling | `cargo-flamegraph`, criterion, zero-cost abstractions, SIMD |
| [15](./15-security-patterns.md) | Security Patterns | secret handling, input validation, TLS, `secrecy`, constant-time ops |

---

## Decision Guide

### What are you building?

| Scenario | Recommended reading |
|----------|---------------------|
| New HTTP service | 01 → 05 → 06 → 03 → 10 |
| CLI tool | 01 → 02 → 03 → 11 |
| Systems / embedded | 01 → 08 → 12 → 07 |
| High-throughput async service | 05 → 07 → 06 → 14 |
| Library / crate | 04 → 02 → 03 → 01 |
| FFI wrapper around C | 12 → 01 → 08 |
| Migrate from Go / C++ | 01 → 02 → 05 → 03 |

### Which smart pointer?

| Situation | Use |
|-----------|-----|
| Heap allocation, single owner | `Box<T>` |
| Single-threaded shared ownership | `Rc<T>` |
| Multi-threaded shared ownership | `Arc<T>` |
| Interior mutability, single-threaded | `RefCell<T>` |
| Interior mutability, multi-threaded | `Mutex<T>` / `RwLock<T>` |
| Cheap `Copy`-like access in a cell | `Cell<T>` |

---

## Frequently Combined Patterns

| Goal | Combine |
|------|---------|
| Resilient async HTTP service | 05 + 06 + 03 + 10 |
| Concurrent data processing pipeline | 05 + 07 + 14 |
| Type-safe configuration | 02 + 03 + 11 |
| Safe C library wrapper | 12 + 01 + 08 |
| Ergonomic public API | 04 + 02 + 13 |
