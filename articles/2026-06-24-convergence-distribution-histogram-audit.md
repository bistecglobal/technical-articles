# Audit — 2026-06-24-convergence-distribution-histogram

**Verdict: PASS**
**Score: 24/25**

## Claim inventory & verdicts

| # | Claim | Verdict | Evidence |
|---|-------|---------|----------|
| 1 | MCD runs a fleet of Claude agents, one subprocess per project | ✅ | Established architecture |
| 2 | Convergence score 0–1, from goal-advancing reply turns over last 24h | ✅ | Page footnote: "derived from goal-advancing reply turns over the last 24h"; `convergenceScore` |
| 3 | Dashboard previously showed fleet mean convergence | ✅ | Header renders `mean` (kept in this view) |
| 4 | Histogram buckets into ten bins | ✅ | `BIN_COUNT = 10` |
| 5 | Bin = `floor(score/10)`, clamped so top bin captures ≥90 | ✅ | `Math.min(BIN_COUNT-1, Math.floor(score/10))` |
| 6 | Colour = red→amber→green hue 0→120 ramp by bin position | ✅ | `binColor`: `hue = (i/(BIN_COUNT-1))*120`, `hsl(hue,75%,55%)` |
| 7 | Header shows mean + count in top bin (≥0.9), green only when >0 | ✅ | `topBin`; color `topBin > 0 ? '#34d399' : '#475569'` |
| 8 | Hand-drawn SVG; bars scaled to busiest bin; count above bar; bin labels | ✅ | `<svg>` bars, `count` text, `binLabel` axis ticks |
| 9 | Hover lists member project slugs, each linking to focus view | ✅ | hover block maps `bins[hover]` → `Link href={/focus/...}` |
| 10 | Agents without a score aren't counted | ✅ | `if (p.convergenceScore == null) continue` |
| 11 | Bar height scales against `Math.max(1, …)` busiest bin | ✅ | `maxCount = Math.max(1, ...bins.map(b => b.length))` |
| 12 | Reuses `/api/fleet`, refreshes every 60s | ✅ | `useFreshness('/api/fleet', 60_000)` |
| 13 | Shape interpretations (unimodal-right, bimodal, left-skew) | ✅ | Valid statistical reading of the histogram; presented as interpretation not measurement |

No fabricated metrics; example shapes are explicitly illustrative.

## Forward-looking scan
None (0 matches).

## SHAs / PR numbers in prose
None.

## Rubric

| Dimension | Score | Notes |
|---|---|---|
| Evidence quality | 5 | Every claim maps to read source |
| Technical depth | 5 | Bin math, hue-by-position rationale, divide-by-zero guard all accurate |
| Clarity for audience | 5 | "mean hid a split fleet" hook + bimodal payoff land cleanly |
| BistecGlobal voice | 5 | Practitioner-focused, worklist-not-status lesson |
| Title specificity | 4 | Specific + human; "The Average Hid a Split Fleet" earns it |

**Total: 24/25 — PASS**
