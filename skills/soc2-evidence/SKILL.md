---
name: soc2-evidence
description: Map SOC 2 controls to actual code and CI artifacts an auditor can verify — access reviews, change management, vulnerability scanning, encryption, monitoring, incident response. Use when starting SOC 2 Type 1/Type 2 prep, working with Vanta/Drata/Secureframe, or responding to a customer security questionnaire that asks for SOC 2 evidence.
---

# SOC 2 Evidence (Engineering View)

SOC 2 isn't a prescriptive standard — auditors evaluate whether your controls *as designed* and *as operating* meet the Trust Services Criteria. The art is mapping abstract criteria (CC6.1, CC7.4, etc.) to concrete code/CI artifacts. This skill is that mapping.

## When to use

- Starting SOC 2 Type 1 or Type 2 prep (typically 6–12 months out from audit).
- Working with Vanta / Drata / Secureframe / Sprinto and they ask "what evidence do you have for control X?"
- Sales-driven: customer asks "are you SOC 2 compliant?" and you need to articulate posture.
- Pre-audit gap assessment.

## When NOT to use

- Looking for legal advice — work with an auditor.
- ISO 27001 — overlaps but distinct framework; mapping is different.
- Pure compliance "checkbox" mode — SOC 2 done well changes how engineering operates.

## Trust Services Criteria you'll actually face

| Category | Required for most | What it covers |
|---|---|---|
| **Common Criteria (CC)** | Yes — required | Security; the bulk of controls |
| **Availability (A)** | Common | Uptime, DR, capacity |
| **Confidentiality (C)** | Common | Encryption, access |
| **Processing Integrity (PI)** | Less common | Data accuracy, completeness |
| **Privacy (P)** | Common for B2C | Notice, consent, data subject rights |

Most SaaS vendors do CC + A + C. PI is often skipped (hard, less audited). Privacy added if processing consumer PII heavily.

## The mapping (sample controls → engineering artifacts)

### CC6.1: Logical access controls

> "The entity implements logical access security software, infrastructure, and architectures over protected information assets."

| Evidence auditor wants | Where in your stack |
|---|---|
| Authentication enforces MFA | Clerk/Okta config screenshot + SSO policy |
| Password complexity policy | Clerk password policy / Okta config |
| Auto-disable inactive accounts | Cron job in Vanta/Drata, or HR-driven offboarding |
| Privileged access reviewed quarterly | Documented review meeting + ticket per user |
| Service accounts have rotation | KMS rotation policy + last rotation date |

### CC6.6: Encryption in transit and at rest

| Evidence | Where |
|---|---|
| TLS 1.2+ enforced on all endpoints | nginx/cloudfront/cloudflare config; SSL Labs scan |
| Database encryption at rest | RDS / Cloud SQL config screenshot |
| S3/GCS bucket encryption | Bucket policy; AWS Config rule |
| Backup encryption | Automated backup config |
| Customer-managed keys (CMK) where applicable | KMS key rotation, separation of duty |

### CC7.1: System monitoring

| Evidence | Where |
|---|---|
| Centralized logging | CloudWatch / Datadog / Loki |
| Alerts on critical events | Sentry / PagerDuty / OpsGenie config |
| Log retention ≥ 1 year | Storage class config |
| Anomaly detection | GuardDuty, custom alerts on auth failures |
| Audit logs of admin actions | CloudTrail + app-level audit log |

### CC7.4: Incident response

| Evidence | Where |
|---|---|
| Documented incident response procedure | `docs/runbooks/incident-response.md` |
| Incident tickets in tracker | Linear / Jira project for incidents |
| Post-mortem for severity 1/2 incidents | Filed within 5 business days |
| On-call rotation | PagerDuty schedule |

### CC8.1: Change management

> "The entity authorizes, designs, develops or acquires, configures, documents, tests, approves, and implements changes to infrastructure, data, software, and procedures to meet its objectives."

| Evidence | Where |
|---|---|
| All code changes via PR | GitHub branch protection rules |
| Required reviewers | CODEOWNERS file; protected branch settings |
| CI runs tests before merge | GitHub Actions workflow + required checks |
| Production deploys are auditable | CD logs, deploy notifications in Slack |
| Database migrations reviewed | PR labels, migration approval workflow |
| Emergency change procedure | Runbook documenting break-glass deploy |

### CC9.1: Risk assessment

| Evidence | Where |
|---|---|
| Annual risk assessment | Spreadsheet / Vanta module |
| Vendor risk reviews | Vendor list + assessment per vendor |
| Penetration test (annual) | Third-party report |
| Vulnerability scanning | Dependabot / Snyk / GitHub Advanced Security |

### A1.2: Availability

| Evidence | Where |
|---|---|
| SLA / SLO documented | Public uptime page or internal SLO doc |
| Backups run successfully | Automated backup logs |
| DR test (annual) | Restore-from-backup test report |
| Capacity monitoring | Dashboards showing headroom |

## The artifact list (what you actually produce)

Across all controls, these are the buckets:

