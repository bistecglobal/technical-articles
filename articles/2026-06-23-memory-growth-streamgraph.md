---
title: "Watching an AI Fleet Remember: A Silhouette Streamgraph of Memory Growth"
project: claude-mcd
tags: [AI, Observability, Data Visualization, Developer Tooling, Agents]
status: draft
date: 2026-06-23
---

A number tells you how much an agent remembers. It can't tell you *how it got there*. Our fleet of Claude agents each keeps a persistent memory — a directory of small Markdown files, one fact per file, that survives across sessions. The dashboard already reported a count per project. But a count is a snapshot, and the question that actually matters about memory is a shape: is this agent steadily accumulating knowledge, did it have one big distillation event last week, or has it gone quiet? You can't see momentum in a single integer.

So we drew the momentum. The `/memory-stream` page in Mission Control renders a silhouette streamgraph — cumulative memory entries per project, flowing left to right over thirty days, each project a coloured band whose thickness is how much it knows. It's the kind of chart that answers a question you didn't know how to ask: *which of my agents are growing, and when did they grow?*

## Dating an event that was never recorded

The first problem is that memory files don't carry a creation timestamp. They're written, occasionally rewritten, and that's it. There's no `created_at` field to bucket by.

Rather than retrofit a schema, the data endpoint uses what the filesystem already knows: file modification time. The `/api/memory/growth` route walks each project's `memory/` directory, skips the `MEMORY.md` index, and stamps every remaining `.md` file with its mtime as a stand-in for when it appeared:

```ts
function entryDates(memDir: string): string[] {
  const files = fs.readdirSync(memDir).filter((f) => f.endsWith('.md') && f !== 'MEMORY.md')
  const dates: string[] = []
  for (const f of files) {
    const stat = fs.statSync(path.join(memDir, f))
    dates.push(dayString(new Date(stat.mtimeMs)))
  }
  return dates
}
```

This is an approximation, and the code says so in a comment — memory files are "append-once mostly," so mtime tracks creation closely but not perfectly. That honesty matters. The chart is a trend visualisation, not an audit log; mtime is exactly precise enough for "is this project growing" and no one is billing against it. Choosing a good-enough signal that already exists beat inventing a precise one that would have required touching every write path.

## Cumulative, with a baseline

A growth chart needs a running total, not a daily delta — you want to see the line climb, not a noisy bar chart of "files touched today." For each of the 30 days in the window, the endpoint counts every entry dated on or before that day:

```ts
const baseline = dates.filter((d) => d < firstDay).length
const daily = days.map((day) => {
  const created = dates.filter((d) => d >= firstDay && d <= day).length
  return { date: day, count: baseline + created }
})
```

The `baseline` is the quiet detail that keeps the chart honest. A project that already had 40 memories before the window opened shouldn't appear to start at zero — that would draw a dramatic fake ramp on day one. Instead, entries predating the window are folded into the baseline, so each band *starts at its true height* and only the genuine growth inside the window shows as movement. Projects are then sorted by total descending, which fixes both the legend order and the stacking order so the picture is stable between refreshes.

## Drawing a stream without a charting library

The visual itself is a *silhouette* streamgraph: instead of stacking bands up from a flat baseline, it centres the total mass vertically so the composite shape flows like a river. There's no D3, no chart dependency — just SVG area paths computed by hand. The centering is one line:

```ts
const offset = (maxTotal - dayTotal) / 2 // silhouette centering
const y0 = padT + innerH - scaleY(below) - scaleY(offset)
const y1 = padT + innerH - scaleY(below + val) - scaleY(offset)
```

For each day, the difference between the window's peak total and that day's total is split evenly above and below, so the stack floats in the middle of the canvas. Each band's vertical extent is its own count stacked above whatever sits below it. The result reads as organic flow rather than a rigid bar stack — thickening where a project accumulates, tapering where it's idle.

Interaction is deliberately minimal: a vertical guide line follows the cursor, and the legend doubles as a readout — hover a day and each project's number switches from its all-time total to that day's count. One affordance, two jobs. The page repolls every 60 seconds, so a distillation run that lands new memories shows up within the minute.

## What the shape tells you

The payoff is pattern recognition at a glance. A band that thickens smoothly is an agent steadily learning. A sudden bulge is a distillation event or a burst of work that generated a lot of facts. A band that's gone flat is an agent that has stopped recording anything — which, for a long-running autonomous agent, is often the first visible sign something is wrong, well before a stall alert fires. The relative thickness across bands shows, instantly, which projects carry the most accumulated context and which are travelling light.

None of that is legible in a table of counts. The same numbers, given a shape and a time axis, become a story about how the fleet is thinking.

## Lessons

- **Reach for the signal that already exists.** File mtime isn't a perfect creation timestamp, but it was free, already accurate enough for a trend, and didn't require changing how memories are written. Match the precision of your data source to the precision your question actually needs.
- **Baselines keep cumulative charts honest.** Folding pre-window history into a starting height stops the chart from inventing dramatic ramps that never happened. The most misleading growth chart is one that secretly starts everyone at zero.
- **You don't always need a charting library.** A silhouette streamgraph is a few lines of stacking math and an SVG path builder. Hand-rolling it kept the bundle lean and the layout fully under control.
- **Let one interaction do double duty.** The legend is also the hover readout. Fewer, multi-purpose affordances beat a wall of tooltips.
- **Absence is a signal.** A band going flat — an agent that stopped remembering — is often the earliest visible symptom of trouble. Visualising growth quietly gives you a stall detector for free.
