---
name: write-adr
description: Write an Architecture Decision Record — Context, Decision, Consequences, Alternatives Considered. Use when making any architecture-shaping choice (DB, framework, service boundary, vendor, data model) so the *reason* survives. ADRs are append-only and superseded, not edited.
---

# Write an ADR

An ADR (Architecture Decision Record) is the institutional memory of *why* a choice was made. Six months later when someone asks "why did we pick Postgres over Mongo?", the answer is in the ADR — not in someone's head, not in a Slack thread, not lost.

## When to use

- Picking a framework, language, DB, or vendor.
- Defining a service boundary or data ownership.
- Choosing a deploy target.
- Adopting (or removing) a major dependency.
- Setting an org-wide convention (e.g. "all services use OTel").
- Reversing a previous ADR.

## When NOT to use

- Implementation detail (variable names, file structure within a service).
- One-off scripts.
- Trivial choices ("which icon library").

If you're not sure: would future-you, joining the team in 12 months, want to understand this decision? If yes, write an ADR.

## The format

Numbered, immutable, in the repo:

```
docs/adr/
  0001-use-postgres-over-mongo.md
  0002-multi-tenant-via-rls.md
  0003-prefer-clerk-over-auth-js.md
  0004-rollback-of-0003-back-to-auth-js.md   ← supersedes 0003
```

ADRs are *appended*, never edited. If you change your mind, write a new ADR that supersedes the old one.

## The template

```markdown
# ADR-0042: Use Postgres for primary data store

- **Status**: Accepted
- **Date**: 2026-05-04
- **Deciders**: @lead-platform, @lead-data
- **Supersedes**: —
- **Superseded by**: —

## Context

What's the situation that forced a choice? What constraints exist?

We're starting a new SaaS product targeting Indian SMBs. We need a primary
data store for user, org, project, and billing data. The product will have
heavy relational queries (joins across users, orgs, memberships, projects)
and moderate write volume. Team has experience with Postgres and Mongo.

Constraints:
- Multi-tenant via row-level security (decided in ADR-0040).
- Compliance: DPDP Act 2023 — must be in India region.
- Budget: < $200/mo for first 12 months.

## Decision

What did we pick?

We will use **Postgres 16** (managed, AWS RDS in `ap-south-1`) as the
primary data store.

## Consequences

What does this commit us to? What gets harder, what gets easier?

**Easier:**
- Relational joins, transactions, and SQL — no app-side joining.
- Row-level security is native (matches ADR-0040).
- pgvector for future vector-search use cases.
- Postgres-friendly dev tools (Drizzle, sqlc, dbt).
- Hiring (most backend devs know Postgres).

**Harder:**
- Schema migrations require discipline (vs. Mongo's schemaless).
- Vertical scaling has a ceiling; sharding is non-trivial.
- Some doc-shaped data (user preferences, feature flags) feels awkward in SQL.

**Locked in to:**
- Postgres-specific extensions (pgvector, RLS).
- AWS RDS as managed provider (could move to Neon / Supabase / self-hosted Postgres).

## Alternatives Considered

What else did we think about? Why didn't we pick them?

### MongoDB

- Pros: schemaless ergonomics; team has prior experience; multi-region replication is easier.
- Cons: weaker transactional model; multi-tenant by collection-prefix or DB-per-tenant feels worse than RLS; team's multi-tenant experience is in SQL.
- **Why not**: Multi-tenant via RLS (ADR-0040) is much cleaner in Postgres.

### CockroachDB

- Pros: Postgres-compatible; horizontally scalable.
- Cons: ~3x cost vs. RDS Postgres; complexity not justified at our scale.
- **Why not**: We don't need horizontal scale yet; can migrate later (it's wire-compatible).

### Supabase

- Pros: Managed Postgres + auth + realtime + storage in one bill.
- Cons: Mixed feelings on Supabase auth's customization; opinions on RLS-everywhere are strong.
- **Why not**: We may revisit; for now, decoupled (Clerk + RDS) gives more flexibility. (See ADR-0043.)

## References

- ADR-0040: Multi-tenancy via Postgres RLS
- ADR-0043: Authentication via Clerk
- Discussion: github.com/acme/platform/issues/89
- Postgres on RDS pricing in ap-south-1: <link>
```

