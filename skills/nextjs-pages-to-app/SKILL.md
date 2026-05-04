---
name: nextjs-pages-to-app
description: Migrate a Next.js Pages-router app to App-router incrementally — coexistence strategy, data-fetching translation (getServerSideProps → Server Components, getStaticProps → static), layouts, auth middleware, API routes → route handlers. Use when migrating an existing Pages-router app, not when starting greenfield.
---

# Migrate Next.js: Pages → App Router

The Pages → App migration looks daunting but Next.js supports both routers in the same project. Migrate incrementally, route by route. This skill is the order and the gotchas.

## When to use

- Existing Pages-router app on Next.js 13+ (App router available since 13.4 stable).
- Want Server Components (RSC), nested layouts, streaming, parallel routes.
- Refactoring opportunity coincides with router migration.

## When NOT to use

- Greenfield app — start with App router directly.
- App on Next < 13 — upgrade Next first.
- Heavy `getStaticPaths`+ISR coverage where Pages router is already optimal — App router's equivalent is fine but the migration win is small.

## The principle: coexist, don't cut over

Both routers can run in the same app. `pages/` and `app/` directories are read together. Migrate route-by-route. **Never** do a big-bang cutover for a non-trivial app.

```
my-app/
  pages/
    api/...                # legacy API routes (still work)
    blog/[slug].tsx        # not yet migrated
    about.tsx              # not yet migrated
  app/
    page.tsx               # migrated home
    layout.tsx
    login/page.tsx         # migrated
    api/v2/users/route.ts  # new route handler
```

If a route exists in both, **app/ wins** in Next 13.4+. Use this to test side-by-side: keep the Pages version live, build the App version at a different path, switch when ready, delete the Pages version.

## Migration order (recommended)

1. **Add `app/layout.tsx` and `app/page.tsx`** — root layout + home. Smallest change to get App router live.
2. **Migrate static pages** (`/about`, `/contact`) — no data fetching, simplest first.
3. **Migrate API routes** to route handlers (`pages/api/*` → `app/api/*/route.ts`) one at a time.
4. **Migrate dynamic Pages with `getServerSideProps`** to Server Components with `async` data fetching.
5. **Migrate Pages with `getStaticProps` / `getStaticPaths`** to App router static rendering with `generateStaticParams`.
6. **Migrate auth / middleware** — `middleware.ts` works for both, but App router enables route-segment-level checks.
7. **Delete Pages router files** when all routes are migrated.

## Data fetching translation

| Pages | App |
|---|---|
| `getServerSideProps` | `async function Page()` Server Component with `await fetch()` (default) |
| `getStaticProps` | `async function Page()` with `fetch(url, { cache: 'force-cache' })` (default) |
| `getStaticPaths` | `export async function generateStaticParams()` |
| `revalidate: 60` | `fetch(url, { next: { revalidate: 60 } })` |
| `next/router` `useRouter` | `useRouter` from `next/navigation` (different API!) |
| `next/head` `<Head>` | `metadata` export or `generateMetadata()` |

### Concrete example: SSR page

```tsx
// pages/users/[id].tsx (Pages router)
export async function getServerSideProps({ params }) {
  const user = await fetch(`https://api.example.com/users/${params.id}`).then(r => r.json());
  return { props: { user } };
}

export default function UserPage({ user }) {
  return <h1>{user.name}</h1>;
}
```

```tsx
// app/users/[id]/page.tsx (App router)
export default async function UserPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;                               // params is a Promise in 15+
  const user = await fetch(`https://api.example.com/users/${id}`).then(r => r.json());
  return <h1>{user.name}</h1>;
}
```

Notes:
- Server Component by default — runs on server, ships zero JS for this component.
- `params` became `Promise` in Next 15 (await it).
- No `getServerSideProps` boilerplate.

### Static page

```tsx
// app/blog/[slug]/page.tsx
export async function generateStaticParams() {
  const posts = await fetch("https://cms.example.com/posts").then(r => r.json());
  return posts.map((p: any) => ({ slug: p.slug }));
}

