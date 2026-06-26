---
title: "The Sankey That Shows How Agents Move: Reconstructing Fleet State From Raw Logs"
project: claude-mcd
tags: [AI, Agents, Observability, DevOps, Claude]
status: draft
date: 2026-06-26
---

Ask an operator running a fleet of autonomous AI agents a simple question — "how much of the time are your agents actually working versus sitting idle versus wedged?" — and watch them struggle. Not because the answer is unknowable, but because nothing records it. An agent's life is a stream of messages and tool calls in a log file. The *states* it moves through — busy, idle, stuck, crashed — are never written down. They only exist in the gaps and the rhythm between events.

Mission Control, the dashboard for our multi-channel Discord agent fleet (MCD), now reconstructs those states after the fact and renders them as a Sankey diagram. You can see, at a glance, that the fleet spends 60% of its time idle, that "active → idle" is the most common transition, and that circuit-breaker trips almost always flow back to idle rather than recovering straight into work. A companion view does the same trick for reliability: it derives a per-tool error rate from the same logs. Neither required adding a single line of instrumentation to the agents.

## The problem: state is implicit

A Claude Code session is recorded as JSONL — one JSON object per line: user messages, assistant messages, tool calls, tool results, each with a timestamp. That log faithfully captures *what happened*. It says nothing about *what state the agent was in*.

"Idle" isn't an event. It's the absence of events for a while. "Stuck" isn't logged; it's inferred from a watchdog. "Circuit open" lives in a separate breaker log entirely. To answer the operator's question you have to fuse multiple signals and reconstruct a timeline of intervals — and that reconstruction is exactly what no single log line gives you.

## Reconstructing the timeline

The state-transitions endpoint defines four states — `idle`, `active`, `stuck`, `circuit-open` — and rebuilds, per project, the intervals the agent spent in each over a 30-day window.

It pulls from two sources. The first is genuine user messages from the transcript. "Genuine" matters: a tool result is also recorded with `role: "user"`, so a naive count would treat every tool round-trip as user activity. The filter excludes them:

```ts
function isGenuineUserMessage(line: JsonlLine): boolean {
  if (line.message?.role !== 'user') return false
  const c = line.message?.content
  if (!Array.isArray(c) || c.length === 0) return true
  return c[0]?.type !== 'tool_result'
}
```

The second source is the circuit-breaker log, `circuit-events.jsonl`, which records `open` and `close` events when MCD's breaker trips a crash-looping subprocess.

Both streams are merged into one chronological event list, and a single pass walks it, opening and closing state intervals as the rules dictate. The core heuristic is a gap threshold: if more than 30 minutes pass between messages, the agent was idle in between.

```ts
} else if (lastMsgTs > 0 && e.ts - lastMsgTs > IDLE_GAP_MS) {
  // gap → idle between lastMsg and now
  if (s === 'active' || s === 'stuck') {
    recordTransition('idle', lastMsgTs + IDLE_GAP_MS)
  }
  recordTransition('active', e.ts)
}
```

Circuit events override message activity — while the breaker is open, messages are ignored, and a `close` returns the agent to idle rather than pretending it leapt straight back to productive work. Every interval boundary is a transition, and the endpoint tallies a full from→to matrix, the count of each transition, the average time spent in the *from* state before leaving it, and each state's share of total tracked time.

One small but honest detail lives in the code: a comment notes that the loop casts the state variable to defeat TypeScript's control-flow narrowing. Across loop iterations, the compiler "forgets" that `recordTransition` can mutate the state, and narrows it to a stale literal. The cast keeps the runtime logic correct where the type system would otherwise mislead. It's the kind of wrinkle you only hit when state machines meet a structural type checker.

## Drawing it

The result is fed to an SVG Sankey. Node height is proportional to how often each state was entered; flow width is proportional to transition count; each state node carries its time-share percentage. The shape tells the story faster than any table:

```
idle ████████████  60%  ──┐
                          ├──► active ██████  31%
active ──────────────────┘        │
                                  └──► idle
stuck ██ 6%   circuit-open █ 3%
```

A fat idle band with a thin trickle into `active` is a fleet that's mostly waiting. A thick `active → stuck` flow is a fleet that keeps wedging. The diagram makes the difference legible in seconds.

## The same trick, for reliability

If you can derive *state* from the transcript, you can derive *failure* from it too. The tool-error-rate endpoint reads the same JSONL and answers a different question: which tools are failing, and how often?

It runs two passes per file. The first builds a map from each tool-use id to its tool name, because the assistant message names the tool but the *result* — where the error lives — only references it by id. The second pass walks the tool results and counts errors:

```ts
const isErr = block.is_error === true || (() => {
  if (Array.isArray(block.content)) {
    return block.content.some((c) => c.type === 'error')
  }
  return false
})()
```

Each tool gets a call count, an error count, a one-decimal error-rate percentage, the timestamp and snippet of its most recent failure, and a 14-day error sparkline. Bot-internal MCP tools (`mcp__mcd__*`) are filtered out by default so the operator sees the tools that actually do work, not the plumbing. The table sorts worst-rate-first, so the flakiest tool is always at the top.

## What we took away

Two things stand out.

First, **the log already knew.** No agent was modified, no new event type was emitted, no schema migrated. State and failure were latent in data we were already keeping — they just needed reconstruction. For anyone running autonomous agents, the transcript is a far richer telemetry source than it looks; most "we need to add metrics for X" instincts are really "we need to read our logs differently."

Second, **derived signals must be honest about their assumptions.** The 30-minute idle threshold is a choice, not a truth. The genuine-user-message filter exists because the obvious count is wrong. Naming those assumptions in the code — and not letting the type checker quietly corrupt a state machine — is what separates an observability tool you trust from one that confidently lies. A Sankey that's subtly miscounting is worse than no Sankey at all.
