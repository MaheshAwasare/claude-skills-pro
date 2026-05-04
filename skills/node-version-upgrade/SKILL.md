---
name: node-version-upgrade
description: Upgrade Node.js (16 → 18 → 20 → 22) — what breaks, what improves, ESM transitions, dependency compatibility, Docker base image swaps, CI matrix updates. Use when a Node version is going EOL or you want native fetch/test runner. Covers Node 16 EOL (2023), 18 EOL (2025), 20 LTS, 22 LTS.
---

# Upgrade Node.js

Node has a 6-month release cadence and a clear LTS policy. Falling behind is technical debt that compounds with each missed version. This skill is the upgrade path with concrete fixes for the breakages you'll hit.

## When to use

- Current Node version is past or near EOL.
- Want native `fetch`, `test`, `--watch`, `--env-file`.
- Dependency requires a newer Node (e.g. Vite 5 dropped Node 16).
- Cleaning up workarounds that pre-date a feature now in core (`node-fetch`, `dotenv`, etc).

## When NOT to use

- Two upgrades back-to-back (16 → 22) on a large app — do 16 → 18 → 20 → 22 incrementally.
- Frozen environment (regulated, can't deploy without recertification) — coordinate with platform team.

## EOL dates (the calendar)

| Node | LTS start | EOL |
|---|---|---|
| 16 | Oct 2021 | Sep 2023 (past EOL) |
| 18 | Oct 2022 | Apr 2025 (past EOL) |
| 20 | Oct 2023 | Apr 2026 |
| 22 | Oct 2024 | Apr 2027 |
| 24 | Oct 2025 | Apr 2028 |

**Rule:** target the latest active LTS. As of this skill's writing, that's 22.

## What changed at each major

### 16 → 18

| Change | Impact |
|---|---|
| Built-in `fetch` (Undici) | Drop `node-fetch`. |
| Built-in `node:test` | Drop `mocha` for many cases (or migrate to vitest). |
| OpenSSL 3 | Some legacy crypto patterns fail (`EVP_DecodeUpdate` deprecation). Set `--openssl-legacy-provider` as escape hatch. |
| `esModuleInterop` interaction | TS 4.x compatibility — may need TS bump. |

Common breakage: `node-fetch@2` is CJS, doesn't tree-shake; `node-fetch@3` is ESM-only (forces ESM transition). Replacing with native `fetch` removes the issue.

### 18 → 20

| Change | Impact |
|---|---|
| `--watch` mode | Drop `nodemon` for simple cases. |
| Stable `node:test` | Drop test runners for libraries. |
| V8 update | Mild perf changes. |
| `URL` parser stricter | Code that relied on lax parsing fails. |
| Permission model (experimental) | New flag `--experimental-permission`. |

Common breakage: `URL` constructor used to coerce malformed inputs; now throws. Audit URL parsing in user-input paths.

### 20 → 22

| Change | Impact |
|---|---|
| `require(esm)` (experimental in 22) | Can `require()` ESM modules. Big for slow ESM migrations. |
| Built-in WebSocket | Drop `ws` for client use cases. |
| V8 12 | Improved perf, especially regex. |
| `--env-file=.env` is stable | Drop `dotenv` for simple loads. |
| `glob()` built-in | Drop `glob` package. |

Less breaking than 18→20. Mostly additive features.

## The upgrade workflow

### 1. Read the migration guide for each version

The Node project publishes migration guides. Read them. Skim the deprecations.

### 2. Update `engines` in `package.json`

```json
{ "engines": { "node": ">=20.0.0 <23.0.0" } }
```

This warns devs and CI on mismatch. Some PaaS providers (Vercel, Railway) read this for runtime selection.

### 3. Bump the Docker base image

```dockerfile
FROM node:22-slim AS deps          # was 18-slim
```

Use `-slim` (Debian, smaller) or `-alpine` (musl, tiniest, sometimes incompatible with native modules) — *not* `node:22` (full Debian, fat).

### 4. Update CI matrix

```yaml
# .github/workflows/ci.yml
strategy:
  matrix:
    node: [20, 22]                 # drop older
```

Test against the *next* LTS too if you can — catches issues before forced upgrade.

### 5. Run tests on each target version locally

```bash
nvm use 22
pnpm install
pnpm test
pnpm build
```

Track what fails. Tests, not just type-checks.

### 6. Audit deprecation warnings

```bash
node --trace-deprecation src/index.js 2>&1 | grep -i deprecat
```

Even if your tests pass, warnings that emit on every request are a sign you'll break next version.

### 7. Audit dependencies for compatibility

```bash
pnpm install
# Read peerDependencies warnings
# Check changelogs for major deps
```

Common: `prisma`, `@nestjs/*`, `@aws-sdk/*` need bumping. Pin known-good versions before upgrading.

### 8. Audit native modules

```bash
pnpm rebuild
# Failures here = native module needs prebuilt or rebuild for Node 22
```

`bcrypt`, `sharp`, `better-sqlite3`, `node-gyp`-built modules can break. Often a version bump fixes; sometimes need to migrate (e.g. `bcrypt` → `bcryptjs` for portability).

### 9. Handle ESM if not already

Node 22 + modern deps push you toward ESM. Many libraries are now ESM-only:

```json
{ "type": "module" }
```

This switches your project to ESM. Consequences:
- `require` no longer works — convert to `import`.
- File extensions in imports become required (`./foo.js`, not `./foo`).
- `__dirname` / `__filename` need polyfills:
  ```js
  import { fileURLToPath } from "node:url";
  const __filename = fileURLToPath(import.meta.url);
  const __dirname = path.dirname(__filename);
  ```

If you can't migrate yet, the `require(esm)` flag in Node 22 lets some ESM libs load from CJS. But ESM is the direction.

### 10. Deploy progressively

- Local dev → Staging → Canary 5% → Full rollout.
- Watch error rate on canary for 24h before full rollout.
- Have a rollback plan (see `plan-the-rollback`) — usually means redeploying the previous container image.

## Native fetch migration (the most common cleanup)

```js
// Old
const fetch = require("node-fetch");
const res = await fetch(url);

// New
const res = await fetch(url);                  // fetch is global
```

If you used `node-fetch` Response shape extensions (`.buffer()`), they map to `arrayBuffer()` in native fetch. `import { Headers, Request, Response }` from `undici` if you need explicit imports.

## Common breakages

| Symptom | Cause | Fix |
|---|---|---|
| `error:0308010C:digital envelope routines::unsupported` | OpenSSL 3 strict mode | `NODE_OPTIONS=--openssl-legacy-provider` (escape) or update crypto |
| `Cannot find module './foo'` | ESM requires `.js` extension | Add extensions to imports |
| `Cannot use import statement outside a module` | CJS file using ESM syntax | Add `"type": "module"` or rename `.mjs` |
| `require is not defined in ES module scope` | ESM context | Use `import`; or `import.meta` polyfill for `require` |
| `Failed to load native binding` | Native module not rebuilt | `pnpm rebuild` or bump the dep |
| `URL constructor: <bad-url> is not a valid URL` | Stricter URL parser | Validate input before passing |

## Anti-patterns

- **Skipping LTS versions** — 16 → 22 directly is harder than 16 → 18 → 20 → 22.
- **Upgrading Node and major deps in one PR** — too many variables. Upgrade Node first, then deps.
- **Pinning `engines: "*"`** — defeats the purpose. Pin a real range.
- **No CI matrix on the new version** — forced upgrade reveals unknown breakages.
- **Ignoring deprecation warnings** — they become errors next major.
- **`--openssl-legacy-provider` as a long-term solution** — buys time only; fix the underlying crypto.
- **Production canary skipped** — Node upgrades have surprised every team that didn't canary.
- **Using `nvm-windows`** in CI — flaky. Use `actions/setup-node` directly.
- **Native modules without explicit version pin** — implicit upgrades break unpredictably.
- **Fork-rolling-your-own polyfills** instead of using a maintained one — eternal maintenance.

## Verify it worked

- [ ] `node -v` matches the target version on every developer machine and in CI.
- [ ] `package.json` `engines.node` is set to a real range.
- [ ] Docker base image bumped; image rebuilds cleanly.
- [ ] CI runs tests against target version (and ideally also +1 LTS).
- [ ] `node --trace-deprecation` shows no warnings from your code.
- [ ] `pnpm install && pnpm rebuild` is clean — no native module failures.
- [ ] All tests pass.
- [ ] Production canary at 5% for 24h shows flat error rate vs. baseline.
- [ ] `node-fetch`, `node-test`, `dotenv` (if not used for advanced features), `glob` are removed where native equivalents replace them.
- [ ] Old Node version base images are removed from CI/prod artifacts (no fallback).
