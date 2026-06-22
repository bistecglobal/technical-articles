---
title: "Your AI Fleet's Black Box: Point-in-Time Snapshots and Alert History for Mission Control"
project: claude-mcd
tags: [AI, DevOps, Observability, Autonomous Agents, Engineering]
status: audited
date: 2026-06-22
---

# Your AI Fleet's Black Box: Point-in-Time Snapshots and Alert History for Mission Control

An agent stalled last Tuesday at 3 PM. A budget threshold fired overnight. You only noticed this morning — and by then, your live dashboard had moved on. If you've run a fleet of autonomous AI agents for any length of time, you know this feeling: the present is visible, but the past is dark.

We've been running 19+ Claude Code agents continuously through Mission Control, the fleet dashboard at the heart of our [MCD](https://github.com/chan4lk/claude-multi-channel-discord) platform. Live metrics — state counts, SSE-pushed diffs, token gauges — tell us what's happening *right now*. They tell us nothing about what happened while we slept. We needed two things: a way to freeze fleet state at any point in time, and a permanent log of every alert that fired. This post is about how we built both.

## The Gap Live Dashboards Can't Fill

Mission Control already had strong real-time visibility: SSE-pushed fleet diffs, stall detection every 5 seconds, budget alerts at 50/80/100% thresholds. All of it ephemeral. Once a project transitioned from `stalled` back to `active`, the stall was gone from the UI. If three agents hit their budget ceiling between midnight and 6 AM, you'd never know unless you happened to be watching.

Real observability requires two things: *snapshots* (what was the state at point T?) and *audit trails* (what events occurred between T₁ and T₂?). Traditional APM tools give you both for infrastructure. We had neither for our agent fleet.

## Fleet Snapshots: Freezing State in SQLite

The snapshot design is deliberately simple. When you click "Take Snapshot" in Mission Control, the `/api/snapshots` route calls `computeFleet()` — the same function that drives the live dashboard — and serialises the result as JSON into a `fleet_snapshots` table in SQLite:

```sql
CREATE TABLE IF NOT EXISTS fleet_snapshots (
  id            INTEGER PRIMARY KEY AUTOINCREMENT,
  label         TEXT NOT NULL DEFAULT '',
  ts            INTEGER NOT NULL DEFAULT (unixepoch()),
  project_count INTEGER NOT NULL DEFAULT 0,
  data          TEXT NOT NULL DEFAULT '{}'
);
```

Each row captures a timestamped blob: total idle/active/stalled/autonomous counts, and for every project — slug, state, monthly tokens used, monthly budget, goal text, and goal status. The label field is freeform: "before deploy", "end of sprint", "after incident". Unlabeled snapshots fall back to a human-readable timestamp in the UI.

What makes this useful is the **A/B diff view**. You select two snapshots from the list; the client fetches both rows, parses their JSON blobs, and runs a `diffSnapshots()` function that classifies every project into one of four buckets: `added`, `removed`, `changed`, or `same`. Only non-`same` rows appear in the diff table, along with a color-coded token delta (green if tokens decreased, amber if they grew). The sort order — removed first, then added, then changed — surfaces the most operationally significant changes at a glance.

The diff runs entirely client-side. No extra server roundtrip, no database join. Two `fetch()` calls and a JavaScript `Map`. For a fleet of 20–30 projects the computation is instantaneous.

## Alert History: Making Transient Signals Permanent

The second table captures every alert event as it fires:

```sql
CREATE TABLE IF NOT EXISTS alert_events (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  ts          INTEGER NOT NULL DEFAULT (unixepoch()),
  slug        TEXT NOT NULL DEFAULT '',
  alert_type  TEXT NOT NULL,
  description TEXT NOT NULL DEFAULT '',
  payload     TEXT NOT NULL DEFAULT '{}'
);
```

Alert types in our fleet are `stall`, `budget`, and `inject`. Each is written to the table at the moment it fires — stall and budget alerts from the SSE broadcaster that already drives real-time push, inject events from the `/api/inject` endpoint that manually resumes a stalled agent. The connection point is a single `insertAlertEvent()` call wired into the existing event paths:

```typescript
// Inside the SSE broadcaster's stall-check loop
for (const s of stalls) {
  insertAlertEvent(
    s.slug,
    'stall',
    `Stall detected: ${s.stallReason}`,
    { stallReason: s.stallReason, stallAgeMins: s.stallAgeMins, checkedAt }
  )
}
```

Because the SSE broadcaster already runs on a 30-second interval and already computed the stalls, wiring in persistence cost exactly one function call per stall event. No new polling loop, no separate process.

The `/alerts` page renders these events in reverse-chronological order with type-badge filtering (clicking `stall` shows only stall events) and a debounced slug filter for drilling into a single project's history. Pagination is cursor-based: each page returns 100 rows and a `nextCursor` value if more exist. The table auto-purges rows older than 30 days on startup — the same retention window used for the existing `events` table.

## One Design Decision That Paid Off

We briefly considered storing snapshots as flat columns (one column per tracked metric) rather than a JSON blob. Flat columns would enable SQL aggregations: average token burn per snapshot, max stall count over time. We chose the blob approach for one reason: the shape of a "project" in our fleet evolves. New fields get added — goal status, budget status, tool counts. A JSON blob absorbs those changes without a schema migration. The few aggregate queries we actually needed (project count, idle/active/stalled/autonomous totals) were worth storing as dedicated columns precisely because we knew their shape wouldn't change.

The lesson: reach for structured columns only when you know the query pattern upfront and the schema is stable. For everything else, store the blob and parse it where you need it.

## What We Learned Running It

Within the first day of shipping snapshots, we caught something we'd missed entirely: two projects that had been consistently `stalled` at every snapshot taken overnight, but were `active` by morning. The heartbeat watchdog was rescuing them — which was working as designed — but the snapshots exposed a pattern we hadn't seen: these two projects were always the first to stall and always the ones that needed injection. That's signal worth acting on.

The alert history log surfaced something else: our budget alerts were firing in clusters. Three or four projects would hit the 80% threshold within the same hour on the same weekday. That's not coincidence — it's a sign they're all doing sprint-end pushes. The log gave us the timestamps to confirm it.

Neither of these insights was available from the live dashboard. Both were immediately obvious once we had persistent history.

## The Bigger Pattern

Live dashboards are for triage. Historical records are for understanding. We keep treating agent fleet management like it's Kubernetes — where a pod restart is atomic and recoverable and you mostly care about right-now. But a Claude Code agent session carries weeks of context, memory distillations, and goal state. When something goes wrong, "what happened and when" matters as much as "what's broken now."

Snapshots and alert history are a small addition to Mission Control — two new SQLite tables, two new pages, close to 900 lines of code across the stack. But they're the difference between operating blind and having a black box you can actually open.

If you're running autonomous agents at any scale, instrument the history early. You'll need it sooner than you think.

---

*Mission Control is part of the open-source [MCD multi-channel Discord platform](https://github.com/chan4lk/claude-multi-channel-discord).*
