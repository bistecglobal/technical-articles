# Audit — 2026-06-24-convergence-context-risk-scatter

**Verdict: PASS**
**Score: 24/25**

## Claim inventory & verdicts

| # | Claim | Verdict | Evidence |
|---|-------|---------|----------|
| 1 | MCD runs a fleet of Claude agents, one subprocess per project | ✅ | Established architecture (prior articles, CLAUDE.md) |
| 2 | Platform tracks context usage + convergence score per project | ✅ | `FleetProject.contextUsagePct`, `convergenceScore` consumed in page |
| 3 | Convergence measures closeness to `GOAL.md` | ✅ | GOAL.md per-project feature (prior shipped); convergenceScore semantics |
| 4 | Scatter: x = context 0–100%, y = convergence 0.0–1.0 | ✅ | `sx`/`sy` scale fns; axis tick labels |
| 5 | Four quadrants; at-risk = high context, low convergence (bottom-right) | ✅ | red `rect` tint at `x≥xT, y below yT`; comment "high context + low convergence = at-risk" |
| 6 | Thresholds 70% context / 0.5 convergence | ✅ | `CTX_THRESHOLD = 70`, `CONV_THRESHOLD = 0.5` |
| 7 | At-risk filter `ctx ≥ 70 AND conv < 50` | ✅ | `atRisk` filter verbatim (`< CONV_THRESHOLD * 100`) |
| 8 | At-risk count surfaced in header | ✅ | header renders `atRisk`, red when >0 |
| 9 | Convergence stored 0–100, displayed 0–1; filter multiplies by 100 | ✅ | `CONV_THRESHOLD * 100` in filter; `/100` in display + axis |
| 10 | Plain hand-drawn SVG, no charting library | ✅ | raw `<svg>`/`<circle>`/`<line>` elements, no chart import |
| 11 | `sy` inverts (`1 - score/100`) for screen orientation | ✅ | `sy = PAD_T + plotH * (1 - convScore / 100)` |
| 12 | Dots colored by state: idle cyan, active green, stalled red, autonomous purple | ✅ | `STATE_COLORS` map verbatim |
| 13 | Hover shows slug, context %, convergence, state + focus link | ✅ | hover breakdown block + `Link href={/focus/...}` |
| 14 | Agents missing either metric are not plotted | ✅ | `points` filter requires both `!= null` |
| 15 | Refreshes every 60s, reuses `/api/fleet` | ✅ | `useFreshness('/api/fleet', 60_000)` |

ASCII quadrant diagram is explicitly illustrative; example agent values (68%/0.45 etc.) are clearly hypothetical, not claimed measurements.

## Forward-looking scan
Matches for "will"/"it'll" are descriptions of *agent runtime behaviour* ("it'll finish before the window fills", "it will reset, forget") — not roadmap/aspirational statements about the product. No forward-looking product claims. PASS.

## SHAs / PR numbers in prose
None.

## Rubric

| Dimension | Score | Notes |
|---|---|---|
| Evidence quality | 5 | Every claim maps to read source; no SHAs in prose |
| Technical depth | 5 | Real constants, filter, scale fns; accurate 0–100/0–1 conversion detail |
| Clarity for audience | 5 | Strong hook, clear quadrant framing, false-alarm payoff |
| BistecGlobal voice | 5 | Practitioner-focused, evidence-grounded |
| Title specificity | 4 | Specific + human; "Two Numbers, One Verdict" earns the colon |

**Total: 24/25 — PASS**
