---
title: "The Average Hid a Split Fleet: A Convergence Histogram for AI Agents"
project: claude-mcd
tags: [AI, DevOps, Observability, Agents, DataViz]
status: draft
date: 2026-06-24
---

A fleet of agents with a mean convergence of 0.5 could be in one of two completely opposite situations. Either every agent is plodding along at exactly half-done — uniformly mediocre but fine — or half the fleet has nailed its goal and the other half is dead in the water. Same average. Wildly different operational reality. And the single number on the dashboard cannot tell them apart.

That ambiguity is why we stopped trusting the mean and drew the distribution instead.

## What convergence measures, and why one number isn't enough

Our Multi-Channel Discord (MCD) platform runs a fleet of long-lived Claude agents, one subprocess per project. Each agent carries a **convergence score** from 0 to 1 — derived from how many of its recent reply turns actually advanced its goal over the last 24 hours. High convergence means the agent is making real progress toward what its `GOAL.md` asks; low convergence means it's spinning.

The mission-control dashboard already showed the fleet's *mean* convergence. Useful as a pulse, useless as a diagnosis. A mean collapses the entire shape of the fleet into a point, and the shape is where the decisions live. A fleet clustered tightly around 0.7 needs nothing. A fleet split into a converged camp and a stuck camp needs you to go find the stuck ones — *now* — and the mean of those two camps can sit at a perfectly reassuring 0.5.

So we built the **Convergence Distribution** view: a histogram of every agent's score, bucketed into ten bins, that makes the *shape* of the fleet the headline instead of its average.

## Ten bins and a colour with meaning

The whole computation is a bucket-and-count. Convergence is stored as an integer 0–100, so the bin is just integer division by ten, clamped so a perfect score lands in the top bin rather than overflowing:

```ts
const BIN_COUNT = 10

// top bin (≥0.9) captures score ≥ 90
function binIndex(score: number): number {
  return Math.min(BIN_COUNT - 1, Math.floor(score / 10))
}
```

Each bar is coloured by *position*, not height — a red-through-amber-to-green ramp walked across the ten bins as a hue sweep:

```ts
// hue 0 (red) → 120 (green) over the ten bins
function binColor(i: number): string {
  const hue = (i / (BIN_COUNT - 1)) * 120
  return `hsl(${hue}, 75%, 55%)`
}
```

This is a small choice that does a lot of work. Because colour encodes the *bin*, not the count, a tall bar on the left is unmistakably red — a big cluster of stuck agents reads as alarming even before you parse the axis. A tall green bar on the right reads as healthy. The eye gets the verdict from colour and the magnitude from height, two channels carrying two independent facts.

## Reading shape, not summary

The header still shows the mean — kept for continuity — but it now sits next to the number that actually matters operationally: how many agents are in the top bin, the `≥0.9` converged camp. That count turns green only when it's non-zero, so an all-stuck fleet doesn't get a falsely reassuring green digit.

The histogram itself is hand-drawn SVG: bars scaled to the busiest bin, the count printed above each bar in its own colour, and bin labels (`0.0–0.1`, `0.1–0.2`, …) along the axis. Hovering a bar lists the actual member projects, each a link straight to that project's focus view:

```ts
for (const p of projects) {
  if (p.convergenceScore == null) continue
  scores.push(p.convergenceScore)
  bins[binIndex(p.convergenceScore)].push(p.slug)
}
```

That hover-to-membership detail is what turns the chart from a status display into a worklist. You see a fat red bar at `0.0–0.1`, you hover it, and you get the exact slugs of the agents that are stuck — one click from intervening on each.

Agents without a score aren't counted; the view only describes the agents it has data for. And the bar heights scale against `Math.max(1, …)` the busiest bin, so an empty fleet doesn't divide by zero — it just draws a flat axis.

## The shapes that actually show up

A histogram earns its place by the shapes it exposes that an average erases:

- **A single tall bar on the right** — the fleet is broadly converged, clustered near done. Nothing to do. The mean would have told you this too, so on this shape the histogram is merely confirmation.
- **A bimodal split** — two humps, one near 0.0 and one near 0.9, with a hollow middle. This is the dangerous one the mean *hides*: the fleet isn't "average," it's two fleets, and one of them needs rescuing. The hover list hands you the casualties.
- **A left-skewed pile** — most agents bunched low. The fleet is broadly stuck, and the green `≥0.9` digit sits at zero. Time to question whether the goals themselves are achievable, not just the agents.

None of these required new instrumentation. The convergence scores were already computed and already served by the fleet endpoint. The histogram is a single client page reading `/api/fleet` — the same source every other mission-control view uses, refreshed every 60 seconds, so it never disagrees with them.

## Lessons worth stealing

**A mean is a hypothesis, not a fact.** The instant you have more than a handful of entities scored on the same scale, the average starts lying about the ones in the tails. A histogram costs one bucketing pass and turns "the fleet is at 0.5" into "the fleet is split, and here's who's on the wrong side."

**Encode the category in the colour, the magnitude in the size.** Mapping hue to bin position rather than bar height lets a glance carry two facts at once — *how bad* (colour) and *how many* (height). Reusing height for both would have wasted a channel.

**Make the chart a worklist.** A distribution that only shows counts is a status report. One that reveals *which* agents sit in the alarming bin, with a link to act on each, is a triage tool. The difference is a hover handler and a list of slugs.

The mean told us the fleet was "doing okay." The histogram told us half of it wasn't — and exactly which half.
