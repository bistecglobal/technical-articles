---
title: "Drawing the Family Tree: Visualizing the Agent Spawn Hierarchy"
project: claude-mcd
tags: [AI, Agents, Observability, DeveloperTooling, Claude]
status: audited
date: 2026-06-25
---

A single Claude agent is easy to reason about. You ask it something, it works, it answers. But the moment that agent starts spawning *other* agents — a planner that delegates to three researchers, each of which spins up a verifier — the mental model collapses. The transcript is still there, line by line, but the *shape* of the work is gone. Who delegated to whom? How deep did the delegation go? Which subagent took on the bulk of the task?

In a flat JSONL transcript, that structure is invisible. You can scroll for ten minutes and never see that turn 47 fanned out into a four-level deep tree of nested agents. The information is technically present and practically unreadable.

Mission Control, the dashboard for our multi-channel Discord agent fleet (MCD), now has a page that reconstructs that shape. The `/agent-tree` view reads an agent's raw transcript and renders a collapsible, color-coded hierarchy of every `Agent` and `Task` call — turning a wall of JSON into a family tree you can actually read.

## The problem: delegation has no native representation

Claude Code records a session as a stream of JSONL lines: user messages, assistant messages, and tool results. When an agent delegates work, it emits a `tool_use` block named `Agent` (or the legacy `Task`) with a `description` and a `prompt`. The subagent's output comes back later as a `tool_result` keyed by the same tool-use id.

Nothing in that stream draws the connection visually. The delegation is encoded as two events separated by however many lines the subagent's work produced. Multiply that across a session with dozens of turns and several spawn points, and the delegation structure — arguably the most important thing about a multi-agent run — is the one thing you can't see.

We wanted a view that answers three questions at a glance: which turns spawned agents, how many agents each turn produced, and how deep the nesting went.

## The approach: rebuild turns, then extract the tree

The work happens in a single API route that reads the transcript and emits a structured tree. It runs in two passes.

**Pass one — reassemble turns.** Claude Code can split one logical assistant response across multiple JSONL lines (multi-step tool use). The route walks the stream and coalesces consecutive assistant lines into one pending turn, accumulating their content blocks and summing token usage. A turn is "closed" when the next user message arrives:

```ts
} else if (msg.role === 'assistant') {
  const blocks = Array.isArray(msg.content) ? msg.content : []
  const tokens = msg.usage
    ? (msg.usage.input_tokens ?? 0) + (msg.usage.output_tokens ?? 0)
    : null
  if (pendingAssistant) {
    pendingAssistant.blocks.push(...blocks)        // multi-step turn
    if (tokens !== null) pendingAssistant.tokens = (pendingAssistant.tokens ?? 0) + tokens
  } else {
    pendingAssistant = { blocks, ts: msg.timestamp ?? '', tokens }
  }
}
```

Tool results are indexed separately in a `Map` keyed by `tool_use_id`, so each spawn can later be paired with the output it produced. Crucially, the route only keeps turns that actually contain an `Agent` or `Task` call — the noise of ordinary tool use never reaches the UI.

**Pass two — extract the nodes.** For each retained turn, `extractAgentNodes` walks the assistant's content blocks and turns every agent spawn into an `AgentNode`:

```ts
for (const block of assistantBlocks) {
  if (block.type !== 'tool_use') continue
  if (block.name !== 'Agent' && block.name !== 'Task') continue

  const input = block.input ?? {}
  const promptText = (input.prompt as string) ?? ''
  const description = (input.description as string) ?? ''
  const label = description || snippet(promptText, 60)
  // ...
}
```

The node carries its tool name, a human label (the agent's `description`, falling back to a 60-character prompt snippet), a longer 120-character prompt snippet for hover detail, its depth, and its children.

## Finding the children

Here is the honest part. Claude Code's transcript records *that* a subagent ran and *what it returned*, but a subagent's own internal spawns live in its own separate transcript, not inline in the parent's result. So the parent stream alone can't give you a guaranteed-complete tree below the first level.

Rather than over-claim, the route does best-effort nested detection. It scans each subagent's returned text for tell-tale patterns — phrases like "spawned subagent" or an `Agent({ description: ... })` literal echoed in the output — and attaches any matches as child nodes, with a hard depth cap to keep the recursion bounded:

```ts
function parseNestedAgents(resultText: string, depth: number, parentId: string): AgentNode[] {
  if (depth > 3) return []
  const agentPatterns = [
    /spawned subagent[:\s]+["']?([^"'\n]{5,80})/gi,
    /Agent\(\s*\{[^}]*description[:\s]+["']([^"']{5,80})/gi,
  ]
  // ...collect matches as child AgentNodes
}
```

This is a pragmatic trade-off: the first level of delegation is exact (it comes straight from the parent's tool-use blocks), and deeper levels are inferred from text. The UI never pretends otherwise — depth is rendered honestly, and a node with no detectable children simply shows none.

Two small recursive helpers, `countAgents` and `maxDepth`, summarise each turn so the header can show "12 agents, 3 levels deep" without expanding anything.

## The view: a tree you can collapse

The frontend renders each turn as a card and each agent as an indented row. Depth drives everything visual: a five-color palette (cyan → purple → green → amber → red) cycles by depth, painting both a thin vertical depth-line and the tool badge, so you can read nesting level by color alone. Rows indent 20 pixels per level, carry a ▸/▾ toggle when they have children, and reveal the full prompt snippet beneath the label.

```
▾ turn 47 · 12 agents · 3 deep · 84k tokens
  ▾ AGENT  plan the migration
    ▸ AGENT  audit imports in module A
    ▸ AGENT  audit imports in module B
    ▾ AGENT  rewrite call sites      2 sub
        AGENT  verify module A
        AGENT  verify module B
```

By default the route returns the last 20 agent-bearing turns, but a `turn` query parameter drills into a single turn — handy when you've spotted one expensive fan-out and want to study just that subtree. A slug selector switches between projects in the fleet, all resolved from the same `channels.json` the rest of Mission Control reads.

## What we took away

Two lessons stand out.

First, **structure is a first-class observability signal, not a derived one.** We already had transcript replay, token accounting, and tool-call stats. None of them surfaced delegation shape, because shape isn't a number — it's a topology. Reconstructing it took a deliberate two-pass parse; it would never have fallen out of a metrics aggregation.

First-level delegation is exact and the deeper inference is clearly bounded — and naming that limit in the code is what keeps the view trustworthy. An observability tool that quietly guesses is worse than one that shows you exactly how much it knows.

Second, **the cheapest place to add this was the data you already keep.** No new logging, no instrumentation in the agents themselves, no schema change. The JSONL transcripts Claude Code writes for every session already contained the entire tree; it just needed someone to read them structurally. For any team running multi-agent workflows, that transcript is a richer source of truth than it first appears — and a family tree is one of the more revealing things hiding inside it.
