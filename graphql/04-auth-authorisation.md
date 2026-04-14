# Authentication & Authorisation

## Category

GraphQL — Security & Auth

## Context

GraphQL collapses multiple endpoints into a single `/graphql` entry point — this changes how auth is applied. **Authentication** (who are you?) is handled at the HTTP layer before the request enters the GraphQL engine. **Authorisation** (what can you do?) must be enforced at the resolver or directive level because different fields within the same query can have different access requirements.

### Auth Enforcement Layers

| Layer | Mechanism | Granularity | When to Use |
|-------|-----------|------------|------------|
| **HTTP middleware** | JWT verify, session check | Per-request | All auth — reject unauthenticated requests before GraphQL parse |
| **Context** | Attach decoded user to `ctx.user` | Per-request | Provide auth data to every resolver |
| **Root resolver** | `if (!ctx.user) throw` | Per operation | Quick gate on sensitive mutations/queries |
| **Schema directive** | `@auth(requires: ADMIN)` on field/type | Per field or type | Declarative, DRY, visible in schema |
| **Service layer** | Authorisation inside business logic | Per entity/row | Row-level security, ownership checks (`loan.borrowerId === ctx.user.id`) |

### Auth Directive Approach

Implementing a custom `@auth` directive transforms field execution — it wraps the original resolver with a permission check:

```graphql
directive @auth(requires: Role!) on FIELD_DEFINITION | OBJECT

enum Role {
  USER
  MANAGER
  ADMIN
}

type Mutation {
  disburseLoan(id: ID!): DisburseLoanPayload! @auth(requires: MANAGER)
  writeLoanOff(id: ID!): WriteOffPayload!     @auth(requires: ADMIN)
}
```

### RBAC Matrix Example

| Role | `loan(id)` | `loans(filter)` | `createLoan` | `disburseLoan` | `writeLoanOff` |
|------|-----------|----------------|-------------|---------------|---------------|
| Unauthenticated | ❌ | ❌ | ❌ | ❌ | ❌ |
| USER | Own loans only | Own loans only | ✅ | ❌ | ❌ |
| MANAGER | All | All | ✅ | ✅ | ❌ |
| ADMIN | All | All | ✅ | ✅ | ✅ |

## Pros

- Directive-based auth centralises access policy in the schema — devs can see permissions without reading resolver code
- Context injection makes the auth user available everywhere without prop-drilling
- Field-level auth enables fine-grained data access — sensitive fields (e.g. `Account.taxId`) can be protected independently
- DRY: `@auth` on a type applies to all its fields — no need to annotate every field individually

## Cons

- Schema directives require a directive transformer or plugin — adds build complexity (MapperKind, `mapSchema`)
- Directive-based auth is not a substitute for business-logic ownership checks — a MANAGER can disburse any loan, not just theirs, unless the resolver also checks ownership
- Unauthenticated introspection leaks your entire data model — disable in production or apply auth to introspection
- Returning different data based on roles from the same field (e.g. masking `ssn` for non-admins) requires resolver logic, not directives

## Design Diagram

```mermaid
flowchart TD
    HTTP[HTTP Request<br/>Authorization: Bearer JWT]
    MW[Express Middleware<br/>verifyJwt → ctx.user]
    GQL[GraphQL Engine<br/>Parse + Validate]
    DIR[@auth Directive Transformer<br/>wraps field resolvers]
    R[Resolver Execution]
    SVC[Service Layer<br/>Ownership check<br/>loan.borrowerId === ctx.user.id]
    DB[(Database)]

    HTTP --> MW -->|ctx.user attached| GQL --> DIR --> R --> SVC --> DB

    style DIR fill:#f0ad4e,color:#000
    style SVC fill:#5bc0de,color:#000
```

## Code Sample

### TypeScript — JWT middleware + context (Express + graphql-yoga)

```typescript
import { createYoga } from 'graphql-yoga';
import express from 'express';
import { jwtVerify, createRemoteJWKSet } from 'jose';

const JWKS_URI = process.env.JWKS_URI!; // e.g. https://auth.example.com/.well-known/jwks.json
const jwks = createRemoteJWKSet(new URL(JWKS_URI));

const yoga = createYoga({
  schema,
  context: async ({ request }) => {
    const auth = request.headers.get('authorization') ?? '';
    const token = auth.startsWith('Bearer ') ? auth.slice(7) : null;

    let user: AppUser | null = null;
    if (token) {
      try {
        const { payload } = await jwtVerify(token, jwks, {
          issuer:   process.env.JWT_ISSUER!,
          audience: process.env.JWT_AUDIENCE!,
        });
        user = {
          id:    payload.sub!,
          email: payload.email as string,
          roles: (payload.roles as string[]) ?? [],
        };
      } catch {
        // Invalid token → unauthenticated context; individual resolvers decide how to respond
      }
    }

    return { user, db, loaders: buildLoaders(db) };
  },
});
```

### TypeScript — `@auth` directive transformer (graphql-tools mapSchema)

```typescript
import { mapSchema, getDirective, MapperKind } from '@graphql-tools/utils';
import { defaultFieldResolver, GraphQLSchema, GraphQLError } from 'graphql';

export function applyAuthDirective(schema: GraphQLSchema): GraphQLSchema {
  return mapSchema(schema, {
    [MapperKind.OBJECT_FIELD](fieldConfig) {
      const directive = getDirective(schema, fieldConfig, 'auth')?.[0];
      if (!directive) return fieldConfig;

      const requiredRole: string = directive['requires'];
      const { resolve = defaultFieldResolver } = fieldConfig;

      return {
        ...fieldConfig,
        resolve(source, args, context, info) {
          const user: AppUser | null = context.user;

          if (!user) {
            throw new GraphQLError('Unauthenticated', {
              extensions: { code: 'UNAUTHENTICATED', http: { status: 401 } },
            });
          }

          if (!user.roles.includes(requiredRole)) {
            throw new GraphQLError(
              `Requires role ${requiredRole}`,
              { extensions: { code: 'FORBIDDEN', http: { status: 403 } } }
            );
          }

          return resolve(source, args, context, info);
        },
      };
    },
  });
}
```

### TypeScript — Field-level masking (row-level security in resolver)

```typescript
const resolvers = {
  Account: {
    // SSN only visible to ADMIN role
    taxId: (parent: Account, _args: unknown, ctx: AppContext): string | null => {
      if (!ctx.user?.roles.includes('ADMIN')) return null;
      return parent.taxId;
    },

    // Loans — filter to own loans for USER role
    loans: async (parent: Account, args: QueryLoansArgs, ctx: AppContext) => {
      const filter = { ...args.filter };

      if (ctx.user?.roles.includes('USER') && !ctx.user.roles.includes('MANAGER')) {
        // Users can only see loans where they are the borrower
        filter.borrowerAccountId = ctx.user.id;
        if (filter.borrowerAccountId !== parent.id) return { edges: [], pageInfo: emptyPageInfo(), totalCount: 0 };
      }

      return ctx.loaders.loanConnection.load({ ...args, filter });
    },
  },
};
```

## References

- [GraphQL Auth Recipes — The Guild](https://the-guild.dev/graphql/yoga-server/docs/features/auth-protect-queries)
- [graphql-shield — Rule-based auth](https://the-guild.dev/graphql/shield)
- [mapSchema directive transformer](https://the-guild.dev/graphql/tools/docs/schema-directives)
- [JWT Verification with jose](https://github.com/panva/jose)
- [OWASP GraphQL Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/GraphQL_Cheat_Sheet.html)
