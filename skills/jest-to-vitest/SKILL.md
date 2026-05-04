---
name: jest-to-vitest
description: Migrate a Jest test suite to Vitest — config, mocks, timers, ESM compatibility, snapshot diff, the API differences and global vs imported usage. Use when test runs are slow, when migrating to Vite (Vitest is the natural pair), or when fighting Jest's ESM support.
---

# Migrate Jest → Vitest

Vitest is largely Jest-API-compatible but runs faster (especially with Vite), supports ESM natively, and has better TS integration. Migration is mostly mechanical — the wins are in test speed and ESM compatibility.

## When to use

- Just migrated to Vite (see `webpack-to-vite`) — Vitest is the matching test runner.
- Jest is slow on a large suite (> 30s).
- Jest's ESM support is fighting you.
- You want native TS support without Babel overhead.

## When NOT to use

- React Native projects — Jest is the established choice; jest-expo specifically.
- Heavy Jest-only plugins with no equivalent (rare; check first).
- Brand-new codebase — start with Vitest directly.

## API compatibility (high)

| Jest | Vitest |
|---|---|
| `describe`, `test`, `it`, `expect` | Same; available globally with `globals: true` |
| `beforeEach`, `afterEach` | Same |
| `jest.fn()`, `jest.spyOn()` | `vi.fn()`, `vi.spyOn()` |
| `jest.mock(path)` | `vi.mock(path)` |
| `jest.useFakeTimers()` | `vi.useFakeTimers()` |
| `expect(x).toBe(y)` | Same |
| `toMatchSnapshot()` | Same |
| `toThrow()`, `rejects.toThrow()` | Same |

About 90% of test files don't need any change beyond replacing `jest.` with `vi.` (or importing).

## Migration steps

### 1. Install Vitest

```bash
pnpm remove jest @types/jest babel-jest ts-jest @swc/jest jest-environment-jsdom
pnpm add -D vitest @vitest/ui jsdom @vitest/coverage-v8
```

### 2. Update `package.json` scripts

```json
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest",
    "test:ui": "vitest --ui",
    "test:coverage": "vitest run --coverage"
  }
}
```

### 3. Configure Vitest

For most apps, add to `vite.config.ts`:

```ts
import { defineConfig } from "vitest/config";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  test: {
    globals: true,                       // expose describe/test/expect as globals
    environment: "jsdom",                // for React component tests
    setupFiles: ["./src/test/setup.ts"],
    coverage: {
      provider: "v8",
      reporter: ["text", "html", "lcov"],
      exclude: ["**/*.config.*", "**/types.ts"],
    },
  },
});
```

If you have a non-Vite project, create `vitest.config.ts` separately:
```ts
import { defineConfig } from "vitest/config";
export default defineConfig({
  test: { globals: true, environment: "node" },
});
```

### 4. Replace `jest.` with `vi.`

Either bulk-rename via codemod or migrate per-file:

```ts
// Before
jest.fn(); jest.mock("./db"); jest.useFakeTimers();
// After
vi.fn(); vi.mock("./db"); vi.useFakeTimers();
```

### 5. Setup file

```ts
// src/test/setup.ts
import "@testing-library/jest-dom/vitest";   // or expect-extending lib of choice
import { afterEach } from "vitest";
import { cleanup } from "@testing-library/react";

afterEach(() => cleanup());
```

### 6. TS types

If using globals (`globals: true`):

```json
// tsconfig.json
{
  "compilerOptions": {
    "types": ["vitest/globals"]
  }
}
```

Without globals, import in each test:
```ts
import { describe, test, expect, vi, beforeEach } from "vitest";
```

Importing is more explicit but fights muscle memory; globals are fine in practice.

## API differences to watch

### `vi.mock` hoisting

```ts
// Vitest hoists vi.mock calls to top of file (like Jest).
// But variable references inside the factory CAN'T be hoisted with the mock:
const mockFn = vi.fn();
vi.mock("./db", () => ({ getUser: mockFn }));   // ❌ ReferenceError

// Fix — use vi.hoisted to define mocks alongside the hoist:
const { mockFn } = vi.hoisted(() => ({ mockFn: vi.fn() }));
vi.mock("./db", () => ({ getUser: mockFn }));
```

### Module mocking shape

Jest:
```ts
jest.mock("./db", () => ({
  __esModule: true,
  getUser: jest.fn(),
}));
```

