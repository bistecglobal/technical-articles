---
article: 2026-06-21-mcd-autonomous-weekly-fleet-report.md
auditor: bistec-articles audit skill
date: 2026-06-21
verdict: PASS (fix applied)
score: 23/25
---

# Audit Report — Autonomous Weekly Fleet Report

## Verdict: PASS — 1 fix applied

Score: **23/25**. One factual error corrected (commit SHA cited as PR number).

---

## Claim Inventory

| # | Claim | Evidence | Verdict |
|---|---|---|---|
| 1 | MCD = one Claude subprocess per Discord channel, managed by `ProjectPool` | Prior published articles, MCD codebase | ✅ |
| 2 | Claude Code writes JSONL to `~/.claude/projects/<encoded-path>/<session-id>.jsonl` | `route.ts:185-186`: `path.join(os.homedir(), '.claude', 'projects', encoded)` + `.jsonl` filter | ✅ |
| 3 | GET `/api/reports/weekly` walks `MCD_CHANNELS_DIR/projects/` | `route.ts:169-175` | ✅ |
| 4 | File-mtime pre-filter excludes files not touched this week | `route.ts:192-195` | ✅ |
| 5 | `analyzeJsonl()` extracts turns, tokens, tool calls, stalls, latency, daily distribution | `route.ts:64-127` | ✅ |
| 6 | Stall = `stop_reason` is `max_tokens` or `end_turn` with no tool calls, turns > 1 | `route.ts:115-119` | ✅ |
| 7 | PR count via `git log --oneline --after=<weekStart> --merges` | `route.ts:130-138` (`countPrs`) | ✅ |
| 8 | Memories counted from `.claude/memory/` or `memory/` directories, mtime filter | `route.ts:140-157` (`countRecentMemories`) | ✅ |
| 9 | MODEL_PRICING: haiku [0.8,4], sonnet [3,15], opus [15,75] | `route.ts:51-55` | ✅ |
| 10 | Cost = `(inputTokens/1M)*inputRate + (outputTokens/1M)*outputRate` | `route.ts:203-205` | ✅ |
| 11 | `impactScore = turns*2 + prCount*20 + toolCalls + memoriesWritten*5` | `route.ts:210` | ✅ |
| 12 | `topByEfficiency` = lowest cost per turn | `route.ts:240-242` | ✅ |
| 13 | `topByActivity` = highest impact score (projects[0] after sort) | `route.ts:228, 239` | ✅ |
| 14 | POST generate saves to `MCD_CHANNELS_DIR/reports/<weekLabel>.json` | `generate/route.ts:29-30` | ✅ |
| 15 | Week label format e.g. `2026-06-W25` | `generate/route.ts:22-24` using `getISOWeek()` | ✅ |
| 16 | `NeonStat` component reused, no new dependencies | `reports/page.tsx:22-31` | ✅ |
| 17 | Sparkline: green uptrend, red downtrend | `reports/page.tsx:43-44` | ✅ |
| 18 | "shipped in MCD PR #3e3f957" | WRONG — `3e3f957` is commit SHA, actual PR is #96 (`git log` shows `Merge pull request #96`) | ❌ Fixed |

---

## Forward-Looking Scan

No matches for "will", "plan to", "coming soon", "in the future", "next step", "roadmap", "soon". **Clean.**

---

## Rubric Scorecard

| Dimension | Score | Notes |
|---|---|---|
| Evidence quality (claims cited) | 5/5 | Every metric/formula directly cited to file:line; 1 error fixed |
| Technical depth | 5/5 | Full extraction pipeline, pricing, impact formula, persistence all covered |
| Clarity for target audience | 4/5 | Well-structured; "what it tells you" section anchors the value clearly |
| BistecGlobal voice | 4/5 | Professional, grounded in concrete implementation |
| Title specificity | 5/5 | Precise, accurate, not generic |

**Total: 23/25 — PASS**

---

## Fix Applied

**Claim 18**: "shipped in MCD PR #3e3f957" → "shipped in MCD PR #96 (commit `3e3f957`)"

`3e3f957` is the commit SHA; the associated merge commit is `0cfc6e1` for `Merge pull request #96 from chan4lk/feat/p53-p55-impact-traceability-weekly-report`.
