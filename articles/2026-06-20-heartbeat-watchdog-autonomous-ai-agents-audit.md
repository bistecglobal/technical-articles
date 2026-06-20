# Audit: Heartbeat Watchdog Pattern for Autonomous AI Agents

**Date:** 2026-06-20  
**Auditor:** bistec-articles agent  
**Verdict:** PASS (with minor fixes applied)  
**Score:** 22/25

---

## Claim Inventory

| # | Claim | Evidence | Status |
|---|---|---|---|
| 1 | claude-mcd is a multi-channel Discord bot managing Claude Code subprocesses | `github.com/chan4lk/claude-multi-channel-discord`, `src/project-pool.ts` | ✅ |
| 2 | Claude Code runs inside a tmux session per project | `claude-process.ts` docstring: "dedicated `claude` CLI inside a detached tmux session" | ✅ |
| 3 | Messages injected via `tmux send-keys` | `claude-process.ts` docstring: "deliver(): tmux send-keys -l" | ✅ |
| 4 | Claude replies through MCP tool | `claude-process.ts`: "--mcp-config pointing at the master HTTP MCP server" | ✅ |
| 5 | Tool-incomplete stall: `tool_use` without `tool_result` | `heartbeat.ts:122-149` | ✅ |
| 6 | Question-unanswered stall: assistant `?` with no user follow-up | `heartbeat.ts:152-193` | ✅ |
| 7 | Process-crashed stall listed as OOM/network/bad restart | `project-pool.ts` PoolEvent type has `crashed` kind — specific causes (OOM etc.) are examples, not enumerated in code | ⚠️ |
| 8 | Transcripts at `~/.claude/projects/<encoded-cwd>/` | `heartbeat.ts:52-53`: `join(homedir(), '.claude', 'projects', encodeProjectCwd(cwd))` | ✅ |
| 9 | `classifyChannel()` reads last 200 entries, commit `9778113` | `heartbeat.ts:94`: `.slice(-200)`, commit `9778113` is `fix(heartbeat-watchdog): T7 — Create src/heartbeat.ts` | ✅ |
| 10 | Code snippet: tool-incomplete loop | Exact match `heartbeat.ts:143-148` | ✅ |
| 11 | Code snippet: question-unanswered check | Exact match `heartbeat.ts:180-193` | ✅ |
| 12 | 30-second activity guard | `heartbeat.ts:77-79`: `Date.now() - latestMtime < 30_000` | ✅ |
| 13 | `staleAfterMinutes` default 60, configurable per-project | `heartbeat.ts:82`: `?? 60` | ✅ |
| 14 | Base stuck threshold 5 min (`STUCK_THRESHOLD_MS`) | `project-pool.ts:54` | ✅ |
| 15 | Flat 5-min threshold caused false positives on long parallel turns | commit `1f3f976` message: "Prevents false-positive kills on channels that routinely run long parallel-subagent turns" | ✅ |
| 16 | Adaptive threshold constants `ADAPTIVE_MULTIPLIER=1.5`, `MAX_ADAPTIVE_THRESHOLD_MS=30*60_000` | `claude-process.ts:44-46` | ✅ |
| 17 | Pool tracks last 5 turn durations (`MAX_TURN_HISTORY=5`) | `claude-process.ts:44` | ✅ |
| 18 | Formula: `min(30min, max(base, ceil(maxTurn × 1.5)))` | `claude-process.ts:1015` | ✅ |
| 19 | Scheduler `envelopeFor()` code snippet | Exact match `scheduler.ts:105-120` | ✅ |
| 20 | Synthetic envelope delivered via `pool.deliver()` — same path as human message | `scheduler.ts` tick → `deps.deliver()`, `project-pool.ts` deliver() handles both | ✅ |
| 21 | 60-second dedup TTL (`MSG_DEDUP_TTL_MS`) | `project-pool.ts:45`: `private static readonly MSG_DEDUP_TTL_MS = 60_000` | ✅ |
| 22 | Scheduler ticks every 60 seconds | `scheduler.ts:44`: `start(intervalMs = 60_000)` | ✅ |
| 23 | `hasFiredToday`/`hasFiredWithin` prevent double-fires after restart | `scheduler.ts` imports from `schedules-config.ts`, comment at line 145 | ✅ |
| 24 | `staleAfterMinutes` per-project config came from PR #43 | **WRONG** — `staleAfterMinutes` is in `fix/heartbeat-watchdog` series, merged as PR #46 (`0f2de9a`). PR #43 (`487420e`) added `stuckThresholdMinutes` CLI flag. | ❌ |
| 25 | `stuckThresholdMinutes` from PR #43 feeds into ProjectPool | commit `487420e`: `feat(set): add --stuck-threshold-minutes flag to !project set (#43)` | ✅ |
| 26 | Mission Control stall alert panel: PR #55, commit `bf68204` | git log: `feat(mc): Stall Alert Panel (P3) + memory-aware heartbeat template (P9)` | ✅ |
| 27 | Fleet Health Bar commit `2e5b3d6` | git log: `feat(mc): Fleet Health Bar — idle/active/stalled/autonomous badges (#52)` | ✅ |
| 28 | "15+ autonomous agent channels simultaneously" | `channels.json` lists 19 active channels | ⚠️ accurate but can be made more precise |

---

## Forward-Looking Scan

Searched for: will, plan to, coming soon, in the future, next step, roadmap, soon.

**Result: 0 matches.** No forward-looking statements found.

---

## Rubric Scorecard

| Dimension | Score | Notes |
|---|---|---|
| Evidence quality (claims cited) | 4/5 | 1 wrong PR attribution fixed; OOM/crash causes slightly hand-wavy |
| Technical depth | 5/5 | Covers transcript parsing, adaptive formula, synthetic injection, dedup — all with code |
| Clarity for target audience | 5/5 | Practitioner-focused, concrete examples, lessons section |
| BistecGlobal voice | 4/5 | Professional and insightful; closing slightly generic |
| Title specificity | 4/5 | Specific and accurate |

**Total: 22/25 — PASS**

---

## Fixes Applied

1. **PR attribution for `staleAfterMinutes`**: Changed "PR #43, commit `487420e`" to "PR #46" for the heartbeat config section. Kept PR #43 for `stuckThresholdMinutes` (correct).
2. **Channel count precision**: Updated "15+" to "19" (verified from `channels.json`).
3. Updated frontmatter `status` to `audited`.
