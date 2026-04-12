# Dependency Injection

## Category
Go, Dependency Injection, Architecture, Wire, Fx

## Context

Go favours **explicit dependency injection** over reflection-based containers. The constructor pattern (`NewX(dep1, dep2) *X`) is idiomatic. For large services, `wire` (compile-time) or `uber-go/fx` (runtime, declarative) remove wiring boilerplate in `main.go`.

| Approach | Tool | Trade-off |
|----------|------|-----------|
| Manual (`main.go`) | none | Verbose but fully explicit; best for small services |
| Compile-time code gen | `google/wire` | No reflection; wiring errors at compile time |
| Runtime container | `uber-go/fx` | Less boilerplate; wiring errors at startup |

### When to use what

- **< 10 components**: manual wiring in `main.go`.
- **10–50 components**: `wire` — keeps compile-time safety.
- **Large platform services with lifecycle hooks**: `fx` — provides `Lifecycle` (OnStart/OnStop) and modules.

---

## Pros

- Manual DI is transparent — no magic, easy to trace, no framework dependency.
- `wire` produces readable generated code — you can inspect what it creates.
- `fx` `Lifecycle` hooks replace ad-hoc goroutine-based startup sequencing.
- Interface-based injection (see [03](./03-interfaces-composition.md)) makes every layer independently testable.
- Avoiding a DI framework means zero framework CVEs and no version lock-in.

---

## Cons

- Large `main.go` manual wiring becomes a maintenance burden beyond ~20 components.
- `wire` requires annotating constructors with `wire.Build` and regenerating on every dependency graph change.
- `fx` uses reflection at startup — wiring errors surface at runtime, not compile time.
- `fx` modules add indirection — tracing which `Provide` satisfies which `Inject` requires `fx.DryRun` or the visualiser.
- Over-abstraction: injecting interfaces for types that have only one implementation adds indirection with no test benefit.

---

## Design Diagram

```mermaid
flowchart TD
    MAIN["main.go<br/>(composition root)"]
    CONFIG["config.Load()"]
    POOL["repository.NewPool(cfg.DSN)"]
    REPO["repository.NewOrderRepo(pool)"]
    SVC["service.NewOrderService(repo, logger)"]
    HANDLER["handler.NewServer(addr, svc)"]

    MAIN --> CONFIG
    MAIN --> POOL
    POOL --> REPO
    REPO --> SVC
    SVC --> HANDLER
    HANDLER -->|srv.Run(ctx)| RUN["Running server"]
```

---

## Code Sample

### Manual DI — small service (recommended default)

```go
// cmd/server/main.go
func run(ctx context.Context) error {
    cfg := config.Load()

    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
    slog.SetDefault(logger)

    pool, err := repository.NewPool(ctx, cfg.DatabaseURL)
    if err != nil {
        return fmt.Errorf("db pool: %w", err)
    }
    defer pool.Close()

    if err := repository.Migrate(ctx, pool); err != nil {
        return fmt.Errorf("migrate: %w", err)
    }

    repo := repository.NewOrderRepo(pool)
    svc  := service.NewOrderService(repo, logger)
    srv  := handler.NewServer(cfg.Addr, svc)

    return srv.Run(ctx)
}
```

### wire — compile-time DI

```go
// wire/wire.go  (only compiled during code generation, not in the binary)
//go:build wireinject

package wire

import (
    "context"
    "github.com/google/wire"
    "github.com/myorg/myservice/config"
    "github.com/myorg/myservice/internal/handler"
    "github.com/myorg/myservice/internal/repository"
    "github.com/myorg/myservice/internal/service"
)

func InitializeServer(ctx context.Context, cfg *config.Config) (*handler.Server, func(), error) {
    wire.Build(
        repository.NewPool,
        repository.NewOrderRepo,
        service.NewOrderService,
        handler.NewServer,
    )
    return nil, nil, nil
}
```

```bash
go run github.com/google/wire/cmd/wire@latest ./wire/
# Generates wire/wire_gen.go with all constructor calls inlined
```

### fx — runtime declarative DI with lifecycle

```go
// cmd/server/main.go
package main

import (
    "go.uber.org/fx"
    "go.uber.org/fx/fxevent"
    "go.uber.org/zap"
)

func main() {
    app := fx.New(
        fx.WithLogger(func(log *zap.Logger) fxevent.Logger {
            return &fxevent.ZapLogger{Logger: log}
        }),
        fx.Provide(
            zap.NewProduction,
            config.Load,
            repository.NewPool,
            repository.NewOrderRepo,
            service.NewOrderService,
            handler.NewServer,
        ),
        fx.Invoke(registerHooks),
    )
    app.Run()
}

func registerHooks(lc fx.Lifecycle, srv *handler.Server, pool *pgxpool.Pool) {
    lc.Append(fx.Hook{
        OnStart: func(ctx context.Context) error {
            go srv.Run(ctx)
            return nil
        },
        OnStop: func(ctx context.Context) error {
            pool.Close()
            return srv.Shutdown(ctx)
        },
    })
}
```

### fx module — group related providers

```go
// internal/repository/module.go
package repository

import "go.uber.org/fx"

var Module = fx.Module("repository",
    fx.Provide(
        NewPool,
        NewOrderRepo,
        NewProductRepo,
    ),
)

// internal/service/module.go
var Module = fx.Module("service",
    fx.Provide(
        NewOrderService,
        NewProductService,
    ),
)

// main.go
app := fx.New(
    repository.Module,
    service.Module,
    handler.Module,
)
```

---

## Related

- [01 — Project Structure](./01-project-structure.md) — `cmd/server/main.go` is the composition root
- [03 — Interfaces & Composition](./03-interfaces-composition.md) — interface-based injection enables mock swapping in tests
- [05 — Testing](./05-testing.md) — manual DI makes injecting mocks in tests trivial
