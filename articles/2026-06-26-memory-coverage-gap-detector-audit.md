# Audit — The Agent That Did the Work But Forgot to Write It Down

**Verdict: PASS** · **Score: 23/25** · Slug: `memory-coverage-gap-detector`
Evidence: claude-mcd commit `0041fd7` (#266) — `apps/mission-control/app/api/memory-gaps/route.ts`, `apps/mission-control/app/memory-gaps/page.tsx`, `components/nav-groups.ts`.

## Claim inventory

| # | Claim | Verdict |
|---|---|---|
| 1 | Four canonical memory types: user / feedback / project / reference | ✅ `MEMORY_TYPES` constant |
| 2 | Memory files named `<type>_<slug>.md` or `<type>.md`, classified by prefix | ✅ `f.startsWith(`${t}_`) || f === `${t}.md`` |
| 3 | Analysis is a `readdirSync` + prefix match, no model/embeddings/DB | ✅ route.ts uses only `fs` |
| 4 | No `memory/` dir → all four types missing + 1 penalty | ✅ `gapCount = missingTypes.length + (hasMemoryDir ? 0 : 1)` |
| 5 | gapCount ranges 0–5 | ✅ 4 types + 1 penalty |
| 6 | Active = JSONL transcript modified < 7 days | ✅ `isActiveProject`, `sevenDaysMs` |
| 7 | Transcript dir derived by encoding project real path | ✅ `projectDir.replace(/[^a-zA-Z0-9]/g,'-')` under `~/.claude/projects` |
| 8 | Default sort gap desc, then active first (API) | ✅ `projects.sort(...)` |
| 9 | Gap color: green 0 / amber 1–3 / red 4+ | ✅ page.tsx inline style |
| 10 | Active = green dot; inactive rows dimmed to 55% opacity | ✅ `opacity: p.isActive ? 1 : 0.55` |
| 11 | "Active projects only" toggle | ✅ `activeOnly` state |
| 12 | Every column header is a sort button; resort by any type | ✅ `SortButton` per column |
| 13 | "All projects fully covered" message when no gaps | ✅ `allCovered` branch |
| 14 | One row/project: type checks, mem-dir check, file count, gap | ✅ table body |
| 15 | Added to Intelligence nav | ✅ `nav-groups.ts` (commit stat) |

ASCII table in prose is labeled illustrative (representative layout) — not a claim of specific project data. ✅

## Forward-looking scan
- "will re-discover the same lessons after its next context reset" — general explanation of the failure mode, not a roadmap/product claim. Acceptable.
- No "coming soon", "plan to", "roadmap", "next step". ✅

## Fixes applied
- Reworded "What it surfaced" → "What it makes visible": removed an unmeasured day-one outcome assertion ("several projects lit up…"), reframed as the design intent / what the tool *can* reveal. Complies with "real outcomes only."

## Rubric

| Dimension | Score |
|---|---|
| Evidence quality | 5 |
| Technical depth | 5 |
| Clarity | 5 |
| BistecGlobal voice | 4 |
| Title specificity | 4 |
| **Total** | **23/25** |

Threshold 20/25 — **PASS**. 1 claim fixed, 0 flagged outstanding.
