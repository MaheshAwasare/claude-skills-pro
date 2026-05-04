---
name: hipaa-audit
description: Audit a SaaS product against HIPAA Security Rule + Privacy Rule — PHI inventory, BAA boundaries, technical safeguards (access, audit, encryption, integrity, transmission), administrative safeguards, breach notification. Use when serving US healthcare customers, processing PHI, or onboarding to a healthcare-adjacent product. Covers Security Rule technical specs in engineering depth.
---

# HIPAA Audit

HIPAA Security Rule + Privacy Rule applies to "Covered Entities" (healthcare providers, plans, clearinghouses) and their "Business Associates" (vendors processing PHI). Most B2B SaaS becomes a Business Associate by signing a BAA. This skill is the engineering-level audit, focused on the Security Rule's technical safeguards.

## When to use

- SaaS company onboarding a US healthcare customer.
- Audit of an existing product's HIPAA posture.
- Mapping engineering controls to BAA obligations.
- Pre-sales — preparing for the customer's HIPAA questionnaire.

## When NOT to use

- Not processing Protected Health Information (PHI) — HIPAA doesn't apply.
- Direct medical practice — full HIPAA Privacy Rule applies; this is broader than this skill covers. Get a HIPAA lawyer.
- Outside the US with no US healthcare customers — not relevant.

## What is PHI?

PHI = Protected Health Information = any individually identifiable health information held or transmitted by a covered entity / business associate.

Identifiable means any of the **18 HIPAA identifiers** in association with health info:
- Name, address (smaller than state), all dates (except year), phone, fax, email, SSN, MRN, health plan ID, account number, certificate/license, vehicle ID, device ID, URL, IP, biometric, photo, any unique ID.

If your product stores any health info connected to any of these → PHI → HIPAA applies.

## The two rules (engineering surface)

| Rule | What it covers |
|---|---|
| **Privacy Rule** | What you can do with PHI, who you can share with, patient rights. |
| **Security Rule** | How to protect electronic PHI (ePHI). Engineering scope. |
| **Breach Notification Rule** | How to handle PHI breaches. |
| **Enforcement Rule** | OCR investigations and penalties. |

Engineering primarily implements Security Rule + Breach Notification.

## Security Rule: three categories of safeguards

### 1. Administrative safeguards

Policy and process, not code. But engineering supports:
- **Security Officer designated** — the named person.
- **Workforce training** — engineers complete annual HIPAA training.
- **Access management** — approval workflow for who gets PHI access.
- **Sanction policy** — what happens if an engineer mishandles PHI.
- **Incident response procedure** — see `write-runbook` + `india-dpdp-act` breach notification patterns.
- **Risk analysis (required) and risk management** — periodic, documented.
- **Contingency plan** — backup, disaster recovery, emergency-mode operations.
- **Business Associate Agreements (BAAs)** with sub-processors.

### 2. Physical safeguards

Mostly handled by your cloud provider's BAA-eligible infrastructure (AWS, GCP, Azure, Vercel, Cloudflare all offer BAAs). You ensure:
- No PHI on workstations / laptops without disk encryption.
- Workstation lock policies enforced via MDM.
- Media disposal for any physical media.
- Facility access controlled (your office; your data center if any).

### 3. Technical safeguards (engineering core)

#### Access control

- **Unique user identification** — every system user has a distinct account. No shared logins.
- **Emergency access procedure** — break-glass admin account, audit-logged, alerted.
- **Automatic logoff** — sessions expire after inactivity (e.g. 15 min for admin, 30 min for users).
- **Encryption and decryption** — see below.

#### Audit controls

> "Implement hardware, software, and/or procedural mechanisms that record and examine activity in information systems that contain or use ePHI."

This is the audit log. **Every read and write of PHI must be logged**, including who, what, when. Audit logs themselves must be tamper-resistant.

```sql
CREATE TABLE audit_log (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  actor_id    text NOT NULL,                      -- user or system that did the thing
  action      text NOT NULL,                      -- 'read_patient', 'update_record', etc.
  resource    text NOT NULL,                      -- e.g. 'patient:p_123'
  metadata    jsonb,                               -- IP, user-agent, route, etc.
  occurred_at timestamptz NOT NULL DEFAULT now()
);

-- Append-only — revoke UPDATE/DELETE for app role
REVOKE UPDATE, DELETE ON audit_log FROM app_user;
```

For larger volumes, push audit logs to immutable storage (S3 Object Lock + KMS, or a managed audit log service).

