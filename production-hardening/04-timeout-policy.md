# Timeout Policy Across Layers

## Category

**Domain:** Production Hardening · **Stack:** TypeScript, Python, YAML · **Scope:** Cascading Timeout Budgets & Deadline Propagation

---

## Context

Without explicit timeouts at every layer of the call stack, a single slow dependency causes threads or event-loop callbacks to pile up indefinitely — eventually exhausting connection pools, memory, and file descriptors. **Timeout budgets** must be structured so that inner timeouts are always shorter than outer ones: the client expects a response before the server's own timeout fires.

### Timeout Layers (outer → inner)

| Layer | Timeout Setting | Recommended Value |
|-------|---------------|------------------|
| **Client / Mobile** | Request timeout | 10–30s |
| **API Gateway** | Backend integration timeout | p99 + 1s buffer |
| **Service → Service** | HTTP client timeout | 5s default; 30s for long ops |
| **Service → Database** | Query timeout / statement_timeout | 5–10s |
| **Service → Cache** | Socket timeout (Redis) | 500ms |
| **Service → Queue** | Publish timeout | 2s |
| **Message Consumer** | Processing timeout / visibility timeout | 2–10× expected duration |

### Timeout Types

| Type | What It Covers | Example |
|------|---------------|---------|
| **Connection timeout** | Time to establish TCP connection | 1–3s |
| **Read timeout** | Time to receive first byte after connection | 5–30s |
| **Write timeout** | Time to finish sending the request body | 5s |
| **Idle timeout** | Time connection can be idle before close | 60–120s |
| **Deadline (context)** | Total wall-clock budget for entire operation | Sum of all inner timeouts |

---

## Pros

- Bounded timeouts prevent thread/connection pool exhaustion during dependency slowdowns
- Context-based deadlines (Go `context`, OTel `baggage`) propagate the remaining budget across service boundaries
- Client-visible `Retry-After` header on 503/504 enables safe retries without thundering herd
- Prometheus histogram on timeout rate surfaces which dependencies have latency regression
- Short timeouts at the cache layer ≤ 500ms ensure Redis latency never degrades the critical path

## Cons

- Timeouts that are too short cause false failures — must be calibrated to actual p99 + jitter
- Propagating deadline context requires explicit wiring at every HTTP/gRPC/DB call — easy to miss
- Retrying on timeout without idempotency guards causes duplicate operations (payments, emails)
- SQS/SNS visibility timeout must be longer than processing timeout or messages get re-delivered while still being processed
- Asymmetric timeouts between services cause confusing error chains: upstream sees timeout but downstream completed successfully

---

## Design Diagram

```mermaid
flowchart LR
    Client[Client\n30s budget] -->|HTTP 30s timeout| GW[API Gateway\n28s timeout]
    GW -->|HTTP 10s| SvcA[Service A\n8s context deadline]
    SvcA -->|HTTP 4s| SvcB[Service B\n3s context deadline]
    SvcB -->|query 2s| DB[(PostgreSQL\nstatement_timeout=2s)]
    SvcB -->|GET 300ms| Cache[(Redis\nsocket_timeout=500ms)]
    SvcA -->|publish 1s| Queue[SQS\npublish timeout 2s]
    SvcB -->|timeout| Error503[503 with\nRetry-After header]
```

---

## Code Sample

### TypeScript — HTTP Client with Timeout + Context Deadline

