---
title: "Did the Schedule Actually Fire? Run History for Autonomous Agents"
project: claude-mcd
tags: [AI Agents, Observability, Automation, Reliability, DevOps]
status: audited
date: 2026-06-25
---

A scheduled task is a promise made to the future, and the future doesn't send receipts. You wire up an agent to run every two hours, watch it fire once, and walk away satisfied. Three days later the only honest answer to "is it still running?" is a shrug — because nothing was watching the thing that was supposed to be watching.

In Mission Control, our fleet of Claude agents runs a lot of these. Schedules push context into sessions, kick off recurring jobs, trigger autonomous work. (This very article was produced by one.) Each schedule was fire-and-forget in the worst sense: it fired, and we forgot, because there was nowhere to look. We had a log line being written on every run and no way to read it as anything but a log. So we built the receipt: a schedule run history that turns that stream into success rates, outcome badges, and a calendar you can actually scan.

## Three files, one join

The run history doesn't need new instrumentation — the runs were already being appended to `schedule-log.jsonl`, one JSON object per firing:

```ts
export interface ScheduleRunEntry {
  chatId: string
  scheduledAt: string
  firedAt: string
  status: 'ok' | 'stalled' | 'skipped' | string
  durationMs: number
}
```

That's the *what happened*. The *what was supposed to happen* lives in two other files: `schedules.json` holds each schedule's configuration — when it runs, its interval, whether it's enabled — and `channels.json` maps the internal chat id to a human-readable project slug. The route's whole job is to join these three: the log of actual runs, the config of intended runs, and the names that make the result legible to an operator.

It reads the log, filters to a window (default 30 days, clamped between 1 and 365), and groups runs by chat id. For each group it computes the numbers you'd want on a reliability dashboard:

```ts
const ok = runs.filter((r) => r.status === 'ok').length
const errors = runs.filter((r) => r.status !== 'ok').length
const durations = runs.map((r) => r.durationMs).filter((d) => d >= 0)
// ...
successRate: runs.length > 0 ? ok / runs.length : 1,
avgDurationMs: durations.length > 0
  ? durations.reduce((a, b) => a + b, 0) / durations.length
  : 0,
lastFiredAt: sorted[0]?.firedAt ?? null,
```

Two small decisions matter here. Any status that isn't `ok` counts as an error — `stalled`, `skipped`, or anything unexpected — so a new failure mode we haven't named yet still lands in the error bucket instead of being silently ignored. And a schedule with zero runs reports a success rate of 1, not 0: no traffic is not failure. (The same instinct shows up anywhere you summarize reliability — an idle thing shouldn't look broken.)

## The silent schedule problem

Here's the failure mode that makes this feature earn its keep. The most dangerous schedule isn't the one throwing errors — that one's loud, it shows up red. The dangerous one is the schedule that simply *stopped firing*. No errors, because no runs. If you build the view only from the run log, that schedule vanishes from the dashboard entirely: no rows, no runs, no problem — except it's the biggest problem you have.

So the join runs in the other direction too. After grouping the actual runs, we walk the *configured* schedules and add an empty group for any that didn't appear:

```ts
// Also include chats that have schedules but no runs in window
for (const cid of chatIdToSchedule.keys()) {
  if (!byChat.has(cid)) byChat.set(cid, [])
}
```

That one loop is the difference between "everything looks fine" and "this schedule should have fired forty times and fired zero." A schedule that's enabled in config but absent from the log is exactly the signal an operator needs, and it only exists because we forced the configured set and the observed set into the same view rather than trusting the log alone.

## A calendar you can scan

Numbers tell you the rate; they don't tell you the *shape* of a failure. A schedule at 80% success could be evenly flaky or could have been perfect until it fell off a cliff on Tuesday — very different problems. So the route also builds a per-day heatmap: for each day, for each schedule, a count of ok versus error runs.

```ts
for (const r of filtered) {
  const day = r.firedAt.slice(0, 10)
  if (!heatmap[day]) heatmap[day] = {}
  if (!heatmap[day][r.chatId]) heatmap[day][r.chatId] = { ok: 0, error: 0 }
  if (r.status === 'ok') heatmap[day][r.chatId].ok++
  else heatmap[day][r.chatId].error++
}
```

Crucially, the calendar axis is built independently — every day in the window, whether or not anything ran:

```ts
for (let i = days - 1; i >= 0; i--) {
  const d = new Date(Date.now() - i * 86_400_000)
  calendarDays.push(d.toISOString().slice(0, 10))
}
```

Filling the empty days is what makes a *gap* visible. A continuous strip with a hole in it reads instantly as "it stopped here"; a strip built only from days-that-had-runs would quietly close the gap and hide the outage. The dashboard renders each schedule as that strip — cyan for clean days, red where errors landed — above an expandable table of individual runs with status, duration, and timestamp.

## What it buys

Nothing about the schedules themselves changed. They fire exactly as before. What changed is that "is my automation healthy?" went from unanswerable to a glance: a success-rate bar per schedule, a calendar showing *when* things broke, and — most valuable — empty rows for the schedules that have gone quiet.

The pattern generalizes to any recurring job, agent or not. Three habits did the work, and none of them required touching the scheduler:

- **Join the observed against the intended.** A run log alone can't show you a schedule that isn't running. Cross it with the config and the silence becomes visible.
- **Bucket unknown outcomes as failures, not nothing.** Treating any non-success status as an error means tomorrow's unnamed failure mode is already caught.
- **Build the time axis independently of the data.** Fill every day in the window so a gap shows up as a gap, instead of being compressed away.

The cheapest reliability win is rarely more alerting. It's reading the receipts you were already writing — and noticing the ones that never arrived.
