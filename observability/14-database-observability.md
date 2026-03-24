# Database Observability

## Category

**Domain:** Observability · **Stack:** Prometheus, PostgreSQL, TypeScript · **Scope:** Query Performance, Connection Pools & DB Health

---

## Context

Database failures account for a disproportionate share of production incidents. Database observability goes beyond simple "is the DB up?" checks — it covers **slow query detection**, **connection pool saturation**, **lock contention**, **index efficiency**, and **replication lag**, all surfaced as Prometheus metrics in Grafana dashboards.

### Key Signals by Layer

| Layer | Signal | Tool | Threshold |
|-------|--------|------|-----------|
| **Query performance** | p99 query duration | pg_stat_statements | > 1s |
| **Connection pools** | Active / waiting connections | PgBouncer exporter | waiting > 0 for > 10s |
| **Lock contention** | Blocked queries | pg_locks | blocked > 5 min |
| **Replication** | Replication lag | pg_stat_replication | > 30s |
| **Index efficiency** | Sequential scans on large tables | pg_statio_user_tables | seq_scan rate > 10/min |
| **Table bloat** | Dead tuple ratio | pgstattuple | dead_tup_ratio > 10% |
| **MySQL** | Slow queries, InnoDB waits | mysql_exporter | slow_queries > 0/min |

### Exporters & Tools

| Tool | Database | What It Exports |
|------|----------|----------------|
| **postgres_exporter** | PostgreSQL | Connection stats, lock info, replication lag, database size |
| **pg_stat_statements** | PostgreSQL | Per-query execution counts, total time, rows, cache hit ratio |
| **PgBouncer exporter** | PostgreSQL via PgBouncer | Pool stats: active, idle, waiting, max_wait seconds |
| **mysqld_exporter** | MySQL / MariaDB | Query rate, InnoDB buffer pool, slow queries, replication |
| **redis_exporter** | Redis | Hit ratio, memory fragmentation, replication offset |
| **mongodb_exporter** | MongoDB | Op counters, replication lag, active connections |

---

## Pros

- `pg_stat_statements` identifies the top-10 slowest queries across the entire fleet without manual `EXPLAIN ANALYZE`
- PgBouncer metrics reveal the exact moment a connection pool saturates — before `timeout` errors reach users
- Prometheus recording rules pre-aggregate slow-query counts for fast dashboard rendering
- Grafana annotations on deploy events let teams correlate query regressions with code changes instantly
- All exporters expose a `/metrics` Prometheus endpoint — zero app code changes required

## Cons

- `pg_stat_statements` must be added to `shared_preload_libraries` — requires PostgreSQL restart
- Exporters running inside the cluster need network access to database endpoints — requires careful secret management
- High-frequency scraping (< 10s) of `pg_stat_statements` adds measurable CPU overhead on busy PostgreSQL instances
- `pg_statio_user_tables` does not directly expose table bloat — pgstattuple requires a schema-level VACUUM to update
- MySQL `performance_schema` must be enabled and fine-tuned — default config incurs memory overhead

---

## Design Diagram

```mermaid
flowchart LR
    PG[PostgreSQL\nRDS / Aurora] -- pg_stat_statements --> PGExp[postgres_exporter\nsidecar/deployment]
    PG -- pg_locks, pg_stat_replication --> PGExp
    PGBouncer[PgBouncer\nconnection pool] --> PGBExp[pgbouncer_exporter]
    MySQL[MySQL / Aurora] --> MySQLExp[mysqld_exporter]
    Redis[Redis / ElastiCache] --> RedisExp[redis_exporter]
    PGExp & PGBExp & MySQLExp & RedisExp -->|/metrics| Prometheus
    Prometheus --> Grafana[Grafana\nDB Dashboards\n+ Alerts]
    Grafana -->|slow query alert| PagerDuty
```

---

## Code Sample

### YAML — postgres_exporter Kubernetes Deployment

```yaml
# k8s/db-observability/postgres-exporter.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-exporter
  namespace: observability
spec:
  replicas: 1
  selector:
    matchLabels: { app: postgres-exporter }
  template:
    spec:
      containers:
        - name: postgres-exporter
          image: quay.io/prometheuscommunity/postgres-exporter:v0.15.0
          env:
            - name: DATA_SOURCE_NAME
              valueFrom:
                secretKeyRef:
                  name: postgres-exporter-secret
                  key: dsn   # postgresql://readonly:pass@rds.host:5432/dbname?sslmode=require
          args:
            - --collector.stat_statements           # enable pg_stat_statements
            - --collector.database_wraparound       # detect XID/MXQ wraparound risk
            - --collector.long_running_transactions # long transaction detection
            - --collector.bloat                     # table/index bloat estimation
            - --log.level=info
          ports:
            - containerPort: 9187
          resources:
            limits:
              cpu: 100m
              memory: 64Mi
---
# Prometheus PodMonitor to scrape db exporter
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: postgres-exporter
  namespace: observability
spec:
  selector:
    matchLabels: { app: postgres-exporter }
  podMetricsEndpoints:
    - port: 9187
      interval: 30s
```

### YAML — Custom pg_stat_statements Queries (postgres_exporter)

