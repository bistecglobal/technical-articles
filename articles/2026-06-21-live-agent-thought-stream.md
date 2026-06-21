---
title: "Live Agent Thought Stream: Visualizing Tool Calls as Floating Particles on the Fleet Graph"
project: claude-mcd
tags: [AI, DevOps, Observability, React, TypeScript]
status: audited
date: 2026-06-21
commit: f291a54
---

# Live Agent Thought Stream: Visualizing Tool Calls as Floating Particles on the Fleet Graph

When you run a fleet of AI agents — each handling a different project, each operating autonomously for hours at a stretch — the hardest operational question is not "is the agent alive?" but "what is it doing right now?" Status indicators like *busy* or *idle* answer the first question. They say nothing about the second.

We built the Live Agent Thought Stream to close that gap. It tails Claude Code JSONL transcripts in real time, detects every tool call an agent makes, and renders each one as a color-coded floating particle that drifts up from that project's node on the fleet graph. Operators can see, at a glance, which agents are reading files, which are fetching web pages, and which are spawning sub-agents — all without touching a terminal. The feature shipped in commit `f291a54` as part of MCD P54.

## The Problem: Status Is Not Observability

The MCD mission-control dashboard already tracked fleet state per project: running, idle, stalled. It displayed health scores, token budgets, and stall alerts via SSE. But operators managing 10 or more simultaneous Claude sessions reported the same frustration: the dashboard told them an agent was busy, but gave no indication of *what kind* of busy. An agent that has been reading files for two minutes looks identical to one that has silently hung inside a long tool call.

Tail-logging individual transcript files was the manual workaround. It worked, but it required a terminal per project and interrupted the higher-level view the dashboard was designed to provide.

## Architecture: Three Layers

The feature is built across three thin layers: the SSE backend, a shared React context, and the graph renderer.

### Layer 1: Transcript Tailing in `sse.ts`

The SSE module already polled fleet state every few seconds and pushed diffs to connected browsers. We added a `checkToolEvents` function that runs inside the same interval (commit `f291a54`, `apps/mission-control/src/sse.ts`).

For each project slug, `checkToolEvents`:
1. Resolves the project path through symlinks via `fs.realpathSync`.
2. Encodes the resolved path into the Claude Code transcript directory naming scheme (`~/.claude/projects/<encoded>/`).
3. Finds the most recently modified `.jsonl` file by `mtime`.
4. Reads only the new lines since the last poll, tracked by a global `toolLineTracker` map keyed to `(slug → { file, lineCount })`.

```typescript
const startLine = (tracker && tracker.file === latestFile)
  ? tracker.lineCount
  : lines.length   // new file: reset cursor to end, don't replay history
toolLineTracker.set(slug, { file: latestFile, lineCount: lines.length })
```

Resetting the cursor to `lines.length` when the file changes prevents replaying an entire previous session when a new transcript begins. Only *new* tool calls, from the moment the browser connects, appear as particles.

Each new line is parsed as a JSONL record. If the record type is `assistant` and any content block has `type === 'tool_use'`, the tool name is extracted and broadcast:

```typescript
if (block.type === 'tool_use' && block.name && !block.name.startsWith('mcp__mcd__')) {
  broadcast({ type: 'tool-event', data: { slug, toolName: block.name } })
}
```

One deliberate filter: any tool whose name starts with `mcp__mcd__` is suppressed. These are the bot's own Discord reply, react, and fetch tools — they're internal bookkeeping, not meaningful signals for an operator watching agent reasoning.

### Layer 2: `FleetContext` Ring Buffer

The existing `FleetContextProvider` already maintained fleet state and stall events from the SSE stream. We extended it with a `toolEvents` array capped at 200 entries (`apps/mission-control/components/FleetContext.tsx`, commit `f291a54`):

```typescript
const MAX_TOOL_EVENTS = 200
let toolEventIdCounter = 0

} else if (parsed.type === 'tool-event') {
  const d = parsed.data as { slug: string; toolName: string }
  const ev: ToolEvent = { slug: d.slug, toolName: d.toolName, id: ++toolEventIdCounter }
  setToolEvents((prev) => [...prev.slice(-MAX_TOOL_EVENTS + 1), ev])
}
```

Each event gets a monotonically increasing integer ID. The graph renderer uses this ID to detect and skip events it has already processed, preventing duplicate particle spawns on re-renders.

