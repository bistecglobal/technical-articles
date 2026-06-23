---
title: "Don't Wait for the Watchdog: Forecasting AI Agent Stalls Before They Fire"
project: claude-mcd
tags: [AI, Observability, DevOps, Agents, Reliability]
status: draft
date: 2026-06-23
---

A watchdog alert is, by definition, an obituary. By the time it fires, your agent has already gone quiet — the context window filled, the model lost the thread, or it crashed and never came back. You find out after the work has stopped, not before.

That asymmetry is the problem with most agent monitoring. It tells you what already broke. When you're running a fleet of autonomous Claude agents around the clock, "already broke" means hours of silent idle time before anyone notices. We wanted to know which agents were *about to* stall — while there was still a session alive to nudge. So we built a stall risk forecaster into Mission Control, our fleet dashboard for the multi-channel Discord bot (MCD).

## Reactive monitoring isn't enough

MCD already had a reactive safety net. A stall alert panel watches each agent's time-since-last-reply and trips when it crosses a watchdog threshold. A circuit breaker backs off agents that crash repeatedly. Both work — but both are triggered by an event that has already happened.

The forecaster answers a different question: *given everything we know right now, how likely is this agent to stall soon?* It produces a single 0–100 score per project, sorted worst-first, so an operator scanning the board sees trouble building before the watchdog ever fires.

## Four signals, weighted

The score is a weighted blend of four heuristics, each capturing a distinct way an agent drifts toward silence:

- **Context pressure (30%)** — how full the context window is, plus a penalty if it's filling fast. An agent at 85% with an estimated fill time under 30 minutes is in more danger than one parked at 85% and idle.
- **Turn-quality trend (30%)** — not just the latest turn-quality score, but whether it's *falling*. A decline of 10+ points across the last three turns adds a penalty. Degrading output often precedes a derailment.
- **Recency (25%)** — time since last reply, expressed as a fraction of that project's own watchdog threshold. At 80% of the threshold, the risk contribution is 80. This is the leading edge of the same signal the reactive panel uses — read earlier.
- **Kill history (15%)** — count of stall, watchdog, or circuit-breaker events for that project in the last 24 hours. A project that has already been killed twice today is fragile.

Each heuristic returns a risk value (0–100) and a weight; the final score is the clamped sum of `risk × weight`. The math is deliberately boring:

```typescript
const score = clamp(contribs.reduce((sum, c) => sum + c.risk * c.weight, 0))
```

The interesting decisions are in what *doesn't* get added.

## Honesty about what you don't know

The most important design choice was how to handle missing signals. Turn-quality data may not exist yet for a freshly started agent. Recency is meaningless for a project that's legitimately idle rather than working. The naive move — treat unknown as zero risk and renormalize the remaining weights — quietly inflates confidence.

We did the opposite. When a signal is unknown, its contribution is simply omitted, and the weights are *not* renormalized:

```typescript
function qualityRisk(slug, qualityBySlug) {
  const scores = qualityBySlug.get(slug)
  if (!scores || scores.length === 0) return null // unknown — don't inflate
  // ...
}

function recencyRisk(p) {
  if (p.state === 'idle') return null // not working — recency is irrelevant
  // ...
}
```

The effect is that an agent we know little about *cannot* score as high as one we have full telemetry on. Context pressure (30%) and kill history (15%) are always present; the other 55% has to be earned with real data. We'd rather under-warn on an unknown agent than cry wolf on every cold start. A forecaster that fires constantly gets muted, and a muted forecaster is worse than none.

## Built on one source of truth

The forecaster doesn't re-derive fleet state. It calls the existing `/api/fleet` endpoint — already the single source of truth for context percentage, project age, state, and per-project watchdog thresholds — and layers risk scoring on top. Kill history comes from the shared alert-events store, filtered to the last 24 hours with a `stall|watchdog|circuit` match. Turn-quality scores come straight from the metrics table.

This matters more than it sounds. Every duplicate derivation of "is this agent stuck" is a chance for two parts of the dashboard to disagree. By reusing `/api/fleet`, the forecaster and the reactive panel are reading the same numbers — the forecaster is just reading them earlier and combining them differently.

## Surfacing the *why*, not just the score

A number alone doesn't tell an operator what to do. So alongside the score, the API returns the factors that actually moved it — filtered to contributions worth at least 8 points and sorted strongest-first:

```typescript
const factors = contribs
  .map((c) => ({ ...c, contribution: c.risk * c.weight }))
  .filter((c) => c.contribution >= 8)
  .sort((a, b) => b.contribution - a.contribution)
  .map((c) => c.label)
```

The labels are human, not numeric: `context at 88%, filling in ~12m`, `quality dropping (now 54)`, `silent 4m / 5m threshold`. A glance tells you whether to compact the context, check in, or wait.

The `/stall-risk` page renders each at-risk project as a colored bar — green below 40, amber at elevated, red above 80 — with its factor chips and a one-click **Pre-inject** button. Pre-inject doesn't restart anything; it opens the inject terminal pre-filled with a gentle check-in: *"Please checkpoint your progress and confirm your next step."* Often that single prompt is enough to pull an agent back on track before it goes quiet.

The same data feeds a compact widget in the floating Fleet Advisor panel, which shows the top three at-risk projects with their scores so the warning is visible from any page, not just the dedicated one.

## What it changed

The shift is from *post-mortem* to *pre-emptive*. The reactive watchdog still exists as the last line of defense — but the forecaster moves the moment of intervention earlier, into the window where the session is still alive and a nudge still lands. Instead of discovering a dead agent and cold-restarting it, an operator sees an amber bar climbing and sends a check-in.

A few lessons held up:

- **Predict from signals you already collect.** We added no new instrumentation. The forecaster is a recombination of context, quality, recency, and event data the fleet was already emitting. The leverage was in the scoring, not in new plumbing.
- **Conservative beats clever for alerting.** Refusing to renormalize around missing data makes the score less mathematically tidy but far more trustworthy. An alert you can ignore is an alert everyone ignores.
- **A score needs a reason and an action.** The number gets attention; the factor chips explain it; the pre-inject button resolves it. Drop any of the three and operators stop using it.

This is unapologetically a heuristic, not a machine-learning model — four hand-weighted signals and some clamps. For forecasting agent stalls across a live fleet, that turned out to be exactly the right amount of sophistication: transparent enough to trust, cheap enough to run every minute, and accurate enough to catch trouble while there's still time to act.