```yaml
# k8s/db-observability/queries.yaml — custom SQL queries for postgres_exporter
# Mount as ConfigMap and pass via --extend.query-path=

pg_slow_queries:
  query: |
    SELECT
      queryid::text                                    AS queryid,
      LEFT(query, 100)                                 AS query_preview,
      calls,
      total_exec_time / 1000                           AS total_exec_seconds,
      mean_exec_time / 1000                            AS mean_exec_seconds,
      (blk_read_time + blk_write_time) / 1000         AS io_seconds,
      rows
    FROM pg_stat_statements
    WHERE mean_exec_time > 500   -- queries averaging > 500ms
    ORDER BY mean_exec_time DESC
    LIMIT 10;
  metrics:
    - queryid:           { usage: LABEL, description: "Query fingerprint ID" }
    - query_preview:     { usage: LABEL, description: "First 100 chars of query" }
    - calls:             { usage: COUNTER, description: "Total executions" }
    - total_exec_seconds: { usage: COUNTER, description: "Total wall-clock seconds" }
    - mean_exec_seconds: { usage: GAUGE, description: "Mean execution time seconds" }
    - io_seconds:        { usage: GAUGE, description: "Time waiting on I/O" }
    - rows:              { usage: COUNTER, description: "Total rows returned" }

pg_bloat:
  query: |
    SELECT
      schemaname,
      tablename,
      n_dead_tup,
      n_live_tup,
      CASE WHEN n_live_tup > 0
        THEN round(100 * n_dead_tup::numeric / n_live_tup, 1)
        ELSE 0
      END AS dead_tup_ratio
    FROM pg_stat_user_tables
    WHERE n_live_tup > 10000     -- only significant tables
    ORDER BY dead_tup_ratio DESC
    LIMIT 20;
  metrics:
    - schemaname:     { usage: LABEL, description: "Schema" }
    - tablename:      { usage: LABEL, description: "Table name" }
    - n_dead_tup:     { usage: GAUGE, description: "Dead tuples" }
    - n_live_tup:     { usage: GAUGE, description: "Live tuples" }
    - dead_tup_ratio: { usage: GAUGE, description: "Percentage dead tuples" }
```

### YAML — Prometheus Alerting Rules for Database

```yaml
# k8s/prometheus/db-alert-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: database-alerts
  namespace: observability
spec:
  groups:
    - name: postgresql
      interval: 30s
      rules:
        # Slow query average exceeds 1 second
        - alert: PostgreSQLSlowQuery
          expr: pg_stat_statements_mean_exec_seconds > 1
          for: 5m
          labels:
            severity: warning
            team: platform
          annotations:
            summary: "PostgreSQL slow query detected"
            description: "Query {{ $labels.query_preview }} has mean exec time {{ $value | humanizeDuration }}"
            runbook: "https://wiki.example.com/runbooks/db-slow-query"

        # Connection pool saturation (PgBouncer)
        - alert: PgBouncerClientWaiting
          expr: pgbouncer_stats_client_wait_time_seconds > 0
          for: 2m
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "PgBouncer connection pool saturated"
            description: "Pool {{ $labels.database }} has clients waiting for {{ $value }}s"

        # Replication lag > 30 seconds
        - alert: PostgreSQLReplicationLag
          expr: pg_replication_lag > 30
          for: 5m
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "PostgreSQL replication lag critical"
            description: "Replica {{ $labels.instance }} is {{ $value }}s behind primary"

        # High dead tuple ratio — AUTOVACUUM may be lagging
        - alert: PostgreSQLHighBloat
          expr: pg_bloat_dead_tup_ratio > 20
          for: 30m
          labels:
            severity: warning
          annotations:
            summary: "PostgreSQL table bloat high"
            description: "Table {{ $labels.schemaname }}.{{ $labels.tablename }} is {{ $value }}% dead tuples — consider VACUUM"

        # Database size growing fast (> 1GB/day)
        - alert: PostgreSQLDatabaseGrowthFast
          expr: increase(pg_database_size_bytes[1d]) > 1073741824
          for: 0m
          labels:
            severity: warning
          annotations:
            summary: "PostgreSQL database growing > 1GB/day"
            description: "Database {{ $labels.datname }} grew {{ $value | humanize1024 }}B in 24h"
```

### TypeScript — App-Level Query Observability (Prisma OTel)

```typescript
// src/db/prisma-client.ts
// Instruments all Prisma queries with OTel spans automatically
import { PrismaClient } from '@prisma/client';
import { PrismaInstrumentation } from '@prisma/instrumentation';
import { registerInstrumentations } from '@opentelemetry/instrumentation';
import { logger } from '../observability/logger';
import { Histogram } from 'prom-client';

registerInstrumentations({
  instrumentations: [
    new PrismaInstrumentation(),  // auto-spans for every Prisma operation
  ],
});

const queryDuration = new Histogram({
  name: 'db_query_duration_seconds',
  help: 'Duration of database queries',
  labelNames: ['model', 'operation', 'success'],
  buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1, 5],
});

export const prisma = new PrismaClient({
  log: [
    { emit: 'event', level: 'query' },
    { emit: 'event', level: 'error' },
    { emit: 'event', level: 'warn' },
  ],
});

// Log slow queries (> 500ms) with structured context
prisma.$on('query', (e) => {
  const durationMs = e.duration;
  queryDuration.observe(
    { model: 'unknown', operation: 'query', success: 'true' },
    durationMs / 1000,
  );
  if (durationMs > 500) {
    logger.warn({ query: e.query, duration_ms: durationMs, params: '[REDACTED]' },
      'slow database query');
  }
});

prisma.$on('error', (e) => {
  logger.error({ message: e.message, target: e.target }, 'prisma error');
});
```
