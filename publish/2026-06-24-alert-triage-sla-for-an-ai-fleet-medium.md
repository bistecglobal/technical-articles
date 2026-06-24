# Your AI Fleet's Alerts Are Lying to You — Until You Give Them a Lifecycle

*An append-only alert table feels complete right up until it's big enough to drown new signal in old. Here's the two-move fix: a triage lifecycle, then an SLA on top of it.*

An alert that nobody can close is not a signal. It's noise wearing a signal's clothes.

That was the quiet failure mode in Mission Control, the dashboard that watches our fleet of Claude agents. Every stall, every budget breach, every memory-drift spike landed in one growing `alert_events` table. The calendar heatmap lit up. The alert-flow Sankey fattened. And none of it told an operator the one thing they actually needed to know: *which of these have I already dealt with?* A three-day-old stall that was fixed an hour after it fired counted exactly the same as a fresh one. The dashboards were technically accurate and operationally useless.

We fixed it in two moves: first by giving every alert an acknowledgement lifecycle, then by measuring how fast we actually work through that lifecycle. The result is something most production teams have for human incidents but almost nobody has for autonomous agents — a triage workflow with an SLA attached.

## The problem: append-only alerting decays into noise

Mission Control's alerting started, as these things do, append-only. An anomaly detector or a budget watcher calls `insertAlertEvent(slug, alertType, description, payload)` and a row lands in SQLite. Every downstream view — the day-and-hour alert calendar, the alert-type flow diagram, the per-project counts — reads straight from that table.

Append-only is the right place to start and the wrong place to stay. With no notion of *handled*, the table only grows, and every aggregate it feeds inherits the same lie: handled and unhandled alerts are indistinguishable. Operators learn to ignore the dashboards, which defeats the entire point of having them. The signal-to-noise ratio doesn't just degrade — it inverts.

## Move one: an acknowledgement lifecycle

The fix had to respect a hard constraint: the alerts table was live in production with thousands of rows and a dozen readers. We couldn't rebuild it. So the migration is purely additive — two nullable columns, applied idempotently at startup:

```ts
// alert triage state: acknowledgement columns (additive migration)
try { db.exec("ALTER TABLE alert_events ADD COLUMN ack_ts INTEGER"); } catch { /* column already present */ }
try { db.exec("ALTER TABLE alert_events ADD COLUMN ack_by TEXT NOT NULL DEFAULT ''"); } catch { /* column already present */ }
```

A `NULL` `ack_ts` *is* the open state. No new table, no enum, no status string to keep in sync — the absence of a timestamp carries the meaning. State transitions are single guarded `UPDATE`s that double as a concurrency check:

```ts
export function acknowledgeAlert(id: number, actor: string): boolean {
  const r = db.prepare(
    `UPDATE alert_events SET ack_ts = unixepoch(), ack_by = ? WHERE id = ? AND ack_ts IS NULL`
  ).run(actor, id)
  return r.changes > 0
}
```

The `AND ack_ts IS NULL` clause means two operators clicking "Ack" on the same alert can't both win — exactly one `UPDATE` reports `changes > 0`, and the function returns whether *this* call performed the transition. The same pattern, inverted, re-opens an alert. No row-locking, no transaction ceremony; the predicate is the lock.

The read path stays backward-compatible through one optional flag. `getAlertEvents` gained `includeAcked`, defaulting to `true` so every existing caller behaves exactly as before:

```ts
if (opts.includeAcked === false) { conditions.push('ack_ts IS NULL') }
```

Then we flipped the dashboards that are supposed to show *current* pressure — the alert calendar and the alert-flow diagram — to pass `includeAcked: false`. Now they count only open signal. The historical views that want the full record simply don't pass the flag. One boolean, two truths, zero regression.

## Acknowledgement is an action, so it's audited

Closing an alert is a state change a human made, and in a fleet that runs largely unattended, every human action deserves a paper trail. The ack endpoint is gated on a real session and writes an audit-log entry on the way through:

```ts
const session = await auth.api.getSession({ headers: await headers() })
if (!session) return Response.json({ error: 'Unauthorized' }, { status: 401 })
// ...
const actor = session.user?.name || session.user?.email || 'unknown'
acknowledgeAlert(alertId, actor)
insertAuditLog({ actor, actor_id: session.user?.id ?? '', verb: 'alert.ack', target: String(alertId) })
```

`POST` acknowledges, `DELETE` re-opens, and each leaves an `alert.ack` / `alert.unack` record. Both validate that the ID is an integer and that the alert exists before doing anything — 400 and 404 respectively — so a malformed click can't write garbage. The actor stamped into `ack_by` is the same one logged, so the dimmed row in the UI ("acked by Chandima, 14:22") and the audit trail never disagree.

## Move two: measure the lifecycle you just created

The moment alerts have an open→acked transition with timestamps on both ends, you can measure the gap. That gap is a service-level metric, and it's the whole point of the second feature: the Alert Response Time view.

The design decision worth calling out is where the math lives. The database layer does *not* compute percentiles. It hands back the rawest possible rows:

```ts
export function getAlertSlaRows(sinceTs: number): AlertSlaRow[] {
  return db.prepare(
    `SELECT alert_type, ts, ack_ts FROM alert_events WHERE ts >= ?`
  ).all(sinceTs) as AlertSlaRow[]
}
```

Three columns over a 30-day window. Everything else — median, p90, ack rate, open backlog, age of the oldest open alert — is computed in the route handler in a single pass. For each acknowledged alert the latency is `ack_ts - ts`; unacknowledged alerts feed the open-backlog count and the oldest-open age instead:

```ts
if (r.ack_ts != null) {
  const lat = Math.max(0, r.ack_ts - r.ts)
  g.latencies.push(lat); allLatencies.push(lat)
} else {
  g.open++; totalOpen++
  const age = Math.max(0, now - r.ts)
  if (g.oldestOpen == null || age > g.oldestOpen) g.oldestOpen = age
}
```

Percentiles use linear interpolation between ranks rather than nearest-rank, so a median over an even number of samples lands between the two middle values instead of snapping to one — honest numbers on small per-type samples, which is the common case for a young fleet.

The view sorts worst-first: largest open backlog, then slowest median time-to-ack. An operator opening the page sees the alert *type* that is both piling up and taking longest to clear, at the top, every time. That's the triage queue, ranked by how much it's hurting.

## What it changed

The two features are deliberately small — a couple of nullable columns, two endpoints, one read-only analytics route — and that's the lesson. The expensive part of observability isn't collecting events; it's the lifecycle and the measurement *on top of* the events. Append-only alerting felt complete right up until the table got big enough to drown the new signal in the old.

Three principles carried the work:

- **Model state as the absence of data where you can.** A `NULL` timestamp is a cleaner "open" flag than a status column, because there's no second source of truth to drift.
- **Make migrations additive and reads backward-compatible.** A nullable column plus a default-on flag let us ship triage to a live table with a dozen readers and zero coordination.
- **Keep aggregation out of the database.** Returning raw `(type, ts, ack_ts)` rows and computing percentiles in one pass in the handler kept the SQL trivial and the statistics — interpolated percentiles, worst-first ranking — fully in code where they're testable.

An alert you can't close isn't telling you anything. Give it a lifecycle, then measure the lifecycle, and the same table that was drowning your operators becomes the queue that tells them exactly what to do next.

---

*Originally published at bistecglobal.com*

**BistecGlobal** builds AI-native enterprise software. Follow us for engineering insights.
