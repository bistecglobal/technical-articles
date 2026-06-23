---
title: "One Number Per Agent: Building a Composite Health Scorecard for an AI Fleet"
project: claude-mcd
tags: [AI, DevOps, Observability, Claude, Engineering]
status: audited
date: 2026-06-23
---

# One Number Per Agent: Building a Composite Health Scorecard for an AI Fleet

We run close to twenty AI agents continuously — each in its own Claude Code session, each working on a different project. On any given morning, a few of them are thriving: turning work out efficiently, memory files tidy, goals hit. Others have quietly degraded. Maybe a project's memory files haven't been updated in three weeks. Maybe an agent is burning 80% of its context window without producing useful output. Maybe it started flagging anomalies in the middle of the night.

For a long time, checking on fleet health meant opening five separate dashboards. Then a new intern joined and asked the obvious question: "Is there one page that tells me which agents to worry about?" There wasn't. Now there is.

## The Problem with Per-Metric Dashboards

We had rich observability before we had a scorecard. The Mission Control dashboard already tracked turn quality over time, memory health across five dimensions, goal keyword hit rates, context pressure, and statistical anomalies. Each page was correct and detailed. None of them answered the question: *which projects need attention right now?*

The problem is that each metric uses a different scale and different failure modes. An agent can have excellent turn quality but completely stale memory. An agent can hit its goals reliably but be approaching context saturation. Comparing agents across metrics required holding five numbers in your head simultaneously and making a judgment call — which is exactly the kind of cognitive load that causes things to slip through.

A composite score forces a prioritization decision, which is uncomfortable but necessary: you have to say which signals matter most.

## What We Measure and Why

The scorecard aggregates five dimensions, each normalized to a 0–100 scale:

**Turn Quality (30%)** measures whether agents are producing meaningful output. We parse Claude Code JSONL transcripts over the last 24 hours, scoring each assistant turn by reply length (longer substantive replies score higher), tool call count (agents doing work), and absence of error patterns. An agent that has been failing silently — short error replies, no tool use — scores low here.

**Memory Health (30%)** is itself a composite of five sub-dimensions: recency (was memory updated in the last 7 days?), coverage (are there at least 3 memory files?), density (do those files contain enough content?), stability (is memory content stable, not thrashing?), and freshness (was memory written after the last session?). An agent whose memory hasn't been touched in three weeks loses most of its recency score before anything else.

**Goal Progress (20%)** measures whether agents are actually working toward their defined goals. Each project can have a GOAL.md — a brief statement of its mission injected at the start of every session. The goal heatmap tracks how often keywords from that goal appear in transcript turns, day by day, over the last 30 days. A low goal progress score often means an agent has drifted from its purpose.

**Context Pressure (10%)** inverts raw context utilization into a health signal. A Claude model has a 200k-token context limit. Agents that consistently push toward that ceiling produce degraded output and miss earlier context. We invert the pressure score so that low context usage = high health.

**Anomaly Score (10%)** also inverts: zero anomalies earns a perfect 100. We use z-score detection with a 7-day baseline to flag deviations in inter-turn gap, tool calls per turn, and output tokens per turn. More anomalies = lower score.

The weights reflect a deliberate opinion: what an agent *produces* (turns) and what it *retains* (memory) are the primary indicators of health. Goals add accountability. Context pressure and anomalies are early-warning signals that matter, but rarely explain current health on their own.

## The Aggregation Logic

The score computation is intentionally simple. Here's the core of it:

```typescript
const W = {
  turnQuality: 0.30,
  memoryHealth: 0.30,
  goalProgress: 0.20,
  contextPressure: 0.10,
  anomalyScore: 0.10,
}

function computeOverall(row: ScorecardRow): number {
  let sum = 0, totalW = 0
  const dims: Array<[keyof typeof W, number | null]> = [
    ['turnQuality', row.turnQuality],
    ['memoryHealth', row.memoryHealth],
    ['goalProgress', row.goalProgress],
    ['contextPressure', row.contextPressure],
    ['anomalyScore', row.anomalyScore],
  ]
  for (const [key, val] of dims) {
    if (val !== null) { sum += val * W[key]; totalW += W[key] }
  }
  if (totalW === 0) return 0
  return Math.round(sum / totalW)
}
```

The null handling matters. When an agent has no goal set, goal progress returns null and is excluded from the weighted average — its 20% weight is redistributed across the remaining dimensions. This prevents new projects from being unfairly penalized for not having goals configured yet.

On the frontend, the page fires six API requests in parallel on load, aggregates their results into per-project rows, and runs `computeOverall` client-side. We deliberately avoided creating a new backend endpoint that combines all six sources — keeping the aggregation on the client means each underlying API stays simple, cached independently, and useful in isolation.

## What the Dashboard Shows

The default sort is *overall ascending* — lowest scores first. This seems counterintuitive until you use it: every time you open the scorecard, the most troubled agents are at the top. You don't have to hunt.

Color coding follows a traffic-light scheme: scores ≥70 are green, 40–69 amber, below 40 red. Rows with critical overall scores get a subtle red background tint so your eye finds them before you read the number.

Clicking any score cell for a specific dimension navigates directly to the detail page for that dimension — turn quality, memory health, goal heatmap, context pressure, or anomalies. Clicking the row itself expands a quick-link accordion with all five links for that project, letting you drill straight into whatever signal triggered the low overall score.

The table auto-refreshes every five minutes. For a fleet that runs continuously, that cadence is fast enough to catch a sudden degradation during a work session without overwhelming the underlying APIs.

## What We Learned Building It

**The hardest decision was the weights.** Anomalies sit at 10% rather than higher for a specific reason: an agent going through a legitimate burst of high activity — many tool calls, dense output — would trigger anomaly flags even when performing well. Treating anomalies as an early-warning signal rather than a primary health indicator, and weighting turn quality and memory at 30% each, produced a score that stayed aligned with what operators could see in the agent's actual output.

**"Worst first" is a policy, not a preference.** The default ascending sort feels wrong if you think of scores as grades. But for operational health monitoring, you are always triaging. Making the worst agents visible immediately changed how the team uses the dashboard — it became a morning check rather than an exploration.

**Null is not zero.** New projects often lack goals or have sparse transcript history. Treating null as 0 would have made every new project look critically unhealthy, which would have destroyed trust in the score. The weight-redistribution approach means a project with no goal data still gets a fair score across the four dimensions it does have.

**One composite score doesn't replace detail pages.** It tells you *who* to look at, not *why* they're struggling. The drill-down links in the expanded row are as important as the score itself. The scorecard is a triage tool — the goal is to get operators to the right detail page faster, not to make detail pages unnecessary.

## Where This Fits

The composite scorecard is now the first page the team opens in Mission Control. It surfaces problems that would have been invisible for days across a fleet of close to twenty agents — a project whose memory stopped updating, an agent slowly filling its context window, a goal that hasn't been pursued in two weeks.

At its core, the scorecard is an argument that fleet health is a real concept worth defining explicitly. Picking five dimensions and assigning them weights is an imperfect judgment call. But a visible, arguable score is more useful than a silent collection of charts that nobody checks.

When you run agents at scale, the question isn't whether they're working — it's whether they're healthy enough to keep working. The scorecard makes that question answerable in under a second.
