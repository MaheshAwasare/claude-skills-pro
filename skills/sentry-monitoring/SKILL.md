---
name: sentry-monitoring
description: Set up Sentry error and performance monitoring — SDK config, sourcemaps, release tracking, sampling, alerts, the noise-reduction patterns that prevent Sentry fatigue, and PII redaction for compliance. Use when adding error monitoring to a new app or cleaning up a noisy Sentry account where every push triggers 200 alerts.
---

# Sentry Monitoring

The default Sentry setup catches exceptions but ships at 100% sample rate, leaks PII, doesn't connect releases, and floods you with noise. This skill is the cleaned-up version.

## When to use

- New app — wire Sentry from day 1.
- Existing Sentry account that has become useless from noise.
- Adding performance monitoring on top of error monitoring.
- Setting up release tracking so you know which deploy broke things.

## When NOT to use

- High-throughput data plane (>100k errors/min) — Sentry's pricing curves up; consider Honeycomb or self-hosted.
- You're already deep on Datadog APM with error tracking — less value-add.

## Setup (Next.js)

```bash
pnpm add @sentry/nextjs
npx @sentry/wizard@latest -i nextjs
```

The wizard creates `sentry.client.config.ts`, `sentry.server.config.ts`, `sentry.edge.config.ts`. Edit each:

```ts
// sentry.server.config.ts
import * as Sentry from "@sentry/nextjs";

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  environment: process.env.NODE_ENV,
  release: process.env.SENTRY_RELEASE,           // git SHA from CI
  tracesSampleRate: process.env.NODE_ENV === "production" ? 0.1 : 1.0,
  profilesSampleRate: 1.0,                       // of sampled traces
  beforeSend(event) {
    // Drop noise
    if (event.exception?.values?.[0]?.type === "ResizeObserver loop limit exceeded") return null;
    return event;
  },
  beforeSendTransaction(event) {
    // Drop healthcheck noise
    if (event.transaction === "GET /healthz") return null;
    if (event.transaction === "GET /readyz")  return null;
    return event;
  },
  integrations: [
    Sentry.httpIntegration(),
  ],
});
```

```ts
// sentry.client.config.ts
import * as Sentry from "@sentry/nextjs";

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  environment: process.env.NEXT_PUBLIC_ENV,
  tracesSampleRate: 0.1,                          // 10% of transactions
  replaysSessionSampleRate: 0.0,                  // off by default — bandwidth + privacy
  replaysOnErrorSampleRate: 1.0,                  // record only failed sessions
  integrations: [
    Sentry.replayIntegration({
      maskAllText: true,                          // PII protection
      blockAllMedia: true,
    }),
  ],
});
```

## Sourcemaps (the bit that makes stack traces readable)

Without sourcemaps, prod stack traces look like `e@/_next/static/chunks/abc.js:1:99999` — useless. Wire them in CI:

```yaml
# .github/workflows/ci.yml — deploy step
- name: Upload sourcemaps to Sentry
  run: npx sentry-cli releases new $SENTRY_RELEASE
  env: { SENTRY_AUTH_TOKEN: ${{ secrets.SENTRY_AUTH_TOKEN }}, SENTRY_ORG: acme, SENTRY_PROJECT: web }

- run: npx sentry-cli releases files $SENTRY_RELEASE upload-sourcemaps .next/static
- run: npx sentry-cli releases finalize $SENTRY_RELEASE
```

`@sentry/nextjs` does this automatically if you set `SENTRY_AUTH_TOKEN` in env. Verify in Sentry → Releases → your release → Source Maps tab.

## Release tracking (deploy-to-error correlation)

Set `release` to your git SHA on every deploy:
```ts
release: process.env.NEXT_PUBLIC_VERCEL_GIT_COMMIT_SHA || process.env.GIT_SHA
```

Sentry's "Suspect Commits" then correlates errors to specific commits — "this error appeared with commit abc123, which was your refactor of the payment flow." One of Sentry's best features; nobody enables it.

## User context (without PII leaks)

```ts
// On login
Sentry.setUser({ id: userId });            // ID only, NOT email/name

// On logout
Sentry.setUser(null);
```

Resist the urge to set `email` or `name` — DPDP/GDPR audit will flag this. ID is enough; you can join in your tools.

