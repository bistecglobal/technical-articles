# MTTR for AI Subprocesses: Putting a Stopwatch on Every Circuit Breaker Trip

*Your circuit breaker already knows when an agent fails. Here's how we turned that silent knowledge into SRE reliability metrics for the price of four lines and a try/catch.*

We already had a circuit breaker around our Claude subprocesses. When an agent crashes hard enough, often enough, in a short enough window, Mission Control stops trying to respawn it and lets it sit. Classic protection — it stops a thrashing process from burning tokens and log space against a wall.

But a circuit breaker that protects you silently has a blind spot the size of a fleet. It tells you *that* something tripped, in the moment, and then forgets. It can't answer the questions an operator actually asks the morning after: Which agent trips the most? When it trips, how long does it stay down before it recovers? Is the situation getting better or worse week over week? Those are reliability questions, and reliability questions have a standard answer language — MTTR, incident frequency, longest outage. We just hadn't been speaking it about our own agents.

So we did the unglamorous thing: we gave every circuit trip a stopwatch and a logbook.

## The cheapest possible event log

The circuit breaker already fired internal events on every state change — `circuit-open` when it gives up on a chatId, `circuit-reset` when the cooldown expires and it's willing to try again. Those events drove in-memory behavior and vanished. The first change is four lines in the pool's event dispatcher: before doing anything else with a circuit event, append it to disk.

```ts
private fireEvent(evt: PoolEvent): void {
  if (evt.kind === 'circuit-open' || evt.kind === 'circuit-reset') {
    this.appendCircuitEvent(evt)
  }
  if (!this.opts.onEvent) return
  // ...existing dispatch
}
```

The append itself writes one JSON object per line to a `circuit-events.jsonl` file inside each project's directory. The detail that makes the whole feature work is on the *close* event: we look up when the circuit opened and stamp the elapsed time right there.

```ts
const ledger = this.failureLedger.get(evt.chatId)
const openAt = ledger?.circuitOpenAt
const entry = {
  ts: new Date().toISOString(),
  slug: evt.slug,
  event: evt.kind === 'circuit-open' ? 'open' : 'close',
  reason: evt.kind === 'circuit-open'
    ? `${evt.failureCount} failures in window`
    : 'auto-reset after cooldown',
  durationMs: evt.kind === 'circuit-reset' && openAt != null
    ? Date.now() - openAt
    : undefined,
}
fs.appendFileSync(logPath, JSON.stringify(entry) + '\n', 'utf-8')
```

That `durationMs` is the entire ballgame. By computing the open-to-close duration at the moment of recovery — when both timestamps are trivially available — we never have to reconstruct it later by pairing opens with closes across a messy, possibly-interleaved log. The hard part of MTTR math is done at write time. The read side just averages a number that's already sitting in the file.

The write is wrapped in a try/catch that swallows failures on purpose. An observability side-effect must never be able to take down the thing it's observing; if the disk is full, the breaker keeps breaking and we lose a log line, not a process.

## Reading it back as reliability metrics

With a durable, append-only event stream per project, the metrics route is almost boring — which is exactly what you want. It walks each project's `circuit-events.jsonl`, parses the lines, and folds them into the numbers every on-call engineer recognizes:

```ts
for (const e of events) {
  if (e.event === 'open') totalOpens++
  if (e.event === 'close') {
    totalCloses++
    if (typeof e.durationMs === 'number') {
      sumDurationMs += e.durationMs
      closesWithDuration++
      if (longestOpenMs === null || e.durationMs > longestOpenMs) {
        longestOpenMs = e.durationMs
      }
    }
  }
}
const mttrMs = closesWithDuration > 0
  ? Math.round(sumDurationMs / closesWithDuration)
  : null
```

Three deliberate choices fell out of this:

- **Totals are all-time; rates are windowed.** `totalOpens` counts the whole logbook so the cumulative damage is honest, but "opens per week" is computed only over the trailing 30 days — recent behavior, not ancient history, predicts tonight.
- **MTTR is `null`, never zero, when there's nothing to average.** A project that has opened but never cleanly closed has *unknown* recovery time, not instant recovery. Reporting a confident `0` there would be a lie that hides the worst case.
- **Unclosed opens are surfaced, not swallowed.** The difference between `totalOpens` and `totalCloses` is the count of circuits that tripped and never came back — the agents that are arguably still down. That gap is shown next to the open count rather than averaged away.

The fleet view sorts projects by total opens, descending — the most fragile agent floats to the top — and renders each row with a color-coded MTTR (green under two minutes, amber under ten, red beyond), the longest single outage, an opens-per-week figure, and a 30-day sparkline of opens-per-day so a creeping trend is visible at a glance.

## What it actually buys you

Nothing about the agents' reliability changed the day we shipped this. No crash was prevented. What changed is that crashing became *legible*. "Agent X is flaky" went from a vibe to a row: opened 14 times, mean recovery 3m12s, longest 22 minutes, two trips still unresolved, trending up. That's a sentence you can prioritize against. It turns a wall of incidents into a ranked work queue.

There's a broader pattern here worth stealing. A circuit breaker — like a retry loop, a rate limiter, or any other piece of defensive machinery — is a place where your system *already knows* something went wrong. That knowledge is the most expensive signal you'll ever collect, because the failure already happened. Letting it evaporate after it changes one in-memory flag is a waste. Append the transition to a flat file with the duration pre-computed at close time, and you've converted a silent safeguard into an SRE dataset for the price of four lines and a `try/catch`.

The most useful reliability metrics in our fleet didn't come from new instrumentation. They came from writing down what the system was already deciding.

---

*Originally published at bistecglobal.com*

**BistecGlobal** builds AI-native enterprise software. Follow us for engineering insights.
