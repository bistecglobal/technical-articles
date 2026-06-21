# Audit: 2026-06-21-cross-session-memory-distillation

**Verdict: PASS (minor fixes applied)**
**Score: 22/25**

---

## Claim Inventory

| # | Claim | Evidence | Status |
|---|-------|----------|--------|
| 1 | MCD runs one Claude subprocess per Discord channel | `server.ts` line 278: "One entry per guild channel"; line 1185: "per-channel claudes" | ✅ |
| 2 | Sessions end on user command (`!kill`) or when agent becomes idle | `src/claude-process.ts` — kill/idle logic present | ✅ |
| 3 | MEMORY.md convention exists per project, picked up at next session | Claude Code auto-memory convention; `src/claude-process.ts` line 404 "preserve hooks, auto-memory" | ⚠️ weak — Claude Code convention, not explicitly wired in MCD code; reworded to "Claude Code's built-in project memory" |
| 4 | Distillation spawns `claude -p` with `--permission-mode auto` | `src/distillation.ts` line 33 | ✅ |
| 5 | 90-second hard timeout with SIGKILL | `src/distillation.ts` lines 13, 42 | ✅ |
| 6 | Retries exactly once on non-zero exit | `src/distillation.ts` lines 95–98 | ✅ |
| 7 | Caller fires without awaiting (`src/claude-process.ts` line 1106) | Verified | ✅ |
| 8 | `onComplete` fires `distillation_complete` audit event | `server.ts` line 1246 | ✅ |
| 9 | `distillOnStop` flag in `src/channels-config.ts` line 109 | Verified | ✅ |
| 10 | Activated via `!project set <slug> --distill-on-stop` | `src/master-commands.ts` line 527 | ✅ |
| 11 | Displayed in `!project show` output | `src/master-commands.ts` line 276 | ✅ |
| 12 | Dashboard 💭 chip in fleet grid (commit `8492b53` P42) | `apps/mission-control/components/InstanceGrid.tsx` lines 155–165 | ✅ |
| 13 | MemoryModal with "Distill now" button | `InstanceGrid.tsx` MemoryModal component | ✅ |
| 14 | `POST /api/memory/[slug]/distill` route | `apps/mission-control/app/api/memory/[slug]/distill/route.ts` | ✅ |
| 15 | Response returns content, size, timestamp | `route.ts` lines 72–77 | ✅ |
| 16 | `GET /api/memory/[slug]` route added in P42 | `apps/mission-control/app/api/memory/[slug]/route.ts` | ✅ |
| 17 | `src/distillation.ts` is 109 lines | git show `feb0ed6` --stat confirms 109 lines | ✅ |
| 18 | No external deps beyond `child_process` and `fs` | `src/distillation.ts` imports: only `node:child_process`, `node:fs` | ✅ |
| 19 | PR #83 introduced P38; PR #88 added P42 | Commit messages and PR references verified | ✅ |
| 20 | "Re-investigation of known facts dropped noticeably" | No measurement exists | ❌ removed — qualitative outcome claim with no evidence |
| 21 | "agents now start sessions with *accurate* context" | Mechanism verified; "accurate" quality is unmeasured | ⚠️ reworded to remove "accurate" qualifier |

---

## Forward-Looking Scan

Searched for: will, plan to, coming soon, in the future, next step, roadmap, soon.

**Result: NONE FOUND.** Article is fully past-tense / present-descriptive.

---

## Rubric Scorecard

| Dimension | Score | Notes |
|---|---|---|
| Evidence quality (claims cited) | 5/5 | Nearly every claim has commit SHA or file:line reference |
| Technical depth | 5/5 | Covers spawn mechanics, timeout, retry, audit trail, dashboard |
| Clarity for target audience | 4/5 | Well-structured; minor: "auto-memory system" was vague |
| BistecGlobal voice | 4/5 | Professional, practitioner-focused throughout |
| Title specificity | 4/5 | Good; could be more specific about "MCD" but readable |

**Total: 22/25 — PASS**

---

## Fixes Applied

1. **Claim 3** — "auto-memory system" reworded to "Claude Code's built-in project memory convention" for precision.
2. **Claim 20** — "Re-investigation of known facts dropped noticeably" removed; replaced with description of mechanism (MEMORY.md written and available at next session start).
3. **Claim 21** — "accurate context" reworded to "context about the project state"; removed unverified quality qualifier.
4. Frontmatter updated to `status: audited`.
