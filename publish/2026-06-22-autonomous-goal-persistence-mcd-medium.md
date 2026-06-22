# Autonomous Goal Persistence: Keeping AI Agents On Task Across Context Resets

*How a single 500-character file solved the most persistent failure mode in our nineteen-agent autonomous software fleet.*

When you run nineteen AI agents simultaneously — each autonomously developing a different software project — one failure mode dominates over all others: **drift**. The agent starts sharp, makes progress, hits a context reset (context window exhaustion, watchdog kill, budget rollover, circuit breaker trip), and when the new session opens, it forgets what it was doing. It still has its system prompt. It still has its memory files. But it has no lightweight reminder of *right now* — the immediate goal that justified the session in the first place.

This is the problem we solved in MCD (the multi-channel Discord bot powering Bistec's AI agent fleet) with a feature called Autonomous Goal Persistence. The implementation is in commit `feb0ed6` (PR #83), with a dashboard editor added in `d8fd032` (PR #89) and a full kanban board in `3f9282e` (PR #93).

## The Problem: Session Boundaries Break Momentum

MCD runs one Claude subprocess per Discord channel, each managing a separate software project. These sessions are long-running — hours to days — but finite. The context window fills, the watchdog kills a stalled agent, or the circuit breaker opens after repeated crashes. Each time, MCD restarts the session fresh.

Claude's persistent context comes from two sources: the system prompt (`CLAUDE.md`) and per-project memory files built up over time by the distillation system. These provide *background* — project conventions, past decisions, recurring patterns. What they don't provide is *foreground*: "I was halfway through implementing the token budget queue drain logic and needed to finish the rollover test."

Without foreground context, agents complete a restart and immediately start looking for what to work on. They re-read the backlog, sometimes pick the same item they just completed (before it was committed), sometimes pick a different one. The result is wasted turns and inconsistent throughput across the fleet.

## The Solution: GOAL.md — One File, One Sentence

The fix is deliberately small. Each project gets an optional `GOAL.md` file — a single-file, plain-text, 500-character-max statement of what the agent is currently working toward. The operator sets it via Discord:

```
!project set bistec-articles --goal "Write and publish the autonomous-goal-persistence article, including audit and publish steps."
```

The file is stored at `projects/<slug>/GOAL.md` (path returned by `projectGoalFile()` in `src/paths.ts`, commit `feb0ed6`). That's it. No schema, no database, no config change required.

## Injection: One-Shot, First Message Only

The key implementation detail is *when* the goal is injected. `ClaudeProjectProcess` (in `src/claude-process.ts`) loads `GOAL.md` once at session construction:

```typescript
const goalPath = projectGoalFile(this.slug)
try {
  if (existsSync(goalPath)) {
    const raw = readFileSync(goalPath, 'utf8').trim()
    if (raw) this.goalText = raw.slice(0, 500)
  }
} catch {
  // Non-fatal — proceed without goal injection.
}
```

The goal text is then prepended to the very first message delivered to Claude that session, via `formatPrompt()`:

```typescript
if (!this.firstMessageSent && this.goalText) {
  this.firstMessageSent = true
  return `<goal>${this.goalText}</goal>\n${channelMsg}`
}
this.firstMessageSent = true
return channelMsg
```

One-shot only. Every subsequent message in the session goes through the normal `<channel>` envelope with no goal prefix. The `<goal>` XML tag gives Claude a structured signal it can recognize — not a user command, not a system prompt fragment, but a distinct context marker.

This design avoids two failure modes: injecting on every message (noisy, wastes tokens per turn) and injecting into the system prompt (would require a session restart to update, and the system prompt already carries heavy project context).

## Operator Interface: Read, Write, Clear

The `!project set --goal` command in `src/master-commands.ts` (commit `feb0ed6`) writes the file directly:

```typescript
const truncated = goalRaw.slice(0, 500)
writeFileSync(goalPath, `${truncated.trim()}\n`, { mode: 0o600 })
```

Clearing is equally simple — pass an empty string and the file is deleted. `!project show` displays the first 200 characters of the current goal alongside heartbeat config, memory distillation status, and git remote info.

## Fleet Visibility

Goal state surfaces in two places in the Mission Control dashboard.

The `InstanceGrid` component (`apps/mission-control/components/InstanceGrid.tsx`, commit `feb0ed6`) renders a purple chip below each project's slug when a goal is set — the first 40 characters of the goal text, full text on hover:

```tsx
<span
  className="text-[0.55rem] font-mono px-1.5 py-0.5 rounded mt-0.5 truncate max-w-[140px]"
  style={{ color: '#a78bfa', background: '#a78bfa15', border: '1px solid #a78bfa30' }}
  title={`Goal: ${fp.goalText}`}
>
  🎯 {fp.goalText.slice(0, 40)}{fp.goalText.length > 40 ? '…' : ''}
</span>
```

The Fleet API (`apps/mission-control/app/api/fleet/route.ts`) reads every project's `GOAL.md` on each poll via `readGoal()`, returning `goalText` and `goalStatus` as part of the `FleetProject` shape.

## Inline Editing and Status Tracking (P44)

The initial implementation (`feb0ed6`) required a Discord command to update goals. Commit `d8fd032` (PR #89) added an inline goal editor directly in the dashboard via a `GoalChip` React component — click the chip to open a textarea and status picker, save sends a `PUT /api/projects/[slug]/goal` request, blank save sends `DELETE`.

The API route (`apps/mission-control/app/api/projects/[slug]/goal/route.ts`) also introduced YAML frontmatter to `GOAL.md`, adding a `status` field that tracks one of three states: `active`, `paused`, or `completed`. Status affects the chip color (purple for active, gray for paused, green for completed).

## Cross-Project Goal Board (P49)

Commit `3f9282e` (PR #93) added a dedicated `/goals` page to Mission Control — a kanban board with Active, Paused, and Completed columns. It calls a new `/api/goals` endpoint that scans all project `GOAL.md` files and returns their parsed content and status. Each card links to the project's audit timeline and exposes the same status-cycle chip as `InstanceGrid`.

At the fleet level, this gives operators a single view of what every agent is currently working toward — without opening Discord or reading individual project state.

## Design Decisions Worth Noting

**500-character cap.** The goal is meant to be a sentence or two — not a planning document. Forcing brevity keeps it scannable at a glance in the dashboard and prevents operators from accidentally turning it into a second system prompt.

**File-based, not config-based.** `GOAL.md` lives in the project directory alongside `MEMORY.md` and `.session-id`. It can be read with a text editor, grepped, version-controlled if the project directory is in git. No schema migrations, no database queries.

**XML-tagged injection, not raw text.** Prepending `<goal>text</goal>` rather than raw text gives Claude a semantic hook. The model treats `<goal>` as a structured signal distinct from the user's actual message, which reduces the chance of the agent treating the goal as an operator command to respond to.

**Load-once, inject-once.** Reading at session construction (not at message delivery) means the file read is synchronous and happens before any concurrency pressure. The one-shot injection means the token cost is paid exactly once per session.

## Outcome

Across Bistec's active fleet of nineteen projects, goal persistence reduces the most common form of agent drift — the session restart that produces random backlog re-selection. Agents that restart mid-task now resume with an explicit statement of what they were working toward, reducing turns spent re-orienting. The `/goals` board gives the operator a fleet-wide priority view that didn't exist before.

The lesson is that the simplest mechanism capable of surviving a process restart is often good enough: write a file, read it on the next start, inject it once.

---

*Originally published at bistecglobal.com*

**BistecGlobal** builds AI-native enterprise software. Follow us for engineering insights.
