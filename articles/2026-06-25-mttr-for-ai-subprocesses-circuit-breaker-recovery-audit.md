# Audit — MTTR for AI Subprocesses: Putting a Stopwatch on Every Circuit Breaker Trip

**Verdict: PASS**
**Score: 24/25**
**Date:** 2026-06-25
**Evidence base:** claude-mcd commits `aba4f72` (P227, #226 cluster) and `bbba5b3` (P233). Files: `src/project-pool.ts`, `apps/mission-control/app/api/circuit-mttr/route.ts`, `app/api/circuit-timeline/route.ts`, `app/circuit-mttr/page.tsx`. Prior context: FailureLedger circuit breaker (blog #11).

## Claim inventory

| # | Claim | Verdict |
|---|-------|---------|
| 1 | Circuit breaker already wraps Claude subprocesses, halts respawn after repeated failures in a window | ✅ verified (FailureLedger; circuit-open on failureCount) |
| 2 | Breaker fires `circuit-open` / `circuit-reset` events on state change | ✅ verified (project-pool.ts fireEvent) |
| 3 | A small change in `fireEvent` appends circuit events to disk before dispatch | ✅ verified (4-line guard added) |
| 4 | Writes one JSON object per line to `circuit-events.jsonl` in each project dir | ✅ verified (appendCircuitEvent) |
| 5 | `durationMs` computed at close time from `circuitOpenAt` (`Date.now() - openAt`) | ✅ verified |
| 6 | Reason strings: "N failures in window" / "auto-reset after cooldown" | ✅ verified |
| 7 | Write wrapped in try/catch that swallows failures (non-critical) | ✅ verified (`catch { /* non-critical */ }`) |
| 8 | MTTR route parses jsonl; totals all-time, opens/week windowed to 30d | ✅ verified (route.ts) |
| 9 | mttrMs = avg durationMs across closes, `null` when none | ✅ verified |
| 10 | Tracks longestOpenMs | ✅ verified |
| 11 | Unclosed = totalOpens − totalCloses surfaced next to open count | ✅ verified (page: "opens with unclosed count") |
| 12 | Fleet view sorts by totalOpens desc | ✅ verified (`sort((a,b)=>b.totalOpens-a.totalOpens)`) |
| 13 | MTTR color green<2m / amber<10m / red beyond | ✅ verified (commit body + page) |
| 14 | Row shows longest, opens/week, 30-day opens/day sparkline | ✅ verified (route + page) |
| 15 | Code snippets (fireEvent, appendCircuitEvent, fold loop, mttr calc) | ✅ verified verbatim against source |

15/15 verified. Zero ⚠️ / ❌.

## Forward-looking scan

No matches for will / plan to / coming soon / in the future / next step / roadmap / soon. Tense is entirely past/present ("we did", "what changed"). PASS.

## Rubric

| Dimension | Score | Notes |
|---|---|---|
| Evidence quality | 5 | Every claim maps to read source; no SHAs/PR# in prose |
| Technical depth | 5 | Real snippets; the write-time durationMs insight is the core and is well-explained |
| Clarity for audience | 5 | SRE framing lands; clean problem→mechanism→payoff arc |
| BistecGlobal voice | 5 | Practitioner, evidence-grounded, transferable lesson |
| Title specificity | 4 | Specific & vivid; "stopwatch" metaphor concrete |

**Total: 24/25** — exceeds 20 threshold.

## Fixes applied

None required. Frontmatter status promoted draft → audited.
