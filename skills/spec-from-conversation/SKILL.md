---
name: spec-from-conversation
description: Take a messy back-and-forth (Slack thread, meeting notes, ad-hoc requirements) and produce a clean PRD/spec — goals, non-goals, user flows, edge cases, acceptance criteria. Use when someone says "just build X" with no written spec, or before a multi-week feature when the requirements live in five Slack threads.
---

# Extract a Spec from a Conversation

A feature without a written spec is a feature that drifts. Drift is when the engineer's interpretation diverges from the asker's intent — discovered three weeks in. This skill converts conversation into a spec that pins down the contract.

## When to use

- Stakeholder said "just build X" with no written requirements.
- Requirements live across multiple Slack threads / meeting notes.
- Engineer is about to start a multi-week feature without a spec.
- Bug ticket that's actually a feature ask in disguise.
- Mid-implementation realization that the spec is ambiguous on a key point.

## When NOT to use

- Trivial changes (typo, lint, single-file fix).
- Already have a spec — propose edits, don't rewrite.
- Discovery / research — spec comes after, not before.

## The spec template

```markdown
# Spec: <feature name>

- **Author**: <name>
- **Date**: 2026-05-04
- **Status**: Draft / In Review / Approved
- **Stakeholders**: <names of people who must agree>

## Problem

What problem does this solve, for whom? In one paragraph.

Bad: "We need a refunds API."
Good: "Customer support is issuing full refunds for partial requests because
the dashboard doesn't expose partial. Refund $-volume is up 8x from change
requests since launch (#2156). Partial refund support reduces $-loss without
changing the support agent workflow."

## Goals

- Concrete, measurable. Avoid "improve UX."
- Each goal is independently testable.

## Non-goals

What this *won't* do. Critical for scope clarity.

Examples:
- "Refund-by-line-item is out of scope; tracked as #2233."
- "We won't support refunds older than 365 days (Razorpay limit)."
- "We won't expose this in the customer-facing UI; agent-only."

## User flows

Happy path, step-by-step. One flow per user role.

### Agent issues a partial refund (happy path)

1. Agent navigates to a customer's payment.
2. Clicks "Refund."
3. Sees "Full refund" / "Partial refund" toggle. Selects partial.
4. Enters amount (in INR; helper text says "remaining: ₹X").
5. Confirms reason (dropdown: cancellation / quality / other).
6. Sees confirmation modal: "Refund ₹X to <method>? Customer will see this in 5–7 days (or instantly via UPI)."
7. Submits. Sees success toast. Refund row appears in payment history.

### Agent issues a partial refund (over-limit)

1. ... step 4: enters ₹2000 when remaining is ₹1500.
2. Field highlights red; helper text becomes "Cannot exceed remaining ₹1500."
3. Submit button disabled.

## Acceptance criteria

A checklist that, when all green, means the feature is done. Each item is
testable.

- [ ] Endpoint `POST /api/refunds` accepts optional `amount` field (paise).
- [ ] When `amount` < `payment.amount_paise`, calls Razorpay with `speed: "optimum"`.
- [ ] When `amount` > `remaining_refundable`, returns 422 with code `EXCEEDS_REMAINING`.
- [ ] When `amount` <= 0, returns 422 with code `INVALID_AMOUNT`.
- [ ] Agent UI shows partial-refund toggle when `remaining_refundable > 0`.
- [ ] Agent UI hides toggle (full-refund-only) when `remaining_refundable == payment.amount_paise`.
- [ ] Webhook `refund.processed` updates `payments.refunded_paise` and `remaining_refundable` recomputes.
- [ ] Audit log captures: agent ID, payment ID, amount, reason.
- [ ] E2E test: agent issues a partial refund; refund row appears in payment history.
- [ ] Documentation: API doc updated with new field.

## Edge cases

- What if the customer's card is now closed? (Razorpay returns specific error; we surface it.)
- What if there are concurrent refund attempts on the same payment? (Row lock on `payments`; second errors with `CONCURRENT_REFUND`.)
- What if the original payment is in a foreign currency? (Out of scope for v1; reject in API.)
- What if partial refund is requested before the original capture is settled? (Razorpay rejects; we pass the error through.)

## Open questions

Things we haven't decided yet. **Must be resolved before implementation.**

- [ ] Should partial refunds count toward the support agent's "refund quota"?
  Owner: @lead-support
- [ ] Do we expose partial refund history to the customer email receipt?
  Owner: @design

## References

- Slack: <link to original thread>
- Issue: #2156
- Razorpay docs: <link>
- Mocks: <Figma link>
```

## How to extract from conversation

### Step 1: read everything

Slack threads, meeting notes, related tickets, comments on the existing PR if any. **Don't skim.** A 10-minute read prevents a 10-day rebuild.

### Step 2: list every requirement claim

Quote-extract every "we need" / "it should" / "the user wants" / "make sure that" sentence into a flat list, with the source.

### Step 3: deduplicate and rank

Many claims are the same idea phrased differently. Group them. Then rank: what's the actual core, what's nice-to-have, what's contradictory?

### Step 4: surface contradictions

When two stakeholders said opposing things, that's the spec's job to surface — not bury. Open a question:

> **Open**: @support said "always email the customer on refund." @design said "don't email if already on the payment confirmation thread." Which?

Send the spec back to both. Don't make the call yourself unless you're authorized.

### Step 5: flush out edge cases

What happens when:
- The user does the action twice quickly?
- The data is in an unexpected state (deleted, archived, in another tenant)?
- The downstream system fails (Razorpay timeout, Postgres pool exhausted)?
- The feature interacts with another in-flight feature?

These rarely surface in conversation. Add them.

### Step 6: get sign-off

Send the spec to the stakeholders. **Get a thumbs-up or written response.** Without sign-off, drift is inevitable — you'll build it; the asker will say "that's not what I meant."

## "But the requirements aren't clear yet"

Then write what you *do* know. Mark the rest as Open Questions with owners. The spec's job at this stage is to surface the unknowns, not pretend they don't exist.

A spec with 8 known requirements and 4 open questions is better than no spec. The 4 open questions force a conversation that *would* have happened mid-implementation, but earlier — when course-correction is cheap.

## Anti-patterns

- **No written spec for multi-week work** — guarantees drift.
- **Spec written by the engineer alone** — captures interpretation, not source intent.
- **Spec skipped because "we discussed it in the meeting"** — meeting memory is unreliable.
- **No non-goals** — scope creeps.
- **No acceptance criteria** — "done" is subjective.
- **No edge cases** — happy path ships, edge cases ship as bugs.
- **Open questions without owners** — they don't get resolved.
- **Spec in a wiki nobody reads** — link from the PR, the ticket, Slack.
- **Spec that's a wall of prose** — use sections, lists, checkboxes. Skim-friendly.
- **Marketing language** — avoid "delight," "seamless," "intuitive." Use observable behavior.
- **"We'll figure that out in code review"** — code review is too late for spec ambiguity.
- **Spec longer than the feature** — diminishing returns. 1–3 pages for most features.

## Verify it worked

- [ ] Problem statement is concrete and links to a real signal (issue, metric, complaint).
- [ ] Goals are independently testable.
- [ ] Non-goals exist and are explicit.
- [ ] User flows for the happy path and at least one error path.
- [ ] Acceptance criteria is a checklist.
- [ ] Edge cases section is non-empty.
- [ ] Every open question has an owner.
- [ ] Stakeholders have signed off in writing.
- [ ] Spec lives in the repo or tracker, linked from the PR.
- [ ] Implementation matches the spec; deviations are tracked as spec updates, not silent code drift.
