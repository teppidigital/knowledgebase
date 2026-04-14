# Pagination Patterns

## Category

GraphQL — Data Fetching

## Context

Pagination is mandatory for any list field that can grow beyond a few hundred records. GraphQL has no built-in pagination — it is a schema design choice. The **Relay Cursor Connection** specification is the dominant convention and is adopted by GitHub, Shopify, Facebook, and most large public GraphQL APIs. It enables stable forward and backward pagination, handles concurrent inserts gracefully, and provides rich metadata through `PageInfo`.

### Pagination Pattern Comparison

| Pattern | Mechanism | Stable? | Backward? | Performance | Best For |
|---------|----------|---------|-----------|------------|---------|
| **Relay Cursor** | Opaque cursor (base64 encoded id/timestamp) | ✅ | ✅ | Excellent — index seek | Default for all production lists |
| **Offset / Limit** | `offset: Int`, `limit: Int` | ❌ (inserts shift pages) | ✅ | Poor at large offsets (full scan) | Simple admin UIs, static data |
| **Keyset** | `after: ID` or `after: DateTime` on indexed field | ✅ | Limited | Excellent | Time-series, high-volume append-only |
| **Page number** | `page: Int, pageSize: Int` | ❌ | ✅ | Poor at high page numbers | Legacy compatibility only |

### Relay Connection Specification

| Type | Field | Description |
|------|-------|-------------|
| `LoanConnection` | `edges` | Array of `LoanEdge` |
| `LoanConnection` | `pageInfo` | Navigation metadata |
| `LoanConnection` | `totalCount` | Total matching records (nullable — expensive COUNT) |
| `LoanEdge` | `node` | The entity |
| `LoanEdge` | `cursor` | Opaque string to pass as `after` or `before` |
| `PageInfo` | `hasNextPage` | Are there more records forward? |
| `PageInfo` | `hasPreviousPage` | Are there more records backward? |
| `PageInfo` | `startCursor` | Cursor of the first edge |
| `PageInfo` | `endCursor` | Cursor of the last edge |

### Cursor Encoding

Cursors are **opaque to clients** — they must not be parsed or constructed. Server-side they encode whatever position information is needed:

```
base64("Loan:2024-01-15T10:30:00Z:loan-uuid-123") → "TG9hbjoyMDI0LTAxLTE1..."
```

## Pros

- Relay cursor pagination is stable — a new record inserted before the cursor position does not shift subsequent pages
- `endCursor` from a page is passed directly as `after` in the next request — simple client implementation
- Cursor can encode composite keys (timestamp + id) for unique, stable ordering on high-volume tables
- `PageInfo.hasNextPage` allows infinite scroll UIs to know when to stop without fetching an additional empty page

## Cons

- `totalCount` requires a separate `COUNT(*)` query — expensive on large tables; make it nullable and only resolve when requested
- Backward pagination (`last`, `before`) is rarely needed and complex to implement correctly in SQL — implement only when the UI actually requires it
- Opaque cursors break if the sort order of the query changes — cursor from one sort cannot be reused with a different sort
- Relay Connection pattern adds verbosity to queries — clients must always traverse `edges[].node` rather than a flat array

## Design Diagram

```mermaid
flowchart LR
    subgraph Page 1
        E1[Loan A\ncursor: abc]
        E2[Loan B\ncursor: def]
        E3[Loan C\ncursor: ghi]
    end

    subgraph Page 2
        E4[Loan D\ncursor: jkl]
        E5[Loan E\ncursor: mno]
        E6[Loan F\ncursor: pqr]
    end

    Client -->|loans first:3| Page 1
    Page1PI[pageInfo\nendCursor: ghi\nhasNextPage: true]
    Client -->|loans first:3 after:ghi| Page 2
    Page2PI[pageInfo\nendCursor: pqr\nhasNextPage: false]

    Page 1 --> Page1PI
    Page 2 --> Page2PI
```

## Code Sample

### GraphQL SDL — Relay-compliant Connection types

```graphql
type LoanConnection {
  edges:      [LoanEdge!]!
  pageInfo:   PageInfo!
  totalCount: Int          # nullable — omit to avoid expensive COUNT
}

type LoanEdge {
  node:   Loan!
  cursor: String!
}

type PageInfo {
  hasNextPage:     Boolean!
  hasPreviousPage: Boolean!
  startCursor:     String
  endCursor:       String
}

type Query {
  loans(
    first:  Int
    after:  String
    last:   Int
    before: String
    filter: LoanFilterInput
  ): LoanConnection!
}
```

### TypeScript — Cursor encoding/decoding

```typescript
export type CursorPayload = { id: string; createdAt: string };

export function encodeCursor(payload: CursorPayload): string {
  return Buffer.from(JSON.stringify(payload)).toString('base64url');
}

export function decodeCursor(cursor: string): CursorPayload {
  try {
    return JSON.parse(Buffer.from(cursor, 'base64url').toString('utf8')) as CursorPayload;
  } catch {
    throw new GraphQLError('Invalid pagination cursor', {
      extensions: { code: 'BAD_USER_INPUT' }
    });
  }
}
```

### TypeScript — Paginated resolver (Prisma + Relay Connection)

```typescript
import { decodeCursor, encodeCursor } from './cursor';
import type { AppContext } from './context';
import type { LoanFilterInput } from './generated/types';

export async function resolveLoansConnection(
  args: { first?: number; after?: string; filter?: LoanFilterInput },
  ctx: AppContext
) {
  const limit = Math.min(args.first ?? 20, 100); // cap page size
  const cursor = args.after ? decodeCursor(args.after) : null;

  const where = buildWhereClause(args.filter);

  // Fetch one extra to determine hasNextPage
  const items = await ctx.db.loan.findMany({
    where: cursor
      ? {
          ...where,
          OR: [
            { createdAt: { lt: new Date(cursor.createdAt) } },
            { createdAt: new Date(cursor.createdAt), id: { lt: cursor.id } },
          ],
        }
      : where,
    orderBy: [{ createdAt: 'desc' }, { id: 'desc' }],
    take: limit + 1,
  });

  const hasNextPage = items.length > limit;
  const edges = items.slice(0, limit).map(loan => ({
    node: loan,
    cursor: encodeCursor({ id: loan.id, createdAt: loan.createdAt.toISOString() }),
  }));

  return {
    edges,
    pageInfo: {
      hasNextPage,
      hasPreviousPage: cursor !== null,
      startCursor: edges[0]?.cursor ?? null,
      endCursor: edges[edges.length - 1]?.cursor ?? null,
    },
    // totalCount omitted here — resolve separately only when field is requested
  };
}
```

### TypeScript — `totalCount` field resolver (only executed when queried)

```typescript
const resolvers = {
  LoanConnection: {
    // totalCount is only resolved if the client includes it in their query
    totalCount: async (
      parent: { _filter: LoanFilterInput },
      _args: unknown,
      ctx: AppContext
    ): Promise<number> => {
      return ctx.db.loan.count({ where: buildWhereClause(parent._filter) });
    },
  },
};
```

## References

- [Relay Cursor Connections Specification](https://relay.dev/graphql/connections.htm)
- [GraphQL Cursor Pagination — Shopify Engineering](https://shopify.engineering/pagination-graphql-shopify)
- [Prisma Cursor-based Pagination](https://www.prisma.io/docs/orm/prisma-client/queries/pagination)
- [GitHub GraphQL Pagination](https://docs.github.com/en/graphql/guides/using-pagination-in-the-graphql-api)
