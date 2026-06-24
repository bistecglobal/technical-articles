# Audit — Is Your Agent Thinking or Just Thrashing?

**Verdict: PASS** · **Score: 24/25** · Claims fixed: 1 (typo "summes"→"sums") · Claims flagged: 0

## Claim inventory

| # | Claim | Verdict | Evidence |
|---|-------|---------|----------|
| 1 | `memory_diff_log` records memory changes with added/removed line counts | ✅ | db fn `SUM(added + removed)` from `memory_diff_log` |
| 2 | `convergence_history` tracks per-agent goal-convergence score over time | ✅ | db fn reads `SELECT slug, score FROM convergence_history` |
| 3 | Join is one db fn summing churn + bracketing convergence per project, 30d window | ✅ | `getMemoryConvergenceXY`; route `WINDOW_DAYS=30` |
| 4 | Convergence reduced to delta (latest − earliest), via date-ordered walk keeping first/last | ✅ | db `ORDER BY ... date ASC`, convBy keeps start + updates end; route `convEnd − convStart` |
| 5 | Churn = total added+removed lines | ✅ | `SUM(added + removed) AS churn` |
| 6 | Fn returns slugs in only one table, padded null/zero; route filters | ✅ | db `slugs = Set([...churnBy, ...convBy])`, `?? null`/`?? 0`; route `.filter(...)` |
| 7 | Route requires ≥1 diff AND ≥2 convergence points (start≠end) | ✅ | `r.diffCount > 0 && r.convPoints >= 2 && convStart != null && convEnd != null` |
| 8 | Dot: churn x, convDelta y, size = diffCount, color by direction | ✅ | page.tsx; route `direction` improving/declining/flat |
| 9 | Quadrant guides: vertical at churn midpoint, horizontal at delta=0; 4 labelled quadrants | ✅ | page lines 88–96 quadrant labels |
| 10 | Bottom-right = thrashing (heavy churn, falling); counted in red header | ✅ | page `thrashing = pts.filter(p => p.churn > xMid && p.convDelta < -0.001)`; red color |
| 11 | Top-right = productive churn (heavy & rising) | ✅ | page label "productive churn — heavy & rising" |
| 12 | Pearson correlation of churn vs convDelta, inline | ✅ | route `pearson()` |
| 13 | Returns null below 3 points (renders n/a) | ✅ | `if (n < 3) return null`; correlationSign 'n/a' |
| 14 | Zero-variance guard returns null (no div-by-zero) | ✅ | `if (dx === 0 \|\| dy === 0) return null` |
| 15 | ±0.3 dead zone: only labelled positive/negative outside it, else 'none' | ✅ | `correlationSign` thresholds `> 0.3` / `< -0.3` / else 'none' |

All 15 claims ✅ verified against merged code on `main` (commit `6865f7f` P201). Distinct from published P176 (#38), which plots memory *size* (log10) vs convergence *score*; this plots churn vs convergence *delta* with Pearson correlation.

## Forward-looking scan
No roadmap/aspirational language. Outcomes stated in past/present. PASS.

## Rubric

| Dimension | Score |
|---|---|
| Evidence quality (no raw SHAs in prose) | 5 |
| Technical depth | 5 |
| Clarity for audience | 5 |
| BistecGlobal voice | 5 |
| Title specificity | 4 |

**Total: 24/25** — exceeds 20 threshold.
