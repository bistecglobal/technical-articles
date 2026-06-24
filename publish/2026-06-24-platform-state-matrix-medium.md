# The Oldest Chart in Statistics, Rebuilt for an AI Agent Fleet

*Why we answered "where are the stalls concentrated?" with a 100-year-old contingency table instead of another bespoke visualization.*

When you run a fleet of autonomous AI agents, the question you ask most often is also the one your dashboard answers worst: *where is the trouble actually concentrated?*

Our Mission Control already had momentum rivers, convergence quadrants, and burn-rate forecasts — beautiful charts, every one of them. But when an operator glanced at the wall and asked "are the stalled agents a Discord problem or a Teams problem?", none of those charts answered cleanly. The information was *there*, scattered across nodes and bubbles, but it took mental arithmetic to extract. So we reached for the oldest tool in the statistician's drawer: the contingency table. Two categorical axes, counts at every intersection, totals in the margins. Karl Pearson would recognize it on sight. We call it the **Platform × State Matrix**.

## The problem with pretty charts

Most fleet visualizations optimize for one variable at a time. A state sunburst tells you how many agents are stalled. A platform breakdown tells you how many run on Teams. But the *interesting* questions are always two-dimensional — they live at the **intersection** of categories:

- Are stalls clustered on one channel adapter, or spread evenly?
- Is autonomous mode something only our Discord projects use?
- Does WhatsApp carry any active load at all, or is it dead weight?

A single-axis chart forces the operator to hold one dimension in their head while scanning the other. That's exactly the kind of cognitive load that gets ignored at 2 a.m. when an agent is wedged. A cross-tab collapses both axes onto one grid, and the answer becomes a glance instead of a calculation.

## The design: a heatmap that's also a count table

The matrix puts platform on the rows (`discord`, `teams`, `whatsapp`) and lifecycle state on the columns (`idle`, `active`, `stalled`, `autonomous`). Every cell holds the exact count of projects at that intersection, and the cell's background opacity scales with that count — so a dense cluster glows and an empty intersection fades to almost nothing. Row totals run down the right margin, column totals across the bottom, and the grand total sits up in the header. It is, deliberately, both a heatmap *and* a readable count table — you never lose the precise number to the color.

The core of it is a nested reduction over the fleet, which we keep memoized so it only recomputes when the fleet payload changes:

```tsx
const grid: Record<string, Record<string, number>> = {}
for (const pf of PLATFORMS) {
  grid[pf] = {}
  for (const st of STATES) grid[pf][st] = 0
}
for (const p of projects) {
  const pf = PLATFORMS.includes(p.platform ?? 'discord') ? p.platform : 'discord'
  const st = STATES.includes(p.state) ? p.state : 'idle'
  grid[pf][st] += 1
}
```

Two details in that loop matter more than they look. First, the grid is **pre-seeded with zeros** for every (platform, state) pair before any project is counted. That guarantees a complete, rectangular table — no missing cells, no "this combination never appeared so the column shifts" bugs that plague naive `groupBy` implementations. Second, every project is **defensively bucketed**: an unknown or missing platform falls back to `discord`, an unrecognized state falls back to `idle`. A surprise enum value from a future adapter can never produce a blank row or crash the render — it just lands in a sensible default bin.

The visual encoding is a single expression. Empty cells render flat; populated cells get a floor of opacity plus a share proportional to the busiest cell in the grid:

```tsx
opacity: n > 0 ? 0.18 + 0.82 * (n / max) : 1
```

That `0.18` floor is doing real work. Without it, a cell with one project in a fleet whose busiest cell holds twenty would be nearly invisible — technically present, practically unreadable. The floor guarantees that *any* nonzero count is legible, while the busiest cell still saturates fully. The normalization is against the per-cell maximum (clamped to at least 1 to avoid dividing by zero on an empty fleet), so the contrast is always tuned to the data actually on screen rather than some fixed scale that goes flat when the fleet is quiet.

## Building on what already exists

The feature that surprised us most about this build was how little it needed. There's no new API endpoint, no new database table, no background job. The page reuses the existing `/api/fleet` response — the same payload that already feeds half the dashboard — and does all of its cross-tabulation client-side inside a `useMemo`. The entire feature is a single ~125-line page component plus one line in the nav config to slot it under the Observability group.

```
                idle   active  stalled  autonomous   Σ
   discord       3       5        1         4       13
   teams         1       2        0         0        3
   whatsapp      0       0        0         0        0
   ────────────────────────────────────────────────────
   Σ             4       7        1         4       16
```

It refreshes on the same 60-second cadence as the rest of Mission Control, and it carries the same freshness badge — a small but non-negotiable habit in our dashboards: every view must visibly admit when its data is stale or when the last fetch failed. A chart that silently shows old numbers is worse than no chart, because it earns trust it hasn't paid for.

## What it taught us

Three things stuck with us after shipping this.

**The margins are the feature.** It's tempting to think the colored cells are the point and the row/column totals are decoration. It's the reverse. The marginal totals are what let an operator instantly sanity-check the grid — "16 agents total, 1 stalled, all of them on Discord" — without summing cells by eye. The heatmap draws the attention; the margins close the loop.

**Reusing the data source is a design decision, not a shortcut.** Every Mission Control view that reads from `/api/fleet` is, by construction, telling the same story as every other view at the same instant. There's no risk of the matrix disagreeing with the sunburst because they queried different snapshots. Picking one fleet endpoint as the single source of truth is what makes a dozen small visualizations feel like one coherent dashboard instead of a dozen apps in a trench coat.

**Old tools earn their keep.** The contingency table predates computing by a century, and it's still the most information-dense way to compare two categorical dimensions. We didn't need to invent a novel visualization to answer "where are the stalls" — we needed to stop avoiding the obvious one. When the question is genuinely two-dimensional and categorical, reach for the cross-tab before you reach for anything fancier.

For teams building their own agent observability: before you commission another bespoke chart, check whether the answer you want lives at the intersection of two axes you already track. If it does, you may already have all the data — you just haven't crossed it yet.

---

*Originally published at bistecglobal.com*

**BistecGlobal** builds AI-native enterprise software. Follow us for engineering insights.