### Layer 3: Particle Rendering in `ProjectGraph.tsx`

The graph renderer consumes `toolEvents` from `FleetContext` and converts each new event into a `ThoughtParticle` — a small data structure with a position derived from the corresponding graph node, a spawn timestamp, and display style:

```typescript
const TOOL_CATEGORIES: Record<string, { color: string; short: string }> = {
  Read:   { color: '#22D3EE', short: 'Read'   },  // cyan  — file ops
  Edit:   { color: '#22D3EE', short: 'Edit'   },
  Write:  { color: '#22D3EE', short: 'Write'  },
  Glob:   { color: '#22D3EE', short: 'Glob'   },
  Grep:   { color: '#22D3EE', short: 'Grep'   },
  WebFetch:   { color: '#F59E0B', short: 'Fetch'    },  // amber — web
  WebSearch:  { color: '#F59E0B', short: 'Search'   },
  Agent:      { color: '#A78BFA', short: 'Agent'    },  // purple — sub-agents
  Workflow:   { color: '#A78BFA', short: 'Workflow' },
}
```

Three design constraints kept the overlay readable rather than noisy:

- **Max 3 particles per node.** When a fourth arrives, the oldest for that node is dropped before the new one is added.
- **3-second lifetime.** A `setTimeout` fires 100 ms after expiry and prunes the particle from state. A separate 1-second interval sweeps any stragglers.
- **Toggle, not always-on.** Pressing `T` shows or hides the overlay; the preference persists in `localStorage`. Operators who prefer a clean graph are not forced to see particles.

The particle spawning hook tracks the last processed event ID in a `useRef` to avoid re-processing events on every render:

```typescript
const lastProcessedToolEventId = useRef(-1)
useEffect(() => {
  if (!showThought || toolEvents.length === 0) return
  const latest = toolEvents[toolEvents.length - 1]
  if (!latest || latest.id <= lastProcessedToolEventId.current) return
  lastProcessedToolEventId.current = latest.id
  // ... spawn particle at node position
}, [toolEvents, showThought])
```

## Design Decisions Worth Noting

**Why tail transcripts rather than instrument the Claude SDK?** MCD launches Claude inside a dedicated tmux session (`tmux new-session -d -s <name> 'claude --mcp-config ...'`); there is no in-process hook. Transcript files are the only authoritative, append-only record of what the agent did. Tailing them is the same approach used by the weekly fleet report (`f291a54` sibling commit `3e3f957`), so the path-encoding and file-selection logic was already proven.

**Why suppress `mcp__mcd__` tools?** These appear in virtually every assistant turn — the bot is constantly replying to Discord, reacting, fetching messages. Showing them as particles would paint every agent as perpetually active, drowning out the signal from actual reasoning tools. The filter costs one string prefix check per tool call.

**Why cap at 3 particles per node?** Canvas nodes are small. More than three overlapping labels become unreadable at fleet scale. Dropping the oldest rather than the newest ensures the display always shows the most recent reasoning, which is the information operators actually want.

## What Operators See

With the thought stream enabled, the fleet graph becomes a live cognitive map. A cyan pulse of `Read` and `Grep` labels rising from a node signals an agent deep in research. An amber `Fetch` or `Search` particle means it went to the web. A purple `Agent` burst means it spawned a subagent — a pattern associated with complex multi-step tasks. Silence from a node that should be busy is itself a signal: no particles means no tool calls, which often precedes a stall alert.

The overlay does not replace stall detection or health scores. It adds a qualitative layer on top: not just "is the agent working?" but "what kind of work is it doing?"

## Lessons Learned

Transcript tailing is surprisingly robust as an instrumentation strategy. Claude Code's JSONL format is append-only and line-delimited, which makes incremental reads safe under concurrent writes. The line-position tracker approach — storing `{ file, lineCount }` per project rather than a byte offset — handles file rotation (new sessions start a new `.jsonl`) correctly without complex inotify or file-watch infrastructure.

The cap-and-drop particle model translated directly from real-time data visualisation practice: when the rate of incoming events exceeds what a human can process, the right answer is not to queue everything but to show only what fits and let older data go. An operator glancing at a fleet graph needs the current state of agent reasoning, not a complete audit trail.

---

The Live Agent Thought Stream is live in the MCD mission-control dashboard (commit `f291a54`, PR #95). Toggle it with `T` on any fleet graph view.
