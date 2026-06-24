---
title: "One Attention Engine, Two Faces: Killing 500 Lines of Duplicated Fleet Logic"
project: claude-mcd
tags: [AI, Observability, Software Design, Refactoring, TypeScript]
status: audited
date: 2026-06-24
---

We had two features that answered the same question in two different voices — and, inevitably, two different answers.

Mission Control watches a fleet of autonomous Claude agents. Two of its surfaces exist to tell an operator *what needs attention right now*. The **Fleet Advisor** does it as a stack of actionable cards: "Context 96% full — inject a compaction prompt." The **Fleet Brief** does it as a narrative digest: "alpha's circuit breaker is open; beta's memory is 18 days stale." Same underlying reality, two presentations. Reasonable enough — until you look at how they were built.

Each surface had grown its own copy of the rules. Each had its own transcript reader, its own budget calculator, its own stall thresholds. And the thresholds had drifted. The Advisor flagged stalls between 30 minutes and 4 hours; the Brief flagged them between 4 and 48 hours. The Advisor knew about thrashing and declining convergence; the Brief didn't. The Brief knew about circuit breakers and context pressure; the Advisor's copy lagged. Two features looking at one fleet, disagreeing about what mattered, because the logic that decided "what matters" had been written twice.

This is the story of collapsing them into a single rule engine — and why the shape of that engine matters more than the line count it saved.

## The core idea: a finding is a fact, not a card

The mistake baked into the old design was presentational thinking. The Advisor code computed *advisor cards*. The Brief code computed *brief messages*. The domain fact — "this project's context window is 96% full and that's a problem" — never existed as its own thing. It was born already dressed for one specific UI.

The fix was to name the fact. We defined a single `Finding` type that carries everything *either* surface could want:

```ts
export interface Finding {
  id: string
  slug: string
  severity: Severity              // 'critical' | 'warn' | 'info' | 'ok'
  signal: SignalKey               // 'circuit' | 'context' | 'stall' | ...
  title: string                   // advisor headline
  explanation: string             // advisor detail
  message: string                 // brief one-liner
  href: string                    // deep-link
  action?: { type: ActionType; payload: string } // present → it's actionable
}
```

A `Finding` is provider-agnostic. It knows the project, the severity, which signal tripped, and how to phrase itself for both a card (`title` + `explanation`) and a one-line digest (`message`). Crucially, the optional `action` is what distinguishes "something you can do" from "something you should know" — and that single field is what lets one rule set feed two surfaces without either one second-guessing the other.

## One registry, edited in one place

Every rule is a pure function from signals to an optional finding. They live in a single exported array, and the comment above it is the whole design philosophy in one line: *edit HERE to add a signal.*

```ts
export const RULES: Rule[] = [
  // Circuit breaker open.
  (s) => s.circuitOpen ? {
    id: `circuit-${s.slug}`, slug: s.slug, severity: 'critical', signal: 'circuit',
    title: `Circuit open: ${s.slug}`,
    explanation: `${s.slug} has failed repeatedly and its circuit breaker is open...`,
    message: `${s.slug}: circuit breaker open — failing repeatedly`,
    href: `/focus/${s.slug}`,
    action: { type: 'command', payload: `!project stop ${s.slug}` },
  } : null,

  // Context window pressure.
  (s) => {
    if (s.contextPct == null) return null
    if (s.contextPct >= 87) return { /* warn/critical inject ... */ }
    if (s.contextPct >= 80) return { /* info inject ... */ }
    return null
  },
  // stall, idle, memory, budget, thrashing, declining, alerts ...
]
```

The unification forced the thresholds to reconcile. The two divergent stall windows — 30m–4h and 4h–48h — merged into one rule covering the whole 30-minute-to-48-hour range, with severity escalating inside it. The Advisor *gained* thrashing and declining-convergence coverage; the Brief *gained* circuit, context, memory, and budget. There's now exactly one definition of "what a stall is," so the two surfaces can no longer disagree — the disagreement was never intentional, just two copies decaying independently.

Adding a signal used to mean editing two files and remembering to keep their thresholds in sync. Now it's one array entry, and both surfaces pick it up for free.

## Read the signals once, run the rules many

The other half of the duplication was in the *reading*. Both old surfaces independently walked each project's JSONL transcripts for mtime, scanned for the latest input-token count, stat-ed `MEMORY.md`, and summed monthly spend. The new engine gathers every project's signals exactly once into a flat `FleetSignals` struct, then runs all rules against it:

```ts
for (const { chatId, slug, monthlyTokenBudget } of entries) {
  const s: FleetSignals = { slug, chatId, ageMs, contextPct, churn, convDelta,
    highChurn, openAlerts, memAgeDays, monthlyUsed, circuitOpen, /* ... */ }

  const before = out.length
  for (const rule of RULES) {
    const f = rule(s, ctx)
    if (f) out.push(f)
  }
  // "Healthy" is derived only when no issue rule fired for this project.
  if (out.length === before) {
    const h = healthyFinding(s)
    if (h) out.push(h)
  }
}
out.sort((a, b) => sevOrder(a.severity) - sevOrder(b.severity) || a.slug.localeCompare(b.slug))
```

Note the "healthy" handling: it isn't a rule in the registry, because a green light is the *absence* of any red one. We only emit it when no issue rule fired for that project — derived state, not asserted state. The final sort is severity-first, so both surfaces consume findings already ordered by urgency.

The two surfaces are now thin adapters over the same `Finding[]`. `toAdvisorCards()` keeps only findings that carry an `action` and takes the top N by severity. `toBriefResult()` maps every finding to its `message` and collapses to an all-nominal state when nothing fired. Neither contains a shred of detection logic.

## A pure core you can actually test

A subtle payoff: the rule registry is pure. It imports `fs`, `path`, and `os` for the signal *readers*, but the SQLite handle — `better-sqlite3`, a native addon — is dynamically imported *inside* the gather function, not at module top level. That one move keeps `RULES`, the adapters, and the `Finding`/`FleetSignals` types importable under Bun's test runner, which can't load the native addon.

So the tests run the real rules against hand-built signal fixtures — no database, no filesystem, no fleet:

```ts
test('a finding with an action surfaces in BOTH advisor and brief', () => {
  const found = runRules(signals({ contextPct: 96 }))
  const ctx = found.find((f) => f.signal === 'context')
  expect(ctx).toBeDefined()
  const cards = toAdvisorCards(found)
  // ...asserts the same finding appears in both surfaces
})
```

That test encodes the entire reason the refactor exists: an actionable finding must appear in *both* surfaces, because there is now only one source. The old design couldn't have had this test — there was no shared thing to assert about.

## What it cost and what it bought

The change removed the per-route rule logic from both endpoints and deleted an earlier interim extraction that had only partially shared the brief logic — a net reduction of around 500 lines of duplicated readers and rules, replaced by one ~450-line module plus its tests. But the line count is the least interesting outcome.

The real win is that "what deserves attention" is now defined exactly once, in a registry an engineer can read top to bottom in a minute, exercised by tests that need no infrastructure. Two takeaways generalise:

- **Name the domain fact before the presentation.** The duplication wasn't a copy-paste accident; it was the direct consequence of computing *cards* and *messages* instead of *findings*. Once the fact had a type, the two surfaces became adapters, and adapters don't drift.
- **Keep the decision core free of native dependencies.** Dynamically importing the database inside the one impure function — instead of at the module top — is what made the rule set unit-testable at all. Purity wasn't an aesthetic; it was the thing that let the test suite exist.

Two features that quietly disagreed about your fleet's health is a bug you can't see in any single screenshot. The fix wasn't more logic — it was one copy of it.
