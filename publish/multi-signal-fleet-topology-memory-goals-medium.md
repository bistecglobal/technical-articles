# Your AI Agent Topology Was Missing Two Signals: Memory and Goals

*How we extended MCD's live fleet graph to surface semantic relationships between agents — not just who's talking, but what they remember and where they're headed.*

When you run a fleet of AI agents in parallel, your first instinct is to watch who's talking to whom. Draw a line when agent A mentions agent B. That's a reasonable start — but it misses something important: two agents sharing almost identical memories and working toward the same stated goal show no edge at all, until one of them happens to drop the other's project slug into a transcript.

That blind spot compounded a second problem: checking each agent's health required clicking into individual project pages. Across fifteen-plus projects, that's fifteen clicks before you even know which one deserves attention.

We fixed both issues in MCD's latest dashboard update.

## The Problem With Transcript-Only Topology

MCD's fleet dashboard had a live D3 topology view for months: a force-directed graph where agents appear as nodes and edges form when one agent's transcript references another's project slug within the last fifteen minutes. Useful for detecting active cross-project coordination. Not useful for answering a different question: *which agents are operating in similar cognitive territory*, even if they've never explicitly mentioned each other?

Every MCD project carries two persistent context files: `MEMORY.md` (accumulated learnings retained across sessions) and `GOAL.md` (the current mission statement, set by the operator). These files often contain overlapping vocabulary — authentication, token-budget, tenant routing — without the agents ever cross-referencing each other. They're working in parallel on related problems, and the topology is silent about it.

## Four Signals, One Edge Weight

We extended the topology to combine four signal types per edge:

1. **Transcript** — how often agents reference each other by slug in the last 15 minutes, normalized over a cap of 10 references
2. **Memory** — Jaccard similarity between keyword sets extracted from each project's `MEMORY.md`
3. **Goal** — Jaccard similarity between keyword sets extracted from each project's `GOAL.md`
4. **Shared remote** — both projects point to the same git remote; a hard signal that always produces a weight of 1.0

The final edge weight:

```typescript
const weight = sameRemote ? 1.0 :
  0.5 * transcriptScore + 0.3 * memSim.score + 0.2 * goalSim.score
```

Transcript gets the highest weight because an active cross-reference is the strongest evidence of coordination. Memory is next — shared learned context suggests agents have encountered the same problems. Goals contribute least since goal keywords tend to be broader and less discriminating.

## Jaccard Similarity at Runtime

Keywords are extracted by stripping punctuation, splitting on whitespace, filtering to tokens of four or more characters, and removing a custom stop-word list. The stop list goes beyond standard English words to exclude terms that would create false overlap across every agent pair: "project", "claude", "code", "file", "there", "just". What remains is domain vocabulary — the terms that actually distinguish what an agent is working on.

The similarity itself:

```typescript
function jaccardSimilarity(a: Set<string>, b: Set<string>): { score: number; shared: string[] } {
  if (a.size === 0 && b.size === 0) return { score: 0, shared: [] }
  const shared: string[] = []
  for (const w of a) { if (b.has(w)) shared.push(w) }
  const union = new Set([...a, ...b]).size
  return { score: union === 0 ? 0 : shared.length / union, shared: shared.slice(0, 5) }
}
```

An edge is created when memory or goal Jaccard exceeds 0.05, meaning at least 5% of the combined keyword vocabulary is shared. Below that threshold most connections involve generic infrastructure terms rather than meaningful conceptual overlap.

The edge type records its origin: `transcript` for active cross-references, `inferred` for memory/goal overlap without any recent transcript mention, and `shared-remote` for same-repository projects. Inferred edges render dashed in the graph — a deliberate visual distinction that tells the operator: this connection is semantic, not observed in real-time communication.

Shared-remote edges render as solid red lines, width 3, immediately separating monorepo subprojects from agents that merely share vocabulary.

The up-to-five shared keywords that caused the edge are stored in the edge's `title` attribute, visible on hover: hovering over a dashed edge might show "Shared: tenant, token, auth, rotation, gateway" — a fast explanation of why two seemingly unrelated projects are linked.

## Fleet Digest: The 30-Second Health Check

The topology answers *how are agents related*. Operators also needed a fast answer to *how is each agent doing right now* — without clicking into fifteen individual project pages.

The Fleet Digest aggregates five dimensions per project into a single table:

- **Context pressure** — percentage of the context window consumed, from the existing context-pressure table
- **Convergence score** — 0–100 measure of whether recent turns are making forward progress
- **Goal advancement** — percentage of the `GOAL.md` checklist completed
- **Alert count** — recent alert events from the current session
- **Turns today** — assistant turns since midnight, counted directly from JSONL transcripts

Projects are automatically flagged when they cross defined thresholds:

```typescript
if (contextPct >= 90) flags.push('context-critical')
else if (contextPct >= 70) flags.push('context-warning')
if (convergence !== null && convergence < 30) flags.push('low-convergence')
if (goalPct !== null && goalPct < 20) flags.push('goal-drift')
if (isStuck) flags.push('stuck')
```

The `stuck` flag deserves a note: it scans the last 24 hours of JSONL transcripts for the word "stuck" in any context. It's a coarse heuristic, but agents consistently use the word when reporting that they're looping or blocked, which makes the scan surprisingly reliable. False positives exist, but they're cheap: a flag prompts a 5-second glance at the session health page, not an intervention.

Each digest run persists in a `digest_log` SQLite table. The UI exposes three tabs: the live project table with color-coded flag chips, a history view of the last 30 digest runs, and a markdown export:

```
# Fleet Digest — 2026-06-22

**16 projects** · 4 alerts · 1 stuck signals

## Project Health
- **agent-nexus**: ctx 45% · cnv 72 · goal 80% · 12 turns ✓
- **keyflow**: ctx 91% · cnv 28 · goal 15% · 3 turns ⚠ context-critical, low-convergence, goal-drift
```

This markdown output is immediately portable. It can be included in operator standup notes, pasted into a Discord message, or fed back to a supervising agent as a fleet status prompt.

## What Changes in Practice

When the expanded topology first loads, inferred edges surface project pairs that had no prior transcript-based connection. Two agents working on adjacent problems — say, different layers of the same authentication stack — can accumulate overlapping memory keywords without ever explicitly referencing each other. The dashed edge between them becomes a prompt to check whether they could share an abstraction, a question that transcript-only topology would never raise.

The Fleet Digest reduces a multi-step health check ritual to a single page load. The typical workflow: open the digest, scan for red flags, confirm which project has the highest context pressure or lowest convergence, then click through to the session health view for that project.

## Three Things Worth Keeping

**Transcript edges are trailing indicators.** Agents have to explicitly mention each other before a transcript edge forms. Memory and goal overlap are earlier signals — they surface shared context before anyone has coordinated. In a fleet where agents work autonomously, that lag matters.

**Jaccard threshold tuning is load-bearing.** The 0.05 floor eliminates most noise from shared infrastructure vocabulary. Drop it to 0.02 and the graph becomes a hairball; raise it to 0.15 and real connections between adjacent projects disappear. Start at 0.05 and watch which edges appear on your specific fleet.

**Aggregate before you drill.** A dashboard that forces per-project clicks punishes operators who are already managing high cognitive load. The digest pattern — compute once, display everything, flag automatically — cuts the noise before anyone has to make a decision.

---

*Originally published at bistecglobal.com*

---

**BistecGlobal** builds AI-native enterprise software. Follow us for engineering insights.
