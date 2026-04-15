# Strangler Fig Integration

## Category

System Integration — Legacy Modernisation & Incremental Migration

## Context

The **Strangler Fig pattern** (named after the fig tree that grows around and eventually replaces its host) migrates a legacy system incrementally by routing traffic through a façade. New functionality is implemented in the modern system; the façade routes each request to either legacy or modern based on feature parity. The legacy system is "strangled" gradually — with zero big-bang cutover risk.

### Migration Phases

| Phase | Routing | What Happens |
|-------|---------|-------------|
| 1. Intercept | 100% → Legacy | Add façade/proxy in front of legacy; no behaviour change |
| 2. Co-existence | Mixed | New features → Modern; old features → Legacy |
| 3. Dark launch | Dual-write | Send to both; compare responses; do not return modern result yet |
| 4. Traffic migration | Increasing % → Modern | Canary release: 5% → 25% → 100% |
| 5. Legacy retirement | 100% → Modern | Façade removed; legacy decommissioned |

### Decision: Which Capabilities to Migrate First?

| Priority | Capability Type | Reason |
|----------|----------------|--------|
| High | New business capabilities needed now | Blocked by legacy limitations |
| High | High-traffic, high cost on legacy | Infrastructure savings |
| Medium | Stable, well-understood features | Lower risk to migrate |
| Low | Rarely used, complex edge cases | Leave until last |
| Defer | Obsolete functionality | Candidate for removal, not migration |

## Pros

- Zero big-bang cutover — teams can migrate at their own pace
- Rollback is trivial: flip a feature flag or route percentage back to legacy
- Dark launch allows response comparison before exposing users to new system
- Legacy keeps running; no service interruption during migration
- Team can validate end-to-end in production with real traffic

## Cons

- Two systems running simultaneously doubles infrastructure cost during migration
- Data duplication or synchronisation between legacy DB and modern DB must be managed (CDC)
- The façade becomes a critical path component — it must be HA
- Long migrations invite "permanent temporary" state — migration stalls if not actively driven
- Integration testing must cover both routing paths

## Design Diagram

```mermaid
flowchart LR
    C[Client] -->|all traffic| F[Façade / Proxy<br/>+ Feature Router]

    F -->|legacy routes| L[Legacy System<br/>🏚️]
    F -->|new routes| M[Modern System<br/>✨]
    F -->|dark launch: both| L
    F -->|dark launch: both| M

    L --- LDB[(Legacy DB)]
    M --- MDB[(Modern DB)]

    CDC[CDC / Debezium] -->|sync data| MDB
    LDB --> CDC

    subgraph Phase 2–3
        F
    end
```

## Code Sample

### TypeScript — Feature-flag-driven façade router

