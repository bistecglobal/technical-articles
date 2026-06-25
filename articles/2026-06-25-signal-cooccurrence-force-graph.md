---
title: "Which Alerts Travel Together? A Co-occurrence Graph for an AI Agent Fleet"
project: claude-mcd
tags: [AI, Observability, DevOps, Data Visualization, Multi-Agent Systems]
status: draft
date: 2026-06-25
---

A stalled agent fires a `stall` alert. Useful. But the alert that matters is the one you can't see in a flat list: every time that agent stalls, its context-pressure warning fired first, and a memory-thrash signal fired alongside. Three separate rows in your alert log. One root cause.

Flat alert streams hide that relationship. You read them top to bottom, ack them one at a time, and never notice that two signals show up together on the same project nine times out of ten. The pattern — not the individual firing — is the diagnosis. So in Mission Control, the dashboard we run over our fleet of Claude agents, we built a view that makes the pattern the primary object: a force-directed graph of which attention signals co-occur, and how strongly.

## The problem with reading alerts one at a time

Mission Control already had a unified attention engine — a single module that scans the fleet and emits *findings*: typed signals like `stall`, `context-pressure`, `budget-burn`, `memory-thrash`, each tagged with a severity. Those findings drive the Fleet Brief and a handful of dashboards. Every time the Brief computes, the findings are also written to an `attention_event` table, building a history of what fired, on which project, on which day.

That history was sitting there as rows. Querying "how many `stall` events last week" was easy. Querying "when `stall` fires, what else tends to fire on the same project" was not — and that second question is the one an operator actually asks when triaging a degrading fleet. Correlated signals point at a shared cause; isolated signals are noise. We wanted the correlation surfaced visually, not reconstructed in someone's head.

This isn't a niche concern. The industry conversation in 2026 has moved hard toward *correlated failure modes* in multi-agent systems — the recognition that agent problems rarely arrive alone and that flat, uptime-style monitoring misses the semantic relationships between them. A co-occurrence graph is a direct, low-tech answer to that: let the edges show you which problems are the same problem.

## Building the graph from event history

The whole feature is one API route and one page. The route, `/api/signal-cooccurrence`, turns the event log into a weighted graph.

The core idea: group attention events into buckets, where each bucket is one project on one day. Within a bucket, every distinct signal is a node firing, and every *pair* of signals that share the bucket is an edge. Repeat across the window, count the firings and the pairings, and you have node weights and edge weights.

```ts
for (const [groupKey, sigSeverity] of groups) {
  const slug = groupKey.includes('|') ? groupKey.split('|')[1]! : groupKey
  const signals = Array.from(sigSeverity.keys()).sort()
  for (const sig of signals) {
    const acc = nodeAcc.get(sig) ?? { count: 0, severity: 'ok', slugs: new Set() }
    acc.count++
    acc.slugs.add(slug)
    if ((SEV_RANK[sigSeverity.get(sig)!] ?? 0) > (SEV_RANK[acc.severity] ?? 0))
      acc.severity = sigSeverity.get(sig)!
    nodeAcc.set(sig, acc)
  }
  // Every unordered signal pair that co-fired in this bucket.
  for (let i = 0; i < signals.length; i++)
    for (let j = i + 1; j < signals.length; j++) {
      const key = `${signals[i]}|${signals[j]}`
      edgeWeight.set(key, (edgeWeight.get(key) ?? 0) + 1)
    }
}
```

A few design decisions are baked into those lines:

- **Bucket = project-day.** Co-occurrence has to mean "on the same project, in the same window," not "somewhere in the fleet this month." Keying buckets by `date|slug` keeps the correlation local and honest. Two signals that fire on different projects are not related.
- **Worst-severity wins per node.** A signal might fire as `warn` one day and `critical` the next. The node carries the worst severity it ever reached, so the graph's color coding answers "how bad does this get," not "how bad was it last time."
- **Signals are sorted before pairing.** Sorting the signal list means each pair is generated in a stable lexicographic order, so `a|b` and `b|a` collapse to the same edge key. No double counting, no directionality we don't have.
- **Healthy is not a signal.** A guard drops anything with an `ok` severity or a `healthy` signal before it ever reaches a bucket. The graph is a map of problems, not a map of everything.

The response ships the nodes, the edges, and the maxima the front end needs to scale its visuals — plus the `dominantSignal`, the single most-fired node, surfaced in the header as the fleet's headline problem.

## Degrading gracefully when there's no history

A graph built from history is empty on a fresh install — no events recorded yet, nothing to correlate. Rather than show a blank canvas, the route degrades to a **live snapshot**. When the `attention_event` table returns nothing, it calls the same attention engine the Brief uses, computes the current findings directly, and buckets them by project (one bucket per project instead of per project-day, since there's only a single point in time):

```
events in history?
  ├── yes → bucket by project-day  → mode: "history"
  └── no  → compute live findings   → mode: "live"
             bucket by project
```

The page badges which mode produced the graph, so an operator is never misled about whether they're looking at a 30-day pattern or a single instant. Same graph-building code path either way — only the bucketing differs.

## Letting the layout do the explaining

The front end deliberately reuses the force simulation that already powers Mission Control's memory graph rather than inventing a second physics engine. Nodes are sized by total firings (square-root scaled, so a signal firing 100 times doesn't dwarf one firing 10 into invisibility). Edges pull harder and draw thicker the more two signals co-occur — the link force shortens the distance and raises the strength in proportion to edge weight:

```ts
.force('link', d3.forceLink(simLinks)
  .id((d) => d.signal)
  .distance((l) => 140 - Math.min(90, (l.weight / maxEdgeWeight) * 90))
  .strength((l) => 0.15 + (l.weight / maxEdgeWeight) * 0.5))
```

The payoff is that tightly-correlated signals physically clump. You don't read the graph so much as glance at it: a dense, short-edged cluster *is* a correlated failure mode. Color tells you how severe it gets; node size tells you how often; proximity tells you what comes with it.

Hovering a node lists the projects firing that signal, each a deep-link straight to that project's Fleet Brief — so the graph isn't a dead-end visualization but the front door to triage. See the cluster, click the worst project, read the brief.

## What it changed

The view costs almost nothing to run — a single read of an existing table, no new data pipeline, a 60-second refresh — because every input already existed. The attention engine, the event history, and the force-sim component were all in place; this feature only connected them. That's the quiet lesson: the most useful observability often isn't new instrumentation, it's a new *projection* of telemetry you're already collecting.

The behavioral change is what we were after. Triage used to start with a list and a guess. It now starts with a shape. When two signals are bound by a thick edge, we stop treating them as two incidents and start hunting for the one cause underneath — which is exactly the correlated-failure problem the multi-agent field keeps warning about, answered with a graph instead of a spreadsheet.

If you run a fleet of anything — agents, services, jobs — and your alerts arrive as a flat stream, try counting co-occurrences before you build another threshold. The relationships between your alerts usually carry more signal than the alerts themselves.

---

*Built into BistecGlobal's Mission Control, the dashboard we use to operate our multi-project Claude agent fleet.*

Sources on the 2026 correlated-failure conversation: [Building Observability for Multi-Agent Systems (Medium)](https://medium.com/data-science-collective/your-ai-agent-isnt-down-it-s-wrong-building-observability-for-multi-agent-systems-aeb9fb6badd3), [Why observability platforms are becoming AI auditing tools (The New Stack)](https://thenewstack.io/agentic-ai-observability-auditing/).
