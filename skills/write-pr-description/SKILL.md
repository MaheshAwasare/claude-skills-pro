---
name: write-pr-description
description: Write a PR description that gets reviewed quickly — the change in one sentence, the why, the test plan, the rollback. Use when opening any non-trivial PR, or when an existing PR description is "see commits" / auto-generated bullet list.
---

# Write a PR Description

A good PR description gets reviewed in 10 minutes. A bad one (or none) gets stalled, rubber-stamped, or starts a thread of clarifying questions. This skill is the template.

## When to use

- Opening a non-trivial PR (>50 lines, new behavior, anything risky).
- Reviewing a PR with a "see commits" or auto-generated description and asking the author to expand.
- Writing the PR description *before* opening, so the description shapes the implementation.

## When NOT to use

- Trivial PRs (typo, dependabot bump, one-line config tweak) — title is enough.

## The template

```markdown
## What

One sentence: what changes, observable to a user or another developer.

## Why

The motivation. What problem does this solve? Link to issue / ticket / Slack thread / metric.

## How

The interesting design choices. Skip the obvious. Mention what you considered and rejected.

## Test plan

- [ ] Step a reviewer or CI can verify
- [ ] Step b
- [ ] Step c

## Rollback plan

How to revert if this misbehaves in prod. Commands or PR. Time to revert.

## Screenshots / video

(For UI changes only.)

## Refs

- Closes #123
- Related to #456
```

Use only the sections that apply. A small backend PR might be: What, Why, Test plan. A UI PR adds Screenshots. A risky migration adds Rollback plan. Don't pad sections that don't help review.

## Each section, tightly

### What

One sentence. Describes change in user-facing or developer-facing terms.

| Bad | Good |
|---|---|
| "Fixes auth" | "OAuth sign-in callback now decodes `state` param before comparing." |
| "Adds endpoint" | "Adds `POST /api/refunds` with partial-amount support via Razorpay Optimum." |
| "Updates package" | "Bumps `@clerk/nextjs` 5.4 → 5.7 to pick up org-invitation webhook fix." |

### Why

The trigger for the change. Examples:
- "Reported by user #4521 — see Sentry issue 88a4."
- "From design doc: docs/design/team-invites.md (slack: #invites-design)."
- "Compliance audit (ticket SEC-89) flagged HMAC SHA-1 usage."
- "Latency p95 on /search jumped 400ms after #2156; this rolls back the embedding lookup change."

If the why isn't obvious from a linked issue, it has to be in the description. Reviewers shouldn't have to ask "why are we doing this?"

### How

Only the *interesting* parts of the implementation. Skip:
- "Added a new file `services/refunds.ts`" — visible in the diff.
- "Wrote tests" — visible in the diff.

Include:
- Design choices a reviewer might question. ("Used a Postgres advisory lock instead of Redis because…")
- Performance trade-offs. ("Computed at write time rather than read time because read traffic is 100x.")
- Security choices. ("Idempotency key required to prevent double-charge on retry.")
- What you tried and rejected. ("Initially used `pg_notify`; switched to Postgres `LISTEN/NOTIFY`-via-pg-boss because we already depend on it.")

### Test plan

A checklist a reviewer or CI can mechanically verify. Examples:

- [ ] `pnpm test` passes (CI).
- [ ] Manual: `curl -X POST .../api/refunds -d '{"amount": 5000}'` returns 201 with refund row.
- [ ] Manual: webhook signature failure → 400, not 500.
- [ ] Browser: invitation accept page renders with org name.
- [ ] Load test: 100 concurrent `POST /charges` doesn't exceed pool of 20 connections.

Don't write a test plan that says "tested locally." Specific actions a reviewer can repeat.

### Rollback plan

For risky PRs only — see `plan-the-rollback` skill. Specific commands or revert PR.

```
**Rollback:** Revert this PR (`git revert <merge-sha>`); no schema changes
to undo. Time to revert: ~5 min (deploy time).
```

For DB-touching PRs:
```
**Rollback:**
1. Revert PR.
2. `ALTER TABLE refunds DROP COLUMN partial_amount_paise;` (column nullable; safe to drop).
3. Total time: ~10 min including deploy.
```

### Screenshots / video

For UI changes — before/after side-by-side. Use Loom for interactions / animations. For multi-state UI, screenshot each state.

### Refs

Issues, related PRs, design docs, Slack threads. Use `Closes #123` to auto-close issues on merge.

## Worked example

```markdown
## What

Adds partial refund support to `POST /api/refunds`. Default behavior
unchanged (full refund); pass `amount` (in paise) for partial.

## Why

Customer support has been issuing full refunds because the dashboard
didn't expose partial. Refund $-volume is up 8x from change requests
since launch — see #2156. Partial refund support reduces $-loss
without changing the support agent workflow.

## How

- Added `amount` (optional, integer paise) to `POST /api/refunds` body.
- When `amount < payment.amount_paise`, calls Razorpay with
  `speed: "optimum"` (UPI/IMPS instant where supported, fallback to
  normal).
- Validates `amount > 0` and `amount <= remaining_refundable`.
- `remaining_refundable = payment.amount - sum(prior refunds)`. Stored
  as a generated column on `payments` for fast access.

Considered: refund-by-line-item mapped to invoice lines. Rejected for
this PR — needs new invoice/line-item models. Tracked as #2233.

## Test plan

- [ ] CI green (unit + integration).
- [ ] Manual: full refund still works (`amount` omitted).
- [ ] Manual: partial refund creates row in `refunds` with correct amount.
- [ ] Manual: over-refund (>remaining) returns 422 with clear message.
- [ ] Manual: webhook `refund.processed` updates `payments.refunded_paise`.

## Rollback plan

Revert this PR; the new `remaining_refundable` generated column is safe
to keep (unreferenced). To fully clean up after revert:
`ALTER TABLE payments DROP COLUMN remaining_refundable;`. Total time
~10 min.

## Refs

Closes #2156
Related #2233
```

A reviewer reading this can: understand the change, agree it's the right scope, know what to spot-check, know what happens if it breaks.

## Anti-patterns

- **"See commits"** — you're outsourcing the review prep to the reviewer.
- **Auto-generated bullet list of file changes** — visible in the diff; adds no info.
- **No why** — reviewer can't judge whether the change is the right approach.
- **No test plan** — reviewer either takes your word or has to figure out the manual tests.
- **Marketing copy** ("This exciting new feature…") — reviewers want facts, not pitch.
- **Two unrelated changes in one PR** — see `shrink-this-pr`. The description can't cleanly cover them.
- **"Tested locally"** as the entire test plan — what did you test? How does the reviewer repeat it?
- **No rollback plan on a migration PR** — see `plan-the-rollback`.
- **Screenshots without captions** — reviewer can't tell which is "before" vs "after."
- **Linking to a Slack thread without summary** — Slack messages disappear into history.

## Verify it worked

- [ ] What is one sentence and concretely describes the change.
- [ ] Why links to a real motivation (issue, ticket, metric, complaint).
- [ ] How surfaces the *interesting* design choices, not the obvious.
- [ ] Test plan is a checklist a reviewer can run, not "tested locally."
- [ ] Rollback plan exists for any DB / config / external-dep change.
- [ ] UI changes have screenshots or Loom links.
- [ ] PR title matches Conventional Commit format (per `write-commit-message`).
- [ ] Reviewer can read the description in < 2 min and have the context to start the review.
- [ ] No "see commits" or "various improvements" anywhere.
