---
name: bun-runtime
description: Use Bun as runtime, package manager, bundler, and test runner — when to choose Bun over Node, the Node.js compatibility gotchas, migration from npm/pnpm, the built-in HTTP server, SQLite, and Bun's built-in test runner. Use when starting a new TS/JS project or evaluating a switch from Node.
---

# Bun Runtime

Bun is a Node-compatible JS runtime, package manager, bundler, and test runner — all one binary, written in Zig. Faster than Node + pnpm + tsc + vitest combined for most workloads. Trade-off: 95% Node compatibility, not 100%.

## When to use

- New TS/JS project — install speed alone is worth it.
- Build/test bottleneck on your CI.
- Edge-deploy compatible (Cloudflare Workers / Vercel) — Bun runs on Vercel as of 2025.
- CLI tools — `bun build --compile` produces single-binary executables.

## When NOT to use

- Production runtime for a Node app with deep ecosystem deps you haven't tested.
- Apps using native modules that haven't been ported (rare; check your `node_modules`).
- Any case where "must be 100% Node identical" is a hard requirement.

## What Bun replaces

| Tool | Replaced by |
|---|---|
| Node | `bun run` (runtime) |
| npm/pnpm/yarn | `bun install` |
| ts-node / tsx | Native — Bun runs `.ts` files directly |
| nodemon | `bun --hot` |
| webpack/esbuild/vite (for libs) | `bun build` |
| jest/vitest | `bun test` |
| dotenv | Built-in `.env` loading |

## Migration from Node + pnpm

```bash
# In existing repo
bun install                              # reads package.json, replaces node_modules
bun pm ls                                # verify deps installed

# Run scripts
bun run dev                              # uses package.json scripts
bun run build

# Replace runtime
node server.js  →  bun server.js
ts-node script.ts  →  bun script.ts      # no compile step

# Replace tests
vitest  →  bun test                      # if you used vitest's globals
```

`package.json` scripts work as-is. `bun install` produces a `bun.lockb` (binary lockfile); commit it; delete `pnpm-lock.yaml` / `package-lock.json` when fully migrated.

## Built-in HTTP server (zero deps)

```ts
// server.ts
const server = Bun.serve({
  port: 3000,
  async fetch(req) {
    const url = new URL(req.url);
    if (url.pathname === "/healthz") return new Response("ok");
    if (url.pathname.startsWith("/api/")) return handleApi(req);
    return new Response("not found", { status: 404 });
  },
});

console.log(`Listening on ${server.port}`);
```

`Bun.serve` is roughly 4x faster than Express for raw throughput. It speaks the standard `Request`/`Response` Web API — code is portable to Cloudflare Workers and Deno.

## Built-in SQLite

```ts
import { Database } from "bun:sqlite";
const db = new Database("app.sqlite");
db.run("CREATE TABLE IF NOT EXISTS users (id INTEGER PRIMARY KEY, email TEXT)");
const insert = db.prepare("INSERT INTO users (email) VALUES (?)");
insert.run("a@b.com");
const rows = db.query("SELECT * FROM users").all();
```

Faster than `better-sqlite3`, no native compile step, ships with the runtime. Replaces a lot of "small persistent state" use cases for CLI tools.

## Built-in test runner

```ts
// users.test.ts
import { describe, expect, test } from "bun:test";
import { greet } from "./users";

describe("greet", () => {
  test("returns hello", () => {
    expect(greet("Mahesh")).toBe("Hello, Mahesh");
  });
});
```

```bash
bun test                                # run all *.test.ts
bun test --coverage                     # built-in coverage
bun test --watch
bun test users                          # name filter
```

Jest-compatible API. Migration from Vitest is mostly imports.

## `.env` loading (built-in)

`Bun automatically loads .env, .env.local, .env.<NODE_ENV>`. No `dotenv` package. `.env.local` overrides `.env`; both are loaded before your app code runs.

## Building a single-binary CLI

```bash
bun build --compile --target=bun-linux-x64 --outfile=mycli ./src/cli.ts
```

Produces a 50–80MB binary that includes the runtime. No `node` install needed on target machine. Cross-compile for darwin-x64, darwin-arm64, linux-x64, linux-arm64, win-x64.

## Performance reality check

- `bun install` is 5–25x faster than `npm install`, 2–10x faster than `pnpm`.
- `bun run` vs `node` runtime is 1.1–1.5x faster on typical Node workloads (not 3x like marketing claims).
- `bun build` is comparable to esbuild.
- `bun test` is 2–5x faster than vitest.

The compounded CI win (install + test + build) is the real story. A 4-min CI becomes 90s.

## Compatibility gotchas

| Issue | Workaround |
|---|---|
| Some `node:` modules incomplete (`node:cluster`, `node:vm`) | Check `bun --print 'process.versions.bun'` and feature support before relying |
| Native modules requiring node-gyp | Most work; some need rebuilding. `bun pm trust <pkg>` |
| `npm`-only postinstall hooks | Bun doesn't run lifecycle scripts by default — `bun install --trust` |
| Differences in `process.env` mutation | Bun freezes `process.env` after start in production; mutate before serving |
| Worker threads API surface | Mostly works; some edge cases differ |

Run your Node test suite under Bun before committing to it as runtime.

## Edge deploy

Cloudflare Workers: standard Worker syntax (no Bun APIs); use Bun for `bun build` step in CI.
Vercel: now supports Bun directly as runtime; specify in `vercel.json` `"build": { "env": { "BUN_RUNTIME": "1" } }` or use `bun install` in build step.

## Anti-patterns

- **Mixing `bun install` and `npm install` in the same repo** — divergent lockfiles, irreproducible builds. Pick one.
- **Committing `node_modules` "for safety"** — lockfile + `bun install` is reproducible.
- **Using `bun:` imports in code that may run on Node** — breaks portability. Either commit fully or feature-detect.
- **Production runtime without testing the actual Node deps** — 5% incompatibility usually means 1–2 packages bite. Test before betting prod on it.
- **Relying on `bun build` for production library bundling without testing the output** — output differs from Vite/Rollup in subtle ways for libs.
- **Using `Bun.serve` then trying to deploy to Lambda** — Lambda expects a handler, not a server. Use Hono or similar abstraction.
- **`bun --hot` in production** — dev-only. `bun run` for prod.
- **Running `bun test` for end-to-end browser tests** — Bun doesn't have a browser. Use Playwright separately.
- **Skipping `bun test --coverage` because "it might miss edges"** — it works fine; the coverage tool is honest about what it sees.

## Verify it worked

- [ ] `bun install` in a fresh clone is < 5s for a typical app.
- [ ] `bun test` passes the same suite that worked under Node + vitest.
- [ ] `bun run build` produces the expected output.
- [ ] All native deps work (`bun pm ls --all` shows no errors).
- [ ] Production app under `bun server.js` handles the same load as `node server.js` without crashing.
- [ ] CI time is materially shorter than the Node baseline.
- [ ] `.env.local` overrides `.env` automatically (no `dotenv` import).
- [ ] Single-binary `bun build --compile` runs on a target machine without Node installed.
- [ ] `bun --hot` reloads the server on file change.
- [ ] No mixing of `npm install` / `bun install` — only `bun.lockb` exists.
