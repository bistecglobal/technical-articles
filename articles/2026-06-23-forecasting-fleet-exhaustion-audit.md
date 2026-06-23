# Audit — When Does the Fleet Run Out? Forecasting Token Burn and Backlog Velocity

**Verdict: PASS** · **Score: 24/25** · Claims fixed: 0 · Claims flagged: 0

## Claim inventory

| # | Claim | Verdict | Evidence |
|---|---|---|---|
| 1 | One Claude subprocess per project, optional `monthlyTokenBudget` | ✅ | `channels.json` projects/defaults carry `monthlyTokenBudget`; burn-rate route reads it |
| 2 | Budget alerts fire at 50/80/100% | ✅ | Prior shipped budget-enforcement feature (blog #16) |
| 3 | Burn-rate reads JSONL transcripts, walks `assistant` records, sums `input_tokens + output_tokens` | ✅ | `burn-rate/route.ts` `computeBurn` |
| 4 | Month-to-date = tokens since UTC first-of-month | ✅ | `monthStart.setUTCDate(1)` + `monthStartMs` filter |
| 5 | Seven daily buckets, oldest→newest | ✅ | `spark[7]`, `dayKeys` loop |
| 6 | `dailyRate = last7Total / 7` | ✅ | exact code |
| 7 | `projectedMonthEnd`, `daysUntilExhausted` division formula | ✅ | exact code |
| 8 | `null` when no budget or rate zero (not Infinity) | ✅ | `daysUntilExhausted = ... ? ... : null` |
| 9 | `/burn-rate` page: sparkline, budget bar, exhaust badge | ✅ | `Sparkline`, `exhaustLabel`, `exhaustColor` (red ≤7/amber/green) |
| 10 | Sorted by `daysUntilExhausted` ascending, fleet total pinned on top | ✅ | `projects.sort` + commit "fleet-total row pinned to top" |
| 11 | Forecast builds on burndown series, calls its `GET` directly, consumes JSON | ✅ | `import { GET as burndownGET }`, `await burndownGET()` |
| 12 | `sevenDayRates` over whole series | ✅ | exact code |
| 13 | Three scenarios: optimistic (fastest), pessimistic (slowest non-zero), expected (trailing 14-day mean) | ✅ | exact code |
| 14 | Projected date `null` when rate ≤ 0 | ✅ | `projectDate` returns null |
| 15 | Fan chart band widens further out | ✅ | band `low`/`high` diverge by rate×d; forecast page renders band |
| 16 | Forecast has no filesystem access of its own | ✅ | route only imports burndown; no `fs` |

## Forward-looking scan
One `will` match (line 9): *"when will we be over budget?"* — rhetorical framing of the user's question, not a product roadmap statement. **Not a violation.** No `plan to`/`coming soon`/`roadmap`/`next step` matches.

## Rubric

| Dimension | Score | Notes |
|---|---|---|
| Evidence quality | 5 | Every claim traces to code; no SHAs in prose |
| Technical depth | 5 | Real code, window rationale, null-honesty, single-source-of-truth design |
| Clarity for audience | 5 | Strong hook, clean arc, generalizable lessons |
| BistecGlobal voice | 4 | Practitioner, evidence-grounded; smoke-detector metaphor leans slightly heavy but earns it |
| Title specificity | 5 | Concrete, names both forecasts |
| **Total** | **24/25** | |

Above 20 threshold. No fixes required.
