---
name: wcag-accessibility-audit
description: Audit a web app against WCAG 2.2 AA — automated tooling (axe, Pa11y, Lighthouse), the manual checks tools miss (keyboard, screen reader, focus, color, motion), and the engineering fixes. Use when adding accessibility to a product, before public launch, after legal pressure, or to satisfy procurement requirements.
---

# WCAG 2.2 AA Audit

WCAG (Web Content Accessibility Guidelines) 2.2 AA is the de facto baseline. Required for US public sector (Section 508), EU public sector (EN 301 549), and increasingly demanded by enterprise procurement. Lawsuits (ADA in the US) drive private-sector compliance. This skill is the engineer's audit + fix list.

## When to use

- Pre-launch accessibility review.
- Procurement / VPAT request from an enterprise customer.
- Reactive: ADA demand letter or accessibility complaint.
- Routine quarterly audit of a public product.

## When NOT to use

- Internal-only tools with no external users (still good practice; lower legal pressure).
- Non-web (native mobile, desktop) — different standards (WCAG still applies, but tooling differs).

## The four POUR principles

WCAG organizes around four principles:
- **Perceivable** — users must perceive the content (alt text, captions, contrast).
- **Operable** — users must operate the UI (keyboard, time, no seizures).
- **Understandable** — text + UI predictable and clear.
- **Robust** — works with assistive tech.

AA is the conformance level for legal-defensibility. AAA is stricter; A is too loose.

## The audit pipeline

### 1. Automated scan (catches ~30% of issues)

```bash
# axe-core (most accurate, broad coverage)
pnpm dlx @axe-core/cli https://example.com

# Pa11y (CI-friendly)
pnpm dlx pa11y https://example.com --standard WCAG2AA

# Lighthouse (built into Chrome)
# DevTools → Lighthouse → Accessibility
```

Run on each page / state. Add to CI:
```yaml
- run: pnpm pa11y-ci --sitemap https://staging.example.com/sitemap.xml
```

Automated tools catch:
- Missing alt text.
- Insufficient color contrast (most cases).
- Missing form labels.
- Broken ARIA.
- Missing landmarks.
- Tab order issues (sometimes).

They miss:
- Whether alt text is *meaningful*.
- Keyboard navigation flow correctness.
- Screen-reader announcement quality.
- Focus management on dynamic content.
- Reading order for complex layouts.
- Accessible names for icon buttons.

### 2. Keyboard-only test (essential)

Unplug your mouse. Navigate the entire app:

| Check | Pass criteria |
|---|---|
| Tab to every interactive element | All buttons/links/inputs reachable |
| Focus is visible | Always — clear outline, not removed |
| Tab order is logical | Top-to-bottom, left-to-right (LTR languages) |
| No keyboard traps | Esc closes modals; can tab out of widgets |
| Skip-to-content link | Top-of-page link to main content |
| Modals trap focus | Focus stays in modal until dismissed |
| Modal close on Esc | Required |
| Dropdowns / menus operable via arrow keys | Standard pattern |
| Custom widgets follow ARIA Authoring Practices | Combobox, Listbox, Tabs, etc. |

### 3. Screen reader test

Use **NVDA** (Windows free), **JAWS** (Windows paid, dominant enterprise), or **VoiceOver** (macOS / iOS). Read your app top to bottom.

Listen for:
- Are images announced with meaningful descriptions?
- Are buttons announced with their purpose? (`<button>Submit</button>` good; `<div onClick>X</div>` silent.)
- Are form fields announced with their label?
- Are errors announced when validation fails?
- Are dynamic updates (toast, modal open) announced via `aria-live`?
- Are headings logical (h1 → h2 → h3, not skipping levels)?
- Are landmarks present (`<header>`, `<nav>`, `<main>`, `<footer>`)?

### 4. Color and contrast

| Element | Required ratio (AA) |
|---|---|
| Body text (small) | 4.5:1 |
| Large text (≥ 18pt or 14pt bold) | 3:1 |
| UI components (button borders, focus indicator) | 3:1 |
| Decorative images | N/A |

Use tools like Stark (Figma plugin), WebAIM Contrast Checker, or DevTools color picker.

**Don't rely on color alone** — error messages need both red color *and* an icon or text. Color-blind users get only the second signal.

### 5. Motion and animation

- Auto-playing content (carousel, video) must be pausable.
- No flashing > 3 times per second (seizure risk).
- Respect `prefers-reduced-motion`:
  ```css
  @media (prefers-reduced-motion: reduce) {
    *, *::before, *::after {
      animation-duration: 0.01ms !important;
      transition-duration: 0.01ms !important;
    }
  }
  ```

### 6. Forms

- Every input has a `<label>` (visible or `aria-label`).
- Required fields marked with both `required` and visible asterisk + text.
- Errors announced (`aria-invalid`, error message in `aria-describedby`).
- Don't disable submit until valid — let user submit and show errors.
- Don't auto-submit on input; use explicit submit.

