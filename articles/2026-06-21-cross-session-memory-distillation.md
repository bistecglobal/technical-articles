---
title: "Cross-Session Memory Distillation: How We Taught Our AI Agents to Remember What Matters"
project: claude-mcd
tags: [AI, Claude, Multi-Agent, Memory, TypeScript, DevOps]
status: draft
date: 2026-06-21
---

# Cross-Session Memory Distillation: How We Taught Our AI Agents to Remember What Matters

Long-running AI agents have a memory problem. Claude's context window is finite, and when a session ends — whether from a restart, a kill command, or a deployment cycle — everything the agent learned during that session evaporates. The next session starts cold.

For MCD (Multi-Channel Discord), our autonomous AI agent platform, this was a real operational pain. Agents working on the same project across many sessions would re-discover the same facts, re-investigate the same files, and make the same tentative decisions. We had a `MEMORY.md` file convention per project, but it relied on agents remembering to write to it during the session — an unreliable bet.

The solution was to stop asking agents to remember on the fly, and instead distill memory automatically after every session ends.

## The Problem: Context Windows Don't Persist

MCD runs one Claude subprocess per Discord channel, with each project directory as the working directory. Sessions start via user messages or scheduled heartbeats and end either on user command (`!kill`) or when the agent becomes idle. Each session is isolated — no state flows between them except what's written to disk.

We had a `MEMORY.md` convention: agents could write key facts there, and it would be picked up by the auto-memory system in the next session. But this required the agent to proactively manage its own memory during a session while also doing its actual work. In practice, it was inconsistent — agents would write thorough notes on one session and skip it entirely the next.

What we needed was a guaranteed, post-session sweep that could reliably extract and persist what mattered.

## The Solution: A Background Distillation Process

The core insight was that distillation should be a platform responsibility, not an agent responsibility. After every session ends, MCD spawns a fresh, short-lived `claude -p` process in the project directory. This process reads the project state and recent session history and rewrites `MEMORY.md` with a merged summary.

The implementation lives in `src/distillation.ts` (commit `feb0ed6`):

```typescript
const DISTILL_PROMPT =
  'Summarise the key facts, decisions, and open questions from this session into MEMORY.md ' +
  'in 500 words or less. Merge with any existing content — do not replace sections that are ' +
  'still relevant. Write only to MEMORY.md; output nothing else.'

export async function runDistillation(opts: {
  slug: string
  cwd: string
  claudeBin?: string
  log: (msg: string) => void
  onComplete?: (result: DistillationResult) => void
}): Promise<DistillationResult>
```

The process is spawned with `claude -p <prompt> --permission-mode auto`, runs in the project directory with full filesystem access, and has a hard 90-second timeout enforced with `SIGKILL`. If it exits non-zero, it retries exactly once. If it times out or both attempts fail, MCD logs the failure and moves on — distillation is best-effort, never blocking.

The caller side in `src/claude-process.ts` (line 1106) fires it without awaiting:

```typescript
// Fire background distillation if configured (P38). Non-blocking.
if (this.opts.distillOnStop && this.projectCwd) {
  runDistillation({ slug, cwd: this.projectCwd, log, onComplete })
}
```

This design choice was deliberate. Session kills need to complete fast — agents are often killed to free resources or restart after a stuck state. Blocking kill on a 90-second distillation job would make the fleet unresponsive. Instead, distillation runs in the background, and the `onComplete` callback fires an audit event (`distillation_complete`) when it's done.

## Design Decisions Worth Examining

**Merge semantics, not replace.** The distillation prompt explicitly instructs Claude to merge new content with existing `MEMORY.md` rather than overwriting it. Early sessions build up foundational facts about the project — architecture decisions, key file paths, known gotchas. Later sessions should add to this, not erase it. The prompt enforces this: "do not replace sections that are still relevant."

**Fixed prompt, no context injection.** We deliberately kept the distillation prompt simple and static. We could have injected the session transcript, recent tool calls, or current file state into the prompt. But that adds latency, cost, and complexity. The distillation process runs in the project directory with `--permission-mode auto`, which means it can read any file it needs on its own. Claude knows what to look for.

**Opt-in per project.** Distillation is not enabled by default. It's activated per project via Discord master command:

```
!project set <slug> --distill-on-stop
```

The `distillOnStop` flag is stored in the channels config (`src/channels-config.ts`, line 109) and displayed in `!project show` output. This lets operators enable it only for projects where persistent memory is useful, avoiding unnecessary Claude API calls on ephemeral or test projects.

**Audit trail integration.** The `onDistillationComplete` callback wires into MCD's existing event bus. When distillation finishes, `server.ts` emits a `distillation_complete` event (line 1246):

```typescript
onDistillationComplete: (result) => {
  mcEmit('distillation_complete', { slug: project.slug, chatId, ...result })
}
```

This means distillation outcomes — success/failure, duration, attempt count — appear in the audit trail alongside session starts, kills, and heartbeats. Operators can see whether distillation is consistently succeeding or regularly timing out.

## Dashboard Visibility

A follow-up change (commit `8492b53`, P42) added first-class dashboard support. The fleet grid now shows a 💭 chip next to project nodes where `MEMORY.md` exists, displaying the file size. Clicking it opens a modal with the full memory content and a "Distill now" button.

The "Distill now" button calls a dedicated Next.js API route:

```
POST /api/memory/[slug]/distill
```

This runs the same distillation logic on demand, without waiting for a session end. The response returns the updated content, size, and timestamp so the modal can refresh inline. This is useful when debugging — you can trigger a distillation immediately after a session to verify what got captured.

The `GET /api/memory/[slug]` route (also added in P42) exposes the raw content and file metadata, letting the dashboard surface staleness — if `lastModified` is several days old relative to recent session activity, memory may be drifting out of date.

## Outcomes

Memory distillation changed how our agents behave across long-running projects. On projects with `distillOnStop` enabled, agents now start sessions with accurate context about the project state — what's been built, what's pending, what was decided and why. Re-investigation of known facts dropped noticeably.

The implementation is also compact. The core `src/distillation.ts` is 109 lines. It has no external dependencies beyond Node's `child_process` and `fs` modules. The retry logic is a simple two-call sequence, not a retry framework. The timeout uses a vanilla `setTimeout` + `SIGKILL`.

## What We Learned

The key lesson was about responsibility allocation. Asking agents to manage their own memory during a session creates a competing priority — the agent has to balance doing useful work against documenting what it's doing. Moving memory management to the platform removes that tension. The agent works; the platform remembers.

The "merge, not replace" constraint in the prompt turned out to be the most important design choice. Without it, late-session distillation would overwrite early-session foundational notes. With it, `MEMORY.md` accumulates across sessions into a genuine project knowledge base.

The opt-in model also proved correct. Not every project benefits from persistent memory. Short-lived or test projects would generate distillation calls with no value. Letting operators choose which projects get it keeps the signal clean.

---

*Claude-mcd is the autonomous AI agent orchestration platform developed by BistecGlobal, connecting Claude agents to Discord channels across multiple projects. PR #83 introduced distillation (P38); PR #88 added the dashboard panel (P42).*
