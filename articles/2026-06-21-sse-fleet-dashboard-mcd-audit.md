---
slug: sse-fleet-dashboard-mcd
audited: 2026-06-21
verdict: NEEDS REVISION (fixes applied inline)
score: 20/25
---

# Audit Report — From Polling to Push: Migrating an AI Fleet Dashboard to Server-Sent Events

## Summary Verdict

**NEEDS REVISION → fixes applied.**  
Score: 20/25 (threshold 20). Three factual errors in the Problem section corrected before publish.

---

## Claim Inventory

| # | Claim | Evidence | Verdict |
|---|-------|----------|---------|
| 1 | Fleet of 19 AI agents | 19 entries in `/home/openclaw/.claude/channels/discord-multi/projects/` | ✅ |
| 2 | commit `d84461d` replaces polling with SSE | `git show d84461d` — confirmed on main | ✅ |
| 3 | **InstanceGrid polled /api/fleet every 30s** | InstanceGrid.tsx polls `/api/instances` + `/api/health`, NOT `/api/fleet`. InstanceGrid had NO diff in d84461d. | ❌ |
| 4 | **FleetHealthBar is a component** | No such component. Fleet health summary lives in `DashboardClient` (page.tsx). Removed poll was in page.tsx `useEffect` → `setInterval(fetchFleet, 30_000)`. | ❌ |
| 5 | **EventFeed polls /api/events every 30s** | EventFeed was already SSE-connected before d84461d. The change added exponential backoff (was fixed 3s) and filters fleet-update/stall-alert events. Not polling. | ❌ |
| 6 | StallAlertPanel polls /api/stalls | Confirmed: `setInterval(fetchStalls, 30_000)` in StallAlertPanel.tsx. Migrated in d84461d. | ✅ |
| 7 | 4 components × 2 endpoints × 2 req/min = 8 req/min | Depends on claims 3–5 which are wrong. Actual removed polls: fleet (2/min) + stalls (2/min) = 4/min per tab. | ❌ (derived from wrong inputs) |
| 8 | Each request re-reads channels.json and transcript mtime | `computeFleet()` calls `loadChannelEntries()` which reads `channels.json` (line 248 fleet-compute.ts) and `getTranscriptMtime()` per project | ✅ |
| 9 | SSE endpoint at /api/events/stream | `apps/mission-control/app/api/events/stream/route.ts` confirmed | ✅ |
| 10 | Broadcaster pushes every 5 seconds | `setInterval(broadcastFleetUpdate, 5_000)` in sse.ts | ✅ |
| 11 | Broadcaster starts on first SSE client, stops on last | `addClient()` calls `startFleetBroadcaster()`; `removeClient()` calls `stopFleetBroadcaster()` when `clients.size === 0` | ✅ |
| 12 | globalThis used to survive Next.js HMR | `const g = globalThis as {...}; g.__mcdClients ??= new Set(); g.__mcdFleetInterval` in sse.ts | ✅ |
| 13 | Keepalive fires every 5 seconds | `setInterval(() => c.enqueue(': keepalive\n\n'), 5_000)` in stream/route.ts | ✅ |
| 14 | FleetContextProvider opens one EventSource per tab | FleetContext.tsx — single EventSource per provider instance | ✅ |
| 15 | Exponential backoff 1s → 30s | `MIN_BACKOFF = 1_000`, `MAX_BACKOFF = 30_000`, `Math.min(backoff * 2, MAX_BACKOFF)` in FleetContext.tsx | ✅ |
| 16 | REST fetch on mount for initial data | `useEffect(() => { fetch('/api/fleet')... fetch('/api/stalls')... }, [])` in FleetContext.tsx | ✅ |
| 17 | stallSource = sseStatus === 'disconnected' ? polledStalls : sseStalls | Exact code at StallAlertPanel.tsx line 145 | ✅ |
| 18 | Status dot shows connected/reconnecting/disconnected | `sseStatus` exposed from FleetContext, used in page.tsx HUD header | ✅ |
| 19 | Migration shipped in PR #77 (345a3de) | `git log`: `345a3de Merge pull request #77 from chan4lk/feat/p28-sse-live-event-stream` | ✅ |
| 20 | computeFleet() runs once per 5s regardless of client count | `broadcastFleetUpdate()` called from single `setInterval` in sse.ts; all clients share one push | ✅ |
| 21 | checkBudgetAlerts fires at 50%/80%/100% thresholds with per-month dedup | `checkBudgetAlerts()` in sse.ts, `budgetAlertState` Map with `slug:threshold:YYYY-MM` key | ✅ |
| 22 | Core change is under 250 lines | stream/route.ts ~36 lines + sse.ts ~97 lines + FleetContext.tsx ~110 lines = ~243 lines | ✅ |

---

## Forward-Looking Scan

Searched for: will, plan to, coming soon, in the future, next step, roadmap, soon.

No forward-looking statements found. ✅

---

## Rubric Scorecard

| Dimension | Score | Notes |
|---|---|---|
| Evidence quality (claims cited) | 3/5 | Core architecture well-cited; 3 claims in Problem section were factually wrong |
| Technical depth | 4/5 | Good coverage of broadcaster, stream route, context, backoff, fallback |
| Clarity for target audience | 4/5 | Clear narrative; architecture diagram would improve it |
| BistecGlobal voice | 4/5 | Professional, practitioner-focused throughout |
| Title specificity | 5/5 | Precise and accurate |

**Total: 20/25**

---

## Fixes Applied

### Fix 1 — Problem section rewritten (Claims 3, 4, 5, 7)

**Old:**
> Before commit `d84461d`, four components each ran independent 30-second `setInterval` loops:
> - **InstanceGrid** — polls `/api/fleet` to render per-agent state chips
> - **FleetHealthBar** — polls `/api/fleet` for aggregate idle/active/stalled/autonomous counts
> - **StallAlertPanel** — polls `/api/stalls` ...
> - **EventFeed** — polls `/api/events` for the latest tool-call events
> Four components × two endpoints × two requests per minute = eight server round-trips per minute, per browser tab.

**Fixed:**
> Before commit `d84461d`, two independent 30-second polling loops ran on every dashboard page load:
> - **DashboardClient** (page.tsx) — polled `/api/fleet` every 30 seconds for aggregate fleet health (idle/active/stalled/autonomous counts)
> - **StallAlertPanel** — polled `/api/stalls` every 30 seconds to surface blocked agents
> Two endpoints × two requests per minute = four server round-trips per minute per browser tab. EventFeed already used a persistent `EventSource` to `/api/events/stream`; the d84461d change added exponential backoff to its reconnection and filtered fleet-update/stall-alert events out of its feed.

---

## Status after fixes

Article updated to `status: audited`. Fixes are surgical — architecture sections unchanged.
