---
title: "When Does the Fleet Run Out? Forecasting Token Burn and Backlog Velocity in Mission Control"
project: claude-mcd
tags: [AI, DevOps, Observability, Forecasting, Developer Tooling]
status: audited
date: 2026-06-23
---

A token budget alert that fires at 100% is a smoke detector that goes off after the house has burned down. By the time a project trips its monthly ceiling, the work has already stopped — the agent is queued, the sprint is stalled, and someone is scrambling to bump a number in a config file. The useful question was never "are we over budget?" It was "*when* will we be over budget?" — and that question has an answer days before the alert.

Mission Control, the dashboard for our multi-channel Claude fleet, already tracked spend and stall risk after the fact. What it lacked was a forward look: a straight line from today's consumption to the day the tank runs dry. We added two — one for tokens, one for backlog throughput — and they share a design principle worth unpacking: a forecast is only trustworthy if you can see the assumptions behind it.

## The problem with a single number

Our fleet runs one Claude subprocess per project, each with an optional `monthlyTokenBudget`. The existing budget machinery was reactive: cross 50%, 80%, 100%, get an alert. Useful, but it answers "where are we now," not "where are we headed." Two projects both sitting at 60% are not equally safe — one might be coasting, the other accelerating into a wall it'll hit before month-end.

The fix is a *rate*. If you know how fast tokens are being spent and how much budget remains, exhaustion is just division. The trick is computing a rate that reflects recent behaviour without overreacting to a single busy afternoon.

## Burn rate: division, done carefully

The `/api/metrics/burn-rate` endpoint reads each project's Claude Code JSONL transcripts directly — the same source of truth Mission Control uses everywhere else — and walks every `assistant` record, summing `input_tokens + output_tokens`. Two accumulators come out of one pass: month-to-date tokens (anything timestamped since the UTC first-of-month) and seven daily buckets for the trailing week.

```ts
const last7Total = spark.reduce((s, v) => s + v, 0)
const dailyRate = Math.round(last7Total / 7)
...
const projectedMonthEnd = monthTokens + dailyRate * daysRemaining
const daysUntilExhausted = budget > 0 && dailyRate > 0
  ? Math.max(0, Math.round((budget - monthTokens) / dailyRate))
  : null
```

A seven-day window is the deliberate choice here. A single day is too noisy — one heavy refactor session would scream "exhaustion imminent." A 30-day average is too sluggish to catch a project that just ramped up. Seven days smooths the spikes while still tracking a real shift in pace within a week.

Note the `null`. When a project has no budget set, or its rate is zero, `daysUntilExhausted` is deliberately *not* a number — not `Infinity`, not `999`. An unknown is rendered as unknown rather than disguised as a confident "you're fine." That honesty propagates to the UI: the `/burn-rate` page shows a sparkline of the week's daily totals, a budget bar, and an exhaust badge — but only where the data supports one.

Sorting closes the loop. Projects are ordered by `daysUntilExhausted` ascending, so whatever is closest to running dry sits at the top of the table, with the fleet total pinned above it. The most urgent thing is always the first thing you see.

## Velocity: the same idea, applied to work

Tokens are one resource; engineering throughput is another. The `/api/metrics/forecast` endpoint asks the parallel question for the backlog: given how fast proposals are being completed, when does the queue clear?

Rather than re-derive history, it builds on the existing burndown series — completed-vs-remaining proposals over time — and computes a spread of seven-day completion rates across the whole series:

```ts
function sevenDayRates(series: BurndownPoint[]): number[] {
  const rates: number[] = []
  for (let i = 7; i < series.length; i++) {
    const delta = series[i].done - series[i - 7].done
    rates.push(delta / 7)
  }
  return rates
}
```

From that distribution come three scenarios. **Optimistic** uses the fastest week the fleet ever sustained; **pessimistic** uses the slowest non-zero week; **expected** uses the trailing 14-day mean. Each rate projects a completion date — and crucially, when a rate is zero or non-positive, the projected date is `null` rather than a fabricated point at infinity.

The payoff is a fan chart on the `/forecast` page. The optimistic and pessimistic rays form the edges of a shaded band, and the band visibly *widens* the further out you look. That widening is the point. A forecast that draws a single confident line into the future is lying; a forecast that shows its uncertainty as a spreading cone is telling the truth about what it does and doesn't know.

## What reusing the pipeline bought us

Neither endpoint invented its own data layer. Burn rate reads the same transcripts the rest of the dashboard parses; the forecast calls the burndown route's `GET` directly and consumes its JSON. That discipline has a concrete consequence: the forecast can never disagree with the burndown chart it sits next to, because they are computed from the identical series. There is no second source of truth to drift out of sync.

It also kept the surface area small. The forecast route is a pure transform over an existing response — a handful of array operations and a date projection — with no filesystem access of its own. The hardest part was not the math; it was deciding what *not* to claim.

## Lessons

A few principles generalise well beyond a token dashboard:

- **Forecast rates, not levels.** A gauge tells you where you are. Multiplying a recent rate by the time remaining tells you where you'll be — which is the only number that lets you act before the wall, not at it.
- **Pick a smoothing window on purpose.** Seven days was chosen to be responsive enough to catch a real change in pace yet stable enough to ignore a single anomalous session. The right window is a judgement call; make it deliberately, not by accident.
- **Render uncertainty honestly.** A `null` projected date and a widening fan band are not gaps in the feature — they *are* the feature. A forecast that hides what it doesn't know is worse than no forecast, because people trust it.
- **One source of truth survives contact with a second feature.** By layering both forecasts on existing endpoints, we guaranteed they agree with the charts beside them. Reuse here wasn't about saving code; it was about preventing contradiction.

The smoke detector still works — alerts still fire at 50, 80, and 100 percent. But now the question that matters most has an answer the moment you open the dashboard: not *are we over*, but *how many days until we are* — and how sure we are about it.
