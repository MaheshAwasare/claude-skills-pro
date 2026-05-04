---
name: razorpay-integration
description: Integrate Razorpay payments end-to-end — orders, subscriptions, webhooks, refunds, signature verification, and the India-specific gotchas (RBI auto-debit mandates, GST invoices, partial COD). Use when adding Razorpay to a Node.js, Python, or Go backend, or when debugging webhook signature failures, subscription state machines, or KYC/onboarding flows.
---

# Razorpay Integration

Razorpay is the dominant payment gateway in India. The official docs are good for individual API endpoints but never tell you the *opinionated path*: which flow to use for your case, where the foot-guns are, and how to keep the webhook handler idempotent. This skill is that opinionated path.

## When to use

- Adding Razorpay payments to a backend (Node / Python / Go).
- Implementing subscriptions with RBI auto-debit mandates (e-mandate / UPI Autopay).
- Debugging webhook signature verification failures.
- Reconciling order/payment/subscription state machines.
- KYC onboarding flows for Razorpay Route (marketplace).

## When NOT to use

- US/EU-only customers — use `stripe-integration` instead.
- You only need a one-time payment link, no programmatic integration — use Razorpay's no-code Payment Pages.
- You need PCI Level 1 self-handling — Razorpay is itself PCI compliant; don't replicate.

## Core concepts (the mental model)

Razorpay's data model has **four objects you must keep distinct**:

| Object | Purpose | Lifetime |
|---|---|---|
| `order` | Server-created intent to charge a specific amount | Until paid or expired |
| `payment` | A specific attempt against an order (can succeed or fail) | Permanent record |
| `subscription` | Recurring auto-charge with a plan | Until cancelled |
| `refund` | Reverse of a captured payment | Permanent record |

**Critical rule:** never trust the client. Always create the `order` server-side, return only the `order_id` to the browser, and verify the signature server-side after payment.

## The standard one-time payment flow

### 1. Server creates the order

```js
// Node.js — apps/api/src/routes/checkout.ts
import Razorpay from "razorpay";

const rzp = new Razorpay({
  key_id: process.env.RAZORPAY_KEY_ID!,
  key_secret: process.env.RAZORPAY_KEY_SECRET!,
});

export async function createOrder(req, res) {
  const { planId, userId } = req.body;
  const plan = await db.plan.findUnique({ where: { id: planId } });
  if (!plan) return res.status(404).end();

  const order = await rzp.orders.create({
    amount: plan.priceInPaise, // ALWAYS in paise (₹100 = 10000)
    currency: "INR",
    receipt: `order_${userId}_${Date.now()}`,
    notes: { userId, planId }, // searchable later in dashboard
  });

  await db.checkoutSession.create({
    data: { razorpayOrderId: order.id, userId, planId, status: "created" },
  });

  res.json({ orderId: order.id, amount: order.amount, keyId: process.env.RAZORPAY_KEY_ID });
}
```

### 2. Browser opens Razorpay Checkout

```html
<script src="https://checkout.razorpay.com/v1/checkout.js"></script>
<script>
  const rzp = new Razorpay({
    key: keyId,                    // from server response
    order_id: orderId,             // from server response
    amount,                        // from server response
    currency: "INR",
    name: "Acme Corp",
    description: "Pro plan",
    handler: async (response) => {
      // response = { razorpay_order_id, razorpay_payment_id, razorpay_signature }
      await fetch("/api/checkout/verify", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(response),
      });
    },
    prefill: { email: user.email, contact: user.phone },
    theme: { color: "#3399cc" },
  });
  rzp.open();
</script>
```

### 3. Server verifies the signature

```js
import crypto from "node:crypto";

export async function verifyPayment(req, res) {
  const { razorpay_order_id, razorpay_payment_id, razorpay_signature } = req.body;

  const expected = crypto
    .createHmac("sha256", process.env.RAZORPAY_KEY_SECRET!)
    .update(`${razorpay_order_id}|${razorpay_payment_id}`)
    .digest("hex");

  if (expected !== razorpay_signature) {
    return res.status(400).json({ error: "invalid signature" });
  }

  // Signature OK — but DO NOT mark fulfilled here.
  // Mark "verified" and let the webhook do final fulfillment.
  await db.checkoutSession.update({
    where: { razorpayOrderId: razorpay_order_id },
    data: { status: "verified", razorpayPaymentId: razorpay_payment_id },
  });

  res.json({ ok: true });
}
```

**Why not fulfill here?** The browser can close, JS can fail, the user can navigate away. The webhook is the source of truth. Fulfill in the webhook handler only.

## Webhooks — the source of truth

### Configure
Dashboard → Settings → Webhooks → add endpoint `https://api.example.com/razorpay/webhook`. Subscribe to at least:
- `payment.captured`
- `payment.failed`
- `order.paid`
- `subscription.charged`
- `subscription.halted`
- `refund.processed`

Razorpay will give you a **webhook secret** distinct from your API secret. Store it as `RAZORPAY_WEBHOOK_SECRET`.

### Verify and handle

