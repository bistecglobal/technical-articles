# Heartbeat Watchdog Pattern for Autonomous AI Agents

*How we built a transcript-scanning, stall-detecting, self-recovering watchdog that keeps 19 Claude Code agents running around the clock without human babysitting.*

When you run AI agents autonomously — no human watching, no manual restart — they stall. Claude Code gets blocked waiting for user input that never comes, a tool call never resolves, or a long turn is interrupted mid-flight. Without a watchdog, that channel is dead until someone notices.

We built a watchdog system in [claude-mcd](https://github.com/chan4lk/claude-multi-channel-discord), our multi-channel Discord bot that manages a fleet of Claude Code subprocesses. It detects stalls by reading conversation transcripts directly, and recovers them by injecting synthetic prompts — without human intervention. Here is exactly how it works.

## The Three Stall Patterns

Claude Code runs as an interactive CLI process inside a tmux session. Messages from Discord (or scheduled tasks) are injected via `tmux send-keys`. When a turn completes, Claude replies through an MCP tool, and the cycle repeats.

Turns fail in three distinct ways:

1. **Tool-incomplete** — Claude issues a `tool_use` call that never gets a `tool_result`. This happens when a subprocess is killed mid-turn or an MCP timeout fires.
2. **Question-unanswered** — Claude asks a clarifying question and waits. In a fully autonomous flow with no human watching, it can wait indefinitely.
3. **Process-crashed** — the tmux session exits and the process disappears.

Each failure mode is detectable from the transcript. Each has a different recovery strategy.

## Transcript-Based Detection

The core insight: Claude Code writes a detailed `.jsonl` transcript for every session under `~/.claude/projects/<encoded-cwd>/`. This transcript is ground truth — every message, every tool call, every result, in order.

`src/heartbeat.ts` (`classifyChannel()`, commit `9778113`) reads the last 200 entries of the latest transcript and looks for two structural anomalies:

```typescript
// Tool-incomplete: tool_use with no matching tool_result
for (const [id, name] of toolUseIds.entries()) {
  if (!toolResultIds.has(id)) {
    stalledReason = 'tool-incomplete'
    snippet = `${name} ${id}`.slice(0, 40)
    break
  }
}

// Question-unanswered: assistant ends with '?' and no user reply follows
if (lastAssistantHasQuestion && !hasSubsequentUser) {
  stalledReason = 'question-unanswered'
}
```

A guard fires before either check: if the transcript file was modified within the last 30 seconds, the agent is still active and the stall analysis is skipped entirely. This prevents false positives on long-running turns that are making progress but haven't produced a reply yet.

The `staleAfterMinutes` threshold (default 60, configurable per-project via `heartbeat.staleAfterMinutes`) controls how old an anomaly must be before it is treated as a real stall. A `tool-incomplete` from 2 minutes ago is likely a transient race; one from 90 minutes ago is a dead channel.

## Adaptive Stuck Threshold

The transcript scanner operates at the session level. The `ProjectPool` (`src/project-pool.ts`) operates at the process level: it tracks when each process last produced a reply and kills and respawns hung processes.

The base threshold is 5 minutes:

```typescript
static readonly STUCK_THRESHOLD_MS = 5 * 60_000
```

A flat 5-minute threshold produced false positives on channels running long parallel-subagent turns — workflows that fan out 10 research agents concurrently and legitimately take 15–20 minutes. The first version killed channels that were genuinely working.

The fix, landed in commit `1f3f976`, was an adaptive threshold:

```typescript
const ADAPTIVE_MULTIPLIER = 1.5
const MAX_ADAPTIVE_THRESHOLD_MS = 30 * 60_000  // hard cap
```

The pool tracks the last 5 turn durations (`MAX_TURN_HISTORY = 5`). The effective threshold becomes `min(30min, max(base, ceil(longest_recent_turn × 1.5)))`. A channel that routinely completes 12-minute turns gets an 18-minute watchdog window; one that always finishes in under 90 seconds stays at 5 minutes.

## Synthetic Message Injection

Recovery runs through the same code path as a real Discord message. The `Scheduler` (`src/scheduler.ts`) wraps the prompt in an `InboundEnvelope`:

```typescript
// envelopeFor() in scheduler.ts
return {
  messageId: `sched-${s.id}-${now.getTime()}`,
  userId: '__mcd_scheduler__',
  username: 'scheduler',
  content: s.prompt + footer,
  ts,
}
```

This envelope goes to `pool.deliver(chatId, envelope)` — identical to the path for a human message. The pool handles spawn-on-demand, dedup (a 60-second TTL prevents a resumed Discord gateway from double-delivering the same message ID), and reply routing. No special cases for synthetic vs. real messages.

The scheduler ticks every 60 seconds. The `hasFiredToday` / `hasFiredWithin` guards in `schedules-config.ts` prevent double-fires after a restart: a missed 09:00 slot that fires at 09:30 does not re-fire when the bot comes back up at 09:35.

## Per-Project Configuration

Not every channel needs the same sensitivity. A human-interactive channel should have a tight threshold; a fully autonomous batch-processing agent should tolerate longer silences.

Per-project heartbeat config (PR #46, `fix/heartbeat-watchdog` series):

```json
{
  "projects": {
    "<channel-id>": {
      "slug": "keyflow",
      "heartbeat": {
        "staleAfterMinutes": 120
      }
    }
  }
}
```

`staleAfterMinutes` gates the transcript scanner. The separate per-project `stuckThresholdMinutes` field (PR #43, commit `487420e`) feeds into `ProjectPool` to set the adaptive watchdog base for that channel.

## What the Fleet Sees

The Mission Control dashboard (PR #55, commit `bf68204`) shows a real-time stall alert panel with the exact reason (`tool-incomplete` or `question-unanswered`) and the relevant snippet — which tool call hung, or what question went unanswered. The Fleet Health Bar (commit `2e5b3d6`) colour-codes every channel as idle, active, stalled, or autonomous.

This surfaces what was previously invisible: which channels are working and which are silently dead.

## Lessons

**Read the transcript, not the process.** We tried health-check HTTP endpoints and `is-alive` process polls first. Both had high false-negative rates — a process that is running but stuck looks healthy. The `.jsonl` transcript tells you *what* is wrong, not just *that* something is wrong.

**Do not kill aggressively.** The adaptive threshold exists because the first flat 5-minute kill window fired on channels running legitimate long-duration parallel workflows. Measuring observed turn history and scaling the window accordingly eliminated that class of false positives.

**Same code path is not optional.** Routing synthetic scheduler messages through the normal `deliver()` path means deduplication, pool eviction logic, and reply routing work identically for both. Every bug fix to the real message path applies automatically to the watchdog path. Side-channel shortcuts accumulate special-case debt.

---

The heartbeat watchdog is now the backbone of running 19 autonomous agent channels simultaneously with no manual intervention. The pattern is straightforward — read the transcript, measure the anomaly age, inject a recovery message — but each of the three details above (transcript scanning, adaptive threshold, same-path injection) took a real production failure to discover.

---

*Originally published at bistecglobal.com*

**BistecGlobal** builds AI-native enterprise software. Follow us for engineering insights.
