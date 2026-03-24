# Traffic Shadowing & Dark Launches

## Category

**Domain:** Production Hardening · **Stack:** Istio, NGINX, TypeScript · **Scope:** Safe Production Validation Without User Impact

---

## Context

**Traffic shadowing** (also called _mirroring_ or _dark launching_) duplicates a percentage of live production requests to a new service version or new code path **without exposing users to the results**. The shadow service processes real traffic, generates real logs and metrics, but its responses are discarded. This reveals real-world performance, error rates, and resource consumption before a release reaches users.

### Shadowing vs Other Strategies

| Strategy          | User Impact                        | Traffic Volume         | Latency Cost      |
| ----------------- | ---------------------------------- | ---------------------- | ----------------- |
| **Shadowing**     | None — responses discarded         | 10–100% of prod        | None (async copy) |
| **Canary**        | Some users (1–10%) get new version | Partial (controlled %) | None              |
| **Blue/Green**    | Instant switch — all users         | 100% immediate         | None              |
| **A/B Testing**   | Both variants active               | Split by user segment  | None              |
| **Local testing** | Test environment only              | Test traffic only      | N/A               |

### Shadowing Use Cases

| Use Case                  | What Shadowing Validates                                                    |
| ------------------------- | --------------------------------------------------------------------------- |
| **DB migration**          | New ORM/query compatibility with production data patterns                   |
| **Algorithm change**      | New pricing/routing/ranking logic — compare outputs without affecting users |
| **Infrastructure change** | New cloud region latency, new DB instance class performance                 |
| **Dependency upgrade**    | New library behaves correctly with real production request shapes           |
| **Performance baseline**  | Measure p99 latency of v2 on production traffic before promoting            |

---

## Pros

- Zero user impact — shadowed responses are silently discarded even if shadow service crashes
- Production traffic patterns reveal edge cases that synthetic tests and staging environments miss
- Istio mirroring is zero-code: configured entirely in VirtualService YAML, no application changes needed
- Shadow service metrics (error rate, latency) appear in the same Grafana dashboards as production — same tooling
- NGINX mirror module enables shadowing without a service mesh for teams on simpler stacks

## Cons

- Shadow requests consume real downstream resources: database writes must be idempotent or shadow DB separated
- Payment and email operations **must** be suppressed in shadow service (flag-gated) to avoid duplicate charges/messages
- Shadow service error rate alone is not sufficient signal — latency comparison requires careful Prometheus labelling
- 100% mirroring doubles RPS on upstream services (auth, DB) — use ≤ 10% if at scale
- Istio `mirror_percent` applies to the shadow copy — does not affect the primary route response time

---

## Design Diagram

```mermaid
flowchart LR
    Client -->|request| Istio[Istio\nVirtualService]
    Istio -->|primary route 100%| V1[payment-service v1\nstable]
    Istio -->|mirror 10% async| V2[payment-service v2\nshadow]
    V1 -->|response| Client
    V2 -->|response discarded| Null[/dev/null]
    V1 & V2 -->|metrics| Prometheus
    Prometheus --> Grafana[Grafana\nv1 vs v2 comparison]
```

---

## Code Sample

### YAML — Istio VirtualService: Traffic Mirroring

```yaml
# k8s/istio/payment-shadow.yaml
# Mirrors 10% of production traffic to the v2 shadow deployment.
# Responses from v2 are automatically discarded by Istio — not returned to clients.
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payment-service
  namespace: production
spec:
  hosts:
    - payment-service
  http:
    - route:
        - destination:
            host: payment-service
            subset: v1 # primary: serves all client responses
            port:
              number: 8080
          weight: 100
      # Mirror a percentage of requests to v2 asynchronously
      mirror:
        host: payment-service
        subset: v2
        port:
          number: 8080
      mirrorPercent: 10 # mirror 10% — use lower value at high RPS
---
# DestinationRule to define v1/v2 subsets by pod label
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-service
  namespace: production
spec:
  host: payment-service
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
```

### YAML — NGINX: Mirror Module (without Istio)

```yaml
# nginx/default.conf — NGINX mirror module for teams without a service mesh
# The mirror subrequest is non-blocking; errors are silently discarded.
server {
  listen 80;

  location /api/ {
    # Mirror 10% of requests by using a sub-location with probability
    mirror /shadow-mirror;
    mirror_request_body on;

    proxy_pass http://payment-service-v1;
  }

  # Mirror target: forward to v2; response is discarded
  location = /shadow-mirror {
    internal;                           # not accessible from outside
    proxy_pass http://payment-service-v2$request_uri;
    proxy_set_header X-Shadow-Request "true";
    proxy_set_header X-Original-URI $request_uri;
  }
}
```

### TypeScript — Shadow Mode Guard (Suppress Side Effects)

```typescript
// src/middleware/shadow-mode.ts
// When a request arrives with the X-Shadow-Request header (set by Istio/NGINX),
// suppress all side effects (emails, payments, audit logs) while still
// executing the full business logic path for diff/comparison.
import type { Request, Response, NextFunction } from "express";

declare global {
  namespace Express {
    interface Request {
      isShadow?: boolean;
    }
  }
}

export function shadowModeMiddleware(
  req: Request,
  _res: Response,
  next: NextFunction,
): void {
  req.isShadow = req.headers["x-shadow-request"] === "true";
  next();
}

// Usage in payment handler — guards against duplicate side effects:
// async function chargePayment(req: Request): Promise<void> {
//   const result = await paymentGateway.charge(req.body);
//   if (!req.isShadow) {
//     await sendPaymentConfirmationEmail(result);
//     await auditLog.write('payment_charged', result);
//   }
// }
```

### TypeScript — Prometheus: Compare v1 vs v2 Metrics During Shadow

```typescript
// Prometheus recording rules for shadow comparison (add to PrometheusRule CR)

// p99 latency: v1 vs v2
// histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{app="payment-service",version="v1"}[5m]))
// histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{app="payment-service",version="v2"}[5m]))

// Error rate comparison
// rate(http_requests_total{app="payment-service",version="v1",status_code=~"5.."}[5m])
// rate(http_requests_total{app="payment-service",version="v2",status_code=~"5.."}[5m])

// Auto-abort shadow experiment if v2 error rate exceeds 5%
// Alert rule:
// - alert: ShadowVersionErrorRateHigh
//   expr: |
//     rate(http_requests_total{version="v2",status_code=~"5.."}[5m])
//     / rate(http_requests_total{version="v2"}[5m]) > 0.05
//   for: 2m
//   annotations:
//     summary: "Shadow v2 error rate exceeds 5% — abort shadow experiment"
```

### YAML — Gradual Promotion After Shadow Validation

```yaml
# Shadow validation checklist before promoting v2 to production:
# 1. v2 error rate ≤ v1 error rate for 30+ minutes
# 2. v2 p99 latency ≤ v1 p99 * 1.1 (within 10% of v1)
# 3. v2 memory usage within expected range
# 4. No unexpected log errors in v2 Loki stream
# 5. No side effects detected (duplicate payments, emails)
#
# On pass: promote to canary (10% user traffic) using Argo Rollouts

apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payment-service
  namespace: production
spec:
  hosts:
    - payment-service
  http:
    - route:
        - destination:
            host: payment-service
            subset: v1
          weight: 90
        - destination:
            host: payment-service
            subset: v2
          weight: 10 # promote from shadow (discarded) to canary (real 10% users)
```
