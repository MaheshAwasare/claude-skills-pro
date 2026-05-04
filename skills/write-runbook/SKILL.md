---
name: write-runbook
description: Generate an operational runbook for a service from its code, config, and dependencies — what to do when it breaks, where to look, how to roll back, who to escalate to. Use proactively before a service goes to production, after a major incident, or when handing off operational ownership.
---

# Write a Runbook

A runbook answers the question: "It's 3am, this service is paging me, what do I do?" Without one, every incident takes 3x longer because the on-call engineer is figuring out *what the service even does*. This skill produces that document.

## When to use

- Before a service goes to production.
- After an incident — the post-mortem reveals gaps in the runbook.
- Handing off ownership of a service to another team.
- Onboarding a new on-call rotation.

## When NOT to use

- Stateless one-off tools nobody pages on — runbooks are for things with availability requirements.
- Internal scripts, dev tooling — overkill.

## The runbook structure

A runbook is *operational reference*, not architecture documentation. It optimizes for the 3am case.

```markdown
# <service-name> Runbook

## At a glance

- **Purpose**: one sentence — what this service does for customers.
- **Stack**: Go service / Postgres / Redis / Razorpay (or whatever).
- **Owners**: team-platform, slack:#team-platform, on-call: pagerduty:platform.
- **Repo**: github.com/acme/<service>
- **Dashboards**: <link> (Grafana / Datadog / Cloudwatch).
- **Logs**: <link> (Loki / Datadog / CloudWatch Logs Insights).
- **SLO**: 99.9% availability, p95 latency < 300ms.

## Architecture (one paragraph)

What does this service depend on? What depends on it? Draw a small diagram if needed.

Example: "<service> handles payment intents. It reads from `users` table, writes to `payments`, calls Razorpay's API, publishes events to the `payments` Kafka topic. Consumed by `billing-worker` and `analytics-pipeline`."

## Common alerts and what to do

### Alert: high error rate

**Signal**: `error_rate{service="payments"} > 1%` for 5 min.

**First check**:
1. Open <dashboard>. Are errors spiking on a specific endpoint?
2. Open <Sentry link>. New issue or existing?
3. Check <Razorpay status page>.

**Likely causes** (ranked):
1. Razorpay API is down (60% of past incidents) — check status page.
2. Bad deploy — see "Recent deploys" section. Roll back if needed.
3. DB connection pool exhausted — check `pg_stat_activity` count.

**If it's Razorpay**: nothing to do code-side; comms via #incidents.
**If it's a deploy**: rollback procedure below.
**If it's pool exhaustion**: scale replicas first; investigate the leak after.

### Alert: high latency

...

### Alert: disk usage > 80%

...

## Recent deploys

Link to the deploy dashboard / commit list. The last 24h of changes is what likely broke things.

## Rollback procedure

Specific commands. See `plan-the-rollback`.

```bash
# Roll back via re-deploy
helm rollback payments <previous-revision>

# Or trigger via CI
gh workflow run rollback.yml -f service=payments -f to=<sha>
```

## Common manual operations

### Drain a queue

```bash
kubectl exec -it deploy/payments -- /bin/sh
> ./bin/admin drain-queue --queue=refunds
```

### Re-process a stuck webhook

```bash
psql ...
> UPDATE webhook_events SET retry_count = 0, status = 'pending' WHERE id = '<id>';
```

### Force a migration

```bash
kubectl run --rm -it migrate --image=acme/payments:latest -- /app/migrate up
```

## Key metrics

| Metric | Healthy | Action threshold |
|---|---|---|
| Request rate | 10–100 req/s | None — informational |
| p95 latency | < 300ms | > 500ms paging |
| Error rate | < 0.1% | > 1% paging |
| DB pool used | < 50% | > 80% paging |

## Dependencies (and what to do when they fail)

| Dep | Failure mode | What to do |
|---|---|---|
| Postgres | Read/write fails | Failover (managed); no manual action. Check primary status. |
| Redis | Cache miss → DB load 5x | Service still works (degraded); scale up DB replicas. |
| Razorpay | API timeouts | Circuit breaker engages; queued retries. Comms only. |
| Brevo | Email send fails | Background queue; retries automatically. Alert if backlog > 1k. |

## Contacts and escalation

- **Primary on-call**: PagerDuty rotation `platform-primary`.
- **Secondary**: `platform-secondary`.
- **Engineering lead**: @lead-platform on Slack, +91-... in emergencies.
- **Vendor escalations**:
  - Razorpay: support@razorpay.com, account manager: <name> <email>.
  - Brevo: support ticket; account: <id>.

## Where the bodies are buried

Quirks that bite new on-call:
- "We restart `payments-worker` every Sunday because of a slow leak we haven't fixed (#1234)."
- "Don't run migrations during invoice generation (UTC 18:00–19:00) — locks `invoices` table."
- "The `legacy/` route is for v1 mobile clients still in the wild; don't delete."

## Post-incident

- File the post-mortem template at <link>.
- Update this runbook with anything missing.
```

## How to write each section well

### "At a glance"

Resist the urge to write for new hires here. The on-call engineer at 3am needs URLs and one-liners. Detailed architecture goes in a separate ARCHITECTURE.md.

### "Common alerts"

For each alert, walk the actual runbook *the first time the alert fires*. What did you actually do? Write that down. The runbook is a record of past incidents, not speculation about future ones.

| Don't write | Do write |
|---|---|
| "Investigate the issue and resolve" | "Open Grafana dashboard X; check error rate by endpoint; if spike on /charges, see [section]" |
| "Roll back if needed" | "`helm rollback payments <prev>` — takes ~2 min." |
| "Contact the team" | "@oncall in #payments-incidents; PagerDuty alert page." |

### "Where the bodies are buried"

This is the most underrated section. Every long-running service has quirks that aren't in the code or comments. Capture them. New on-call won't know about them otherwise.

## Keeping it alive

A runbook decays. Every incident must touch the runbook:

- Was the runbook *correct*? If not, fix it.
- Was the runbook *complete*? If not, add the missing alert / step / contact.
- Was a new failure mode found? Add it.

If your post-mortem doesn't have an "Update runbook" action item, the runbook is rotting.

## Anti-patterns

- **Architecture document masquerading as runbook** — operational steps go here, not system design.
- **"Investigate and fix"** as a step — useless. Specific actions only.
- **No links** — every URL belongs here. Don't make on-call hunt.
- **Out-of-date contacts** — old on-call rotation, departed engineer, dead Slack channel.
- **No ranked likely causes** — alphabetical or random ordering wastes triage time.
- **No "where the bodies are buried"** — all non-obvious knowledge is in someone's head.
- **One huge runbook** for many services — break per service.
- **Runbook in a wiki nobody finds** — link from PagerDuty alert payload, dashboard, README.
- **Generated from code without curation** — code has architecture, not what to do when it breaks.

## Verify it worked

- [ ] On-call engineer can resolve a known alert by following the runbook only — no Slack questions.
- [ ] Every paging alert has a section in "Common alerts."
- [ ] Rollback commands are literal — copy-pasteable.
- [ ] All listed URLs work.
- [ ] Dependencies and failure modes are documented.
- [ ] "Where the bodies are buried" section exists and is non-empty for any service > 6 months old.
- [ ] Post-mortems update the runbook; this is a documented step in the post-mortem template.
- [ ] Linked from PagerDuty alerts so it appears in the page.
- [ ] On-call rotation rotation walks through the runbook when joining (training step).
- [ ] Last reviewed date is within the last 6 months.