## Custom contexts and breadcrumbs

```ts
Sentry.setTag("plan", user.plan);                          // searchable in UI
Sentry.setContext("subscription", { id, status, plan });   // appears alongside error
Sentry.addBreadcrumb({ category: "checkout", message: "started checkout", level: "info" });
```

Tags are bounded enums (`plan`, `region`, `feature_flag_x`). Don't tag with user IDs — high cardinality, no value.

## Catching what `console.error` doesn't

```ts
// Wrap key business operations
try {
  await chargeCustomer(id);
} catch (err) {
  Sentry.captureException(err, {
    tags: { operation: "charge_customer" },
    extra: { customerId: id },
  });
  throw err;                              // re-throw so caller still handles
}
```

For unhandled rejections / uncaught exceptions, Sentry's SDK auto-installs handlers. But ad-hoc `try/catch` blocks that swallow errors silently won't trigger Sentry — wrap with `captureException`.

## Performance monitoring (transactions)

Auto-instrumentation traces:
- HTTP routes
- Outbound HTTP (fetch, axios)
- Postgres / MySQL / Redis (with `dbIntegration`)
- Next.js page loads + API routes

Custom spans for business operations:
```ts
import * as Sentry from "@sentry/nextjs";

await Sentry.startSpan({ name: "charge_customer", op: "business" }, async (span) => {
  span.setAttribute("customer.id", customerId);
  return chargeCustomer(customerId);
});
```

## Alert rules (the part nobody configures)

In Sentry → Alerts → Create:
1. **New Issue**: alert when a brand-new error type appears in production. Slack/email.
2. **Issue regression**: alert when a resolved issue starts firing again.
3. **Volume spike**: alert when an existing issue's rate jumps >5x. Slack only.
4. **Performance regression**: alert when a transaction's p95 increases >50% over 24h.
5. **Crash-free sessions** (mobile/replay): alert when below 99.5%.

Skip the "any error in prod" alert — that's how Sentry fatigue starts.

## Anti-patterns

- **`tracesSampleRate: 1.0` in production** — fills your event quota in a week. 5–10% is plenty for most services.
- **No sourcemaps in prod** — stack traces are minified noise. Sourcemap upload should be a CI step.
- **No `release` set** — can't correlate errors to deploys; "suspect commits" feature does nothing.
- **Setting user.email or user.name** — PII leak; DPDP/GDPR audit finding. Use `id` only.
- **No `beforeSend` filter** — `ResizeObserver loop limit`, `Non-Error promise rejection`, browser extension errors, healthcheck transactions — pure noise.
- **One Sentry project for everything** — alerts blur, dashboards get crowded. Project per service.
- **Replay session sample rate > 0** — bandwidth + privacy concerns. `replaysOnErrorSampleRate: 1.0` is the right move.
- **Ignoring "issue regression"** — fixed bugs that come back are the most valuable signals. Configure the alert.
- **Letting `user_id` flow as a tag** — high cardinality, breaks search. Set as `user.id` (Sentry handles separately).
- **Instrumenting in dev with prod DSN** — pollutes prod data. Use `enabled: process.env.NODE_ENV !== 'development'` or env-specific DSNs.
- **Catching exceptions and swallowing them silently** — Sentry only sees what's thrown or `captureException`'d.

## Verify it worked

- [ ] An intentional error in prod produces a Sentry issue within 30s with a readable (de-minified) stack trace.
- [ ] Issue links to the suspect commit in Sentry UI.
- [ ] User context shows `id` only, no `email`/`name` (privacy audit).
- [ ] `tracesSampleRate` ≤ 0.1 in prod; quota usage stable month-over-month.
- [ ] `beforeSend` drops at least: `ResizeObserver` errors, `/healthz` and `/readyz` transactions, browser extension errors.
- [ ] Replay only captures sessions that error.
- [ ] Sourcemaps for the latest release visible in Sentry → Releases → Source Maps.
- [ ] At least 4 alert rules configured: new issue, regression, volume spike, perf regression.
- [ ] Production deploy creates a new Sentry release tagged with git SHA.
- [ ] Local `pnpm dev` doesn't send events to prod Sentry project.
