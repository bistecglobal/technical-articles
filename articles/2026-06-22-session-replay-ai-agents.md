---
title: "When Your AI Agent Goes Wrong, You Have a JSONL File. We Built a Debugger."
project: claude-mcd
tags: [AI, debugging, observability, developer-tooling, claude-code, devops]
status: audited
date: 2026-06-22
---

# When Your AI Agent Goes Wrong, You Have a JSONL File. We Built a Debugger.

Something unexpected happened at turn 34.

You can tell because the agent's next response changed character — it stopped making forward progress and started repeating itself. You know this because you eventually grepped through the transcript. Two hundred lines of JSON to find one wrong tool call.

That was the problem we kept hitting with our AI fleet. When an agent misbehaved, the raw evidence was there — Claude Code writes every turn, every tool call, every assistant reply to a JSONL file. But navigating that file felt like reading a flight recorder without a cockpit view. We built one.

## The Transcript Is Already There

Claude Code writes a `.jsonl` file for every session — one line per event: user messages, assistant replies, tool invocations, tool results. For a typical two-hour autonomous agent session, that's several hundred lines of dense JSON.

The information is complete. Timestamps, tool inputs, tool outputs, error flags — it is all there. The problem is that it is not human-readable at speed. `grep` finds strings. It does not show you *context*. It does not let you step through what the agent was doing turn by turn, see which tools it called and whether they succeeded, or spot where its reasoning started to drift.

So we built a session replay mode into our Mission Control dashboard.

## What a Turn Actually Contains

The first design decision was figuring out what a "turn" means in a JSONL transcript.

A Claude Code transcript does not have clean turn boundaries. You get a stream of event records: `user` type, `assistant` type, `tool` type — interleaved, with timestamps that may be zero or missing. Our parser walks the records and uses a state machine: a `user` record with a real timestamp starts a new turn; subsequent `assistant` and tool records accumulate until the next real user record appears.

Each turn becomes a `ReplayTurn` object:

```typescript
interface ReplayTurn {
  turnIndex: number
  userText: string
  assistantText: string   // full reply, truncated at 2000 chars
  toolCalls: ReplayToolCall[]
  diffFromPrev: string    // line-level diff vs previous assistant text
  startEpoch: number
  durationMs: number
}
```

The `diffFromPrev` field is what surprised us the most in practice. It computes a simple line diff between consecutive assistant replies. When you see the diff shrink to nothing — `(no text change from previous turn)` — you know the agent is stuck. When you see dozens of lines flip from green to red, you know something significant just changed.

## Tool Call Inspection

Tool calls are where most agent bugs live.

A tool invocation in the JSONL has two parts that arrive at different times: the `tool_use` block in the assistant message, and the `tool_result` block in the following user message. We correlate them by `tool_use_id`, extract timing from the timestamp gap between the two records, and package everything into a `ReplayToolCall`:

```typescript
interface ReplayToolCall {
  name: string
  input: string    // truncated JSON, 300 chars max
  output: string   // truncated, 400 chars max
  status: 'ok' | 'error'
  durationMs: number
}
```

In the UI, each tool call renders as a collapsible row. Color-coded by tool type: Bash is orange, file reads are blue, writes and edits are cyan, agent spawns are purple, MCP tools are teal. Error rows turn red. You can expand any row to see the truncated input and output inline.

The truncation is intentional. A full Bash output can be megabytes. We cap inputs at 300 characters and outputs at 400. Enough to see what happened — not so much that the UI becomes unusable.

## Navigating Time

The replay page works like a video player, but for agent thought.

Previous and Next buttons step one turn at a time. A timeline scrubber jumps to any turn directly. Keyboard shortcuts: left and right arrows for step-by-step, Space to toggle auto-play. Auto-play advances one turn every three seconds — slow enough to read, fast enough to find the interesting moment.

Escape goes back. That one felt important to get right. Operators often land on the replay page mid-investigation, mid-panic. They should be able to hit Escape and return to the fleet view without hunting for a back button.

The entry points are wired where they make sense: a "⏮ Replay" button on the per-project detail page and on the Turn Flame Graph, which shows per-turn token spend over time. If you spot a suspicious spike in the flame graph, one click drops you into replay at that project.

## Finding the Latest Session

The API endpoint resolves which transcript to load using a path encoding that mirrors how Claude Code organises its own data.

Claude Code stores transcripts under `~/.claude/projects/<encoded-path>/`, where the encoded path is the project's real filesystem path with all non-alphanumeric characters replaced by hyphens. We resolve the project symlink to its real path, apply the same encoding, then find the newest `.jsonl` file by `mtime`.

```typescript
function findLatestJsonl(slug: string, mcdDir: string): string | null {
  const projectPath = path.join(mcdDir, 'projects', slug)
  let realPath = projectPath
  try { realPath = fs.realpathSync(projectPath) } catch { return null }
  const encoded = encodeProjectCwd(realPath)
  const transcriptDir = path.join(os.homedir(), '.claude', 'projects', encoded)
  // ... find newest .jsonl by mtime
}
```

No configuration. No separate log aggregation pipeline. The data is already on disk; we just needed to know where to look.

## What Watching Agents in Slow Motion Teaches You

We expected session replay to help with debugging. What we did not expect was how much it would change *how we write prompts*.

When you watch an agent step through thirty turns, patterns emerge that are invisible in the output. You see the turns where it re-reads the same file three times before acting — a sign the spec was ambiguous. You see turns where the diff strip shows massive churn — the agent revising its own output repeatedly — and you learn to trace that back to an acceptance criterion that was too open-ended.

The line diff was the insight we underestimated. Reading assistant text turn by turn is slow. Reading what *changed* turn by turn is fast. The diff collapses a twenty-turn session into five or six moments that actually mattered.

We also found that error tool calls cluster. One bad tool call rarely crashes an agent; it usually triggers a recovery loop. Replay makes those loops obvious because you can see the same tool appearing three times in four turns, each time with a slightly different input.

## The Takeaway for AI Observability

Every team running AI agents long enough will accumulate JSONL. Most will grep it. Some will write one-off scripts to parse it. The session replay pattern offers a third option: treat the transcript as structured data, define a clean turn model, and build a debugger on top.

The key engineering choices that made it work:

**Correlate by ID, not position.** Tool calls and their results appear at different indices in the stream. Position-based parsing breaks on nested tool calls. ID-based correlation (`tool_use_id`) is stable.

**Truncate early.** Raw tool outputs are unreadable. 300/400 character caps give you enough signal to diagnose most issues without drowning the UI.

**The diff is the unit of attention.** Step-by-step text comparison between turns surfaces what matters faster than reading full replies sequentially.

When an agent does something unexpected at turn 34, you should be able to find it in thirty seconds. Now we can.
