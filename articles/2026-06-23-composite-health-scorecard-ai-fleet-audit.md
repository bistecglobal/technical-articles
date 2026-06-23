# Audit: composite-health-scorecard-ai-fleet
Date: 2026-06-23

## Verdict: PASS (with minor fixes applied)

---

## Claim Inventory

| # | Claim | Verdict | Evidence |
|---|-------|---------|----------|
| 1 | "We run fifteen AI agents continuously" | ⚠️ weak | channels.json shows 19 configured channels; "fifteen" is an undercount |
| 2 | "each in its own Claude Code session" | ✅ | claude-process.ts — one ClaudeProjectProcess per slug/chatId |
| 3 | Dashboard tracked turn quality, memory health (5 dims), goal keyword hit rates, context pressure, anomalies | ✅ | All 5 APIs exist: /api/turn-quality, /api/memory-health, /api/goal-heatmap, /api/context-pressure, /api/anomalies |
| 4 | Turn Quality weight 30% | ✅ | W.turnQuality = 0.30 in scorecard/page.tsx |
| 5 | "over the last 24 hours" (turn quality) | ✅ | cutoff = Date.now() - 24 * 3_600_000 in turn-quality/route.ts |
| 6 | Scored by reply length, tool call count, absence of error patterns | ✅ | lenScore (40%), toolScore (30%), errorPenalty (30%) in scoreGroup() |
| 7 | Memory Health weight 30% | ✅ | W.memoryHealth = 0.30 in scorecard/page.tsx |
| 8 | Five sub-dimensions: recency, coverage, density, stability, freshness | ✅ | MemoryHealthDimensions interface and scoreRecency/scoreCoverage/scoreDensity/scoreStability functions in memory-health/route.ts |
| 9 | "recency: updated in last 7 days" | ✅ | scoreRecency: ageDays < 7 → 100 |
| 10 | "coverage: at least 3 memory files" | ✅ | scoreCoverage: fileCount >= 3 → 100 |
| 11 | "density: files contain enough content" | ✅ | scoreDensity: wordCount >= 500 → 100, linear below |
| 12 | "stability: memory stable, not thrashing" | ✅ | scoreStability: inverse stale fraction |
| 13 | "freshness: memory written after last session" | ✅ | 0 or 100 binary based on memory vs transcript mtime |
| 14 | "memory not touched in three weeks loses most of its recency score" | ✅ | At 21 days: (30-21)/23 * 100 ≈ 39 — 61% of recency score lost |
| 15 | Goal Progress weight 20% | ✅ | W.goalProgress = 0.20 |
| 16 | GOAL.md "injected at the start of every session" | ✅ | formatPrompt() in claude-process.ts prepends <goal>...</goal> on firstMessageSent=false |
| 17 | "goal heatmap tracks...day by day, over the last 30 days" | ✅ | GoalHeatmapResponse.dates = "last 30 days" in goal-heatmap/route.ts |
| 18 | Context Pressure weight 10% | ✅ | W.contextPressure = 0.10 |
| 19 | "A Claude model has a 200k-token context limit" | ✅ | MODEL_CONTEXT: all Claude variants = 200_000 in context-pressure/route.ts |
| 20 | "invert the pressure score so low context usage = high health" | ✅ | cpMap[p.slug] = Math.max(0, 100 - p.score) in scorecard/page.tsx |
| 21 | Anomaly Score weight 10% | ✅ | W.anomalyScore = 0.10 |
| 22 | "zero anomalies earns a perfect 100" | ✅ | count=0 → Math.round(Math.max(0, 100 - 0)) = 100 |
| 23 | "z-score detection with a 7-day baseline" | ✅ | sevenDaysMs = 7 * 24 * 60 * 60 * 1000 in anomalies/route.ts |
| 24 | "deviations in inter-turn gap, tool calls per turn, output tokens per turn" | ✅ | metrics array: interTurnGapMins, toolCallsPerTurn, outputTokensPerTurn |
| 25 | Code snippet (computeOverall) | ✅ | Exact code from scorecard/page.tsx |
| 26 | Null handling: goal null excluded, weight redistributed | ✅ | totalW accumulates only non-null dimensions |
| 27 | "six API requests in parallel on load" | ✅ | Promise.all([fleetRes, tqRes, mhRes, ghRes, cpRes, anRes]) — 6 fetches |
| 28 | "Default sort: overall ascending (worst first)" | ✅ | sortKey='overall', sortAsc=true; for numeric, ascending = lowest first |
| 29 | "≥70 green, 40–69 amber, <40 red" | ✅ | scoreColor function in scorecard/page.tsx |
| 30 | "Rows with critical overall scores get a subtle red background tint" | ✅ | rowBg = rgba(239,68,68,0.05) when overall < 40 |
| 31 | "Clicking score cell navigates to detail page for that dimension" | ✅ | COLUMNS array — each column has href pointing to detail page |
| 32 | "Clicking row expands quick-link accordion with all five links" | ✅ | DETAIL_LINKS has 5 entries; expanded row renders them |
| 33 | "auto-refreshes every five minutes" | ✅ | setInterval(() => void fetchData(), 5 * 60 * 1000) |
| 34 | "We iterated on them three times before shipping" | ⚠️ weak | scorecard/page.tsx has only one commit (1757044); no prior weight history in git |
| 35 | "Early versions weighted anomalies higher (20%)" | ⚠️ weak | No prior version of the file in git history to confirm |
| 36 | "Moving anomalies to 10% and giving that weight to turn quality" | ⚠️ weak | Same — single-commit file |

---

## Forward-Looking Scan

Searched for: "will", "plan to", "coming soon", "in the future", "next step", "roadmap", "soon"

**No forward-looking statements found.** ✅

---

## Rubric Scorecard

| Dimension | Score | Notes |
|---|---|---|
| Evidence quality (claims verifiable, no raw SHAs in prose) | 5/5 | All technical claims verified against production code; no SHAs in prose |
| Technical depth | 5/5 | Aggregation logic explained with real code; scoring formulas grounded in APIs |
| Clarity for target audience | 4/5 | Well-paced; the "null is not zero" section is the clearest lesson |
| BistecGlobal voice | 5/5 | Practitioner-focused, first-person, specific without being dry |
| Title specificity | 5/5 | "One Number Per Agent" is memorable and precise |

**Total: 24/25** — PASS

---

## Fixes Applied

1. **Claim #1**: "fifteen AI agents" → "close to twenty AI agents" (channels.json has 19 configured channels)
2. **Claims #34-36**: Removed specific "three times" iteration count and "early versions (20%)" claim; replaced with a design-intent framing that is fully verifiable: anomalies at 10% is a deliberate choice to treat them as early-warning rather than primary signal.
