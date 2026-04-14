# Stripe — Idempotency and Distributed Rate Limiting at Scale

## Category

Scaling, API Design, Rate Limiting, Idempotency, Distributed Systems, Redis, Financial Systems

## Scale at the Time

| Metric | Value |
|--------|-------|
| API requests | Hundreds of millions per day |
| Payment volume | Hundreds of billions of dollars/year |
| Merchant integrations | Millions |
| Failure modes | Partial network failures, timeouts, client retries |
| Consistency requirement | Exactly-once charge semantics |
| Rate limiting scope | Per API key, per endpoint, per merchant |

---

## Initial Architecture

Stripe's API is a stateless HTTP REST service. Merchants integrate directly, sending credit card charges, creating subscriptions, or manipulating payment objects. The API is intentionally simple: POST to create, GET to read.

The challenge is that HTTP is not reliable: requests can time out, networks drop connections, and clients cannot know whether the server processed the request before the failure. A naive merchant that retries a timed-out charge request risks charging the customer twice.

---

## The Problem

### 1. Duplicate Charges from Client Retries
A merchant sends `POST /charges` for $100. The network drops after Stripe processes the charge but before the 200 response reaches the merchant. The merchant's retry logic fires and sends an identical $100 charge request. Without safeguards, the customer is charged twice.

This is not hypothetical — it happens at scale continuously. Every network partition, timeout, or DNS hiccup is a potential source of duplicate mutations.

### 2. Retry Storms Under Partial Degradation
When Stripe experiences partial degradation (high latency on one service), merchants with aggressive retry logic send repeated requests. The retry volume amplifies the original load, potentially turning a partial degradation into a full outage — the same vicious cycle as the OpenAI PostgreSQL case.

### 3. One Bad API Key Exhausting Resources
A poorly implemented merchant integration (or a merchant under attack) can send an unbounded stream of API requests. Without rate limiting, one API key can saturate Stripe's workers, degrading performance for all other merchants.

### 4. Rate Limit Accuracy Across Distributed API Servers
Stripe runs many API server instances behind a load balancer. A simple per-server rate limit is inaccurate: a merchant can make N requests to each server and exceed the intended global rate limit by a factor of M (server count).

---

## The Solution

### S1. Idempotency Keys

Stripe introduced the `Idempotency-Key` request header. Merchants generate a unique key per logical operation (e.g., UUID) and include it on every mutating request.

**Behaviour:**
1. First request with key K → Stripe processes it, stores result associated with K in an idempotency store (Redis or DB)
2. Second request with same key K → Stripe detects the duplicate, returns the cached response immediately **without re-processing**
3. After 24 hours → idempotency key expires; if sent again, treated as a new request

Key design constraints:
- If the first request is still **in progress**, the second request with the same key blocks until the first completes (using a distributed lock on the key)
- If the first request **failed with a client error (4xx)**, the key is not consumed — retry is safe
- If the first request **failed with a server error (5xx)**, the key is stored with the failure result; retrying returns the same error (prevents infinite retry attempts masking bugs)

```
Client → POST /charges
         Idempotency-Key: a3b7c9d1-...

Stripe checks Redis:
  Key exists + completed → return cached response (no charge)
  Key exists + in flight → acquire lock, wait, return result
  Key not found → process charge, store result in Redis, return response
```

### S2. Idempotency Store Design

The idempotency store must be:
- **Fast** — checked on every mutating request (low-latency read/write)
- **Atomic** — key creation and result storage must be atomic (race conditions between concurrent retries must be impossible)
- **Consistent** — a key stored must survive a server restart

Stripe uses Redis with Lua scripts for atomic conditional operations:

```lua
-- Atomic: set key only if not already set; return existing value if set
local existing = redis.call('GET', KEYS[1])
if existing then
  return existing
end
redis.call('SETEX', KEYS[1], ARGV[2], ARGV[1])  -- ARGV[1]=payload, ARGV[2]=TTL
return nil
```

### S3. Multi-Layer Rate Limiting

Stripe applies rate limits at multiple independent layers:

