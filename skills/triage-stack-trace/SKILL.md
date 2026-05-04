---
name: triage-stack-trace
description: Take a raw stack trace (or error log) and produce a ranked list of hypotheses + the single file most worth opening first. Use when an error appears in logs, a Sentry issue lands, or a user reports a stack trace from production. Pairs with find-real-bug for the deeper investigation.
---

# Triage a Stack Trace

A stack trace is overwhelming until you know the algorithm to read it. This skill is the algorithm: skim, rank, pick the one file to open first.

## When to use

- Error / exception logged in production with a stack trace.
- Sentry issue with multiple events and need to find the root file.
- Failing test outputting a long Python/Java/Node stack.
- User-reported error message that includes a partial trace.

## When NOT to use

- One-line error with no trace (different problem — needs reproduction first).
- Stack trace from C++ crash dump (use a debugger; this skill is for higher-level languages).

## The algorithm

1. **Find the root cause line** — the last "your code" frame, not the framework's. Most stacks bottom out in framework code (Express, FastAPI, Django middleware) — that's not your bug; the frame *above* it is.
2. **Find the error message** — the first line, usually. Note exact wording — that's your search key.
3. **Note the values, if any** — many runtimes include args: `TypeError: Cannot read property 'name' of undefined at User.greet (user.ts:12)`. The `undefined` is the data hint.
4. **Find adjacent context** — from logs/Sentry, the request body, user ID, time. These shape the hypotheses.
5. **Rank hypotheses** — 1–3 candidate causes ordered by likelihood.
6. **Pick the one file to open** — usually the file at step 1. State it clearly.

## Reading a Node stack

```
TypeError: Cannot read property 'email' of undefined
    at sendInvitationEmail (services/email.ts:45:18)
    at processInvitation (services/invitations.ts:88:5)
    at async createInvitation (routes/invitations.ts:23:3)
    at async Layer.handle [as handle_request] (express/router/layer.js:95:5)
    at async next (express/router/route.js:144:13)
    ...
```

| Step | Action |
|---|---|
| Skip frames in `node_modules/` and `express/` | Not your bug |
| Stop at first `services/` or `routes/` frame | Your code |
| Top: `services/email.ts:45` reads `.email` on `undefined` | The line that threw |
| Caller: `services/invitations.ts:88` — what was passed? | Probably the missing context |

**Hypothesis:** `sendInvitationEmail` was called with a user object that doesn't have `email`. Either the user wasn't loaded properly, or the lookup returned null and wasn't checked.

**Open first:** `services/invitations.ts:88` — see what's passed to `sendInvitationEmail`. The bug is more likely there than at the throw point.

## Reading a Python stack

```
Traceback (most recent call last):
  File "/app/api/routes/charges.py", line 42, in create_charge
    return await svc.create_charge(db, user, payload)
  File "/app/api/services/payments.py", line 78, in create_charge
    razorpay_charge = await razorpay.create_order(amount=payload.amount)
  File "/app/.venv/lib/python3.12/site-packages/razorpay/client.py", line 156, in create_order
    raise BadRequestError(response.text)
razorpay.errors.BadRequestError: amount must be at least 100 paise
```

| Step | Action |
|---|---|
| Bottom of trace = where it actually raised | razorpay SDK |
| Top of "your code" = where to focus | `payments.py:78` |
| Error message: "amount must be at least 100 paise" | Validation hint — what was the amount? |

**Hypothesis:** `payload.amount` is < 100. Why? Either client sent it (validation gap), or our code does something to it (rounding to 0?).

**Open first:** `services/payments.py:78` — and look upward to see what's in `payload.amount`. Likely the schema doesn't enforce min, or there's a paise/rupee unit confusion.

## Reading a Java stack

Java stacks are long because of frameworks (Spring, Hibernate). Same algorithm: skip framework, find your code.

```
java.lang.NullPointerException: Cannot invoke "Project.getName()" because the return value of "ProjectRepository.findById(...)" is null
    at com.acme.service.ProjectService.rename(ProjectService.java:54)
    at com.acme.controller.ProjectController.rename(ProjectController.java:23)
    at jdk.internal.reflect.GeneratedMethodAccessor123.invoke(...)
    ...
```

Java 14+ helpfully points to the null in the message. **Hypothesis:** `findById` returned null (Optional.empty()/null), and the caller didn't check. **Open first:** `ProjectService.java:54`. Likely fix: change to `findById(...).orElseThrow(...)` or explicit null check.

## Patterns by error type

| Error | Likely cause | First check |
|---|---|---|
| `null/undefined` access | Upstream returned nothing; not checked | The query/fetch above the failure point |
| `TypeError` | Wrong type passed | The function signature vs caller |
| Timeout | Downstream slow or pool exhausted | DB pool stats; downstream latency |
| `Connection refused` | Service not up; wrong host/port | `nc -vz`, `ping`, env vars |
| `signature invalid` (HMAC) | Body mutated, wrong secret, encoding | Raw body discipline; secret env var |
| `Permission denied` | IAM/role issue | The role's policy + the resource's permissions |
| Out of memory | Leak or too-large allocation | Heap dump; recent code touching big collections |
| Deadlock | Two paths acquire locks in different order | Recent locking changes; query plans |

## Multiple events, one Sentry issue

If Sentry shows the same error fired 100 times across different users, look for:

- **Common request body shape** — same field, missing or weird.
- **Common user/org** — bug only for one tenant; that tenant has bad data.
- **Common time** — fired after a deploy / config change.
- **Common request path** — only one endpoint, narrows the surface.

Sentry's "Tags" panel lists these automatically. Use them.

## When the trace points at framework code

If the only "your code" frame is `index.js:1`, your stack trace is incomplete:

- **Async loss** — Node before 12 lost stacks across `await`. Modern Node should preserve. Check Node version.
- **Stripped sourcemaps** — see `sentry-monitoring`. Upload sourcemaps so prod traces are readable.
- **Bundler obfuscation** — webpack/esbuild may inline; sourcemaps fix.

Without good traces, you're guessing. Fix the trace pipeline first.

## Anti-patterns

- **Reading the trace bottom-up** — that's the framework. Top-down (or "first your-code frame") is the data.
- **Pasting trace into Google verbatim** — the message + line number are too specific. Strip values, keep the error class, search.
- **Treating the throw point as the bug** — the *cause* is usually one frame up, not the throw.
- **Ignoring the error message** — "TypeError" alone is generic; "Cannot read property X of undefined" tells you which property and that the value was undefined.
- **Triaging multiple unrelated errors as one issue** — Sentry sometimes groups them. Verify by inspecting individual events.
- **Acting on hypothesis without verifying** — "probably a race condition" without proof leads to the wrong fix.
- **Skipping the request context** — the body/user/headers are often the data hint.

## Verify it worked

- [ ] Error message + first your-code frame identified.
- [ ] 1–3 hypotheses ranked by likelihood.
- [ ] One specific file:line called out as "open first."
- [ ] Request context (user, body, time) noted if available.
- [ ] If multiple events, common factors identified.
- [ ] Sourcemap / trace pipeline confirmed working (de-minified frames).
- [ ] If hypothesis confirmed, hand off to `find-real-bug` for the fix.
- [ ] If hypothesis wrong, articulate why — surfacing the next likely cause.
