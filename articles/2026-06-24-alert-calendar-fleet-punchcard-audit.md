# Audit — 2026-06-24-alert-calendar-fleet-punchcard

**Verdict: PASS**
**Score: 24/25**

## Claim inventory & verdicts

| # | Claim | Verdict | Evidence |
|---|-------|---------|----------|
| 1 | MCD runs a fleet of Claude agents, one subprocess per project | ✅ | Established platform architecture (prior published articles, CLAUDE.md) |
| 2 | `alert_events` table stores timestamp + type | ✅ | `db.ts:556` query reads `ts`, `alert_type` from `alert_events` |
| 3 | Alert types include stalls, budget breaches, anomaly flags, limit hits | ✅ | Prior shipped features feed alert_events (anomalies, budget, stall-risk, limit-hit) |
| 4 | 7×24 grid over last 30 days | ✅ | `route.ts` `DAYS_BACK = 30`; grid `Array.from({length:7})` × `{length:24}` |
| 5 | Feature is two files — API route + client page | ✅ | `api/alert-calendar/route.ts` + `alert-calendar/page.tsx` |
| 6 | SQL uses `strftime('%w'…)`, `strftime('%H'…)`, `localtime`, `GROUP BY` | ✅ | `db.ts:558-564` verbatim |
| 7 | `%w` = 0 Sunday … 6 Saturday | ✅ | SQLite strftime spec; matches `DOW` array order in page |
| 8 | Grouping also by `alert_type` for per-cell breakdown | ✅ | `GROUP BY dow, hour, alert_type`; `cell.byType` accumulation |
| 9 | Grid pre-seeded zero-filled | ✅ | `route.ts` grid init with `{count:0, byType:{}}` |
| 10 | Tracks max + busiest, nulls busiest if silent | ✅ | `if (busiest && busiest.count === 0) busiest = null` |
| 11 | Second pass walks all 168 cells | ✅ | `7×24 = 168`; nested loop `d<7`, `h<24` |
| 12 | Intensity `count/max`, transparent→red ramp `rgba(239,68,68,…)` | ✅ | `page.tsx` `0.15 + intensity * 0.85` |
| 13 | Empty cells faint grey `rgba(148,163,184,0.06)` | ✅ | `page.tsx` verbatim |
| 14 | Header shows busiest window + total in range | ✅ | header strip renders `busiest` + `total` |
| 15 | Hover reveals per-type breakdown | ✅ | hover breakdown block maps `byType` |
| 16 | Refreshes every 60s with freshness badge | ✅ | `useFreshness('/api/alert-calendar', 60_000)` + `FreshnessBadge` |
| 17 | GitHub popularised 7×24 punchcard | ✅ | General industry fact |

ASCII diagram and the `Mon 09:00 · 14` example are explicitly illustrative; format matches `${DOW} ${fmtHour} · ${count}`. No fabricated metrics presented as measured outcomes.

## Forward-looking scan
None found (`will / plan / soon / roadmap / next step / coming` — 0 matches).

## SHAs / PR numbers in prose
None.

## Rubric

| Dimension | Score | Notes |
|---|---|---|
| Evidence quality | 5 | Every claim maps to read source; no SHAs in prose |
| Technical depth | 5 | Real SQL + TS snippets, accurate algorithm description |
| Clarity for audience | 5 | Strong hook, clear operational framing |
| BistecGlobal voice | 5 | Practitioner-focused, evidence-grounded |
| Title specificity | 4 | Specific + human; question form on the edge of clickbait but earns it |

**Total: 24/25 — PASS**

No fixes required beyond status bump.