| Layer | Scope | Algorithm | Storage |
|-------|-------|-----------|---------|
| **API key limit** | Per merchant API key | Token bucket | Redis |
| **Endpoint limit** | Per API key + endpoint | Token bucket | Redis |
| **Global limit** | All traffic, circuit breaker | Leaky bucket / concurrency limit | In-memory |
| **Burst allowance** | Short-term spikes | Token bucket burst capacity | Redis |

Each layer can independently reject requests before reaching downstream services.

### S4. Token Bucket Algorithm (Per API Key)

The token bucket allows sustainable rate limits with burst tolerance:
- Bucket starts full at capacity C
- Each request consumes 1 token
- Tokens refill at rate R per second
- If bucket is empty → 429 Too Many Requests

At scale with many API keys across many Redis nodes, tokens are maintained as a Redis hash with atomic Lua-based debit operations.

### S5. Exponential Backoff Enforcement in SDKs

Stripe provides official SDKs (Python, Ruby, Node.js, Java, Go, PHP) that implement retry logic with exponential backoff and jitter. Stripe cannot control merchant retry behaviour, but it can make the correct behaviour the default for those using SDKs.

Retry conditions:
- Connection errors → always retry
- 429 Too Many Requests → retry with backoff (respect `Retry-After` header)
- 500/503 → retry with backoff (idempotency key prevents duplicate processing)
- 4xx client errors → never retry (merchant bug; retry will give same error)

---

## Key Learnings

1. **Idempotency is a first-class API design requirement for financial APIs** — any mutating endpoint that can be retried must be idempotent; implement it from day one, not as an afterthought
2. **Idempotency keys must use distributed locks to prevent in-flight duplicates** — without locking, two simultaneous requests with the same key can both process before either stores the result
3. **Store idempotency key outcomes for failures too** — if a request failed with a 5xx, storing that outcome prevents infinite retry loops that mask bugs
4. **Rate limiting must be global, not per-server** — use a shared store (Redis) to maintain token buckets; per-server limits allow adversarial circumvention and inaccurate enforcement
5. **Retry storms are a client-side problem you solve server-side** — enforce backoff via your SDK; add Retry-After headers so clients know when to retry; implement circuit breakers
6. **Token bucket beats fixed window for API rate limiting** — fixed windows allow double the intended rate at window boundaries; token bucket is smooth and allows controlled bursting
7. **Embed retry logic in official SDKs** — you cannot rely on merchants implementing correct retry behaviour; make the correct behaviour automatic in the SDK

---

## Architecture Diagram

```mermaid
sequenceDiagram
    participant Client as Merchant (client)
    participant LB as Load Balancer
    participant API as Stripe API Server
    participant RateLimit as Rate Limiter (Redis)
    participant Idempotency as Idempotency Store (Redis)
    participant Charge as Charge Service
    participant DB as Payment Database

    Client->>LB: POST /charges<br/>Idempotency-Key: uuid-123<br/>Amount: $100

    LB->>API: Route request

    API->>RateLimit: check_token_bucket(api_key, endpoint)
    alt Rate limit exceeded
        RateLimit-->>API: 0 tokens remaining
        API-->>Client: 429 Too Many Requests<br/>Retry-After: 2
    else Rate limit OK
        RateLimit-->>API: consume 1 token, OK

        API->>Idempotency: get_or_lock(key="uuid-123")
        alt Key exists (completed)
            Idempotency-->>API: cached_response
            API-->>Client: 200 (deduplicated response, no charge)
        else Key in flight
            Idempotency-->>API: wait for lock
            Idempotency-->>API: cached_response (other request completed)
            API-->>Client: 200 (deduplicated response)
        else Key not found
            Idempotency-->>API: lock acquired
            API->>Charge: process_charge(amount=$100, card=...)
            Charge->>DB: INSERT INTO charges (...)
            DB-->>Charge: charge_id=ch_abc123
            Charge-->>API: {id: "ch_abc123", status: "succeeded"}
            API->>Idempotency: store_result(key="uuid-123", result={...}, ttl=24h)
            API-->>Client: 200 {id: "ch_abc123", status: "succeeded"}
        end
    end
```

