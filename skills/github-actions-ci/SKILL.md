---
name: github-actions-ci
description: Production GitHub Actions patterns — matrix builds, caching, OIDC to AWS/GCP, reusable workflows, secrets discipline, concurrency control, and the small fixes that turn 12-minute CI into 3-minute CI. Use when setting up CI for a new repo, optimizing slow workflows, or hardening secrets posture.
---

# GitHub Actions CI

GitHub Actions is the dominant CI for new projects. The official docs cover features in isolation; this skill covers how to compose them for a fast, secure, maintainable CI.

## When to use

- New repo, setting up CI from scratch.
- Existing CI is slow (>5 min for a backend; >2 min for a frontend lib).
- Migrating from Travis/CircleCI/Jenkins.
- Hardening secrets after a leak / audit finding.

## When NOT to use

- Self-hosted runners with strict isolation needs at scale → look at Buildkite or Argo Workflows.
- Monorepo with 50+ services and complex DAG → Bazel + Bazel-cache is faster.

## The principles

1. **Cache aggressively, invalidate explicitly.** A 5-min CI is usually 4 min of dependency installs.
2. **OIDC always, long-lived secrets never.** Federate to AWS/GCP/Azure; keep secrets only for things without an OIDC story (npm token, Docker Hub).
3. **Concurrency cancels in-flight on push.** Push 3 commits in 30s → 1 run, not 3.
4. **One reusable workflow per shape.** "Build & test a Node app" is one file consumed by 12 repos via `uses:`.
5. **Pin to SHAs in security-sensitive jobs.** `actions/checkout@v4` → `actions/checkout@b4ffde65...`. Tags can be moved.

## Standard Node/TS workflow (well-cached)

```yaml
# .github/workflows/ci.yml
name: ci
on:
  pull_request:
  push:
    branches: [main]

concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.ref != 'refs/heads/main' }}

jobs:
  ci:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with: { version: 9 }
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm typecheck
      - run: pnpm lint
      - run: pnpm test
      - run: pnpm build
```

Key bits:
- **`concurrency` with `cancel-in-progress`** — pushing more commits cancels the older run on PR branches but never on `main` (where you want a clean record).
- **`cache: pnpm`** in `setup-node` — persists `~/.pnpm-store` keyed on `pnpm-lock.yaml`. Saves ~60s per run.
- **`--frozen-lockfile`** — fails if lockfile is out of sync, surfacing dependency drift early.
- **`timeout-minutes`** — runaway jobs don't burn 6 hours of credits.
- **`permissions: contents: read`** — start with least privilege, grant per-job below.

## Matrix build (one job, many configs)

```yaml
jobs:
  test:
    strategy:
      fail-fast: false           # don't cancel siblings on first failure
      matrix:
        node: [18, 20, 22]
        os: [ubuntu-latest, macos-latest, windows-latest]
        exclude:
          - { os: windows-latest, node: 18 }   # skip combos you don't ship
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: ${{ matrix.node }} }
      - run: npm ci
      - run: npm test
```

`fail-fast: false` is non-obvious — default is `true`, which kills sibling jobs as soon as one fails. Half-information runs help nobody.

## OIDC to AWS (no long-lived keys)

```yaml
deploy:
  needs: ci
  runs-on: ubuntu-latest
  permissions:
    id-token: write              # required for OIDC
    contents: read
  steps:
    - uses: actions/checkout@v4
    - uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::123456789012:role/github-deploy
        aws-region: us-east-1
    - run: aws s3 sync ./out s3://my-bucket
```

The IAM role's trust policy condition restricts to your repo + branch:

```json
{
  "StringEquals": { "token.actions.githubusercontent.com:aud": "sts.amazonaws.com" },
  "StringLike":   { "token.actions.githubusercontent.com:sub": "repo:acme/myrepo:ref:refs/heads/main" }
}
```

