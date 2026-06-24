# Is Your Agent Thinking or Just Thrashing? Correlating Memory Churn Against Progress

*Heavy memory churn looks the same whether an agent is making a breakthrough or spinning in circles. The second axis is what tells them apart.*

An agent rewriting its memory furiously looks identical, from the outside, whether it's making a breakthrough or spinning in circles. Both produce a wall of diffs.

That ambiguity nagged at us across the fleet of autonomous Claude agents in Mission Control. We log every change an agent makes to its working memory, and we separately track a *convergence score* — how close each agent is to its goal. High memory churn was easy to see. What it *meant* was not. Heavy churn could be an agent productively reshaping its understanding on the way to a solution, or an agent thrashing — rewriting the same decisions over and over, going nowhere. Same signal, opposite conclusions.

So we put the two series on the same chart and asked the only question that disambiguates them: when this agent churns its memory, does its convergence go *up* or *down*?

## Two tables that never talked to each other

The raw material already existed in two separate places. `memory_diff_log` records every memory change with added and removed line counts. `convergence_history` records each agent's goal-convergence score over time. Neither knew about the other.

The join is a single database function that sums churn from one table and brackets convergence from the other, per project, over a 30-day window:

```sql
SELECT slug, SUM(added + removed) AS churn, COUNT(*) AS diffCount
  FROM memory_diff_log WHERE ts >= ? GROUP BY slug
```

```sql
SELECT slug, score FROM convergence_history WHERE date >= ? ORDER BY slug ASC, date ASC
```

Convergence is reduced to a *delta* — latest score minus earliest in the window — by walking the date-ordered rows and keeping the first and last per slug. Churn is the total lines added plus removed. One number for "how much did it rewrite," one for "did it get closer."

The function deliberately returns slugs that appear in only one table, padded with nulls and zeros. That's a small but important choice: rather than silently inner-joining and hiding partial data, it hands the caller everything and lets the route decide what's analysable. The route then requires both signals to be real — at least one memory diff *and* at least two convergence points, so start and end actually differ:

```ts
.filter((r) => r.diffCount > 0 && r.convPoints >= 2 && r.convStart != null && r.convEnd != null)
```

An agent with churn but no convergence history, or convergence but no memory writes, simply isn't on the chart. You can't correlate a pair when half of it is missing.

## The chart that answers the question

Each agent becomes a dot: memory churn on the x-axis, convergence delta on the y-axis, sized by how many separate diffs it made, colored by whether it improved, declined, or stayed flat. Two guide lines — a vertical one at the churn midpoint, a horizontal one at zero delta — split the plane into four quadrants, and the quadrants *are* the interpretation:

- **Top-right — heavy churn, rising convergence:** productive churn. The agent is rewriting a lot and getting closer. This is what good work looks like.
- **Bottom-right — heavy churn, falling convergence:** thrashing. Lots of rewriting, losing ground. This is the quadrant that matters.
- **Top-left — light churn, rising:** quiet progress.
- **Bottom-left — light churn, falling:** quietly stuck.

The whole point is that the dangerous quadrant — bottom-right — is invisible if you only watch churn. A thrashing agent and a productive one both sit far to the right. Only the vertical position tells them apart. The page counts the bottom-right population and puts it in the header in red:

```ts
const thrashing = pts.filter((p) => p.churn > xMid && p.convDelta < -0.001)
```

A non-zero thrashing count is a direct prompt to intervene: those agents are busy and going backwards.

## A correlation coefficient, with an honesty deadzone

Beyond the per-agent quadrants, there's a fleet-level question: across all agents, does more churn *tend* to go with more progress, or less? That's a Pearson correlation between churn and convergence delta, computed inline:

```ts
function pearson(pts: MemoryConvergencePoint[]): number | null {
  const n = pts.length
  if (n < 3) return null
  const mx = pts.reduce((s, p) => s + p.churn, 0) / n
  const my = pts.reduce((s, p) => s + p.convDelta, 0) / n
  let num = 0, dx = 0, dy = 0
  for (const p of pts) {
    num += (p.churn - mx) * (p.convDelta - my)
    dx += (p.churn - mx) ** 2
    dy += (p.convDelta - my) ** 2
  }
  if (dx === 0 || dy === 0) return null
  return Math.round((num / Math.sqrt(dx * dy)) * 100) / 100
}
```

Two guards keep this number honest. First, it returns `null` below three points — a correlation over two dots is meaningless, and `null` renders as "n/a" rather than a fake `1.0`. Second, the zero-variance check: if every agent has identical churn or identical delta, the denominator collapses and the coefficient is undefined, so it returns `null` instead of dividing by zero.

And then the interpretation refuses to over-read a weak signal. The raw coefficient is only labelled directional outside a ±0.3 dead zone:

```ts
const correlationSign =
  correlation == null ? 'n/a'
  : correlation > 0.3 ? 'positive'
  : correlation < -0.3 ? 'negative'
  : 'none'
```

A coefficient of 0.12 is not "weakly positive" — it's `none`. Calling a near-zero correlation "positive" is exactly the kind of statistical theatre that erodes trust in a dashboard. The dead zone makes the dashboard say "no real relationship" when there isn't one.

## What it changed

The feature reframed a number we'd been staring at for weeks. Memory churn had been a curiosity — interesting, ambiguous, unactionable. Plotted against convergence, it became a diagnosis. "beta churned 400 lines" turned into "beta churned 400 lines and lost ground — it's thrashing, go look," while another agent's equally heavy churn read as healthy because it was climbing.

Three things generalise beyond agent fleets:

- **A single metric is often ambiguous until you pair it with an outcome.** Churn alone couldn't distinguish progress from thrashing. The second axis — did it actually get closer? — is what made the first one mean something.
- **Pad the join, filter at the edge.** Returning slugs present in only one table, then filtering in the route, kept the data layer dumb and honest and put the "what's analysable" decision in one visible place.
- **Build the dead zone into the statistic, not the reader's head.** An `n < 3` null and a ±0.3 directional threshold stop the dashboard from dressing up noise as a finding. The honesty is in the code, so every viewer gets it for free.

An agent that's working hard and an agent that's stuck both generate a lot of diffs. The difference is never in how much they write — it's in whether the writing takes them anywhere. Put progress on the other axis, and the wall of diffs finally tells you which is which.

---

*Originally published at bistecglobal.com*

**BistecGlobal** builds AI-native enterprise software. Follow us for engineering insights.
