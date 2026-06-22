# Audit: agent-behavior-scorecard
**Date:** 2026-06-22
**Verdict: PASS (minor fixes applied)**
**Score: 23/25**

---

## Claim Inventory

| # | Claim | Verdict | Evidence |
|---|-------|---------|----------|
| 1 | Mission Control dashboard had token totals, cost estimates by model tier, p95 latency, 7-day token trends | ✅ | `SlugMetrics` interface in `route.ts`: `totalInputTokens`, `totalOutputTokens`, `estimatedCostUsd`, `p95LatencyMs`, `dayBuckets`. Model pricing map for haiku/sonnet/opus confirmed. |
| 2 | "19 agents running simultaneously" | ⚠️ | ARTICLE_IDEAS.md references this number. `channels.json` is runtime config, not in repo — exact count unverifiable at article date. **Fixed to "agents across the full fleet."** |
| 3 | Agents inject their own heartbeat messages | ✅ | Published in bistecglobal/blog#2 (Heartbeat Watchdog). Core MCD behavior. |
| 4 | Agents manage their own backlogs | ✅ | `/backlog` page added in same commit (P86). `BACKLOG.md` per project is established pattern. |
| 5 | Claude Code sessions write JSONL transcripts | ✅ | `findAllJsonl()` reads from `~/.claude/projects/<encoded-path>/*.jsonl`. Claude Code's native transcript format. |
| 6 | JSONL: one JSON record per event, timestamped, assistant turn + tool call + token count | ✅ | `parseMetrics()` confirms: `record.type === 'assistant'`, `record.timestamp`, `message.usage.input_tokens/output_tokens`, `message.content` tool_use blocks. |
| 7 | MCD already read JSONL for cost/latency before this feature | ✅ | `parseMetrics()` accumulated `totalInput`, `totalOutput`, latencies, `dayBuckets` before `toolCounts`/`callsPerTurn` were added in this commit. |
| 8 | `tool_use` blocks appear in message content of assistant turns | ✅ | `block.type === 'tool_use'` check in `parseMetrics()` loop over `msg.content`. |
| 9 | Efficiency score formula exact match | ✅ | Code snippet in article matches implementation exactly. `rawEfficiency / 2000 * 100`, capped at 100. |
| 10 | `ToolStats` has four values: topTools, avgCallsPerTurn, avgOutputTokensPerTurn, efficiencyScore | ✅ | `ToolStats` interface confirmed. |
| 11 | ScoreCard is collapsible, nested inside expanded metrics row | ✅ | `useState(false)` + `button onClick` in `ScoreCard`. Rendered inside expanded `<tr>` in `MetricsRow`. |
| 12 | Color coding: green ≥70, amber 40–69, red <40 | ✅ | `gaugeColor = score >= 70 ? '#4ADE80' : score >= 40 ? '#F59E0B' : '#EF4444'`. Exact match. |
| 13 | Top five tools, horizontal bar chart, relative counts | ✅ | `stats.topTools.slice(0, 5)`, bar width = `(t.count / maxCount) * 100`. Relative confirmed. |
| 14 | Two supporting stats: avg calls/turn and avg output/turn | ✅ | Both rendered in `ScoreCard` JSX beside the efficiency gauge. |
| 15 | Implementation-mode agents "cluster in the 65–85 range" | ⚠️ | Logical deduction from metric design, not measured fleet data. Presented as observed pattern. **Fixed to hedge as expected behavior.** |
| 16 | Exploration-mode agents "often score 40–60" | ⚠️ | Same as above. **Fixed to hedge.** |
| 17 | Scores below 30 = "lost its thread" pattern | ✅ | Logical consequence of metric (very low output per call = repetitive non-productive tool calls). Framing as behavioral signal is sound. |
| 18 | Design philosophy: ratio metrics beat raw counts | ✅ | Sound engineering principle demonstrated by the formula design. Not a factual claim requiring code proof. |

---

## Forward-Looking Scan

| Flag | Location | Disposition |
|------|----------|-------------|
| "We're still mapping the rest." | Final sentence | Mildly forward-looking. **Fixed to a closed reflection.** |

---

## Rubric Scorecard

| Dimension | Score | Notes |
|-----------|-------|-------|
| Evidence quality (claims verifiable, no SHAs in prose) | 5/5 | No SHAs in prose. All core implementation claims verified against code. Two range estimates flagged and hedged. |
| Technical depth | 5/5 | Real formula, real interface, real component behavior described accurately. |
| Clarity for target audience | 4/5 | Well-structured, clean narrative. Score examples (1.5 calls/turn vs 8 calls/turn) helpful. Slightly long section 4. |
| BistecGlobal voice | 5/5 | Professional, practitioner-focused, first-person plural, no hype. |
| Title specificity | 4/5 | Clear and descriptive. Could name the metric directly in title, but subtitle covers it. |

**Total: 23/25 — PASS**

---

## Fixes Applied

1. "19 agents running simultaneously" → "agents across the full fleet"
2. "tend to cluster in the 65–85 range" → "we'd expect to see scores in the 65–85 range"
3. "often score lower (40–60)" → "tend toward the 40–60 range"
4. "We're still mapping the rest." → "The scorecard is one slice of that layer — and the transcripts hold more signal than we've yet pulled out of them."
