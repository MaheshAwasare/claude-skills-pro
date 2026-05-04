---
name: scaffold-fullstack-app
description: Scaffold a single-tenant full-stack app — Next.js (App Router) + Postgres + Auth.js + Brevo email + Razorpay/Stripe + observability. Use for solo or small-team apps where every user owns their own data, not multi-tenant SaaS. Less ceremony than scaffold-saas-starter; no orgs, no RLS.
---

# Scaffold a Single-Tenant Full-Stack App

The lighter cousin of `scaffold-saas-starter`. For apps where the tenancy boundary is the *user*, not an organization — internal tools, indie SaaS, side projects, B2C consumer apps. No row-level security ceremony, no Clerk billing, just enough structure to ship.

## When to use

- Indie SaaS / side project — you want production-quality but not enterprise scaffolding.
- Internal tool — small team, single org effectively.
- B2C consumer app where each user owns their own data.
- Pre-PMF, want to ship in days.

## When NOT to use

- B2B with teams/orgs/invites → `scaffold-saas-starter`.
- Need shared workspaces / collaboration → `scaffold-saas-starter`.
- Multi-region or compliance constraints (HIPAA, regulated) — needs more than this scaffold.

## Decisions made for you

| Decision | Choice | Why |
|---|---|---|
| Auth | Auth.js (NextAuth v5) | Free, no vendor lock-in, magic-link + OAuth out of box |
| DB | Postgres + Drizzle | Same as SaaS scaffold, less the RLS overhead |
| Tenancy | `user_id` column on owned rows | No RLS needed; single-tenant boundary |
| Web | Next.js 16 App Router | Matches SaaS scaffold for muscle memory |
| Email | Brevo | Free tier covers most indie launches |
| Payments | Razorpay or Stripe (skip if not needed) | Don't wire until you charge |
| Background jobs | `pg-boss` on same DB | Avoid Redis until needed |
| Deploy | Vercel + Neon (free tiers) | $0 to first user |

## File structure

```
my-app/
  src/
    app/
      (marketing)/page.tsx              # public landing
      (auth)/sign-in/page.tsx
      (app)/                            # authenticated
        dashboard/page.tsx
        settings/page.tsx
      api/
        auth/[...nextauth]/route.ts
        webhooks/
          razorpay/route.ts             # only if billing
          brevo/route.ts
    lib/
      auth.ts                           # auth() helper
      db.ts                              # Drizzle client
      email.ts                           # Brevo wrapper
      billing.ts                         # only if billing
    db/
      schema.ts
      migrations/
  .env.example
  drizzle.config.ts
  next.config.ts
  package.json
  README.md
  docker-compose.yml                    # local Postgres
```

Note: no `apps/` monorepo split. One Next.js project, one DB package. If you outgrow it, lift to `apps/web` later — 30 min refactor.

## Auth.js v5 minimal setup

```ts
// src/lib/auth.ts
import NextAuth from "next-auth";
import GitHub from "next-auth/providers/github";
import { DrizzleAdapter } from "@auth/drizzle-adapter";
import { db } from "./db";

export const { auth, handlers, signIn, signOut } = NextAuth({
  adapter: DrizzleAdapter(db),
  providers: [GitHub],
  pages: { signIn: "/sign-in" },
  callbacks: {
    session: async ({ session, user }) => {
      session.user.id = user.id;
      return session;
    },
  },
});
```

```ts
// src/middleware.ts
export { auth as middleware } from "@/lib/auth";

export const config = {
  matcher: ["/((?!api/auth|_next/static|_next/image|favicon.ico|sign-in).*)"],
};
```

That's it for auth. No Clerk dashboard, no org UI to build, no per-tenant config.

## Schema rule: every owned row has `user_id`

```ts
// src/db/schema.ts
import { pgTable, uuid, text, timestamp, integer } from "drizzle-orm/pg-core";

export const users = pgTable("users", {
  id: text("id").primaryKey(),                  // from Auth.js
  email: text("email").notNull().unique(),
  name: text("name"),
  createdAt: timestamp("created_at").notNull().defaultNow(),
});

export const projects = pgTable("projects", {
  id: uuid("id").primaryKey().defaultRandom(),
  userId: text("user_id").notNull().references(() => users.id, { onDelete: "cascade" }),
  name: text("name").notNull(),
  createdAt: timestamp("created_at").notNull().defaultNow(),
});
```

**Discipline (no RLS net):** every query against `projects` MUST filter by `userId`. Bake this into a helper:

```ts
// src/lib/db-helpers.ts
export async function listMyProjects() {
  const session = await auth();
  if (!session?.user?.id) throw new Error("unauthenticated");
  return db.select().from(projects).where(eq(projects.userId, session.user.id));
}
```

Don't expose raw `db` to route handlers — only helpers that have already filtered by user. This is the *poor person's RLS*.

## What the scaffold emits

1. The directory tree above.
2. Auth.js v5 with GitHub + Google + Email magic-link providers configured (env-toggle).
3. Drizzle schema with `users`, `accounts`, `sessions` (Auth.js tables) + example `projects` table with `user_id`.
4. Brevo email helper with magic-link template.
5. Optional Razorpay/Stripe billing (asks at scaffold time; skips if "not yet").
6. `.env.example` with every var.
7. `docker-compose.yml` for local Postgres.
8. GitHub Actions: typecheck, lint, build, migration dry-run.

## Anti-patterns

- **No `userId` filter on a query** — single bug = full leak. Helper-only access pattern catches it at code review.
- **Storing user data without `onDelete: "cascade"`** — GDPR/DPDP "right to delete" requires the cascade work. Setting it later requires backfill.
- **Auth.js Email provider without configuring Brevo** — magic links fail silently. Wire email at the same time as auth, not later.
- **Building auth UI from scratch** — Auth.js gives you sane defaults. Don't.
- **Wiring billing before you charge** — adds maintenance load with no revenue. Skip until you have a real plan to sell.
- **No `.env.example`** — same problem as in SaaS scaffold; ship it day 1.
- **Vercel + Vercel Postgres** — Vercel Postgres is expensive for small apps. Use Neon free tier; switch later if needed.
- **No background jobs at all, "we'll cron it"** — first delayed task you need (welcome email after signup, retention campaign) becomes a hack. `pg-boss` is 50 lines.

## Verify it worked

- [ ] `pnpm dev` starts Next.js with Postgres connected.
- [ ] GitHub sign-in completes round-trip; user row appears in `users` table.
- [ ] `listMyProjects()` returns 0 rows when called as a different user.
- [ ] Magic link email arrives via Brevo (check spam in test).
- [ ] `pnpm build && pnpm start` succeeds — no dev-only imports leak.
- [ ] First Vercel deploy: < 15 minutes from `git init` to `https://*.vercel.app/sign-in` working.
- [ ] CI passes on a fresh push.
- [ ] `DELETE FROM users WHERE id='xxx'` cascades and removes their projects (no orphans).
