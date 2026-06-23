# Audit — Three Caches, One Runner: keyflow E2E CI Caching

**Verdict: PASS** · **Score: 23/25** · Claims fixed: 0 · Claims flagged: 0

## Claim inventory

| # | Claim | Verdict | Evidence |
|---|---|---|---|
| 1 | keyflow = Next.js app, Prisma/Postgres, Playwright E2E | ✅ | `e2e.yml`: prisma generate/migrate/seed, `npm run build`, `npm run test:e2e` |
| 2 | Workflow installs deps, installs browser, builds, seeds DB, runs tests | ✅ | `e2e.yml` step list |
| 3 | Three GHA caches keyed on content hashes (lockfile / source) | ✅ | `actions/cache@v4` × 3 in `e2e.yml` (21d07d0) |
| 4 | `npm ci` skipped on warm cache via `if: cache-hit != 'true'` | ✅ | exact step in `e2e.yml` |
| 5 | Playwright Chromium cached, keyed on lockfile | ✅ | `playwright-chromium-${{ hashFiles('package-lock.json') }}` |
| 6 | Next.js `.next/cache` keyed on lockfile+source, `restore-keys` prefix fallback | ✅ | exact cache step |
| 7 | All workflows moved to `runs-on: self-hosted` | ✅ | commit 8c78d0a "switch all workflows to self-hosted runner"; `e2e.yml` |
| 8 | Self-hosted box lacked Docker; dropped buildx, switched to `podman build`/`push`, docker→podman symlink, user socket | ✅ | commit 00cd3ca body + diff |
| 9 | Postgres via `podman run postgres:16` on port 5436, `pg_isready` poll | ✅ | `e2e.yml` Start postgres step |
| 10 | PR runs feature spec or `@smoke`; push runs full suite | ✅ | exact bash in Run E2E tests step |
| 11 | Branch slug maps to `e2e/<slug>.spec.ts` | ✅ | `SLUG="${BRANCH#*/}"; FEATURE_SPEC="e2e/${SLUG}.spec.ts"` |

## Forward-looking / unmeasured-outcome scan
- Line 9 "twenty minutes" — used as a **hypothetical** framing ("If the end-to-end suite takes twenty minutes…"), not an asserted measured keyflow figure. Acceptable.
- **No** "5 minutes" / "5min" / achieved-speedup claim anywhere — the commit's "~20min → ~5min target" was a *target*, deliberately not stated as an outcome (editorial: real outcomes only). ✅
- No `will`/`plan to`/`coming soon`/`roadmap`/`next step` matches.

## Rubric

| Dimension | Score | Notes |
|---|---|---|
| Evidence quality | 5 | Every claim maps to `e2e.yml` or a migration commit; no SHAs in prose; avoided the unmeasured metric |
| Technical depth | 5 | Cache-key design, restore-keys fallback, buildx→podman edge, selective spec running |
| Clarity for audience | 5 | Clean problem→levers→lessons arc for a DevOps reader |
| BistecGlobal voice | 4 | Practitioner, grounded; opening "paid for in attention" framing strong |
| Title specificity | 4 | Concrete ("Three Caches, One Runner"); intentionally drops the unverified speed number |
| **Total** | **23/25** | |

Above 20 threshold. No fixes required.
