---
title: "The Agent That Did the Work But Forgot to Write It Down"
project: claude-mcd
tags: [AI, Observability, DevOps, Developer Tooling, AI Agents]
status: draft
date: 2026-06-26
---

An AI agent can run for weeks, ship dozens of pull requests, and learn a hundred hard-won lessons about a codebase — and still wake up tomorrow knowing none of it. The work happened. The memory didn't.

That gap is invisible on most dashboards. An agent that's busy *looks* healthy: turns are flowing, tokens are burning, PRs are merging. But if it never persists what it learned, every context reset wipes the slate clean and the next session re-discovers the same things from scratch. High activity, thin memory — the most expensive failure mode there is, because it hides behind the appearance of productivity.

Our Mission Control platform (the dashboard behind the Multi-Channel Discord fleet, MCD) now has a page built specifically to catch it: the **Memory Coverage Gap Detector**.

## The convention that makes gaps measurable

You can't detect a gap without first agreeing on what "complete" looks like. MCD agents follow a structured memory convention: every persisted fact is a single Markdown file in a per-project `memory/` directory, and every file declares one of four canonical types.

- **user** — who the operator is: role, expertise, preferences.
- **feedback** — guidance the agent was given on how to work, including the *why*.
- **project** — ongoing goals and constraints not derivable from the code or git history.
- **reference** — pointers to external resources: URLs, dashboards, tickets.

This isn't bureaucracy for its own sake. Each type answers a different question the agent will face after its context resets. An agent with zero `feedback` files has no record of corrections it's received. An agent with zero `project` files has forgotten *why* it's doing what it's doing. The four types are a checklist for "has this agent written down the things it will need to remember."

Because the convention is encoded in the filename — files are named `<type>_<slug>.md` or simply `<type>.md` — coverage becomes something you can compute by reading a directory listing. No parsing, no embeddings, no model call. Just a prefix match.

## A directory listing, classified

The detector's API route is deliberately boring, and that's the point. It walks the projects directory, resolves each project's real path (following symlinks), and classifies the contents of its `memory/` folder:

```ts
const MEMORY_TYPES = ['user', 'feedback', 'project', 'reference'] as const

const files = fs.readdirSync(memDir).filter((f) => f.endsWith('.md'))
for (const f of files) {
  for (const t of MEMORY_TYPES) {
    if (f.startsWith(`${t}_`) || f === `${t}.md`) typeCounts[t]++
  }
}

const missingTypes = MEMORY_TYPES.filter((t) => typeCounts[t] === 0)
```

The whole analysis is a `readdirSync` and a prefix check. A project with no `memory/` directory at all is treated as maximally gapped — it gets all four types marked missing, plus one extra penalty point for having no memory store whatsoever:

```ts
gapCount: missingTypes.length + (hasMemoryDir ? 0 : 1)
```

So `gapCount` runs from 0 (fully covered) to 5 (no memory directory at all). That single integer is what the table sorts on, worst-first.

## Activity is the other half of the signal

A gap only matters if the agent is doing something. A dormant project with an empty `memory/` folder isn't a problem — it just hasn't started. The danger is the *active* agent with thin memory.

So the detector cross-references each project against its Claude Code transcripts. An agent is "active" if it has a JSONL transcript modified within the last seven days:

```ts
function isActiveProject(projectDir: string): boolean {
  const sevenDaysMs = 7 * 24 * 3_600_000
  const encoded = projectDir.replace(/[^a-zA-Z0-9]/g, '-')
  const transcriptDir = path.join(os.homedir(), '.claude', 'projects', encoded)
  const files = fs.readdirSync(transcriptDir).filter((f) => f.endsWith('.jsonl'))
  if (files.length === 0) return false
  const mtimes = files.map((f) => fs.statSync(path.join(transcriptDir, f)).mtimeMs)
  return (Date.now() - Math.max(...mtimes)) < sevenDaysMs
}
```

The transcript directory name is derived by encoding the project's real path — Claude Code stores session history under a slugified version of the working directory, so the detector reconstructs that name to find the right transcripts. The page surfaces activity as a filled green dot, dims inactive rows to 55% opacity, and offers an "Active projects only" toggle so an operator can collapse the table down to exactly the agents that are working *and* under-documenting.

## Reading the table

The page renders one row per project: a column per memory type (green check or red cross), a memory-directory check, total file count, and the gap count. The gap column is color-coded so the eye lands on trouble first — green at zero, amber for one to three missing types, red for four or more:

```ts
color: p.gapCount === 0 ? '#22C55E'
     : p.gapCount >= 4 ? '#EF4444'
     : '#F59E0B'
```

Every column header is a sort button. Default sort is by gap count descending, but an operator can resort by any single memory type to answer targeted questions — "which active agents have never recorded a single piece of feedback?" is one click on the `feedback` header. And when the fleet is in good shape, the table doesn't render at all; it's replaced by a single line: *All projects fully covered — no memory gaps detected.*

```
project        active  mem  user feed proj ref  files  gap
─────────────────────────────────────────────────────────
new-onboarding   ●      ✗    ✗    ✗    ✗    ✗     0      5
keyflow-okr      ●      ✓    ✓    ✗    ✗    ✓     6      2
specclaw         ●      ✓    ✓    ✓    ✓    ✓    14      0
```

## What it surfaced

The most useful thing the detector did on day one was contradict the rest of the dashboard. Several projects that scored well on activity metrics — plenty of turns, healthy token spend — lit up amber and red on memory coverage. They were doing the work and not writing it down. That's precisely the agent that will burn the same tokens re-learning the same lessons after its next context reset, and it's invisible to any metric that only counts throughput.

The detector reframes a fuzzy worry ("are our agents actually accumulating knowledge?") as a sortable integer per project. It turns "we should probably check on memory hygiene sometime" into a red number at the top of a table.

## Lessons worth stealing

**Encode your conventions in something machine-readable.** The entire feature is possible only because memory files carry their type in the filename. A naming convention you can `grep` is a convention you can measure, audit, and build a dashboard on. If the type lived only inside the file, or only in someone's head, there'd be nothing to count.

**Pair every "is it happening" signal with a "does it matter" signal.** A gap count alone produces false alarms on dormant projects. Activity alone hides the agents that look busy but learn nothing. The detector is useful precisely because it joins the two — coverage *and* recency — and lets the operator filter on the intersection.

**The cheapest detector is often a directory listing.** No model call, no embeddings, no database. A `readdirSync`, a prefix match, and a file mtime were enough to surface a failure mode that the heavier observability machinery missed entirely. When you can compute the answer from the filesystem, do that first.

An agent's memory is its only continuity across resets. A page that tells you, at a glance, which agents are quietly losing theirs is worth far more than the hundred lines of code it took to build.
