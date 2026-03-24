# Production Hardening

> **15 patterns** for making services survive real-world production: safe termination, resource governance, failure injection, zero-user-impact releases, and certification before launch.

---

## Pattern Index

| # | Pattern | Key Tools | TL;DR |
|---|---------|-----------|-------|
| 01 | [Graceful Shutdown & Signal Handling](01-graceful-shutdown.md) | Node.js, Python, tini | SIGTERM → drain → close pool → SIGKILL in 60s; `preStop` sleep for LB propagation |
| 02 | [Resource Limits, Requests & VPA](02-resource-limits-vpa.md) | Kubernetes VPA, LimitRange, ResourceQuota | Right-size CPU/memory; Guaranteed QoS; VPA recommendations without restarts |
| 03 | [Pod Disruption Budgets](03-pod-disruption-budgets.md) | Kubernetes PDB, Kyverno | Block simultaneous evictions; percentage-based `maxUnavailable`; multi-AZ spread |
| 04 | [Timeout Policy Across Layers](04-timeout-policy.md) | Express, aiohttp, Istio | Cascading timeout budgets: client → gateway → service → DB → cache |
| 05 | [Load Shedding & Backpressure](05-load-shedding.md) | Express, KEDA, SQS | Concurrency limiter; token bucket; queue-depth backpressure with KEDA scale-out |
| 06 | [Connection Pool Tuning](06-connection-pool-tuning.md) | PgBouncer, node-postgres, SQLAlchemy | Pool sizing formula; PgBouncer transaction mode; pool saturation alerts |
| 07 | [Admission Controllers & Guardrails](07-admission-controllers.md) | Kyverno, OPA Gatekeeper | Block `:latest` tags; inject `securityContext`; require resource limits; registry allow-list |
| 08 | [Runtime Security with Falco & seccomp](08-runtime-security-falco.md) | Falco, Falcosidekick, seccomp | eBPF syscall detection; shell-spawn alerts; `RuntimeDefault` seccomp profile |
| 09 | [Chaos Engineering](09-chaos-engineering.md) | Chaos Mesh, LitmusChaos | Pod kill, network delay, CPU stress; steady-state validation; CI chaos gates |
| 10 | [Rollback Strategy & Feature Flags](10-rollback-feature-flags.md) | ArgoCD, Argo Rollouts, OpenFeature | Automated SLO-triggered rollback; progressive delivery analysis; kill-switch flags |
| 11 | [Startup Ordering & Init Containers](11-startup-ordering.md) | Kubernetes init containers, Helm hooks | Wait-for-DB; migration init; sidecar ordering; warm-up health endpoint |
| 12 | [Memory Leak Detection & OOM Prevention](12-memory-leak-detection.md) | V8, tracemalloc, Prometheus | Heap snapshot at 85%; `--max-old-space-size`; growth trend alerting |
| 13 | [Dependency Health Gating](13-dependency-health-gating.md) | TypeScript, FastAPI, Kubernetes | Hard vs soft deps; refuse readiness until DB/Redis pass; soft deps degrade gracefully |
| 14 | [Traffic Shadowing & Dark Launches](14-traffic-shadowing.md) | Istio, NGINX mirror | Mirror 10% of prod traffic to v2; discard responses; compare metrics side-by-side |
| 15 | [Production Readiness Review](15-production-readiness-review.md) | Kyverno, CI, GitHub | Bronze/Silver/Gold gate; automated PRR validation script; GitHub issue template |

---

## Decision Guide

```
Service crashes during rolling update?     → Graceful shutdown (01) + PDB (03)
OOMKilling pods?                           → Resource limits + VPA (02) + memory leak (12)
Slow dependency taking down the service?   → Timeout policy (04) + load shedding (05)
DB connection errors under load?           → Connection pool tuning (06)
Bad image configurations reaching K8s?     → Admission controllers (07)
Container escape or unexpected behaviour?  → Runtime security Falco (08)
Not sure resilience mechanisms work?       → Chaos engineering (09)
Need to roll back quickly?                 → Rollback + feature flags (10)
Service starts before DB is ready?         → Startup ordering + init containers (11)
Memory growing over days?                  → Memory leak detection (12)
Service starts but immediately errors?     → Dependency health gating (13)
Want to validate v2 on real traffic?       → Traffic shadowing (14)
New service going to production?           → Production Readiness Review (15)
```

### Hardening Priority by Tier

| Tier | Must-Have Patterns |
|------|-------------------|
| **Any service** | 01, 02, 03, 04, 13 |
| **Stateless HTTP API** | + 05, 06, 12 |
| **Revenue-critical** | + 07, 08, 09, 10, 15 |
| **New major version** | + 11, 14 |

---

## How These Patterns Interconnect

```
Deployment
│
├── 11-startup-ordering       — init containers wait for DB, run migration
│     └── 13-dependency-gating — readiness gate: don't enter LB until deps pass
│
├── 01-graceful-shutdown      — on SIGTERM: drain, close pool, exit clean
│     └── 03-pdb              — K8s never takes down more pods than budget allows
│
├── 02-resource-limits        — right-sized CPU/memory limits + VPA tuning
│     └── 12-memory-leak      — heap monitor alerts before OOMKill fires
│
├── 04-timeout-policy         — cascading deadlines prevent thread pile-ups
│     └── 05-load-shedding    — concurrency limiter + 503 for excess requests
│           └── 06-pool-tuning — DB connections sized to match concurrency limit
│
├── 07-admission-controllers  — policy guardrails enforce all of the above at deploy time
│     └── 08-runtime-security — Falco detects runtime violations post-deploy
│
├── 09-chaos-engineering      — validates that all the above actually works under failure
│
├── 10-rollback-feature-flags — SLO breach triggers automatic rollback or flag disable
│     └── 14-traffic-shadowing — validate v2 on real traffic before canary promotion
│
└── 15-prr-checklist          — gate that certifies all patterns are in place before launch
```

---

## Related Sections

- [Observability](../observability/README.md) — Metrics, tracing, alerting, SLOs
- [DevSecOps](../devsecops/README.md) — CI security scanning, secret scanning, SBOM
- [Security](../security/README.md) — Network policies, secrets management, mTLS
- [DevOps](../devops/README.md) — CI/CD pipelines, deployment strategies
- [System Design](../system-design/README.md) — Circuit breaker, rate limiting, bulkhead
