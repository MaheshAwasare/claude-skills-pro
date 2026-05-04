---
name: webpack-to-vite
description: Migrate from Webpack to Vite — config translation, plugin equivalents, dev-server differences (HMR), env var handling, asset imports, the issues unique to React/Vue/Svelte migrations. Use when an existing webpack project's dev experience is too slow or maintenance burden too high.
---

# Migrate Webpack → Vite

Vite's dev experience (sub-second cold start, instant HMR) is dramatic compared to webpack. Migration is moderately complex; the key insight is that **Vite is config-by-omission** — most webpack config disappears entirely.

## When to use

- Webpack dev server takes > 10s to start, > 200ms HMR.
- Maintenance of webpack config is a tax — too many plugins, brittle config.
- Project is Vue/Svelte (Vite is more native).
- React project willing to pay one-time migration cost for ongoing dev-time win.

## When NOT to use

- Heavy use of webpack-specific plugins with no Vite equivalent (rare, but check).
- Module Federation (some Vite plugins exist but maturity varies).
- Server-side render with Webpack-managed SSR (use Next.js or Nuxt instead — they handle their own bundler).

## Core differences

| Webpack | Vite |
|---|---|
| Bundles everything in dev | Native ESM in dev, no bundling |
| Loaders for transforms | Plugins (Rollup-compatible) |
| Long config | Convention over config |
| Plugin ecosystem decade-deep | Smaller; usually has the equivalent |
| Source maps slow | Native ESM = browser-resolved sources |
| `process.env.NODE_ENV` | `import.meta.env.MODE` (and `process.env` polyfill) |

## Migration steps

### 1. Install Vite + framework plugin

```bash
pnpm remove webpack webpack-cli webpack-dev-server html-webpack-plugin \
  babel-loader css-loader style-loader file-loader url-loader
pnpm add -D vite @vitejs/plugin-react              # for React
# or @vitejs/plugin-vue, @sveltejs/vite-plugin-svelte
```

### 2. Move `index.html` to project root

Webpack: `src/index.html` referenced via `HtmlWebpackPlugin`.
Vite: `index.html` *at the project root*. It IS the entry point.

```html
<!-- index.html -->
<!doctype html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>My App</title>
</head>
<body>
  <div id="root"></div>
  <script type="module" src="/src/main.tsx"></script>      <!-- entry here -->
</body>
</html>
```

### 3. Create `vite.config.ts`

```ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import path from "node:path";

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: { "@": path.resolve(__dirname, "./src") },        // matches tsconfig paths
  },
  server: {
    port: 3000,
    proxy: {
      "/api": "http://localhost:4000",                       // backend during dev
    },
  },
  build: {
    outDir: "dist",
    sourcemap: true,
  },
});
```

That's a typical config. For most apps, this replaces 200 lines of webpack config.

### 4. Replace `process.env.X` references

Vite exposes env vars via `import.meta.env`. Two options:

**Option A — full migration (preferred):**
```ts
// before
const apiUrl = process.env.REACT_APP_API_URL;
// after
const apiUrl = import.meta.env.VITE_API_URL;
```

Rename `REACT_APP_*` → `VITE_*` in `.env` files.

**Option B — polyfill (for migration):**
```ts
// vite.config.ts
import { defineConfig, loadEnv } from "vite";
export default defineConfig(({ mode }) => {
  const env = loadEnv(mode, process.cwd(), "REACT_APP_");
  return {
    define: {
      "process.env": Object.fromEntries(
        Object.entries(env).map(([k, v]) => [k, JSON.stringify(v)])
      ),
    },
  };
});
```

Use B during the migration; convert to A as you touch files.

### 5. Update import paths

Vite is stricter about imports:

| Issue | Fix |
|---|---|
| `import './styles'` (no extension) for non-JS | Add extension: `import './styles.css'` |
| Importing `.svg` as React component | Use `vite-plugin-svgr` or `?react` query |
| Importing JSON | Works natively |
| `require()` calls | Convert to `import` |

### 6. Asset handling

Webpack:
```ts
import logo from "./logo.png";                             // gets URL via file-loader
```

Vite: same syntax. Built-in. **Add to TypeScript declarations** if TS complains:
```ts
// src/vite-env.d.ts
/// <reference types="vite/client" />
declare module "*.svg?react" {
  import { FC, SVGProps } from "react";
  const C: FC<SVGProps<SVGSVGElement>>;
  export default C;
}
```

### 7. CSS handling

