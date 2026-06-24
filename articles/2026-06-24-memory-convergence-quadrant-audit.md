# Audit — Does a Bigger Memory Make a Smarter Agent? Memory-Convergence Quadrant

**Verdict: PASS** · **Score: 24/25** · Claims fixed: 0 · Claims flagged: 0

## Claim inventory

| # | Claim | Verdict | Evidence |
|---|---|---|---|
| 1 | `/memory-convergence` page plots agents on memory-size × convergence axes | ✅ | `memory-convergence/page.tsx` scatter, `sx(logBytes)`/`sy(conv)` |
| 2 | x = memory size, y = `convergenceScore` (0–1), colour = goal status | ✅ | `Pt` mapping; `goalColor()` |
| 3 | Two dashed guides at memory midpoint & 50% convergence → 4 quadrants | ✅ | `<line ... x={sx(xMid)}>`, `y={sy(0.5)}`, dasharray |
| 4 | Quadrant labels: lean&converging / heavy&converging / lean&stalled / heavy&stalled–prune | ✅ | four `<text>` quadrant labels |
| 5 | Bottom-right (heavy+stalled) = prune candidates; count in header, red when >0 | ✅ | `pruneCandidates = pts.filter(p => p.logBytes > xMid && p.conv < 0.5)`; header colored `#ef4444` |
| 6 | x-axis log-scaled via `log10`, midpoint split on log range, `Math.max(1,…)` guard, conv clamped [0,1] | ✅ | exact code in `Pt` map + `xMid` memo |
| 7 | Coarse Pearson correlation reported as word-bucketed hint with ±0.3 dead zone | ✅ | `corrHint` memo exact |
| 8 | Returns `n/a` when fewer than 3 points | ✅ | `if (n < 3) return 'n/a'` |
| 9 | No backend; reuses `/api/fleet`, derives everything client-side | ✅ | `useFreshness<FleetResponse>('/api/fleet', …)`; page is `'use client'` |
| 10 | Only plots projects with both memory footprint AND convergence score | ✅ | `.filter(p => p.memoryStatus?.exists && sizeBytes>0 && convergenceScore != null)` |
| 11 | Tooltip shows real bytes + conv %; points deep-link to focus view | ✅ | hover tooltip `fmtBytes` + conv; commit notes "deep-links to /focus" |
| 12 | 60s refresh | ✅ | `useFreshness('/api/fleet', 60_000)` |

## Forward-looking scan
- Line 67 "it **will** tell you … how many there are" — describes **current** behaviour of the shipped feature, not a roadmap/future claim. Acceptable.
- No `plan to`/`coming soon`/`in the future`/`next step`/`roadmap`/`soon` matches.

## Rubric

| Dimension | Score | Notes |
|---|---|---|
| Evidence quality | 5 | Every claim maps to `page.tsx`; no SHAs in prose |
| Technical depth | 5 | log-scale rationale, quadrant triage, Pearson honesty (dead zone + n/a) |
| Clarity for audience | 5 | "plot the assumption you never checked" framing; clean arc |
| BistecGlobal voice | 4 | Practitioner, grounded; resists correlation→causation overclaim |
| Title specificity | 5 | Poses the exact question the feature answers |
| **Total** | **24/25** | |

Above 20 threshold. No fixes required.
