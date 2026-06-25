# Audit — Context Runway: Knowing How Many Turns an AI Agent Has Left

**Verdict: PASS**
**Score: 24/25**
**Date:** 2026-06-25
**Evidence base:** claude-mcd commit `099ca54` (P219). File: `apps/mission-control/app/api/context-horizon/route.ts` (+ `context-horizon/page.tsx`, commit body).

## Claim inventory

| # | Claim | Verdict |
|---|-------|---------|
| 1 | Each Claude session leaves a JSONL transcript, one line per event | ✅ verified |
| 2 | Context size = input_tokens + cache_read_input_tokens + cache_creation_input_tokens | ✅ verified (verbatim) |
| 3 | Reads most-recent transcript per project (by mtime), pulls usage off assistant turns, sorts by timestamp | ✅ verified (findMostRecentJsonl, extractUsageLines) |
| 4 | Rolling window = last 6 turns (5 deltas), only positive deltas counted | ✅ verified (`slice(-6)`, `if (delta > 0)`) |
| 5 | avgGrowthPerTurn = mean of positive deltas | ✅ verified |
| 6 | turnsRemaining = floor((limit − current)/avgGrowth); 0 if no headroom | ✅ verified |
| 7 | Model context limit treated as 200K | ✅ verified (`MODEL_CONTEXT_LIMIT = 200_000`) |
| 8 | Inter-turn interval averaged over same window, gaps > 1h excluded | ✅ verified (`dt < 3_600_000`) |
| 9 | estimatedHoursRemaining = turnsRemaining × avgInterval; null when no data | ✅ verified |
| 10 | Reports `unknown` (not healthy) when growth data absent | ✅ verified (status logic, `avgGrowthPerTurn` null) |
| 11 | Projects sorted ascending by turnsRemaining (soonest-to-exhaust first) | ✅ verified (`?? 9999` sort) |
| 12 | Thresholds: critical < 5, warning < 10, ok beyond | ✅ verified |
| 13 | UI: runway bars, "Needs Reset" panel, header critical/warning counts | ✅ verified (commit body + response fields) |

13/13 verified. Zero ⚠️ / ❌.

## Forward-looking scan

No matches for will / plan to / coming soon / in the future / next step / roadmap / soon (conditional "you'll underestimate" is instructional, not roadmap). Outcomes past/present tense. PASS.

## Rubric

| Dimension | Score | Notes |
|---|---|---|
| Evidence quality | 5 | Every claim maps to read source; no SHAs/PR# in prose |
| Technical depth | 5 | cache-token insight + derivative-vs-position framing well grounded |
| Clarity for audience | 5 | Clean arc; runway metaphor consistent |
| BistecGlobal voice | 5 | Practitioner, transferable lessons |
| Title specificity | 4 | Specific & concrete; slightly long |

**Total: 24/25** — exceeds threshold.

## Fixes applied

None required. Frontmatter status promoted draft → audited.
