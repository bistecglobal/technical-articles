# Audit — The Average Is Lying to You

**Verdict: PASS (after one fix)**
**Score: 23/25**

## Claim inventory

| # | Claim | Verdict | Evidence |
|---|---|---|---|
| 1 | Latency = user msg → first assistant reply | ✅ | `parseTurns` state machine, route.ts:68-89 |
| 2 | Reads Claude Code JSONL transcripts, no new instrumentation | ✅ | `findJsonlFiles` reads `~/.claude/projects/<encoded>/*.jsonl` |
| 3 | Rejects synthetic tool_result user turns | ✅ | `content[0]?.type !== 'tool_result'` route.ts:80 |
| 4 | Drops pairings ≥1h | ✅ | `delta >= 0 && delta < 3600` route.ts:84 |
| 5 | Six percentiles p10/25/50/75/90/99, linear interpolation | ✅ | `percentile()` route.ts:95-102; called 150-155 |
| 6 | Box=p25–p75, whiskers=p10–p90, dot=p50 | ✅ | page.tsx box/whisker/circle render 131-150; legend line 75 |
| 7 | Sorted worst-first by p90 | ✅ | `projects.sort((a,b)=>b.p90-a.p90)` route.ts:164 |
| 8 | Median colour: <30s green, 30–120 amber, >120 red | ✅ | `p50Color` page.tsx:15-19 |
| 9 | <3 samples excluded | ✅ | `if (turns.length < 3) continue` route.ts:141 |
| 10 | Trend = recent-7d p90 vs prior-7d p90, ±10% dead band | ✅ | `computeTrend` route.ts:111-117 |
| 11 | Each window needs ≥3 samples else null/stable | ✅ | `p90OfWindow` window.length<3 → null, route.ts:106 |
| 12 | Glyph ↓ improving / → stable / ↑ degrading | ✅ | `trendGlyph` page.tsx:21-25 |
| 13 | Before/after p90 values shown | ⚠️→✅ | Shown in **summary table** trend column (page.tsx:198-202), NOT hover tooltip. **FIXED** prose. |
| 14 | Opening "average: 41s" | ✅ | Illustrative rhetorical device (strawman), not a claimed production benchmark or achievement. Acceptable. |
| 15 | Two failure modes (long-tail vs drifted-right) | ✅ | Generic descriptions of what percentile/trend views surface; consistent with implemented p99 + p90-trend signals. No fabricated metric. |

## Forward-looking scan
- "the tail tells you what the mean never will" — present tense, no roadmap language. ✅
- No "will/plan to/coming soon/roadmap" referring to unbuilt features. ✅

## Rubric

| Dimension | Score |
|---|---|
| Evidence quality | 5 |
| Technical depth | 5 |
| Clarity | 5 |
| BistecGlobal voice | 4 |
| Title specificity | 4 |
| **Total** | **23/25** |

## Fixes applied
1. Claim #13: "shown on hover" → "shown inline in the summary table" (hover tooltip shows percentile breakdown, not the 7d before/after).
