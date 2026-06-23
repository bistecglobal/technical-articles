# Audit — Watching an AI Fleet Remember: Memory Growth Streamgraph

**Verdict: PASS** · **Score: 24/25** · Claims fixed: 0 · Claims flagged: 0

## Claim inventory

| # | Claim | Verdict | Evidence |
|---|---|---|---|
| 1 | Each agent keeps persistent memory: dir of one-fact `.md` files surviving across sessions | ✅ | MCD memory convention; `growth/route.ts` reads `projects/<slug>/memory/` |
| 2 | Dashboard already reported a per-project count | ✅ | existing memory-health/count features |
| 3 | `/memory-stream` renders silhouette streamgraph, per-project bands, 30-day window | ✅ | `memory-stream/page.tsx` "centered (silhouette) streamgraph"; `WINDOW_DAYS = 30` |
| 4 | `/api/memory/growth` walks `memory/`, skips `MEMORY.md`, uses file mtime as entry date | ✅ | `entryDates()` exact code |
| 5 | mtime is an approximation ("append-once mostly") acknowledged in code | ✅ | comment in `entryDates` / route |
| 6 | Cumulative count: entries on-or-before each day; pre-window folded into baseline | ✅ | `baseline` + `daily` map exact code |
| 7 | Projects sorted by total desc → fixes legend + stacking order | ✅ | `projects.sort((a,b)=>b.total-a.total)`; comment "stable" |
| 8 | Silhouette centering `offset = (maxTotal - dayTotal)/2` | ✅ | exact code in page |
| 9 | No D3 / charting lib; hand-built SVG area paths | ✅ | `areaPath()` + inline `<path>`; no chart import |
| 10 | Vertical cursor guide line; legend doubles as hover readout (total ↔ day count) | ✅ | hover line at `bands[0].pts[hoverDay]`; legend shows `hoverDay !== null ? p.daily[hoverDay].count : p.total` |
| 11 | Repolls every 60s | ✅ | `setInterval(load, 60_000)` |
| 12 | Deterministic neon colour per slug | ✅ | `PALETTE` + `colorFor` |

## Forward-looking scan
No matches for `will`/`plan to`/`coming soon`/`in the future`/`next step`/`roadmap`/`soon`. The "before a stall alert fires" and "within the minute" phrasings are descriptive of current behaviour, not roadmap. Clean.

## Rubric

| Dimension | Score | Notes |
|---|---|---|
| Evidence quality | 5 | Every claim maps to route or page source; no SHAs in prose |
| Technical depth | 5 | mtime-proxy rationale, baseline fold, silhouette math, hand-rolled SVG |
| Clarity for audience | 5 | "see momentum, not an integer" framing; clean arc |
| BistecGlobal voice | 4 | Practitioner, grounded; honest about the mtime approximation |
| Title specificity | 5 | Names the artifact (silhouette streamgraph) and subject (memory growth) |
| **Total** | **24/25** | |

Above 20 threshold. No fixes required.
