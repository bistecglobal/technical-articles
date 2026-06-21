---
title: "From Polling to Push: Migrating an AI Fleet Dashboard to Server-Sent Events"
project: claude-mcd
tags: [AI, DevOps, React, SSE, Architecture, Dashboard]
status: audited
date: 2026-06-21
---

# From Polling to Push: Migrating an AI Fleet Dashboard to Server-Sent Events

When you operate a fleet of 19 AI agents running concurrently across isolated Discord channels, latency in the monitoring dashboard has real consequences. A stalled agent that isn't surfaced for 30 seconds is an agent burning compute silently. This is the story of how we replaced the Mission Control dashboard's polling architecture with a Server-Sent Events stream — cutting request volume while delivering live fleet state.

## The Problem: N×2 Polling Requests Per Minute

Mission Control is the web-based operator interface for [claude-mcd](https://github.com/chan4lk/claude-multi-channel-discord), Bistec's multi-channel AI orchestration platform. It shows real-time status for every Claude subprocess, flags stalled agents, and provides an event feed of tool invocations across the fleet.

Before commit `d84461d`, two independent 30-second polling loops ran on every dashboard page load:

- **DashboardClient** (`page.tsx`) — polled `/api/fleet` every 30 seconds for aggregate fleet health (idle/active/stalled/autonomous counts and per-project state chips)
- **StallAlertPanel** — polled `/api/stalls` every 30 seconds to surface agents that hadn't emitted output within their stall threshold

Two endpoints × two requests per minute = four server round-trips per minute per browser tab. Each request re-read `channels.json` and the transcript mtime for every project from disk (`computeFleet()` in `src/fleet-compute.ts`). With a dashboard open across several operators — or with multiple tabs during development — the load compounded linearly.

`EventFeed` already used a persistent `EventSource` connection to `/api/events/stream` rather than polling; the `d84461d` change added exponential backoff to its reconnection logic and filtered `fleet-update` and `stall-alert` event types out of its feed (those now route exclusively through `FleetContext`).

Beyond the server cost, the UX problem was worse: a stalled agent could go unnoticed for up to 30 seconds, and there was no visual indicator telling an operator whether the data on screen was fresh or stale.

## The Solution: One EventSource Per Tab

The fix was to add a Server-Sent Events endpoint at `/api/events/stream` that broadcasts fleet diffs every 5 seconds and distribute that stream via a React context (`FleetContext`) so every component receives live state without issuing its own poll.

SSE was the right transport here. The dashboard is read-heavy — operators watch agents, they don't command them at high frequency. SSE delivers server-push over a plain HTTP response with built-in reconnection semantics, requires no WebSocket upgrade, and works cleanly with Next.js `force-dynamic` routes. A single persistent connection replaces the N polling timers.

## Architecture: Three Layers

### Layer 1 — The Broadcaster (`src/sse.ts`)

The server side maintains a `Set<ReadableStreamDefaultController>` of connected clients and a single `setInterval` that fires every 5 seconds, computing fleet state and pushing it to all connected clients (commit `d84461d`, `src/sse.ts`):

```typescript
function startFleetBroadcaster(): void {
  if (g.__mcdFleetInterval != null) return
  broadcastFleetUpdate()
  g.__mcdFleetInterval = setInterval(broadcastFleetUpdate, 5_000)
}

function stopFleetBroadcaster(): void {
  if (g.__mcdFleetInterval != null) {
    clearInterval(g.__mcdFleetInterval)
    g.__mcdFleetInterval = undefined
  }
}
```

`startFleetBroadcaster()` is called when the first SSE client connects; `stopFleetBroadcaster()` fires when the last client disconnects. No clients, no interval — the broadcaster only runs when someone is watching.

One implementation detail worth noting: the client set and the interval are stored on `globalThis` rather than in module scope. Next.js hot module replacement re-executes module code on each change, which would reset module-level variables and create duplicate intervals. Storing on `globalThis` survives HMR and ensures a single broadcaster regardless of how many times the module reloads during development.

### Layer 2 — The Stream Route (`app/api/events/stream/route.ts`)

The Next.js route handler returns a `ReadableStream` with `Content-Type: text/event-stream`. Each connecting client is registered with `addClient()` and deregistered via the `cancel()` hook when the connection closes. A keepalive comment-line (`': keepalive\n\n'`) fires every 5 seconds to prevent proxy timeouts:

```typescript
export async function GET(): Promise<Response> {
  let controller!: ReadableStreamDefaultController
  const stream = new ReadableStream({
    start(c) {
      controller = c
      addClient(controller)
      const hb = setInterval(() => {
        try { c.enqueue(': keepalive\n\n') } catch { clearInterval(hb) }
      }, 5_000)
    },
    cancel() {
      removeClient(controller)
    },
  })
  return new Response(stream, {
    headers: { 'Content-Type': 'text/event-stream; charset=utf-8', 'Cache-Control': 'no-cache' },
  })
}
```

### Layer 3 — `FleetContext` (client)

`FleetContextProvider` opens one `EventSource` per browser tab and distributes `fleet-update` and `stall-alert` events to any component that calls `useFleet()`. Reconnection uses exponential backoff capped at 30 seconds (`FleetContext.tsx`, commit `d84461d`):

```typescript
const MIN_BACKOFF = 1_000
const MAX_BACKOFF = 30_000

es.onerror = () => {
  setSseStatus('reconnecting')
  es.close()
  timerRef.current = setTimeout(() => {
    backoffRef.current = Math.min(backoffRef.current * 2, MAX_BACKOFF)
    connect()
  }, backoffRef.current)
}
```

On first connect, REST endpoints (`/api/fleet`, `/api/stalls`) are hit once for immediate data so the UI is populated before the first SSE push arrives. Components that relied on polling now call `useFleet()` to read from context — a one-line change for each.

## Graceful Degradation

Cutting over to SSE entirely would create a single point of failure. Instead, the implementation treats SSE as the primary path and REST polling as the fallback.

`StallAlertPanel` is the clearest example. It maintains a `polledStalls` state updated by a 30-second interval, but only uses that state when `sseStatus === 'disconnected'`:

```typescript
const stallSource = sseStatus === 'disconnected' ? polledStalls : sseStalls
```

When SSE reconnects, the component seamlessly switches back to pushed data. Operators see a status dot in the HUD header showing `connected`, `reconnecting`, or `disconnected`, so the data freshness is always visible.

## Outcomes

The migration shipped in PR #77 (`345a3de`). With the change live:

- **Request count** dropped from 4 fleet/stall polling round-trips per minute per tab to a single persistent SSE connection plus one REST fetch on mount.
- **Update latency** dropped from up to 30 seconds to 5 seconds for fleet-state and stall-alert data.
- **Server disk reads** are amortized — `computeFleet()` runs once every 5 seconds on the server regardless of how many clients are connected, rather than once per poll per component per tab.
- The budget-alert broadcaster added in the same commit (`checkBudgetAlerts()` in `src/sse.ts`) piggybacks on the same SSE pipe, firing `budget-alert` events when per-project token spend crosses 50%, 80%, or 100% thresholds — with per-threshold, per-month deduplication baked in.

## Lessons

**SSE fits monitoring dashboards.** A dashboard that primarily reads state benefits from server push far more than a full-duplex WebSocket. The EventSource API has built-in reconnection, works over HTTP/1.1 and HTTP/2, and is supported in every modern browser without polyfills.

**React context as a distribution bus.** Lifting the `EventSource` into a context provider means any new dashboard component can subscribe to live fleet state with a single `useFleet()` call. Adding a new widget no longer adds a new polling loop.

**Fallback is not optional.** SSE connections drop — proxies time out, network hiccups happen, the server restarts. Keeping REST endpoints live and having components detect `sseStatus === 'disconnected'` meant zero regression in reliability while gaining the push-delivery benefits.

**`globalThis` for Next.js singletons.** Any stateful server resource in a Next.js app that must survive hot module replacement needs to live on `globalThis`. Module scope is reset on each HMR cycle, which creates duplicate intervals and memory leaks in development — the pattern here is a reliable workaround.

---

The full implementation lives in the `chan4lk/claude-multi-channel-discord` repository. The core of the change — broadcaster, stream route, and React context — is under 250 lines and can be adapted to any Next.js application that needs real-time push without the overhead of WebSockets.