```typescript
// strangler/facade-router.ts
import express, { Request, Response, NextFunction } from 'express';
import { createProxyMiddleware, Options } from 'http-proxy-middleware';

// ── Feature flag store (in production: LaunchDarkly, OpenFeature, or DB table) ─
interface RouteConfig {
  path: string;               // e.g. "/v1/payments"
  method: string;             // "GET", "POST", "*"
  modern: boolean;            // true = route to modern system
  darkLaunch?: boolean;       // send to both; return legacy response
  modernPercent?: number;     // 0–100: canary % to modern
}

const routeRegistry: RouteConfig[] = [
  // Payments endpoint — migrated to modern
  { path: '/v1/payments',          method: 'POST',  modern: true },
  { path: '/v1/payments',          method: 'GET',   modern: true },
  // Statement endpoint — dark launch (dual-write, compare, return legacy)
  { path: '/v1/accounts/*/statements', method: 'GET', modern: false, darkLaunch: true },
  // Card management — still on legacy
  { path: '/v1/cards',             method: '*',     modern: false },
];

// ── Proxies ───────────────────────────────────────────────────────────────────
const LEGACY_URL = process.env.LEGACY_URL ?? 'http://legacy-core:8080';
const MODERN_URL = process.env.MODERN_URL ?? 'http://modern-svc:3000';

const proxyOptions = (target: string): Options => ({
  target,
  changeOrigin: true,
  on: {
    error: (err, _req, res) => {
      console.error(`[proxy] error → ${target}:`, err.message);
      (res as Response).status(502).json({ error: 'Gateway error' });
    },
  },
});

const legacyProxy = createProxyMiddleware(proxyOptions(LEGACY_URL));
const modernProxy = createProxyMiddleware(proxyOptions(MODERN_URL));

// ── Routing logic ─────────────────────────────────────────────────────────────
function resolveConfig(method: string, path: string): RouteConfig | null {
  for (const cfg of routeRegistry) {
    const pathPattern = cfg.path.replace(/\*/g, '[^/]+');
    const pathRegex = new RegExp(`^${pathPattern}(/|$)`);
    const methodMatch = cfg.method === '*' || cfg.method === method.toUpperCase();
    if (methodMatch && pathRegex.test(path)) return cfg;
  }
  return null;
}

function isCanaryModern(percent: number): boolean {
  return Math.random() * 100 < percent;
}

// ── Express façade ────────────────────────────────────────────────────────────
const app = express();

app.use(async (req: Request, res: Response, next: NextFunction) => {
  const config = resolveConfig(req.method, req.path);

  if (!config) {
    // Unknown route — fall through to legacy as default
    return legacyProxy(req, res, next);
  }

  // Add correlation ID for observability across both systems
  const correlationId = (req.headers['x-correlation-id'] as string) ?? crypto.randomUUID();
  req.headers['x-correlation-id'] = correlationId;
  req.headers['x-routed-by'] = 'strangler-facade';

  if (config.darkLaunch) {
    // Fire request to modern too, but discard its response
    req.headers['x-dark-launch'] = 'true';
    void fetch(`${MODERN_URL}${req.path}`, {
      method: req.method,
      headers: req.headers as HeadersInit,
    }).then(r => {
      console.log(`[dark-launch] ${req.method} ${req.path} modern=${r.status}`, {
        correlationId,
      });
    }).catch(err => {
      console.warn(`[dark-launch] modern error for ${req.path}:`, err.message);
    });
    // Still serve from legacy
    return legacyProxy(req, res, next);
  }

  if (config.modernPercent !== undefined) {
    const useModern = isCanaryModern(config.modernPercent);
    console.log(
      `[canary] ${req.method} ${req.path} → ${useModern ? 'modern' : 'legacy'} (${config.modernPercent}%)`,
      { correlationId },
    );
    return useModern
      ? modernProxy(req, res, next)
      : legacyProxy(req, res, next);
  }

  return config.modern
    ? modernProxy(req, res, next)
    : legacyProxy(req, res, next);
});

app.listen(8000, () => console.log('[facade] running on :8000'));
```

### YAML — Kubernetes Ingress with weighted traffic split (canary migration)

```yaml
# ingress-strangler.yaml — nginx weighted canary between legacy and modern
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: payments-ingress
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "25"       # 25% → modern
    nginx.ingress.kubernetes.io/canary-by-header: "X-Use-Modern"  # opt-in header
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /v1/payments
            pathType: Prefix
            backend:
              service:
                name: modern-payment-service
                port: { number: 3000 }
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: payments-ingress-legacy
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /v1/payments
            pathType: Prefix
            backend:
              service:
                name: legacy-core-banking
                port: { number: 8080 }
```

### YAML — Feature flag config (OpenFeature / LaunchDarkly)

```json
{
  "flags": {
    "payments-use-modern": {
      "variants": { "legacy": false, "modern": true },
      "defaultVariant": "legacy",
      "targeting": [
        {
          "if": [{ "in": [{ "var": "user.country" }, ["GB", "IE"]] }, "modern", "legacy"]
        }
      ]
    }
  }
}
```

## References

- [Martin Fowler — Strangler Fig Application](https://martinfowler.com/bliki/StranglerFigApplication.html)
- [Microsoft Azure Architecture — Strangler Fig](https://learn.microsoft.com/en-us/azure/architecture/patterns/strangler-fig)
- [OpenFeature — Open Standard for Feature Flags](https://openfeature.dev/)
- [http-proxy-middleware](https://github.com/chimurai/http-proxy-middleware)
- [Kubernetes Nginx Ingress Canary](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#canary)
