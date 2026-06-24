# Audit — When Will This Agent Actually Finish?

**Verdict: PASS** · **Score: 24/25** · Claims fixed: 0 · Claims flagged: 0

## Claim inventory

| # | Claim | Verdict | Evidence |
|---|-------|---------|----------|
| 1 | Agents carry a 0–100 convergence score recorded over time in `convergence_history` | ✅ | prior P182/P183 features; `getConvergenceSparklines` reads the series |
| 2 | Forecast reuses existing series — no new table/collector | ✅ | commit `f276ed6` "Pure client compute on getConvergenceSparklines — no new DB fn" |
| 3 | Reads last 30 days of points | ✅ | route `WINDOW_DAYS = 30`, `getConvergenceSparklines(WINDOW_DAYS)` |
| 4 | Ordinary least-squares fit of score vs day-index, written inline | ✅ | route `linearFit()` exact |
| 5 | `etaDays = ceil((TARGET − current) / slope)` | ✅ | route body |
| 6 | TARGET = 90, MIN_POINTS = 3 | ✅ | route constants |
| 7 | Projects with <3 points silently dropped | ✅ | `.filter((s) => s.points.length >= MIN_POINTS)` |
| 8 | ±0.05/day deadband classifies rising / stalled / declining | ✅ | status branches `slopeR > 0.05` / `< -0.05` / else stalled |
| 9 | ETA emitted only for rising agents | ✅ | `etaDays` set only in `rising` branch; null otherwise |
| 10 | Sort: soonest ETA first, then rising/stalled/declining/reached | ✅ | `rank` map + `projects.sort` |
| 11 | Headline `reachingWithinWindow` counts agents forecast to hit target ≤30d | ✅ | `projects.filter(p => p.etaDays != null && p.etaDays <= WINDOW_DAYS)` |
| 12 | Per-row sparkline with dashed target line + dashed forecast extension (rising only) | ✅ | `page.tsx` target `<line>` + conditional forecast `<line>` gated on `p.etaDays != null` |
| 13 | Slope arrow ↗/→/↘ + status chip (`~7d`/stalled/declining/✓ reached) | ✅ | `slopeArrow()`, `statusLabel()` |
| 14 | Solid line = history, dashed = projection | ✅ | historical `path` solid; forecast line `strokeDasharray="3 2"` |

All 14 claims ✅ verified against merged code on `main` (commit `f276ed6` P202).

## Forward-looking scan
Article is *about* a forecasting feature, so "will / finish / about to" appear frequently — every instance describes the shipped feature's output (projected ETAs) or operator questions it answers, NOT unbuilt roadmap items. No aspirational product claims. PASS.

## Rubric

| Dimension | Score |
|---|---|
| Evidence quality (no raw SHAs in prose) | 5 |
| Technical depth | 5 |
| Clarity for audience | 5 |
| BistecGlobal voice | 5 |
| Title specificity | 4 |

**Total: 24/25** — exceeds 20 threshold. No fixes required.
