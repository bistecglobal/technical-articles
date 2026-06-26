---
title: "One Heatmap, Every Tool: Seeing What Your AI Agent Fleet Actually Does"
project: claude-mcd
tags: [AI, Observability, DevOps, Developer Tooling, Data Visualization]
status: audited
date: 2026-06-26
---

You run twenty autonomous coding agents. One of them has quietly made four thousand `Edit` calls this month and almost no `Read` calls. Is that a productive refactor agent — or one editing blind and hoping? You have no idea, because the only place that answer lives is twenty separate transcript folders, and nobody reads transcript folders.

This is the blind spot of fleet operations. We measure *that* agents are working — turn counts, token burn, stall alerts — but rarely *how* they work. Tool usage is the closest thing an LLM agent has to a behavioral fingerprint: a research agent leans on `WebSearch` and `WebFetch`, a builder lives in `Edit` and `Bash`, a planner barely touches the filesystem at all. When you can't see the fingerprint, you can't tell a healthy agent from a thrashing one until something breaks.

Mission Control — the observability layer for our multi-channel Discord agent fleet — already had a per-agent tool scorecard. What it lacked was the *cross-project* view: every project and every tool on one canvas, so the outlier jumps out instead of hiding in a drill-down. So we built one. It's a single page, `/tool-heatmap`, and it answers the question "who uses what, and how much" in one glance.

## The shape of the answer is a contingency table

The right visual was settled before we wrote a line: a project × tool matrix. Rows are projects, columns are tools, each cell's color encodes how many times that project called that tool. It's the oldest idea in statistics — a two-way frequency table — with the cell *numbers* swapped for cell *color* so the eye can scan hundreds of values at once.

The data already existed. Every Claude Code session writes a JSONL transcript, and every tool invocation appears as a `tool_use` content block inside an assistant message. We didn't need a new event pipeline, a database table, or instrumentation in the agents themselves. We needed to read what was already on disk.

The API route walks each project's transcript directory and counts blocks:

```ts
function countToolCalls(jsonlPaths: string[], cutoffMs: number): Record<string, number> {
  const counts: Record<string, number> = {}
  for (const p of jsonlPaths) {
    let lines: string[]
    try { lines = fs.readFileSync(p, 'utf-8').split('\n').filter(Boolean) } catch { continue }
    for (const raw of lines) {
      let line: JsonlLine
      try { line = JSON.parse(raw) } catch { continue }
      if (line.timestamp) {
        const tsMs = Date.parse(line.timestamp)
        if (isNaN(tsMs) || tsMs < cutoffMs) continue
      }
      const content = line.message?.content
      if (!Array.isArray(content)) continue
      for (const block of content) {
        if (block.type === 'tool_use' && block.name) {
          counts[block.name] = (counts[block.name] ?? 0) + 1
        }
      }
    }
  }
  return counts
}
```

Note the defensive posture: a malformed line is skipped, not fatal. Transcripts are append-only logs written live by a running process — a half-flushed final line is normal, and the scan has to tolerate it rather than 500 the whole page.

## Picking what to show, and what to drop

A fleet can have far more than 20 projects and far more than 30 distinct tools (MCP servers each contribute their own). A heatmap with 60 columns of one-pixel cells communicates nothing. So the route ranks and clamps:

```ts
// Sort projects by total tool calls desc, max 20
perProject.sort((a, b) =>
  Object.values(b.counts).reduce((s, v) => s + v, 0) -
  Object.values(a.counts).reduce((s, v) => s + v, 0)
)
const topProjects = perProject.slice(0, 20)

// Aggregate tool totals across the shown projects, pick top 30
const topTools = Object.entries(fleetToolCounts)
  .sort((a, b) => b[1] - a[1])
  .slice(0, 30)
  .map(([t]) => t)
```

The cap is honest rather than silent: the page header reports "N projects × M tools," so an operator can see when the view is truncated. The matrix is then a dense `number[][]`, plus `rowTotals` and `colTotals` — the marginal sums that turn a bare grid into a proper contingency table. The row bar on the right tells you which agent is busiest overall; the column bar along the bottom tells you which tool the *fleet* depends on most.

## Why the color is gamma-corrected

The naïve choice is a linear color scale: cell brightness proportional to count. It looks reasonable and hides everything. Tool usage is heavily skewed — one agent making 4,000 `Bash` calls compresses every other cell into the same near-black shade, and the interesting low-frequency signal (the agent that called `WebFetch` exactly twice) vanishes.

The fix is a perceptual curve:

```ts
function cellColor(count: number, max: number): string {
  if (count === 0 || max === 0) return 'rgba(255,255,255,0.03)'
  const t = Math.pow(count / max, 0.55)
  const r = Math.round(8 + t * (34 - 8))
  const g = Math.round(15 + t * (211 - 15))
  const b = Math.round(28 + t * (238 - 28))
  return `rgb(${r},${g},${b})`
}
```

The `Math.pow(count / max, 0.55)` exponent lifts low values up the brightness curve so they're distinguishable, while the dominant cell still reads as the brightest. Zero gets its own near-transparent treatment — an empty cell should look *absent*, not merely dim, so the eye separates "never used this tool" from "used it once."

## Rendering: SVG, not a chart library

The whole matrix is hand-drawn SVG `<rect>` and `<text>` elements — no charting dependency. For a grid of fixed-size cells this is less code than configuring a library, and it gives exact control over the two readability details that actually matter on a dense grid:

- **Rotated column labels.** Tool names are written at −55° so 30 of them fit without overlapping, and the `mcp__server__` prefix is stripped (`tool.replace(/^mcp__[^_]+__/, '')`) because the server namespace is noise once you're looking at the tool itself.
- **A hover tooltip** that resolves any cell back to its exact `project · tool · count`, because color tells you *roughly* and operators eventually want *exactly*.

The result is one SVG that scrolls horizontally if the fleet is wide, with a color-scale legend underneath so the gradient isn't a mystery.

## What it cost, and what it surfaced

The entire feature is four files: a 126-line API route, a 221-line page, one nav entry, and a backlog note. No schema migration, no new storage, no agent-side instrumentation — it reads transcripts that were already being written and computes everything on request with a 30-day window. That's the leverage of building observability on top of an append-only log: the data was always there; we just hadn't looked at it from this angle.

And the angle is the point. The per-agent scorecard answers "is *this* agent efficient?" The heatmap answers a question you can't ask one agent at a time: "which agent is the *outlier*?" A row that's all `Edit` and no `Read`. A column showing one project hammering a tool every other agent ignores. A nearly empty row from an agent that's supposedly active. Those patterns are invisible in twenty separate views and obvious in one.

## Takeaways

- **Your agents already emit a behavioral fingerprint.** Tool-call distribution is it. If you're running a fleet and not aggregating it, you're flying on token counts alone.
- **Build observability on the log you already have.** The transcript JSONL was the source of truth; the feature was a read, not a new pipeline. Look for the data before you instrument for it.
- **Skewed data needs a non-linear scale.** A linear heatmap of power-law data is a black rectangle with one bright corner. Gamma-correct the color, and give zero its own visual identity.
- **Truncate loudly.** Any "top N" view is lying by omission unless it tells you it truncated. Put the real counts in the header.

The cross-project tool heatmap ships in Mission Control today under Observability. It didn't make our agents smarter — but it made the question "what is this fleet actually doing?" answerable in a single glance, which is most of the battle.
