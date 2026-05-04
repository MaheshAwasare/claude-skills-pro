---
name: clerk-auth
description: Integrate Clerk for authentication and team/org management — JWT verification on backends, organizations and roles, webhook sync to your DB, and the SSR/middleware patterns for Next.js App Router. Use when adding auth to a new app and outsourcing UI/orgs/invites instead of building Auth.js from scratch.
---

# Clerk Auth

Clerk's pitch: skip building 6 weeks of auth UI (sign-in, sign-up, password reset, email verification, MFA, invitations, orgs, roles). Free up to 10k MAU. Trade-offs: vendor dependency, opinionated UI surfaces, more expensive at scale than Auth.js.

## When to use

- B2B SaaS with teams/orgs/invites — Clerk's Organization model is the killer feature.
- You want sign-in UI in 30 minutes, not 30 hours.
- Compliance posture (SOC2, audit logs) outsourced.
- You'll spend more on engineering hours building auth than Clerk's monthly cost.

## When NOT to use

- B2C consumer scale (>100k MAU) where Clerk's pricing curves up — Auth.js + your own DB.
- Strict data residency (must keep all PII on your infra) — self-host (Keycloak) or Auth.js.
- You need deeply custom flows Clerk's hooks don't support.

## Core concepts

| Concept | Meaning |
|---|---|
| **User** | A person, by email/phone |
| **Organization** | A team/workspace; users have roles within it |
| **Membership** | User × Org with a role (`admin`/`basic_member` or custom) |
| **Session** | A signed JWT for a user, scoped to an active org |
| **Webhook** | Clerk → your server on user/org changes |

## Next.js App Router setup (the pattern)

```bash
pnpm add @clerk/nextjs
```

```tsx
// src/app/layout.tsx
import { ClerkProvider } from "@clerk/nextjs";

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <ClerkProvider>
      <html lang="en"><body>{children}</body></html>
    </ClerkProvider>
  );
}
```

```ts
// src/middleware.ts
import { clerkMiddleware, createRouteMatcher } from "@clerk/nextjs/server";

const isProtected = createRouteMatcher(["/dashboard(.*)", "/api/(?!public)(.*)"]);

export default clerkMiddleware((auth, req) => {
  if (isProtected(req)) auth.protect();
});

export const config = {
  matcher: ["/((?!_next|.*\\..*).*)", "/(api|trpc)(.*)"],
};
```

```tsx
// src/app/sign-in/[[...sign-in]]/page.tsx
import { SignIn } from "@clerk/nextjs";
export default function Page() { return <SignIn />; }
```

That's the whole UI. You can theme via `appearance` prop or replace entirely with `<SignIn.Root>` headless components.

## Server-side: getting the user/org

```ts
// In a Server Component or Route Handler
import { auth, currentUser } from "@clerk/nextjs/server";

export async function GET() {
  const { userId, orgId, orgRole } = await auth();
  if (!userId) return new Response("unauth", { status: 401 });
  if (!orgId)  return new Response("pick an org", { status: 400 });
  // userId is e.g. "user_2abc..."
  // orgId  is e.g. "org_2def..."
  // orgRole is e.g. "org:admin"
  return Response.json({ userId, orgId });
}
```

`auth()` is the only thing you need 90% of the time. `currentUser()` fetches full profile (slower, costs an API call).

## Verifying Clerk JWT in non-Next backend (Go example)

```go
import (
    "github.com/clerkinc/clerk-sdk-go/v2"
    clerkhttp "github.com/clerkinc/clerk-sdk-go/v2/http"
)

clerk.SetKey(os.Getenv("CLERK_SECRET_KEY"))

mux := http.NewServeMux()
mux.Handle("/api/", clerkhttp.RequireSession(myHandler))
```

Inside handler:
```go
sess, _ := clerk.SessionClaimsFromContext(r.Context())
userID := sess.Subject       // user_xxx
orgID, _ := sess.ActiveOrganizationID()
```

## Webhook sync to your DB

