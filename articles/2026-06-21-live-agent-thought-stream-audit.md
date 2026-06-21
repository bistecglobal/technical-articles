# Audit: 2026-06-21-live-agent-thought-stream

**Verdict: PASS (with one claim fix applied)**
**Score: 23/25**

---

## Claim Inventory

| # | Claim | Evidence | Status |
|---|-------|----------|--------|
| 1 | Fleet state tracked per project: running, idle, stalled | Prior MCD articles; dashboard codebase | ✅ |
| 2 | Dashboard displays health scores, token budgets, stall alerts via SSE | Prior MCD articles (blog#10, blog#11) | ✅ |
| 3 | Thought stream tails Claude Code JSONL transcripts in real time | `checkToolEvents()` in `sse.ts`, commit `f291a54` | ✅ |
| 4 | Color-coded floating particles drift up from project nodes | `TOOL_CATEGORIES` + particle rendering in `ProjectGraph.tsx`, commit `f291a54` | ✅ |
| 5 | Feature shipped in commit `f291a54` as MCD P54 | `git log` confirmed | ✅ |
| 6 | `checkToolEvents` runs inside existing SSE interval | `broadcastFleetUpdate()` calls `try { checkToolEvents(mcdDir) }` | ✅ |
| 7 | Symlinks resolved via `fs.realpathSync` | `getLatestTranscriptFile()`, commit `f291a54` | ✅ |
| 8 | Transcript directory: `~/.claude/projects/<encoded>/` | `claude-process.ts:113` + `sse.ts:encodeProjectCwd` | ✅ |
| 9 | Latest `.jsonl` selected by `mtime` | `statSync().mtimeMs` loop in `getLatestTranscriptFile` | ✅ |
| 10 | `toolLineTracker` map keyed to `slug → { file, lineCount }` | `__mcdToolLineTracker` global in `sse.ts` | ✅ |
| 11 | Cursor reset to `lines.length` on file change prevents history replay | Code: `startLine = lines.length` when `tracker.file !== latestFile` | ✅ |
| 12 | `mcp__mcd__*` tools suppressed | `!block.name.startsWith('mcp__mcd__')` in `checkToolEvents` | ✅ |
| 13 | `FleetContextProvider` previously held fleet state and stall events | `FleetContext.tsx` prior state; confirmed in diff | ✅ |
| 14 | `toolEvents` capped at 200 entries | `MAX_TOOL_EVENTS = 200` in `FleetContext.tsx` | ✅ |
| 15 | Monotonically increasing integer IDs on events | `++toolEventIdCounter` | ✅ |
| 16 | Max 3 particles per node | `MAX_PARTICLES_PER_NODE = 3` in `ProjectGraph.tsx` | ✅ |
| 17 | 3-second lifetime, `setTimeout` + 100ms buffer, 1-second sweep interval | `PARTICLE_LIFETIME_MS = 3000`, `setTimeout(..., PARTICLE_LIFETIME_MS + 100)`, `setInterval` prune | ✅ |
| 18 | Toggle `T` key, persisted in `localStorage` | `THOUGHT_STORAGE_KEY`, keyboard handler in `ProjectGraph.tsx` | ✅ |
| 19 | "MCD runs Claude as a subprocess via `claude --print`" | **WRONG** — MCD uses `tmux new-session -d -s <name> 'claude --mcp-config ...'` (interactive tmux). See `claude-process.ts:16` | ❌ Fixed |
| 20 | Weekly fleet report sibling commit `3e3f957` | Commit `3e3f957` confirmed in git log (blog#14) | ✅ |
| 21 | JSONL is append-only and line-delimited | Claude Code transcript format; confirmed via `claude-process.ts:84-87` | ✅ |
| 22 | Feature live in dashboard, commit `f291a54`, PR #95 | Confirmed | ✅ |

---

## Forward-Looking Scan

No instances of "will", "plan to", "coming soon", "in the future", "next step", "roadmap", or "soon" found. **PASS.**

---

## Rubric Scorecard

| Dimension | Score | Notes |
|---|---|---|
| Evidence quality (claims cited) | 4/5 | One claim wrong and fixed; all others have concrete references |
| Technical depth | 5/5 | All three layers covered with actual code; design rationale explained |
| Clarity for target audience | 4/5 | Clear narrative arc; could define JSONL for less-technical readers |
| BistecGlobal voice | 5/5 | Professional, practitioner-focused, no puffery |
| Title specificity | 5/5 | Names the exact mechanism (particles, fleet graph) |

**Total: 23/25 — PASS**

---

## Fixes Applied

**Claim #19** — replaced `claude --print` with correct tmux invocation:

- Before: `"MCD runs Claude as a subprocess via \`claude --print\`; there is no in-process hook."`
- After: `"MCD launches Claude inside a dedicated tmux session (\`tmux new-session -d -s <name> 'claude --mcp-config ...'\`); there is no in-process hook."`

Source: `claude-process.ts` line 16.
