# Design Patterns & Anti-Patterns

## Category

GraphQL — Schema & Types

## Context

GraphQL gives you enormous design freedom — which means there are many ways to do it wrong. This guide catalogues the most impactful **design patterns** (proven solutions to recurring problems) and **anti-patterns** (common approaches that cause long-term pain) observed across production GraphQL APIs.

---

## Design Patterns

### 1. Domain-Driven Schema

Model the schema around **business domain concepts**, not database tables or REST endpoints.

| ❌ DB-Driven                 | ✅ Domain-Driven                                      |
| ---------------------------- | ----------------------------------------------------- |
| `type loan_applications`     | `type LoanApplication`                                |
| `applicationId`, `createdTs` | `id`, `createdAt`                                     |
| `acct_ref_fk`                | `borrower: Account!`                                  |
| `int_rate_bps`               | `interestRate: Float!` (in percent, not basis points) |

The schema is your **public API contract** — clients should never need to know how data is stored internally.

---

### 2. Error Union (Result Type)

Return business errors in the `data` payload — not as thrown exceptions in `errors[]`. Thrown exceptions are for infrastructure failures only.

```graphql
# Pattern: every mutation returns a payload with loan OR errors
type CreateLoanPayload {
  loan: Loan # null when errors present
  errors: [UserError!]! # empty on success
}

type UserError {
  message: String!
  code: String!
  field: [String!] # dotted path for form validation
}
```

**Why:** Clients can switch on `errors[0].code` (machine-readable) and bind `field` to form inputs. Thrown GraphQL errors require checking the `errors[]` array at the transport level — same work, less information.

---

### 3. Relay Connection for All Paginated Lists

Wrap every list that can grow with a Relay-compliant `Connection` type from day one — adding it later is a breaking change.

```graphql
# Pattern: never return bare arrays at root query level
type Query {
  loans(first: Int, after: String, filter: LoanFilterInput): LoanConnection! # ✅
  # NOT: loans: [Loan!]!                                                       # ❌
}

type LoanConnection {
  edges: [LoanEdge!]!
  pageInfo: PageInfo!
  totalCount: Int # nullable — expensive; only resolve when requested
}
```

---

### 4. Command / Query Separation in Mutations

Each mutation does **one thing** with a name that describes the intent — not a generic CRUD verb. This mirrors CQRS and makes schema auditing easy.

```graphql
# Pattern: intent-named mutations
type Mutation {
  approveLoanApplication(id: ID!): ApproveLoanPayload!
  disburseLoan(id: ID!, disbursementDate: DateTime!): DisburseLoanPayload!
  markLoanAsOverdue(id: ID!): MarkOverduePayload!
  writeLoanOff(id: ID!, reason: String!): WriteOffPayload!
}

# Anti-pattern: generic CRUD
type Mutation {
  updateLoan(id: ID!, input: UpdateLoanInput!): Loan # ❌ — what changed? undiscoverable
}
```

---

### 5. Nullable Fields for Partial Success

Make fields nullable unless they genuinely **can never be absent**. A non-null field in a failed child resolver propagates the null up the tree, potentially wiping out sibling data.

```graphql
type Loan {
  id: ID! # always present — non-null OK
  reference: String! # always present — non-null OK
  borrower: Account # nullable — allows partial response if account lookup fails
  score: CreditScore # nullable — only exists after scoring completes
}
```

---

### 6. Interfaces and Unions for Polymorphism

Model "one of several shapes" explicitly — do not resort to a generic `JSON` scalar or an over-loaded catch-all type.

```graphql
# Pattern: Union for search results across multiple entity types
union SearchResult = Loan | Account | Transaction

type Query {
  search(term: String!): [SearchResult!]!
}

# Pattern: Interface for shared audit fields
interface Auditable {
  createdAt: DateTime!
  updatedAt: DateTime!
  createdBy: String!
}

type Loan       implements Auditable { ... }
type Account    implements Auditable { ... }
type Transaction implements Auditable { ... }
```

---

### 7. Context-Based Dependency Injection

Pass all infrastructure (DB, DataLoaders, logger, auth user) via `context` — never import global singletons inside resolvers.

```typescript
// Pattern: inject everything via context
const resolvers = {
  Query: {
    loan: (_parent, args, ctx) => ctx.db.loan.findUnique({ where: { id: args.id } }),
  },
  Loan: {
    borrower: (parent, _args, ctx) => ctx.loaders.accountById.load(parent.borrowerAccountId),
  },
};

// Anti-pattern: global import inside resolver (untestable)
import { prisma } from '../../db';  // ❌
const resolvers = { Query: { loan: (_, args) => prisma.loan.findUnique(...) } };
```

---

### 8. Named Operations on Every Client Request

Every query, mutation, and subscription sent from a client **must have an operation name**. Anonymous operations cannot be traced, searched in Studio, or used with persisted queries.

