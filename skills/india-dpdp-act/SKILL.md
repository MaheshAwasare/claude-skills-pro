---
name: india-dpdp-act
description: Implement India's Digital Personal Data Protection Act 2023 — consent management, data fiduciary obligations, breach notification, data principal rights, cross-border transfer rules, and the architectural choices for Indian data residency. Use when building or auditing a product serving Indian users; few Claude resources cover this and the penalties for non-compliance are real (₹250 crore max).
---

# India DPDP Act 2023

The Digital Personal Data Protection Act 2023 (DPDP Act) is India's first comprehensive privacy law, modelled loosely on GDPR but with important differences. Penalties go up to ₹250 crore (~$30M USD) per violation. This skill is the engineering checklist.

## When to use

- Building a product that processes data of users in India (regardless of where the company is based).
- Pre-launch privacy review for an Indian-facing app.
- Audit of an existing product against DPDP requirements.
- Setting up consent flows, data principal request handling, breach response.

## When NOT to use

- Pure B2B with no consumer PII (less stringent — but business contact data is still PII).
- No Indian users and no plans to expand — defer.
- Looking for legal advice — this is engineering scope; lawyer-review is mandatory before launch.

## The actors and definitions

| Term | What it means |
|---|---|
| **Data Principal** | The individual whose data is processed. (GDPR equivalent: data subject.) |
| **Data Fiduciary** | The entity deciding how/why data is processed. (GDPR: data controller.) |
| **Data Processor** | Third party processing on behalf of fiduciary. (GDPR: same.) |
| **Significant Data Fiduciary (SDF)** | Designated by central government based on volume/sensitivity. Higher obligations. |
| **Consent Manager** | A registered intermediary that helps data principals manage consents (uniquely Indian concept). |

The DPDP Rules (operationalizing the Act) were notified in 2025. Some specifics will continue to evolve — keep tracking the Ministry of Electronics and IT (MeitY) updates.

## Core obligations (engineering view)

### 1. Lawful basis = consent (mostly)

DPDP recognizes only two lawful bases:
- **Consent** — explicit, informed, unconditional, free, specific.
- **Legitimate uses** (limited list — employment, govt benefits, medical emergencies, etc.).

Unlike GDPR, **"legitimate interest" is NOT a basis**. You can't process data because it's "useful for the business."

**Implementation:** every data collection point needs a consent gate. Generic "I agree to the privacy policy" is not enough — must be specific per purpose.

```html
<!-- BAD -->
<input type="checkbox" /> I agree to the Terms and Privacy Policy

<!-- GOOD — specific, granular -->
<fieldset>
  <legend>I consent to:</legend>
  <label><input type="checkbox" name="consent_account" required /> Creating my account using my email and phone</label>
  <label><input type="checkbox" name="consent_marketing" /> Receiving marketing emails about new features</label>
  <label><input type="checkbox" name="consent_analytics" /> Anonymous analytics to improve the product</label>
</fieldset>
```

Each consent must be:
- **Logged** (timestamp, IP, user agent, exact text shown).
- **Withdrawable** as easily as it was given.
- **Specific** to a purpose; one consent ≠ all consents.

### 2. Notice requirements

Before consent, you must show, in clear language:
- What data will be collected.
- What it's used for.
- How to withdraw consent.
- How to file a complaint with the Data Protection Board.
- Contact details of the Data Protection Officer (mandatory for SDFs; recommended for all).

The notice must be available in **English and any of the 22 official Indian languages** specified in Schedule 8 of the Constitution. In practice: English + Hindi minimum; add regional based on user base.

### 3. Data minimization and purpose limitation

Collect only what you need for the stated purpose. Don't collect "for future use."

```ts
// BAD — collecting too much for sign-up
{ email, phone, address, dob, occupation, income, pan_number }

// GOOD — sign-up needs minimum; collect more later when needed
{ email, phone }                        // sign-up
// On first KYC need:
{ pan_number, address }                 // separate consent
```

### 4. Storage and retention limits

Data must be deleted when:
- The purpose for which it was collected is fulfilled (and no other lawful retention applies).
- Consent is withdrawn (and no statutory retention requires keeping it).
- Data Principal requests erasure.

**Implementation:** every table with PII needs a retention policy. Build deletion jobs from day 1.

```sql
-- Example schema with retention metadata
CREATE TABLE user_uploads (
  id              uuid PRIMARY KEY,
  user_id         uuid NOT NULL,
  storage_path    text NOT NULL,
  consent_purpose text NOT NULL,                              -- 'profile_avatar', 'kyc', etc.
  retention_until timestamptz NOT NULL,                       -- pre-computed
  created_at      timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX uploads_retention_idx ON user_uploads (retention_until);
```

Nightly job: `DELETE WHERE retention_until < now()`. Audit-log each deletion.

### 5. Data principal rights (engineering interfaces)

The user can request:

| Right | What you must do |
|---|---|
| **Access** | Provide a copy of their data within reasonable time |
| **Correction** | Update incorrect data |
| **Erasure** | Delete data (subject to legal retention exceptions) |
| **Grievance** | A way to complain; you must respond |
| **Nominate** | Designate someone to exercise rights on their behalf (e.g. on death) |

