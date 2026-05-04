---
name: find-dead-code
description: Locate unused code with confidence tiers — definitely dead, probably dead, risky to touch. Uses static analysis (knip, ts-prune, ruff, deadcode) plus reflection-aware heuristics for things tools miss (DI, JSON config, dynamic imports). Use during periodic cleanup, before a refactor, or when onboarding to figure out what's still load-bearing.
---

# Find Dead Code

Dead code is the silent tax — slow tests, larger bundles, mental load, security surface. But "delete it" is risky: code that *looks* dead can be reflected, dynamically imported, called via DI, or referenced from JSON config. This skill is the methodology for high-confidence deletion.

## When to use

- Periodic cleanup (quarterly).
- Before a major refactor — know what you can safely remove.
- Onboarding a new engineer — show them what's still alive.
- Bundle size investigation — find what's shipping that nobody uses.
- Security audit — fewer code paths is fewer attack surfaces.

## When NOT to use

- During active feature development — defer until things stabilize.
- In test code — dead tests are a different audit (see `audit-tautological-tests`).
- Highly dynamic codebases (heavy plugin systems, Ruby on Rails magic) — false positives dominate; manual review only.

## Three confidence tiers

| Tier | Definition | Action |
|---|---|---|
| **Definitely dead** | Static analyzer says unused; no string match in repo; not exported from package. | Delete. Open PR. |
| **Probably dead** | Static analyzer says unused, but might be referenced by reflection, DI, JSON, dynamic imports. | Comment out + smoke test + delete next sprint if no break. |
| **Risky** | Old code with no recent changes, but maybe still called from a rarely-run path. | Add deprecation log + observe metrics + delete after a release window. |

## Tier 1: definitely dead

Static analyzers find these:

| Language | Tool | What it finds |
|---|---|---|
| JS/TS | `knip` | unused files, exports, dependencies, devDeps, types |
| TS | `ts-prune` | unused exports |
| Python | `vulture`, `ruff` (`F401`, `F841`) | unused imports, unused variables, unused functions |
| Go | `staticcheck`, `unused` | unused functions, unused fields, unreachable code |
| Rust | `cargo-machete`, `cargo udeps` | unused deps |
| Java | IntelliJ inspections, `error-prone` | unused methods, unused params |

```bash
# JS/TS — the killer
pnpm dlx knip
```

```text
Unused files (3):
  src/legacy/old-importer.ts
  src/utils/deprecated-formatter.ts
  src/components/UnusedModal.tsx

Unused exports (12):
  src/api/client.ts: unusedHelper, anotherOne
  ...

Unused dependencies (2):
  lodash (replaced by built-ins)
  moment (replaced by date-fns)
```

For each finding, verify with grep:
```bash
grep -r "deprecated-formatter" --exclude-dir=node_modules .
```

If grep finds zero references and the file isn't an entry point, it's tier 1. Delete.

## Tier 2: probably dead

Things static analyzers miss:

- **DI containers** — `register('userService', UserService)`. The string registration looks "unreferenced" to the analyzer but is invoked at runtime.
- **JSON config refs** — webpack/Vite plugin lists, `package.json` script targets, `tsconfig.json` paths.
- **Dynamic imports** — `import(\`./plugins/${name}\`)`. Static analysis can't follow.
- **Reflection** — Spring `@Component`, Angular `@Injectable`, Python `getattr`, JS `obj[methodName]()`.
- **Test fixtures referenced by name** — `loadFixture("user-with-three-orgs")`.
- **Public API surface** — exported from a published package; consumers you don't see.
- **CLI commands** — registered as a subcommand by string name.
- **Event handlers** — subscribed via name.

For these, the safe sequence:

1. **Search broadly** — grep for the name across the repo, with case-insensitivity:
   ```bash
   grep -ri "deprecatedFormatter" --include="*.ts" --include="*.json" --include="*.yaml" .
   ```
2. **Search outside the repo** — published package? Check downstream consumers; npm `package-lock`s; internal monorepos.
3. **Comment-out, don't delete** — leave the code in but commented; deploy. Wait a week (or a release cycle).
4. **Watch logs** — if an alert fires, you found a hidden caller. Restore.
5. **If silent, delete** — confidence is now high.

