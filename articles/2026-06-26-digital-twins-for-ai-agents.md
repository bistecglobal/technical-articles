---
title: "Digital Twins for AI Agents: Finding the Project That Behaves Just Like This One"
project: claude-mcd
tags: [AI, Agents, Observability, MachineLearning, Claude]
status: draft
date: 2026-06-26
---

When you run twenty autonomous AI agents at once, they stop being individuals and start being a population. And populations have a useful property: members resemble each other. The agent that's quietly burning through its context window at 2am almost certainly has a twin three projects over — same churn, same tool-call rhythm, same memory footprint — that hit the same wall last week. If you could find that twin automatically, triage would stop being twenty separate investigations and become "oh, it's *that* pattern again."

That's the idea behind Project Twin Analysis, a view we added to Mission Control, the dashboard for our multi-channel Discord agent fleet (MCD). It reduces each agent to a five-number behavioural fingerprint, then computes which agents are most alike — a nearest-neighbour lookup for an entire fleet, built entirely from logs we already keep.

## The problem: similarity is in the eye, not the data

Operators already form intuitions like "these two projects feel similar." But that intuition lives in their head, doesn't scale past a handful of projects, and evaporates when the operator changes. There's no artifact that says *project A behaves like project B, and here's why*.

The raw material for that artifact exists — every agent leaves a detailed JSONL transcript and a memory directory — but it's the wrong shape. A transcript is a time series of events; "similarity" is a property of *pairs* of agents. To get from one to the other you need to collapse each agent to a comparable vector, and then compare vectors.

## Five numbers that describe an agent

The twins endpoint distills each project, over a rolling 7-day window, into a five-feature vector:

- **turns_per_day** — how busy the agent is (genuine user messages per day)
- **tool_call_rate** — tool calls per user message: how tool-heavy its work is
- **memory_file_count** — how much persistent knowledge it has accumulated
- **context_pressure_pct** — how full its most recent session's context window is
- **avg_tokens_per_turn** — how verbose each assistant turn runs

Each is reconstructed from the transcript. Token-based features read the `usage` block off assistant messages; activity features count messages and tool-use blocks. As in our other transcript views, a tool result is recorded with `role: "user"`, so the "genuine message" filter excludes those to avoid double-counting tool round-trips as human activity:

```ts
if (role === 'user' && content.length > 0 && content[0]?.type !== 'tool_result') {
  userMessages++
}
if (role === 'assistant') {
  if (content.some(c => c.type === 'tool_use')) toolCalls++
  // ...accumulate usage tokens, count turns
}
```

Context pressure is estimated from the *most recently modified* transcript file — the agent's live session — measured against a 200k-token model window. The others aggregate across the whole window.

## Normalise first, or one feature eats the rest

Here's the subtle part. These five numbers live on wildly different scales: `avg_tokens_per_turn` is in the tens of thousands, `tool_call_rate` is around 1, `memory_file_count` is single digits. Run a raw similarity on those and the token count drowns out everything else — every comparison would really just be "do these agents use a similar number of tokens?"

So before comparing, each feature is min-max normalised *across the fleet* to a 0–1 range:

```ts
function normalize(values: number[]): number[] {
  const min = Math.min(...values)
  const max = Math.max(...values)
  if (max === min) return values.map(() => 0)
  return values.map(v => (v - min) / (max - min))
}
```

Normalisation is done per feature, across all projects — so each number becomes "where does this agent sit relative to the rest of the fleet on this dimension?" That reframing is what makes the comparison meaningful: twins are agents that occupy similar *positions* in the fleet, not agents that happen to share an absolute magnitude.

## Cosine similarity, and explaining the match

With every agent reduced to a normalised five-vector, similarity is a textbook cosine between each pair:

```ts
function cosine(a: number[], b: number[]): number {
  let dot = 0, magA = 0, magB = 0
  for (let i = 0; i < a.length; i++) {
    dot += a[i]! * b[i]!
    magA += a[i]! * a[i]!
    magB += b[i]! * b[i]!
  }
  if (magA === 0 || magB === 0) return 0
  return dot / (Math.sqrt(magA) * Math.sqrt(magB))
}
```

Cosine measures the *angle* between vectors — the shape of an agent's behaviour — rather than how far apart they sit. Two agents can differ in overall intensity yet still be twins if their profile across the five dimensions points the same way.

Only pairs scoring 0.80 or higher are kept and surfaced, sorted strongest-first. A raw similarity score, though, is a number without a reason. So each twin pair is also tagged with its *shared features* — the dimensions where the two agents sit within 0.2 of each other on the normalised scale:

```ts
const sharedFeatures = FEATURE_NAMES.filter(f => {
  const diff = Math.abs(projects[i].normalized[f] - projects[j].normalized[f])
  return diff <= 0.2
})
```

Now the result reads like an explanation, not an oracle: *"these two are 0.94 similar, and they share turns_per_day, tool_call_rate, and context_pressure."* The frontend renders this three ways — a similarity matrix, a ranked list of the strongest pairs, and a side-by-side bar comparison of the two agents' feature profiles.

## What we took away

Two lessons generalise beyond this one feature.

First, **the hard part of "find similar things" is rarely the math — it's the feature design and the normalisation.** Cosine similarity is ten lines anyone can write. Choosing five features that actually characterise an agent, and normalising them so no single scale dominates, is where the signal comes from. Skip the normalisation and you get a confident, precise, useless answer.

Second, **a similarity score needs to carry its reasoning.** A bare "0.94" invites blind trust; a "0.94, sharing these three dimensions" invites a judgement. For an observability tool whose whole job is to help a human triage faster, the shared-feature tags matter as much as the score itself — they turn a black box into a starting point for investigation.

The whole thing rides on data we were already keeping. No new metrics pipeline, no agent changes — just a different lens on the transcripts, asking not "what is this agent doing?" but "which other agent is it secretly the same as?"