---

## Code / Config

### Idempotency key middleware (TypeScript / Express)

```typescript
import Redis from 'ioredis';
import { Request, Response, NextFunction } from 'express';

const redis = new Redis({ host: 'idempotency-redis' });
const IDEMPOTENCY_TTL = 86400; // 24 hours
const LOCK_TTL = 30;           // 30 seconds lock while in-flight

async function idempotencyMiddleware(
  req: Request,
  res: Response,
  next: NextFunction
): Promise<void> {
  const idempotencyKey = req.headers['idempotency-key'] as string;

  if (!idempotencyKey || req.method === 'GET') {
    return next(); // idempotency only for mutating requests
  }

  const storeKey = `idempotency:${req.user.id}:${idempotencyKey}`;
  const lockKey  = `idempotency_lock:${req.user.id}:${idempotencyKey}`;

  // Check for existing result
  const existing = await redis.get(storeKey);
  if (existing) {
    const { status, body } = JSON.parse(existing);
    res.status(status).json(body);
    return;
  }

  // Acquire lock (NX = only set if not exists)
  const locked = await redis.set(lockKey, '1', 'EX', LOCK_TTL, 'NX');
  if (!locked) {
    // Another request is processing this key — poll briefly
    await waitForResult(storeKey, 10_000);
    const result = await redis.get(storeKey);
    if (result) {
      const { status, body } = JSON.parse(result);
      res.status(status).json(body);
      return;
    }
    res.status(409).json({ error: 'Idempotency key conflict — retry later' });
    return;
  }

  // Intercept response to store in idempotency store
  const originalJson = res.json.bind(res);
  res.json = (body: unknown): Response => {
    const payload = JSON.stringify({ status: res.statusCode, body });
    redis.setex(storeKey, IDEMPOTENCY_TTL, payload).then(() => {
      redis.del(lockKey);
    });
    return originalJson(body);
  };

  next();
}

async function waitForResult(key: string, maxWaitMs: number): Promise<void> {
  const start = Date.now();
  while (Date.now() - start < maxWaitMs) {
    const result = await redis.get(key);
    if (result) return;
    await new Promise((r) => setTimeout(r, 100));
  }
}
```

### Token bucket rate limiter (Redis + Lua)

```typescript
const TOKEN_BUCKET_SCRIPT = `
  local key     = KEYS[1]
  local capacity = tonumber(ARGV[1])
  local rate     = tonumber(ARGV[2])   -- tokens per second
  local now      = tonumber(ARGV[3])   -- current unix ms

  local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
  local tokens     = tonumber(bucket[1]) or capacity
  local last_refill = tonumber(bucket[2]) or now

  -- Refill tokens based on elapsed time
  local elapsed = (now - last_refill) / 1000   -- seconds
  tokens = math.min(capacity, tokens + elapsed * rate)

  if tokens < 1 then
    return 0   -- rate limited
  end

  tokens = tokens - 1
  redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
  redis.call('EXPIRE', key, 3600)
  return 1   -- allowed
`;

async function checkRateLimit(
  apiKey: string,
  endpoint: string,
  capacity = 100,
  ratePerSecond = 10
): Promise<boolean> {
  const key = `ratelimit:${apiKey}:${endpoint}`;
  const result = await redis.eval(
    TOKEN_BUCKET_SCRIPT,
    1,
    key,
    capacity.toString(),
    ratePerSecond.toString(),
    Date.now().toString()
  );
  return result === 1;
}
```

---

## References

- [Stripe Engineering — Designing Robust and Predictable APIs with Idempotency](https://stripe.com/blog/idempotency) (2016)
- [Stripe Engineering — Rate Limiters](https://stripe.com/blog/rate-limiters) (2017)
- [Stripe API Reference — Idempotent Requests](https://stripe.com/docs/api/idempotent_requests)
- [Redis Documentation — Lua Scripting](https://redis.io/docs/manual/programmability/lua-api/)
- [Token Bucket Algorithm](https://en.wikipedia.org/wiki/Token_bucket)
