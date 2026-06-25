# Context Runway: Knowing How Many Turns an AI Agent Has Left Before It Forgets

*A "78% context used" gauge tells you where you are. A runway tells you how many turns until the wall — and that's the number you can act on.*

A long-running AI agent doesn't crash when it runs out of context window. It does something worse: it quietly hits the wall, the harness compacts or resets the conversation, and the agent carries on having silently lost the thread of what it was doing. If you're running one agent and watching it, you notice. If you're running a fleet of them unattended, the first sign is usually a confused turn three messages after the damage was done.

We wanted to see the wall coming. Not "this agent is at 80% context" — a number that tells you where you are but not how fast you're getting closer — but an actual estimate: *this agent has about four turns left before it's full.* That's a runway. And a runway is the kind of number you can act on before it hits zero.

## Where the real context size hides

Every Claude session leaves a JSONL transcript on disk, one line per event. The naive way to measure context usage is to read the `input_tokens` off the latest assistant turn. That undercounts badly, because with prompt caching most of the conversation isn't in `input_tokens` at all — it's in the cache. The tokens are still occupying the window; they're just being billed and transported differently.

So the per-turn context size we actually care about is the sum of three fields:

```ts
const total =
  (u.input_tokens ?? 0) +
  (u.cache_read_input_tokens ?? 0) +
  (u.cache_creation_input_tokens ?? 0)
```

That sum is the true footprint pressing against the model's limit. The route walks the most-recent transcript for each project, pulls this number off every assistant turn that carries usage, and sorts the series by timestamp. The latest value is "where we are now"; the shape of the series is "how fast we're moving."

## Growth rate, not position

A percentage-used gauge is a position. A runway is a derivative. To get one from the other you need a growth rate, and context growth per turn is noisy — a turn that reads a big file jumps; a turn that just says "ok" barely moves; a compaction can make it *drop*. Averaging raw deltas including the negative ones would smear the trend.

The fix is a rolling window of the last few turns, counting only the turns where context actually grew:

```ts
const window = turns.slice(-6)            // last 5 deltas
const deltas: number[] = []
for (let i = 1; i < window.length; i++) {
  const delta = window[i].totalInputTokens - window[i - 1].totalInputTokens
  if (delta > 0) deltas.push(delta)       // ignore compaction drops
}
const avgGrowthPerTurn =
  Math.round(deltas.reduce((a, b) => a + b, 0) / deltas.length)

const remaining = MODEL_CONTEXT_LIMIT - latest.totalInputTokens
const turnsRemaining = remaining > 0 ? Math.floor(remaining / avgGrowthPerTurn) : 0
```

That `turnsRemaining` is the headline. Against a 200K window, an agent currently at 150K growing ~10K per turn has five turns of runway. The rolling window keeps the estimate responsive — it reflects what the agent is doing *now*, not an all-session average dragged down by a quiet warm-up.

## Turns are honest; hours are a bonus

Turns-remaining is the metric we trust, because it depends only on token math. But operators also want a wall-clock answer — *do I have an hour, or ten minutes?* — so we convert turns to time using the agent's recent pace.

The trap here is idle time. An autonomous agent might fire ten turns in two minutes, then sit for six hours waiting on a schedule. Fold those idle gaps into the average inter-turn interval and your time estimate becomes meaningless. So the interval average over the same recent window throws out any gap longer than an hour:

```ts
const dt = new Date(window[i].timestamp).getTime()
         - new Date(window[i - 1].timestamp).getTime()
if (dt > 0 && dt < 3_600_000) intervals.push(dt) // ignore idle gaps > 1h
```

Multiply the surviving average interval by `turnsRemaining` and you get an estimated hours-remaining — explicitly a secondary, best-effort figure. When there's no clean interval data, it stays `null` rather than inventing a number. The same discipline runs through the whole route: no growth data means `avgGrowthPerTurn` is `null` and the agent is reported as `unknown`, not falsely healthy. An honest "I don't know" beats a confident wrong answer every time someone's going to act on it.

## The fleet view

With a runway per agent, the dashboard becomes a triage list. Projects are sorted by turns-remaining ascending — the agent closest to the wall is at the top — and bucketed by a simple threshold: **critical under 5 turns, warning under 10, ok beyond.** Each row is a horizontal runway bar, and a "Needs Reset" panel collects the agents that need a fresh session before they degrade. The header carries the only two counts an operator scanning the fleet needs: how many critical, how many warning.

## What it changed

This didn't make context windows bigger or agents more efficient. What it bought is *lead time*. Before, an agent hitting its limit was discovered after the fact, by reading a transcript and noticing the conversation had been reset under it. Now it's a bar climbing toward red with a number attached — enough warning to rotate the session deliberately instead of letting the harness do it abruptly mid-task.

Two ideas here transfer to anyone running LLM agents at scale, regardless of stack. First: **measure the context you'll actually be charged for and bounded by — cache reads included — not just the uncached input tokens**, or you'll wildly underestimate how full the window is. Second: **a position is not a forecast.** A "78% used" gauge feels like monitoring, but it can't tell you whether you have an hour or thirty seconds. Take the derivative — growth per turn over a short rolling window — and divide. The runway it gives you is the difference between reacting to an outage and preventing one.

---

*Originally published at bistecglobal.com*

**BistecGlobal** builds AI-native enterprise software. Follow us for engineering insights.
