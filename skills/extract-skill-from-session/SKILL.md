---
name: extract-skill-from-session
description: Take a finished workflow you found yourself doing and convert it into a reusable SKILL.md file. Captures the pattern, anti-patterns, and verification checklist before the knowledge fades. Use after solving a non-trivial problem you'll likely face again, or after teaching someone the same thing twice.
---

# Extract a Skill from a Session

The most valuable patterns are the ones you've just used successfully — they're concrete, fresh, and tested. This skill is the discipline of capturing them before context fades.

## When to use

- You just finished a non-trivial task and the approach was good.
- You found yourself re-explaining the same workflow to a teammate twice.
- You hit a problem, found the way through, and wrote a Slack message about it (that message is the seed).
- A senior engineer walked you through a process you'd want to repeat.
- After an incident, the response process worked well and should be reusable.

## When NOT to use

- One-off problem unlikely to recur.
- Domain-specific knowledge that doesn't generalize.
- The pattern is already in an existing skill — improve that one instead.
- You're not sure you did it right yet — wait until you've validated.

## The conversion

A finished session has four kinds of content:
1. **The goal** — what you set out to do.
2. **The path** — what you did.
3. **The dead ends** — what you tried that didn't work.
4. **The verification** — how you knew you were done.

A SKILL.md captures all four, generalized.

## SKILL.md template

```markdown
---
name: <kebab-case-name>
description: <one line — what triggers this skill, what it does. Specific enough that Claude can decide when to load it.>
---

# <Title>

<One-paragraph framing — the problem this solves, the audience, the trade-off being made.>

## When to use

- Concrete trigger 1
- Concrete trigger 2
- ...

## When NOT to use

- Concrete anti-trigger 1
- Concrete anti-trigger 2

## <The mental model / core concepts>

<Whatever conceptual scaffolding the reader needs. Tables work well.>

## <Concrete how-to>

<Code blocks, commands, file structures. Specific.>

## <More how-to or worked example>

<A real example, not a toy.>

## Anti-patterns

- **<Name>** — why it's wrong, what to do instead. Each one has a *reason*.

## Verify it worked

- [ ] Specific check 1
- [ ] Specific check 2
```

## How to extract from a session

### Step 1: write the description first

In one sentence: when does this skill apply, and what does it do? If you can't write this sentence, the skill is too vague. Examples of good descriptions:

- "Trace a bug to root cause instead of patching the symptom; refuses shortcuts like `try/catch` until the actual cause is understood."
- "Convert a finished workflow into a reusable SKILL.md."
- "Send transactional email via Brevo with proper SPF/DKIM/DMARC and webhook handling."

If your sentence has the word "improve" or "better" or "general," it's too vague.

### Step 2: list the triggers

When did you actually want this skill? Not when you *might* want it — when you *did*. Examples:

- "When a test is failing intermittently."
- "When opening a PR with > 500 lines."
- "When adding payments for India customers."

These are the items that go under "When to use." Be concrete.

### Step 3: list the anti-triggers

When would invoking this skill be wrong? "When NOT to use" prevents over-application. Skills loaded too eagerly waste context.

### Step 4: extract the mental model

What did the reader need to *understand* before the how-to made sense? Often a table, sometimes a diagram, sometimes a one-paragraph framing. If you skip this and jump to commands, the reader copies without understanding.

### Step 5: write the how-to

Take the actual steps you did. Generalize file paths, names, values. Keep enough specificity that a reader can execute (no `<your-app>/...` placeholders — use realistic paths like `apps/api/src/...`).

### Step 6: write anti-patterns from your dead ends

Every dead end in your session is an anti-pattern. Capture them:

- "I first tried X — failed because of Y."
- "Initial attempt was Z — wrong because W."

These are the most valuable parts of the skill. Random docs don't have anti-patterns; they don't know which mistakes are common. Your session does.

### Step 7: write the verification checklist

How did you know you were done? Each item is a concrete check the user can perform. Specific commands, observable outcomes. "Tests pass" is too vague; "`pnpm test` passes with zero failures and coverage > 80%" is specific.

## What makes a SKILL.md "good"

| Property | What it looks like |
|---|---|
| **Specific triggers** | "When adding Razorpay" not "when working on payments" |
| **Decision tables** | When the reader has a choice, show the choice + criteria |
| **Real examples** | `apps/api/src/routes/checkout.ts:54` not `<your file>` |
| **Anti-patterns with reasons** | "Don't use Y because Z happens" not "Don't use Y" |
| **Verification checklist** | Mechanical checks, not "looks good" |
| **Bounded scope** | One topic, ~150–250 lines, one mental model |
| **Anti-padding** | No "Introduction," no "In conclusion," no marketing |

## Sizing a skill

| Skill size | Use case |
|---|---|
| 100–150 lines | Workflow / meta-skills (find-real-bug, write-commit-message) |
| 150–250 lines | Tool integration with examples (razorpay-integration, kubernetes-helm) |
| 250–400 lines | Complex platform skills with multiple flows (scaffold-saas-starter, opentelemetry-instrument) |
| 400+ lines | Probably two skills — split. |

When unsure, lean shorter. A 150-line skill that's read is more useful than a 400-line skill that's skimmed.

## Picking the name

- Kebab-case (`razorpay-integration`, not `razorpayIntegration` or `razorpay_integration`).
- Verb-first if it's a workflow (`find-real-bug`, `shrink-this-pr`).
- Tool name + action if it's an integration (`razorpay-integration`, `kubernetes-helm`).
- Don't include the bucket name (don't call it `meta-find-real-bug` or `platform-razorpay`).

## Anti-patterns

- **Vague description** — "Helps with debugging." Doesn't tell Claude when to load it.
- **No anti-patterns section** — the part that makes a skill load-bearing.
- **Verification = "looks good"** — not testable; not falsifiable.
- **Wrapping the official docs** — if the skill just paraphrases the README, it adds no value.
- **Including obscure edge cases your team never hits** — bloats the skill.
- **No "When NOT to use"** — skill applies too eagerly.
- **Marketing tone** — "delightfully simple," "powerful." Audience is engineers; talk like one.
- **Stale skill** — patterns evolve; reviewing every quarter prevents rot.
- **Over-extracting** — not every solved problem deserves a skill. The bar: "would I use this again, and would I want my future-self to remember it?"
- **Under-extracting** — solving the same problem three times in three months means there's a skill missing.

## Verify it worked

- [ ] The skill has a specific, triggerable description (one sentence).
- [ ] "When to use" lists concrete triggers.
- [ ] "When NOT to use" prevents over-application.
- [ ] Anti-patterns section exists; each anti-pattern has a reason.
- [ ] Verification checklist is mechanical, not "looks good."
- [ ] You can describe the skill in one sentence to a teammate; they'd know when to invoke it.
- [ ] The skill is < 400 lines; if longer, split.
- [ ] Examples reference realistic file paths and values.
- [ ] No marketing language.
- [ ] The skill saves time on the *next* invocation — the value is forward, not retrospective.
