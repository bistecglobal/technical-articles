# Audit — One Heatmap, Every Tool (tool-usage-heatmap)

**Verdict: PASS**
**Score: 24/25**
**Claims: 17 verified, 0 weak, 0 unverified · 0 flagged · 0 fixed**

## Claim inventory

| # | Claim | Evidence | Verdict |
|---|---|---|---|
| 1 | Project × tool matrix, rows=projects cols=tools, color encodes count | `api/tool-heatmap/route.ts` ToolHeatmapResponse + page.tsx HeatMatrix | ✅ |
| 2 | Reads existing JSONL transcripts, `tool_use` content blocks | `countToolCalls` parses `block.type === 'tool_use'` | ✅ |
| 3 | `countToolCalls` snippet | Verbatim from route.ts | ✅ |
| 4 | Malformed line skipped not fatal | try/catch `continue` on JSON.parse | ✅ |
| 5 | Top 20 projects by total, top 30 tools by fleet total | `slice(0,20)` / `slice(0,30)` in route.ts | ✅ |
| 6 | rowTotals/colTotals marginal sums | computed in route.ts, rendered as bars | ✅ |
| 7 | Gamma `Math.pow(count/max, 0.55)` color curve | `cellColor` verbatim | ✅ |
| 8 | Zero cell near-transparent `rgba(255,255,255,0.03)` | `cellColor` guard | ✅ |
| 9 | Hand-drawn SVG, no chart library | page.tsx uses raw `<svg>`/`<rect>`/`<text>`, no imports of chart lib | ✅ |
| 10 | Column labels rotated −55° | `transform={rotate(-55...)}` | ✅ |
| 11 | `mcp__server__` prefix stripped | `.replace(/^mcp__[^_]+__/, '')` | ✅ |
| 12 | Hover tooltip project·tool·count | TooltipState + onMouseEnter | ✅ |
| 13 | 4 files: 126-line route, 221-line page, nav entry, backlog note | `git show --stat c2a4a2e`: route 126, page 221, nav-groups 1, BACKLOG 120 | ✅ |
| 14 | 30-day window | default `days=30`, cutoffMs | ✅ |
| 15 | Added under Observability nav | commit msg + nav-groups.ts change | ✅ |
| 16 | Per-agent tool scorecard already existed | prior P d9b4814 / blog#24 ScoreCard | ✅ |
| 17 | No schema migration / new storage / agent instrumentation | diff touches only 4 files, no migration/db | ✅ |

## Forward-looking scan
No matches for will/plan to/coming soon/in the future/next step/roadmap/soon. "ships today" and "It didn't make our agents smarter" are present/past tense. Clean.

## Rubric

| Dimension | Score |
|---|---|
| Evidence quality (no raw SHAs in prose) | 5 |
| Technical depth | 5 |
| Clarity for audience | 5 |
| BistecGlobal voice | 4 |
| Title specificity | 5 |
| **Total** | **24/25** |

No fixes required. Status → audited.
