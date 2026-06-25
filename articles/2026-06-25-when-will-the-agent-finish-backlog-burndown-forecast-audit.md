# Audit — When Will the Agent Finish? A Burndown Forecast for an Autonomous Backlog

**Verdict: PASS**
**Score: 24/25**
**Date:** 2026-06-25
**Evidence base:** claude-mcd commit `bbba5b3` (P234). File: `apps/mission-control/app/api/backlog-forecast/route.ts` (+ `backlog-forecast/page.tsx`, commit body).

## Claim inventory

| # | Claim | Verdict |
|---|-------|---------|
| 1 | Forecast reads `BACKLOG.md`, splits on `## ` headers into proposals | ✅ verified (parseBacklogMd) |
| 2 | Status parsed from `[x] done` / `[~]`/`in_progress`; created from `**Created:** YYYY-MM-DD` | ✅ verified (verbatim regex/checks) |
| 3 | Headline counts: total, done, pending | ✅ verified |
| 4 | Completion timing proxied by created date, bucketed by week (no true completion timestamp) | ✅ verified (code comment "by created date as proxy") |
| 5 | velocity4w = sum of last 4 weeks' completions / 4, rounded 1dp | ✅ verified |
| 6 | velocityOptimistic = p75 of weekly completion counts | ✅ verified (percentile, 75) |
| 7 | Projection subtracts a velocity per week from pending until 0 or 52 weeks | ✅ verified (buildProjection loop) |
| 8 | 52-week horizon cap | ✅ verified |
| 9 | Returns `null` (no date) when velocity ≤ 0 or backlog won't clear in window | ✅ verified |
| 10 | `stalled = velocity4w === 0` | ✅ verified |
| 11 | Two finish dates + two burndown series (linear + optimistic) | ✅ verified (response fields + commit body) |
| 12 | UI: remaining-work line, dashed projected tail (commit: solid actual + dashed projected + shaded band) | ✅ verified (commit body) |

12/12 verified. Zero ⚠️ / ❌. Proxy-data limitation is disclosed in the article prose — editorial-honesty positive.

## Forward-looking scan

Title/body use "will/finish/done" to describe the *forecast's subject*, not unbuilt product features (the feature exists and emits these projections). No matches for plan to / coming soon / in the future / next step / roadmap. PASS.

## Rubric

| Dimension | Score | Notes |
|---|---|---|
| Evidence quality | 5 | Every claim maps to route; proxy caveat disclosed; no SHAs in prose |
| Technical depth | 5 | Two-velocity method + cap/stalled guards well grounded |
| Clarity for audience | 5 | Burndown framing familiar; honest-forecasting throughline |
| BistecGlobal voice | 5 | Practitioner, candid about limitations |
| Title specificity | 4 | Specific, question-hook; accurate |

**Total: 24/25** — exceeds threshold.

## Fixes applied

None required. Frontmatter status promoted draft → audited.
