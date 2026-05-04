---
name: stripe-integration
description: Integrate Stripe end-to-end — Checkout, Customer Portal, subscriptions with proration, webhooks with idempotency, refunds, invoicing, tax. Use when adding Stripe to a Node/Python/Go backend or debugging webhook signature/idempotency issues. Pairs with razorpay-integration for India/SE Asia.
---

# Stripe Integration

Stripe's docs are excellent for individual endpoints. This skill is the *flow* — which endpoints to compose, where the foot-guns are, and how to keep the webhook handler idempotent.

## When to use

- Adding Stripe to a global SaaS (US/EU customers).
- Subscription billing with plans, trials, proration.
- Debugging webhook signature failures or duplicate fulfillment.
- Migrating off Razorpay/Paddle to Stripe.

## When NOT to use

- India-only — `razorpay-integration` handles RBI mandates better.
- Marketplace with split payouts — use Stripe Connect (different scaffolding).

## Core objects

| Object | Purpose |
|---|---|
| `Customer` | Long-lived person/org record |
| `Product` + `Price` | Catalog. A product has many prices (monthly/annual/currency) |
| `Subscription` | A customer's active recurring purchase of a price |
| `Invoice` | Generated per billing cycle; can be paid, void, or open |
| `Checkout Session` | Hosted payment page; produces a subscription or one-time payment |
| `Customer Portal` | Hosted UI for cancel, update card, view invoices |
| `Event` | Webhook payload for state changes |

**Critical:** never store card details. Use Checkout or Elements; Stripe holds the PAN.

## The 80% flow: Checkout + Customer Portal + webhook

### 1. Create a Checkout Session

```ts
// apps/api/src/routes/checkout.ts
import Stripe from "stripe";
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, { apiVersion: "2024-12-18.acacia" });

export async function startCheckout(req, res) {
  const { userId, priceId } = req.body;
  const user = await db.user.findUnique({ where: { id: userId } });
  if (!user) return res.status(404).end();

  // Reuse existing Stripe customer if we have one
  let customerId = user.stripeCustomerId;
  if (!customerId) {
    const c = await stripe.customers.create({ email: user.email, metadata: { userId } });
    customerId = c.id;
    await db.user.update({ where: { id: userId }, data: { stripeCustomerId: customerId } });
  }

  const session = await stripe.checkout.sessions.create({
    mode: "subscription",
    customer: customerId,
    line_items: [{ price: priceId, quantity: 1 }],
    success_url: `${process.env.APP_URL}/billing/success?session_id={CHECKOUT_SESSION_ID}`,
    cancel_url: `${process.env.APP_URL}/billing`,
    allow_promotion_codes: true,
    automatic_tax: { enabled: true },                    // requires Stripe Tax
    subscription_data: { metadata: { userId } },         // surfaces in webhook
  });

  res.json({ url: session.url });
}
```

### 2. Customer Portal for self-service

```ts
export async function openPortal(req, res) {
  const user = await db.user.findUnique({ where: { id: req.userId } });
  const portal = await stripe.billingPortal.sessions.create({
    customer: user.stripeCustomerId,
    return_url: `${process.env.APP_URL}/billing`,
  });
  res.json({ url: portal.url });
}
```

Configure Portal in Dashboard → Settings → Customer Portal: enable cancel, update card, change plan, view invoices. **You don't need to build any of this UI.**

### 3. Webhook handler (the source of truth)

```ts
import express from "express";
const router = express.Router();

router.post(
  "/stripe/webhook",
  express.raw({ type: "application/json" }),                  // raw body!
  async (req, res) => {
    const sig = req.header("stripe-signature");
    let event: Stripe.Event;
    try {
      event = stripe.webhooks.constructEvent(req.body, sig!, process.env.STRIPE_WEBHOOK_SECRET!);
    } catch {
      return res.status(400).end();
    }

    const seen = await db.webhookEvent.findUnique({ where: { id: event.id } });
    if (seen) return res.status(200).json({ deduped: true });

    await db.$transaction(async (tx) => {
      await tx.webhookEvent.create({ data: { id: event.id, type: event.type } });
      await handle(event, tx);
    });

    res.status(200).json({ ok: true });
  }
);

async function handle(event: Stripe.Event, tx) {
  switch (event.type) {
    case "customer.subscription.created":
    case "customer.subscription.updated":
      await upsertSubscription(tx, event.data.object as Stripe.Subscription);
      break;
    case "customer.subscription.deleted":
      await markSubscriptionCanceled(tx, event.data.object as Stripe.Subscription);
      break;
    case "invoice.payment_failed":
      await flagPaymentFailed(tx, event.data.object as Stripe.Invoice);
      break;
    case "invoice.payment_succeeded":
      await markInvoicePaid(tx, event.data.object as Stripe.Invoice);
      break;
  }
}
```