CSS Modules: filename must be `*.module.css`. (Same as Webpack convention with css-loader.)

Sass/Less: `pnpm add -D sass` — Vite picks it up automatically. No config needed.

PostCSS: works via `postcss.config.js` at the root.

Tailwind: works as documented; the v4 PostCSS-less version works particularly well with Vite.

### 8. Plugin equivalents (the cheat sheet)

| Webpack | Vite |
|---|---|
| `HtmlWebpackPlugin` | Built-in (index.html at root) |
| `MiniCssExtractPlugin` | Built-in for prod |
| `DefinePlugin` | `define:` in config |
| `CopyWebpackPlugin` | `vite-plugin-static-copy` |
| `ProvidePlugin` (e.g. global jQuery) | Discouraged; use ES imports |
| `BundleAnalyzerPlugin` | `rollup-plugin-visualizer` |
| `webpack-bundle-analyzer` | `rollup-plugin-visualizer` |
| `tsconfig-paths-webpack-plugin` | Works via `vite-tsconfig-paths` |
| `@svgr/webpack` | `vite-plugin-svgr` |
| `webpack-pwa-manifest` / `workbox-webpack-plugin` | `vite-plugin-pwa` |

### 9. Update scripts in `package.json`

```json
{
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  }
}
```

### 10. Update CI / Docker

```dockerfile
# Build stage
RUN pnpm build                          # vite build, outputs to dist/
# Serve stage
COPY --from=build /app/dist /usr/share/nginx/html
```

## Common gotchas

- **`require` calls in code** — Vite errors. Convert to `import`.
- **CommonJS dependencies** — Vite handles with `optimizeDeps`. If a dep fails: add it to `optimizeDeps.include` in vite.config.
- **Polyfills for Node globals** (`Buffer`, `process`) — install `vite-plugin-node-polyfills` if a dep needs them in browser. Better: replace the dep with a browser-compatible alternative.
- **Dynamic imports with template literals** — `import(\`./locales/${lang}.js\`)` — Vite needs explicit glob: `import.meta.glob('./locales/*.js')`.
- **Web Workers** — Vite uses `?worker` query: `import Worker from "./my-worker.ts?worker"`.
- **Tests** — webpack often shared config with Jest. Vite + Vitest is the natural pair (see `jest-to-vitest`).
- **Storybook** — Storybook 7+ has Vite builder; older versions need migration.
- **Source maps in production** — `build.sourcemap: true` outputs `.map` files; serve them or upload to Sentry per `sentry-monitoring`.

## Performance comparison

After migration, expect:

| Metric | Webpack | Vite |
|---|---|---|
| Cold dev start | 10–60s | <2s |
| HMR after edit | 200–2000ms | <50ms |
| Production build time | similar | similar to Rollup |
| Production bundle size | similar | similar |

The dev-time delta is the win. Production builds are comparable.

## Anti-patterns

- **Trying to preserve webpack config 1:1** — Vite is convention-first. Trust the defaults; configure only what's different.
- **Migrating *and* refactoring in one PR** — too many variables. Migrate first; refactor later.
- **Ignoring `optimizeDeps` warnings** — slow startup or runtime errors. Add deps to `optimizeDeps.include`.
- **Not migrating tests** — running webpack-built tests in a Vite project doubles maintenance.
- **Production using webpack, dev using Vite** — divergent bundles, divergent bugs. Migrate fully or don't.
- **Polyfilling Node globals** as a default — the deps making you do this are a smell. Prefer browser-native alternatives.
- **Not benchmarking** — claim "Vite is faster" without measuring. Sometimes the win is smaller than expected; sometimes much larger.
- **Migrating before TS/ESM cleanup** — fix `.ts` extensions, mixed CJS/ESM first; then migrate.
- **Skipping production preview** — `vite preview` shows the actual prod build; test it before deploying.

## Verify it worked

- [ ] `pnpm dev` starts in < 2 seconds.
- [ ] HMR after a code edit shows in < 200 ms.
- [ ] `pnpm build && pnpm preview` produces a working production preview.
- [ ] Production bundle size is comparable to webpack output (within 10%).
- [ ] All tests pass under the new test runner.
- [ ] Source maps work in production debugger.
- [ ] CI is faster (build step time reduced).
- [ ] No `webpack`, `webpack-cli`, `webpack-dev-server` in dependencies.
- [ ] `process.env.X` references are migrated to `import.meta.env.VITE_X` (or polyfill is removed after migration).
- [ ] The dev experience improvement is real-world (devs report it; not just on-paper).
