---
title: "Two Numbers, One Verdict: Triaging AI Agents on a Convergence × Context-Risk Scatter"
project: claude-mcd
tags: [AI, DevOps, Observability, Agents, DataViz]
status: draft
date: 2026-06-24
---

An agent that is 90% of the way to its goal with plenty of context window left is fine. An agent that is barely converging *and* nearly out of context is about to stall hard — and it looks identical to the first one in a list of running processes.

That blind spot is what pushed us to plot two numbers against each other instead of reading them in separate columns.

Our Multi-Channel Discord (MCD) platform runs a fleet of long-lived Claude agents, one subprocess per project. Two of the metrics it already tracked told very different stories. **Context usage** — how full the model's context window is — is a pressure gauge: the closer to 100%, the sooner the agent hits a reset and loses its working memory. **Convergence score** — how close the agent is to satisfying its `GOAL.md` — is a progress gauge. Read alone, each is ambiguous. High context usage on an agent that's basically *done* is no problem; it'll finish before the window fills. High context usage on an agent that's *nowhere near done* is a slow-motion failure: it will reset, forget, and start over, possibly forever.

The danger isn't either number. It's the *combination*. And combinations are exactly what a table hides and a scatter plot reveals.

## The two-axis verdict

So we built the **Convergence × Context Risk** view: every agent in the fleet plotted as a dot, with context usage on the x-axis (0–100%) and convergence on the y-axis (0.0–1.0). One glance places every agent in one of four quadrants:

```
 conv 1.0 ┤  ·        ·                 done, plenty of room  →  ignore
          │      ·                      done, tight context   →  fine, finishing
      0.5 ┼───────────────┬──────────
          │   ·       ·    │  ● ●  ●     ← AT-RISK: far from done,
          │        ·       │     ●          low on context = stalls hardest
      0.0 ┤                │
          └────────────────┴──────────
          0%        70% ctx          100%
```

The bottom-right quadrant — high context usage, low convergence — is the only one that matters operationally. We tint it red and call it the at-risk zone. Agents that land there are burning through their context window without making progress toward their goal. They are the ones to intervene on *first*: re-scope the goal, inject a summary, or restart with a tighter prompt before the reset wipes hours of work.

## The thresholds are the whole design

The implementation is a single client page that reuses the fleet's existing `/api/fleet` endpoint — no new backend. The intelligence is two constants and one filter:

```ts
const CTX_THRESHOLD = 70   // % context used
const CONV_THRESHOLD = 0.5 // convergence

const atRisk = points.filter(
  (p) => (p.contextUsagePct ?? 0) >= CTX_THRESHOLD
      && (p.convergenceScore ?? 0) < CONV_THRESHOLD * 100
).length
```

Those two dashed guide lines — 70% context, 0.5 convergence — slice the plane into quadrants, and the count of agents in the red quadrant is surfaced as a single number in the header. If it's zero, the fleet is healthy on this axis; if it's not, you know exactly how many agents need attention before you even look at the chart.

A subtle but important detail: convergence is *stored* on a 0–100 scale but *displayed* 0.0–1.0, so the filter compares against `CONV_THRESHOLD * 100` while the axis labels show `0.50`. Getting that conversion wrong would silently mis-classify every agent — the kind of bug that doesn't crash, it just lies.

## Rendering is plain SVG

There's no charting library here. The plot is hand-drawn SVG, which keeps the page dependency-free and the coordinate maths explicit:

```ts
// x: contextUsagePct 0–100; y: convergenceScore 0–100 (shown 0–1)
const sx = (ctxPct: number)   => PAD_L + (ctxPct / 100) * plotW
const sy = (convScore: number) => PAD_T + plotH * (1 - convScore / 100)
```

The `sy` inversion (`1 - score/100`) is the usual screen-vs-graph flip — higher convergence sits higher on the page. Each agent is a circle colored by its fleet state (cyan idle, green active, red stalled, purple autonomous), so a *red dot already sitting in the red quadrant* is the strongest possible signal: a stalled agent that the risk model would have flagged anyway. Hovering any dot reveals its slug, exact context %, convergence, and state, with a link straight to that agent's focus page.

Agents that don't report both metrics simply aren't plotted — the view only makes claims it has data for. The page refreshes every 60 seconds off the same fleet response the rest of mission control uses, so it never disagrees with the other dashboards.

## What the scatter caught that the table didn't

The payoff of a 2-D view is the agents in the *corners* — the ones that look unremarkable on either single metric.

- An agent at **68% context, 0.45 convergence** reads as "fine" in two separate columns. On the scatter it sits right against the at-risk boundary, and you watch it drift in.
- A **stalled (red) dot already deep in the red quadrant** confirms the failure was predictable: it ran low on context while far from its goal, exactly the pattern the quadrant is built to flag. The scatter turns a post-mortem into a forecast.
- An agent at **95% context but 0.92 convergence** sits in the safe top-right. The naive "high context = danger" alert would page you about it; the scatter tells you to leave it alone — it's about to finish.

That last case is the one that justifies the whole view. A single-metric context alarm cries wolf on every agent near the end of a long task. Crossing it with convergence silences the false alarms and keeps the real one.

## Lessons worth stealing

**One metric is a number; two metrics are a diagnosis.** Most agent dashboards show metrics in columns and leave the correlation to the operator's head. The moment a failure mode lives in the *interaction* of two signals — and most interesting ones do — a scatter with a tinted danger quadrant turns invisible risk into a glance.

**Make the threshold the product.** The entire feature is two constants and a filter. We didn't train a model to predict stalls; we drew two lines where domain knowledge said the danger starts and counted who fell on the wrong side. It's interpretable, it's tunable, and an operator can argue with it — which is exactly what you want from a triage tool.

**Reuse the source of truth.** By reading the same `/api/fleet` endpoint every other view uses, this page can never contradict them. New visualisation, zero new backend, zero new way for the dashboards to disagree.

A list of agents tells you what's running. A scatter of context against convergence tells you which one is quietly running out of road.
