# Watching the Watchers: Webhook Delivery Health and Feed Freshness for an AI Fleet

*Your monitoring dashboard stays green because the thing feeding it died. Here's how we closed two silent failure modes in our AI fleet's observability stack.*

An observability stack has one job: tell you when something breaks. So who tells you when the observability stack itself breaks?

That is not a rhetorical flourish. In Mission Control — the dashboard we run over our fleet of Claude agents — every alert about a stalled agent, a context blowout, or a memory drift is only useful if two invisible things are working. First, the alert has to actually leave the building: the outbound webhook that pushes it to Discord or a pager has to fire and get a `2xx` back. Second, the data the alerts are computed from has to be *fresh* — a "no anomalies detected" board is reassuring right up until you realize the table feeding it stopped receiving rows two days ago.

Both of those are silent failure modes. A broken webhook doesn't page you — it *is* the thing that pages you. A stale feed doesn't throw an error — it just quietly serves yesterday's truth. We shipped two views in Mission Control to drag both into the light: a **Webhook Delivery Health** board and a **Feed Freshness Wall**.

## The outbound pipeline you never look at

Mission Control records every webhook delivery attempt in a `webhook_deliveries` table — timestamp, status, response code, error string. We had per-webhook delivery logs for ages, but nobody scrolls a log to ask "is my alert pipeline healthy *overall*?" You need aggregation, and you need it sorted worst-first so the broken thing is at the top.

The data access is deliberately boring — one indexed range scan, aggregation in the caller:

```ts
export function getWebhookDeliveriesSince(sinceTs: number): WebhookDeliveryRow[] {
  return db.prepare(
    `SELECT * FROM webhook_deliveries WHERE ts >= ? ORDER BY ts ASC`
  ).all(sinceTs) as WebhookDeliveryRow[]
}
```

The `/api/webhook-health` route does the work: it buckets a rolling 7-day window of deliveries by webhook, then for each one computes a success rate, a response-code distribution, a per-day volume series, and the most recent error. One detail that turned out to matter — rows come back oldest-first, so the *last* failure the loop sees is automatically the most recent one. No second sort, no `MAX(ts)` subquery:

```ts
for (const r of rows) {
  const code = r.response_code != null ? String(r.response_code)
    : (r.status === 'timeout' ? 'timeout' : 'err')
  codeMap.set(code, (codeMap.get(code) ?? 0) + 1)
  if (r.status !== 'success') {
    // rows are oldest-first, so the last failure seen is the most recent
    lastFailureTs = r.ts
    lastError = r.error ?? `HTTP ${r.response_code ?? '?'}`
  }
}
```

Two design choices are worth calling out. A webhook with **zero** deliveries in the window reports a success rate of 100, not 0 — no traffic is not the same as total failure, and treating it as a red bar would cry wolf every time a quiet integration sat idle. And the cards sort by lowest success rate first, breaking ties by volume, so the noisiest broken webhook floats to the top of the grid. The healthy threshold sits at 99%; anything below that with real traffic counts toward a "degraded" tally in the header.

The page renders each webhook as a card: a success-rate gauge (green ≥99, amber ≥90, red below), a small volume sparkline, response-code chips, and the last error verbatim. The point is to answer "is the alert pipeline trustworthy?" in one glance — and if not, *which* hook and *what* code.

## The feeds that go quiet without complaining

The second blind spot is subtler. Mission Control's analytics — convergence trends, anomaly z-scores, burn-rate forecasts — all read from a handful of producer tables. If a background job that populates `convergence_history` dies, nothing errors. The convergence page just keeps showing the last value it has, looking perfectly healthy while being completely wrong.

So we built a Feed Freshness Wall that probes eight key data tables for two numbers: the newest row's timestamp, and how many rows landed in the last 24 hours.

The catch is that these tables don't agree on what time even *is*. Some store unix seconds in a `ts` column. Some store a date string. One stores an ISO hour like `2026-06-24T13`. The freshness probe normalizes all of them to epoch seconds in SQL, with the table names fully hardcoded — never caller-supplied — so the dynamic-looking query carries no injection surface:

```ts
const cut = `unixepoch() - 86400`
return [
  q('fleet_snapshots',
    `SELECT MAX(ts) AS lastTs, SUM(ts >= ${cut}) AS count24h FROM fleet_snapshots`),
  q('convergence_history',
    `SELECT CAST(strftime('%s', MAX(date)) AS INTEGER) AS lastTs,
            SUM(date >= date('now','-1 day')) AS count24h FROM convergence_history`),
  q('turn_quality',
    `SELECT CAST(strftime('%s', replace(MAX(hour),'T',' ')||':00:00') AS INTEGER) AS lastTs,
            SUM(replace(hour,'T',' ')||':00:00' >= datetime('now','-1 day')) AS count24h FROM turn_quality`),
  // ...five more
]
```

Freshness, though, is not one-size-fits-all — and this is the part that took the most thought. A fleet snapshot should land every couple of hours; if it's six hours old, something is wrong. But `alert_events` is *event-driven*: a quiet fleet that hasn't alerted in three days is good news, not a dead feed. Treating both with the same staleness rule would flood the board with false alarms.

So each feed gets its own expected cadence, and the `/api/freshness` route classifies against it: `healthy` if the newest row is within cadence, `late` if it's older, `silent` if the table is empty entirely.

```ts
const FEEDS = {
  fleet_snapshots:  { label: 'Fleet Snapshots',  cadenceSec: 2 * 3600 },
  alert_events:     { label: 'Alert Events',      cadenceSec: 7 * 86400 }, // event-driven; quiet ≠ broken
  digest_log:       { label: 'Digest Log',        cadenceSec: 36 * 3600 },
  // daily-rolled tables get a 36h grace so a late nightly job doesn't flip red
}
```

The wall then sorts most-stale-first — silent tables, then late ones, then by age descending — and renders each as a row with a colored pulsing dot. The thing that's broken is always at the top.

## What it changed

The honest outcome is not a dramatic number — it's a category of question we can now answer in seconds instead of never. Before these views, "is the alert pipeline actually delivering?" required grepping delivery logs, and "is this analytics page showing live data or a frozen snapshot?" had no answer at all short of opening the database. Both are now a single page in the Observability nav group, each on a short auto-refresh.

The deeper lesson is one we keep relearning: **a monitoring system is itself a system, and it deserves the same scrutiny it gives everything else.** The most dangerous failures in an observability stack are the quiet ones — the dashboard that stays green because the thing feeding it died. The fix isn't more alerts; it's instrumenting the *meta-layer* — the delivery mechanism and the data freshness — so that "all clear" actually means all clear.

The two patterns are cheap and portable. If you push alerts over webhooks, aggregate the delivery log into a success rate and sort it worst-first. If you compute health off producer tables, probe each one's newest row against a *per-feed* expected cadence — not a global timeout — so event-driven feeds don't drown you in false staleness. Neither needs a new dependency. Both close a hole you probably didn't know you had.

---

*Originally published at bistecglobal.com*

**BistecGlobal** builds AI-native enterprise software. Follow us for engineering insights.
