# Performance & Caching

## Category

GraphQL — Operations

## Context

GraphQL's flexible query model creates performance challenges that REST APIs avoid by design. Each request can fetch a unique shape of data, making HTTP-level response caching (CDN, Varnish) ineffective by default. Production GraphQL deployments address this with **persisted queries** (Automatic Persisted Queries / Trusted Documents), **response-level caching** with `@cacheControl` hints, and **resolver-level caching** via DataLoader and in-process caches.

### Caching Layers

| Layer | Mechanism | Granularity | Tools |
|-------|----------|------------|-------|
| **CDN / HTTP cache** | Persisted queries via GET | Per query document | CDN + APQ |
| **Response cache** | `@cacheControl` directive + in-memory LRU | Per response | `@apollo/server-plugin-response-cache` |
| **Resolver cache** | DataLoader per-request cache + Redis | Per entity key | DataLoader, Redis |
| **DB query cache** | ORM query-level caching, Postgres prepared statements | Per SQL pattern | Prisma accelerate, connection pool |

### Automatic Persisted Queries (APQ)

APQ replaces the full query string with its SHA-256 hash in the request:

1. Client sends `{ extensions: { persistedQuery: { sha256Hash: "abc..." } } }` — no query body
2. If the server has the hash cached, it executes; response is returned
3. If the hash is unknown, server returns `PERSISTED_QUERY_NOT_FOUND`
4. Client retries with `{ query: "...", extensions: { persistedQuery: ... } }` — server stores and executes
5. Subsequent requests use the hash only — saves bandwidth and enables GET requests (CDN cacheable)

### `@cacheControl` Scope

| `scope` | Meaning |
|---------|---------|
| `PUBLIC` | Safe to cache in a shared CDN — no user-specific data |
| `PRIVATE` | User-specific data — cache only in browser/client, never in CDN |

| `maxAge` | Meaning |
|---------|---------|
| `0` | No caching |
| `60` | Cache for 60 seconds |
| Inherited | Child fields inherit the lowest `maxAge` of any ancestor |

## Pros

- APQ with CDN GET requests can serve popular read queries with sub-millisecond CDN hit latency
- `@cacheControl` is declarative in the schema — cache policy lives alongside the type definition
- Per-request DataLoader caching prevents redundant DB round-trips within a single query execution
- Response cache at the server eliminates DB hits for identical queries within the `maxAge` window

## Cons

- APQ cache misses add one round-trip — cold request is slightly slower than without APQ
- Response caching is only effective for identical queries — shape variations (different fields) are always cache misses
- `@cacheControl(maxAge: 0)` on any field in the query tree forces the entire response to `maxAge: 0`
- Redis response caches must be invalidated on data mutations — requires explicit purge logic

## Design Diagram

```mermaid
flowchart LR
    C[Client] -->|GET /graphql?extensions={sha256}&variables={}| CDN[CDN<br/>CloudFront / Fastly]
    CDN -->|Cache HIT| C
    CDN -->|Cache MISS| R[Apollo Router<br/>or GraphQL Server]
    R -->|APQ hash lookup| AC[APQ Cache<br/>Redis / in-memory]
    AC -->|Hash known: execute| EXEC[Execute query]
    AC -->|Hash unknown| C2[Return PersistedQueryNotFound]
    EXEC --> RC[Response Cache<br/>check by hash + vars]
    RC -->|Cache HIT| C
    RC -->|MISS: execute resolvers| DB[(Database)]
    DB --> RC --> C

    style CDN fill:#ff9900,color:#000
    style RC fill:#5bc0de,color:#000
```

## Code Sample

### TypeScript — Apollo Server response cache plugin + `@cacheControl`

```typescript
import { ApolloServer } from '@apollo/server';
import responseCachePlugin from '@apollo/server-plugin-response-cache';
import { KeyValueCache } from '@apollo/utils.keyvaluecache';
import { createClient } from 'redis';

// Wire up a Redis-backed cache
const redisClient = createClient({ url: process.env.REDIS_URL });
await redisClient.connect();

const cache: KeyValueCache = {
  async get(key) { return redisClient.get(key) ?? undefined; },
  async set(key, value, options) {
    await redisClient.set(key, value, { EX: options?.ttl ?? 60 });
  },
  async delete(key) { await redisClient.del(key); },
};

const server = new ApolloServer({
  schema,
  plugins: [
    responseCachePlugin({
      // Key the cache by user to avoid PRIVATE data leakage
      sessionId: ({ contextValue }) =>
        (contextValue as AppContext).user?.id ?? null,
    }),
  ],
  cache,
});
```

### GraphQL SDL — `@cacheControl` on types and fields

```graphql
# Public reference data — safe to cache in CDN for 5 minutes
type LoanType @cacheControl(maxAge: 300, scope: PUBLIC) {
  code:        String!
  label:       String!
  description: String
}

type Loan @cacheControl(maxAge: 60, scope: PRIVATE) {
  id:        ID!
  reference: String!
  amount:    Float!
  status:    LoanStatus!
  # This field is real-time — override to never cache
  currentBalance: Float! @cacheControl(maxAge: 0)
  # Static after creation — longer cache
  type:      LoanType!  @cacheControl(maxAge: 300, scope: PUBLIC)
}
```

### TypeScript — APQ with Apollo Client (sends GET for persisted queries)

```typescript
import { ApolloClient, InMemoryCache, HttpLink } from '@apollo/client';
import { createPersistedQueryLink } from '@apollo/client/link/persisted-queries';
import { sha256 } from 'crypto-hash';

const persistedQueriesLink = createPersistedQueryLink({
  sha256,
  useGETForHashedQueries: true, // Enables CDN caching via GET
});

const client = new ApolloClient({
  link: persistedQueriesLink.concat(
    new HttpLink({ uri: 'https://api.example.com/graphql' })
  ),
  cache: new InMemoryCache(),
});
```

### TypeScript — Cache invalidation after mutation

```typescript
// After updating a loan, purge cached responses that include it
async function invalidateLoanCache(loanId: string, cache: KeyValueCache): Promise<void> {
  // APQ responses are cached under a hash of query + variables
  // For targeted invalidation, tag-based cache (e.g. Surrogate-Key header with CDN)
  // is more practical. Here we use a key pattern for response cache:
  const pattern = `responsecache:*loan*${loanId}*`;
  const keys = await redisClient.keys(pattern);
  if (keys.length > 0) {
    await redisClient.del(keys);
  }
}
```

## References

- [Apollo Automatic Persisted Queries](https://www.apollographql.com/docs/apollo-server/performance/apq/)
- [Apollo Response Cache Plugin](https://www.apollographql.com/docs/apollo-server/performance/caching/)
- [Cache Control Directive](https://www.apollographql.com/docs/apollo-server/performance/caching/#adding-cache-hints-to-your-schema)
- [Apollo Router Caching](https://www.apollographql.com/docs/router/configuration/response-cache/)
- [CDN Caching with GraphQL](https://www.apollographql.com/blog/graphql-at-the-edge)
