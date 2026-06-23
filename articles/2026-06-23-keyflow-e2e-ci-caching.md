---
title: "Three Caches, One Runner: How We Cut the Repeated Work Out of a Next.js E2E Pipeline"
project: keyflow
tags: [DevOps, CI/CD, GitHub Actions, Playwright, Next.js]
status: audited
date: 2026-06-23
---

Every green checkmark on a slow CI pipeline is paid for in attention. If the end-to-end suite takes twenty minutes, nobody waits for it — they push, switch tasks, and come back later to find out whether the thing they've already forgotten about passed. The feedback loop that was supposed to catch regressions becomes a notification you skim. So when keyflow's Playwright E2E workflow started feeling like that, the fix wasn't to add machines. It was to stop doing the same expensive work on every run.

keyflow is our OKR and performance-management platform — a Next.js app with a Postgres-backed Prisma layer and a Playwright suite that drives the real UI. The E2E workflow did everything from scratch each time: install dependencies, install a browser, build the app, seed a database, run the tests. Most of that is identical from run to run. The redesign attacked the repetition on two fronts — caching and where the job runs.

## The two biggest time sinks

Profile any Node E2E pipeline and the same two villains show up: `npm ci` and the production build. Installing dependencies pulls a full tree over the network; building a Next.js app recompiles webpack bundles that barely changed since last time. Both are mostly cache-misses-by-default — the work is repeated not because it changed, but because nothing told CI it hadn't.

The fix is three layers of GitHub Actions cache, each keyed on exactly what would invalidate it.

```yaml
- name: Cache node_modules
  id: cache-node-modules
  uses: actions/cache@v4
  with:
    path: node_modules
    key: node-modules-${{ hashFiles('package-lock.json') }}

- name: Install dependencies
  if: steps.cache-node-modules.outputs.cache-hit != 'true'
  run: npm ci --legacy-peer-deps
```

That `if:` is the whole point. On a warm cache, `npm ci` is *skipped entirely* — not sped up, not partially reused, simply not run. The lockfile hash is the cache key, so the moment a dependency changes the key misses and a fresh install runs. Correctness is preserved; the redundant work is not.

The Playwright browser gets the same treatment, keyed on the same lockfile so the Chromium binary is restored rather than re-downloaded. The third cache is the interesting one — the Next.js build:

```yaml
- name: Cache Next.js build
  uses: actions/cache@v4
  with:
    path: .next/cache
    key: nextjs-${{ hashFiles('package-lock.json') }}-${{ hashFiles('src/**/*.ts', 'src/**/*.tsx', 'src/**/*.css') }}
    restore-keys: |
      nextjs-${{ hashFiles('package-lock.json') }}-
```

This key has two parts: the lockfile hash *and* a hash of the source tree. A precise key means a perfect hit reuses everything; but the `restore-keys` fallback is what makes it pay off on the common case. When the source has changed — so the exact key misses — Actions falls back to the most recent cache sharing the lockfile prefix. Next.js's incremental webpack cache then rebuilds only what actually changed, instead of compiling from cold. You get a warm build even when the code is new.

## Moving the work closer to the metal

Caching cuts what the job does; the runner determines how fast it does it. All workflows were moved off GitHub-hosted runners onto a self-hosted one — `runs-on: self-hosted` — which keeps the Actions cache on local disk and removes the cold-start tax of a fresh ephemeral VM per run.

That move had a sharp edge worth flagging for anyone considering it: the self-hosted box didn't have Docker. The workflow had been using `docker/setup-buildx-action`, which doesn't cooperate with podman. The fix was to drop the buildx action entirely and call `podman build` / `podman push` directly, with a `docker`→`podman` symlink and the podman user socket enabled so tooling expecting a Docker socket still works. The Postgres service likewise runs as a plain `podman run` of `postgres:16` on port 5436, with a `pg_isready` poll loop instead of a managed service container. Self-hosting trades the convenience of a clean managed environment for speed and control — and you pay for it in setup details like this one.

## Only run the tests that matter

The last lever isn't caching at all — it's *not running tests you don't need*. The workflow branches on event type:

```bash
if [ "${{ github.event_name }}" = "pull_request" ]; then
  BRANCH="${{ github.head_ref }}"
  SLUG="${BRANCH#*/}"
  FEATURE_SPEC="e2e/${SLUG}.spec.ts"
  if [ -f "$FEATURE_SPEC" ]; then
    npm run test:e2e -- "$FEATURE_SPEC"
  else
    npm run test:e2e -- --grep "@smoke"
  fi
else
  npm run test:e2e
fi
```

A pull request from a branch like `claude/progress-milestone-toasts` is mapped to `e2e/progress-milestone-toasts.spec.ts`. If that spec exists, the PR runs only it; if not, it runs the `@smoke` subset. The full suite is reserved for pushes to `main`. The convention — branch name implies feature spec — turns the pipeline into a fast, targeted check during review and a thorough one at the gate.

## Lessons

The pipeline is unremarkable now, which is the goal. A few takeaways generalise:

- **Cache on what invalidates, not on time.** Every cache here is keyed on a content hash — the lockfile, the source tree — so a hit is always safe and a change always misses. Time-based or branch-based cache keys quietly serve stale artifacts; content keys can't.
- **Skip, don't speed up.** The biggest win wasn't a faster `npm ci` — it was a conditional that doesn't run it at all on a warm cache. Look for whole steps you can guard out, not just steps to optimise.
- **`restore-keys` is where layered caches earn their keep.** A perfectly precise key rarely hits during active development. The prefix fallback gives you a warm-enough starting point, and an incremental builder does the rest.
- **Self-hosting is a tradeoff, not a free lunch.** You gain local cache and no VM cold-start; you inherit responsibility for the environment — which is how a missing Docker binary becomes a buildx-to-podman migration.
- **The cheapest test is the one you correctly skip.** Mapping branch names to feature specs let PRs run a focused check while keeping the full suite as the merge gate.

None of these are exotic. They're the difference between a pipeline people trust enough to watch and one they've trained themselves to ignore.
