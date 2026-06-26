---
title: "Eleven Dashboards, One Question: Where Do I Look First?"
project: claude-mcd
tags: [AI, DevOps, Observability, Agents, Mission Control]
status: audited
date: 2026-06-26
---

# Eleven Dashboards, One Question: Where Do I Look First?

An operator running a fleet of autonomous AI agents has the opposite of an information problem. There is a context-pressure page, a circuit-breaker timeline, a watchdog-kill log, a composite health scorecard, an anomaly detector, a burn-rate forecast — each one a sharp, well-built lens on a single failure mode. Open any one of them and you learn something true. The trouble is the question that comes *before* any of them: out of a dozen projects and a dozen dashboards, which agent needs me **right now**?

That question has no home on a per-signal page. Context pressure lives on one screen, a tripped circuit breaker on another, and a quietly degrading health score on a third. To answer "what should I touch first?" the operator was effectively running a manual join across four data sources in their head, on a 30-second loop, all day. The Operator Inbox in Mission Control — the control plane for our multi-channel Discord agent fleet — exists to do that join for them.

## One queue, four signals

The design goal was deliberately narrow: do not invent new telemetry. Every signal the inbox surfaces already existed somewhere in the system. The job was aggregation and ranking, not measurement.

The `/api/inbox` route pulls from four sources, each with its own threshold:

- **Context pressure ≥ 80%** — read from the agent's Claude Code transcript. The route walks the last 20 JSONL entries of each session, finds the most recent `input_tokens` usage, and expresses it as a percentage of the 200k context window.
- **Open circuit breakers** — the last event in each project's `circuit-events.jsonl` is `open`, meaning a crashing subprocess has tripped its breaker.
- **Watchdog kills in the last 30 minutes** — a recent entry in `watchdog-kills.jsonl`, meaning an agent was detected stuck and killed.
- **Health score < 50** — a composite score recomputed inline from circuit opens, kills, and context pressure over a trailing 7-day window.

Each becomes a uniform `InboxAlert`, regardless of where it came from:

```ts
export type AlertType =
  | 'context-pressure' | 'circuit-open' | 'watchdog-kill' | 'low-health'

export interface InboxAlert {
  id: string
  slug: string
  type: AlertType
  severity: 'critical' | 'warning'
  message: string
  ts: string
}
```

That flat shape is the whole point. The page rendering the inbox does not know or care that context pressure came from a transcript and a tripped breaker came from an event log. They are all alerts with a severity and a project, and they all sort against each other.

## Severity is relative, not absolute

A list is not a queue. The value of the inbox is the ordering — and the ordering encodes a small amount of operational judgment.

Severity is not a fixed property of an alert type; it is computed from how bad the specific reading is. Context at 80% is a `warning`; at 90% it is `critical`. A health score of 49 is a `warning`; below 30 it is `critical`. The same signal escalates itself as it worsens:

```ts
if (pct >= 80) {
  addAlert({
    id: `ctx-${slug}`, slug, type: 'context-pressure',
    severity: pct >= 90 ? 'critical' : 'warning',
    message: `Context at ${pct}% — approaching 200k limit`,
    ts: nowIso,
  })
}
```

The final sort puts all criticals above all warnings, then orders by timestamp descending within each band:

```ts
alerts.sort((a, b) => {
  if (a.severity !== b.severity) return a.severity === 'critical' ? -1 : 1
  return b.ts.localeCompare(a.ts)
})
```

So the agent about to hit a context wall sits above the one merely running warm, and the breaker that tripped two minutes ago sits above the one that tripped an hour back. The top of the list is, by construction, the answer to "where do I look first?"

## Deduping the noise

A struggling agent rarely fails one way. A project hammering its context limit will often also carry a low health score, because context pressure is itself an input to that score. Surface both and the operator reads two alarms for one underlying problem — exactly the cognitive load the inbox set out to remove.

The route guards against this with a `seen` set keyed on `slug:type`:

```ts
function addAlert(a: InboxAlert) {
  const key = `${a.slug}:${a.type}`
  if (seen.has(key)) return
  seen.add(key)
  alerts.push(a)
}
```

One alert per project per signal type, no matter how many times the underlying log mentions it.

## Triage, not just display

Every alert card carries three affordances, because a triage queue you can only look at is half a tool. The card links straight to the project, and — the useful part — to the *right* specialized dashboard for that alert type. A context-pressure alert deep-links to the context-pressure page; a tripped breaker links to the circuit timeline:

```ts
const TYPE_DRILL: Record<AlertType, string> = {
  'context-pressure': '/context-pressure',
  'circuit-open': '/circuit-timeline',
  'watchdog-kill': '/watchdog-kills',
  'low-health': '/health-score',
}
```

The inbox answers "where do I look first?"; the Spotlight link then takes the operator to the page that answers "what exactly is wrong?" The eleven dashboards did not go away — the inbox became the index that points into them.

The third affordance is a **Dismiss 10m** button. Acknowledged alerts are stashed in `localStorage` with a 10-minute expiry, then auto-pruned. This deliberately avoids a server-side acknowledgement model: a dismissal is a personal, short-lived "I've seen this, stop showing it while I work" — not a durable state change. If the condition is still true ten minutes later, it comes back. The page refreshes every 30 seconds, so the queue stays live without the operator touching it.

## What it changed

The inbox added no new measurement and no new storage — roughly 230 lines of API and 210 of UI, sitting entirely on top of signals the fleet already emitted. That is the design lesson worth carrying: in a mature observability system, the missing piece is frequently not another metric but a **prioritized join across the metrics you have**.

Before the inbox, answering "which agent needs me?" meant opening four pages and mentally merging them on a loop. After it, that answer is a single ranked list that maintains itself — critical first, deduped, deep-linked, and self-clearing when the underlying condition resolves. The specialized dashboards remain the place to diagnose; the inbox is simply the front door that tells you which one to walk through.

For anyone building control planes over autonomous systems — agent fleets, job runners, device groups — the pattern generalizes cleanly. Normalize your heterogeneous signals into one alert shape, let severity float with the reading rather than the source, dedupe per entity per type, and rank so the top row is always the next action. The hardest observability problem at scale is rarely seeing everything. It is knowing what to look at first.
