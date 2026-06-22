# Audit: When Your AI Agent Goes Wrong, You Have a JSONL File. We Built a Debugger.

**Article:** `2026-06-22-session-replay-ai-agents.md`
**Audited:** 2026-06-22
**Verdict:** PASS

---

## Claim Inventory

| # | Claim | Evidence (internal) | Status |
|---|-------|---------------------|--------|
| 1 | "Claude Code writes every turn, every tool call, every assistant reply to a JSONL file" | Parser in `route.ts` reads from `~/.claude/projects/<encoded>/` — JSONL confirmed as Claude Code's session format | ✅ |
| 2 | "stream of event records: `user` type, `assistant` type, `tool` type — interleaved, with timestamps that may be zero or missing" | `route.ts`: `rec._ts = r.timestamp ? new Date(r.timestamp).getTime() : 0`; checks `rec._ts > 0` before using | ✅ |
| 3 | "`user` record with a real timestamp starts a new turn; subsequent records accumulate until the next real user record" | `parseReplay()`: `if (rec.type === 'user' && !rec.isMeta && rec._ts > 0)` → inner while breaks on `next.type === 'user' && !next.isMeta` | ✅ |
| 4 | `ReplayTurn` interface fields (turnIndex, userText, assistantText, toolCalls, diffFromPrev, startEpoch, durationMs) | Exact interface declared in `route.ts` lines 17–25 | ✅ |
| 5 | "assistantText truncated at 2000 chars" | `MAX_TEXT_CHARS = 2000` in `route.ts` line 37 | ✅ |
| 6 | "`(no text change from previous turn)` when diff is empty" | `lineDiff()` returns `'(no text change)'`; `DiffStrip` renders `"(no text change from previous turn)"` on match — `page.tsx` lines 73-76 | ✅ |
| 7 | "tool_use block in assistant message, tool_result in following user message, correlated by tool_use_id" | `toolResultTs`, `toolResultOutput`, `toolResultError` maps keyed by `block.tool_use_id` — `route.ts` lines 134-151 | ✅ |
| 8 | `ReplayToolCall` interface (name, input, output, status, durationMs) | Exact interface in `route.ts` lines 8-15 | ✅ |
| 9 | "input truncated at 300 chars, output at 400" | `MAX_INPUT_CHARS = 300`, `MAX_OUTPUT_CHARS = 400` — `route.ts` lines 34-35 | ✅ |
| 10 | "Color-coded: Bash=orange, reads=blue, writes/edits=cyan, agent spawns=purple, MCP=teal, errors=red" | `toolColor()` in `page.tsx` lines 10-18; `isErr` styling uses `#EF4444` red — lines 34-68 | ✅ |
| 11 | "Previous/Next buttons, timeline scrubber, arrow keys, Space to toggle auto-play" | `prev()`/`next()` fns, `scrubRef`, keyboard handler with ArrowRight/Left, Space — `page.tsx` lines 165-186 | ✅ |
| 12 | "Auto-play advances one turn every three seconds" | `setInterval(..., 3000)` — `page.tsx` line 160 | ✅ |
| 13 | "Escape goes back" | `router.back()` on `e.key === 'Escape'` — `page.tsx` lines 174-175 | ✅ |
| 14 | "⏮ Replay button on per-project detail page and Turn Flame Graph" | Commit stat: `app/flamegraph/page.tsx` and `app/projects/[slug]/page.tsx` both modified in PR #124 | ✅ |
| 15 | "encoded path is real filesystem path with all non-alphanumeric characters replaced by hyphens" | `encodeProjectCwd()`: `realPath.replace(/[^a-zA-Z0-9]/g, '-')` — `route.ts` line 39 | ✅ |
| 16 | "find newest .jsonl by mtime" | `fs.statSync(...).mtimeMs` comparison loop — `route.ts` lines 53-60 | ✅ |
| 17 | `findLatestJsonl` code snippet in article | Exact match to `route.ts` lines 42-62 (abbreviated with `// ...`) | ✅ |
| 18 | Lessons (re-reads same file, diff churn, error clustering) | Observational claims framed as "we found" / "we saw" — not presented as metrics | ⚠️ Weak — acceptable framing |
| 19 | "diff collapses a twenty-turn session into five or six moments" | Experiential claim, framed as observation not measurement | ⚠️ Weak — acceptable |

---

## Forward-Looking Scan

Scanned for: "will", "plan to", "coming soon", "in the future", "next step", "roadmap", "soon".

- **"Every team running AI agents long enough will accumulate JSONL. Most will grep it."** — General observation about universal developer behavior, not a product feature announcement. Not a forward-looking product claim. **No flag.**
- **"They should be able to hit Escape"** — Past-tense design rationale, not a future promise. **No flag.**

**0 forward-looking product statements found.**

---

## Rubric Scorecard

| Dimension | Score | Notes |
|---|---|---|
| Evidence quality (claims verifiable, no raw SHAs in prose) | 5/5 | All 17 technical claims verified against source. Zero SHAs or PR numbers in article prose. |
| Technical depth | 5/5 | Parser internals, data structures, path encoding, UI keyboard/autoplay all covered |
| Clarity for target audience | 5/5 | Strong narrative arc, concrete hook, good "flight recorder" analogy, takeaways actionable |
| BistecGlobal voice | 4/5 | Professional and practitioner-focused; slightly more conversational than prior articles — deliberately so, per style research |
| Title specificity | 5/5 | Human, specific, attention-grabbing — matches trending style research findings |

**Total: 24/25 — PASS**

---

## Fixes Applied

None required. Article passes all checks with no modifications needed.
