# Audit — Don't Wait for the Watchdog: Forecasting AI Agent Stalls Before They Fire

**Verdict: PASS**
**Score: 24/25**
**Claims fixed: 0 · Claims flagged: 0**

Evidence base: `apps/mission-control/app/api/stall-risk/route.ts`, `app/stall-risk/page.tsx`, `components/FleetAdvisorPanel.tsx` (commit `a67e551`, PR #166, on `main`). Supporting: circuit breaker `FailureLedger` (blog #11), reactive stall panel P3.

## Claim inventory

| # | Claim | Verdict | Evidence |
|---|---|---|---|
| 1 | Reactive stall alert panel trips on time-since-reply crossing watchdog threshold | ✅ | P3 panel referenced in route doc-comment; `stuckThresholdMinutes` used |
| 2 | Circuit breaker backs off repeatedly-crashing agents | ✅ | FailureLedger, blog #11 |
| 3 | Score is 0–100 per project, sorted worst-first | ✅ | `projects.sort((a,b)=>b.score-a.score)`; clamp 0–100 |
| 4 | Four heuristics weighted 30/30/25/15 | ✅ | weights in `contextRisk`/`qualityRisk`/`recencyRisk`/`historyRisk` |
| 5 | Context: usage % + penalty if fill ETA < 30m | ✅ | `risk = clamp(pct + (risingFast ? 25 : 0))`, `eta < 30` |
| 6 | Quality: latest score, +penalty if falling 10+ over last 3 turns | ✅ | `declining = recent[0]-recent[last] >= 10`, `+15` |
| 7 | Recency: time-since-reply as fraction of project threshold (80%→80) | ✅ | `(ageMins/threshold)*100` |
| 8 | Kill history: stall/watchdog/circuit events in last 24h | ✅ | `getAlertEvents`, 24h cutoff, `/stall|watchdog|circuit/i` |
| 9 | Score = clamped sum of risk×weight | ✅ | exact reduce shown |
| 10 | Missing signals omitted, weights NOT renormalized | ✅ | `qualityRisk`/`recencyRisk` return null |
| 11 | Context+kill always present (45%); other 55% earned with data | ✅ | only quality+recency are nullable |
| 12 | Reuses `/api/fleet` as single source of truth | ✅ | `fetch('/api/fleet')`; doc-comment states it |
| 13 | factors filtered ≥8 pts, sorted strongest-first | ✅ | `.filter(c=>c.contribution>=8).sort(...)` |
| 14 | Factor labels (context%, quality dropping, silent m/threshold) | ✅ | label template strings match verbatim |
| 15 | Page color bands: green <40, amber ≥40, red ≥80 | ✅ | `riskColor` thresholds |
| 16 | Pre-inject opens inject terminal with exact check-in message | ✅ | `PRE_INJECT_MESSAGE` matches quoted text |
| 17 | Fleet Advisor widget shows top-3 at-risk with scores | ✅ | `.filter(p=>p.score>=40).slice(0,3)` |
| 18 | No new instrumentation; recombines existing data | ✅ | all inputs from existing endpoints/tables |
| 19 | Runs ~every minute | ✅ | `/stall-risk` page 60s refresh |

## Forward-looking scan
No roadmap/aspirational language. "next step" appears only inside the quoted pre-inject prompt (a literal string), not as a future claim. ✅

## Rubric

| Dimension | Score |
|---|---|
| Evidence quality | 5 |
| Technical depth | 5 |
| Clarity | 5 |
| BistecGlobal voice | 5 |
| Title specificity | 4 |

**Total: 24/25** — clears 20 threshold. No raw SHAs/PR numbers in prose. Approved for publish.
