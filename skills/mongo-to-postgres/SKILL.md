---
name: mongo-to-postgres
description: Migrate from MongoDB to Postgres — schema design from documents, query translation, transactional cutover plan with dual-write window, JSONB for unstructured fields, and the operational shift from "no schema" to "managed schema." Use when MongoDB has become a maintenance liability or relational queries dominate access patterns.
---

# Migrate MongoDB → Postgres

The migration is more conceptual than mechanical. MongoDB's "no schema" becomes Postgres's "explicit schema (with JSONB for fluid bits)." Done well, you gain transactions, joins, and 30-year-old query optimization. Done poorly, you reinvent Mongo in Postgres and miss the wins.

## When to use

- Mongo cluster has become a cost / maintenance liability.
- Query patterns are heavily relational (lots of `$lookup`, multi-collection joins).
- Data integrity issues from missing schema constraints.
- Need ACID transactions across collections (yes, Mongo has multi-doc transactions, but Postgres's are first-class).

## When NOT to use

- True document-shaped data (logs, events, analytics) — Mongo is fine; or move to a column store.
- Hyper-write-heavy workloads where you've tuned Mongo's sharding well.
- The team has no Postgres experience and hires can't fill it — operational risk dominates.

## The mental shift

| Mongo | Postgres equivalent |
|---|---|
| Database | Database |
| Collection | Table |
| Document | Row |
| Field (any shape) | Column (typed) — or JSONB for fluid |
| Embedded document | JSONB column or separate table |
| `$lookup` | JOIN |
| Index | Index (more types: B-tree, GIN, BRIN, GiST) |
| Capped collection | Partition + retention |
| `_id` (ObjectId) | UUID or bigserial |
| Replica set | Streaming replication / managed Postgres |
| Sharded cluster | Citus / Yugabyte / app-level sharding (rarely needed) |

## Schema design (the critical step)

### Normalize what's relational

Mongo doc:
```js
{
  _id: ObjectId(...),
  email: "a@b.com",
  orgs: [
    { id: "org_1", role: "admin" },
    { id: "org_2", role: "member" },
  ]
}
```

Postgres:
```sql
CREATE TABLE users (
  id    uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  email text UNIQUE NOT NULL
);

CREATE TABLE memberships (
  user_id uuid REFERENCES users(id) ON DELETE CASCADE,
  org_id  text NOT NULL,
  role    text NOT NULL CHECK (role IN ('admin', 'member', 'viewer')),
  PRIMARY KEY (user_id, org_id)
);
```

Now memberships are queryable, joinable, indexed, and constraint-enforced.

### Keep document-shaped what is document-shaped

User preferences, feature flags per user, settings — fluid, app-evolved. Use JSONB:

```sql
CREATE TABLE users (
  ...
  settings jsonb NOT NULL DEFAULT '{}'::jsonb
);

CREATE INDEX users_settings_theme_idx ON users ((settings->>'theme'));
```

JSONB supports indexing on specific paths, partial indexes on conditions, and the `?`/`@>`/`@?` operators for queries. It's the escape hatch for "we don't want to add a column for this."

**Rule:** if a field is queried with a constraint (filter / sort), it's a column. If it's bag-of-data the app reads-and-writes whole, it's JSONB.

## Query translation

| Mongo | Postgres |
|---|---|
| `find({ status: "active" })` | `WHERE status = 'active'` |
| `find().sort({ createdAt: -1 }).limit(10)` | `ORDER BY created_at DESC LIMIT 10` |
| `find({ orgIds: "org_1" })` | `WHERE org_id = 'org_1'` (after normalization) |
| `aggregate([{$lookup: ...}])` | `JOIN` |
| `aggregate([{$group: {_id: "$status", count: {$sum: 1}}}])` | `SELECT status, COUNT(*) GROUP BY status` |
| `findOneAndUpdate(...)` | `UPDATE ... RETURNING *` |
| `bulkWrite([...])` | `INSERT ... ON CONFLICT ... DO UPDATE` |
| `$text` search | `to_tsvector` + `to_tsquery` (full-text), or `pg_trgm` (fuzzy) |

For developers used to Mongoose's "fluent" API, Drizzle / Kysely / Prisma in Node feel similar; in Python, SQLAlchemy 2.x is the right level. Avoid hand-coded SQL strings everywhere — use a query builder.

## Cutover strategy: dual-write + replay

Big-bang cutover is the wrong move for any non-trivial app. Use **dual-write + replay**:

### Phase 1: prepare (week 0)

- Build the Postgres schema.
- Build a sync worker that replays Mongo oplog → Postgres (or equivalent for your Mongo provider).
- Validate: after running for 24h, both stores have the same data.

### Phase 2: dual-write (weeks 1–2)

- Code path: write to Mongo (primary), then to Postgres (mirror). Reads still from Mongo.
- Sync worker stops; both writes are explicit.
- If Postgres write fails, *do not fail the request* — just alert. (We're still in Mongo-primary mode.)
- Reconciliation job runs nightly: any drift between Mongo and Postgres → log + manual fix.

### Phase 3: read flip (week 3)

- Behind a feature flag, route reads to Postgres.
- A/B by tenant or % traffic. Compare results between stores; alert on mismatch.
- Roll forward when match rate is 100% across a 48h window.

### Phase 4: write flip (week 4)

- Postgres becomes primary write. Mongo becomes mirror.
- Reverse dual-write direction. Now Mongo write failures don't block.

### Phase 5: Mongo retire (week 5+)

- After 1–2 weeks of clean Postgres-primary, stop writing to Mongo.
- Decommission Mongo cluster.

This sequence keeps a rollback path open at every phase. If something goes wrong in phase 3, you flip reads back to Mongo (no data loss).

## Operational changes

| Capability | Mongo | Postgres |
|---|---|---|
| Backups | Atlas / mongodump | `pg_dump` + WAL archiving / managed |
| Replicas | Replica set | Streaming replication |
| Horizontal scale | Sharding | Citus, Yugabyte, or app-level (rarely needed) |
| Schema changes | Code change only | Migrations (golang-migrate, alembic, drizzle-kit, etc) |
| Secondary indexes | Single-collection only | Cross-table via FKs |
| Geo queries | `$geoWithin` | PostGIS |

The biggest operational shift: **schema migrations**. Your team needs to learn migration discipline (dev → staging → prod, reversible, atomic). See `database-migrations` for patterns.

## Performance comparison

For mostly-relational workloads, Postgres usually wins. For document-heavy, write-amplification-tolerant workloads, Mongo can be cheaper at scale.

Don't migrate based on "Postgres is faster" handwaving. Benchmark your actual workload on both. Common surprises:

- Postgres on a single node handles more writes than people expect.
- Mongo's sharding overhead (mongos, balancer) often adds latency; Postgres single-node beats it.
- JSONB queries are *not* slower than native columns at most scales.

## Anti-patterns

- **Big-bang cutover** — irreversible mistakes.
- **Modeling Postgres as if it were Mongo** — single giant `data jsonb` column. You lose all the wins.
- **Modeling Mongo data as if it were Postgres** — tons of small tables for what was naturally embedded. Data is hot-hot, joins everywhere.
- **No dual-write reconciliation job** — drift goes undetected; one customer's data ends up in two states.
- **Skipping the sync worker** — you'll discover edge cases mid-cutover instead of in week 0.
- **Hand-translating queries without a query builder** — SQL injection risk, maintenance pain.
- **Migrating to "make Postgres the only DB" without justification** — sometimes the right answer is Mongo for 20% of your data, Postgres for the 80%.
- **Forgetting indexes** — Mongo's auto-index on `_id` lulls people. Postgres needs explicit `CREATE INDEX` for non-PK lookup columns.
- **Not using JSONB where it fits** — duplicating Mongo's flexibility loss for no gain.
- **Forgetting the team's Postgres skills aren't there** — operational risk during cutover. Train first.

## Verify it worked

- [ ] Schema reviewed: relational fields are columns; document-shaped fields are JSONB.
- [ ] Sync worker runs; Mongo and Postgres data match after a 24h soak.
- [ ] Dual-write succeeds; reconciliation job reports zero drift over 48h.
- [ ] Feature-flagged read traffic routed to Postgres; results match Mongo for 48h+.
- [ ] Write flip completed; Mongo is now mirror.
- [ ] Backups, monitoring, alerting, runbooks set up for Postgres (see `write-runbook`).
- [ ] Indexes match query patterns; `EXPLAIN ANALYZE` on hot queries is fast.
- [ ] Application code's data layer is on a query builder; raw SQL is rare and reviewed.
- [ ] Mongo-related infra (drivers, libraries, monitoring) removed from the codebase after retirement.
- [ ] Cost is materially lower (the usual reason for the migration).