```graphql
# Pattern: always named
query GetLoanDashboard($id: ID!) {
  # ✅ — visible in traces
  loan(id: $id) {
    id
    reference
    status
  }
}

# Anti-pattern: anonymous
query {
  loan(id: "123") {
    id
    reference
  }
} # ❌ — appears as "anonymous" in all tooling
```

---

### 9. `@deprecated` + Sunset Lifecycle

Never remove a field without first deprecating it and monitoring for zero usage. The three-phase process:

```
Phase 1: Add new field; deprecate old field with reason + migration path
Phase 2: Monitor field usage (Apollo Studio / Hive) until all clients migrated
Phase 3: Remove field (in a coordinated deployment)
```

```graphql
type Loan {
  # Phase 1 — old name deprecated, new name added
  reference: String! @deprecated(reason: "Use loanReference. Sunset: 2025-Q3")
  loanReference: String!
}
```

---

### 10. DataLoader for Every Relation Field

Any field resolver that fetches a related entity from a list parent **must use a DataLoader**. This is not optional in production.

```typescript
// Pattern: always use DataLoader for relation fields
Loan: {
  borrower: (parent, _args, ctx) => ctx.loaders.accountById.load(parent.borrowerAccountId),  // ✅
}

// Anti-pattern: direct DB call in relation resolver → N+1
Loan: {
  borrower: (parent, _args, ctx) => ctx.db.account.findUnique({ where: { id: parent.borrowerAccountId } }),  // ❌
}
```

---

## Anti-Patterns

### A1. The God Schema

**Problem:** One enormous schema for an entire organisation with hundreds of types, cross-domain relations, and shared types — results in merge conflicts, slow composition, unclear ownership.

**Symptom:** Every PR touches the same `schema.graphql` file. No team knows who owns which types.

**Fix:** Federation (Apollo Federation v2) — each team owns a subgraph with clear type ownership. Use `@key` for cross-subgraph entity references.

---

### A2. `JSON` Scalar Everywhere

**Problem:** Using `scalar JSON` for flexible data avoids modelling effort but destroys type safety, breaks code generation, makes documentation impossible, and forces every client to guess the shape at runtime.

```graphql
type Loan {
  metadata: JSON # ❌ — what's in here? No one knows.
  riskFlags: JSON # ❌ — treat as a black box
}
```

**Fix:** Model the data explicitly. If the shape is truly dynamic, use a `Map` scalar with a known value type, or a `[KeyValuePair!]!` type.

```graphql
type LoanMetadata {
  source: String!
  riskScore: Int
  reviewedBy: String
}

type Loan {
  metadata: LoanMetadata # ✅
}
```

---

### A3. Exposing the Database Schema Directly

**Problem:** Schema types mirror DB table columns — including internal IDs, snake_case names, FK columns, and implementation details like `account_id_fk`.

```graphql
type loan { # ❌ lowercase snake_case
  loan_id: ID!
  acct_ref_fk: ID! # internal FK exposed
  int_rate_bps: Int! # basis points — clients need to convert
  created_ts: String # not a DateTime
}
```

**Fix:** Transform at the resolver layer — the schema reflects the domain, the resolver maps from DB model.

---

### A4. Non-Null Mutation Results

**Problem:** Mutation root fields are declared non-null (e.g. `createLoan!: Loan!`) — if the resolver throws, the error **propagates up to the root** and nulls the entire `data` object.

```graphql
type Mutation {
  createLoan(input: CreateLoanInput!): Loan! # ❌
}
```

**Fix:** Return a nullable payload type or a dedicated result type. A resolver returning null or throwing should never wipe out the entire `data` response.

```graphql
type Mutation {
  createLoan(input: CreateLoanInput!): CreateLoanPayload! # ✅ — payload is non-null, loan inside is nullable
}
```

---

### A5. Deeply Nested Mutations

**Problem:** Mirroring REST sub-resources as nested mutations is confusing, hard to authorise, and breaks the command-intent principle.

```graphql
type Mutation {
  order: OrderMutations!
}

type OrderMutations {
  items: OrderItemMutations! # ❌ — deeply nested
}

type OrderItemMutations {
  add(orderId: ID!, productId: ID!): Order!
}
# Client: mutation { order { items { add(orderId: "1", productId: "2") { ... } } } }
```

**Fix:** Flat mutations at the root with explicit IDs as arguments.

```graphql
type Mutation {
  addOrderItem(
    orderId: ID!
    productId: ID!
    quantity: Int!
  ): AddOrderItemPayload! # ✅
}
```

---

### A6. Missing Pagination on Lists

**Problem:** Returning bare arrays at query root fields with no size limit — a single query can return millions of records.

```graphql
type Query {
  transactions: [Transaction!]! # ❌ — unbounded list
}
```

**Fix:** Always wrap growable lists in a Connection type with `first`/`after` arguments and a server-side default limit.

---

### A7. Reusing Object Types as Input Types

**Problem:** Using the same type for both query results and mutation inputs — inputs cannot have circular references or resolver methods, and they may include fields that were not authored by the client (like `id`, `createdAt`).

