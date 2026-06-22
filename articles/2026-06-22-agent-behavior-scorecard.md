---
title: "The Agent Behavior Scorecard: Measuring Tool-Call Efficiency Across an AI Fleet"
project: claude-mcd
tags: [AI, DevOps, Observability, Agent Engineering, Developer Tooling]
status: draft
date: 2026-06-22
---

# The Agent Behavior Scorecard: Measuring Tool-Call Efficiency Across an AI Fleet

You know your AI agent spent $14 this month and averages 2.3 seconds per turn. But do you know what it's actually doing between those turns?

For months, the Mission Control dashboard for our multi-agent fleet answered every cost and latency question you could ask. Token totals, cost estimates by model tier, p95 latency, 7-day token trends — all there. What it couldn't answer was the one question that started nagging every time an agent ran long: is this agent working, or is it spinning?

Tokens and latency measure *how much* an agent does. They say nothing about *how it does it*.

## The Gap Between Cost and Behavior

An agent can burn 200K tokens in a day two very different ways. In the first scenario, it reads 40 files, writes a comprehensive spec, opens a PR, and runs tests — each tool call producing a burst of meaningful output. In the second, it reads the same three files 30 times, calls a search tool repeatedly with slight variations, and produces modest results. Same token count. Wildly different behavior.

With a single agent, you notice this by watching transcripts. With 19 agents running simultaneously across projects — each operating autonomously, injecting its own heartbeat messages, managing its own backlog — transcript watching doesn't scale. You need a signal that summarizes behavioral quality at a glance.

That signal is tool-call efficiency.

## Mining Behavior from Transcripts

Every Claude Code session writes a JSONL transcript — one JSON record per event, timestamped, capturing every assistant turn, tool call, and token count. We were already reading these files to compute cost and latency metrics. The behavioral data was sitting right there; we just weren't extracting it.

For each assistant turn in the transcript, we count how many `tool_use` blocks appear in the message content. We track which tools were called and how many times across all turns, and we record the output token count for each turn. That's three numbers per turn: tool calls made, tools used, and tokens written.

Aggregated across all sessions for a project:

```typescript
const rawEfficiency = totalToolCalls > 0 ? totalOutput / totalToolCalls : 0
const efficiencyScore = Math.min(100, Math.round((rawEfficiency / 2000) * 100))
```

The efficiency score answers a simple question: on average, how many output tokens does each tool call produce? We normalize against 2,000 — a rough empirical threshold for "one tool call produced a meaningful amount of work." A score of 80 means the agent averages 1,600 output tokens per tool call. A score of 20 means it averages 400 — lots of tool activity, modest output.

## The ScoreCard Component

The `ToolStats` object returned by the metrics API carries four values: the top tools by call count, average calls per turn, average output tokens per turn, and the efficiency score. These surface in the fleet dashboard as a collapsible `ScoreCard` nested inside each project's expanded metrics row.

The efficiency gauge is color-coded: green for scores at or above 70, amber for 40–69, red below 40. The top five tools render as a horizontal bar chart — relative counts, not absolute — so you see the *shape* of an agent's tool usage at a glance. A bar chart dominated by `Read` and `Edit` tells a different story than one dominated by `WebSearch` and `Bash`.

Two supporting stats sit beside the gauge: average tool calls per turn and average output tokens per turn. Together these give you the raw inputs to the efficiency score and let you distinguish between two different ways of scoring, say, 60. An agent making 1.5 calls/turn with 1,200 tokens/turn behaves differently from one making 8 calls/turn with 6,400 tokens/turn, even if the ratio is similar.

## What the Scorecard Revealed

Once we had the scorecard running across the fleet, a few patterns became visible immediately.

Projects with long-running autonomous tasks — agents working through backlogs, implementing specs across multiple files — tend to cluster in the 65–85 range. They read context, then write substantial output: code, commits, documentation. The ratio holds because each tool call has a clear purpose.

Projects in exploration mode — researching a topic, mapping a codebase, generating proposals — often score lower (40–60). This is expected and fine. A research phase involves many reads and searches that produce modest immediate output. Knowing this is *expected* behavior is itself useful; a 55 score during a research phase is healthy, where a 55 score during a known implementation phase is worth investigating.

The clearest red flags are scores below 30 combined with a high call count and modest output per turn — the signature of an agent that has lost its thread. It's reading the same context repeatedly, calling search tools in loops, not committing to an action. Before the scorecard, this would show up as unexpectedly high token spend with little to show for it. Now it's visible in the behavior signal before the bill arrives.

## Lessons We Took Away

**JSONL transcripts contain more than you think.** We had been computing cost and latency from these files for months. Tool-call frequency, tool distribution, and output-per-call were all there — we just weren't looking. Before building a new telemetry pipeline, check whether the data you already collect contains the signal you need.

**Ratio metrics beat raw counts.** Total tool calls is a noisy number — it correlates with how much work the agent did, which correlates with how long it ran. Output tokens per tool call filters out duration and focuses on the quality of each action. This is the same reason p95 latency is more useful than total request count.

**Behavioral context makes cost data actionable.** An efficiency score of 45 with an amber gauge prompts a useful question: is this agent in exploration mode, or is something wrong? The score alone doesn't answer it, but it surfaces the question before the next token budget review. That's the goal — not to replace human judgment, but to give it a timely prompt.

Observability for AI agents is still a young discipline. Most of what exists borrows from infrastructure monitoring: cost, latency, uptime. Those metrics matter, but agents are different from services. They make decisions. They choose which tools to call and in what order. The behavioral layer — what an agent actually does with the context and tools it's given — is where the interesting signal lives.

The scorecard is one slice of that layer. We're still mapping the rest.