```typescript
// src/http/resilient-client.ts
// AbortController propagates cancellation when the outer context deadline is reached
import { trace, context, propagation } from '@opentelemetry/api';
import { logger } from '../observability/logger';

export interface RequestOptions {
  timeoutMs?: number;
  retries?: number;
}

export async function fetchWithTimeout<T>(
  url: string,
  init: RequestInit = {},
  options: RequestOptions = {},
): Promise<T> {
  const { timeoutMs = 5_000, retries = 0 } = options;
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(), timeoutMs);

  // Inject OTel trace context headers for distributed tracing
  const headers: Record<string, string> = { ...(init.headers as Record<string, string>) };
  propagation.inject(context.active(), headers);

  try {
    const response = await fetch(url, {
      ...init,
      headers,
      signal: controller.signal,
    });

    if (!response.ok) {
      throw new Error(`HTTP ${response.status} from ${url}`);
    }
    return response.json() as Promise<T>;
  } catch (err: unknown) {
    if ((err as Error).name === 'AbortError') {
      logger.warn({ url, timeoutMs }, 'request timed out');
      throw Object.assign(new Error(`Timeout after ${timeoutMs}ms calling ${url}`), {
        code: 'ETIMEDOUT',
        retryable: true,
      });
    }
    throw err;
  } finally {
    clearTimeout(timer);
  }
}

// --- Usage ---
// const data = await fetchWithTimeout<Order>(
//   'http://order-service/orders/123',
//   { method: 'GET' },
//   { timeoutMs: 3_000, retries: 2 },
// );
```

### TypeScript — Express Global Timeout Middleware

```typescript
// src/middleware/request-timeout.ts
// Responds with 503 if a handler has not completed within the budget.
// Ensures the client always gets a response — avoids hanging connections.
import type { Request, Response, NextFunction } from 'express';

export function requestTimeout(limitMs: number) {
  return (req: Request, res: Response, next: NextFunction): void => {
    const timer = setTimeout(() => {
      if (!res.headersSent) {
        res
          .status(503)
          .set('Retry-After', '5')
          .json({ error: 'request_timeout', retryAfterSeconds: 5 });
      }
    }, limitMs);

    res.on('finish', () => clearTimeout(timer));
    res.on('close',  () => clearTimeout(timer));
    next();
  };
}

// app.use(requestTimeout(10_000)); // 10s global budget
```

### Python — aiohttp Client with Per-Request Deadline

```python
# src/http_client.py
import asyncio
import aiohttp
import logging
from opentelemetry.propagate import inject

log = logging.getLogger(__name__)

_timeout = aiohttp.ClientTimeout(
    total=5.0,        # end-to-end wall-clock budget
    connect=1.0,      # TCP handshake
    sock_read=4.0,    # time to read response body
)


async def get_json(url: str, deadline_seconds: float | None = None) -> dict:
    """Fetch JSON with a cascading deadline.

    If the caller passes a deadline_seconds value (e.g. remaining context budget),
    we use the tighter of that and the default _timeout.
    """
    effective = _timeout
    if deadline_seconds is not None and deadline_seconds < 5.0:
        effective = aiohttp.ClientTimeout(total=deadline_seconds, connect=min(1.0, deadline_seconds))

    headers: dict[str, str] = {}
    inject(headers)  # OTel trace propagation

    try:
        async with aiohttp.ClientSession(timeout=effective) as session:
            async with session.get(url, headers=headers) as resp:
                resp.raise_for_status()
                return await resp.json()
    except asyncio.TimeoutError:
        log.warning("request timed out", extra={"url": url, "deadline": effective.total})
        raise
```

### YAML — PostgreSQL Statement Timeout + Istio VirtualService

```yaml
# PostgreSQL connection string parameter (per-session timeout)
# postgresql://user:pass@host:5432/db?options=-c%20statement_timeout%3D5000

# --- OR set globally in postgresql.conf ---
# statement_timeout = '5000ms'    # kill any query running longer than 5s
# lock_timeout = '2000ms'         # fail fast on lock contention
# idle_in_transaction_session_timeout = '30000ms'  # kill idle-in-transaction

---
# Istio VirtualService — timeout + retry policy for inter-service calls
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payment-service
  namespace: production
spec:
  hosts:
    - payment-service
  http:
    - timeout: 8s               # service-mesh level timeout (inner of GW 28s)
      retries:
        attempts: 2
        perTryTimeout: 3s       # each individual attempt budget
        retryOn: "5xx,reset,connect-failure,retriable-4xx"
      route:
        - destination:
            host: payment-service
            port:
              number: 8080
```
