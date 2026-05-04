---
name: graphql-relay
description: Build a GraphQL API following Relay specifications — schema design, global IDs, connections (cursor pagination), DataLoader for N+1, persisted queries, auth at field level, and the operational discipline that prevents schema drift. Use when designing a new GraphQL API, not when adding fields to an existing one.
---

# GraphQL with Relay Conventions

GraphQL is two products: a query language (great) and a server pattern (foot-guns everywhere). Relay's spec is the cleaned-up server pattern. Adopt it even if your client isn't Relay — it eliminates 80% of GraphQL pathologies.

## When to use

- New GraphQL API.
- You need fine-grained client-controlled data fetching (multi-team mobile/web).
- Federation across services with Apollo Federation v2.

## When NOT to use

- Simple CRUD app — REST or tRPC is faster to build.
- Hyper-low-latency public API — query parsing + N+1 risk overhead.
- Internal admin tool — overkill.

## Core Relay conventions

1. **Global IDs.** Every node has a globally unique opaque ID, base64-encoded `{type}:{rowid}`.
2. **`Node` interface.** Any object with a global ID implements `Node`.
3. **Connections.** Lists are `<Type>Connection` with `edges`, `pageInfo`, cursor pagination.
4. **Mutations as input/payload pairs.** `Input!` argument, `Payload` return with `clientMutationId`.

These four conventions remove ambiguity and enable client-side caching that actually works.

## Schema example

```graphql
interface Node {
  id: ID!
}

type User implements Node {
  id: ID!                                          # global ID
  email: String!
  projects(first: Int, after: String): ProjectConnection!
}

type Project implements Node {
  id: ID!
  name: String!
  owner: User!
}

type ProjectConnection {
  edges: [ProjectEdge!]!
  pageInfo: PageInfo!
}

type ProjectEdge {
  node: Project!
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}

type Query {
  node(id: ID!): Node                              # generic fetch by global ID
  viewer: User                                     # authenticated user
}

input CreateProjectInput {
  name: String!
  clientMutationId: String                         # for optimistic updates
}

type CreateProjectPayload {
  project: Project!
  clientMutationId: String
}

type Mutation {
  createProject(input: CreateProjectInput!): CreateProjectPayload!
}
```

## Global IDs

```ts
import { Buffer } from "node:buffer";

export function toGlobalId(type: string, id: string): string {
  return Buffer.from(`${type}:${id}`).toString("base64url");
}

export function fromGlobalId(globalId: string): { type: string; id: string } {
  const [type, id] = Buffer.from(globalId, "base64url").toString().split(":");
  return { type, id };
}
```

Why opaque: clients can't accidentally couple to the underlying ID format (UUID vs autoincrement vs Stripe ID). You can refactor freely.

## DataLoader (the only solution to N+1)

Without DataLoader, asking for 100 projects each with `.owner` runs 101 queries. With DataLoader, batched into one.

```ts
import DataLoader from "dataloader";

export function makeUserLoader(db) {
  return new DataLoader<string, User | null>(async (ids) => {
    const rows = await db.user.findMany({ where: { id: { in: [...ids] } } });
    const byId = new Map(rows.map((r) => [r.id, r]));
    return ids.map((id) => byId.get(id) ?? null);
  });
}

// In context (per-request)
export async function context({ req }) {
  return {
    userId: await verifyAuth(req),
    loaders: {
      user: makeUserLoader(db),
      project: makeProjectLoader(db),
    },
  };
}

// In resolver
const Project = {
  owner: (parent, _args, { loaders }) => loaders.user.load(parent.ownerId),
};
```

**Critical:** loaders are *per-request*. Caching across requests = stale data leaks. New context, new loaders.

## Connection pagination (cursor, not offset)

```graphql
projects(first: 20, after: "Y3Vyc29yMTIz")
```

