---
name: threat-modeling-stride
description: Threat-model a system using STRIDE — Spoofing, Tampering, Repudiation, Information Disclosure, Denial of Service, Elevation of Privilege. Walks through data flow diagrams, trust boundaries, and per-component threat enumeration with mitigations. Use before adding auth/payments/PII handling, after a near-miss, or as part of a secure SDLC.
---

# STRIDE Threat Modeling

STRIDE is the most-used threat modeling methodology. It scales from a 30-minute napkin model to a multi-day deep dive. This skill is the engineering-grade walkthrough that produces an actionable risk register.

## When to use

- Designing new auth, payments, or PII-handling flows.
- Architecture review for a new service.
- Before significant external exposure (public API, mobile app, third-party integration).
- After a near-miss / minor incident, before the next variant happens.
- Annual security review (lighter pass).

## When NOT to use

- Internal tooling with low blast radius (still useful, but proportional effort).
- Pure compliance-driven model with no engineering follow-through — wasted artifact.

## STRIDE in one table

| Letter | Threat | Property violated |
|---|---|---|
| **S**poofing | Pretending to be someone else | Authentication |
| **T**ampering | Modifying data/code maliciously | Integrity |
| **R**epudiation | Denying having done something | Non-repudiation |
| **I**nformation Disclosure | Leaking data | Confidentiality |
| **D**enial of Service | Making system unusable | Availability |
| **E**levation of Privilege | Gaining unauthorized capabilities | Authorization |

For each component / flow / trust boundary, ask: how could each of these go wrong?

## The workflow

### 1. Draw the data flow diagram (DFD)

Identify:
- **External entities** (users, third-party services).
- **Processes** (your services, functions).
- **Data stores** (DBs, S3, queues).
- **Data flows** (arrows between).
- **Trust boundaries** (where trust changes — internet ↔ VPC, app ↔ DB).

Don't over-engineer the diagram. A whiteboard photo is fine. The point is to make implicit interactions explicit.

```
                Internet (untrusted)
                       |
                       v
                   [Cloudflare]
                       |  --- Trust boundary ---
                       v
                   [API gateway]
                       |
                       v
              [auth-service] -----> [Postgres (auth)]
                       |
                       v
              [payment-service] --> [Postgres (payments)]
                       |
                       +-> [Razorpay API] (third-party, external trust)
                       |
                       +-> [Brevo API]
```

### 2. Per component, walk STRIDE

For each process, data store, and data flow, ask the 6 STRIDE questions. Not every threat applies to every element — fine.

#### Example: `payment-service` process

| STRIDE | Threat | Mitigation in place | Residual |
|---|---|---|---|
| S | Attacker calls payment-service directly bypassing API gateway | Service has no public IP; mTLS between gateway and service | Low |
| T | Attacker tampers payment-amount in transit | TLS 1.2+; signed JWT contains user_id, server reads amount from DB by plan_id (not request body) | Low |
| R | User claims they didn't authorize a charge | Audit log captures user_id, ip, ua, timestamp on every charge intent | Medium — needs alert on unusual patterns |
| I | Logs leak card data | PII filter strips card-shaped patterns; only token IDs logged | Low |
| D | Attacker floods endpoint to exhaust Razorpay rate limit | Rate limit per user (10/min); per IP fallback | Low |
| E | Customer escalates to admin via crafted role claim | Role assignment server-side only; JWT roles re-verified per request | Low |

#### Example: `Postgres (payments)` data store

| STRIDE | Threat | Mitigation | Residual |
|---|---|---|---|
| S | App connects with wrong credentials | Per-service DB user; password from secret manager | Low |
| T | Direct DB modification by SRE outside of audit | RBAC: only on-call has prod access; access JIT-granted; all queries logged | Medium |
| R | DB admin makes change, denies it | All admin queries logged with `pg_audit`; logs to immutable storage | Low |
| I | Backup leaks PAN | We don't store PAN; backups encrypted in any case | Low |
| D | Single DB → outage | Multi-AZ replica; tested failover | Low |
| E | App role can grant itself superuser | Role separation: `app_user` cannot GRANT | Low |

### 3. Score and prioritize

DREAD is a scoring framework but mostly deprecated. Practical: **likelihood × impact** on a 1–3 scale = priority.

```
Risk = max(STRIDE-class likelihood × impact)
```

Track in a register:
| ID | Threat | Likelihood | Impact | Mitigation owner | Status |
|---|---|---|---|---|---|
| TM-001 | Race condition allows double-refund | Medium | High | @payments | In progress |
| TM-002 | Webhook replay attack | Low | Medium | @platform | Mitigated (idempotency) |
| TM-003 | Privilege escalation via role bug | Low | High | @security | Mitigated (server-side check) |

