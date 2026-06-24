---
title: "Does a Bigger Memory Make a Smarter Agent? A Diagnostic Quadrant for an AI Fleet"
project: claude-mcd
tags: [AI, Observability, Data Visualization, Agents, Developer Tooling]
status: audited
date: 2026-06-24
---

There's a comfortable assumption baked into how we build long-running AI agents: more memory is better. Give the agent a bigger store of accumulated facts and it should make more progress toward its goal. It's intuitive enough that nobody checks it. But across a fleet of autonomous Claude agents, the assumption is testable — and the moment you plot it, you find agents sitting on a fat pile of memory that are going nowhere. Those are the ones worth finding, because a heavy memory that isn't buying progress isn't an asset. It's dead weight to prune.

That's the question Mission Control's `/memory-convergence` page exists to answer, in one glance: *does memory footprint actually buy goal progress?* It plots every agent as a single bubble on two axes and lets the shape of the cloud tell you whether the comfortable assumption holds for your fleet right now.

## Two axes, four quadrants

The chart is a scatter plot with deliberately chosen axes. The x-axis is each project's memory size; the y-axis is its `convergenceScore` — a 0-to-1 measure of how close the agent is to its goal. Colour encodes goal status (active, completed, paused, none). Two dashed guide lines — one at the midpoint of the memory range, one at 50% convergence — carve the plane into four quadrants, each with a plain-language label:

- **lean & converging** (low memory, high progress) — efficient agents
- **heavy & converging** (high memory, high progress) — earning their footprint
- **lean & stalled** (low memory, low progress) — stuck, but not from bloat
- **heavy & stalled** (high memory, low progress) — **prune candidates**

That bottom-right quadrant is the whole point. An agent with a large memory and low convergence is carrying context that isn't translating into progress — exactly the population you'd want to inspect, trim, or restart. The page counts them and surfaces the number in the header, coloured red when it's non-zero, so the triage signal is visible before you even read the plot.

## Why the x-axis is logarithmic

Memory sizes across a fleet don't spread evenly — one agent might hold a few kilobytes while another holds several megabytes. On a linear axis the small ones collapse into a smear against the left edge and all the resolution goes to one outlier. So the x-axis is log-scaled:

```ts
.map((p) => ({
  slug: p.slug,
  bytes: p.memoryStatus!.sizeBytes,
  logBytes: Math.log10(Math.max(1, p.memoryStatus!.sizeBytes)),
  conv: Math.max(0, Math.min(1, p.convergenceScore!)),
  goalStatus: p.goalStatus ?? 'none',
  goalText: p.goalText ?? '',
}))
```

`log10` of the byte count turns a multiplicative spread into an additive one, so a 10× difference is a constant distance regardless of where you are on the axis. The quadrant divider then sits at the *midpoint of the log range* — splitting the fleet by order of magnitude rather than by raw bytes, which is the honest way to say "heavy" when sizes span decades. Note the guards: `Math.max(1, …)` keeps `log10` defined for an empty store, and convergence is clamped to `[0, 1]` so a stray score can't fling a bubble off the canvas.

## A correlation hint that refuses to oversell

The tempting next move is to draw a trend line and declare "memory drives convergence." The page deliberately doesn't. Instead it computes a coarse Pearson correlation between log-memory and convergence and reports it as a *hint*, in words:

```ts
const r = num / Math.sqrt(dx * dy)
if (r > 0.3) return `positive (${r.toFixed(2)})`
if (r < -0.3) return `negative (${r.toFixed(2)})`
return `weak/none (${r.toFixed(2)})`
```

Two things make this honest rather than misleading. First, it returns `'n/a'` when there are fewer than three points — you can't claim a correlation from two agents, so it doesn't try. Second, it buckets the result into *positive / negative / weak-or-none* with a ±0.3 dead zone, rather than printing a precise-looking coefficient that invites over-reading. A fleet-wide `r` of 0.12 is noise; calling it "weak/none (0.12)" says exactly that. The hint answers "is there even a relationship worth thinking about?" and stops there — it never pretends to establish that memory *causes* progress.

## Built on what was already there

The page adds no backend. It reads the existing `/api/fleet` endpoint — the same fleet snapshot every other Mission Control view consumes — and derives everything client-side: the points, the axis domains, the quadrant split, the prune list, the correlation. It only plots projects that have *both* a memory footprint and a convergence score, silently skipping the rest, so a missing signal never produces a misleading bubble at the origin. A bubble's tooltip shows the real byte size and convergence percent; the points deep-link into the per-agent focus view for the actual inspection. The plot refreshes on the same 60-second cadence as the rest of the dashboard.

That reuse is a recurring discipline in this dashboard: a new analytical view should be a new *lens* on the canonical fleet data, not a new data pipeline that can drift out of agreement with the others.

## Lessons

- **Plot the assumption you've never checked.** "More memory → more progress" sounds obvious enough that nobody verifies it. A two-axis scatter turns a belief into something you can see — and the disconfirming cases (heavy but stalled) are the actionable ones.
- **Log-scale spans that cross orders of magnitude.** When values range from kilobytes to megabytes, a linear axis wastes itself on the outlier. `log10` restores resolution and makes "heavy" mean "an order of magnitude more," which is what you actually mean.
- **Report correlation as a hint, not a verdict.** A dead zone around zero, a word instead of a bare number, and an explicit `n/a` below the sample floor keep a quick statistic from masquerading as a finding. Show the relationship's strength; don't imply causation.
- **A new view, not a new pipeline.** Deriving everything from the shared fleet endpoint means this chart can never silently disagree with the others. Add lenses, not sources of truth.

The quadrant won't tell you *why* an agent is heavy and stalled. But it will tell you, in one red number, how many of them there are — and that's the difference between suspecting your fleet is carrying dead weight and knowing which agents to open first.
