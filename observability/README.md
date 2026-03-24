# Observability

> **15 production-proven patterns** for instrumenting, monitoring, and understanding distributed systems — from raw telemetry collection to organizational maturity.

---

## Pattern Index

| # | Pattern | Key Tools | TL;DR |
|---|---------|-----------|-------|
| 01 | [OpenTelemetry Instrumentation](01-opentelemetry-instrumentation.md) | OTel SDK (Node.js, Python), K8s Operator | Auto-instrument services with zero-code OTel SDK; inject manual spans for business logic |
| 02 | [Distributed Tracing](02-distributed-tracing.md) | Grafana Tempo, OTel Collector, W3C TraceContext | Propagate context across services and queues; tail-based sampling for cost control |
| 03 | [Metrics & Prometheus](03-metrics-prometheus.md) | Prometheus, prom-client, kube-prometheus-stack | RED metrics (rate/errors/duration) with recording rules and remote write to Mimir |
| 04 | [Structured Logging](04-structured-logging.md) | Pino, structlog, Fluent Bit, Grafana Loki | JSON logs with trace context injection; Fluent Bit DaemonSet forwarding to Loki |
| 05 | [SLIs, SLOs & Error Budgets](05-sli-slo-error-budgets.md) | Prometheus recording rules, Grafana | Define availability + latency SLOs; calculate burn rate and time-to-exhaustion |
| 06 | [SLO-Based Alerting](06-alerting-slo-burn-rate.md) | AlertManager, PagerDuty, Prometheus | Multi-window multi-burn-rate (MWMB) alerting; AlertManager routing and inhibition |
| 07 | [Grafana Dashboards as Code](07-grafana-dashboards-as-code.md) | Terraform Grafana provider, Grafonnet | Version-controlled dashboards; Jsonnet reusable panel library; CI/CD provisioning |
| 08 | [Health Checks & Probes](08-health-checks-probes.md) | Kubernetes probes, Blackbox Exporter | Liveness vs readiness vs startup probes; dependency-aware health endpoints |
| 09 | [Synthetic Monitoring](09-synthetic-monitoring.md) | k6, Blackbox Exporter, GitHub Actions | Scripted user flows running every 5 min; SSL expiry and DNS alerts |
| 10 | [Real User Monitoring (RUM)](10-real-user-monitoring.md) | OTel Web, web-vitals, Grafana Faro | Capture Core Web Vitals (LCP/INP/CLS) from real browsers; frontend–backend trace correlation |
| 11 | [Continuous Profiling](11-continuous-profiling.md) | Grafana Pyroscope, pprof, eBPF | Always-on CPU and memory profiling; flame graph correlation with traces and metrics |
| 12 | [Log–Trace Correlation & Exemplars](12-log-correlation-tracing.md) | OTel, Prometheus Exemplars, Grafana | Inject trace_id into logs; Prometheus exemplars link metric spikes to traces |
| 13 | [OTel Collector Pipeline](13-otel-collector-pipeline.md) | OTel Collector Contrib, OTTL | Fan-out telemetry to multiple backends; tail-based sampling; spanmetrics connector |
| 14 | [Database Observability](14-database-observability.md) | postgres_exporter, pg_stat_statements, PgBouncer | Slow query detection; connection pool saturation; replication lag alerts |
| 15 | [Observability Maturity Model](15-observability-maturity.md) | Prometheus, Grafana, culture | Four Golden Signals, USE/RED methods, 5-level maturity model, toil tracking |

---

## Decision Guide

### Which signal should I start with?

```
Is the system completely dark? → Start with uptime checks + basic metrics (01, 03)
Getting too many noisy alerts? → Implement SLOs + burn-rate alerting (05, 06)
Can't reproduce a production bug? → Add distributed tracing + log correlation (02, 04, 12)
Users report slowness but metrics look fine? → Add RUM + synthetic monitoring (09, 10)
Can't find which code is slow? → Add continuous profiling (11)
Database causing problems? → Database observability (14)
Want to route telemetry without code changes? → OTel Collector pipeline (13)
Want to measure observability capability gaps? → Maturity model (15)
```

### Telemetry Stack Choices

| Need | Recommended Open-Source Stack | Managed Alternative |
|------|-------------------------------|---------------------|
| Traces | Grafana Tempo | Datadog APM, AWS X-Ray |
| Metrics | Prometheus + Mimir | Datadog Metrics, CloudWatch |
| Logs | Grafana Loki + Fluent Bit | Elasticsearch + Filebeat |
| Profiles | Grafana Pyroscope | Datadog Continuous Profiler |
| Dashboards | Grafana | Datadog, Dynatrace |
| Alerting | Prometheus + AlertManager | PagerDuty, Opsgenie |
| Frontend | OTel Web + Grafana Faro | Datadog RUM, New Relic Browser |

### Observability vs Monitoring

| Concept | Monitoring | Observability |
|---------|-----------|--------------|
| **Question** | "Is it broken?" | "Why is it broken?" |
| **Approach** | Pre-defined dashboards | Ad-hoc queries on raw signals |
| **Signals** | Metrics (aggregate) | Metrics + logs + traces + profiles |
| **Coverage** | Known failure modes | Unknown failure modes |
| **Tooling** | Alerting rules | Distributed tracing, structured logging |

---

## Tool Ecosystem

```
┌─────────────────────────────────────────────────────────────────┐
│                    Grafana Dashboards / Alerting                │
├──────────────┬──────────────┬──────────────┬───────────────────┤
│ Grafana Tempo│ Prometheus / │  Grafana Loki│  Grafana           │
│ (traces)     │ Mimir        │  (logs)      │  Pyroscope         │
│              │ (metrics)    │              │  (profiles)        │
├──────────────┴──────────────┴──────────────┴───────────────────┤
│               OTel Collector (receive / process / export)       │
├─────────────────────────────────────────────────────────────────┤
│  App SDK        │  eBPF Agent   │  k6 Synthetics │  Blackbox Exp │
│  OTel SDK       │  (Pyroscope)  │  (synthetic)   │  (uptime)     │
│  web-vitals/    │               │                │               │
│  Grafana Faro   │               │                │               │
│  (RUM)          │               │                │               │
└─────────────────────────────────────────────────────────────────┘
```

---

## Related Sections

- [DevOps](../devops/README.md) — CI/CD pipelines, deployment strategies
- [DevSecOps](../devsecops/README.md) — Security scanning and compliance
- [FinOps](../finops/README.md) — Cloud cost observability
- [Backend](../backend/README.md) — Service resilience patterns (circuit breaker, retry)
- [System Design](../system-design/README.md) — Distributed systems foundations
