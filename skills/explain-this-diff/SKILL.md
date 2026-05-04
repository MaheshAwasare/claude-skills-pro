---
name: explain-this-diff
description: Read a git diff and produce a plain-English narrative for a reviewer who wasn't in the original conversation — what changed, why, what's risky, what to spot-check. Use when the diff is non-trivial and the PR description is missing or auto-generated, or when handing context to a reviewer who's joining late.
---

# Explain This Diff

The reviewer doesn't know what you know. A diff without narrative forces them to reconstruct intent from code, which is slow and error-prone. This skill produces the narrative.

## When to use

- Opening a non-trivial PR — generates the description.
- Handing a stale PR to a fresh reviewer mid-week.
- Annotating a long-running feature branch for someone joining the project.
- Code archaeology — explaining a historical commit to current you.

## When NOT to use

- Trivial diffs (typo, lint fix) — title is enough.
- Generated code changes (lockfile, generated proto) — note them and skip the body.

## The narrative shape

For any diff, produce:

1. **One-sentence summary** — what changes from the user's or developer's perspective.
2. **Why** — motivation. Pull from issue, commit messages, conversation context.
3. **The interesting bits** — design choices a reviewer might question.
4. **The boring bits** — large file moves, generated code, mechanical refactor — call them out and tell the reviewer to skim.
5. **What to spot-check** — specific files or lines worth careful attention.
6. **What's risky** — failure modes worth thinking through.

Skip sections that don't apply. Better short and dense than long and fluffy.

## Reading a diff (the algorithm)

1. **List the file categories** — group changes by type:
   - Source files (where the actual change lives).
   - Test files (signal: the change is tested).
   - Generated files (skim only).
   - Config / lockfile (note version bumps).
   - Docs.
2. **Identify the "main" change** — the file with the most logic-changing lines is usually it.
3. **Trace its callers** — who else changed because of this? Those changes follow from the main change.
4. **Look for behavior changes** — `if/else` rearrangements, new branches, default value changes. Highlight.
5. **Look for state changes** — schema migrations, config flags, env var additions. These are riskiest.
6. **Look for what's missing** — no test? No error handling? No null check on user input?

## Worked example

**Diff:** 12 files changed, +340 lines / -85 lines.

```
M  apps/api/src/routes/refunds.ts          (+95 / -10)
M  apps/api/src/services/payments.ts       (+40 / -5)
M  packages/db/schema.ts                   (+8 / 0)
A  packages/db/migrations/0042_add_remaining_refundable.sql  (+12 / 0)
M  apps/api/src/services/refunds.test.ts   (+120 / -40)
A  apps/api/src/services/refunds.fixtures.ts (+50 / 0)
M  apps/web/src/app/billing/refunds/page.tsx (+15 / -10)
M  README.md                               (+5 / 0)
M  pnpm-lock.yaml                          (skip)
```

### Narrative produced

> **Summary:** Adds partial-refund support to `POST /api/refunds`. Previously the endpoint only did full refunds.
>
> **Why:** Customer support has been issuing full refunds for partial requests because the dashboard didn't expose partial. Tracked as #2156.
>
> **Main change:** `apps/api/src/routes/refunds.ts` — accepts optional `amount` field; when present and < payment amount, calls Razorpay with `speed: "optimum"` (UPI/IMPS instant). Logic delegated to `services/payments.ts` which validates the amount against `remaining_refundable`.
>
> **DB change:** `migrations/0042_add_remaining_refundable.sql` adds a generated column on `payments`:
> ```sql
> remaining_refundable bigint GENERATED ALWAYS AS (amount_paise - refunded_paise) STORED
> ```
> Generated column = no app code maintains it; Postgres keeps it in sync with `refunded_paise`.
>
> **Tests:** `services/refunds.test.ts` is mostly rewritten — old tests were happy-path only. New file `refunds.fixtures.ts` is the test data builder.
>
> **UI:** `apps/web/.../page.tsx` — small change to show partial-refund option when remaining > 0.
>
> **Skim:** `pnpm-lock.yaml` (no real dep changes), `README.md` (one-line note about new endpoint shape).
>
> **Spot-check:**
> - `services/payments.ts:54` — the validation `if (amount > remaining_refundable)` returns 422. Is the error message clear?
> - `services/payments.ts:78` — `speed: "optimum"` falls back to `"normal"` per Razorpay docs; verify the path you'd want for >₹50k refunds (Optimum doesn't always cover those).
> - `migrations/0042` — generated column is `STORED`, so it occupies disk; `VIRTUAL` would save space but Postgres only supports STORED.
>
> **Risks:**
> - If `refunded_paise` is ever updated outside the refund flow (e.g. manual SQL fix), `remaining_refundable` updates correctly (Postgres handles it).
> - Race condition: two simultaneous partial refund requests could both pass validation and over-refund. Mitigation in this PR: row-lock on `payments` row in the service. Verify the lock is actually held (search for `SELECT ... FOR UPDATE`).

A reviewer reading this can review the PR in 10 minutes instead of 30.

## What "risk" actually means

Not "could this break?" — every change could break. Specifically:

- **Concurrency hazards** — what if two requests hit this at the same time?
- **Migration / rollback** — see `plan-the-rollback`.
- **Data loss** — DELETE / UPDATE without WHERE; truncating tables; column drops.
- **Performance regression** — N+1 introduced; new query without an index; cache invalidated too aggressively.
- **Security** — input not validated; auth check missing; secret in code.
- **Backward compatibility** — API contract changes that older clients can't handle.

Call out the specific risk, not "this is risky."

## Anti-patterns

- **Re-listing every file changed** — diff already shows that. Group and summarize.
- **Restating diff lines as English** — "now the function returns true if X." Useless. The diff says it.
- **No "spot-check" section** — reviewer has no anchor for where to focus.
- **Hedging language** ("This might be risky") — be concrete: "Race condition possible because lines 12–18 are not atomic; mitigation needed."
- **Skipping the "why"** — reviewer can't judge if the approach is right without knowing the goal.
- **Including history of what you tried before** unless directly informative — past attempts belong in commit messages, not PR explanations.
- **No explicit risk callout** — reviewer should not have to *find* the risks.
- **Auto-generated bullet list** ("Updated foo.ts. Updated bar.ts. ...") — adds zero info.

## Verify it worked

- [ ] One-sentence summary that a non-author understands.
- [ ] Why is concrete (issue link, metric, user report — not "improvement").
- [ ] Main change identified by file/section, not "the changes."
- [ ] Boring sections (lockfile, generated) are flagged as skim-only.
- [ ] Spot-check section has specific file:line references.
- [ ] Risks are concrete, not handwaved.
- [ ] A reviewer joining cold can review in < 30 min using only the narrative + diff.
- [ ] Generated code / mechanical refactor is *not* explained line by line — flagged for skim.