```js
// apps/api/src/routes/razorpay-webhook.ts
import express from "express";
import crypto from "node:crypto";

export const router = express.Router();

// IMPORTANT: raw body, not JSON-parsed. Order matters.
router.post(
  "/razorpay/webhook",
  express.raw({ type: "application/json" }),
  async (req, res) => {
    const signature = req.header("X-Razorpay-Signature");
    const expected = crypto
      .createHmac("sha256", process.env.RAZORPAY_WEBHOOK_SECRET!)
      .update(req.body) // raw Buffer
      .digest("hex");

    if (expected !== signature) return res.status(400).end();

    const event = JSON.parse(req.body.toString());
    const eventId = req.header("X-Razorpay-Event-Id"); // for idempotency

    // Idempotency: bail if already processed.
    const seen = await db.webhookEvent.findUnique({ where: { id: eventId } });
    if (seen) return res.status(200).json({ ok: true, deduped: true });

    await db.$transaction(async (tx) => {
      await tx.webhookEvent.create({ data: { id: eventId, type: event.event } });
      await handle(event, tx);
    });

    res.status(200).json({ ok: true });
  }
);

async function handle(event, tx) {
  switch (event.event) {
    case "payment.captured":
      await fulfillOrder(tx, event.payload.payment.entity);
      break;
    case "subscription.charged":
      await extendSubscription(tx, event.payload.subscription.entity);
      break;
    case "subscription.halted":
      await markSubscriptionHalted(tx, event.payload.subscription.entity);
      break;
    case "refund.processed":
      await markRefunded(tx, event.payload.refund.entity);
      break;
    // ignore unknown events — Razorpay adds new ones
  }
}
```

## Subscriptions (RBI e-mandate)

Since Sept 2021, RBI rules require explicit per-charge mandates for recurring auto-debit > ₹15,000 (raised over time; check current cap). Razorpay handles the mandate flow but you need to know:

1. **Create a Plan** in dashboard or via API — `period: "monthly"`, `interval: 1`, `item.amount`.
2. **Create Subscription** server-side; redirect user to authorization URL.
3. User completes mandate (UPI Autopay / debit card / net banking with bank OTP).
4. Webhook `subscription.activated` fires.
5. On each cycle, webhook `subscription.charged` fires (or `subscription.pending` → `subscription.halted` if mandate fails).

```js
const sub = await rzp.subscriptions.create({
  plan_id: "plan_XXXXX",
  total_count: 12,            // 12 cycles for annual cap
  customer_notify: 1,
  notes: { userId },
});

// Redirect browser to:
//   https://rzp.io/i/<sub.short_url>
// Or use Checkout with subscription_id instead of order_id.
```

**State machine** to mirror in your DB:

```
created → authenticated → active → charged (recurring)
                              ↓
                          halted | cancelled | expired | completed
```

Drive UI off your DB state, not Razorpay's API — webhooks are eventually consistent (typically <30s, occasionally minutes).

## Refunds

```js
// Full refund
await rzp.payments.refund(paymentId, { speed: "normal" });

// Partial — amount in paise
await rzp.payments.refund(paymentId, { amount: 5000, speed: "optimum" });

// Notes for reconciliation
await rzp.payments.refund(paymentId, {
  amount: 5000,
  notes: { reason: "partial_cancellation", ticketId: "T-1234" },
});
```

`speed: "optimum"` uses instant refund rails (UPI/IMPS) where supported — costs more but lands in seconds. Default `normal` takes 5–7 working days for cards.

## Anti-patterns

- **Trusting client-side `amount`** — *always* use the server-stored amount when creating the order. Client can tamper.
- **Fulfilling on browser callback (`handler`) instead of webhook** — browser can die after capturing. Webhook is the only reliable signal.
- **JSON-parsing the webhook body before signature verification** — Express's `express.json()` middleware will run first if you don't carve out the route. Signature must be computed against the raw bytes.
- **No idempotency on webhook handler** — Razorpay retries on 5xx and on timeout. Without dedup by `X-Razorpay-Event-Id`, you'll double-fulfill.
- **Storing amounts as floats / rupees** — always integer paise. Floats lose precision; rupees with two decimals invite mistakes.
- **Mixing test and live keys in one env** — separate `.env.test` / `.env.production`. Test keys start `rzp_test_`, live keys `rzp_live_`. Logs leak secrets when the wrong env loads in production.
- **Ignoring `subscription.halted`** — when a mandate fails (insufficient balance, expired card), Razorpay emits `halted`, *not* a fresh `charged`. If you only listen to `charged`, paying users silently drift to expired.
- **Using API key for webhook verification** — webhooks have their own secret. They are not interchangeable.

## Verify it worked

- [ ] Test card `4111 1111 1111 1111` with any future expiry and any CVV creates a successful payment in test mode.
- [ ] Webhook signature verification rejects a tampered body (flip one byte → 400).
- [ ] Replaying the same webhook twice results in only one fulfillment (idempotency check).
- [ ] Server creates the `order` with the price *from the database*, not the request body.
- [ ] Amounts in code and DB are integer paise everywhere — grep for `amount.*\.\d` should return nothing in payment paths.
- [ ] Subscription state in your DB is updated only via webhooks, not the success callback.
- [ ] Refund creates a record in your DB before calling Razorpay (so a network failure leaves a reconcilable trail).
- [ ] `RAZORPAY_KEY_SECRET` and `RAZORPAY_WEBHOOK_SECRET` are different env vars and both are required at boot.
