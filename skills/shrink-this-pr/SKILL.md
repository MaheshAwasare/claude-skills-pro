---
name: shrink-this-pr
description: Take a PR that's grown too large to review and split it into mergeable, ordered chunks with a dependency graph. Use when you (or someone else) opens a PR with >500 lines, mixes refactoring with features, or touches >10 files for one logical change.
---

# Shrink This PR

PRs over ~400 lines get rubber-stamped or stalled. Both outcomes are bad. This skill is the algorithm for splitting a fat PR into a stack of small, reviewable, mergeable PRs.

## When to use

- PR is >500 lines (excluding lockfiles, generated code).
- PR mixes mechanical refactor with new feature work.
- PR touches >10 files for one logical change.
- Reviewer says "I can't review this" or "lgtm" without comments (the latter is worse).
- You're about to *open* a PR like this — better to split before review than after.

## When NOT to use

- Changes are atomic (one schema migration + the matching code change).
- Time-critical hotfix where review-friendliness is secondary.
- Generated/scripted refactor of 1000 files (mass rename) — single PR is fine.

## The split algorithm

1. **List the categories of change in the diff.** Typical categories:
   - Mechanical refactor (rename, move, extract).
   - Type/lint cleanup.
   - Dead code removal.
   - New behavior (the feature).
   - Tests for the new behavior.
   - Config/infra changes.
   - Documentation.
2. **Order them by dependency.** Refactor before feature; feature before tests of feature; config before code that needs config.
3. **Group small categories** if they truly co-depend; split aggressively otherwise.
4. **Each chunk must merge independently.** "Won't compile without #2" means the split is wrong.
5. **Each chunk must be a real improvement on its own.** "Just refactor — no behavior change" PRs are easy to review and uncontroversial.

## Target sizes

| PR type | Lines (excluding generated) | Reviewer time |
|---|---|---|
| Mechanical refactor | up to ~800 | 5–10 min |
| Pure feature | up to ~300 | 15–30 min |
| Schema migration | as small as possible | varies |
| Bug fix | up to ~100 | 5 min |
| Config / dependency bump | as small as possible | 5 min |

If the feature genuinely needs 1000 lines of code, split *the feature itself* into milestones (skeleton → endpoint A → endpoint B → glue UI → polish).

## Concrete example

**Original PR (1400 lines):** "Add team invitations."

**Split into:**

1. **Refactor**: extract `email/templates.ts` registry from inline strings (~200 LOC, no behavior change). Reviewable as a pure refactor.
2. **Schema**: add `invitations` table with migration (~80 LOC). Independent — table can exist with no consumers.
3. **API skeleton**: `POST /invitations`, `GET /invitations/:token`, `POST /invitations/:token/accept`, returning 501 not-implemented for now (~150 LOC). Routing wired but logic stubbed.
4. **Email integration**: implement `sendInvitationEmail` using the registry from #1 (~120 LOC + tests).
5. **Accept flow**: implement `POST /invitations/:token/accept` end-to-end (~250 LOC + tests).
6. **UI**: invitation form + accept page (~400 LOC).
7. **Webhook events**: emit `invitation.created` and `invitation.accepted` to your event stream (~100 LOC).

Six small PRs replace one big one. Each merges independently. Production stays in a known-good state if PR #5 is delayed.

## How to actually do the split

```bash
# Start from your big branch
git checkout big-feature
git checkout -b split/01-refactor

# Cherry-pick or restage just the refactor parts
git reset --soft main                       # everything becomes uncommitted
git add -p                                  # interactively pick refactor hunks
git checkout -- .                           # reset everything else
git commit -m "refactor: extract email templates registry"

# Open PR for #01
gh pr create

# Now branch from #01 for #02
git checkout -b split/02-schema split/01-refactor
git checkout big-feature -- migrations/*.sql packages/db/schema.ts
git commit -m "feat: add invitations table"
gh pr create --base split/01-refactor

# ... etc
```

Stack the PRs (each based on the previous). Tools like `gh stack` or `Graphite` make this easier.

## When the work is intertwined

Sometimes a "refactor + feature" really *is* mechanically inseparable. Test:

- Can the refactor be merged with `// TODO: use this` comments and the feature follow later? If yes, split.
- Does the refactor change behavior? If yes, it's not a refactor — it's a feature with cleanup.
- Is the "refactor" actually 50 line moves + 5 logic tweaks? Pull out the moves first; the 5 tweaks become a clean feature PR.

If you genuinely cannot split, write the PR description so a reviewer can read it in chunks: a "Read this in order" guide pointing to specific files.

## Anti-patterns

- **Splitting after the giant PR is up** — political cost. Try to split before opening.
- **PR that "won't compile until #2 lands"** — the split is wrong. Stub or feature-flag.
- **Splitting by file** — one PR per file makes 50 PRs nobody reads. Split by *concept*, not by location.
- **A "tests" PR after the feature PR** — code without tests shouldn't merge. Tests in the same PR as the feature.
- **Force-push history rewrites mid-review** — review comments orphan. Open a follow-up PR instead.
- **Mixing dependency upgrades with feature work** — bump deps in their own PR; isolate the variable.
- **"It's actually fine, just review it"** — reviewer fatigue ships bugs. Respect the reviewer's time.
- **A 1000-line "refactor" that also "happens to fix bug X"** — the bug fix needs its own PR with a test; reviewers can't audit refactors and behavior changes simultaneously.

## Verify it worked

- [ ] Each PR is < 400 lines (excluding generated/lockfiles).
- [ ] Each PR has a clear, independent purpose stated in the description.
- [ ] Each PR merges cleanly — no "depends on PR X" beyond the immediate parent in a stack.
- [ ] Each PR is *valuable on its own* (a refactor that improves the code without the feature is still valuable).
- [ ] Tests live with the code that needs them, not in a separate "tests PR."
- [ ] Production stays in a known-good state at every merge — no half-feature in main.
- [ ] PR descriptions explain how the chunks compose ("part 3/7 of team-invites").
- [ ] Reviewer can review each PR in < 30 min.
