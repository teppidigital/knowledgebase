# Error Handling

## Category
Rust, Error Handling, Result, thiserror, anyhow

## Context

Rust has no exceptions. Errors are values returned as `Result<T, E>`. The `?` operator propagates errors up the call stack ergonomically. Two crates dominate production error handling: **`thiserror`** for library errors (typed, structured) and **`anyhow`** for application errors (context-rich, boxed).

| Tool | Best for | Error type |
|------|----------|-----------|
| `thiserror` | Libraries / domain layer | Structured enum — callers can `match` on variants |
| `anyhow` | Application / binary layer | Boxed `dyn Error` — add context with `.context()` |
| `?` operator | Propagate any error that converts via `From` | — |
| `std::error::Error` | Base trait all error types implement | — |

### When to use which

- **`thiserror`** in library crates and domain/repository layers where callers need to distinguish error kinds.
- **`anyhow`** in `main`, CLI, and integration glue where you want to add context and propagate upward.
- Mixing: domain returns `thiserror` types; service layer wraps with `anyhow` for rich context chains.

---

## Pros

- The type system forces every error path to be acknowledged — no silent swallowing.
- `?` makes error propagation as concise as exception-based languages while remaining explicit.
- `thiserror` derives `Display` and `Error` from `#[error("...")]` doc strings — no boilerplate.
- `anyhow::Context::context()` and `with_context()` add human-readable context to any error.
- Error chains are preserved: `anyhow` walks `source()` to print the full cause chain.

---

## Cons

- Returning `anyhow::Error` from a library crate couples callers to `anyhow` — use `thiserror` for public APIs.
- Large match arms on error enums become verbose — consider `is_not_found()` helper methods.
- `Box<dyn Error>` (used by `anyhow`) erases the concrete type — no programmatic matching without downcasting.
- Explicit `From` implementations for error conversion can add boilerplate when not using `#[from]`.
- Panics (`unwrap()`, `expect()`) bypass the type system — treat as bugs; use in tests or truly unreachable paths only.

---

## Design Diagram

```mermaid
flowchart TD
    DB["sqlx::Error<br/>(database driver)"]
    REPO["RepositoryError<br/>(thiserror enum)<br/>NotFound / DbError / Conflict"]
    SVC["anyhow::Error<br/>+ .context('create order')"]
    HANDLER["HTTP handler<br/>match on RepositoryError<br/>errors.Is equivalent via downcast"]

    DB -->|#[from]| REPO
    REPO -->|? operator| SVC
    SVC --> HANDLER
```

---

## Code Sample

### thiserror — typed domain errors

```rust
// src/domain/error.rs
use thiserror::Error;

#[derive(Debug, Error)]
pub enum OrderError {
    #[error("order {id} not found")]
    NotFound { id: String },

    #[error("order {id} is in terminal state {status}")]
    InvalidTransition { id: String, status: String },

    #[error("duplicate order for idempotency key {key}")]
    Conflict { key: String },

    #[error("database error")]
    Database(#[from] sqlx::Error),   // Automatic From impl via #[from]
}
```

### Repository layer returning typed errors

```rust
// src/repository/order.rs
use crate::domain::{Order, OrderError, OrderId};

pub async fn find_by_id(pool: &sqlx::PgPool, id: &OrderId) -> Result<Order, OrderError> {
    sqlx::query_as::<_, Order>("SELECT * FROM orders WHERE id = $1")
        .bind(&id.0)
        .fetch_optional(pool)
        .await?                         // sqlx::Error → OrderError::Database via #[from]
        .ok_or_else(|| OrderError::NotFound { id: id.0.to_string() })
}
```

### anyhow — application layer with context

```rust
// src/service/order.rs
use anyhow::{Context, Result};
use crate::domain::{OrderError, OrderId};

pub async fn get_order(pool: &sqlx::PgPool, id: &OrderId) -> Result<Order> {
    crate::repository::order::find_by_id(pool, id)
        .await
        .with_context(|| format!("fetching order {}", id.0))
    // Result<Order, anyhow::Error> — context chain preserved
}
```

### ? operator — propagate up

```rust
async fn handle_create_order(
    pool: &sqlx::PgPool,
    req: CreateOrderRequest,
) -> Result<Order, OrderError> {
    let customer = find_customer(pool, &req.customer_id).await?;   // ? propagates OrderError
    let order = Order::new(customer, req.amount)?;                  // ? propagates validation error
    save_order(pool, &order).await?;                                // ? propagates database error
    Ok(order)
}
```

### Matching on domain errors in HTTP handlers

```rust
use axum::http::StatusCode;
use axum::response::{IntoResponse, Response};

impl IntoResponse for OrderError {
    fn into_response(self) -> Response {
        let (status, message) = match &self {
            OrderError::NotFound { id } =>
                (StatusCode::NOT_FOUND, format!("order {id} not found")),
            OrderError::Conflict { key } =>
                (StatusCode::CONFLICT, format!("duplicate key {key}")),
            OrderError::InvalidTransition { .. } =>
                (StatusCode::UNPROCESSABLE_ENTITY, self.to_string()),
            OrderError::Database(_) => {
                tracing::error!("database error: {self}");
                (StatusCode::INTERNAL_SERVER_ERROR, "internal error".to_string())
            }
        };
        (status, message).into_response()
    }
}
```

### Avoid panics — use expect with context as documentation

```rust
// In tests — panics are acceptable
let order = find_order(&id).unwrap();

// In application code — prefer ? or provide a meaningful message
// BAD:
let config = std::env::var("DATABASE_URL").unwrap();

// GOOD: fail fast at startup with a clear reason
let config = std::env::var("DATABASE_URL")
    .expect("DATABASE_URL environment variable is required");
```

---

## Related

- [02 — Type System](./02-type-system.md) — `Result` and `Option` are core enum types
- [05 — Async & Tokio](./05-async-tokio.md) — `?` works identically in `async fn` returning `Result`
- [06 — HTTP Services](./06-http-services.md) — `IntoResponse` impl on error type enables clean Axum error handling