```ts
// Resolver
async function projects(_parent, { first = 20, after }, ctx) {
  const cursor = after ? decodeCursor(after) : null;
  const rows = await db.project.findMany({
    where: cursor ? { createdAt: { lt: cursor.createdAt } } : {},
    orderBy: { createdAt: "desc" },
    take: first + 1,                     // fetch one extra to detect hasNextPage
  });
  const hasNextPage = rows.length > first;
  const nodes = rows.slice(0, first);
  return {
    edges: nodes.map((n) => ({ node: n, cursor: encodeCursor({ createdAt: n.createdAt }) })),
    pageInfo: {
      hasNextPage,
      hasPreviousPage: !!after,
      startCursor: nodes[0] && encodeCursor({ createdAt: nodes[0].createdAt }),
      endCursor: nodes.at(-1) && encodeCursor({ createdAt: nodes.at(-1)!.createdAt }),
    },
  };
}
```

Offset pagination breaks under writes (rows shift). Cursor pagination is stable.

## Persisted queries (security + perf)

In production, accept only pre-registered queries by hash. Clients send `{ extensions: { persistedQuery: { sha256Hash: "..." } } }` instead of the raw GraphQL.

Benefits:
- **Smaller payloads** (bytes, not strings).
- **Attack surface reduction** — can't run arbitrary queries.
- **CDN-cacheable** — GET request with hash in URL.

Apollo Server has built-in support; tooling auto-generates the hash registry from your client codebase.

## Auth at the field level

```ts
const Project = {
  // Public field
  name: (parent) => parent.name,
  // Owner-only field
  apiKey: (parent, _args, ctx) => {
    if (parent.ownerId !== ctx.userId) throw new Error("forbidden");
    return parent.apiKey;
  },
};
```

For more than 3 such checks, use a directive:
```graphql
type Project {
  name: String!
  apiKey: String! @auth(role: OWNER)
}
```

## Query complexity / depth limits

```ts
import { createComplexityLimitRule } from "graphql-validation-complexity";
import depthLimit from "graphql-depth-limit";

const server = new ApolloServer({
  schema,
  validationRules: [
    depthLimit(10),
    createComplexityLimitRule(1000, { onCost: (cost) => log.info(cost) }),
  ],
});
```

Without these, a malicious client can submit `{ user { friends { friends { friends { ... } } } } }` and DoS your DB.

## Anti-patterns

- **N+1 without DataLoader** — guaranteed perf bug. Loader-everything from day 1.
- **Returning database row IDs as `id`** — couples client to schema; can't refactor. Use global IDs.
- **Offset pagination (`limit/offset`)** — breaks under writes, doesn't scale past ~10k rows.
- **No depth/complexity limits** — DoS waiting to happen.
- **One huge resolver file** — split per type. `User.ts`, `Project.ts`, etc.
- **Mutations that return only `Boolean`** — clients can't update local cache. Always return the affected entity.
- **Loader cache shared across requests** — stale-data leak between users.
- **Persisting queries skipped for "later"** — almost never gets done. Bake into client codegen from day 1.
- **No `viewer` query** — every client has to remember its own user ID. `viewer` is the canonical "who am I?" entry point.
- **Public-field auth in resolver code** — easy to forget. Use directives or shadow types.
- **Federation v1** — deprecated. v2 only for new projects.

## Verify it worked

- [ ] Fetching 100 projects with their owners hits DB ≤ 2 times (one for projects, one batched for users).
- [ ] Asking for `node(id: "<globalId>")` returns the correct typed entity.
- [ ] Pagination is cursor-based; mid-write list is consistent (no skipped or duplicate rows).
- [ ] Production accepts only persisted queries; raw GraphQL POST is rejected (or feature-flagged off).
- [ ] Query depth > 10 is rejected with a useful error.
- [ ] Per-request DataLoader; user A's request never sees cached entities from user B's.
- [ ] `viewer` query returns the authenticated user; null when unauthenticated.
- [ ] Mutations return the affected entity, not just `Boolean`.
- [ ] Field-level auth: querying `apiKey` of someone else's project errors, doesn't leak.
- [ ] Schema is in version control; CI runs schema diff and fails on breaking changes.