export default async function PostPage({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params;
  const post = await fetch(`https://cms.example.com/posts/${slug}`).then(r => r.json());
  return <article>{post.body}</article>;
}
```

## API route → route handler

```ts
// pages/api/users.ts
export default async function handler(req, res) {
  if (req.method === "GET") {
    const users = await db.user.findMany();
    return res.json(users);
  }
  res.status(405).end();
}
```

```ts
// app/api/users/route.ts
import { NextResponse } from "next/server";

export async function GET() {
  const users = await db.user.findMany();
  return NextResponse.json(users);
}

export async function POST(req: Request) {
  const body = await req.json();
  // ...
}
```

`route.ts` exports per-method functions (GET, POST, PUT, DELETE, etc). Cleaner than the if-method dance.

## Layouts (the killer feature of App router)

In Pages router, layout sharing was hacky (`getLayout` on the page component). App router has nested layouts:

```tsx
// app/dashboard/layout.tsx
export default function DashboardLayout({ children }: { children: React.ReactNode }) {
  return (
    <div className="dashboard">
      <Sidebar />
      <main>{children}</main>
    </div>
  );
}
```

All routes under `/dashboard/*` automatically use this layout. The layout *doesn't* re-render on navigation between sub-pages — state survives.

## Client Components: the `'use client'` boundary

Server Components are the default. The moment you need `useState`, `useEffect`, event handlers, or browser APIs:

```tsx
// app/components/Counter.tsx
'use client';

import { useState } from "react";

export function Counter() {
  const [n, setN] = useState(0);
  return <button onClick={() => setN(n + 1)}>{n}</button>;
}
```

Server Components can import Client Components freely (passes them as serialized boundaries). Client Components can't import Server Components directly — pass them via `children`.

## Common gotchas

- **`next/router` → `next/navigation`** — different API. `useRouter` from `navigation` doesn't have `.query`; use `useSearchParams` and `useParams` separately.
- **`<Head>` → `metadata`** — static metadata via `export const metadata`; dynamic via `generateMetadata()`.
- **`getInitialProps`** — only used in custom `_app` / `_document`; no equivalent in App router. Migrate by moving logic to layouts.
- **CSS Modules + global CSS** — work the same; just import in `layout.tsx` instead of `_app.tsx`.
- **`pages/_app.tsx`** — replace with `app/layout.tsx` (root layout). Move providers there.
- **`pages/_document.tsx`** — replace with the `<html>` and `<body>` tags in `app/layout.tsx`.
- **Auth via NextAuth.js v4** — works with both, but v5 (Auth.js) is preferred for App router.
- **`getServerSideProps` returning `notFound: true`** — replace with `notFound()` from `next/navigation` (throws specific error caught by `not-found.tsx`).
- **Streaming + Suspense** — App router supports `<Suspense>` natively. Old Pages router relied on `dynamic(... { ssr: false })` workarounds.

## Performance differences

- **First Load JS** drops significantly when pages are Server Components.
- **TTFB** can be slower for SSR pages that are now streamed (intentional — first byte arrives sooner; later bytes stream).
- **Static pages** behave the same.
- **Data fetching is automatically deduped** in Server Components — calling `fetch(url)` twice in a tree fetches once.

## Anti-patterns

- **Big-bang migration** — too risky for any app with users. Coexist.
- **Mixing client and server logic in one component** — split. Server Component does data fetching; Client Component handles interactivity.
- **`'use client'` at the top of the entire app** — defeats RSC. Push the boundary as deep as possible.
- **`getServerSideProps` left in `pages/`** during migration — fine, but plan its removal. Don't add new ones.
- **Forgetting to `await params`** in Next 15+ — runtime warning becomes an error.
- **Importing client-only libraries in Server Components** — error at build time. Wrap in Client Component.
- **Auth check in `middleware.ts` only** — supplement with route-segment checks in App router for richer DX.
- **Moving everything to App router before testing** — migrate one route, deploy, observe, then continue.
- **Not testing the exact metadata output** — App router's `metadata` shape is different; SEO regressions silent.
- **Using `useEffect` for data fetching in App router** — that's a Pages-router pattern. Use Server Components.

## Verify it worked

- [ ] Both `pages/` and `app/` coexist; routes from both render correctly.
- [ ] At least one route is migrated end-to-end (URL works, data fetched, rendering correct).
- [ ] First Load JS for a migrated route is smaller than the Pages-router version.
- [ ] Metadata (title, OG image) on migrated routes matches Pages router output.
- [ ] Auth still works on migrated routes (middleware applies; redirects correctly).
- [ ] No `getServerSideProps` / `getStaticProps` left in `app/`.
- [ ] No `'use client'` files importing from `next/headers` or `next/cookies` (server-only).
- [ ] Streaming works on migrated SSR routes (visible via DevTools network tab — chunked response).
- [ ] Tests pass; visual regression tests pass.
- [ ] Production deploy: error rate flat or down; latency p95 stable or better.
