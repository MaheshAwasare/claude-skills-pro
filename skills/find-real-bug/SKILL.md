---
name: find-real-bug
description: Trace a bug to its root cause instead of patching the symptom. Refuses shortcuts like "wrap in try/catch", "add a null check", or "just retry it" until the actual cause is understood. Use when a test is failing intermittently, a metric just spiked, an error appears in logs, or a user reports something "weird."
---

# Find the Real Bug

The most common failure mode in debugging is treating the *symptom* as the bug. Wrapping a NullPointerException in `try/catch` makes it go away — and ships the bug. This skill enforces the discipline of finding the actual cause.

## When to use

- Failing test, especially intermittent.
- Production error spike.
- User report of "weird" behavior with no obvious trigger.
- Metric regression (latency, error rate, cost) of unknown origin.
- Code review where you're tempted to "just add a null check."

## When NOT to use

- You already know the cause and the fix is mechanical (e.g. typo, missed import).
- Time-critical hotfix where root cause analysis happens *after* the bleed stops — but commit to the post-mortem.

## The five questions (answer in order)

1. **What is the symptom, exactly?** Not "login is broken." But: "POST /api/login returns 500 with body `{...}` when email contains `+`."
2. **What changed?** A deploy, an upstream config, a dependency upgrade, a data migration, traffic pattern? Use `git log`, deploy log, dashboards.
3. **What is the failing line of code?** Not the route, not the controller — the literal line that raised. Read it.
4. **Why does that line fail?** State it as a sentence: "It fails because `user.profile` is null when the user signed up via OAuth and skipped onboarding."
5. **Why is the state that way?** Iterate up the cause chain until you reach a *design* issue, not a *programming* issue.

The fix lives at the highest level you can change — not the level where the symptom appears.

## The "five whys" example

> **Symptom:** Subscription renewal fails for 3% of users.
> **Why?** The renewal job throws "amount must be positive."
> **Why?** Some users have `subscriptions.amount_paise = 0`.
> **Why?** They were grandfathered onto a free trial that wasn't migrated when we removed the free plan.
> **Why?** The plan removal migration didn't touch existing subscriptions.
> **Why?** The migration was reviewed as "remove plan, no impact" because the staging DB had no users on free plans.
>
> **Real bug:** Migration didn't backfill grandfathered users. Fix at *that* level: write a one-off migration to assign these users to the new "free tier" plan with `amount_paise = 0` *plus* update the renewal job to skip zero-amount subs explicitly. Don't hide the symptom by `if (amount <= 0) skip;` without understanding why.

## The shortcuts to refuse

| Shortcut | Why it's wrong |
|---|---|
| Wrap in `try/catch` and log | Doesn't fix the bug, hides it from monitoring |
| Add a `null` check / optional chain | The null came from somewhere; that's the bug |
| Add a retry loop | Retries multiply load; if the underlying issue is correctness, retries don't help |
| Bump the timeout | Hides slow code instead of fixing it |
| `if (process.env.NODE_ENV !== 'production')` skip | Production bug appears in production; staging mask is dangerous |
| Pin to old dependency | Defers the issue and accumulates lock-in |
| "Restart the service every 4 hours" | Memory leak symptom suppression; the leak is the bug |
| Cache the bad response | Caches the bad behavior; users see stale wrong data |

These shortcuts are sometimes correct *as the second-line response* — but only after the root cause is understood and explicitly accepted as out-of-scope ("we know the bug; here's a temporary mitigation while we work on the real fix").

## Diagnostic toolkit

| Tool | When |
|---|---|
| `git log -p <file>` / `git blame` | "Why is this line written this way?" |
| `git bisect` | "Find the commit that broke X" — automated bisection |
| `git log --oneline --since=<date>` | What deployed recently |
| Production trace ID → distributed trace | What was the request actually doing |
| `EXPLAIN ANALYZE` (DB) | Where time is spent |
| `kubectl describe pod` / `events` | What the orchestrator thinks happened |
| `strace` / `dtrace` | Syscall-level for stuck processes |
| `pg_stat_activity` | What queries are running right now |

Don't reach for any of these until you've answered question 3 (the failing line). Speculative tooling without a hypothesis is busy-work.

## Reproducing the bug

If it's not reproducible, you don't understand it. Cheap reproductions:

1. **Capture the failing input.** Logs, request body, DB row state. If you can't capture, log more (and redeploy) until you can.
2. **Reproduce locally** with the captured input. If it doesn't reproduce locally, the difference between local and prod *is* the bug — find it (data, version, race, env var).
3. **Write a failing test** with the captured input. The test goes red. Fix until it goes green. Now you have a regression net.

Skipping step 3 is the most common mistake. A bug fix without a regression test is a bug waiting to come back.

## Intermittent / "flaky" bugs

Treat "flaky" as "deterministic, you don't see the variable yet." Common variables:

- **Time** — DST boundaries, timezone, clock skew, leap seconds.
- **Order** — test pollution, async race, parallel writes.
- **Volume** — fine at 10 RPS, breaks at 100. Look for resource exhaustion (file descriptors, connection pool, memory).
- **Locale** — non-ASCII data, RTL languages, unusual currencies.
- **Network** — DNS, slow upstream, partial reads.

Add logging for the suspected variables; let the bug happen; read the logs.

## When to stop

Stop when:
- You can articulate the cause as a single sentence that another engineer agrees with.
- The fix lives at the level of the cause, not the symptom.
- A regression test fails before the fix and passes after.
- You've audited for the same class of bug elsewhere ("are there other migrations that didn't backfill?").

## Anti-patterns

- **"It's flaky, retry"** — flakiness is unsolved bugs. Document it as such; quarantine the test (don't delete) while you investigate.
- **Fixing the line that throws instead of the line that produced the bad state** — symptom-level fix, ships the bug.
- **Reading logs without `git log`** — half the picture. Code state matters.
- **No reproduction → guess fix → ship → "looks ok"** — bugs come back. Reproduce or you don't know.
- **No regression test** — the bug will return at the worst time.
- **"Probably a race condition" without proof** — fine as a hypothesis, not a fix. Confirm.
- **Solo debugging for >2 hours** — fresh eyes are cheap. Get someone to walk through with you.

## Verify it worked

- [ ] You can describe the cause in one sentence ("X happens because Y when Z").
- [ ] The fix is at the level of the cause, not the symptom.
- [ ] A regression test exists, fails before the fix, passes after.
- [ ] You've audited for the same class of bug elsewhere in the codebase.
- [ ] If the bug was a deploy regression, you understand which commit introduced it.
- [ ] If intermittent, you can now reliably reproduce it (or have a clear hypothesis confirmed by adding logging).
- [ ] The PR description states the cause, not just "fixes #1234".
- [ ] No `try/catch`-and-log added without an explicit comment why.
