---
name: plan-the-rollback
description: Before any risky change — schema migration, deploy, infra cutover, vendor swap — write the rollback plan first. Use proactively before "let's just push it" moments. Refuses to proceed without a documented undo path or a deliberate, justified accept-the-risk decision.
---

# Plan the Rollback

The riskiest moment of any change is "we'll figure it out if it breaks." That's not a plan. This skill enforces writing the rollback *before* doing the thing.

## When to use

- About to apply a DB migration (especially destructive: drop, alter type, rename, NOT NULL).
- Cutting over to a new vendor (auth, payments, email, hosting).
- Changing a config that affects all users (feature flag default, rate limit, cache TTL).
- Renaming an API field consumed by clients.
- Deploying a new service that takes traffic for the first time.
- Any change where "if this breaks at 2am, what do we do?" doesn't have a one-sentence answer.

## When NOT to use

- Pure code change with green tests, no schema or external service impact.
- Reverting a recent change (rollback *is* the plan).

## The four-question rollback plan

Before the change, write down answers to all four:

1. **What signal tells us to roll back?** Concrete metric or alert. "User complaints" is not a signal — too late.
2. **What is the rollback action, in commands?** The literal commands or PR to revert. Not "we'd revert" — the diff.
3. **What is the rollback's blast radius?** What state does the rollback restore? What does it not?
4. **How long does rollback take?** Seconds (config flag), minutes (deploy revert), or hours (data migration reversal)?

If any answer is "we'd figure it out," stop and figure it out *now*.

## Worked example: schema migration

> **Change:** Add `NOT NULL` constraint to `users.phone`.

| Q | Answer |
|---|---|
| Signal | Errors on `INSERT INTO users` from sign-up flow; alert on `users_signup_failed_total` increment. |
| Rollback action | `ALTER TABLE users ALTER COLUMN phone DROP NOT NULL;` Drop happens in same transaction as deploy revert. |
| Blast radius | Rows inserted between bad deploy and rollback have `phone` populated (good); no other state change. |
| Time | ~5s for ALTER on a 50M-row table (constraint drop is metadata-only). |

Now we can ship safely. **The rollback plan exposed:** if the constraint added required a backfill that we forgot, sign-up would 100% break — the rollback action above wouldn't have caught that. Updated plan: **add constraint as `NOT VALID` first, backfill in batches, then `VALIDATE CONSTRAINT`** — three reversible steps.

## Worked example: vendor cutover

> **Change:** Migrate from SendGrid to Brevo.

| Q | Answer |
|---|---|
| Signal | `email_send_failure_rate` > 1%, OR open rate drops > 30% over 6h, OR DKIM/SPF errors in logs. |
| Rollback action | Toggle `EMAIL_PROVIDER=sendgrid` env var; restart pods. Code keeps both providers behind an interface for 30 days. |
| Blast radius | None — both providers can send to the same address; suppression list is a shared DB table, not provider-managed. |
| Time | ~2 min (env var change + rolling restart). |

**The rollback plan exposed:** if we cut over without keeping the SendGrid client code, rollback is a deploy *and* a code revert. Solution: ship the dual-provider code in PR #1, do the cutover in PR #2 (env var only). Each step is independently reversible.

## Categories of risky change and their rollback patterns

### Schema migrations

| Operation | Reversible? | Rollback pattern |
|---|---|---|
| `CREATE TABLE` | Easy | `DROP TABLE` (if no data) |
| `ADD COLUMN NULLABLE` | Easy | `DROP COLUMN` (data loss accepted) |
| `ADD COLUMN NOT NULL` | Hard | Add as nullable + backfill + `SET NOT NULL` separately |
| `RENAME COLUMN` | Hard | Add new + dual-write + backfill + drop old |
| `ALTER COLUMN TYPE` | Hard | Add new column with new type + backfill + swap |
| `DROP COLUMN` | Loss | Take a backup; consider if "drop" is really needed |
| `DROP TABLE` | Loss | Backup; rename to `_archived` instead |

The rule: **multi-step migrations beat one-step migrations**. Each step reversible, the composite reversible. Pre-deploy migration → deploy code → post-deploy migration is the classic pattern.

### Feature flags

The cheapest rollback. Wrap the change in a flag; "rollback" is flipping the flag. But:
- The fallback path must be tested. Flag flips that crash because the off-path is broken aren't a rollback.
- Flag flips have their own risk window — consider what state was written while the flag was on.

### API field changes

| Change | Rollback |
|---|---|
| Add field | Easy — clients ignore. |
| Remove field | Hard — clients break. Deprecate first, remove after migration window. |
| Rename field | Hard — emit both for a window, then remove old. |
| Change type | Hard — same as rename in practice. |

### Deploys

Rollback = redeploy previous version. Make sure:
- Previous artifact is still available (not GC'd).
- DB schema is compatible with the previous code (forward-compatible migrations).
- Config is compatible (no required env vars added).

### Infra cutover (DNS, load balancer, CDN)

DNS rollback is slow (TTL). For low-TTL changes, plan the cutover to start when traffic is low; have the reverse change pre-staged.

## The forward-compatible deploy

For zero-downtime, schema and code changes follow this pattern:

1. **Migration A**: schema change that's backward-compatible (old code still works).
2. **Deploy**: new code that works with both old and new schema.
3. **Migration B**: schema change that's forward-only (cleanup).

If anything goes wrong between 1 and 2, rollback = revert deploy (no migration to undo). Between 2 and 3, rollback = revert deploy again. Between 3 and "stable," rollback is harder — but you've validated the full system.

## Anti-patterns

- **"It worked in staging"** — staging has 1% of prod data and 0% of prod traffic. Plan the rollback anyway.
- **Single-PR migration that combines schema + code change** — you can't roll back one without the other; window of incompatibility.
- **Rollback plan that says "we'd revert and figure it out"** — not a plan.
- **Rollback that requires the same on-call who shipped the change** — what if they're asleep / on holiday? Documented commands.
- **No alarm wired to the signal** — manual monitoring fails at 3am. Alert + paging.
- **`DROP COLUMN` in the same migration as deploying code that doesn't read it** — if the deploy is reverted, the old code can't read the dropped column. Two-phase: deploy first, drop later.
- **Rollback "tested" only on the happy path** — try it on the unhappy path: rollback after a row was inserted via the new code path. Does data make sense?
- **Using `DROP IF EXISTS ... CASCADE`** in rollback scripts — silently nukes related objects. Be explicit.
- **No backup before destructive change** — for "this should be reversible" operations, take the snapshot anyway. Disk is cheap.
- **Rollback documentation buried in a runbook nobody finds at 3am** — paste the commands into the deploy PR description.

## Verify it worked

- [ ] Rollback plan is in writing in the PR description before the PR is merged.
- [ ] All four questions are answered specifically (signal, action, blast radius, time).
- [ ] Rollback action is the literal commands or PR — not prose.
- [ ] An on-call who didn't write the change can execute the rollback.
- [ ] For schema changes: dry-run the rollback in a staging copy of prod data.
- [ ] For destructive operations: backup taken; confirmation of restorability.
- [ ] Alert wired to the rollback signal.
- [ ] If rollback requires multiple steps, they're ordered and each is reversible.
- [ ] The plan is reviewed by someone other than the author.
- [ ] After the change, the rollback plan is preserved (in the PR / runbook) for future audit.
