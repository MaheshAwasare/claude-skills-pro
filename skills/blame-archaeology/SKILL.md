---
name: blame-archaeology
description: Trace why a piece of code exists via git history — original commit, the PR, the issue, the discussion, the constraints. Use when about to delete or refactor old-looking code, when behavior is mysterious, or when bisecting a regression. Refuses surface-level "this looks ancient, delete it" decisions.
---

# Blame Archaeology

Old code is often older for a reason. The rule "anything > 2 years old should be deleted" ships bugs. This skill is the disciplined way to find out *why* before you touch.

## When to use

- About to delete code that "looks unused" or "looks old."
- Refactoring a confusing function whose original intent isn't clear.
- Bisecting a regression — finding the commit that introduced behavior X.
- Onboarding to a codebase and trying to understand why X is the way it is.
- Code review of a delete PR — verify the deleter did this homework.

## When NOT to use

- Definitely-dead code from a static analyzer with full reference search done (see `find-dead-code` tier 1).
- Trivial surface fix (rename a variable in a 5-line function).
- New code (no history yet).

## The archaeology workflow

### 1. `git blame` the line

```bash
git blame -L 50,80 path/to/file.ts
```

Get the commit SHA for the line in question. Example output:
```
abc123de (Mahesh 2024-08-12) function chargeCustomer(id: string, amount: number) {
abc123de (Mahesh 2024-08-12)   if (amount < 100) throw new Error("min 100");
def456gh (Priya 2024-09-03)    if (currency !== "INR") throw new Error("INR only");
```

### 2. Read the commit

```bash
git show abc123de
```

Read the message and the full diff. The message often tells you 80% of why. If it's `"fix bug"` or `"update file"`, you have a useless message — go to step 3.

### 3. Find the PR

```bash
git log --merges --grep="abc123de" --oneline
# Or for GitHub:
gh pr list --search "abc123de"
```

The PR description is usually richer than the commit message. It may link to the issue, design doc, or Slack thread.

### 4. Find the issue / design doc

PR descriptions often say `Closes #1234` or link to a design doc. Open it. The issue is the *why*.

### 5. Look at the broader context

```bash
git log --oneline --all path/to/file.ts | head -20
```

What's the file's history? Is this line part of a series of related changes? Were there reverts? Was it modified recently?

### 6. Check who else touched this

```bash
git log --format='%an' path/to/file.ts | sort -u
```

If three engineers wrote this code over years, ask one of them on Slack if they remember.

### 7. Search for related discussion

`git log --all --grep="<keyword>"` for any commit referencing related concepts. `gh issue list --state all --search "<keyword>"`. Slack archive search if you have it.

## The `git bisect` flow (for regressions)

If "feature X used to work, now it doesn't":

```bash
git bisect start
git bisect bad                          # current commit is bad
git bisect good v1.4.0                  # last known good
# Git checks out the midpoint; test it
# Tell git the result:
git bisect good                         # or `bad`
# Repeat until git names the culprit commit
git bisect reset
```

Automated bisect with a script:
```bash
git bisect run ./scripts/check-feature-x.sh
```

The script returns 0 (good) or non-zero (bad). Git automates the binary search. Lethal for "which of these 200 commits broke it."

## Reading old code well

Common patterns in old code that look "wrong" but are correct:

| Pattern | Likely reason |
|---|---|
| Hardcoded value with no comment | Was a config until config system existed |
| Defensive null check on something that "can't be null" | Production crash in 2018 you don't remember |
| Long if-else instead of polymorphism | Was simpler at the time; polymorphism added cost |
| Manual SQL instead of ORM | Performance fix; ORM was generating bad queries |
| `setTimeout(..., 0)` after a function call | Race condition fix in a specific browser |
| `// TODO: refactor this` from 2019 | The TODO is now load-bearing |
| Two near-identical functions | Diverged from one; the divergence is the bug fix |

When you find code that *seems* wrong, the question to ask is: "If this is wrong, why has it been working in production for 4 years?" Either:
- It's wrong and there's a latent bug (rare with old code).
- The "wrong" reading is missing context that the archaeology will reveal.
- It was correct for the constraint at the time; the constraint may have changed (now you can refactor).

## When the code is genuinely orphaned

After archaeology, you might find:

- The original author left the company.
- The PR was an unreviewed merge.
- The issue is closed with no resolution notes.
- The Slack thread is archived and unsearchable.

In this case, the code is genuinely under-documented. Your archaeology *report* becomes the new documentation:

```ts
// Originally added in PR #234 (Aug 2024) by @mahesh-old.
// Per git history + #2156: enforced because Razorpay's API silently
// rejected sub-100-paise amounts with a generic 400, so we 422 early.
// Razorpay raised this minimum in 2025; check if still needed.
if (amount < 100) throw new Error("min 100");
```

This comment, born of archaeology, is gold for the next person. Don't write it for trivia — write it when archaeology reveals something the code itself doesn't say.

## Decisions you can defend after archaeology

Archaeology earns you the right to:

- **Delete the code** — "I traced this back to PR #234, fixing a Razorpay issue that was resolved server-side in 2025. Verified with manual test that the error case no longer occurs. Removing."
- **Refactor without losing behavior** — "Original intent was X (per #234). My refactor preserves X with cleaner structure."
- **Add a comment that captures intent** — "Why this is here" survives future archaeology.
- **Decide it's still load-bearing** — "Looks ancient, but the regression test still fails when removed; we still need this."

Decisions you can't defend without archaeology:

- "I don't recognize this; deleting."
- "This pattern is outdated; refactoring."
- "This `if` looks redundant; removing."

## Anti-patterns

- **Surface-level "this looks ancient"** without history check.
- **Deleting based on `git log --since=2y` showing no recent changes** — old code is often *correct* code.
- **Accepting commit message as full history** — many messages are useless; PR + issue + Slack is the trail.
- **Not bisecting when bisect is the right tool** — manual hunting through 100 commits.
- **Trusting your reading of "weird" code** without checking the constraint context — what was true in 2019 may not be true today, but it was for a reason.
- **Adding archaeology comments to every line** — only where the code itself doesn't reveal the intent. Don't comment-spam.
- **Skipping archaeology because "I'm in a hurry"** — the bug your refactor reintroduces will cost more time than the archaeology would have.
- **Asking "who can I blame for this?"** in Slack — `git blame` doesn't assign blame, it traces context. Phrase as "who has context on X?"

## Verify it worked

- [ ] Used `git blame` + `git show` to find the commit.
- [ ] Found the PR; read its description.
- [ ] Found and read the linked issue / design doc.
- [ ] Articulated the *original constraint* and *whether it still holds*.
- [ ] Decision (keep / delete / refactor) is defended by the archaeology, not by surface reading.
- [ ] If the code is kept, a comment captures the intent (when archaeology revealed something non-obvious).
- [ ] If the code is removed/refactored, the PR description references the archaeology done.
- [ ] For regressions: `git bisect` identified the introducing commit; the fix targets that, not the symptom.