You want a *local mirror* of users/orgs in your DB for joins, RLS, foreign keys. Clerk webhooks keep it in sync.

```ts
// src/app/api/webhooks/clerk/route.ts
import { Webhook } from "svix";
import { headers } from "next/headers";

export async function POST(req: Request) {
  const payload = await req.text();
  const h = await headers();
  const wh = new Webhook(process.env.CLERK_WEBHOOK_SECRET!);
  let evt: any;
  try {
    evt = wh.verify(payload, {
      "svix-id":         h.get("svix-id")!,
      "svix-timestamp":  h.get("svix-timestamp")!,
      "svix-signature":  h.get("svix-signature")!,
    });
  } catch {
    return new Response("bad sig", { status: 400 });
  }

  switch (evt.type) {
    case "user.created":
    case "user.updated":
      await db.user.upsert({
        where:  { id: evt.data.id },
        create: { id: evt.data.id, email: evt.data.email_addresses[0].email_address },
        update: { email: evt.data.email_addresses[0].email_address },
      });
      break;
    case "user.deleted":
      await db.user.delete({ where: { id: evt.data.id } });
      break;
    case "organization.created":
    case "organization.updated":
      await db.org.upsert({
        where:  { id: evt.data.id },
        create: { id: evt.data.id, name: evt.data.name, slug: evt.data.slug },
        update: { name: evt.data.name, slug: evt.data.slug },
      });
      break;
    case "organizationMembership.created":
    case "organizationMembership.updated":
      await db.membership.upsert(/* ... */);
      break;
  }
  return new Response("ok");
}
```

Configure webhook URL in Clerk Dashboard → Webhooks. Subscribe to all `user.*`, `organization.*`, `organizationMembership.*` events.

## Roles + permissions

Clerk supports custom roles per org with permissions strings. Default is `org:admin` and `org:member`.

```ts
const { has } = await auth();
if (!has({ permission: "org:billing:manage" })) {
  return new Response("forbidden", { status: 403 });
}
```

Define permissions in Dashboard → Roles & Permissions. Server-side checks always; never trust the client to enforce roles.

## Anti-patterns

- **Storing user/org rows only at first login** — leaves orphans. Webhook-sync from day 1.
- **Reading `evt.data.email_addresses[0]`** without null check — users can have no verified email mid-flow. Always defensive.
- **Trusting `orgRole` from client** — `useOrganization()` returns it but server-side `auth()` is the only authoritative source.
- **Building org/team UI from scratch** — defeats the point. Use `<OrganizationSwitcher>` and `<OrganizationProfile>`.
- **Forgetting `<ClerkProvider>` in layout** — hooks fail silently.
- **Using `currentUser()` everywhere** — costs an API call per. `auth()` returns userId/orgId from JWT, no API roundtrip.
- **Webhook without signature verification** — anyone can post fake user-create events to your endpoint.
- **Multi-tenant rows without `org_id` foreign key to your synced `org` table** — RLS depends on this.
- **Treating Clerk user_id as nullable in DB** — it's the primary key from day 1; never null.
- **Test/prod use same Clerk instance** — leaks emails to real users on staging. Separate instances.

## Verify it worked

- [ ] Sign-in flow works in 3 clicks; user lands at `/dashboard` with a session.
- [ ] Server-side `auth()` returns userId/orgId; rejects `/dashboard` for signed-out user (middleware).
- [ ] Webhook fires on user.created → row appears in your DB within 5s.
- [ ] Webhook signature verification rejects tampered payloads.
- [ ] Creating an org via `<OrganizationSwitcher>` produces an `org` row in your DB.
- [ ] Inviting a user creates a pending invite; accepting it produces a `membership` row.
- [ ] `has({ permission: 'org:admin' })` returns true only for admins.
- [ ] `currentUser()` is called rarely; most checks use `auth()`.
- [ ] Production and staging use separate Clerk instances (different publishable + secret keys).
- [ ] Deleting a Clerk user webhook-cascades the deletion in your DB.
