---
title: "When Will This Agent Actually Finish? Fitting a Trend Line to Goal Convergence"
project: claude-mcd
tags: [AI, Observability, Forecasting, Statistics, Autonomous Agents]
status: draft
date: 2026-06-24
---

A convergence score of 62 tells you where an agent is. It tells you nothing about whether it's getting there.

That's the gap we kept hitting in Mission Control, the dashboard watching our fleet of autonomous Claude agents. Each agent carries a *convergence score* — a 0-to-100 read on how close it is to its stated goal. We had the number, we had it over time, and we still couldn't answer the operator's actual question: *is this one going to make it, and roughly when?* A score sitting at 62 could be an agent three days from done, or one that flatlined a week ago and will sit at 62 forever. The snapshot can't tell them apart. The slope can.

So we built a forecast: fit a trend line to each agent's convergence history and project the day it crosses "healthy." Here's the whole thing, including the parts where naive math would have lied to operators.

## The data was already there — we just weren't reading it forward

Mission Control already records a `convergence_history` row per project over time. Every prior view of it was backward-looking: a sparkline, a distribution histogram, a day-over-day movers list. All of them answer "what happened." None answer "what's about to."

The forecast reuses that exact series — no new table, no new collector. It reads each project's last 30 days of convergence points and does one thing the other views don't: extrapolates.

## Least-squares, by hand, on a tiny series

The core is an ordinary least-squares fit of score against day-index. It's a dozen lines, and writing it out (rather than reaching for a stats library) keeps the whole forecast auditable:

```ts
// Least-squares slope/intercept of score over day-index (0..n-1).
function linearFit(scores: number[]): { slope: number; intercept: number } {
  const n = scores.length
  const mx = (n - 1) / 2
  const my = scores.reduce((s, v) => s + v, 0) / n
  let num = 0, den = 0
  for (let i = 0; i < n; i++) {
    num += (i - mx) * (scores[i] - my)
    den += (i - mx) ** 2
  }
  const slope = den === 0 ? 0 : num / den
  return { slope, intercept: my - slope * mx }
}
```

`slope` is the score gained per day. That single number is the forecast's engine: positive and steep means closing fast; near zero means stuck; negative means losing ground. From it, the ETA falls out as simple arithmetic — how far is the current score from the target of 90, divided by how fast it's moving:

```ts
etaDays = Math.ceil((TARGET - current) / slope)
```

## Where naive forecasting lies — and the three guards against it

A trend-line forecast is trivially easy to make *wrong* in ways that erode operator trust. Three decisions in the route exist purely to keep it honest.

**Guard one: don't forecast from noise.** A slope fit over two points is meaningless. The route requires at least three convergence points before a project is forecast at all (`MIN_POINTS = 3`); everything below that is silently dropped rather than shown as a confident-looking ETA built on nothing.

**Guard two: a deadband around zero.** Real convergence series wobble. An agent that's genuinely stalled will still post a slope of +0.01/day from noise, and dividing the distance-to-target by that microscopic slope yields an ETA of *thousands of days* — technically a number, operationally a lie. So the status classifier ignores any slope inside ±0.05/day:

```ts
if (current >= TARGET) {
  status = 'reached'
} else if (slopeR > 0.05) {
  status = 'rising'
  etaDays = Math.ceil((TARGET - current) / slope)
} else if (slopeR < -0.05) {
  status = 'declining'
} else {
  status = 'stalled'   // inside the deadband — no ETA
}
```

An ETA is emitted *only* for genuinely rising agents. Stalled and declining agents get a status and no number, because the honest answer to "when will it finish?" for a flatlined agent is "not on this trajectory" — not a fabricated date.

**Guard three: rank by what an operator should look at first.** The list sorts soonest-ETA first, then rising-without-ETA, then stalled, then declining, with already-reached agents last. The operator's eye lands on the agent about to cross the line, and the dead weight sinks to the bottom:

```ts
const rank = { rising: 0, stalled: 1, declining: 2, reached: 3 }
projects.sort((a, b) => {
  if (a.etaDays != null && b.etaDays != null) return a.etaDays - b.etaDays
  if (a.etaDays != null) return -1
  if (b.etaDays != null) return 1
  return rank[a.status] - rank[b.status]
})
```

The response also rolls up a single headline number: how many agents are forecast to reach the target *within the 30-day window*. That's the fleet-level "how many will land soon" in one integer.

## Drawing the future as a dashed line

The forecast earns its keep visually. Each row is a per-agent sparkline of the historical scores, with a dashed target line across it — and, for rising agents only, a dashed extension from the latest point projected forward to where it meets the target:

```tsx
{/* forecast extension to target */}
{p.etaDays != null && (
  <line x1={lastX} y1={lastY} x2={SW} y2={targetY}
        stroke={color} strokeWidth={1} strokeDasharray="3 2" opacity={0.7} />
)}
```

Solid line is what happened; dashed line is the projection. A slope arrow (↗ / → / ↘), the current score, and a status chip (`~7d`, `stalled`, `declining`, `✓ reached`) sit alongside. An operator scanning the page sees, per agent, both the trajectory and the verdict — and the dashed segment makes the *basis* for the ETA visible, not just the conclusion.

## What it changed

The forecast turned a static health number into a question an operator can actually act on. "alpha is at 62" became "alpha is rising, ~7 days out" or "alpha stalled — it's been flat for a week, go look." That second reading is the one that prompts intervention, and it was invisible in every backward-looking view we'd built before.

Three things generalise from this beyond agent fleets:

- **Forecasting is often a reading you already have, extrapolated.** We added no new data — the convergence series existed. The value was entirely in projecting it forward instead of only plotting it backward.
- **A deadband is the difference between a forecast and a fabrication.** Dividing a distance by a near-zero, noise-driven slope produces enormous, confident, wrong numbers. Refusing to forecast inside ±0.05/day — and showing "stalled" instead — is what keeps the feature trustworthy.
- **Make the projection visible, not just the conclusion.** The dashed line to the target line lets an operator sanity-check the ETA at a glance. A bare number ("7 days") invites blind trust; a drawn trajectory invites judgement.

A score says where an agent is. A slope says where it's going. For anything that's supposed to make progress on its own, the second question is the one worth answering.
