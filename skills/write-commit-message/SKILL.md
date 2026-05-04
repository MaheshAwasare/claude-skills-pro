---
name: write-commit-message
description: Write a Conventional Commit message from a staged diff — type, scope, subject, body that explains *why*, footer for breaking changes and issue refs. Use after staging changes when the auto-generated message would be a useless "update files" or generic bullet list.
---

# Write a Commit Message

A commit message is the only thing future-you will see when running `git blame`. The diff shows *what*; the message must show *why*. This skill produces messages that age well.

## When to use

- After staging a change, before committing.
- Reviewing a PR with default-generated commit messages ("Update foo.ts").
- Squashing a stack of commits into one — write the squash message properly.

## When NOT to use

- WIP/scratch commits on a feature branch that will be squashed (use `wip:` and move on).
- Auto-generated commits (release-please, dependabot) — let the tools handle.

## The format (Conventional Commits + body)

```
<type>(<scope>): <subject>      ← line 1, ≤ 72 chars

<body — wrapped at 72>          ← optional but usually needed
Why this change.
What it replaces.
Trade-offs accepted.

<footer>                         ← optional
BREAKING CHANGE: <description>
Refs: #123
```

### Types (use these, not custom ones)

| Type | When |
|---|---|
| `feat` | New user-visible feature |
| `fix` | Bug fix |
| `perf` | Performance improvement |
| `refactor` | Code change without behavior change |
| `test` | Add/change tests only |
| `docs` | Documentation only |
| `build` | Build system, package manager, deps |
| `ci` | CI configuration |
| `chore` | Maintenance, config, version bumps |
| `revert` | Reverts a previous commit |

`feat:` and `fix:` are user-visible. The rest aren't. Use `feat!:` / `BREAKING CHANGE:` for breaking changes — release-please uses these to bump major versions.

### Subject line rules

- **Imperative mood**: "add invoice export" not "added" or "adds."
- **Lowercase first letter** (after the type prefix).
- **No trailing period.**
- **Under 72 chars** — git tooling wraps poorly past this.

| Bad | Good |
|---|---|
| `Updated files` | `fix(auth): allow + in email addresses for sign-up` |
| `Various improvements` | `perf(search): cache embeddings for 1h, drop p95 from 800ms to 90ms` |
| `WIP` (on main) | `feat(billing): generate GST invoice PDF on payment capture` |

### Body rules (the part that matters in 6 months)

The body answers **why**, not what. The diff shows what.

- **Why was this needed?** (issue, user complaint, regression, design choice)
- **What did you reject?** (alternatives considered, why rejected)
- **What's the risk / trade-off?**

Hard rule: if the body could be deleted without losing information, delete it. If you find yourself writing "this commit changes X to do Y" — that's the diff. Cut it.

### Footer rules

- `BREAKING CHANGE: <details>` — required for major-bumping changes.
- `Refs: #123` or `Fixes: #123` — links to issues.
- `Co-authored-by: Name <email>` — for pair work.

## Worked examples

### Bug fix

```
fix(payments): trim whitespace from razorpay webhook signature

Some proxies in the path were appending a trailing newline to the
X-Razorpay-Signature header. Our HMAC compare was strict, so 100% of
webhooks from those paths were rejected (~0.4% of total volume).

We now trim header whitespace before comparison. Razorpay's signature
spec doesn't permit whitespace in the actual hex digest, so this is
safe.

Refs: #2147
```

### Feature

```
feat(billing): support partial refunds via Razorpay Optimum

Customer support was issuing full refunds because the dashboard didn't
expose partial. Refund volume up 8x from the change request itself,
unsustainable on full-refund only.

Adds POST /api/refunds with optional amount field. When amount is less
than the original payment, uses Razorpay's "optimum" speed path
(instant refund via UPI/IMPS where supported).

Considered: refund line-items mapped to invoice lines (rejected for v1
scope; coming as #2233).

Refs: #2156
```

### Refactor (no behavior change)

```
refactor(email): extract Brevo template IDs into TEMPLATES registry

Template IDs were hardcoded in 12 call sites. Moving the OTP template
to a different ID required 12 edits and a deploy.

Now in src/email/templates.ts; call sites reference TEMPLATES.otp etc.
No behavior change — same template IDs, same parameters.
```

### Breaking change

```
feat(api)!: rename /v1/users/{id}/projects to /v1/projects?owner={id}

Resource-as-collection-member is harder to filter and paginate. Moving
projects to a top-level collection enables shared-with-me queries we
need for #1890.

BREAKING CHANGE: Clients calling /v1/users/{id}/projects must move to
/v1/projects?owner={id}. The old route returns 410 with a Link header
pointing to the new URL during a 30-day deprecation window.
```

## Squash messages

When merging a PR with `Squash and merge`, the squash message replaces 20 WIP commits. Don't accept GitHub's default (which concatenates the WIP titles). Edit it before clicking Confirm.

A squash message is just a normal commit message — apply the rules above. The PR description usually has the right content; copy from there.

## Multi-paragraph bodies

Break with blank lines:

```
fix(scheduler): skip jobs in suspended state on resume

[paragraph 1: the bug]

[paragraph 2: the fix]

[paragraph 3: what was rejected]
```

Three paragraphs is plenty. If you're writing five, you're either explaining the diff (cut) or this should be a design doc / ADR (link to it from the body).

## Anti-patterns

- **`Update foo.ts`** / `Various changes` — useless. Future-you can't `git log --grep` for this.
- **`WIP` on main** — squash before merging.
- **Body that paraphrases the diff** — adds zero info. Delete the body.
- **Subject lines that describe the file changed** — describe the *behavior* changed.
- **Past tense** ("added X") — convention is imperative ("add X"). Reads as a command to the codebase.
- **Period at end of subject** — visually noisy in `git log --oneline`.
- **Conventional Commit type for trivial changes** (`chore: rename variable`) — fine in moderation, but `chore:` everywhere makes types meaningless. Use real types.
- **Empty body when one is needed** — bug fixes especially. Why was it broken? Future blame walker needs that.
- **Co-authored lines for solo work** — dilutes the convention.
- **Issue refs in subject** ("(#123) fix login") — put them in the footer.
- **Mixing two changes in one commit** — split before committing. `git add -p` is your friend.

## Verify it worked

- [ ] Subject line is ≤ 72 chars, imperative, lowercase, no period.
- [ ] Type prefix matches one of the standard types.
- [ ] Subject describes the *change*, not the file.
- [ ] If the body exists, it explains *why*, not *what*.
- [ ] Breaking changes have `BREAKING CHANGE:` footer.
- [ ] Issue refs in footer, not subject.
- [ ] No "WIP" or "checkpoint" survives onto main (squashed).
- [ ] `git log --oneline` of the last 30 commits is readable as a changelog.
- [ ] If running release-please / semantic-release, the version bumps match expectations.
