---
name: mcd-circuit-breaker-installer-audit
article: articles/2026-06-21-mcd-circuit-breaker-installer.md
audited: 2026-06-21
verdict: PASS (with fixes applied)
score: 22/25
---

# Audit: Circuit Breaker for AI Subprocesses

## Summary Verdict

**PASS** — 2 factual errors fixed before publish, 1 minor claim strengthened. Score 22/25.

---

## Claim Inventory

| # | Claim | Verdict | Evidence |
|---|-------|---------|----------|
| 1 | "Running 19 AI agents simultaneously" | ❌ unverified | No file in claude-mcd names 19 as the fleet size. Removed in fix. |
| 2 | PR #81 on claude-multi-channel-discord | ✅ verified | `git log` confirms commit `2a86715` merged as PR #81 |
| 3 | One Claude subprocess per channel, managed by ProjectPool in server.ts | ✅ verified | `server.ts` architecture, `src/project-pool.ts` |
| 4 | bin/install.sh 273 lines, bin/install.ps1 167 lines, bin/mcd-setup.js 14 lines | ✅ verified | `wc -l` output matches exactly |
| 5 | Non-interactive env vars: MCD_BOT_TOKEN, MCD_USER_ID, MCD_MASTER_CHANNEL | ✅ verified | `bin/install.sh` lines 6, 134–135, 143–150 |
| 6 | OS detection via uname -s, bun via bun.sh/install, tmux via apt-get/brew | ✅ verified | `bin/install.sh` lines 26–80 |
| 7 | Linux: systemd at ~/.config/systemd/user/mcd.service, fallback to /etc/systemd/system/ | ✅ verified | `bin/install.sh` lines 176–206 |
| 8 | macOS: launchd plist at ~/Library/LaunchAgents/com.github.chan4lk.mcd.plist | ✅ verified | `bin/install.sh` line 211 |
| 9 | "Windows: Task Scheduler via Register-ScheduledTask" | ❌ wrong | `bin/install.ps1` uses NSSM (primary) → sc.exe (fallback). No Task Scheduler. Fixed. |
| 10 | FailureLedger fields: count, windowStart, backoffMs, circuitOpen, circuitOpenAt? | ✅ verified | `src/project-pool.ts` lines 22–28, commit `2a86715` |
| 11 | Constants: 30min window, 5 failures, 10min reset, [5s/10s/30s/2m/5m] backoff | ✅ verified | `src/project-pool.ts` lines 85–88 |
| 12 | onExit calls recordFailureAndMaybeRespawn on non-zero exit | ✅ verified | `src/project-pool.ts` line 479 |
| 13 | Code snippets for circuit-open logic and deliver check | ✅ verified | Match exactly in `src/project-pool.ts` lines 536–552, 120–131 |
| 14 | Circuit auto-resets after 10 minutes without manual intervention | ✅ verified | `src/project-pool.ts` lines 123–130 |
| 15 | server.ts writes circuit-state.json | ✅ verified | `server.ts` grep confirms write path |
| 16 | fleet-compute.ts reads circuit-state.json | ✅ verified | `apps/mission-control/src/fleet-compute.ts` lines 284–318 |
| 17 | /api/fleet exposes circuitOpen per project | ✅ verified | `apps/mission-control/app/api/fleet/route.ts` added in PR #81 stat |
| 18 | fleet-compute auto-expires entries older than 10 min | ✅ verified | `fleet-compute.ts` line 313 comment: "auto-expire after 10 min" |
| 19 | "6 new test cases (test cases 13–14)" | ⚠️ weak | PR has 2 test blocks (13, 14) with 7 individual check() assertions (3+4). Fixed to "2 test blocks with 7 assertions". |
| 20 | Test 13: 5 crashes → 4 respawn-scheduled + circuit-open with failureCount 5 | ✅ verified | `src/project-pool.test.ts` lines 434–442 |
| 21 | Test 14: message dropped → circuit-reset → new spawn after time advance | ✅ verified | `src/project-pool.test.ts` lines 471–484 |
| 22 | Tests inject mock now() to simulate time | ✅ verified | `src/project-pool.test.ts` lines 425, 462 |

---

## Forward-Looking Scan

Search terms: "will", "plan to", "coming soon", "in the future", "next step", "roadmap", "soon"

**Result: No forward-looking statements found.**

---

## Rubric Scorecard

| Dimension | Score | Notes |
|---|---|---|
| Evidence quality (claims cited) | 4/5 | 2 errors caught and fixed; all code references are precise |
| Technical depth | 5/5 | FailureLedger design, backoff sequence, circuit state file, test injection — all covered |
| Clarity for target audience | 5/5 | Practitioner-level; code snippets placed at the right moments |
| BistecGlobal voice | 4/5 | Strong. Intro hook ("2 AM crash loop") slightly generic but acceptable |
| Title specificity | 4/5 | Specific; covers both themes (installer + circuit breaker) |

**Total: 22/25** (threshold: 20/25) → **PASS**

---

## Fixes Applied

1. **Claim 1 — removed "19 AI agents"**: Changed intro from "Running 19 AI agents simultaneously" to "Running a fleet of AI agents simultaneously" (specific number not grounded in any file).

2. **Claim 9 — Windows service registration**: Changed "Task Scheduler entry via PowerShell's `Register-ScheduledTask`" to "NSSM service (falling back to sc.exe)" to match actual `bin/install.ps1` implementation.

3. **Claim 19 — test count**: Changed "6 new test cases (test cases 13–14)" to "2 new test blocks (13–14) covering 7 individual assertions" to match actual test structure in `src/project-pool.test.ts`.
