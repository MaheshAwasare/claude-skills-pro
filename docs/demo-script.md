# Demo recording script (Loom)

A 90-second walkthrough that demonstrates the value. Three takes — pick the cleanest. Loom auto-captions; speak clearly.

## Setup (do once)

- Clean Claude Code window. Good font size (16pt+).
- Two terminal tabs side by side: **Claude Code** | **`tree skills/` output**.
- Have these phrases ready to type:
  1. `scaffold a saas app for indian b2b customers with razorpay billing and brevo email`
  2. `the test for chargeCustomer fails intermittently — find the real bug`
  3. `audit my repo against dpdp act 2023`

## Script (90 seconds)

### 0:00 — 0:15 — Hook
> "Anthropic's Claude Code skills are great, but if you're shipping products in India, doing compliance work, or starting new SaaS apps from scratch, there are gaps. I built **claude-skills-pro** — 51 production-grade skills filling those gaps."

[Show repo README on screen.]

### 0:15 — 0:35 — Install
> "Installation is one symlink line. Skills auto-load when Claude Code starts."

[Run the symlink command. Show `ls ~/.claude/skills/` with 51 entries.]

### 0:35 — 1:05 — The signature demo (pick ONE)

**Option A — scaffold-saas-starter:**
> "Here's a multi-tenant SaaS scaffolder with Postgres RLS, Clerk, Razorpay, and Brevo wired in. Watch."
[Type the scaffold prompt. Show the directory tree appearing. Highlight `withTenant` RLS helper. Highlight `templates.ts` for Brevo.]

**Option B — find-real-bug:**
> "Here's the meta-skill that changes how Claude debugs. It refuses to wrap a `try/catch` until it finds the root cause."
[Show a failing test. Watch Claude trace the bug, refuse the easy fix, propose a real one.]

**Option C — india-dpdp-act:**
> "And compliance. India's DPDP Act 2023 has 250-crore rupee penalties. This skill maps every requirement to engineering controls."
[Run the audit prompt. Show the gap report.]

### 1:05 — 1:25 — Range
> "There are 51 skills total — 22 platform integrations, 15 workflow patterns, 7 framework migrations, 7 compliance frameworks. Each has anti-patterns, code examples, and a verification checklist."

[Quick scroll through README's roadmap section. Pause on the table.]

### 1:25 — 1:30 — CTA
> "MIT licensed. Link in description. Star the repo if it saves you a week."

## Recording tips

- **Don't tab away from your demo to your browser/Slack.** Pause Loom; cut.
- **Keep the cursor still** unless pointing at something specific.
- **Re-record the 0:00 hook** until it lands. The first 8 seconds decide retention.
- **End with the CTA on screen for 2 full seconds** so the viewer's eye lands on it.

## After recording

- Copy the Loom URL.
- In `README.md`, replace the `Loom walkthrough coming soon` line with:
  ```markdown
  > **Demo:** [![Watch the 90-second walkthrough](docs/demo-thumbnail.png)](https://www.loom.com/share/YOUR_VIDEO_ID)
  ```
- Take a clean screenshot of one moment from the video, save as `docs/demo-thumbnail.png`.
- Commit + push.

## Alternative: animated terminal GIF

If recording yourself feels heavy, record a `terminalizer` or `asciinema` session:

```bash
# asciinema (lighter, no video)
asciinema rec demo.cast
# play locally; if good:
asciinema upload demo.cast

# Or export as GIF via agg:
agg demo.cast docs/demo.gif
```

Then in README:
```markdown
![Demo](docs/demo.gif)
```

GIFs are stickier on GitHub (auto-play in the README) than Loom thumbnails (need a click).