No `AWS_ACCESS_KEY_ID` in GitHub secrets → no rotation, no leak risk.

## Reusable workflow (shared across repos)

```yaml
# In acme/.github repo: .github/workflows/node-app.yml
name: node-app
on:
  workflow_call:
    inputs:
      node-version: { type: string, default: '20' }
      pnpm-version: { type: string, default: '9' }

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with: { version: ${{ inputs.pnpm-version }} }
      - uses: actions/setup-node@v4
        with: { node-version: ${{ inputs.node-version }}, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm typecheck && pnpm lint && pnpm test && pnpm build
```

```yaml
# Each consuming repo: .github/workflows/ci.yml
on: [push, pull_request]
jobs:
  ci:
    uses: acme/.github/.github/workflows/node-app.yml@main
    with:
      node-version: '22'
```

12 repos, 1 workflow, 1 review per change. Big win on a small team.

## Speed tricks

- **Docker layer cache** — use `docker/build-push-action` with `cache-from: type=gha` and `cache-to: type=gha,mode=max`.
- **Parallelize independent steps** as separate `jobs:` not sequential `steps:`. Free CPU, free wall-clock.
- **Restore-only cache for flaky builds** — `actions/cache/restore@v4` then `actions/cache/save@v4` only on `main`. Avoids each PR poisoning cache.
- **`runs-on: ubuntu-22.04`** instead of `ubuntu-latest` if you need predictable behavior. `latest` floats.
- **`paths:` filter** on `on: pull_request:` — don't run frontend CI for backend-only PRs.

## Secrets posture

```yaml
permissions:                    # repo-wide default in GH settings: read-only
  contents: read

# Per-job grants only:
publish:
  permissions:
    contents: write             # to push tags
    packages: write             # to push to ghcr
    id-token: write             # OIDC
  steps:
    - run: echo "${{ secrets.NPM_TOKEN }}" | npm login --auth-type=legacy
```

GitHub repo setting → "Default workflow permissions: Read repository contents and packages permissions". Then grant up per-job.

## Anti-patterns

- **`AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` in repo secrets** — long-lived, leakable, you forget to rotate. Use OIDC.
- **`pull_request_target`** without understanding the security model — it runs with repo secrets *on the base branch* but checks out *the PR's code*. Easy to leak secrets to a fork.
- **`actions/checkout@main`** — supply chain risk. Pin to a tag or SHA.
- **No `concurrency:` group** — every push spawns a run; old runs eat minutes for nothing.
- **`fail-fast: true` (default) on matrix** — partial signal that wastes a re-run.
- **Caching `node_modules`** — non-portable across Node versions, often slower than re-installing from a `~/.pnpm-store` cache.
- **One giant 1000-line workflow file** — split into reusable workflows or per-purpose files.
- **`run: |` with unquoted variables** — shell injection from PR titles, branch names. Always quote: `run: echo "$TITLE"` not `run: echo $TITLE`.
- **Using `set-output` (deprecated)** — use `$GITHUB_OUTPUT` redirection.
- **Self-hosted runners on default config** — they cache previous job state. Without isolation, one PR can poison the next.

## Verify it worked

- [ ] CI runs in under 5 min for a typical Node backend (median).
- [ ] Cache hit rate > 80% (visible in Actions logs).
- [ ] Pushing 3 commits to a PR in 1 min results in 1 successful run, not 3.
- [ ] No `AWS_ACCESS_KEY_ID` / `GCP_SA_KEY` secrets in repo settings.
- [ ] Default workflow permissions are `read`; jobs grant up explicitly.
- [ ] Failing one matrix cell does not cancel the others.
- [ ] Reusable workflow consumed by ≥ 2 repos and updated by changing one file.
- [ ] Untrusted PRs (forks) cannot read repo secrets.
- [ ] All third-party actions are pinned to tags or SHAs (no `@main`).
- [ ] `timeout-minutes` set on every long-running job.
