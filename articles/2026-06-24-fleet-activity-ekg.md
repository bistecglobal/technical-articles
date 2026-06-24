---
title: "An EKG for an AI Agent Fleet: Five Event Streams on One Waveform"
project: claude-mcd
tags: [AI, DevOps, Observability, Agents, DataViz]
status: draft
date: 2026-06-24
---

A hospital monitor doesn't show you one number. It shows you a *waveform* — and a clinician reads the shape, the rhythm, and the flatlines before they ever read a digit. We wanted that for our agent fleet: not "how many alerts today," but the living pulse of everything happening across it, where a dead patch or a sudden spike jumps out before you've parsed a single label.

Our Multi-Channel Discord (MCD) platform runs a fleet of long-lived Claude agents, one subprocess per project. Activity lands in five separate places: alerts fire, context gets injected, agents write to memory, digests get generated, broadcasts go out. Each had its own log, its own page, its own way of being checked. None of them told you whether the *fleet as a whole* was busy, idle, or quietly flatlined. So we built the **Fleet Activity EKG** — a 48-hour, multi-lane waveform that puts all five pulses on one screen.

## Five sources, five tables, one query

The first problem is that the five event types don't live together. Alerts and injects share a table but are distinguished by type; memory writes, digests, and broadcasts each have their own. A single query function pulls just the timestamps from each, since timestamps are all the waveform needs:

```ts
export function getEkgTimestamps(sinceTs: number): EkgTimestamps {
  const col = (sql: string, ...p: unknown[]): number[] =>
    (db.prepare(sql).all(...p) as { ts: number }[]).map((r) => r.ts)
  return {
    alerts:     col(`SELECT ts FROM alert_events WHERE ts >= ? AND alert_type != 'inject'`, sinceTs),
    injects:    col(`SELECT ts FROM alert_events WHERE ts >= ? AND alert_type =  'inject'`, sinceTs),
    memory:     col(`SELECT ts FROM memory_diff_log WHERE ts >= ?`, sinceTs),
    digests:    col(`SELECT ts FROM digest_log WHERE ts >= ?`, sinceTs),
    broadcasts: col(`SELECT CAST(strftime('%s', ts) AS INTEGER) AS ts FROM broadcasts
                      WHERE deleted_at IS NULL AND CAST(strftime('%s', ts) AS INTEGER) >= ?`, sinceTs),
  }
}
```

Two details worth noting. **Alerts and injects are split off the same table** by `alert_type` — they're genuinely different signals (a problem versus a deliberate nudge) and deserve their own lanes. And **broadcasts store their timestamp as an ISO string**, not a Unix integer like everything else, so `strftime('%s', ts)` normalises it inline — both in the SELECT and the WHERE — so the rest of the pipeline sees one uniform timestamp type. Pull only what you'll plot, and reconcile the schema quirks at the edge.

## Bucketing into a heartbeat

The window is 48 hours, divided into hourly bins. The route pre-builds an empty, zero-filled bin for every hour oldest-to-newest, then drops each timestamp into its slot by integer division:

```ts
const WINDOW_HOURS = 48
const HOUR = 3600

const now      = Math.floor(Date.now() / 1000)
const nowHour  = now - (now % HOUR)                    // snap to the top of the hour
const startHour = nowHour - (WINDOW_HOURS - 1) * HOUR  // oldest bin's start

for (const key of SOURCE_ORDER) {
  for (const t of ts[key]) {
    const idx = Math.floor((t - startHour) / HOUR)
    if (idx < 0 || idx >= WINDOW_HOURS) continue       // guard out-of-window
    bins[idx].counts[key]++
    bins[idx].total++
  }
}
```

Snapping `now` to the top of the hour matters: it keeps the rightmost bar stable instead of having the whole chart shimmer as seconds tick by. The out-of-range guard means a stray timestamp — clock skew, a backfilled row — can never write past the end of the array. Pre-seeding every hour, including the dead ones, is what gives the waveform its honest gaps: a quiet 3am reads as a row of flat stubs, not a compressed chart that hides the silence.

## The scaling trick that keeps rare events visible

Here's the choice that makes the EKG actually readable. Five event types fire at wildly different rates — memory writes might number in the dozens per hour while a broadcast happens twice a day. If every lane shared one y-scale, broadcasts would be an invisible smear along the axis while memory dominated the whole chart.

So each lane scales to *its own* busiest hour, independently:

```ts
const laneMax: Record<string, number> = {}
for (const s of sources) {
  laneMax[s.key] = Math.max(1, ...bins.map((b) => b.counts[s.key]))
}
// per-bar height, against this lane's own max:
const h = n === 0 ? 2 : Math.max(3, Math.round((n / max) * 40))
```

Each lane becomes its own EKG trace, normalised to its own range, stacked into a five-row monitor. You're no longer comparing absolute volumes across sources — you're reading the *rhythm* of each independently. Did memory writes flatline at the same hour alerts spiked? That's the kind of correlation the shape reveals and a table of totals buries. The `Math.max(1, …)` floor keeps an all-zero lane from dividing by zero, and the `n === 0 ? 2` rule draws a visible stub for empty hours so a flatline still reads as a line, not a void.

## Reading the monitor

Each lane carries its source colour — alerts red, injects cyan, memory purple, digests amber, broadcasts green — and hovering any bar surfaces its hour and exact count. The route also walks the bins to find the single **busiest hour** across all sources combined, surfaced as a headline so the eye has an anchor: "the fleet peaked at 2pm yesterday." The whole monitor refreshes every 60 seconds off a dedicated `/api/ekg` endpoint.

What the waveform exposes that five separate logs couldn't:

- **A fleet-wide flatline.** All five lanes go to stubs at once — nothing is happening anywhere. On a healthy autonomous fleet that's a red flag, not a rest, and no single log would have shown you the *simultaneity*.
- **A spike that crosses lanes.** Alerts jump and injects jump in the same hour — something went wrong and the system (or an operator) responded. Seeing both pulses align in time is the story; either alone is half of it.
- **A diurnal rhythm.** Memory and digest lanes breathing on a daily cycle is the normal heartbeat. Once you know the healthy rhythm, the *arrhythmia* is what you watch for.

## Lessons worth stealing

**Pull only what you plot.** The waveform needs timestamps, nothing else — so the query returns bare timestamp arrays from five tables, not hydrated rows. Less data, simpler bucketing, and the schema differences (split-by-type, ISO-vs-epoch) get reconciled in one small function instead of leaking into the view.

**Scale lanes independently when rates differ by orders of magnitude.** A shared y-axis is honest about absolute volume and useless for reading rhythm. When the question is "what's the *shape* of each stream," per-lane normalisation is the difference between a chart you can read and one where the loudest source drowns the rest.

**Pre-seed the empty buckets.** The silences are data. Zero-filling every hour, including the dead ones, is what lets a flatline look like a flatline — and on a fleet that's supposed to be working around the clock, the flatline is often the most important thing on the screen.

A pile of event logs tells you what happened. A waveform tells you the fleet's rhythm — and lets you feel the skipped beat before you go counting.
