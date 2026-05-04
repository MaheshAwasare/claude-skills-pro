---
name: scaffold-new-app
description: Interactive scaffolder that picks a stack from a few questions, then bootstraps the right project (SaaS / fullstack / Go service / CLI / etc) wired end-to-end. Routes to the matching scaffold-* skill. Use when starting any new project — saves the "should I use Next.js or just Express?" decision spiral.
---

# Scaffold a New App

Meta-skill that asks the right questions, picks a stack, and routes to the appropriate scaffold-* skill. Stops the "decision paralysis" tax of starting a new project.

## When to use

- Starting any new project where the stack isn't pre-decided.
- Helping someone else start a project and they don't know what to pick.
- Replacing a hacked-together prototype with a production-shaped scaffold.

## When NOT to use

- Project is already underway — adding scaffolding to an existing app is surgery, not scaffolding.
- The user has already declared a stack ("scaffold a Rails app") — go straight to that scaffold.
- Constrained environment (regulated, customer-mandated stack) — follow the constraint.

## The decision tree

Walk through these questions with the user, *in order*. Stop at the first answer that determines the stack.

### Q1: What is this project for?

| Answer | Likely stack |
|---|---|
| B2B SaaS, multi-tenant | → `scaffold-saas-starter` |
| Indie SaaS / B2C app | → `scaffold-fullstack-app` |
| Internal tool | → `scaffold-fullstack-app` (or low-code if it's truly simple) |
| API-only service | → `scaffold-go-microservice` (or `fastapi-production` if Python team) |
| CLI tool | → `scaffold-cli-tool` |
| Mobile app | → `react-native-expo` (cross-platform) or native-only stack |
| Static marketing site | → Astro or plain Next.js (minimal) |
| Background data pipeline | → `dbt-data-modeling` for SQL, custom Python/Go for everything else |
| ML inference API | → `fastapi-production` |

### Q2: Who's the team?

| Answer | Adjustment |
|---|---|
| Solo founder | Optimize for speed-to-deploy → indie SaaS scaffold + Vercel + Neon |
| Small team (2–5) | Same indie scaffold; pin tooling versions |
| Medium team (6–20) | SaaS scaffold; consider monorepo with apps/* packages |
| 20+ engineers | Get a senior architect; this skill isn't for you |

### Q3: What's the geographic / compliance footprint?

| Answer | Adjustment |
|---|---|
| India primary | Razorpay over Stripe; consult `india-dpdp-act` skill |
| EU primary | GDPR posture from day 1; data residency in EU |
| US/global | Stripe; consider `pci-dss-checklist` for payment-handling scope |
| HIPAA | Specialist review needed; baseline scaffold + `hipaa-audit` skill |
| None / dev tool | Skip compliance skills |

### Q4: What's the deployment target?

| Answer | Adjustment |
|---|---|
| Vercel + Neon | Default for Next.js scaffolds |
| Cloudflare | `cloudflare-workers` scaffold instead |
| Railway / Fly | Same scaffolds; tweak deploy config |
| Self-hosted Kubernetes | `scaffold-go-microservice` + `kubernetes-helm` |
| Lambda | Probably wrong choice for a new app — challenge it |

### Q5: Will this need real-time features?

| Answer | Adjustment |
|---|---|
| No | Default scaffolds |
| Chat / collaborative | Add Durable Objects or Pusher; consider Cloudflare Workers |
| Live dashboards | Server-Sent Events from API (cheaper than WebSocket) |
| Multi-cursor / OT | This is a specialist build; out of scope for scaffolding |

## The output

After the questions, this skill produces:

1. **A stack decision summary** — the actual stack picked, written down in `docs/stack.md`.
2. **The route to the right `scaffold-*` skill** — which then emits the project.
3. **A list of follow-on skills the user will likely need** — e.g. `razorpay-integration`, `clerk-auth`, `sentry-monitoring`.

## When defaults are wrong

Push back on the user when their request implies a poor fit:

- **"I want to build a SaaS in Rust"** — challenge: do you actually have Rust expertise? Productivity hit is significant for non-CPU-bound web work.
- **"I want to use [obscure framework]"** — challenge: who else on the team can maintain this?
- **"I want microservices from day 1"** — challenge: do you have the team size to operate them? Default to a modular monolith.
- **"I want both Stripe and Razorpay wired"** — challenge: do you have the geographic split to justify this maintenance load?
- **"I want my own auth"** — challenge: have you used Clerk/Auth0? Building auth is six weeks you don't get back.

The right answer is sometimes "I want X anyway, here's why" — that's fine. But surface the trade-off; don't silently scaffold a regrettable choice.

## Anti-patterns

- **Scaffolding without asking** — produces wrong stack 60% of the time.
- **Asking 20 questions** — most decisions can be defaulted; only 3–5 actually drive the stack.
- **Defaulting to the latest fashion** — "Use the new shiny" is a costly default. Recommend boring tech with track record.
- **Picking microservices for "future scale"** — you don't have scale; you have an extra week of ops setup.
- **Picking a backend language nobody on the team knows** — productivity loss outweighs theoretical benefits.
- **Not committing the stack decision to the repo** — six months later, nobody remembers why you picked X.
- **Skipping the compliance question for an India/EU/healthcare app** — late compliance work is 10x harder than early.

## Verify it worked

- [ ] User answered Q1; you've picked the matching scaffold-* skill.
- [ ] Adjustments from Q3 (compliance) and Q4 (deploy target) are applied.
- [ ] `docs/stack.md` exists in the repo with the decision and rationale.
- [ ] User has a list of follow-on skills they'll likely invoke.
- [ ] Push-back happened if the request implied a poor fit.
- [ ] The scaffold runs on the user's machine without manual intervention beyond `pnpm install`.
- [ ] The user can describe the stack in one sentence ("Next.js App Router + Postgres RLS + Clerk + Razorpay + Brevo, on Vercel + Neon").
