# Audit: fleet-state-snapshots-alert-history

**Verdict: PASS (23/25) — minor fixes applied**
**Date: 2026-06-22**

---

## Claim Inventory

| # | Claim | Verdict |
|---|---|---|
| 1 | 19+ Claude Code agents running continuously | ✅ verified |
| 2 | SSE stall detection every 30 seconds | ❌ actual interval is 5s (`setInterval(..., 5_000)` in `sse.ts`) |
| 3 | budget alerts at 50/80/100% thresholds | ✅ verified in `sse.ts` thresholds array |
| 4 | `/api/snapshots` calls `computeFleet()` | ✅ verified in `app/api/snapshots/route.ts` |
| 5 | `fleet_snapshots` SQL schema | ✅ exact match with `db.ts` |
| 6 | per-project blob: slug, state, tokens, budget, goalText, goalStatus | ✅ verified in `route.ts` snapshotData object |
| 7 | unlabeled snapshots fall back to timestamp in UI | ✅ verified via `formatTs()` in `snapshots/page.tsx` |
| 8 | `diffSnapshots()` with added/removed/changed/same | ✅ exact match |
| 9 | sort order: removed, added, changed, same | ✅ verified via `order` const in `diffSnapshots` |
| 10 | diff is client-side, two fetch calls, JavaScript Map | ✅ verified in `loadDiff()` |
| 11 | `alert_events` SQL schema | ✅ exact match with `db.ts` |
| 12 | Alert types: stall, budget, watchdog, inject | ❌ `watchdog` in UI filter list but no `insertAlertEvent` call writes it; active types are stall, budget, inject |
| 13 | Stall+budget from SSE broadcaster; inject from `/api/inject` | ✅ verified |
| 14 | TypeScript stall-insertion snippet | ✅ exact match with `sse.ts` |
| 15 | SSE broadcaster 30-second interval (repeated from claim 2) | ❌ it's 5s |
| 16 | `/alerts` reverse-chronological, type-badge filtering | ✅ verified |
| 17 | Debounced slug filter | ✅ `debounceRef` + 300ms timeout in `alerts/page.tsx` |
| 18 | Cursor pagination, 100 rows, `nextCursor` | ✅ verified in `fetchAlerts` |
| 19 | 30-day auto-purge on startup | ✅ `DELETE FROM alert_events WHERE ts < ?` in `db.ts` |
| 20 | ~700 lines of code | ❌ commit has 887 insertions; fixed to ~900 |
| 21 | Two projects consistently stalling overnight | ⚠️ anecdotal/operational |
| 22 | Budget alerts clustering same hour | ⚠️ anecdotal/operational |

## Forward-Looking Scan

No "will", "plan to", "coming soon", "roadmap", "soon" found. PASS.

## Rubric

| Dimension | Score | Notes |
|---|---|---|
| Evidence quality | 3/5 | Three factual errors found and fixed |
| Technical depth | 5/5 | SQL schemas, TS snippet, diff algorithm, design rationale |
| Clarity | 5/5 | Narrative arc clear, accessible |
| BistecGlobal voice | 5/5 | Professional, practitioner-focused |
| Title specificity | 5/5 | Specific and human |

**Total: 23/25 — PASS**

## Fixes Applied

1. "stall detection every 30 seconds" → "stall detection every 5 seconds"
2. Removed `watchdog` from listed active alert types (UI filter includes it as future-proofing; no writes exist)
3. "roughly 700 lines" → "close to 900 lines"