```graphql
type Mutation {
  createLoan(loan: Loan!): Loan! # ❌ — Loan is an output type, not an input type
}
```

**Fix:** Always define separate `input` types for mutations.

```graphql
input CreateLoanInput {
  borrowerAccountId: ID!
  amount: Float!
  termMonths: Int!
}
```

---

### A8. Blocking I/O in Resolvers Without `async/await`

**Problem:** Performing synchronous CPU-heavy work or accidentally blocking the event loop in a resolver stalls the entire Node.js process for all concurrent requests.

```typescript
// Anti-pattern: CPU-heavy synchronous work in resolver
Loan: {
  riskScore: (parent) => {
    // Synchronous heavy computation — blocks event loop
    return expensiveCpuBoundCalculation(parent.financials); // ❌
  };
}
```

**Fix:** Offload to a worker thread or an external service; always return a `Promise`.

---

### A9. Silently Returning `null` for Auth Denied

**Problem:** Returning `null` from a resolver when the user is not authorised is indistinguishable from "not found" at the client. The client cannot distinguish missing data from access denied.

```typescript
// Anti-pattern: silent null for auth
loan: async (_, args, ctx) => {
  if (!ctx.user?.roles.includes("MANAGER")) return null; // ❌ — looks like not found
  return ctx.db.loan.findUnique({ where: { id: args.id } });
};
```

**Fix:** Throw a `GraphQLError` with `extensions.code = 'FORBIDDEN'` so clients can distinguish auth errors from absence.

---

### A10. `__typename` in `__resolveType` Based on Snake-Case Heuristics

**Problem:** Guessing the type in `__resolveType` based on field presence or string patterns is fragile — adding a field to one type breaks the discriminator.

```typescript
// Anti-pattern: fragile heuristic
__resolveType(obj) {
  if ('loanId' in obj && 'accountId' in obj) return 'FullRecord';  // ❌ — breaks when you add fields
  if ('loanId' in obj) return 'LoanSummary';
  return 'AccountSummary';
}
```

**Fix:** Always include `__typename` on objects returned from resolvers that participate in interfaces or unions.

```typescript
// Pattern: explicit __typename
__resolveType(obj) {
  return obj.__typename ?? null;  // ✅
}

// In the resolver that builds the object:
return { __typename: 'LoanSummary', ...loan };
```

---

## Quick Reference Card

| #   | Pattern / Anti-pattern       | Verdict         | Key Rule                                                   |
| --- | ---------------------------- | --------------- | ---------------------------------------------------------- |
| P1  | Domain-driven schema         | ✅ Pattern      | Schema = business domain, not DB tables                    |
| P2  | Error union                  | ✅ Pattern      | Business errors in `data`, infra errors in `errors[]`      |
| P3  | Relay Connection             | ✅ Pattern      | All growable lists use Connection from day one             |
| P4  | Command mutations            | ✅ Pattern      | Mutations named by intent: `approveLoan`, not `updateLoan` |
| P5  | Nullable for partial success | ✅ Pattern      | Non-null only when guaranteed                              |
| P6  | Interface / Union            | ✅ Pattern      | Explicit polymorphism — never `JSON`                       |
| P7  | Context DI                   | ✅ Pattern      | All infra via `ctx` — no global imports in resolvers       |
| P8  | Named operations             | ✅ Pattern      | Every client operation has a name                          |
| P9  | Deprecation lifecycle        | ✅ Pattern      | Deprecate → monitor → remove                               |
| P10 | DataLoader for relations     | ✅ Pattern      | Mandatory for all list-parent relation fields              |
| A1  | God schema                   | ❌ Anti-pattern | Split into subgraphs                                       |
| A2  | `JSON` scalar overuse        | ❌ Anti-pattern | Model the shape                                            |
| A3  | DB schema exposed            | ❌ Anti-pattern | Transform at resolver layer                                |
| A4  | Non-null mutation results    | ❌ Anti-pattern | Return payload type                                        |
| A5  | Nested mutations             | ❌ Anti-pattern | Flat mutations with IDs                                    |
| A6  | Unbounded lists              | ❌ Anti-pattern | Always paginate                                            |
| A7  | Input reuses output type     | ❌ Anti-pattern | Separate `input` types                                     |
| A8  | Blocking I/O in resolver     | ❌ Anti-pattern | Always async; offload CPU work                             |
| A9  | Silent null for auth denied  | ❌ Anti-pattern | Throw `FORBIDDEN` GraphQLError                             |
| A10 | Heuristic `__resolveType`    | ❌ Anti-pattern | Always set `__typename` explicitly                         |

## References

- [Principled GraphQL](https://principledgraphql.com/)
- [Shopify GraphQL Design Tutorial](https://github.com/Shopify/graphql-design-tutorial)
- [GraphQL Best Practices — graphql.org](https://graphql.org/learn/best-practices/)
- [Production Ready GraphQL (book)](https://book.productionreadygraphql.com/)
- [The Guild Blog](https://the-guild.dev/blog)
