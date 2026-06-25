# Audit — Did the Schedule Actually Fire? Run History for Autonomous Agents

**Verdict: PASS**
**Score: 24/25**
**Date:** 2026-06-25
**Evidence base:** claude-mcd commit `0670b47` (P217). File: `apps/mission-control/app/api/schedule-runs/route.ts` (+ `schedule-history/page.tsx`, commit body).

## Claim inventory

| # | Claim | Verdict |
|---|-------|---------|
| 1 | Runs appended per firing to `schedule-log.jsonl` with chatId/scheduledAt/firedAt/status/durationMs | ✅ verified (ScheduleRunEntry, readJsonl) |
| 2 | status is ok / stalled / skipped / other | ✅ verified (type union) |
| 3 | `schedules.json` holds config (at, interval, enabled, runCount); `channels.json` maps chatId→slug | ✅ verified |
| 4 | Route joins the three files | ✅ verified |
| 5 | Window default 30, clamped 1–365 | ✅ verified (`Math.min(Math.max(...,1),365)`) |
| 6 | Groups runs by chatId | ✅ verified |
| 7 | Computes ok count, error count, avgDurationMs, successRate, lastFiredAt | ✅ verified |
| 8 | Any non-`ok` status counts as error | ✅ verified (`r.status !== 'ok'`) |
| 9 | Zero runs → successRate 1 | ✅ verified (`runs.length > 0 ? ... : 1`) |
| 10 | Configured schedules with no runs included as empty groups | ✅ verified (forced loop over chatIdToSchedule keys) |
| 11 | Heatmap built day→chatId→{ok,error} | ✅ verified |
| 12 | Calendar axis built independently, every day in window | ✅ verified (calendarDays loop) |
| 13 | Groups sorted by lastFiredAt desc | ✅ verified |
| 14 | UI: cyan(ok)/red(error) calendar strip + expandable run table (status/duration/timestamp) | ✅ verified (commit body) |
| 15 | "This article was produced by a schedule" | ✅ verified (scheduled task context) |

15/15 verified. Zero ⚠️ / ❌.

## Forward-looking scan

No matches for plan to / coming soon / in the future / next step / roadmap. "the future doesn't send receipts" is metaphor; "tomorrow's failure mode is already caught" describes existing behavior. PASS.

## Rubric

| Dimension | Score | Notes |
|---|---|---|
| Evidence quality | 5 | Every claim maps to route; no SHAs in prose |
| Technical depth | 5 | Observed-vs-intended join + independent time axis are the real insights, well grounded |
| Clarity for audience | 5 | "Silent schedule" framing lands; self-referential hook strong |
| BistecGlobal voice | 5 | Practitioner, transferable |
| Title specificity | 4 | Specific question-hook; accurate |

**Total: 24/25** — exceeds threshold.

## Fixes applied

None required. Frontmatter status promoted draft → audited.