The events you actually need (everything else can be ignored at first):
- `customer.subscription.created` / `updated` / `deleted`
- `invoice.payment_succeeded` / `payment_failed`
- `checkout.session.completed` (for one-time payments)

## State machine to mirror in your DB

```
incomplete → trialing → active → past_due → canceled
                            ↓
                         unpaid (after dunning fails)
```

Drive UI from `subscriptions.status` in *your* DB, updated only by webhooks. Don't query Stripe per page load.

## Subscription changes (proration)

```ts
// Upgrade plan with prorated charge mid-cycle
await stripe.subscriptions.update(subId, {
  items: [{ id: subItemId, price: newPriceId }],
  proration_behavior: "create_prorations",        // default; bills the diff now
});

// Or end-of-period change without proration
await stripe.subscriptions.update(subId, {
  items: [{ id: subItemId, price: newPriceId }],
  proration_behavior: "none",
  billing_cycle_anchor: "unchanged",
});
```

Pick once and document. Mixing creates confusion when customers see weird invoice line items.

## Test cards

| Number | Behavior |
|---|---|
| `4242 4242 4242 4242` | Always succeeds |
| `4000 0027 6000 3184` | Requires 3DS authentication |
| `4000 0000 0000 9995` | Insufficient funds (decline) |
| `4000 0000 0000 0341` | Charge succeeds, attached card later fails |

Test mode vs live mode keys are visually distinct (`sk_test_...` vs `sk_live_...`). Wrap creation in a check at boot:

```ts
if (process.env.NODE_ENV === "production" && !process.env.STRIPE_SECRET_KEY?.startsWith("sk_live_")) {
  throw new Error("Production env using test Stripe key — refusing to start");
}
```

## Anti-patterns

- **Trusting client-side `priceId`** — server must verify the price the user selected is one of the offered plans (e.g. enum or DB lookup), not free-form.
- **JSON-parsing webhook before signature verification** — `express.json()` middleware will run first if not carved out. Use `express.raw()` on this route only.
- **Fulfilling on `checkout.session.completed` redirect callback** — browser can die. Webhook is canonical.
- **No idempotency on webhook handler** — Stripe retries on 5xx; without `event.id` dedup, you'll double-fulfill.
- **Storing subscription state by querying Stripe per request** — slow, hits API limits, and Stripe is eventually consistent. Mirror in your DB.
- **One Stripe account for prod + staging** — staging emails and charges leak. Use separate accounts.
- **Per-user `priceId` in DB** — Stripe's price IDs are environment-scoped. Sync prod and staging price IDs by name/lookup_key, not by ID.
- **Custom retry/dunning logic in your code** — Stripe Smart Retries handles this; configure in dashboard, don't reinvent.
- **Hardcoding tax rates** — Stripe Tax handles 70+ jurisdictions. Enable `automatic_tax: { enabled: true }`.
- **Fetching `event.data.object` and assuming it's still current** — Stripe events can be delayed. Re-fetch the object via API if you need latest state.

## Verify it worked

- [ ] Test card `4242...` produces a successful subscription; row in your DB has `status: 'active'`.
- [ ] Tampered webhook body is rejected (signature check works).
- [ ] Replaying same `event.id` twice → only one fulfillment.
- [ ] Customer Portal link opens with cancel + update card + change plan; no custom UI built.
- [ ] `customer.subscription.deleted` webhook updates DB to `canceled`.
- [ ] Failed payment (`4000 0000 0000 9995`) triggers `invoice.payment_failed` → DB `past_due`.
- [ ] Production env refuses to boot with `sk_test_` key.
- [ ] `stripeCustomerId` is on the user row before any subscription is created.
- [ ] Stripe Tax shows tax on test invoice for an EU customer.
- [ ] Prod + staging are separate Stripe accounts (different `STRIPE_SECRET_KEY`).
