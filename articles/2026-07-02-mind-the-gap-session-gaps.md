---
title: "Mind the Gap: Telling a Stalled Agent From a Sleeping One"
project: multi-channel-discord
tags: [AI, DevOps, Observability, Agents, Monitoring]
status: draft
date: 2026-07-02
---

A dashboard tile that says an agent has been quiet for 40 hours tells you almost nothing. Is it stalled — wedged mid-task, waiting on a reply that will never come? Or is it simply asleep, a low-traffic project that only wakes when someone pings it? Those two states look identical from the outside, and treating them the same is how you either drown in false alarms or miss a genuinely stuck agent for days.

Mission Control, the observability layer for our multi-channel Discord agent fleet, now draws the distinction visually. The **Session Gap Analysis** view renders each project as a swimlane: green where the agent was genuinely active, amber where it went silent for more than a day, red where it crossed three. One glance separates the marathon runners from the patients who flatlined.

## The problem with "last seen"

Every fleet dashboard has a "last active" column. It is the single least useful number we track, because it collapses two different questions into one:

1. *How long since this agent did anything?* (recency)
2. *Is this silence normal for this agent?* (context)

A project that fires once a week and just went 40 hours quiet is fine. A project that normally responds every few minutes and went 40 hours quiet is on fire. A raw timestamp can't tell them apart. What you actually want is the **shape** of the silence over time — not one number, but the full rhythm of active and idle stretches.

## Counting silence honestly

The naive way to measure a gap is to diff transcript timestamps. That breaks immediately, because a Claude Code transcript is full of entries that aren't real activity: tool results streaming back, injected system reminders, heartbeat prompts. If you count those, a wedged agent whose watchdog keeps poking it looks perfectly healthy — the transcript never goes quiet even though nothing is happening.

So the gap detector only counts *genuine* inbound messages. Reading each project's JSONL transcripts, it keeps a timestamp only when the entry is a real user turn, and filters out the noise:

```typescript
if (e.type !== 'user') continue
const ts = e.timestamp
if (!ts || ts < cutoff) continue
const content = e.message?.content
// Skip tool_result messages (not genuine user input)
if (Array.isArray(content) && content.length > 0) {
  const first = content[0] as Record<string, unknown>
  if (first.type === 'tool_result') continue
}
// Skip meta/system messages
if (typeof content === 'string' && content.startsWith('<system-reminder')) continue
timestamps.push(ts)
```

Those two `continue` guards are the whole trick. A `tool_result` is the fleet talking to itself; a `<system-reminder` is the harness injecting context. Neither is a human or a peer agent breaking the silence, so neither resets the clock. What survives is the true pulse of the conversation.

## From timestamps to swimlanes

With a clean, sorted list of genuine message times per project, the gaps fall out with a single pass. Two thresholds classify severity — 24 hours turns a gap amber, 72 hours turns it red — and one detail matters more than it looks:

```typescript
// Include synthetic gap from last message to now
const points = [...sortedTs.map(t => Date.parse(t)), nowMs]
const starts = [cutoffMs, ...sortedTs.map(t => Date.parse(t))]

for (let i = 0; i < starts.length; i++) {
  const gapMs = points[i] - starts[i]
  if (gapMs >= GAP_YELLOW_MS) {
    gaps.push({
      start: new Date(starts[i]).toISOString(),
      end: new Date(points[i]).toISOString(),
      durationHours: Math.round((gapMs / 3600_000) * 10) / 10,
      severity: gapMs >= GAP_RED_MS ? 'red' : 'yellow',
    })
  }
}
```

The `nowMs` appended to `points` is the *synthetic gap* — the stretch from the last real message to right now. Without it, an agent that stopped responding two days ago would show a clean transcript with no closing gap, because the silence hasn't been "ended" by a new message. That trailing gap is exactly the one an operator cares about most: the one that's still open.

The API then reports, per project, its longest gap, its current (open) gap, its last genuine message, and message count — sorted so the most-idle projects float to the top:

```typescript
projects.sort((a, b) => b.currentGapHours - a.currentGapHours)
```

The front end paints each project's timeline as a horizontal bar. Green segments are active windows, amber and red are the classified gaps, and a slate-grey segment marks a window with no activity at all. A 7/14/30-day toggle changes the zoom. The result reads like a heart monitor for the fleet: you scan down the left edge for the reds, and the swimlane to their right tells you instantly whether that red is a first-time flatline or the tail of a chronically quiet project.

## What the view actually surfaces

The payoff is triage speed. Before, "agent X is idle" was the start of an investigation — open the transcript, scroll, figure out whether the quiet was expected. Now the swimlane answers the follow-up question before you ask it:

- A bar that's **green then abruptly red at the right edge** is a healthy agent that just stalled. Investigate now.
- A bar that's **amber-striped throughout** is a low-traffic project doing what it always does. Leave it.
- A bar that's **all slate** never woke up in the window. Different problem entirely — probably never got a task.

The severity thresholds are deliberately coarse. We resisted the urge to make them configurable per project, because the point of the view is a fast fleet-wide read, not a precise SLA. A day of silence is worth a glance; three days is worth a click. Finer tuning belongs in the alerting layer, not the eyeball layer.

## Lessons

Two things carried over from building this that apply to any agent-observability work.

**Filter for intent, not for volume.** The temptation with transcripts is to treat every line as a signal. But an autonomous fleet generates enormous self-talk — tool calls, results, injected context — and if your "activity" metric counts that, it measures the harness, not the work. Deciding *what counts as a real event* was 90% of the value here, and it was a handful of `continue` statements.

**Always close the open interval.** Time-series views that only plot recorded events quietly lie about the present, because the most important interval — from the last event to *now* — has no closing data point. Appending `now` as a synthetic boundary is a one-line fix that turns a historical chart into a live one. It's the difference between a dashboard that shows you what happened and one that shows you what's happening.

A stalled agent and a sleeping one are both quiet. The job of the tooling is to make sure you never confuse the two — and the cheapest way to do that is to stop counting the noise and start drawing the silence.