## Tier 3: risky

Old code paths that are *probably* dead but were once live. Examples:

- An admin endpoint used once a quarter for a manual data fix.
- A migration helper for a long-finished migration.
- A fallback for an old client version.

For these, **add observability before deletion**:

```ts
// Tier 3: probably dead but unsure
export function legacyExportV1(req, res) {
  log.warn("legacyExportV1 called", {
    user: req.user?.id,
    userAgent: req.headers["user-agent"],
    ip: req.ip,
  });
  // ... actual code
}
```

Deploy, wait a release window (2–4 weeks), check logs. If zero invocations, delete with confidence.

For old client-version fallbacks: increment metric, alert if non-zero after the deprecation window.

## The cleanup PR shape

Don't delete 50 things in one PR. Bundle by *concept*:

- One PR per dead feature.
- One PR for unused dependencies.
- One PR for dead utility functions in one module.

Each PR is small enough that a reviewer can verify each deletion. Mass deletes hide one accidental load-bearing item.

```
chore(cleanup): remove legacy v1 invoice exporter

Tier 1: knip flagged. grep confirms no remaining callers in main repo
or downstream packages. Last commit on these files: 2024-08-12 (#1234).

- src/exporters/invoice-v1.ts (deleted)
- src/exporters/invoice-v1.test.ts (deleted)
- Tests passing.
```

## What to look for during PR review of a delete

- Was a string match in JSON / YAML / config done?
- Is this code in a published package's public API?
- Is there a test that *tests against the deletion target*? Delete that test too.
- Is this referenced via DI / reflection / dynamic import?
- Does removing this require any DB cleanup (e.g. cron job rows)?

## Tools roundup

| Tool | Language | Notes |
|---|---|---|
| `knip` | JS/TS | Best-in-class; covers files/exports/deps |
| `ts-prune` | TS | Older, narrower; knip subsumes |
| `depcheck` | JS/TS | Just deps |
| `madge --circular` | JS/TS | Find circular deps; partial cleanup signal |
| `vulture` | Python | Finds unused functions/classes; tune confidence |
| `ruff` | Python | Catches unused imports/vars on every save |
| `staticcheck` | Go | Part of standard tooling |
| `unused` | Go | Standalone unused-detector |
| `cargo udeps` | Rust | Unused crates (requires nightly) |
| `cargo-machete` | Rust | Unused deps; stable Rust |
| IntelliJ inspections | Java | Right-click → Analyze → Unused Declarations |

## Anti-patterns

- **Deleting based on "I don't recognize this"** — gut isn't a static analyzer. Run the tools.
- **Deleting before searching outside the file/module** — DI, config, plugins reference by name.
- **Mass-delete PR with 50 deletions** — reviewer can't verify each. Split.
- **Deleting test fixtures referenced by name** — silent test breakage. Search by string content.
- **Deleting code without a deprecation log first** for tier 3 — get evidence before removing.
- **Deleting in a published package without a major version bump** — breaks downstream.
- **Believing high coverage means everything is reachable** — coverage measures the path tests exercise; rare prod paths aren't tested either.
- **Deleting comment-blocks of "old code preserved for reference"** without checking git history — history is your reference; delete.
- **Keeping `// TODO: remove this` in code** — same problem inverted. Either remove now or open an issue with a date.

## Verify it worked

- [ ] A static analyzer ran and produced findings (knip, vulture, staticcheck, etc).
- [ ] Findings sorted into tiers 1/2/3.
- [ ] Tier 1 deletions are PRs that reviewers can verify in <10 min each.
- [ ] Tier 2 has commented-out code with a date; revisit calendar entry exists.
- [ ] Tier 3 has deprecation logging; observability is in place.
- [ ] No deletion broke something silently — CI green, smoke tests pass.
- [ ] Repo size / bundle size measurable improvement after cleanup.
- [ ] PR descriptions document the analyzer's flag and the search done before delete.
- [ ] Cleanup is repeatable — a quarterly run produces a clean report (means no new dead code, or you're removing it as you go).
