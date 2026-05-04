---
name: gdpr-dpia
description: Conduct a GDPR Data Protection Impact Assessment from code, data flows, and processing purposes — identify high-risk processing, document mitigations, ROPA (records of processing activities). Use when EU users are involved and you're starting new processing, or when an audit asks for the DPIA you don't have.
---

# GDPR DPIA (Data Protection Impact Assessment)

A DPIA is a structured risk analysis required under GDPR Article 35 for "high-risk" processing. It's also the artifact every audit, customer security review, and ICO query asks for. This skill produces a defensible DPIA from your codebase reality — not a copy-pasted template.

## When to use

- Starting a new product/feature processing EU personal data.
- Adding processing of "special category" data (health, biometrics, religion, politics, sexual orientation, etc.).
- Large-scale systematic monitoring (CCTV, behavioral analytics).
- Profiling or automated decision-making with legal/significant effect.
- Audit / customer security review demands the DPIA.
- Onboarding to a product that doesn't have one — retroactive DPIA is better than no DPIA.

## When NOT to use

- Pure B2B with no consumer data and no employee monitoring — usually below DPIA threshold.
- Processing types explicitly listed by your DPA (Data Protection Authority) as low-risk.
- One-off, ad-hoc data use — usually doesn't trigger DPIA but document the assessment.

## When DPIA is mandatory

Per GDPR Art. 35(3), you MUST do a DPIA for:
1. Systematic and extensive evaluation/profiling with legal effects.
2. Large-scale processing of special category data.
3. Systematic monitoring of public areas.

Plus your DPA's published "blacklist" — for many EU DPAs this includes biometrics, employee monitoring, AI decision-making, etc.

When in doubt, do it. The DPIA is *also* useful internally even when not strictly required.

## DPIA structure (the template)

```markdown
# DPIA: <feature/product/system name>

- **Author**: <name>, <role>
- **Date**: 2026-05-04
- **Reviewers**: DPO, Legal, Security, Engineering Lead
- **Status**: Draft / In Review / Approved / Superseded

## 1. Description of processing

What does this system do, in plain English?

- Categories of data subjects (users, employees, prospects, etc.).
- Categories of personal data processed.
- Volume estimate (how many people, how often).
- Data sources (collected directly / from third parties / observed / inferred).
- Recipients (other internal systems, processors, third parties).
- International transfers (which countries; mechanism: SCCs, adequacy decision, etc.).
- Retention periods.
- Technical and organizational measures (TOMs) in place.

## 2. Necessity and proportionality

Why is this processing necessary? Could the same outcome be achieved with less data?

- Lawful basis under Art. 6 (consent / contract / legal obligation / vital interests / public interest / legitimate interests).
- For special category data: Art. 9 condition.
- Data minimization analysis: are we collecting only what's necessary?
- Storage limitation: are retention periods justified?

## 3. Risks to data subject rights and freedoms

For each risk: likelihood (low/medium/high) × severity (low/medium/high) = inherent risk.

| # | Risk | Likelihood | Severity | Inherent | Mitigation | Residual |
|---|---|---|---|---|---|---|
| 1 | Unauthorized access via breach | Medium | High | High | Encryption at rest, RBAC, audit logs, breach detection | Low-Medium |
| 2 | Inaccurate profile leading to wrong decisions | Medium | Medium | Medium | Human review before decision, correction interface | Low |
| 3 | Unlawful cross-border transfer | Low | High | Medium | All processors in EU or via SCCs + TIA | Low |
| 4 | Excessive retention | High (default) | Medium | High | Auto-delete jobs at 12-month mark | Low |

## 4. Mitigations (the engineering-meaningful part)

Concrete measures, with owners and dates:

- [ ] **Encryption at rest**: Postgres TDE enabled (RDS managed). Owner: @platform. Status: ✅
- [ ] **RBAC**: Roles: admin, support, analyst, user. Reviewed quarterly. Owner: @security. Status: ✅
- [ ] **Audit logging**: All admin reads of user data are logged with `who/what/when`. Retention 1y. Owner: @platform. Status: ✅
- [ ] **Subject access portal**: `/me/data-export` returns user's data within 5 min. Owner: @product. Status: 🚧
- [ ] **Right to erasure**: `/me/delete-account` triggers cascade + scheduled hard-delete. Owner: @product. Status: ✅
- [ ] **Breach detection**: WAF anomaly alerts → on-call. Owner: @security. Status: ✅
- [ ] **DPO contact**: dpo@example.com, published in privacy notice. Owner: @legal. Status: ✅
- [ ] **Data Processing Agreement (DPA) with all sub-processors**: Stripe, Brevo, Sentry, Vercel. Owner: @legal. Status: ✅

## 5. Outcome

- **Risk level after mitigation**: Low / Medium / High.
- **Decision**: proceed / proceed with conditions / consult DPA / do not proceed.
- **Conditions**: <if any>.
- **DPO sign-off**: <date>.
- **Re-review trigger**: re-do this DPIA if any of: 1) new data category added, 2) new sub-processor, 3) processing purpose changes, 4) volume increases by 10x.
```

