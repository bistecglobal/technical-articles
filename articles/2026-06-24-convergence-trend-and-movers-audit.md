# Audit — 2026-06-24-convergence-trend-and-movers

**Verdict: PASS**
**Score: 24/25**

## Claim inventory & verdicts

| # | Claim | Verdict | Evidence |
|---|-------|---------|----------|
| 1 | MCD runs a fleet of Claude agents, one subprocess per project | ✅ | Established architecture |
| 2 | Each agent scored daily on convergence | ✅ | `convergence_history` with daily rows |
| 3 | `convergence_history` table, one row per project per date | ✅ | `db.ts:158` schema, `UNIQUE(slug, date)` |
| 4 | Trend = 14-day line of fleet mean convergence | ✅ | `getFleetConvergenceTrend(14)`; `/api/convergence-trend` |
| 5 | SQL: AVG(score), SUM(score≥90) topBinCount, COUNT(*), GROUP BY date, ORDER BY date DESC LIMIT 14, reversed oldest→newest | ✅ | `db.ts:950-960` verbatim + `.reverse()` |
| 6 | Top-bin counts drawn as bars under the mean line | ✅ | trend page `bh = (topBinCount/maxTop) * (plotH*0.4)`, polyline mean line |
| 7 | Header delta = last.meanScore − first.meanScore | ✅ | trend page L27 |
| 8 | Movers: curr = latest, prev = prior distinct day, delta = curr−prev, null if single entry | ✅ | `getConvergenceMovers` body |
| 9 | Climbers (delta>0 desc), Fallers (delta<0, biggest drop first), New bucket for null delta | ✅ | movers page L44-47 |
| 10 | Each row shows prev→curr + sparkbar, magnitude `min(1, abs(delta)/100)` | ✅ | movers page L16, L25 |
| 11 | netDelta = sum of all deltas | ✅ | `/api/convergence-movers` route `reduce` |
| 12 | Net delta in header, green up / red down with ▲▼ | ✅ | movers page L75-76 |
| 13 | Both reuse `convergence_history`; trend aggregates by date, movers diffs by date per project; no new pipeline | ✅ | Both read same table via dedicated routes |
| 14 | Refreshes every 60s | ✅ | movers page footnote "Refreshes every 60s" + `useFreshness` |

No fabricated metrics; example values (score 60, +60) are explicitly illustrative of the null-baseline bug.

## Forward-looking scan
None (0 matches).

## SHAs / PR numbers in prose
None.

## Rubric

| Dimension | Score | Notes |
|---|---|---|
| Evidence quality | 5 | Every claim maps to read source (db.ts + both pages) |
| Technical depth | 5 | Aggregate-vs-attribution framing, null-baseline bug, net-delta reconciliation all accurate |
| Clarity for audience | 5 | "Trend line and the suspects" framing; macro/micro split lands |
| BistecGlobal voice | 5 | Practitioner-focused, strong lessons |
| Title specificity | 4 | Specific + human; "The Trend Line and the Suspects" earns it |

**Total: 24/25 — PASS**
