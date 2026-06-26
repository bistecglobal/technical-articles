# Audit — The Sankey That Shows How Agents Move

**Verdict: PASS**
**Score: 24/25**

Evidence: `apps/mission-control/app/api/state-transitions/route.ts`, `apps/mission-control/app/api/tool-error-rate/route.ts`, `app/state-transitions/page.tsx`, `app/tool-error-rate/page.tsx` (commit `181baff`, on `main`).

## Claim inventory

| # | Claim | Verdict | Evidence |
|---|---|---|---|
| 1 | Four states: idle/active/stuck/circuit-open | ✅ | StateLabel type, STATES array (route:7,208) |
| 2 | 30-day window, per project, all projects | ✅ | windowMs = 30*24h (route:196); loops slugs |
| 3 | Combines genuine user messages + circuit-events.jsonl | ✅ | extractMessageTimestamps + extractCircuitEvents |
| 4 | tool_result-only "user" lines excluded as not genuine | ✅ | isGenuineUserMessage (route:60-65) |
| 5 | 30-min gap = idle | ✅ | IDLE_GAP_MS = 30*60*1000 (route:104) |
| 6 | Events merged into one chronological list, single pass builds intervals | ✅ | buildStateIntervals (route:112-181) |
| 7 | Messages ignored while circuit open; close → idle | ✅ | route:152, 154-155 |
| 8 | from→to matrix: count, avg duration in from-state, state time-share % | ✅ | transCount/transTotal, Transition, StateTotal |
| 9 | TS control-flow narrowing cast comment | ✅ | route:147-148 |
| 10 | SVG Sankey: node height ∝ entries, flow width ∝ count, time% labels | ✅ | commit msg + state-transitions/page.tsx |
| 11 | tool-error-rate reads same JSONL | ✅ | tool-error-rate/route.ts findJsonlFiles |
| 12 | Two-pass: toolUseId→name map, then tool_result errors | ✅ | parseJsonl (route:93-142) |
| 13 | is_error via flag or content type 'error' | ✅ | route:126-131 |
| 14 | Per tool: calls, errors, 1-decimal rate%, last error ts+snippet, 14d sparkline | ✅ | ToolErrorStat, buildSparkline |
| 15 | mcp__mcd__* filtered unless include_mcd=1 | ✅ | route:118, 165 |
| 16 | Table sorts worst-rate-first | ✅ | sort errorRate desc (route:205) |
| 17 | No agent instrumentation added — derived from existing logs | ✅ | both routes read-only over transcripts |

17/17 verified. No ⚠️/❌.

## Forward-looking scan
No will / plan / coming soon / roadmap / next step describing unbuilt work. Clean.

## Rubric

| Dimension | Score |
|---|---|
| Evidence quality | 5 |
| Technical depth | 5 |
| Clarity for audience | 5 |
| BistecGlobal voice | 5 |
| Title specificity | 4 |

**Total: 24/25** — exceeds 20 threshold. No fixes required.
