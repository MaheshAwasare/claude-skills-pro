---
name: supabase-backend
description: Use Supabase as a managed Postgres + auth + storage + realtime backend — RLS policies, edge functions, the auth-and-RLS handshake, Realtime channels, and the migration discipline that prevents prod drift. Use when starting a new app on Supabase, not when adding Supabase features piecemeal to an existing one.
---

# Supabase Backend

Supabase is "Firebase but Postgres." Best fit: small teams, prototype-to-prod path, want auth+DB+storage+realtime in one bill. Trade-off: row-level security is your security model — get it wrong and your whole DB is exposed.

## When to use

- New app, want managed Postgres + auth + storage + realtime in one provider.
- Frontend-heavy apps (Next.js, Expo) where direct DB access from the client is desirable.
- Replacing Firebase for relational use cases.

## When NOT to use

- You need fine-grained server-side control of every query → use plain Postgres + your own API.
- Stateless services on Cloudflare/Vercel only — Supabase is overkill.
- Latency-sensitive at scale — Supabase RPC adds ~50ms over direct Postgres.

## The mental model: RLS is your security

In Supabase, your client (browser/mobile) gets a JWT and queries Postgres directly via PostgREST. **The only thing protecting your data is row-level security.** No RLS = full DB exposed to anyone with your anon key (which is in your client bundle).

```sql
-- Always: enable RLS on every table
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;

-- Then: explicit policies for each operation
CREATE POLICY "users see own projects" ON projects
  FOR SELECT USING (auth.uid() = user_id);

CREATE POLICY "users insert own projects" ON projects
  FOR INSERT WITH CHECK (auth.uid() = user_id);

CREATE POLICY "users update own projects" ON projects
  FOR UPDATE USING (auth.uid() = user_id);

CREATE POLICY "users delete own projects" ON projects
  FOR DELETE USING (auth.uid() = user_id);
```

`auth.uid()` is a built-in function returning the user's UUID from the JWT. **`USING` clause** filters which rows the user sees. **`WITH CHECK` clause** validates which rows they can insert/update.

## Schema rule: every business table has `user_id` (or `org_id`)

```sql
CREATE TABLE projects (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     uuid NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  name        text NOT NULL,
  created_at  timestamptz NOT NULL DEFAULT now()
);
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;
```

`auth.users` is Supabase's managed users table. Don't fight it; reference it.

## Client setup (Next.js)

```ts
// src/lib/supabase/server.ts
import { createServerClient } from "@supabase/ssr";
import { cookies } from "next/headers";

export async function getSupabase() {
  const cookieStore = await cookies();
  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll: () => cookieStore.getAll(),
        setAll: (toSet) => toSet.forEach(({ name, value, options }) => cookieStore.set(name, value, options)),
      },
    }
  );
}
```

```ts
// src/lib/supabase/client.ts (for client components)
import { createBrowserClient } from "@supabase/ssr";

export const supabase = createBrowserClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
);
```

The anon key is fine to ship — RLS is what gates access. The **service_role key** is the dangerous one (bypasses RLS); never ship it.

## Migrations: use the CLI, never the dashboard

```bash
# Local dev
supabase start                           # spins up Docker Postgres
supabase migration new add_projects      # creates supabase/migrations/<ts>_add_projects.sql
# Edit the SQL
supabase db reset                        # apply locally
supabase db push                         # apply to remote linked project
```

Schema changes via the dashboard SQL editor are **not** in migrations and won't replay on a fresh dev DB. Hard rule: any DDL goes through `supabase migration new`.

## Auth helpers

```ts
// Sign up (emails OTP / verification link via configured SMTP)
const { data, error } = await supabase.auth.signUp({ email, password });

// Sign in
await supabase.auth.signInWithPassword({ email, password });

// Magic link (passwordless)
await supabase.auth.signInWithOtp({ email });

// Get current user (server)
const { data: { user } } = await supabase.auth.getUser();
```

Configure SMTP in dashboard → Auth → Email. Default uses Supabase's relay which is rate-limited and from `noreply@mail.app.supabase.io` — switch to Brevo/Resend/SES before prod.

## Realtime

Subscribe to row-level changes:
```ts
supabase
  .channel("projects")
  .on("postgres_changes", { event: "*", schema: "public", table: "projects", filter: `user_id=eq.${userId}` }, (payload) => {
    console.log("change:", payload);
  })
  .subscribe();
```

Realtime respects RLS — users only get events for rows they can read. Convenient *and* secure.

## Storage

```ts
const { data, error } = await supabase.storage.from("avatars").upload(`${userId}/profile.png`, file);
const { data: { publicUrl } } = supabase.storage.from("avatars").getPublicUrl(`${userId}/profile.png`);
```

Bucket policies use the same RLS-style:
```sql
CREATE POLICY "users upload to own folder" ON storage.objects
  FOR INSERT WITH CHECK (bucket_id = 'avatars' AND (storage.foldername(name))[1] = auth.uid()::text);
```

## Edge Functions (Deno) for server-only logic

```ts
// supabase/functions/charge/index.ts
import { serve } from "https://deno.land/std/http/server.ts";

serve(async (req) => {
  const { customerId, amount } = await req.json();
  // Use service_role here for admin actions
  // ...
  return new Response(JSON.stringify({ ok: true }));
});
```

Edge Functions are where you put webhooks and admin operations that need to bypass RLS. They run on Deno + Cloudflare Workers infra; cold start ~50ms.

## Anti-patterns

- **Tables without `ENABLE ROW LEVEL SECURITY`** — exposed to anyone with anon key. Add to migrations day 1.
- **Forgetting `WITH CHECK` on INSERT/UPDATE policies** — users can write rows they couldn't read. Asymmetric data leak.
- **Service_role key in frontend** — full DB bypass. Ship anon key only; service_role lives in edge functions / your backend.
- **Schema changes via dashboard SQL editor** — not in migrations, drift between dev and prod.
- **Default Supabase email sender in prod** — rate limits hit users; emails land in spam. Configure SMTP before launch.
- **Not handling `auth.users` cascade** — referenced from many tables; if you skip `ON DELETE CASCADE`, deletions break.
- **Storing PII without RLS** — Supabase's Postgres is just Postgres; same compliance posture (GDPR/DPDP) applies.
- **No local Postgres for dev** — `supabase start` is one command. Dev-against-prod is how data gets corrupted.
- **Realtime channel without `filter`** — fires for every row in the table, network and RLS overhead.
- **`select *` in production queries** — every PostgREST endpoint accepts column selection. Be explicit; reduces bandwidth and accidental column leaks.

## Verify it worked

- [ ] Every table in `public` schema has RLS enabled (`SELECT relname FROM pg_class WHERE relrowsecurity = false AND relnamespace = 'public'::regnamespace` returns 0 user-tables).
- [ ] User A cannot read User B's projects (manual test with two test accounts).
- [ ] `WITH CHECK` clauses present on all INSERT/UPDATE policies.
- [ ] Service_role key is in `.env.local` only, not committed; not loaded in browser bundle.
- [ ] All schema changes are in `supabase/migrations/`; no dashboard-only DDL.
- [ ] `supabase db reset` rebuilds DB cleanly from migrations.
- [ ] Production SMTP configured; magic-link email arrives within 30s, lands in inbox.
- [ ] Realtime channel only fires for rows the user can read (manual test).
- [ ] Storage bucket has folder-isolation policy; user can't write into another user's folder.
- [ ] Edge function works with `service_role` for admin actions; doesn't expose service key to client.
