---
title: "The Average Is Lying to You: Percentile Latency for an AI Agent Fleet"
project: claude-mcd
tags: [AI, Observability, DevOps, SRE, Developer Tooling]
status: draft
date: 2026-06-26
---

A single number told us our agents were healthy. "Average response time: 41 seconds." Fine, right? Then an operator complained that one project felt sluggish — every command seemed to hang. The average said nothing was wrong. The average was lying.

That gap between a comforting mean and a painful tail is the oldest trap in performance monitoring, and it turns out AI agent fleets fall into it just as readily as web services do. When you run a fleet of long-lived Claude agents, each one churning through a transcript of user prompts and tool calls, "how fast does it respond" is not one number. It's a distribution. And the interesting part — the part that makes an operator wince — lives in the tail.

This is how we built per-project response-latency distributions into Mission Control, the dashboard that watches our multi-channel agent fleet, and what the percentiles showed us that the averages hid.

## The signal was already on disk

Every Claude Code session writes a JSONL transcript — one JSON record per line, each stamped with an ISO timestamp and a role. We didn't need to instrument anything new or add a timing wrapper around the agents. The latency signal was already sitting in those files; we just had to read it the right way.

The definition we settled on is deliberately simple: **response latency is the time from a user message to the first assistant reply that follows it.** That's the human-perceptible wait — you type, you watch the cursor, the agent starts talking. Tool calls and the model's internal back-and-forth happen *after* that first reply, so they don't count against the number an operator actually feels.

Parsing the transcript is a small state machine. Walk the records in order; when you see a genuine user message, remember its timestamp; when the next assistant message arrives, the delta is one sample.

```typescript
let pendingUserTs: number | null = null

for (const raw of lines) {
  const rec = JSON.parse(raw)
  const ts = rec.timestamp ? Date.parse(rec.timestamp) : NaN
  if (isNaN(ts)) continue

  const role = rec.message?.role
  const content = rec.message?.content ?? []

  if (role === 'user' && content.length > 0 && content[0]?.type !== 'tool_result') {
    pendingUserTs = ts
  } else if (role === 'assistant' && pendingUserTs !== null) {
    const delta = (ts - pendingUserTs) / 1000
    if (delta >= 0 && delta < 3600) {
      turns.push({ userTs: pendingUserTs, assistantTs: ts, deltaSeconds: delta })
    }
    pendingUserTs = null
  }
}
```

Two filters matter here. The `content[0]?.type !== 'tool_result'` check rejects the synthetic "user" turns that Claude Code emits when a tool returns — those aren't a human waiting, they're plumbing, and counting them would flood the data with near-zero deltas. And the `delta < 3600` guard drops any pairing longer than an hour, which is almost always an agent that sat idle overnight and resumed the next morning, not a real one-hour think.

## From samples to a shape

Once you have a list of deltas per project, the headline question is which numbers to surface. We compute six percentiles — p10, p25, p50, p75, p90, p99 — using linear interpolation between sorted samples:

```typescript
function percentile(sorted: number[], p: number): number {
  if (sorted.length === 0) return 0
  const idx = (p / 100) * (sorted.length - 1)
  const lo = Math.floor(idx)
  const hi = Math.ceil(idx)
  if (lo === hi) return sorted[lo]!
  return sorted[lo]! + (sorted[hi]! - sorted[lo]!) * (idx - lo)
}
```

Those six numbers map cleanly onto a box-and-whisker plot — the box spans p25 to p75, the whiskers reach p10 and p90, and a dot marks the median. One row per project, sorted worst-first by p90. An operator scanning the chart sees instantly which agents have a *long tail* versus which are merely slow on average. A project can have a tidy 8-second median and a p99 of three minutes — that's the profile of an agent that's usually snappy but occasionally stalls hard, and it's exactly the pathology a mean would smother.

We also colour-code each row by its median: green under 30 seconds, amber from 30 to 120, red beyond. The threshold isn't science — it's the boundary where our operators start to *feel* the wait.

Projects with fewer than three samples are dropped entirely. A percentile computed from two data points is theatre, not measurement, and showing it would invite false confidence.

## Is it getting better or worse?

A distribution is a snapshot. The next question an operator asks is "is this trending the wrong way?" So alongside the percentiles we compute a trend by comparing the p90 of the last seven days against the p90 of the seven days before that:

```typescript
function computeTrend(recent: number | null, prior: number | null): LatencyTrend {
  if (recent === null || prior === null || prior === 0) return 'stable'
  const change = (recent - prior) / prior
  if (change < -0.1) return 'improving'
  if (change > 0.1) return 'degrading'
  return 'stable'
}
```

We deliberately track the trend on **p90, not the mean** — the tail is where regressions show up first and where they hurt most. A 10% dead band keeps the arrows from flickering on noise; anything inside ±10% reads as "stable." Each window needs at least three samples of its own, or the comparison returns `null` and the project is reported as stable rather than fabricating a trend from one data point. The result is a single glyph per project — ↓ improving, → stable, ↑ degrading — with the actual before/after p90 values shown on hover.

## What the percentiles showed

The mean had been hiding two distinct failure modes. The first was the long-tail agent: respectable median, ugly p99, caused by occasional heavy tool sequences that blocked the next reply. The second was the genuinely-slow agent whose entire distribution had drifted right over a couple of weeks — invisible in a snapshot, obvious the moment the trend arrow turned red.

Neither was visible in "average: 41 seconds." Both were obvious in a box plot you can read in two seconds.

There's a broader lesson here that has nothing to do with AI specifically. We didn't add a metrics pipeline, a time-series database, or agent-side instrumentation. The data was already being written to disk as a side effect of how the agents run; the work was entirely in *reading it honestly*. The percentile is forty-year-old SRE wisdom. The novelty is only that the thing being measured is a language model in a loop, and the same discipline that keeps web services honest keeps agent fleets honest too.

If you're operating anything autonomous and long-running, audit your dashboards for averages. Each one is a place a problem can hide in plain sight. Replace it with a distribution, and the tail tells you what the mean never will.