You need **a self-service portal** or a documented manual process. Self-service is better — auditable, low-overhead.

```
GET /api/me/data-export        → produces JSON dump of user's data
POST /api/me/correction         → submits a correction request
POST /api/me/deletion           → initiates deletion flow (with confirmation)
GET /api/me/consents            → lists active consents
POST /api/me/consents/<id>/withdraw → withdraws a consent
```

### 6. Children's data

For users under 18, you need **verifiable parental consent**. Behavioural tracking and targeted advertising for children are *prohibited*. Most products simply ban under-18 sign-ups for now — verifying parent consent at scale is operationally hard.

If you allow under-18: design specific flows (parent-child account linkage, OTP to parent's phone for consent, etc.). Don't fudge it.

### 7. Cross-border transfers

Transfer to "any country except those notified by the Central Government" is allowed *unless* a sectoral law (RBI for fintech, IRDAI for insurance, MoH for health) restricts it.

In practice:
- **Fintech (RBI):** payment data must be stored only in India. (Razorpay/payment gateway handles this for you in their domain.)
- **Health (MoH):** health records have stricter rules; consult.
- **General SaaS:** can use US/EU cloud regions, but a future MeitY notification could restrict. **Hedge by running in `ap-south-1` (Mumbai) by default** for Indian user data.

### 8. Breach notification

A "personal data breach" must be reported to the Data Protection Board (DPB) and to affected data principals **without delay**. DPDP Rules allow up to 72 hours to the DPB but expect "as soon as practicable" to users.

**Implementation:**
- **Detection:** WAF / IDS / anomaly alerts that page on-call.
- **Response runbook:** see `write-runbook` skill. Should include the breach notification template.
- **Audit log:** what was accessed, when, by whom — required for the DPB submission.

### 9. Significant Data Fiduciary (SDF) extras

If notified as an SDF (typically high user volume / sensitive data):
- Appoint a **Data Protection Officer** based in India.
- Conduct **Data Protection Impact Assessment (DPIA)** before high-risk processing.
- Appoint an **independent auditor**.
- File **periodic compliance reports** with DPB.

Most startups won't be SDFs initially, but the threshold is set by government notification; can change. Build the discipline early.

## Architecture choices that make compliance easier

| Choice | Why |
|---|---|
| **Postgres in `ap-south-1` (Mumbai)** | Default residency; reduces cross-border concerns. |
| **Tagged data fields by `consent_purpose`** | Enables purpose-limited queries and selective deletion. |
| **Audit log of data access** (who read what, when) | Required for breach investigations and DPB queries. |
| **Soft delete + hard delete jobs** | Soft delete for short-window recovery; scheduled hard-delete enforces retention. |
| **Per-purpose data partitioning** | Marketing data in one set of tables; account data in another. Different retention, different deletion. |
| **No PII in logs** | Mask emails, phones, IDs. Use OTel attributes for trace correlation, not PII. |
| **No PII in error tracking (Sentry)** | See `sentry-monitoring` — user.id only, never email/name. |
| **Separate DB for analytics** | Aggregated, anonymized; easier to argue retention beyond the source data. |

## Anti-patterns

- **One "I agree to terms" checkbox** — not consent for DPDP. Granular per-purpose.
- **Consent bundled with service ("you must consent to use the app")** — invalid; must be optional for non-essential purposes.
- **Consent not logged** — when a user disputes, you have no proof.
- **Keeping data "in case it's useful later"** — purpose limitation violation.
- **No data export / deletion endpoint** — manual process at human speed; doesn't scale and DPB will notice.
- **PII in error logs / traces / analytics** — leaks across multiple boundaries.
- **No DPO or unclear contact for grievances** — required notice item.
- **US-only cloud region for Indian users without sectoral exemption** — risk if MeitY restricts later.
- **Treating DPDP as "GDPR but cheaper"** — DPDP has no legitimate-interest basis. Audit your "legitimate interest" claims.
- **No breach detection** — you can't notify what you don't know about.
- **Letting third-party SDKs collect data without explicit listing in notice** — every SDK that touches user data is a sub-processor; must be disclosed.

## Verify it worked

- [ ] Privacy notice exists in English + Hindi (and other regional languages for major user bases).
- [ ] Every data collection has a specific, granular consent checkbox.
- [ ] Consent log table records: user_id, purpose, consent_text_shown, timestamp, ip, user_agent.
- [ ] Withdrawal of consent is one click and triggers data handling change immediately.
- [ ] Self-service endpoints: data export, correction request, deletion request.
- [ ] Retention policy documented per data type; scheduled deletion job runs and is audited.
- [ ] No PII in logs / traces / Sentry / analytics events (audit by sampling 100 random log lines).
- [ ] Indian user data is stored in ap-south-1 (or equivalent India region) by default.
- [ ] Breach response runbook exists; on-call is trained.
- [ ] DPO contact and grievance officer contact published in notice.
- [ ] Lawyer has reviewed the privacy notice and consent flows.
- [ ] If SDF: DPIA documented, independent auditor engaged, periodic reports planned.
