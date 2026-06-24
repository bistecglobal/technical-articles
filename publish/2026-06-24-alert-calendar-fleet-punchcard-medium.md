# When Do Your AI Agents Cry for Help? A Day-and-Hour Heatmap of Fleet Alerts

*We knew how often our agent fleet threw alerts. We had no idea when — so we rebuilt the GitHub commit punchcard for incidents.*

We knew *how often* our AI agent fleet threw alerts. We had no idea *when*.

The dashboard for our Multi-Channel Discord (MCD) platform — the control plane that runs a fleet of Claude agents, one subprocess per project — had a perfectly good alert log. Stalls, token-budget breaches, anomaly flags, limit hits: every incident landed in an `alert_events` table with a timestamp and a type. You could scroll the list. You could filter it. What you could not do was answer the one question that actually changes how you run an autonomous fleet: *does trouble cluster?*

Because if it does — if alerts pile up every weekday at 09:00 when the scheduler kicks off a batch of heartbeat injects, or every Sunday night when nobody is watching — then the fix isn't a better alert. It's a schedule change, a budget bump, or a guardrail timed to the danger window. A flat list buries that signal under chronology. We needed the oldest trick in operational analytics: a punchcard.

## The punchcard, reborn for agents

GitHub popularised the 7×24 commit punchcard a decade ago — seven rows for days of the week, twenty-four columns for hours, each cell shaded by activity. It works because human and machine behaviour is *periodic*. Things happen on weekly and daily cycles, and a heatmap makes those cycles impossible to miss.

Agent alerts are no different. So we built the **Alert Calendar**: a single page in MCD's mission control that renders every alert from the last 30 days as a 7×24 grid, each cell's intensity proportional to how many alerts fired in that day-of-week × hour-of-day slot.

The whole feature is two files — a SQL-backed API route and a client page — and most of the intelligence lives in one query.

## Let the database do the bucketing

The naive approach pulls every alert row into the app and buckets it in JavaScript. We pushed the bucketing into SQLite instead, where `strftime` already knows how to extract calendar fields from a Unix timestamp:

```sql
SELECT CAST(strftime('%w', ts, 'unixepoch', 'localtime') AS INTEGER) AS dow,
       CAST(strftime('%H', ts, 'unixepoch', 'localtime') AS INTEGER) AS hour,
       alert_type,
       COUNT(*) AS count
  FROM alert_events
 WHERE ts >= ?
 GROUP BY dow, hour, alert_type
```

`%w` gives day-of-week (0 = Sunday through 6 = Saturday); `%H` gives the 24-hour clock; `localtime` converts the stored epoch to the operator's timezone so the buckets line up with the working day they actually live in. Grouping by `alert_type` as well as the time slot means each cell carries not just a count but a breakdown — so a hover can tell you the 09:00 spike was *stalls*, not budget breaches.

The API route then folds those rows into a pre-seeded 7×24 grid, accumulating per-type counts and tracking two scalars as it goes — the maximum cell value (for colour scaling) and the busiest slot:

```ts
for (const c of cells) {
  if (c.dow < 0 || c.dow > 6 || c.hour < 0 || c.hour > 23) continue
  const cell = grid[c.dow][c.hour]
  cell.count += c.count
  cell.byType[c.alert_type] = (cell.byType[c.alert_type] ?? 0) + c.count
  total += c.count
}
```

A second pass walks all 168 cells to find `max` and the `busiest` window, then nulls out `busiest` if the fleet was silent — no point pointing at a 0-count "peak". The grid is always fully materialised (zero-filled), so the front end never has to reason about missing cells.

## The front end is almost boring — which is the point

The page maps the grid straight onto coloured squares. Intensity is `count / max`, fed into a transparent-to-red ramp:

```ts
const intensity = max > 0 ? count / max : 0
// ...
background: count === 0
  ? 'rgba(148,163,184,0.06)'           // empty: faint grey
  : `rgba(239,68,68,${0.15 + intensity * 0.85})`,  // active: red, scaled
```

Empty slots get a barely-there grey so the grid keeps its shape; active slots glow red, the busiest one near-solid. A header strip surfaces the two numbers that matter at a glance — the busiest window (`Mon 09:00 · 14`) and total alerts in range — and hovering any cell reveals its per-type breakdown. That's the entire interaction model. No drill-downs, no modals.

```
        00  03  06  09  12  15  18  21
  Sun    ·   ·   ·   ·   ▪   ·   ·   ▪
  Mon    ·   ·   ·  ███  ▪   ▪   ·   ·
  Tue    ·   ·   ·  ██   ▪   ·   ·   ·
  Wed    ·   ·   ·  ██   ·   ▪   ·   ·
  Thu    ·   ·   ·  ██   ▪   ·   ·   ·
  Fri    ·   ·   ·  ▪    ·   ·   ·   ·
  Sat    ▪   ·   ·   ·   ·   ·   ·   ·
                    ↑ weekday-morning cluster
```

The whole page refreshes every 60 seconds off the same `/api/alert-calendar` endpoint, sharing MCD's standard freshness badge so an operator can tell stale data from live.

## What the grid actually tells you

The value of a punchcard isn't the chart — it's the *shape* it exposes. A few patterns we now read instantly that the old list hid:

- **A vertical stripe at one hour** means a scheduled job is the culprit. Alerts that cluster at a single clock time, regardless of day, almost always trace back to a cron-style trigger — a batch of injects, a budget-rollover check — not the agents themselves.
- **A solid block across weekday mornings** is load-shaped: the fleet degrades when the most projects are active at once. That's a capacity signal, not a bug.
- **A lonely weekend cell** is the one you'd never catch by watching live — trouble that fires when no human is in the loop, which is exactly the case autonomous fleets exist to handle and exactly when you most want a guardrail.

None of these require a new alert type or more instrumentation. They fall out of timestamps we were already storing, reorganised along the two axes that human operations actually run on.

## Lessons worth stealing

**Push periodicity into the store.** The temptation with time-series data is to stream it to the client and bucket there. But `strftime` with `localtime` does the calendar maths correctly — including the timezone conversion that trips up hand-rolled date logic — in a single grouped query. Less data over the wire, no off-by-one-hour bugs.

**Pre-materialise the grid.** Returning a fully zero-filled 7×24 structure means the UI is pure rendering. Sparse server responses push gap-handling into view code, where it multiplies.

**A 30-year-old visualisation still earns its place.** We didn't invent anything here. We took the commit-punchcard idiom and pointed it at agent alerts. The reason it works is the same reason it worked for commits: the underlying process is cyclical, and a heatmap is the most honest way to show a cycle. When you're operating a fleet that runs around the clock without you, knowing *when* it struggles is often more actionable than knowing *what* went wrong.

The list told us our agents had problems. The calendar told us when to be ready for them.

---

*Originally published at bistecglobal.com*

**BistecGlobal** builds AI-native enterprise software. Follow us for engineering insights.
