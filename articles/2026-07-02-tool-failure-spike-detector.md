---
title: "When Three Agents Fail at Once: Correlating Tool Errors Across a Fleet"
project: claude-mcd
tags: [AI, Observability, DevOps, Reliability, Developer Tooling]
status: draft
date: 2026-07-02
---

A single agent throwing a tool error tells you almost nothing. Maybe it fed `Edit` a stale string. Maybe it asked `Bash` to `cd` into a directory it deleted two turns ago. Errors are the background noise of an autonomous agent doing real work — you can't alert on every one without training yourself to ignore the alerts.

But when *three* agents all fail on the *same* tool inside the *same* five minutes, that's not noise. That's a signal: an MCP server fell over, a web endpoint started rate-limiting, a shared credential expired. The tool layer broke, and every agent leaning on it is about to waste turns retrying into a wall. The hard part isn't detecting errors — it's telling the one from the many.

Mission Control, the observability layer for our multi-channel agent fleet, now does exactly that. The tool-failure spike detector reads what every agent already writes, correlates errors *across* projects, and fires only when a failure is fleet-wide. Here's the design.

## The signal is correlation, not count

The core decision: a spike is defined by how many *distinct projects* fail on a tool in a window, not by raw error count. One agent erroring 50 times on `Edit` is a stuck agent — a different problem. Three agents each erroring once on `WebFetch` in the same five minutes is an outage.

So the detector buckets errors into 5-minute windows keyed by `tool::bucketStart`, accumulates the *set* of affected project slugs per bucket, and only calls it a spike when that set crosses three:

```ts
const BUCKET_MS = 5 * 60 * 1000
const SPIKE_MIN_PROJECTS = 3
const SEVERITY_MEDIUM = 5
const SEVERITY_HIGH = 8

function severity(affected: number): SpikeSeverity {
  if (affected >= SEVERITY_HIGH) return 'high'
  if (affected >= SEVERITY_MEDIUM) return 'medium'
  return 'low'
}
```

Severity scales with breadth — how many projects are hit — because a tool failing across eight agents is a bigger fire than one failing across three. Counting projects rather than errors is what makes the alert trustworthy enough to act on.

## Reading errors from logs nobody has to emit

There's no new telemetry pipeline here. Every Claude Code session already writes a JSONL transcript, and a failed tool call already appears as a `tool_result` block with `is_error: true`. The detector reads that.

The subtlety is that the error block says *which call* failed (`tool_use_id`) but not *which tool*. The tool name lives on the earlier `tool_use` block in the assistant message that issued the call. So the scan makes one pass, building a map from call id to tool name as it goes, then resolves each error against it:

```ts
if (obj.type === 'assistant' && obj.message?.content) {
  for (const block of obj.message.content) {
    if (block.type === 'tool_use' && block.id && block.name) {
      toolNames.set(block.id, block.name)
    }
  }
}

if (obj.type === 'tool_result' && obj.is_error && obj.tool_use_id && obj.timestamp) {
  const tsMs = new Date(obj.timestamp).getTime()
  if (tsMs >= cutoffMs) {
    const toolName = toolNames.get(obj.tool_use_id) ?? 'unknown'
    errors.push({ tool: toolName, tsMs })
  }
}
```

Because the transcript is append-only and written live, the parser tolerates a malformed final line rather than failing the whole scan — a half-flushed record is normal for a running agent.

## Making a full-fleet scan cheap enough to run every 30 seconds

Scanning every transcript of every project on each request would be wasteful, so two cheap guards keep it fast. First, files whose modification time is older than the 24-hour window are skipped before they're ever opened — an agent that hasn't run since yesterday can't contribute to a spike now:

```ts
const st = fs.statSync(file)
if (st.mtimeMs < cutoff24h) continue
```

Second, the whole response is memoized for two minutes. The page auto-refreshes every 30 seconds, but repeated hits inside the window return the cached scan instead of re-reading the disk. The result is a spike view fresh enough to catch an outage as it starts, without hammering the filesystem.

## What the endpoint returns

One `GET` returns three views of the same data, so the UI never has to stitch them together:

- **`currentSpikes`** — spikes in the two most recent buckets, sorted by error count, for the "something is on fire right now" banner.
- **`history`** — the trailing spike timeline (capped at 200 entries), so an operator can see whether this is a one-off or a recurring 9am pattern.
- **`heatmapBuckets`** — *every* tool × bucket combination, not just the ones that spiked (capped at 500), rendered as a tool-by-time heatmap where hovering a cell reveals exactly which project slugs were affected.

That last view matters: the heatmap shows the sub-threshold errors too, so you can watch a failure spread from one project to two to three and catch it crossing the line, rather than only seeing it after it's already an alert.

## What it cost, and why the shape is right

The feature is two files behind the API — a 210-line route and its page — plus a sibling alert-delivery audit log that shipped alongside it. No schema change, no agent instrumentation, no new events. It reads transcripts that were always being written and correlates them on demand.

The leverage is entirely in the framing. Per-agent error monitoring drowns you: every autonomous agent generates recoverable errors constantly, and alerting on them individually is useless. Cross-agent correlation flips the noise into signal — it only speaks up when the failure is bigger than any one agent, which is precisely when a human should look. The threshold (three projects) is the whole idea; the code is just bookkeeping around it.

## Takeaways

- **Correlate across instances to separate systemic failure from local noise.** In any fleet — agents, services, workers — the count of *distinct affected instances* is a far better alarm trigger than raw error count.
- **Mine the logs you already have.** The error events and the tool names were both in the transcript; the feature was a read and a join, not a new pipeline.
- **Show the sub-threshold data too.** A heatmap that includes the near-misses lets an operator watch a problem spread and act before it trips the alert.
- **Keep a full-fleet scan cheap with mtime skips and a short cache.** You can afford to look at everything, often, if you skip what can't have changed and memoize the rest.

The tool-failure spike detector ships in Mission Control today. It didn't make our agents fail less — but it made the difference between "one agent is stuck" and "the tool layer is down" visible in a single glance.