## How to fill it from code

### Step 1: ROPA-style data inventory

Walk the database schema. For each table with PII:

```yaml
table: users
data_categories: [contact (email, phone), account (password_hash, created_at)]
purpose: account_authentication
lawful_basis: contract
retention: until_account_deleted_plus_30_days
sub_processors: [clerk (auth)]
encryption: at_rest_yes, in_transit_yes
access: app_role_user (read_own), app_role_support (read_others, audit_logged)
```

Build this inventory from `\d table_name` (Postgres) or your ORM models. It feeds section 1 of the DPIA.

### Step 2: data flow diagram

Draw or describe: where does data come from, where does it go?

```
[User browser] --HTTPS--> [API (vercel)]
                             |--Postgres (Neon EU)
                             |--Brevo (transactional email, EU)
                             |--Stripe (payment processor, US/EU)
                             |--Sentry (error tracking, EU region)
                             |--PostHog (analytics, self-hosted EU)
```

Each arrow is a transfer. Each node is a processor. Each cross-border arrow needs an Art. 46 mechanism (SCCs).

### Step 3: identify high-risk paths

For each data flow, ask:
- Does this leave the EU? (transfer risk)
- Is there a join across data categories that could re-identify? (linkability risk)
- Does this feed an automated decision? (Art. 22 risk)
- Is this special category data? (Art. 9 risk)
- Is volume large? ("large scale" is fuzzy; > tens of thousands is usually large)

### Step 4: mitigations from code

For each risk, what does your code/infra do today?

- "Encryption at rest" → check Postgres / S3 / Brevo / Stripe configs. If on, ✅. If not, action item.
- "Audit logging" → check that admin reads of PII tables emit a log line with operator ID. If not, action item.
- "Right to erasure" → is there a deletion endpoint? Does it cascade? Are backups handled (e.g. encrypted with per-user key, or out-of-scope by retention)?
- "Subject access" → can a user get their data within 30 days (GDPR limit)? Self-service or manual?

The DPIA's value is exposing the gaps. Those become engineering tickets.

## Sub-processors (the pile most teams ignore)

Every SaaS in your stack that touches personal data is a sub-processor:

| Provider | Data | Processor type | DPA signed? |
|---|---|---|---|
| AWS / Vercel / Cloudflare | All hosted data | Infra | ✅ |
| Postgres (managed: Neon, RDS) | All PII | Infra | ✅ |
| Stripe / Razorpay | Payment + customer email | Sub-processor | ✅ |
| Brevo / Resend / SendGrid | Email recipient + content | Sub-processor | ✅ |
| Sentry | Error logs (with user.id, NOT user.email per skill `sentry-monitoring`) | Sub-processor | ✅ |
| PostHog / Amplitude | Behavioral + PII if not stripped | Sub-processor | ✅ |
| Clerk / Auth0 | Identity, email, sessions | Sub-processor | ✅ |
| Algolia | Search index of records | Sub-processor | ✅ |

Each needs:
- A signed DPA (most have a self-serve template you e-sign).
- Listed in your privacy notice ("we share data with these providers").
- Reviewed when added/removed (small change to the DPIA).

## Anti-patterns

- **DPIA template copy-pasted with company name swapped** — useless under audit. Has to reflect actual processing.
- **DPIA done before code, never updated** — drifts with reality.
- **"Legitimate interests" claimed without a balancing test** — must be documented per processing.
- **No retention periods** — the most common audit finding.
- **Sub-processors not listed in privacy notice** — Art. 14 violation.
- **Audit logs not protected from tampering** — required for security incident investigation.
- **No subject access portal; manual handling** — slow, error-prone, hard to audit.
- **Cross-border transfers without SCCs / TIA** — Schrems II risk.
- **DPIA covers feature X but new feature Y has different risks** — re-DPIA when scope changes meaningfully.
- **No DPO when one is required** — Art. 37 trigger thresholds matter; check.
- **Treating DPIA as a compliance checkbox** — it's a design tool. The output should change your design.

## Verify it worked

- [ ] DPIA exists in writing, signed off by DPO and legal.
- [ ] Each section has concrete content; no [TBD] left.
- [ ] Data inventory matches the actual schema (sample 5 random PII tables).
- [ ] Data flow diagram matches the actual deployment.
- [ ] Each risk has a documented mitigation; mitigations are real (not aspirational).
- [ ] Each sub-processor has a signed DPA filed; list is current.
- [ ] Privacy notice lists every sub-processor.
- [ ] Retention periods are codified (deletion jobs running) — not just policy text.
- [ ] Subject access requests work end-to-end and complete within 30 days.
- [ ] DPIA review date set; calendar reminder exists.
- [ ] Engineering tickets opened for any gaps the DPIA exposed.
- [ ] If high residual risk, DPA consultation considered (Art. 36).
