---
title: "The Watchdog's Logbook: Reading When and Why an AI Fleet Gets Killed"
project: claude-mcd
tags: [AI, Observability, DevOps, Autonomous Agents, Mission Control]
status: audited
date: 2026-07-02
---

# The Watchdog's Logbook: Reading When and Why an AI Fleet Gets Killed

A stuck agent is invisible until it isn't. It looks alive — a tmux session, a running process, a spinner that never resolves — right up until a watchdog decides it has been silent too long and kills it. Do that across nineteen autonomous projects for a few weeks and you have a pile of terminated sessions and no memory of them. Was it one flaky project or the whole fleet? Late at night or mid-afternoon? Always mid-tool-call, or scattered? Without a record, every kill is a surprise you can't learn from.

We already had the record. What we didn't have, until recently, was a way to *read* it.

## The problem: kills that leave no trace

Our multi-channel Discord bot runs one Claude subprocess per project. Each subprocess is supervised by a stuck-watchdog: if an agent goes silent past an adaptive threshold — a base timeout stretched to 1.5× its own longest observed turn, capped at 30 minutes — the pool kills the tmux session and respawns on the next message. That keeps a wedged agent from holding a slot forever.

The kill itself was clean. The *forensics* were not. When the watchdog fired, the session died and the reason evaporated. Operators would notice an agent had restarted and have no way to ask the obvious follow-up questions: how often does this happen, to whom, and what was the agent doing at the moment it froze?

## Step one: make every kill leave a note

The fix begins where the kill happens. When the process pool terminates a session with reason `watchdog`, it now appends a single line to a per-project `watchdog-kills.jsonl` before moving on:

```ts
private appendWatchdogKill(): void {
  try {
    const mcdDir = process.env.MCD_CHANNELS_DIR
    if (!mcdDir) return
    const lastToolCall = (() => {
      let latest: { toolName: string; startMs: number } | null = null
      for (const v of this.transcriptPendingTools.values()) {
        if (!latest || v.startMs > latest.startMs) latest = v
      }
      return latest?.toolName ?? null
    })()
    const entry = JSON.stringify({
      ts: new Date().toISOString(),
      slug: this.slug,
      runtimeMs: this.spawnedAtMs !== null ? Date.now() - this.spawnedAtMs : null,
      lastToolCall,
      reason: 'watchdog',
    }) + '\n'
    appendFileSync(join(mcdDir, 'projects', this.slug, 'watchdog-kills.jsonl'), entry)
  } catch {
    // Non-fatal — never let kill log failure block the kill itself
  }
}
```

Two decisions matter here. First, the `lastToolCall` is reconstructed from the agent's *pending* tool map — the tools that started but never reported completion. When an agent hangs, the tool it hung inside is exactly the one still sitting in that map. That field turns "it froze" into "it froze during a web fetch." Second, the entire append is wrapped in a swallow-everything `catch`. A logging failure must never block the kill it is trying to record. The watchdog's job is to free the slot; bookkeeping is strictly secondary.

## Step two: turn the logbook into a picture

A JSONL file scattered across nineteen project directories is evidence, not insight. The `/watchdog-kill-patterns` page reads every log through a single API route, filters by project slug and date range, and renders three views that answer three different questions.

**When do kills cluster?** A 7×24 heatmap — day of week against hour of day — bins every kill into a UTC cell. The server pre-seeds a full grid so empty cells render as empty rather than missing:

```ts
const heatmapMap = new Map<string, number>()
for (const e of filtered) {
  const d = new Date(e.ts)
  const key = `${d.getUTCDay()}-${d.getUTCHours()}`
  heatmapMap.set(key, (heatmapMap.get(key) ?? 0) + 1)
}
const heatmap: HeatmapCell[] = []
for (let day = 0; day < 7; day++) {
  for (let hour = 0; hour < 24; hour++) {
    heatmap.push({ hour, day, count: heatmapMap.get(`${day}-${hour}`) ?? 0 })
  }
}
```

The client shades each cell on an amber-to-red ramp scaled to the busiest cell, so the eye lands on the hot corner immediately. A cluster at 02:00 tells a different story from an even wash across the week: the first points at a scheduled job or an off-hours load spike; the second points at a chronically fragile agent.

**What were they doing when they froze?** A top-ten bar chart ranks the `lastToolCall` values. If eight of your last ten kills happened inside the same tool, you don't have a fleet problem — you have a tool problem. That is the single most actionable signal the logbook produces, and it costs nothing beyond a `Map` and a sort.

**Is context pressure to blame?** For each project with kills, the route estimates current context usage by reading the newest transcript file and pulling the token counts off the last usage record:

```ts
const total = (usage.cache_read_input_tokens ?? 0) + (usage.input_tokens ?? 0)
return Math.min(100, Math.round((total / 200000) * 100))
```

Plotting kill count against context percentage separates agents that die because they're overloaded from agents that die for some other reason entirely. A project pinned near 100% context that keeps getting killed is a candidate for compaction or a shorter leash; a project killed at 20% context is hanging on something external.

Above all three sits a single fleet-wide badge: kills per day over the trailing seven days. One number to tell you whether the fleet is getting more or less stable.

## What it changed

The value wasn't a dramatic bug fix — it was the end of guessing. Kills stopped being individual surprises and became a dataset. "This agent got killed again" became "this agent gets killed every night around 2 AM, always inside the same tool, at 90% context" — a sentence you can actually act on. The heatmap turns a fragile project into a bright cell; the tool chart names the culprit; the scatter tells you whether to shorten the leash or fix the tool.

There is a deliberate honesty in the design worth calling out. The log records only `watchdog` kills — not idle evictions, pool-full reclaims, or clean shutdowns. Those are healthy lifecycle events and would only add noise to a chart meant to surface *distress*. Scoping the logbook to genuine stalls is what keeps the hot cells meaningful.

## Lessons for anyone supervising autonomous agents

- **Log at the moment of death, not after.** The most useful field — the tool the agent was stuck inside — only exists in the pending-tools map at kill time. Reconstruct it a second later and it's gone.
- **Never let telemetry block the operation it observes.** The kill has to succeed even if the disk is full. Wrapping the append in a catch-all is not sloppiness; it's the correct priority.
- **Separate distress from routine.** An observability surface that logs every lifecycle event drowns the signal. Record only the events that mean something went wrong.
- **Three cheap cross-tabs beat one clever model.** A day/hour bin, a tool frequency count, and a context scatter are all elementary aggregations. Together they answer *when*, *what*, and *why* — which is most of an incident review.

A watchdog that kills silently keeps your fleet alive. A watchdog that keeps a logbook teaches you why it had to.
