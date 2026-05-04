---
name: pci-dss-checklist
description: PCI DSS 4.0 engineering checklist — scope reduction first (what you hand to Stripe/Razorpay vs. keep), SAQ tier selection, the controls you actually own, network segmentation, and how to keep PAN data out of your systems. Use when a customer asks "are you PCI compliant?" or before designing payment integration.
---

# PCI DSS 4.0 (Engineering Checklist)

PCI DSS is the data security standard for payment card data. The single best PCI strategy is **scope reduction**: hand cardholder data to a PCI-compliant processor and never touch it. This skill is the playbook to do that — and the controls you still own.

## When to use

- Building or auditing payment integration.
- Customer / acquirer asks for SAQ (Self-Assessment Questionnaire).
- Considering whether to use Stripe Elements / Razorpay Checkout vs. a custom card form.
- Sales-driven "are you PCI compliant?" question.

## When NOT to use

- You don't accept card payments at all — not in scope.
- High-volume merchant (Level 1: > 6M transactions/year) — engage a QSA (Qualified Security Assessor); this skill is too high-level.
- Issuing cards / processor / gateway roles — entirely different scope.

## Scope reduction is the only sane strategy

PCI DSS applies to "Cardholder Data Environment" (CDE) — anywhere PAN (Primary Account Number) is processed, transmitted, or stored.

**Goal: keep your CDE tiny or non-existent.**

| What you do | Your CDE |
|---|---|
| Embed Stripe Elements / Razorpay Checkout (iframe) | Almost zero — PAN never touches your servers |
| Accept PAN in your form, post to processor | Big CDE — your front-end + backend + logs in scope |
| Store last4 of card | Small — last4 isn't PAN; less restrictive |
| Tokenized card with processor | Small — token isn't card data |
| Store full PAN | Maximum scope; almost nobody should do this |

The right shape for almost every SaaS:
1. **Browser**: Stripe Elements / Razorpay Checkout (iframe). Customer types card into the iframe; PAN goes directly to processor; you get a token.
2. **Backend**: stores token (e.g. `pm_xxx`), customer ID, last4, expiry, brand. Never PAN.
3. **Charges**: server tells processor "charge this token for $X."

This pattern keeps you on **SAQ A** (the lightest tier).

## SAQ tiers (which one applies)

| Tier | When |
|---|---|
| **SAQ A** | E-commerce; outsourced PAN entry/handling entirely (Stripe Elements, hosted checkout). Minimal controls. |
| **SAQ A-EP** | Your site partially affects PAN entry (e.g. JS that wraps the iframe). More controls. |
| **SAQ D** | You handle PAN at all. Heavy controls (200+ requirements). |

The leap from A-EP to D is huge. Stay on A or A-EP.

## What "SAQ A" requires you to do

Even SAQ A — the easiest tier — has real engineering implications:

1. **Maintain firewall config** — restrict inbound to your servers; only allow what's needed.
2. **No vendor defaults** — change default passwords, disable unused services.
3. **Protect stored cardholder data (the little you have)** — last4 isn't sensitive but treat metadata carefully.
4. **Encrypt transmission of cardholder data** — TLS 1.2+ on all payment-related routes.
5. **Antivirus on systems that handle the integration** (mostly N/A for cloud-only setups; documented).
6. **Develop and maintain secure systems** — patch CVEs, secure SDLC.
7. **Restrict access to cardholder data by business need-to-know** — RBAC.
8. **Identify and authenticate access** — unique IDs, MFA where applicable.
9. **Restrict physical access** — handled by your cloud provider.
10. **Track and monitor all access** — audit logs.
11. **Test security systems regularly** — quarterly external scans, annual pen test.
12. **Maintain information security policy** — written, reviewed annually.

These map to dozens of sub-requirements in the SAQ document. Most are "yes/no, attach evidence."

## Engineering controls (the practical list)

### 1. Use processor-hosted card entry

Stripe Elements / Razorpay Checkout. **Don't build your own card form.** Even if you encrypt and pass through, you've expanded scope to A-EP at minimum.

```html
<!-- Right way -->
<div id="card-element"></div>
<script>
  const stripe = Stripe(publishableKey);
  const elements = stripe.elements();
  const card = elements.create('card');
  card.mount('#card-element');
  // ...
</script>
```

The iframe means PAN never enters your DOM. Your servers never see it.

### 2. Store only tokens

Schema:
```sql
CREATE TABLE payment_methods (
  id              uuid PRIMARY KEY,
  user_id         uuid NOT NULL REFERENCES users(id),
  provider        text NOT NULL,                                -- 'stripe', 'razorpay'
  provider_token  text NOT NULL,                                -- e.g. 'pm_1abc'
  last4           text NOT NULL,                                -- ok to store
  brand           text NOT NULL,                                -- visa, mastercard, etc.
  exp_month       smallint NOT NULL,
  exp_year        smallint NOT NULL,
  created_at      timestamptz NOT NULL DEFAULT now()
);
```