```html
<!-- Bad -->
<input placeholder="Email" />

<!-- Good -->
<label for="email">Email <span aria-hidden="true">*</span><span class="sr-only">(required)</span></label>
<input id="email" type="email" required aria-describedby="email-error" />
<div id="email-error" role="alert"><!-- populated on error --></div>
```

### 7. Common framework gotchas

| Framework | Issue |
|---|---|
| React | `<div onClick>` instead of `<button>` — not focusable, not keyboard-operable |
| Tailwind | `outline-none` removed without focus replacement — visible focus disappears |
| Component libraries (older Material UI, Bootstrap modals) | Pre-WAI-ARIA patterns — verify; modern libs (Radix, Headless UI) are better |
| Next.js | `<Link>` is a button-like; `passHref` only matters for legacy patterns now |
| Modal libraries | Some don't trap focus or restore focus on close |

Use accessible primitives: **Radix UI**, **Headless UI**, **React Aria** by Adobe. They handle the WAI-ARIA patterns correctly. Don't roll your own dropdown / dialog / combobox.

### 8. Images

```html
<!-- Decorative — empty alt -->
<img src="hero-bg.jpg" alt="" />

<!-- Informative — descriptive -->
<img src="chart.png" alt="Revenue Q3 2025: $1.2M, up 15% from Q2" />

<!-- Functional (button-image) — describe action -->
<button><img src="trash.svg" alt="Delete project" /></button>

<!-- Same-page text already describes — empty alt -->
<a href="/profile">
  <img src="avatar.jpg" alt="" />
  <span>Profile</span>
</a>
```

The most common bug: alt="image" or alt="photo" — useless. Either describe the content or use `alt=""` for decorative.

### 9. Tables

```html
<table>
  <caption>Quarterly revenue 2025</caption>
  <thead>
    <tr><th scope="col">Quarter</th><th scope="col">Revenue</th></tr>
  </thead>
  <tbody>
    <tr><th scope="row">Q1</th><td>$800k</td></tr>
    <tr><th scope="row">Q2</th><td>$1.0M</td></tr>
  </tbody>
</table>
```

`scope="col"` and `scope="row"` make screen reader announcements much clearer.

### 10. Dynamic content

```html
<!-- Toast / notification -->
<div role="status" aria-live="polite">Settings saved.</div>

<!-- Critical alert -->
<div role="alert" aria-live="assertive">Error: payment declined.</div>

<!-- Loading state -->
<div role="status" aria-busy="true">Loading...</div>
```

`aria-live="polite"` — announces when screen reader is idle. `assertive` — interrupts. Use polite by default.

## VPAT (Voluntary Product Accessibility Template)

Enterprise procurement asks for a VPAT — a document mapping your product to WCAG / Section 508 criteria. Generate it from your audit. Honest VPATs note "Partially Supports" with explanation; lying gets caught.

## Anti-patterns

- **Trusting Lighthouse 100% score** — Lighthouse catches ~30%; the rest needs human + screen reader.
- **Adding `tabindex="0"` to non-interactive divs** — makes them focusable but not operable. Use `<button>`.
- **`tabindex="-1"` on interactive elements** — removes them from tab order silently.
- **`outline: none`** without alternative — focus invisible.
- **Carousels that auto-advance with no pause** — WCAG fail.
- **Color as the only signal** for state (red/green) — fail for color-blind.
- **`<a href="#" onClick>` instead of `<button>`** — semantic mismatch.
- **Modal that doesn't trap focus or restore it on close** — keyboard user gets stranded.
- **Placeholder as the label** — disappears on focus; screen readers may skip.
- **`role="button"` on a `<div>`** — re-implementing what `<button>` already does, missing keyboard support.
- **Skip alt text on real images** — most common failure.
- **Skip captions on videos** — fail for deaf/hard-of-hearing.
- **Arial-* attributes mis-used** ("aria-haspopup": true on a list that doesn't have a popup) — worse than no aria.

## Verify it worked

- [ ] axe + Pa11y + Lighthouse all green or document any "won't fix" with reason.
- [ ] Keyboard-only test: complete all primary user flows without mouse.
- [ ] Focus indicators visible everywhere.
- [ ] No keyboard traps (Esc closes modals; Tab eventually exits).
- [ ] Screen reader test: complete sign-up + checkout + settings change.
- [ ] All form inputs have associated labels.
- [ ] All form errors announced.
- [ ] All non-decorative images have meaningful alt.
- [ ] Color contrast checked: body text ≥ 4.5:1; large text + UI ≥ 3:1.
- [ ] `prefers-reduced-motion` respected.
- [ ] No flashing > 3 Hz.
- [ ] Heading structure logical (h1 → h2 → h3, no skips).
- [ ] Landmarks present (`<header>`, `<nav>`, `<main>`, `<footer>`).
- [ ] VPAT prepared if customers ask.
- [ ] Accessibility added to CI (axe/pa11y in PR pipeline).
- [ ] At least one user with assistive tech experience has tested key flows.
