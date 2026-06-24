# Scrubbing Through Time: A VCR for Your AI Agent Fleet's History

*A live dashboard tells you what is. A scrubber tells you what was — and when it turned.*

"It was fine an hour ago." Every operator of a live system has said it, and every live dashboard makes it impossible to prove. A real-time view is excellent at telling you what *is*. It is useless at telling you what *was* — and "what was, and when did it change?" is the first question you ask the moment something looks wrong.

Our Multi-Channel Discord (MCD) platform runs a fleet of long-lived Claude agents, one subprocess per project, and its mission-control dashboard had exactly this gap. We could see the current breakdown — how many agents idle, active, stalled, autonomous — refreshed live. We could *not* drag backward to the moment three agents went stalled at once and watch it happen. The data existed: the platform already wrote labeled fleet snapshots into a `fleet_snapshots` table. What was missing was a way to *play them back*.

So we gave the fleet a scrubber — a VCR transport for history.

## A slider over saved state

The **Snapshot Scrubber** is one page. It loads every stored snapshot, lines them up oldest to newest, and puts a slider under them. Drag the slider and the dashboard re-renders the fleet *as it was* at that instant. Hit Play and it auto-advances one snapshot per second, animating the fleet's evolution like time-lapse footage.

Each snapshot is a row with a timestamp, an optional label, and a JSON blob of the state counts. The first job is to parse and order them, defensively:

```ts
for (const r of rows) {
  let d: Record<string, unknown> = {}
  try { d = JSON.parse(r.data) } catch { /* skip malformed */ }
  out.push({
    id: r.id, ts: r.ts, label: r.label,
    idle: Number(d.idle ?? 0),
    active: Number(d.active ?? 0),
    stalled: Number(d.stalled ?? 0),
    autonomous: Number(d.autonomous ?? 0),
  })
}
return out.sort((a, b) => a.ts - b.ts) // oldest → newest
```

A malformed `data` blob doesn't crash the page — it degrades to zeros and the timeline survives. That matters for a replay tool: the *one* corrupt snapshot is often the interesting one, right next to the incident, and you don't want it taking the whole view down with it.

## The two details that make it feel like a tool, not a demo

A slider over an array is trivial. What separates a usable scrubber from a toy is how it handles two awkward moments: live data arriving underneath you, and the end of the tape.

**Live data arriving underneath you.** The page still refreshes every 60 seconds — it's reading the same snapshot store the rest of mission control uses. So new snapshots appear while you're scrubbing. If the selection index stayed fixed, your view would silently jump to a *different* snapshot as the array grew. Instead, the page tracks the set size and snaps the selection to the newest entry whenever the count changes:

```ts
const lastLen = useRef(0)
useEffect(() => {
  if (snaps.length !== lastLen.current) {
    setIdx(Math.max(0, snaps.length - 1))
    lastLen.current = snaps.length
  }
}, [snaps.length])
```

Land on the page and you're looking at *now* — the freshest snapshot — which is the sane default. Start dragging and you take manual control.

**The end of the tape.** Auto-play that loops back to the start is disorienting; you can't tell a fresh run from a replay. So Play advances once per second and simply *stops* when it reaches the newest snapshot:

```ts
timer.current = setInterval(() => {
  setIdx((i) => {
    if (i >= snaps.length - 1) { setPlaying(false); return i }
    return i + 1
  })
}, 1000)
```

It plays to the present and holds there. Any manual drag of the slider also cancels playback — you grabbed the wheel, the autopilot lets go.

## Showing the moment and the arc at once

For the selected snapshot, four tiles show that instant's counts — idle (cyan), active (green), stalled (red), autonomous (purple) — the same colour language used across mission control, so the scrubber reads consistently with every other view.

But a single frame loses the story. The point of a replay is the *trend*, so under the tiles sits a sparkline of `active + autonomous` — the "doing real work" count — across the entire snapshot history, with an amber marker pinned to wherever the slider sits:

```ts
const maxBusy  = Math.max(1, ...snaps.map((s) => s.active + s.autonomous))
const xAt = (i: number) => snaps.length <= 1 ? SPARK_W / 2 : (i / (snaps.length - 1)) * SPARK_W
const yAt = (v: number) => SPARK_H - (v / maxBusy) * (SPARK_H - 6) - 3
```

The `Math.max(1, …)` guard keeps the y-scale from dividing by zero when the fleet was entirely idle across every snapshot. The marker moves with the slider, so you see *both* the snapshot you're inspecting and where it falls in the larger arc — the spike you're standing on, and the slope that led to it.

## Why reuse beat building

The entire feature is a single client page. It introduces no new database table, no new endpoint, no new write path. It reads the `fleet_snapshots` data that was already being captured and the `/api/snapshots` endpoint that already served it.

That's the deliberate choice worth calling out. The snapshots were being recorded for a different feature — a state-diff view that compares any two labeled points in time. The raw material for *replay* was sitting in the same table, unused along the time axis. Building the scrubber meant recognising that an existing data source answered a new question, not standing up new infrastructure to answer it.

## Lessons worth stealing

**If you're snapshotting state, you're one slider away from time travel.** Plenty of systems periodically dump their state for backup or diffing. The moment that history is ordered and addressable, a scrubber is a small client-side layer on top — and it turns "it was fine an hour ago" from an excuse into a replayable claim.

**Handle the live-update collision explicitly.** A replay control sitting on top of auto-refreshing data has a real correctness bug waiting: the array grows and your index points at the wrong frame. Snapping to newest on a length change is a few lines, and skipping it makes the tool quietly lie.

**Play to the present, then stop.** Looping playback feels like a screensaver. Halting at the live edge makes the scrubber feel like an instrument — it ends where reality is.

A live dashboard answers "what is happening." A scrubber answers "what happened, and when did it turn" — and most of the time, that's the question you actually walked up to the screen with.

---

*Originally published at bistecglobal.com*

**BistecGlobal** builds AI-native enterprise software. Follow us for engineering insights.