No PAN. No CVV (NEVER store CVV — explicit PCI prohibition). Token + last4 + brand is enough for UX.

### 3. Logging discipline

PAN must NEVER appear in logs:
```ts
// BAD
log.info("processing payment", { card: req.body.card_number });

// GOOD — server doesn't see card_number; only token
log.info("processing payment", { token: req.body.payment_method_id });
```

Audit your log pipeline. If you accept any card-like field in any request body and it's logged, you've expanded scope.

### 4. Network segmentation

If your CDE is non-existent (SAQ A path), this is moot. If you handle anything PAN-adjacent:
- Separate VPC / subnet for payment-handling services.
- Inbound restricted to ELB; ELB restricted to specific IPs.
- Outbound restricted to processor IPs.

### 5. Encryption everywhere

- TLS 1.2+ on all customer-facing endpoints.
- HSTS header on payment pages.
- TLS 1.2+ to processor APIs (default everywhere).
- DB encryption at rest (managed services do this).
- Backups encrypted.

### 6. Access control

- Production access to payment-handling services restricted to a small set of engineers.
- MFA enforced.
- Just-in-time access with audit log.
- Quarterly access review.

### 7. Vulnerability management

- Quarterly external network scans by an ASV (Approved Scanning Vendor) — required for SAQ A and above.
- Internal vulnerability scanning continuous (Snyk / Dependabot / Trivy).
- Patch critical vulns within 30 days; high within 90.
- Annual penetration test (covers payment flow).

### 8. Logging and monitoring

- Audit log of who accessed payment-related data.
- Failed authentication alerts.
- Anomalous payment volume alerts.
- Logs retained 1 year (PCI 10.7).

### 9. Secure development

- Security training annually for engineers in scope.
- Code review required for payment-handling code.
- Threat modeling for new payment flows (see `threat-modeling-stride`).
- OWASP Top 10 review on payment-related code.

### 10. Incident response

- Runbook for "card data breach" — per-incident-type playbook.
- Notification chain documented.
- Customer notification, processor notification, card brand notification (Visa, Mastercard) within hours.

## Common scope creep

Watch for:

- **Engineer takes a screen recording for debugging** — captures customer typing PAN. Now your laptop is in scope.
- **Customer pastes PAN into a support chat** — now your support tool is in scope.
- **Logs include full request body on payment routes** — PAN ends up in CloudWatch / Datadog. CloudWatch is now in scope.
- **Backup of payment-handling DB** — backup destination in scope.
- **CDN / WAF logs full URL with query params containing card data** — provider in scope.

The fix: **never let PAN reach you**. If support gets a screenshot with card numbers, redact and delete. If logs include card-shaped data, redact at the log layer (PII filter).

## Working with QSAs (Qualified Security Assessors)

Level 1 merchants and many Level 2 require a QSA-conducted assessment leading to a Report on Compliance (RoC). For SAQ levels (Level 3/4), self-assessment.

If you're going for SOC 2 anyway, get a QSA who can help align controls — many of the same controls cover both.

## Anti-patterns

- **Building your own card form to "save on Stripe fees"** — explodes scope. Your costs (compliance time, audit, breach risk) wildly exceed Stripe fees.
- **PAN in logs** — instant scope expansion of your log pipeline.
- **Storing CVV "to make rebilling easier"** — explicit PCI prohibition. Use stored payment methods (tokens) and processor's customer-on-file flows.
- **Backing up PAN-containing tables to S3 without explicit handling** — backup is in scope.
- **Treating SAQ A as zero work** — there are real controls; "outsourced" doesn't mean "no responsibilities."
- **Skipping ASV scans** — required quarterly; failure shows up in your audit.
- **No annual pen test** — required for any non-trivial SAQ.
- **Production engineer takes PAN screenshots** — laptop in scope; usually breach reporting.
- **Customer service tools logging chat content with PAN** — Intercom, Zendesk in scope.
- **"We're SOC 2, so we're PCI"** — different frameworks. Some overlap, but PCI has specific requirements.
- **Token reuse across processors** — Stripe and Razorpay tokens aren't interchangeable. Map per-provider.

## Verify it worked

- [ ] Card entry uses processor-hosted iframe; your servers never see PAN.
- [ ] DB schema stores only tokens, last4, brand, expiry; no PAN, no CVV anywhere.
- [ ] Log pipeline tested: pasting a card-like number in a request body does NOT appear in logs (PII filter active).
- [ ] TLS 1.2+ enforced on all payment-related routes; SSL Labs A grade.
- [ ] Quarterly ASV scans scheduled; recent scan passed.
- [ ] Annual penetration test conducted; report on file.
- [ ] Audit logs capture admin access to payment_methods table.
- [ ] Production access to payment-handling services requires MFA and is JIT.
- [ ] Incident response runbook covers payment-data breach scenario.
- [ ] SAQ document filled out and submitted to acquirer (or held for customer/processor review).
- [ ] Annual security training completed by all engineers touching payment code.
- [ ] No CVV stored anywhere; codebase grep confirms.
