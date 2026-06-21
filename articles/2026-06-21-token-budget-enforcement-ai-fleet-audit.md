# Audit: 2026-06-21-token-budget-enforcement-ai-fleet

**Verdict: PASS**
**Score: 23/25**
**Claims verified: 19/20 | Fixed: 2 | Flagged ❌: 0**

---

## Claim Inventory

| # | Claim | Verdict | Evidence |
|---|-------|---------|----------|
| 1 | "fleet of 19+ Claude Code agents" | ✅ | 19 project dirs in `/home/openclaw/.claude/channels/discord-multi/projects/` |
| 2 | "one agent per Discord channel" | ✅ | MCD architecture — `ProjectPool` keyed by `chatId` |
| 3 | "PR #84 (commit `17a0f12`)" | ✅ | `git log`: `17a0f12 feat(mc): Token Budget Enforcement & Alerts (P40) (#84)` |
| 4 | "`monthlyTokenBudget` was already stored in `channels-config.ts` — but purely decorative" | ⚠️ | Field was in `channels.json` and read by fleet route since PR #74 (`6c3d8ad`), but Zod schema in `channels-config.ts` was NOT updated until PR #84. Article said "channels-config.ts" specifically — fixed to "channels config JSON". |
| 5 | "fleet dashboard showed raw token counts from JSONL transcripts" | ✅ | PR #74 added gauge + `monthlyTokensUsed` to `/api/fleet` (`6c3d8ad`) |
| 6 | "Token counts from `assistant`-type records in JSONL, filtered to UTC year-month" | ✅ | `computeMonthlyTokensUsed()` in `src/project-pool.ts`, commit `17a0f12` |
| 7 | Code snippet `computeMonthlyTokensUsed` | ✅ | Exact code from `src/project-pool.ts` diff, commit `17a0f12` |
| 8 | "Budget alerts at 50%, 80%, 100% — once per threshold per month" | ✅ | `checkBudgetThresholds()`, `budgetAlertFired` Map, `src/project-pool.ts` |
| 9 | "`budgetAlertFired: Map<chatId, Set<50|80|100>>`" | ✅ | Field declaration in `ProjectPool` class, commit `17a0f12` |
| 10 | `checkBudgetThresholds` code snippet | ✅ | Exact code from diff |
| 11 | `checkMonthRollover()` code snippet | ✅ | Exact code from diff |
| 12 | "`budget-restored` pool event fires with drained count" | ✅ | `PoolEvent` union type and `drainBudgetQueues()` in `src/project-pool.ts` |
| 13 | `server.ts` Discord master-channel notifications | ✅ | Exact code from `server.ts` diff, commit `17a0f12` |
| 14 | "SSE broadcaster... 5-second fleet tick" | ✅ | `sse.ts` line 161: `setInterval(broadcastFleetUpdate, 5_000)` |
| 15 | "`budgetAlertState: Map<string, string>` keyed by `slug:threshold:YYYY-MM`" | ✅ | `sse.ts` diff: `const stateKey = \`${project.slug}:${key}:${yearMonth}\`` |
| 16 | `computeBudgetStatus` function | ✅ | Exact code from `apps/mission-control/src/fleet-compute.ts`, commit `17a0f12` |
| 17 | "amber for warning (≥50%), red for critical (≥80%), grey for exhausted" | ✅ | `InstanceGrid.tsx` diff: `#F59E0B`=warning, `#EF4444`=critical, `#94a3b8`=exhausted |
| 18 | "integration test suite added in #85 (commit `7a9b31b`)" | ✅ | `git log`: `7a9b31b test: add integration test suite (10 test groups, budget enforcement + MCP round-trip) (#85)` |
| 19 | "reading JSONL files on every `deliver()` call for budgeted projects" | ✅ | `computeMonthlyTokensUsed` called inline in `deliver()` |
| 20 | "pool fires Discord notifications; SSE layer fires browser toasts" | ✅ | Two separate mechanisms: `server.ts` event listener + `sse.ts` `checkBudgetAlerts()` |

---

## Forward-Looking Flags

| Location | Text | Action |
|----------|------|--------|
| Design Decisions section | "If the fleet grew to hundreds, a cached counter with a short TTL would be the natural next step." | Reframed as design consideration, not roadmap item. Removed "next step" language. |

---

## Rubric Scorecard

| Dimension | Score | Notes |
|---|---|---|
| Evidence quality (claims cited) | 5/5 | Every code block exact; SHAs and PR numbers for all major claims |
| Technical depth | 5/5 | All 4 layers explained with real implementation detail; dedup logic clarified |
| Clarity for target audience | 4/5 | Dense in places; context is sufficient for practitioner audience |
| BistecGlobal voice | 4/5 | Professional, grounded, practitioner-focused throughout |
| Title specificity | 5/5 | Precise: names the mechanism (SSE alerts, queue drain, monthly rollover) |

**Total: 23/25 — PASS**

---

## Fixes Applied

1. **Claim #4** — Changed "already stored in `channels-config.ts`" to "already present in the channels config JSON" to accurately reflect that the field was in the raw JSON and read by the fleet route (PR #74), but the Zod schema in `channels-config.ts` was not updated until PR #84.

2. **Forward-looking statement** — Removed "the natural next step" framing. Reworded to describe the scaling tradeoff without implying planned work.
