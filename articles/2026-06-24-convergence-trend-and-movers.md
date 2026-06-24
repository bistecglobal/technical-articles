---
title: "The Trend Line and the Suspects: Two Views That Make AI Fleet Progress Accountable"
project: claude-mcd
tags: [AI, DevOps, Observability, Agents, DataViz]
status: audited
date: 2026-06-24
---

A fleet-wide trend line is great at telling you the average is sliding and useless at telling you *who* dragged it down. A per-agent leaderboard is great at naming names and useless at telling you whether today was normal or a crisis. You need both, and you need them to agree — because the moment a trend dips, the only useful next question is "which agents, and by how much?"

Our Multi-Channel Discord (MCD) platform runs a fleet of long-lived Claude agents, one subprocess per project, each scored daily on **convergence** — how much its recent work actually advanced its goal. We were already storing that score every day in a `convergence_history` table, one row per project per date. From that single table we built two views that answer two different questions, and the trick is that they're wired to the same source so they can never tell contradicting stories.

## View one: is the whole fleet improving?

The **Convergence Trend** is a 14-day line of the fleet's mean convergence. Its job is the macro question — is the fleet, in aggregate, getting better at its goals or drifting away from them? The aggregation is a single grouped query, returned oldest-to-newest so it charts directly:

```sql
SELECT date,
       AVG(score)                                   AS meanScore,
       SUM(CASE WHEN score >= 90 THEN 1 ELSE 0 END) AS topBinCount,
       COUNT(*)                                      AS projectCount
  FROM convergence_history
 GROUP BY date
 ORDER BY date DESC
 LIMIT 14
```

Two series come out of that. The mean is the line itself. But a mean can drift for boring reasons — one agent's bad day, a project added or removed — so alongside it the query counts how many agents sat in the *top bin* (score ≥ 90) each day. Those converged-agent counts are drawn as faint bars *under* the mean line, so you read the average and the "how many actually finished" count in one glance. A mean that holds steady while the top-bin count quietly drains is a fleet coasting on a few strong performers — exactly the kind of erosion a single line would hide.

The header reduces the fortnight to one number: the change from the first day in the window to the last (`last.meanScore − first.meanScore`). Up is good, down is a question. The line gives you the shape; the delta gives you the verdict.

## View two: who moved, and by how much?

The trend tells you *that* the fleet shifted. It cannot tell you *who*. That's the **Convergence Movers** view — a day-over-day leaderboard of change. For each agent it takes the latest score as `curr`, the prior distinct day's score as `prev`, and the difference as the delta:

```ts
const curr = rows[i].score
const prev = (rows[i + 1]?.slug === slug) ? rows[i + 1].score : null
out.push({ slug, prev, curr, delta: prev == null ? null : curr - prev })
```

Agents are split into three columns. **Climbers** (positive delta) are sorted biggest-gain first; **Fallers** (negative delta) biggest-drop first; and agents with only a single day of history land in a **New** bucket — they have a score but no yesterday to compare against, so forcing a delta would be a lie. Each row shows `prev → curr` and a sparkbar whose length is the magnitude of the change, clamped:

```ts
const mag = Math.min(1, Math.abs(delta) / 100)
```

That `New` bucket is the detail that keeps the view honest. A naive implementation treats a missing `prev` as zero and reports a brand-new agent at score 60 as a `+60` superstar climber. It isn't moving; it just arrived. Separating "no comparison possible" from "no change" is the difference between a leaderboard you trust and one that invents heroes.

## The number that ties them together

Both views are bound to the same `convergence_history` data, and the Movers view surfaces one figure that reconciles them: the **net delta**, the sum of every agent's day-over-day change.

```ts
const netDelta = movers.reduce((sum, m) => sum + (m.delta ?? 0), 0)
```

Green-up, red-down, it's the single-day analogue of the trend line's slope. When the trend dips, the net delta tells you whether it was broad (everyone down a little) or concentrated (one agent fell off a cliff), and the Fallers column hands you the suspect. The macro view raises the alarm; the micro view assigns the blame. Same table, two altitudes.

## Why two views beat one dashboard

It's tempting to cram both into a single screen. We deliberately didn't. The trend answers a *strategic* question you ask weekly — "is our fleet getting better?" — and rewards a calm, zoomed-out line. Movers answers an *operational* question you ask daily — "what changed since yesterday, and who do I poke?" — and rewards a ranked, scannable list. Different cadence, different shape, different page.

What they share is the wiring. Neither introduced a new data pipeline. The daily convergence score was already being written; the trend view aggregates it across projects by date, and the movers view diffs it across dates by project. Two questions, two axes of the same table, zero new instrumentation.

## Lessons worth stealing

**Pair the aggregate with the attribution.** A trend line that only goes down is an anxiety generator. Ship it next to the per-entity movers that explain it, and a dip becomes a worklist instead of a worry. The aggregate raises the question; the breakdown answers it.

**Never fake a baseline.** The most common bug in any day-over-day metric is treating "no prior data" as "zero." It manufactures phantom swings and erodes trust in the whole view. A dedicated bucket for first-appearance entities costs a null check and buys credibility.

**Encode the same metric at two altitudes, not two metrics.** The trend and the movers aren't different measurements — they're the *same* convergence score grouped two ways. Because they read one table, they can never disagree, and an operator can move from "the fleet slipped this week" to "this agent slipped today" without ever wondering if the two screens are measuring the same thing.

The line tells you the fleet moved. The leaderboard tells you who moved it. Together, they turn "convergence is down" from a shrug into an address.
