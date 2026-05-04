---
name: brevo-email
description: Send transactional and marketing email via Brevo (formerly Sendinblue) — templates, lists, contact attributes, webhooks, bounce/complaint handling, and the inbox-deliverability setup (SPF, DKIM, DMARC, dedicated IP) that determines whether your mail lands in inbox or spam. Use when adding email to a Node, Python, or Go backend, or migrating from SendGrid/Postmark for cost.
---

# Brevo (Sendinblue) Email

Brevo is the cheapest credible alternative to SendGrid/Postmark — generous free tier (300/day), pay-as-you-go above. The trade-off is a thinner SDK and fewer deliverability tools, so you compensate with discipline. This skill is the disciplined path.

## When to use

- Transactional email (signup, password reset, receipts, invoices) on a budget.
- Lists + campaigns alongside transactional, in one platform.
- Migrating off SendGrid / Mailgun / Postmark for cost.
- Indian senders — Brevo's EU base means simpler DPDP/GDPR posture than US providers.

## When NOT to use

- High-volume (>1M/month) where Postmark or AWS SES becomes cheaper.
- You need built-in IP warmup automation — Brevo's is manual.
- Heavy sub-second SLAs — Brevo's API is fine, but Postmark is faster.

## Core concepts

| Concept | What it is | Gotcha |
|---|---|---|
| **Transactional** | One-to-one (receipts, OTP) | Different sending pool from marketing |
| **Campaign / Marketing** | One-to-many (newsletters) | Lower deliverability priority |
| **Template** | Server-stored HTML with variables | Versioned by ID — keep IDs in your code |
| **Contact** | Person row with attributes & list memberships | Updated on every send by default; opt out |
| **Sender** | Verified `from:` address or domain | Domain auth is mandatory for production |
| **Webhook** | Brevo → your server delivery events | Event types: `delivered`, `opened`, `click`, `bounce`, `spam`, `unsubscribe` |

## Sending transactional email (Node)

```js
// apps/api/src/email/brevo.ts
import * as brevo from "@getbrevo/brevo";

const api = new brevo.TransactionalEmailsApi();
api.setApiKey(brevo.TransactionalEmailsApiApiKeys.apiKey, process.env.BREVO_API_KEY!);

export async function sendOTP(to: { email: string; name: string }, code: string) {
  const email = new brevo.SendSmtpEmail();
  email.templateId = 7;                                  // created in Brevo dashboard
  email.to = [{ email: to.email, name: to.name }];
  email.params = { code, ttlMinutes: 10 };               // injected into template
  email.headers = { "X-Mailer": "acme-otp-v1" };
  email.tags = ["otp", "auth"];                          // searchable in dashboard

  const res = await api.sendTransacEmail(email);
  return res.body.messageId;                             // store for webhook correlation
}
```

Always pick **template + params** over building HTML in code — non-engineers can fix typos in the dashboard without a deploy, and you get a per-template open/click breakdown.

## Why a template ID belongs in code, not env

Anti-pattern most teams hit: putting `BREVO_OTP_TEMPLATE_ID=7` in `.env`. Then the dev forgets to set it on staging and OTPs blast a marketing template. **Hard-code the IDs in a single `email/templates.ts` and review changes in PR**:

```ts
// apps/api/src/email/templates.ts
export const TEMPLATES = {
  otp: 7,
  passwordReset: 12,
  invoicePaid: 18,
  trialExpiring: 22,
} as const;
```

If you have separate prod/staging Brevo accounts, sync the IDs across them (Brevo lets you export/import templates).

## Sending in bulk (one API call, many recipients)

```ts
email.messageVersions = users.map(u => ({
  to: [{ email: u.email, name: u.name }],
  params: { firstName: u.firstName, magicLink: signFor(u.id) },
}));
// Up to 1000 versions per call — much faster than 1000 calls.
```

## Webhooks (delivery, bounces, complaints)

Configure: Dashboard → Settings → Transactional → Email → Settings → Webhooks → add URL.

Events you must handle:

```ts
// apps/api/src/routes/brevo-webhook.ts
import { Router } from "express";

export const router = Router();

router.post("/brevo/webhook", async (req, res) => {
  // Brevo signs webhooks via shared-secret header; verify in production.
  if (req.header("X-Mailin-Tag") !== process.env.BREVO_WEBHOOK_TOKEN) {
    return res.status(401).end();
  }

  const event = req.body; // { event, email, message-id, ts, ... }

  switch (event.event) {
    case "delivered":
      await db.emailLog.update({ where: { messageId: event["message-id"] }, data: { status: "delivered" } });
      break;
    case "hard_bounce":
    case "blocked":
      await suppressEmail(event.email, event.event);     // never send to this address again
      break;
    case "spam":
      await suppressEmail(event.email, "complaint");
      await alertOps(`Spam complaint: ${event.email}`);  // these hurt sender reputation
      break;
    case "unsubscribe":
      await db.user.update({ where: { email: event.email }, data: { marketingOptIn: false } });
      break;
  }

  res.status(200).end();
});
```

**Suppression list discipline:** maintain your *own* suppression table; don't rely on Brevo's. Reasons:
1. You'll switch ESP someday — your suppression list is portable.
2. Brevo only suppresses for that account; if you have multi-account/region, it doesn't carry.

## Deliverability setup (the part nobody documents)

Inbox vs spam is 80% sender reputation, 20% content. Sender reputation comes from three DNS records you must set on your sending domain (`mail.example.com`, NOT the apex `example.com`):

### 1. SPF
```
mail.example.com.  TXT  "v=spf1 include:spf.brevo.com ~all"
```

### 2. DKIM
Brevo provides two `mail._domainkey.<your-domain>` CNAMEs in dashboard → set them.

### 3. DMARC (start permissive, tighten later)
```
_dmarc.example.com.  TXT  "v=DMARC1; p=none; rua=mailto:dmarc@example.com; pct=100"
```
After 2 weeks of clean reports, raise to `p=quarantine`, then `p=reject`.

### 4. Use a subdomain for sending
Send from `mail.example.com` or `notifications.example.com`, not the apex. Reasons:
- Apex domain reputation is shared with marketing site, employee Gmail, etc.
- A campaign that gets flagged shouldn't poison support email.

## Marketing lists & contacts

```ts
import * as brevo from "@getbrevo/brevo";
const contacts = new brevo.ContactsApi();
contacts.setApiKey(brevo.ContactsApiApiKeys.apiKey, process.env.BREVO_API_KEY!);

await contacts.createContact({
  email: "user@example.com",
  attributes: { FIRSTNAME: "Mahesh", PLAN: "pro", SIGNUP_AT: new Date().toISOString() },
  listIds: [3],         // "Pro plan customers"
  updateEnabled: true,  // upsert, not duplicate
});
```

For onboarding/lifecycle automation, use Brevo's "Automation" canvas with `event tracker` — fires on attribute changes from the API.

## Anti-patterns

- **Sending from apex domain** — see deliverability section. Use a subdomain.
- **No DKIM** — modern providers (Gmail/Apple Mail) increasingly reject unsigned mail. DKIM is mandatory.
- **Hardcoding HTML in code instead of templates** — every typo fix becomes a deploy; non-eng can't help.
- **No suppression table on your side** — you'll lose it on ESP migration.
- **Treating bounces as transient** — `hard_bounce` and `blocked` are permanent. Sending again hurts your reputation. Suppress immediately.
- **Sending marketing through transactional template ID** — Brevo's pricing and deliverability differ. Don't smuggle.
- **One Brevo account for prod + staging** — staging will leak emails to real users. Use separate accounts (free tier is enough for staging).
- **Ignoring `complained` / `spam` events** — even one complaint per thousand hurts sender score. Pause sending to that address forever.
- **No DMARC, or starting at `p=reject`** — start `p=none`, observe, tighten. Jumping to `reject` will silently break legit mail.

## Verify it worked

- [ ] `dig TXT mail.example.com` shows SPF record including `spf.brevo.com`.
- [ ] `dig CNAME mail._domainkey.example.com` resolves to a Brevo selector.
- [ ] `dig TXT _dmarc.example.com` returns a v=DMARC1 record.
- [ ] Sending a test email and viewing source in Gmail shows `SPF: PASS`, `DKIM: PASS`, `DMARC: PASS`.
- [ ] `mail-tester.com` score on a real send is ≥ 9/10.
- [ ] Webhook handler dedups by `message-id` (replay test).
- [ ] Hard bounce → suppression row written → next send to that address is blocked at app layer.
- [ ] Template IDs are in source code, not env vars.
- [ ] Production and staging use separate Brevo accounts (different `BREVO_API_KEY`).
- [ ] Sending domain is a subdomain, not the apex.