### 4. Mitigate or accept

For each finding, three outcomes:
- **Mitigate** — engineering ticket, deadline, owner.
- **Accept** — risk is below threshold; documented with rationale.
- **Transfer** — insurance / contract / customer responsibility.

"We'll think about it later" is not an outcome. Accept or mitigate, document the choice.

### 5. Re-model on change

Threat model isn't a one-time artifact. Re-do when:
- New trust boundary (new third-party integration, new exposure).
- New data category (now handling PII / PHI / PAN).
- Architecture change (new service, new data store).
- Major incident.

## Common threats by component

### API endpoints (the most common attack surface)

| Threat | Mitigation |
|---|---|
| IDOR (insecure direct object reference) | Authorization check on every resource access; never trust path params alone |
| Mass assignment | Schema validation; allowlist fields, not denylist |
| Rate-limit bypass | Per-user + per-IP rate limits |
| Injection (SQL, NoSQL, command, LDAP) | Parameterized queries; input validation; never string-concat into queries |
| SSRF (server-side request forgery) | Restrict outbound; deny private IPs; validate URLs |
| XXE (XML external entity) | Disable DTD processing; prefer JSON |
| Replay attacks | Nonces / idempotency keys / timestamp + signature |
| Excessive data exposure | Don't `SELECT *`; explicit field allowlist per endpoint |

### Authentication

| Threat | Mitigation |
|---|---|
| Credential stuffing | Rate limit; CAPTCHA after N failures; breach corpus checks |
| Password spray | Same as above; alert on distributed attempts |
| Session hijacking | HttpOnly + Secure + SameSite cookies; rotate on auth events |
| Token theft (XSS) | CSP; HttpOnly cookies; short-lived access + refresh tokens |
| Token reuse after logout | Server-side session revocation; short JWT lifetimes |

### Webhooks

| Threat | Mitigation |
|---|---|
| Forged webhook | HMAC signature verification with correct secret + raw body |
| Replay attack | Idempotency keys / event ID tracking |
| Webhook DoS | Queue + worker; webhook handler returns 200 fast |

### Payments / billing

| Threat | Mitigation |
|---|---|
| Price tampering | Server-side price lookup; ignore client-supplied amounts |
| Refund-then-chargeback fraud | Velocity checks; manual review for first refunds |
| Bypassing payment by manipulating order state | State machine in DB; transitions enforced |
| Double-fulfillment on retry | Idempotent webhook handler |

### File uploads

| Threat | Mitigation |
|---|---|
| Malicious file (executable, zip-bomb) | Content-type allowlist; size limits; AV scan |
| Path traversal | Generated filenames, never user-controlled |
| Storage abuse / cost attack | Per-user quota; cost alerts |
| XSS via SVG upload | SVG sanitization or store as image only |
| Content disclosure via guessable URL | Signed URLs with expiration; or auth check on download |

## Anti-patterns

- **Threat model that's a generic list copy-pasted** — useless. Must be specific to your DFD.
- **No mitigations or owners** — produces an artifact, not a change.
- **Modeled at design, never updated** — drifts; new threats unaddressed.
- **Scoped only to features being shipped** — misses systemic threats (logging, monitoring).
- **No re-model after major change** — surprises in production.
- **DREAD scores treated as precise** — they're rough heuristics. Ranges, not points.
- **Skipping repudiation threats** — almost everyone does. Audit log is the answer.
- **No external red team / pen test** — your threat model is what you can imagine. External eyes find what you missed.
- **Not threat-modeling third-party integrations** — your threat surface includes your sub-processors.
- **Treating compliance audit as substitute for threat modeling** — audit checks controls; threat modeling identifies what controls you need.

## Verify it worked

- [ ] DFD exists for the system; trust boundaries marked.
- [ ] STRIDE walked for each process, data store, and external entity.
- [ ] Threat register populated with > 0 findings (no system has zero threats).
- [ ] Each finding has likelihood × impact scoring.
- [ ] Each finding has mitigation, owner, and status.
- [ ] High-priority threats are mitigated or formally accepted.
- [ ] Threat model linked from architecture doc / ADR.
- [ ] Re-model triggers documented (when to update).
- [ ] Mitigations are *real* — code/config changes, not aspirational notes.
- [ ] At least one finding led to an actual code change.
- [ ] An external pen test or red team has validated key controls (annually).