Vitest:
```ts
vi.mock("./db", () => ({
  getUser: vi.fn(),
  default: vi.fn(),                    // for default exports
}));
```

`__esModule: true` isn't needed in Vitest.

### Timers

```ts
// Jest's "modern" timers are default in Vitest
vi.useFakeTimers();
vi.setSystemTime(new Date("2026-01-01"));
vi.advanceTimersByTime(1000);
vi.useRealTimers();
```

### `jest.requireActual` → `vi.importActual`

```ts
vi.mock("./db", async () => {
  const actual = await vi.importActual<typeof import("./db")>("./db");
  return { ...actual, getUser: vi.fn() };
});
```

### Snapshot location

By default, both put snapshots in `__snapshots__/` next to the test. No change.

Inline snapshots:
```ts
expect(result).toMatchInlineSnapshot();
```

Same syntax; output format is similar.

### Coverage tool

Jest uses Istanbul; Vitest defaults to V8. V8 is faster, slightly less granular. For CI compatibility (Coveralls, Codecov), the `lcov` report works the same.

If you specifically need Istanbul: `--coverage.provider=istanbul`.

## React Testing Library

Works identically. The only change: import `'@testing-library/jest-dom/vitest'` (not `'@testing-library/jest-dom'`) for matcher integration in setup.

## MSW (Mock Service Worker)

Works identically. No changes.

## Common gotchas

- **Tests using `jest.requireActual`** — convert to `vi.importActual` (async). Some tests need `await`.
- **Tests with `.babelrc` for Jest only** — Vitest doesn't need Babel; remove if Jest was the only consumer.
- **Tests that assumed Jest's `jest.config.js` rootDir** — Vitest uses Vite's project root. Paths may shift.
- **`jest-resolve` custom resolvers** — port to Vite's resolve config.
- **Watch mode behavior** — Vitest's watch mode is more aggressive (re-runs related tests on every file change). Faster but louder.
- **Test ordering** — both run files in parallel by default. If you have order-dependent tests, that's a test smell — fix the test, don't pin the runner.
- **`describe.skip` + `it.only` interaction** — same semantics in Vitest.
- **Tests asserting on `toBeInTheDocument()` etc.** — make sure you import jest-dom matchers in setup.
- **Custom global setup files** — Vitest's `globalSetup` is a separate option from `setupFiles` (runs once before all tests, vs. before each test file).

## Speed wins

| Suite size | Jest typical | Vitest typical |
|---|---|---|
| < 100 tests | 5–15s | 2–5s |
| 100–1000 tests | 30–120s | 10–30s |
| 1000+ tests | 2–5min | 30–90s |

The bigger the suite, the bigger the win. Watch mode HMR is the killer feature day-to-day.

## Anti-patterns

- **Migrating with `globals: false` AND not adding imports** — every test fails with "describe is not defined." Pick one.
- **Mixing Jest and Vitest in same repo long-term** — pick one.
- **Not running on CI before merging migration** — local pass ≠ CI pass.
- **Keeping Jest config files around as backup** — `jest.config.js` left behind confuses tools.
- **Using `vi.mocked(fn)` without `as Mock`** — TS-compatible patterns matter; `vi.mocked` provides typed mocks.
- **Testing `vi.useFakeTimers` interactions in parallel tests** — timers are per-instance but module mocks aren't. Watch for cross-test pollution.
- **Snapshot updates without review** — `vitest -u` updates all snapshots; review the diff.
- **Running coverage only in CI** — local `--coverage` is fast enough now.
- **Configuring `pool: 'forks'`** unless you need it — default `threads` is faster.

## Verify it worked

- [ ] `pnpm test` runs Vitest, not Jest.
- [ ] Test count matches before and after migration (no silently-skipped tests).
- [ ] Total runtime is materially faster than Jest baseline.
- [ ] Watch mode reruns affected tests in < 200 ms.
- [ ] Coverage report is generated; `lcov` consumed by your CI service.
- [ ] No `jest.config.*` or Jest-only Babel config remains.
- [ ] All `jest.` references replaced with `vi.`.
- [ ] Snapshots match (no false positives from differing snapshot format).
- [ ] CI green; coverage threshold maintained.
- [ ] Devs report better feedback loop (HMR speed, error messages).
