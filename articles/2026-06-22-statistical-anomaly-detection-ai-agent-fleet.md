---
title: "When Your AI Agents Start Behaving Strangely: Statistical Anomaly Detection for Claude Fleets"
project: claude-mcd
tags: [AI, DevOps, Observability, Claude, Monitoring]
status: audited
date: 2026-06-22
---

# When Your AI Agents Start Behaving Strangely: Statistical Anomaly Detection for Claude Fleets

Somewhere in your fleet of AI agents, one of them is doing something odd. Maybe it's making three times as many tool calls as usual. Maybe it's been silent for forty minutes when it normally responds every two. You won't know until a human notices — which, in an autonomous fleet running around the clock, might be hours too late.

We hit this problem at BistecGlobal running a fleet of autonomous Claude Code agents across 19 active projects. Our Mission Control dashboard already showed agent health scores, token budgets, and session replays. But health scores aggregate history — they tell you how an agent has been performing, not whether it's *currently acting weird*. We needed something more like an on-call alert: flag agents whose recent behavior has deviated significantly from their own baseline.

The answer turned out to be a technique borrowed straight from SRE playbooks: z-score anomaly detection, applied to behavioral metrics extracted from Claude Code JSONL transcripts.

## The Data Was Already There

Claude Code records every conversation turn to a JSONL file under `~/.claude/projects/`. Each assistant turn carries a timestamp, output token count, and an array of tool use blocks. That's three natural behavioral dimensions:

- **Inter-turn gap** — how many minutes elapsed since the previous turn
- **Tool calls per turn** — how many tools the agent invoked
- **Output tokens per turn** — how much the agent wrote

These metrics vary considerably across projects (a code-generation agent writes more than a planning agent), but each project has its own stable pattern over days of operation. The insight is that the right baseline for "normal" is a project's own recent history — not a fleet-wide average.

## The Algorithm: Self-Calibrating Thresholds

For each project, the detection algorithm reads all transcript JSONL files and extracts a `TurnStat` record for every assistant turn in the last seven days:

```typescript
interface TurnStat {
  tsMs: number
  outputTokens: number
  toolCallCount: number
  interTurnGapMins: number
}
```

It then splits the window into two groups: the **baseline** (all turns except the last three) and the **recent** window (the last three turns). The split is deliberate — three turns is enough signal to detect a behavioral shift without being so short that a single unusual turn triggers a false alarm.

For each metric, the algorithm computes the baseline mean and sample standard deviation, then calculates a z-score for the recent window:

```typescript
function sampleStd(values: number[], mean: number): number {
  if (values.length < 2) return 0
  const variance = values.reduce((s, v) => s + Math.pow(v - mean, 2), 0) / (values.length - 1)
  return Math.sqrt(variance)
}

// Later in analyzeProject():
const mean = baselineVals.reduce((s, v) => s + v, 0) / baselineVals.length
const std = sampleStd(baselineVals, mean)
if (std < 0.01) continue   // skip flat-line metrics — no signal in zero-variance data

const recentMean = recentTurns.reduce((s, t) => s + m.get(t), 0) / recentTurns.length
const zScore = Math.abs((recentMean - mean) / std)
if (zScore < 2) continue

entries.push({
  severity: zScore >= 3 ? 'critical' : 'warn',
  zScore: Math.round(zScore * 100) / 100,
  // ...
})
```

Two guard conditions prevent noise. First, the algorithm requires at least 7 turns in the window and 4 baseline turns — projects with insufficient history are skipped entirely rather than generating unreliable scores. Second, metrics whose standard deviation is near zero (an agent that always makes exactly 2 tool calls per turn, for example) are skipped — a flat baseline means any deviation looks extreme, but there's no statistical information to work with.

Thresholds follow the standard σ convention from industrial process control:
- **≥ 2σ** → warn (behavior is unusual)
- **≥ 3σ** → critical (behavior is very unusual)

## What the Dashboard Shows

The `/anomalies` page in Mission Control displays detected anomalies in a sortable table. Each row shows the project slug, which metric triggered, the current 3-turn average, the baseline mean ± stddev, the z-score, severity, and a sparkline of the last 20 turns with the anomalous region highlighted in red.

```
Project       Metric              Current   Baseline (mean ± σ)   Z-score   Severity
───────────────────────────────────────────────────────────────────────────────────
gasflow       tool calls/turn     12.3      3.1 ± 1.4             6.57σ     ● CRITICAL
keyflow       inter-turn gap      47.2 min  8.3 ± 6.1             6.38σ     ● CRITICAL
specclaw      output tokens/turn  4820      1240 ± 890            4.02σ     ▲ WARN
```

The page auto-refreshes every 60 seconds and adds severity filter chips so operators can focus on critical anomalies first. Clicking a project slug navigates to its detail page for deeper investigation.

## Design Decisions Worth Calling Out

**Sample standard deviation, not population.** The baseline window is a sample of the agent's behavior — it doesn't represent all possible turns. Using `n - 1` in the variance denominator gives a more conservative (wider) spread estimate, which means higher thresholds are required to trigger. This reduces false positives for short-history projects.

**Three recent turns, not one.** A single extreme turn — an unusually large response, a burst of tool calls to handle a complex request — shouldn't fire an alert. Averaging the last three turns smooths over one-off spikes while remaining sensitive to a sustained behavioral shift.

**Per-metric independence.** Each of the three metrics is evaluated independently. An agent can be flagged for anomalous token output without triggering on tool calls, which helps operators understand *why* the behavior is unusual — is it writing more? Using more tools? Or stalling?

**No external instrumentation.** The entire feature reads from JSONL files that Claude Code already writes. There's no tracing SDK to integrate, no sidecar to deploy, no schema to migrate. The data is already there — the analysis was the missing piece.

## What It's Designed to Detect

The three behavioral dimensions map to distinct failure modes in autonomous agents.

**Context runaway** — output tokens per turn spiking above baseline. As an agent's context window fills, responses can grow longer as the model re-explains prior reasoning or attempts to compensate for context compression. An upward shift in token output is often the earliest detectable symptom, visible before the agent stalls or repeats itself.

**Stall after dependency failure** — inter-turn gap climbing above baseline. When an MCP tool times out or a subprocess hangs, the agent stops producing turns. A sustained silence that exceeds 2–3 standard deviations from the project's normal cadence surfaces as a high-severity anomaly on the inter-turn gap metric.

**Exploration spiral** — tool calls per turn rising above baseline. A planning agent handed an ambiguous task may loop through repeated file reads and searches without converging. The tool call rate is the signal: if the agent is making significantly more tool calls per turn than its own baseline, it is likely iterating without progress rather than executing.

## Where This Sits in the Observability Stack

Anomaly detection fills a specific gap between two existing monitoring layers. The token burn rate gauge shows real-time fleet activity — it tells you *something* is happening. The health score trends tell you how an agent has performed over weeks. Anomaly detection occupies the middle ground: it watches for *behavioral departure from the agent's own norm*, in near-real-time, without requiring operators to define manual thresholds for each project.

For teams building autonomous agent pipelines, the general principle is broadly applicable: the behavioral signal is already in your transcript data. What was missing was the statistics to surface it.

---

*The fleet anomaly detection page and its supporting API are part of BistecGlobal's Mission Control dashboard, an open-source component of the MCD platform for autonomous AI agent orchestration.*
