---
name: scaffold-saas-starter
description: Scaffold a production-ready multi-tenant SaaS starter — Next.js (App Router) + Postgres with row-level security + Clerk organizations + Razorpay/Stripe billing + Brevo email + observability. Use when starting a new SaaS app and you want the boring decisions made well, not when adding tenancy to an existing app.
---

# Scaffold a Multi-Tenant SaaS Starter

The opinionated, "I want to ship in a week, not architect for three months" SaaS scaffold. Multi-tenant by row-level security (not separate databases), org-based not user-based, billing wired from day one.

## When to use

- New SaaS product, day 1.
- You know it'll be multi-tenant (B2B with teams/orgs).
- You're OK committing to: Next.js App Router, Postgres, Clerk, Razorpay or Stripe, Brevo, Vercel/Railway.

## When NOT to use

- Adding tenancy to an existing app — surgery, not scaffolding. Different problem.
- Single-tenant SaaS (one customer, one DB) — use `scaffold-fullstack-app` instead.
- B2C consumer app where "user" is the tenancy boundary — over-engineered.
- You need separate-DB-per-tenant (regulated/HIPAA/sovereignty) — this scaffold uses RLS, not DB-per-tenant.

## The architectural decisions (made for you)

| Decision | Choice | Why |
|---|---|---|
| Tenancy model | Row-level security in Postgres | One DB, one schema; cheapest scale; auditable |
| Tenant identity | Clerk Organization | Outsource org/team UI + invites + SSO |
| Auth | Clerk | Free tier covers 10k MAU, no auth bugs to ship |
| DB | Postgres + Drizzle | Drizzle migrations + RLS friendliness > Prisma |
| Web | Next.js 16 App Router | Server Components reduce auth-passing boilerplate |
| Background jobs | Postgres + `pg-boss` | One less infra dep until you outgrow it |
| Email | Brevo (or Resend if budget allows) | See `brevo-email` skill |
| Payments | Razorpay (India) or Stripe (global) | See respective skills |
| Observability | OpenTelemetry → Grafana Cloud free tier | One vendor, traces+metrics+logs |
| Deploy | Vercel (web) + Railway/Neon (DB) | Lowest friction to first prod deploy |

## File structure

```
my-saas/
  apps/
    web/                          # Next.js
      src/
        app/
          (marketing)/            # public pages
          (auth)/                 # sign-in, sign-up
          (app)/                  # authenticated app
            [orgSlug]/            # tenant scope in URL
              dashboard/
              billing/
              settings/
          api/
            webhooks/
              clerk/route.ts      # org/user sync
              razorpay/route.ts   # payments
              brevo/route.ts      # email events
        lib/
          db.ts                   # Drizzle client (sets app.org_id GUC)
          auth.ts                 # Clerk helpers, getOrgId()
          billing.ts              # plan -> price mapping
          email.ts                # Brevo wrapper
          rls.ts                  # withTenant(orgId, fn)
        middleware.ts             # Clerk + org redirect
  packages/
    db/
      schema.ts                   # Drizzle tables
      migrations/                 # SQL incl. RLS policies
      seed.ts
    config/                       # shared eslint, tsconfig
  .github/workflows/ci.yml
  docker-compose.yml              # local Postgres
  .env.example
  README.md
```

## The non-obvious bit: Postgres RLS for tenancy

Every tenant-scoped table has an `org_id` column and a policy:

```sql
-- packages/db/migrations/0001_init.sql
CREATE TABLE projects (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id      text NOT NULL,                -- Clerk org id (org_xxx)
  name        text NOT NULL,
  created_at  timestamptz NOT NULL DEFAULT now()
);

ALTER TABLE projects ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON projects
  USING (org_id = current_setting('app.org_id', true));
```

Then your DB client *always* sets `app.org_id` per request:

