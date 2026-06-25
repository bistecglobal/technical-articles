# Audit — Which Alerts Travel Together? A Co-occurrence Graph for an AI Agent Fleet

**Verdict: PASS**
**Score: 24/25**
**Source of truth:** claude-mcd commit `5fb93ca` (PR #212), files `app/api/signal-cooccurrence/route.ts`, `app/signal-graph/page.tsx`, `components/nav-groups.ts`, `src/db.ts`.

## Claim inventory

| # | Claim | Verdict | Evidence |
|---|---|---|---|
| 1 | Unified attention engine emits typed findings with severity | ✅ | `computeFindings` in `lib/attention-findings`, signal+severity fields |
| 2 | Findings written to `attention_event` table on each Brief compute | ✅ | `getAttentionEvents` reads `attention_event`; P209 |
| 3 | Route `/api/signal-cooccurrence` builds weighted graph | ✅ | route.ts `buildGraph`, `CooccurrenceResponse` |
| 4 | Bucket = project-day in history mode (`date\|slug`) | ✅ | `const key = \`${e.date}\|${e.slug}\`` |
| 5 | Each signal pair in a bucket forms an edge; weight = co-occurrence count | ✅ | nested loop over sorted signals, `edgeWeight` |
| 6 | Worst severity wins per node | ✅ | `SEV_RANK` comparison in nodeAcc |
| 7 | Signals sorted before pairing → stable `a\|b` key, no double count | ✅ | `signals = Array.from(...).sort()` |
| 8 | Healthy/ok signals dropped before bucketing | ✅ | `isAttention` guard |
| 9 | Response includes maxima + `dominantSignal` (most-fired node) | ✅ | response object, `nodes[0]?.signal` |
| 10 | Degrades to live snapshot when no history; buckets by project | ✅ | `mode = 'live'`, `computeFindings`, `key = f.slug` |
| 11 | Page badges history vs live mode | ✅ | header mode badge in page.tsx |
| 12 | Reuses MemoryGraph force simulation | ✅ | d3 forceSimulation, comment "mirrors MemoryGraph's pattern" |
| 13 | Node size sqrt-scaled by firings | ✅ | `nodeRadius`: `Math.sqrt(count/max)` |
| 14 | Link distance/strength scale with edge weight | ✅ | forceLink `.distance`/`.strength` formulas |
| 15 | Hover lists firing projects with deep-links to /brief | ✅ | hover card maps `slugs` to `/brief?slug=` links |
| 16 | 60-second refresh | ✅ | `useFreshness(..., 60_000)` |
| 17 | Single read of existing table, no new pipeline | ✅ | route reads `attention_event` only; 4-file diff |
| 18 | Nav entry under Observability | ✅ | nav-groups.ts `/signal-graph` entry |

All 18 claims verified. No ❌, no ⚠️.

## Forward-looking scan
No matches for will/plan/coming soon/roadmap/next step/soon used as future promises. "we built / we wanted / it now starts" all historical. PASS.

## Rubric

| Dimension | Score | Notes |
|---|---|---|
| Evidence quality | 5 | All claims trace to code; no SHAs/PR# in prose |
| Technical depth | 5 | Bucketing, edge-pair logic, force formulas, degrade path |
| Clarity for audience | 5 | Strong hook, ASCII diagram, readable code snippets |
| BistecGlobal voice | 5 | Practitioner, evidence-grounded |
| Title specificity | 4 | Specific + human; slightly question-style |

**Total: 24/25 — PASS.**

## Fixes applied
None required. Frontmatter status → audited.
