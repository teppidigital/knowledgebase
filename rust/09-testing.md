# Testing & Property Testing

## Category
Rust, Testing, Mocking, Property Testing, Integration Tests

## Context

Rust's standard test runner is built into `cargo`. Unit tests live in the same file as the code under test (in a `#[cfg(test)]` module). Integration tests live in a `tests/` directory. The ecosystem adds **`mockall`** for mocks, **`proptest`** for property-based testing, and **`rstest`** for parameterised tests.

| Tool | Purpose |
|------|---------|
| `#[test]` | Standard unit test |
| `#[tokio::test]` | Async unit test |
| `tests/` directory | Integration tests (separate crate) |
| `mockall` | Auto-generate mock implementations of traits |
| `proptest` | Property-based (randomised) testing |
| `rstest` | Parameterised / fixture-based tests |
| `insta` | Snapshot testing |
| `testcontainers` | Spin up Docker containers in tests |
| `wiremock` | HTTP mock server |

---

## Pros

- Unit tests in the same file as production code have access to private functions and types.
- `#[tokio::test]` makes async test setup identical to sync — no manual runtime construction.
- `mockall::automock` generates mock structs from trait definitions — same ergonomics as Mockito/mockery.
- `proptest` finds edge cases that table-driven tests miss by generating random inputs guided by shrinking.
- `cargo test` runs tests in parallel by default; `-- --test-threads=1` for sequential.

---

## Cons

- `mockall` struggles with traits that have associated types or complex lifetimes.
- `proptest` requires defining strategy types — adds learning curve for non-FP teams.
- Integration tests in `tests/` are compiled as a separate crate — they cannot access private types.
- `testcontainers` requires Docker in CI; containers add startup latency (~1–5 s per test binary).
- `cargo test` does not output `println!` on passing tests by default — add `-- --nocapture` for debugging.

---

## Design Diagram

```mermaid
flowchart TD
    UNIT["Unit Tests<br/>mod tests inside src/<br/>private access<br/>mockall mocks"]
    INT["Integration Tests<br/>tests/ directory<br/>public API only<br/>testcontainers / wiremock"]
    PROP["Property Tests<br/>proptest — random input<br/>shrinking on failure"]

    UNIT -->|cargo test| CI
    INT -->|cargo test --test| CI
    PROP -->|cargo test| CI
    CI["CI Pipeline"]
```

---

## Code Sample

### Unit test with mockall

```rust
// src/service/order.rs
#[cfg_attr(test, mockall::automock)]
#[async_trait::async_trait]
pub trait OrderRepo: Send + Sync {
    async fn find_by_id(&self, id: &OrderId) -> Result<Order, OrderError>;
    async fn save(&self, order: &Order) -> Result<(), OrderError>;
}

#[cfg(test)]
mod tests {
    use super::*;
    use mockall::predicate::*;

    #[tokio::test]
    async fn get_order_returns_order_when_found() {
        let mut mock = MockOrderRepo::new();
        let order_id = OrderId(Uuid::new_v4());
        let expected = Order::fixture(order_id.clone());

        mock.expect_find_by_id()
            .with(eq(order_id.clone()))
            .times(1)
            .returning(move |_| Ok(expected.clone()));

        let svc = OrderService::new(Arc::new(mock));
        let result = svc.get_order(&order_id).await;
        assert!(result.is_ok());
        assert_eq!(result.unwrap().id, order_id);
    }

    #[tokio::test]
    async fn get_order_returns_not_found_error() {
        let mut mock = MockOrderRepo::new();
        let order_id = OrderId(Uuid::new_v4());
        let id_str = order_id.0.to_string();

        mock.expect_find_by_id()
            .returning(move |_| Err(OrderError::NotFound { id: id_str.clone() }));

        let svc = OrderService::new(Arc::new(mock));
        let result = svc.get_order(&order_id).await;
        assert!(matches!(result, Err(OrderError::NotFound { .. })));
    }
}
```

### rstest — parameterised tests

```rust
#[cfg(test)]
mod tests {
    use rstest::rstest;

    #[rstest]
    #[case(1000, "USD", true)]
    #[case(0,    "USD", false)]
    #[case(-1,   "USD", false)]
    #[case(1000, "XX",  false)]
    fn validate_order_request(#[case] amount: i64, #[case] currency: &str, #[case] expected: bool) {
        let req = CreateOrderRequest { amount_cents: amount, currency: currency.to_string(), .. Default::default() };
        assert_eq!(req.validate().is_ok(), expected);
    }
}
```

### proptest — property-based testing

```rust
#[cfg(test)]
mod proptests {
    use proptest::prelude::*;

    proptest! {
        // For any non-empty string that looks like an ID, parsing must not panic
        #[test]
        fn parse_order_id_does_not_panic(s in "[a-z0-9-]{1,64}") {
            let _ = s.parse::<OrderId>();  // must not panic
        }

        // Round-trip: serialise → deserialise must produce the same value
        #[test]
        fn order_json_roundtrip(amount in 1i64..1_000_000i64, currency in "[A-Z]{3}") {
            let order = Order::fixture_with(amount, &currency);
            let json = serde_json::to_string(&order).unwrap();
            let back: Order = serde_json::from_str(&json).unwrap();
            prop_assert_eq!(order.amount, back.amount);
            prop_assert_eq!(order.currency, back.currency);
        }
    }
}
```

### Integration test with testcontainers + sqlx

```rust
// tests/order_repo_integration.rs
use testcontainers::{clients::Cli, images::postgres::Postgres};
use sqlx::PgPool;

#[tokio::test]
async fn test_save_and_find_order() {
    let docker = Cli::default();
    let postgres = docker.run(Postgres::default());
    let port = postgres.get_host_port_ipv4(5432);
    let dsn = format!("postgres://postgres:postgres@127.0.0.1:{port}/postgres");

    let pool = PgPool::connect(&dsn).await.unwrap();
    sqlx::migrate!("./migrations").run(&pool).await.unwrap();

    let repo = PostgresOrderRepo::new(pool);
    let order = Order::fixture(OrderId(uuid::Uuid::new_v4()));

    repo.save(&order).await.unwrap();
    let found = repo.find_by_id(&order.id).await.unwrap();
    assert_eq!(found.id, order.id);
}
```

### wiremock — HTTP mock server

```rust
// tests/payment_client_test.rs
use wiremock::{MockServer, Mock, ResponseTemplate, matchers::{method, path}};

#[tokio::test]
async fn payment_client_handles_202() {
    let server = MockServer::start().await;

    Mock::given(method("POST"))
        .and(path("/payments"))
        .respond_with(ResponseTemplate::new(202).set_body_json(json!({ "status": "accepted" })))
        .mount(&server)
        .await;

    let client = PaymentClient::new(server.uri());
    let result = client.charge("order-1", 1000).await;
    assert!(result.is_ok());
}
```

---

## Related

- [03 — Error Handling](./03-error-handling.md) — test `Err` variants with `assert!(matches!(...))`
- [04 — Traits & Generics](./04-traits-generics.md) — `mockall::automock` requires `Send + Sync` trait bounds
- [10 — Database Patterns](./10-database-patterns.md) — testcontainers provides a real Postgres for repo integration tests