1. **Configurations** — screenshots or as-code definitions (Terraform, IaC).
2. **Logs and reports** — exported from systems, retained.
3. **Tickets** — change requests, access requests, incidents in Linear/Jira.
4. **Policies** — written documents (security policy, data retention, etc.).
5. **Training records** — annual security training completion.
6. **Reviews** — periodic access reviews, vendor reviews.

GRC tools (Vanta, Drata, Secureframe) automate evidence collection by hooking into AWS, GitHub, Okta, etc., and pulling configurations / running checks continuously.

## Engineering-side actions to make audit easy

### 1. Branch protection on `main`

```yaml
# Repo settings → Branches → main
require_pull_request_reviews: true
required_approving_review_count: 1
dismiss_stale_reviews: true
require_code_owner_reviews: true
require_status_checks_to_pass: true
required_status_checks: [ci, security-scan]
require_signed_commits: true
require_linear_history: true
include_administrators: true
```

This single config covers a chunk of CC8.1.

### 2. CODEOWNERS file

```
# .github/CODEOWNERS
*                          @platform-team
/infra/                    @platform-team @security-team
/migrations/               @platform-team @data-team
/payments/                 @payments-team @security-team
```

Required reviewers for sensitive paths. Auditors love this.

### 3. Mandatory CI security scan

```yaml
# .github/workflows/security.yml
on: [pull_request]
jobs:
  scan:
    steps:
      - uses: github/codeql-action/init@v3
      - uses: github/codeql-action/analyze@v3
      - uses: aquasecurity/trivy-action@0.20.0     # container scan
      - uses: snyk/actions/node@master              # dep scan
```

Required check on `main`. Evidence of "automated vulnerability scanning."

### 4. Centralized logging with retention

CloudWatch log groups with `retention_in_days = 365` (or longer). Application logs flow to CloudWatch / Loki / Datadog. CloudTrail captures all AWS API calls; retain ≥ 1y.

### 5. Audit log table for admin actions

Every admin action in your app writes to an audit_log table:
```ts
await db.auditLog.create({ data: {
  actor: adminUser.id,
  action: "view_user_pii",
  resource: targetUserId,
  context: { ip, userAgent, route },
}});
```

Auditors will sample this. Make it real.

### 6. Quarterly access review

Cron a script that exports user list with roles → manager reviews → ticket attached as evidence. Tools like Vanta automate the workflow.

### 7. Annual penetration test

Hire a reputable firm (HackerOne, Bishop Fox, Cobalt). Get the report. Track findings to closure.

## Type 1 vs Type 2

- **Type 1**: design of controls at a point in time. Shorter; cheaper. Good first audit.
- **Type 2**: design + operating effectiveness over a period (typically 6 or 12 months). What customers actually want.

Most SaaS targets Type 2 with a 12-month observation window after a Type 1.

## GRC tool tips

Vanta / Drata / Secureframe / Sprinto automate ~70% of evidence collection. They'll flag failing checks daily. Worth the cost; saves the engineering team from manual evidence gathering.

But:
- Tools detect what's *connected*. They won't catch the third-party service that's not integrated.
- "Vanta says we're 99% compliant" ≠ "auditor will pass us 99% compliant." Tools are heuristics.
- Engineering still has to *build* the controls; tools just monitor them.

## Anti-patterns

- **Treating SOC 2 as a checkbox** — controls have to actually operate. "We have a policy" without enforcement fails Type 2.
- **Buying a GRC tool first, building controls later** — tool monitors emptiness.
- **Branch protection skipped for "speed"** — change management is the most-audited control area.
- **No CODEOWNERS** — auditor question: "who reviews production-impacting code?"
- **Audit log writable by application** — must be tamper-resistant.
- **Vendor list out of date** — listing only old vendors, missing the 5 added this year.
- **No backup restore test** — backups exist but never tested. Auditor asks "when did you last restore?"
- **Pen test findings ignored** — needs a tracked remediation path.
- **Access review = "I looked at the list once"** — must be documented; outcome (approve/revoke per user) recorded.
- **Generic incident response policy nobody's read** — auditor asks "walk me through what would happen for incident X."

## Verify it worked

- [ ] Controls mapped to evidence in a spreadsheet (or GRC tool).
- [ ] Branch protection active on default branch; bypasses require approval.
- [ ] CODEOWNERS file enforces reviewers on sensitive paths.
- [ ] CI runs security scans (CodeQL, Snyk, Trivy) and blocks merges on critical findings.
- [ ] All admin actions in app are audit-logged; sample query produces sensible results.
- [ ] Access reviews documented quarterly with manager sign-off.
- [ ] Backup restore tested annually; report retained.
- [ ] Penetration test report from current year on file; findings tracked to closure.
- [ ] Incident response runbook exists; on-call has been trained.
- [ ] Encryption at rest and in transit verified by config and external scan.
- [ ] No production access from personal laptops without MDM-enforced disk encryption.
- [ ] All workforce members completed annual security training.
- [ ] Vendor list current; BAAs / DPAs / SOC 2 reports collected from material vendors.
