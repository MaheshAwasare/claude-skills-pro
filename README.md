# claude-skills-pro

> 51 production-grade Claude Code skills covering the gaps Anthropic doesn't ship — Razorpay, Brevo, India-DPDP compliance, full-stack scaffolding, framework migrations, and the workflow meta-skills that change how Claude operates.

[![Status](https://img.shields.io/badge/skills-51%2F51-brightgreen)](#the-51-skills-roadmap)
[![License](https://img.shields.io/badge/license-MIT-blue)](LICENSE)

A curated, opinionated, MIT-licensed plugin for [Claude Code](https://claude.com/claude-code). Every skill is medium-depth (~150–250 lines) with concrete code examples, anti-patterns, and a "verify it worked" checklist — not a one-paragraph prompt wrapper.

## Why this exists

Anthropic's official skills cover the basics across most languages. This repo deliberately fills the gaps:

- **India-first payments + email** — Razorpay and Brevo, written by someone who actually ships with them.
- **Real scaffolding** — multi-tenant SaaS, Go microservices, CLI tools — wired end to end (DB + auth + payments + email + CI), not a `create-next-app` wrapper.
- **Workflow meta-skills** — `find-real-bug`, `shrink-this-pr`, `plan-the-rollback` change how Claude *operates*, not just what it knows.
- **Compliance Anthropic hasn't shipped** — DPDP Act 2023 (India), HIPAA, SOC2 evidence collection, WCAG audit.
- **Migrations** — concrete before/after for Next.js Pages→App, Webpack→Vite, Mongo→Postgres, etc.

## Install

```bash
# Clone into your Claude Code plugins directory
git clone https://github.com/MaheshAwasare/claude-skills-pro.git
```

Then point Claude Code at the `skills/` directory, or symlink individual skills into `~/.claude/skills/`.

Per-skill install:
```bash
ln -s "$(pwd)/skills/razorpay-integration" ~/.claude/skills/razorpay-integration
```

## The 51 skills (roadmap)

Status legend: ✅ shipped · 🚧 in progress · ⬜ planned · ⭐ signature skill

### Bucket B — Platform / Tool (22) — *shipping first*

| # | Skill | Status |
|---|---|---|
| 1 | [`razorpay-integration`](skills/razorpay-integration) ⭐ | ✅ |
| 2 | [`brevo-email`](skills/brevo-email) ⭐ | ✅ |
| 3 | [`scaffold-saas-starter`](skills/scaffold-saas-starter) ⭐ | ✅ |
| 4 | [`scaffold-fullstack-app`](skills/scaffold-fullstack-app) | ✅ |
| 5 | [`scaffold-go-microservice`](skills/scaffold-go-microservice) | ✅ |
| 6 | [`scaffold-cli-tool`](skills/scaffold-cli-tool) | ✅ |
| 7 | [`terraform-patterns`](skills/terraform-patterns) | ✅ |
| 8 | [`kubernetes-helm`](skills/kubernetes-helm) | ✅ |
| 9 | [`github-actions-ci`](skills/github-actions-ci) | ✅ |
| 10 | [`opentelemetry-instrument`](skills/opentelemetry-instrument) | ✅ |
| 11 | [`stripe-integration`](skills/stripe-integration) | ✅ |
| 12 | [`clerk-auth`](skills/clerk-auth) | ✅ |
| 13 | [`supabase-backend`](skills/supabase-backend) | ✅ |
| 14 | [`sentry-monitoring`](skills/sentry-monitoring) | ✅ |
| 15 | [`dbt-data-modeling`](skills/dbt-data-modeling) | ✅ |
| 16 | [`graphql-relay`](skills/graphql-relay) | ✅ |
| 17 | [`grpc-services`](skills/grpc-services) | ✅ |
| 18 | [`bun-runtime`](skills/bun-runtime) | ✅ |
| 19 | [`fastapi-production`](skills/fastapi-production) | ✅ |
| 20 | [`react-native-expo`](skills/react-native-expo) | ✅ |
| 21 | [`cloudflare-workers`](skills/cloudflare-workers) | ✅ |
| 22 | [`algolia-search`](skills/algolia-search) | ✅ |

### Bucket A — Workflow / Meta (15)

| # | Skill | Status |
|---|---|---|
| 23 | [`scaffold-new-app`](skills/scaffold-new-app) ⭐ | ✅ |
| 24 | [`find-real-bug`](skills/find-real-bug) ⭐ | ✅ |
| 25 | [`shrink-this-pr`](skills/shrink-this-pr) | ✅ |
| 26 | [`write-commit-message`](skills/write-commit-message) | ✅ |
| 27 | [`explain-this-diff`](skills/explain-this-diff) | ✅ |
| 28 | [`triage-stack-trace`](skills/triage-stack-trace) | ✅ |
| 29 | [`audit-tautological-tests`](skills/audit-tautological-tests) | ✅ |
| 30 | [`extract-skill-from-session`](skills/extract-skill-from-session) | ✅ |
| 31 | [`find-dead-code`](skills/find-dead-code) | ✅ |
| 32 | [`write-runbook`](skills/write-runbook) | ✅ |
| 33 | [`write-adr`](skills/write-adr) | ✅ |
| 34 | [`plan-the-rollback`](skills/plan-the-rollback) | ✅ |
| 35 | [`spec-from-conversation`](skills/spec-from-conversation) | ✅ |
| 36 | [`blame-archaeology`](skills/blame-archaeology) | ✅ |
| 37 | [`write-pr-description`](skills/write-pr-description) | ✅ |

### Bucket D — Migration / Upgrade (7)

| # | Skill | Status |
|---|---|---|
| 38 | [`nextjs-pages-to-app`](skills/nextjs-pages-to-app) | ✅ |
| 39 | [`node-version-upgrade`](skills/node-version-upgrade) | ✅ |
| 40 | [`java-8-to-21`](skills/java-8-to-21) | ✅ |
| 41 | [`webpack-to-vite`](skills/webpack-to-vite) | ✅ |
| 42 | [`jest-to-vitest`](skills/jest-to-vitest) | ✅ |
| 43 | [`mongo-to-postgres`](skills/mongo-to-postgres) | ✅ |
| 44 | [`python-2-to-3`](skills/python-2-to-3) | ✅ |

### Bucket C — Compliance / Regulated (7)

| # | Skill | Status |
|---|---|---|
| 45 | [`india-dpdp-act`](skills/india-dpdp-act) ⭐ | ✅ |
| 46 | [`gdpr-dpia`](skills/gdpr-dpia) | ✅ |
| 47 | [`hipaa-audit`](skills/hipaa-audit) | ✅ |
| 48 | [`soc2-evidence`](skills/soc2-evidence) | ✅ |
| 49 | [`pci-dss-checklist`](skills/pci-dss-checklist) | ✅ |
| 50 | [`wcag-accessibility-audit`](skills/wcag-accessibility-audit) | ✅ |
| 51 | [`threat-modeling-stride`](skills/threat-modeling-stride) | ✅ |

## Skill format

Every skill follows the same structure:

```
skills/<skill-name>/
  SKILL.md          # frontmatter + body
  examples/         # optional: runnable code samples
  templates/        # optional: file templates the skill emits
```

The SKILL.md body always includes:
1. **When to use** / **When NOT to use**
2. **Core concepts** (the mental model)
3. **Concrete code/config examples**
4. **Anti-patterns** (what NOT to do, with reasons)
5. **Verify it worked** (a checklist)

## Contributing

PRs welcome. See [CONTRIBUTING.md](CONTRIBUTING.md). The bar:
- Medium depth (~150–250 lines).
- Concrete examples, not vague advice.
- An anti-patterns section (this is what differentiates a real skill from a prompt).
- A verification checklist.

## License

MIT — see [LICENSE](LICENSE). Use freely in commercial and open-source projects.
