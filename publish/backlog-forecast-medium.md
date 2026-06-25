# When Will the Agent Finish? A Burndown Forecast for an Autonomous Backlog

*Our AI agents pull their own work from a BACKLOG.md. Here's how we turned "I think it's making progress" into a pair of honest finish dates.*

We run AI agents that pull their own work. Each project carries a `BACKLOG.md` — a list of proposals the agent picks up, implements, merges, and checks off, mostly without a human in the loop. It's a satisfying setup right up until someone asks the obvious question: *when is it going to be done?*

For a human team you'd open a burndown chart. For an autonomous agent we had nothing — just a markdown file with some boxes ticked and some not. You could eyeball it and guess. So we built the thing that turns "I think it's making progress" into "at the current rate, cleared in about six weeks" — a backlog forecast that reads the same file the agent works from and projects the finish line.

## The backlog is already the data

The nice part about agents that track work in a flat file is that the file *is* the dataset. No separate tracking system to reconcile — the source of truth and the metrics source are the same document. The forecast route reads `BACKLOG.md` and parses it into proposals, splitting on section headers and pulling two facts from each: its status and when it was created.

```ts
for (const section of sections) {
  let status: 'done' | 'pending' | 'in_progress' = 'pending'
  let created: string | null = null

  for (const line of lines) {
    if (line.includes('[x] done')) { status = 'done'; break }
    if (line.includes('[~]') || line.includes('in_progress')) { status = 'in_progress' }
  }
  for (const line of lines) {
    const m = line.match(/\*\*Created:\*\*\s*(\d{4}-\d{2}-\d{2})/)
    if (m) { created = m[1]; break }
  }
  proposals.push({ status, created })
}
```

From that you get the headline counts for free — total, done, pending — but counts aren't a forecast. A forecast needs a *rate*.

## Velocity, two ways

The honest caveat up front: a markdown checkbox records *that* a proposal is done, not *when* it was finished. We don't have a completion timestamp, so we use the proposal's created date as a proxy for which week the work landed in, and bucket completed proposals by week from there. It's a rough placement, and we treat it as one — but across many proposals it's enough to establish a cadence.

With completed work grouped into weekly buckets, we compute velocity two ways on purpose:

```ts
// Recent reality: average of the last four weeks.
const recent4w = historical.filter((h) => h.weekStart >= fourWeeksAgo)
const velocity4w = recent4w.length > 0
  ? Math.round((recent4w.reduce((s, h) => s + h.completed, 0) / 4) * 10) / 10
  : 0

// Optimistic: the 75th-percentile week the agent has ever had.
const allVelocities = historical.map((h) => h.completed).sort((a, b) => a - b)
const velocityOptimistic = percentile(allVelocities, 75)
```

The four-week average is what the agent is *actually* doing lately — the number you should plan against. The p75 is a "good week" rate: not the absolute best, which would be a fluke, but the pace the agent hits when things are going well. Showing both gives an honest spread between the realistic finish and the if-it-keeps-rolling finish, instead of pretending a single number is destiny.

## Burning down to a date

Projection is deliberately simple: take the pending count, subtract a velocity each week, and walk forward until it hits zero — or until you've gone 52 weeks and clearly aren't converging.

```ts
function buildProjection(vel: number): string | null {
  if (vel <= 0) return null
  let rem = totalPending
  let week = todayWeek
  for (let i = 0; i < 52 && rem > 0; i++) {
    week = addWeeks(week, 1)
    rem = Math.max(0, rem - vel)
  }
  return rem <= 0 ? week : null   // the week it clears, or null if it never does
}
```

Two design choices fall out of this. The 52-week cap is a guard against false precision: an agent crawling at a near-zero rate would otherwise "finish" in the year 2090, a number worse than useless. Past a year we'd rather say nothing. And the function returns `null`, not a date, whenever velocity is zero or the backlog won't clear in the window. That `null` becomes an explicit **stalled** flag — `stalled = velocity4w === 0` — so a frozen agent reads as "no estimate," never as a confident far-future date that implies healthy progress.

Run both velocities through the same projection and you get two finish dates and a burndown line for each: a realistic estimate from recent pace, an optimistic one from good-week pace. The dashboard draws the remaining-work line sloping toward the axis, with the projected tail dashed, so "are we converging or flat-lining?" is answerable at a glance.

## What it actually tells you

The forecast didn't make any agent faster. What it changed is that autonomous progress became *legible* and *honest*. Before, "the agent is working through the backlog" was a vibe. Now it's a pair of dates with a visible gap between them, sitting on top of a line you can watch bend — or not. A flat burndown line and a `stalled` flag is an early warning that an agent has quietly stopped pulling work, long before anyone would've noticed the backlog wasn't shrinking.

The transferable lesson is about forecasting honesty more than about agents. Three habits did the heavy lifting, and they apply to any burndown you build:

- **Be explicit about proxy data.** We don't have a true completion timestamp, so we said so and used created-date as a stand-in — rather than dressing a proxy up as a measurement.
- **Show a range, not a point.** A realistic rate and an optimistic rate bracket the truth far better than one falsely confident date.
- **Refuse to forecast when you can't.** Capping the horizon and emitting a `stalled` flag instead of a year-2090 finish date keeps the output trustworthy. The most valuable thing a forecast can say is sometimes "no estimate" — and meaning it.

---

*Originally published at bistecglobal.com*

**BistecGlobal** builds AI-native enterprise software. Follow us for engineering insights.
