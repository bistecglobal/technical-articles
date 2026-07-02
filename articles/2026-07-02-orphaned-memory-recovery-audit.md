# Audit — The Note Nobody Links To

**Slug:** 2026-07-02-orphaned-memory-recovery
**Verdict:** PASS
**Score:** 24/25
**Evidence base:** `apps/mission-control/app/api/memory-recovery/route.ts`, `apps/mission-control/app/memory-recovery/page.tsx` (commit `4dadf07`, PR #283, merged to main)

## Claim inventory

| # | Claim | Verdict | Evidence |
|---|---|---|---|
| 1 | Agents keep knowledge as a directory of small Markdown files, one fact per file | ✅ | `scanCurrentOrphans` reads each project's `memory/*.md`; matches MCD memory convention |
| 2 | Files reference each other with `[[note]]` wiki links | ✅ | `extractLinks` regex `/\[\[([^\]]+)\]\]/g` |
| 3 | Orphan = zero inbound AND zero outbound links | ✅ | route.ts:179 `inDegree==0 && outDegree==0` |
| 4 | `/api/memory-recovery` rebuilds the graph from scratch each call, walking every project's `memory/` dir | ✅ | `export const dynamic = 'force-dynamic'`; `getProjectSlugs` + per-project read in `scanCurrentOrphans` |
| 5 | `extractLinks` code snippet | ✅ | verbatim from route.ts:91-93 |
| 6 | Link resolution prefers same-project match, falls back cross-project | ✅ | route.ts:167-169 `sameProject ?? candidates[0]` |
| 7 | Self-links don't count | ✅ | route.ts:170 `targetId === node.id) continue` |
| 8 | Dangling links to a non-existent slug don't count | ✅ | candidates drawn only from `slugToIds` built off real notes |
| 9 | Endpoint never deletes a memory file | ✅ | no `unlink`/`rm` anywhere in route.ts |
| 10 | State in single `memory-recovery.json`, `0600` perms | ✅ | route.ts:71 `writeFileSync(..., { mode: 0o600 })` |
| 11 | `OrphanRecord` interface snippet | ✅ | verbatim route.ts:7-14 |
| 12 | New orphans appended with `firstFlaggedAt` timestamp | ✅ | route.ts:267-279 |
| 13 | Auto-resolve: `relinked` if file still exists, `deleted` if gone | ✅ | route.ts:296-299 |
| 14 | `ignored` set manually via POST | ✅ | POST handler route.ts:346 |
| 15 | Open-orphan table sorted by age; amber ≥3 days, red ≥7 days | ✅ | page.tsx:19-23 `ageColor`; openOrphans sorted desc by daysOrphaned |
| 16 | Eight-week stacked bar chart (relinked/deleted/ignored/still-open) | ✅ | `buildWeeklyHistory` loops `i=7..0` (8 buckets); `StackedBarChart` + `BAR_COLORS` |
| 17 | Per-project sparklines of open-orphan trend, ranked by current count | ✅ | `buildProjectSparklines` sorts by last week's count desc; `Sparkline` component |
| 18 | No new database or background job; runs on demand | ✅ | JSON-file ledger; graph built inside GET handler |
| 19 | (Lesson) many orphans were newly-written facts, not old ones | ⚠️ | Interpretive/anecdotal — plausible given detector is structural not age-based; no fabricated metric, no numeric claim |
| 20 | (Lesson) "relinked" outcome only visible because system waits instead of deleting | ✅ | logically follows from claims 9 + 13 |

## Forward-looking scan

- Line 9 "it *will* start writing things down" — rhetorical present tense, not a roadmap/plan claim. **No action.**
- No matches for plan to / coming soon / in the future / next step / roadmap / soon.
- **Clean.** No forward-looking violations.

## Rubric

| Dimension | Score | Notes |
|---|---|---|
| Evidence quality | 5 | Every technical claim maps to read source; no SHAs/PR numbers in prose |
| Technical depth | 5 | Graph in/out-degree logic, lifecycle reconciliation, real code snippets |
| Clarity for audience | 5 | Clean arc: problem → graph model → mechanics → lessons |
| BistecGlobal voice | 5 | Practitioner, evidence-grounded, no hype |
| Title specificity | 4 | "The Note Nobody Links To" — evocative + specific; not generic |

**Total: 24/25** (threshold 20). ✅

## Notes on claim 19

Sole ⚠️ is an interpretive lesson, not a measurable outcome claim — no numbers, no "we reduced X%". Framed as observation ("The first surprise was how many…"), consistent with editorial rules which bar *fabricated* metrics, not qualitative reflection. Left as-is.

## Fixes applied

None required — no ❌ claims, no forward-looking violations. Frontmatter bumped to `status: audited`.