Retain audit logs for **6 years** (HIPAA's documentation retention).

#### Integrity

- **Mechanism to authenticate ePHI** — that data hasn't been altered or destroyed without detection.
- Implementation: row-level checksums for sensitive tables, or DB triggers that audit-log changes, or full WAL archiving.

#### Person or entity authentication

- **Authentication of users** — passwords + MFA (HHS guidance increasingly expects MFA).
- For machine-to-machine: scoped service accounts with rotation.

#### Transmission security

- **TLS for all PHI in transit** — TLS 1.2+, modern ciphers. No HTTP for PHI.
- **Encryption of email containing PHI** — if you must email PHI, use TLS-enforced email (Brevo with TLS REQUIRED) or end-to-end (S/MIME, PGP).
- **Webhooks containing PHI** — TLS only; signed payloads (HMAC verification).

## Encryption (the practical guide)

HIPAA "addressable" specifications mean *you must implement OR document why not*. Encryption at rest and in transit are essentially mandatory in 2026.

| Data | Standard |
|---|---|
| At rest | AES-256 (RDS encryption, S3 SSE-KMS, full-disk encryption) |
| In transit | TLS 1.2+, prefer 1.3, no SSLv3 / TLS 1.0 / TLS 1.1 |
| Backups | Encrypted (managed services do this) |
| Replicas | Same encryption as primary |
| Logs containing PHI | Encrypted at rest |
| Local dev | NO real PHI on dev machines, ever |

**Key management:** AWS KMS, GCP KMS, or Azure Key Vault. Customer-managed keys (CMK) preferred over service-managed for the audit story.

## BAA (Business Associate Agreement) discipline

Every sub-processor that touches PHI needs a signed BAA:

| Vendor | BAA available? | Notes |
|---|---|---|
| AWS | Yes | Required for HIPAA-eligible services list |
| GCP | Yes | Workspace and certain GCP services |
| Azure | Yes | Many services covered |
| Cloudflare | Yes (Enterprise) | Required for HIPAA |
| Vercel | Yes (Enterprise) | Pro/Hobby tiers don't qualify |
| Stripe | Yes (with restrictions) | PHI not allowed in metadata |
| Twilio | Yes (some products) | SMS/Voice need specific config |
| Sendgrid / Brevo | Yes (with care) | Don't put PHI in email body if avoidable |
| Sentry | Yes | Self-host or specific BAA tier |
| Datadog / New Relic | Yes (with BAA tier) | |

If a vendor doesn't offer a BAA → it cannot process PHI on your behalf. Period.

## Breach notification

A breach = unauthorized acquisition, access, use, or disclosure of unsecured PHI.

- **< 500 individuals affected**: notify HHS annually. Notify each affected individual within 60 days.
- **≥ 500 individuals**: notify HHS *and* the media within 60 days. (Yes, public.)
- **Encrypted data lost**: usually NOT a breach (encryption safe-harbor) — provided keys weren't also compromised.

Engineering's role:
- **Detection** — alerting on anomalous access patterns.
- **Investigation** — audit logs must support reconstructing what was accessed.
- **Containment** — kill compromised credentials, rotate keys, block IPs.
- **Reporting hand-off** — investigation report to legal/Privacy Officer, who handles HHS notification.

## Common engineering gaps in HIPAA audits

1. **Audit log gaps** — reads not logged, only writes; or admin actions not logged.
2. **Local dev environments with real PHI** — must use synthetic data; deidentified data with formal de-id process.
3. **Logs containing PHI** — application logs, error tracking (Sentry), Cloudwatch — all flow PHI to non-BAA'd places.
4. **Email of PHI** — accidentally putting PHI in subject line or body without encrypted email.
5. **No de-identification process** — claiming data is "de-identified" without applying the Safe Harbor or Expert Determination method.
6. **Backups not encrypted** — managed services do this; self-hosted often doesn't.
7. **No MFA on admin** — increasingly considered required.
8. **No incident response runbook** — `write-runbook` covers shape; ensure breach scenario specifically.
9. **Sub-processors without BAAs** — adding a tool without checking.
10. **Production access to PHI by engineering on-demand without per-access logging** — must be JIT, audited.

## Anti-patterns

- **"We're encrypted, we're HIPAA-compliant"** — encryption is one safeguard. There are dozens.
- **PHI in client-side code, browser localStorage, or non-PHI logs** — leaks across scopes.
- **Treating BAA signing as the end of compliance** — it's the start.
- **Production data in test environments** — biggest source of accidental disclosures.
- **No automated detection of unusual access patterns** — breaches go undetected for months.
- **Audit logs writable by app accounts** — not tamper-resistant.
- **PHI in URL query params** — logged everywhere (web servers, CDN, browser history).
- **"Encryption in transit" with TLS 1.0** — not encryption-as-required; configure modern only.
- **De-identified data still has direct identifiers in joinable form** — re-identification trivial; not actually de-identified.
- **Annual security training skipped or perfunctory** — required; not optional.

## Verify it worked

- [ ] PHI inventory: every table and field that contains PHI is documented.
- [ ] BAA on file with every sub-processor that may touch PHI.
- [ ] All ePHI is encrypted at rest (AES-256) and in transit (TLS 1.2+).
- [ ] MFA enforced for all users with PHI access; admin and service accounts especially.
- [ ] Audit log captures every read and write of PHI; logs are tamper-resistant; retained 6 years.
- [ ] Anomaly detection on PHI access patterns; alerts to security on-call.
- [ ] Automatic session logoff configured; verified on staging.
- [ ] Emergency access (break-glass) is alerted on every use.
- [ ] No PHI in application logs (sample 100 random log lines to verify).
- [ ] No PHI in error tracking (Sentry user.id only, no email/name).
- [ ] No real PHI on developer machines; synthetic data fixtures used.
- [ ] Incident response runbook covers HIPAA breach steps.
- [ ] Annual security training completed by all workforce members.
- [ ] Risk analysis documented; updated yearly.