## Status values

| Status | Meaning |
|---|---|
| **Proposed** | In discussion, not yet adopted. |
| **Accepted** | The current decision. |
| **Deprecated** | We don't do this anymore, but no replacement yet. |
| **Superseded** | Replaced by ADR-XXXX. Keep the file; add link. |

## Numbering

Use 4-digit zero-padded numbers (`0001`, `0042`, `0150`). Never reuse a number even if an ADR is superseded — the superseded ADR stays in the repo.

## "Decision" should be one sentence

The "Decision" section is the one-liner. If you can't summarize the decision in one sentence, you don't have one decision; you have many entangled ones — split them.

## "Consequences" must include downsides

ADRs that only list upsides are marketing, not engineering records. Future-you needs to know what you signed up for. Be honest:

- What's harder now?
- What can't we easily change later?
- What did we accept as a known cost?

## "Alternatives Considered" is the highest-value section

Six months later, someone will say "should we switch from X to Y?" — and the answer to "did we already consider Y?" lives in this section. Don't skip it.

For each alternative:
- Pros (a sentence or two).
- Cons (a sentence or two).
- **The single reason we didn't pick it.** This is what future-you needs.

## Keep ADRs short

A long ADR is unread. Aim for 1–2 pages. If a decision needs more, link to a design doc and summarize here.

## Reversing a decision

If you change your mind, **don't edit the old ADR**. Write a new one:

```markdown
# ADR-0080: Switch from Clerk to Auth.js (supersedes ADR-0043)

- **Status**: Accepted
- **Date**: 2026-09-15
- **Supersedes**: ADR-0043

## Context

ADR-0043 picked Clerk for auth. After 6 months of production:

- Clerk's pricing curves up at our growth rate; projected $X/yr by Q4.
- Org/team UI customizations we need are limited.
- Auth.js v5 has matured significantly since ADR-0043.

## Decision

Migrate to Auth.js (NextAuth v5).

## Consequences

...
```

The old ADR remains. Anyone reading 0043 sees "Superseded by 0080" at the top.

## Anti-patterns

- **No "Alternatives Considered"** — kills the long-term value.
- **Editing an old ADR after it's accepted** — destroys history. Supersede instead.
- **Decisions without context** — context is *why* the decision made sense at the time.
- **Marketing-tone consequences** — only listing upsides.
- **ADR for trivial things** — variable naming, file paths. Keep the bar high.
- **No status field** — readers don't know if this is current.
- **Buried in a wiki** — should be in the code repo, version-controlled.
- **No deciders listed** — when challenged, no one knows who to talk to.
- **Decisions we wish we made vs decisions we made** — write the actual decision, even if you'd want a different one in hindsight. Then write a superseding ADR.
- **Vague or aspirational ADRs** ("We will be cloud-native") — non-falsifiable; useless. Concrete or skip.

## Verify it worked

- [ ] ADRs live in `docs/adr/NNNN-slug.md`, version-controlled.
- [ ] Each has Context, Decision, Consequences, Alternatives Considered.
- [ ] Status field is present and current.
- [ ] No ADR has been edited since being marked Accepted.
- [ ] Reversed decisions have new ADRs that supersede the old; old ADRs link to the new.
- [ ] An engineer joining the team can read the ADRs in order and understand the architecture's *why*.
- [ ] Decisions in flight are written as Proposed ADRs in PRs, discussed via PR review.
- [ ] PRs that change architecture link to or update an ADR.
- [ ] No "we should write an ADR for that" lives in Slack history without a follow-up.
