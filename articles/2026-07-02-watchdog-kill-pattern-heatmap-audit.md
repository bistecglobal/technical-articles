# Audit — The Watchdog's Logbook: Reading When and Why an AI Fleet Gets Killed

**Verdict: PASS — 24/25**

Feature verified merged to `main` in claude-mcd (PR #281, commit `0278ed0`; kill-log writer PR #238, commit `70f7001`). Files on `origin/main`:
- `apps/mission-control/app/api/watchdog-kill-patterns/route.ts`
- `apps/mission-control/app/watchdog-kill-patterns/page.tsx`
- `src/claude-process.ts` (`appendWatchdogKill`, adaptive threshold constants)

## Claim inventory

| # | Claim | Verdict | Evidence |
|---|-------|---------|----------|
| 1 | One Claude subprocess per project | ✅ | ProjectPool per-slug process model, `claude-process.ts` |
| 2 | Adaptive watchdog threshold = 1.5× longest turn, capped at 30 min | ✅ | `ADAPTIVE_MULTIPLIER = 1.5`, `MAX_ADAPTIVE_THRESHOLD_MS = 30 * 60_000`, formula `claude-process.ts:1180` |
| 3 | Respawns on next message | ✅ | `notifyCrash` — "respawning on next message" |
| 4 | Kill with reason `watchdog` appends to per-project `watchdog-kills.jsonl` | ✅ | `kill()` calls `appendWatchdogKill()` only when `reason === 'watchdog'`; path `projects/<slug>/watchdog-kills.jsonl` |
| 5 | `lastToolCall` reconstructed from pending-tools map | ✅ | `transcriptPendingTools`, picks latest `startMs` |
| 6 | Append wrapped in swallow-everything catch, non-fatal | ✅ | try/catch with "never let kill log failure block the kill itself" |
| 7 | `/watchdog-kill-patterns` reads all logs via one route, filters by slug + date range | ✅ | `route.ts` GET reads `filterSlug`, `since`, `until`, iterates `targetSlugs` |
| 8 | 7×24 UTC heatmap day×hour, pre-seeded full grid | ✅ | `getUTCDay()`/`getUTCHours()` keys; nested 7×24 loop seeds zeros |
| 9 | Amber-to-red ramp scaled to busiest cell | ✅ | `heatColor(count, maxCount)`, `page.tsx:10-18` |
| 10 | Top-ten bar chart of `lastToolCall` | ✅ | `.sort().slice(0, 10)` → `precedingTools` |
| 11 | Context usage = (cache_read + input) / 200000 | ✅ | `getContextPct`, `route.ts:92-93` |
| 12 | Scatter of kill count vs context pct per project | ✅ | `contextPressure` array (`contextPct`, `killCount`) |
| 13 | Fleet badge = kills/day over trailing 7 days | ✅ | `killRatePer7d = round((killsLast7d / 7)*10)/10` |
| 14 | Log records only `watchdog` kills, not idle-evict/pool-full/shutdown | ✅ | kill reason union; append gated on `reason === 'watchdog'` |
| 15 | "nineteen autonomous projects" fleet size | ⚠️ | Consistent with series/fleet framing (ARTICLE_IDEAS "19 AI agents"); scene-setting, not a product metric — acceptable |

## Forward-looking scan

No violations. Searched: will / plan to / coming soon / in the future / next step / roadmap / soon. "candidate for compaction or a shorter leash" is operator advice, not a roadmap claim. All tense is present/historical.

## Rubric

| Dimension | Score | Notes |
|---|---|---|
| Evidence quality | 5 | Every technical claim maps to code; no SHAs in prose |
| Technical depth | 5 | Kill-time reconstruction, three cross-tabs, context estimation all explained |
| Clarity | 5 | Clear when/what/why framing; strong hook |
| BistecGlobal voice | 5 | Practitioner, forensic, evidence-grounded |
| Title specificity | 4 | Specific and human; slightly literary |
| **Total** | **24/25** | |

## Fixes applied
- Frontmatter `status: draft → audited`. No prose changes required.