```ts
// apps/web/src/lib/rls.ts
import { db } from "./db";
import { sql } from "drizzle-orm";

export async function withTenant<T>(orgId: string, fn: (tx: typeof db) => Promise<T>): Promise<T> {
  return db.transaction(async (tx) => {
    await tx.execute(sql`SELECT set_config('app.org_id', ${orgId}, true)`);
    return fn(tx);
  });
}
```

Now even if a developer forgets a `WHERE org_id = ?` clause, Postgres rejects the cross-tenant read. **This is the single most important pattern in the scaffold** — it converts a "one bug = data leak" risk into "one bug = empty result."

## What the scaffold emits (when this skill runs)

Running this skill produces:

1. The directory tree above.
2. **Drizzle schema** with `org_id` on every business table + RLS policies.
3. **Clerk middleware** that:
   - Redirects `/` to `/<defaultOrgSlug>/dashboard` if signed in.
   - Forces org selection if user has none.
   - Exposes `getOrgId()` server helper.
4. **Razorpay or Stripe** scaffold (asks which), wired to a `subscriptions` table and a `subscription.charged` webhook.
5. **Brevo** scaffold with template IDs in `email/templates.ts` and SPF/DKIM/DMARC docs in README.
6. **OpenTelemetry** auto-instrumentation for HTTP + Postgres + outbound HTTP.
7. **GitHub Actions** with: typecheck, lint, build, drizzle migration check, OWASP dep scan.
8. **README** with first-run instructions including the RLS test (`SELECT *` from another tenant returns 0 rows).
9. **`.env.example`** listing every required secret with comments.

## The three things to do *before* writing code

1. **Decide org vs project as tenant boundary.** Org = team (Slack-like). Project = workspace inside an org (Linear-like). The scaffold defaults to org. If you need both, add a `project_id` column and a second RLS policy — don't try to retrofit later.
2. **Pick payment provider once.** Don't dual-wire. Switching is a refactor; supporting both is a tax on every billing change.
3. **Set the data residency story.** EU customers? Run Postgres in EU. Indian DPDP? See `india-dpdp-act` skill. Decide before the first row of customer data lands.

## Anti-patterns

- **`org_id = req.params.orgId`** in queries instead of session-derived. URL params can be tampered with. Always derive from Clerk session server-side.
- **Schema-per-tenant** as a first move — kills migration ergonomics, query consolidation, and Postgres dies at ~10k schemas. Use RLS.
- **Building org/team/invite UI from scratch** — Clerk/Auth0 give you this. Saves a month.
- **Skipping RLS, "we'll add WHERE clauses everywhere"** — one missed clause = breach. RLS is the safety net; turn it on day 1.
- **Stripe + Razorpay both wired** — pick one. Geographic segmentation is the only reason to dual-wire, and it's almost never worth the complexity until you have signal.
- **Logs/metrics afterthought** — OTel instrumentation adds 30 lines if done now, 3 weeks if added later when you're firefighting.
- **No `.env.example`** — onboarding pain compounds. Every required secret listed with a comment, day 1.
- **`createdAt` in app code instead of DB default** — clock skew bugs. `DEFAULT now()` always.

## Verify it worked

- [ ] User in Org A cannot read Org B's projects: `SET app.org_id = 'org_B'; SELECT * FROM projects WHERE org_id = 'org_A';` returns 0 rows.
- [ ] Forgetting `withTenant()` causes the query to return 0 rows (RLS engaged), not all rows.
- [ ] Razorpay/Stripe webhook successfully creates a subscription row, idempotent on retry.
- [ ] Brevo welcome email fires on org creation; template ID is in code.
- [ ] OpenTelemetry trace shows: `request → Clerk → Postgres withTenant → Brevo` as a single trace.
- [ ] CI passes typecheck, lint, build, drizzle dry-run migration.
- [ ] `.env.example` has zero real secrets and every var the app reads.
- [ ] First deploy to Vercel + Neon takes < 30 minutes from scaffold to login.
