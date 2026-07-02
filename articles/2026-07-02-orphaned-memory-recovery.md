---
title: "The Note Nobody Links To: Tracking Orphaned Memory Across an AI Agent Fleet"
project: claude-mcd
tags: [AI, Agents, Observability, DevOps, Knowledge-Management]
status: draft
date: 2026-07-02
---

Give an AI agent a persistent memory and it will start writing things down. Give it a *fleet* of persistent memories — one per project, dozens of them — and you get a quieter problem: notes that no longer connect to anything. A fact gets written, the thing it referenced gets renamed or deleted, and the note is left floating. It's still on disk. It's still loaded into context. But nothing points to it and it points to nothing. It's dead weight the agent keeps paying for.

We built a page in our mission-control dashboard to find those notes and watch what happens to them. This is how it works, and why "orphaned" turned out to be a more useful signal than "old."

## Memory as a graph, not a pile

Each agent in our fleet keeps its knowledge as a directory of small Markdown files — one fact per file. Files reference each other with wiki-style links: `[[some-other-note]]`. That convention is what turns a folder of notes into a graph. A well-connected memory looks like a web; a healthy fact is reachable from other facts and reaches out to them in turn.

An **orphan** is the opposite: a note with zero inbound links and zero outbound links. Nothing cites it, and it cites nothing. It is a disconnected island in the knowledge graph. That's the exact definition the detector uses — not "this file is old," not "this file is big," but structurally isolated.

The distinction matters. An old note that's still heavily linked is probably load-bearing. A brand-new note with no connections is more suspicious than a two-month-old one that half the memory points at. Age tells you how long something has sat; orphan-status tells you whether it's actually wired into anything.

## Building the graph on every scan

The `/api/memory-recovery` endpoint rebuilds the link graph from scratch each time it's called. It walks every project's `memory/` directory, reads each `.md` file, and extracts its outbound links with a single regular expression over the `[[...]]` syntax:

```ts
function extractLinks(content: string): string[] {
  return [...content.matchAll(/\[\[([^\]]+)\]\]/g)].map((m) => m[1]!.trim())
}
```

Then it resolves those links into a degree count. Each note is a node keyed by `project/file`. For every outbound link, it finds the target note — preferring a match inside the same project, falling back to a cross-project match — and increments the target's in-degree and the source's out-degree:

```ts
for (const node of nodes.values()) {
  let resolvedOut = 0
  for (const targetSlug of node.outSlugs) {
    const candidates = slugToIds.get(targetSlug) ?? []
    const sameProject = candidates.find((c) => c.startsWith(`${node.project}/`))
    const targetId = sameProject ?? candidates[0]
    if (!targetId || targetId === node.id) continue
    resolvedOut++
    inDegree.set(targetId, (inDegree.get(targetId) ?? 0) + 1)
  }
  outDegree.set(node.id, resolvedOut)
}
```

A note is flagged as an orphan only when both its in-degree and out-degree land on zero. Self-links don't count, and links pointing at a slug that no note actually owns don't count either — a dangling `[[link]]` to a note that was deleted doesn't rescue the file that contains it. That's deliberate: a note whose only connection is a broken pointer really is disconnected.

## Flagged is not the same as gone

The interesting design decision wasn't detection — it was refusing to act on it automatically. The endpoint never deletes a memory file. It only *tracks* one across a lifecycle.

State lives in a single `memory-recovery.json` file (written with `0600` permissions, since agent memory can hold sensitive context). Every record carries the note's id, the project, when it was `firstFlaggedAt`, and a resolution that starts as `null`:

```ts
export interface OrphanRecord {
  id: string
  project: string
  file: string
  firstFlaggedAt: string
  resolvedAt?: string
  resolution: 'relinked' | 'deleted' | 'ignored' | null
}
```

On each scan the endpoint reconciles this ledger with reality. New orphans get appended with a fresh `firstFlaggedAt` timestamp. Then it looks at every still-open record and asks a simple question: *is this note still an orphan?*

If a previously-flagged note has dropped off the current orphan list, something changed it for the better — so the endpoint auto-resolves it. It checks whether the file still exists on disk: if it does, the agent (or an operator) must have linked it back into the graph, so the resolution is `relinked`. If the file is gone, the resolution is `deleted`. The only manual action is `ignored`, set through a `POST` when an operator decides an isolated note is fine as-is.

```ts
const filePath = path.join(realPath, 'memory', file!)
const exists = fs.existsSync(filePath)
rec.resolution = exists ? 'relinked' : 'deleted'
rec.resolvedAt = nowIso
```

The result is a feedback loop with no destructive step in it. The system points at a problem and then records how the humans and agents actually resolved it — it never resolves it *for* them.

## What the page shows

The dashboard turns that ledger into three reads:

- **An open-orphan table**, sorted by how long each note has been orphaned. Anything past three days goes amber; past seven, red. The colour is the nudge — a note that's been disconnected for a week is either genuinely stale or a link somebody forgot to write.
- **An eight-week stacked bar chart** of resolutions — relinked, deleted, ignored, still-open — so you can see whether orphans are being cleaned up or just accumulating.
- **Per-project sparklines** of the open-orphan trend, ranked by current count, so the noisiest memories surface first.

None of this required a new database or a background job. The graph is cheap to rebuild, the ledger is one JSON file, and the whole thing runs on demand when you open the page.

## What we learned

The first surprise was how many orphans were *newly written* facts, not ancient ones. An agent would record something mid-session and never get around to linking it — the note was correct, just stranded. Age-based staleness checks would never have caught those, because the notes were hours old. Structural isolation caught them immediately.

The second was that "relinked" is the outcome you actually want, and it's only visible because the system waits instead of deleting. Had detection been wired straight to a delete, every one of those stranded-but-correct notes would have been thrown away. By treating an orphan as a *question* rather than a verdict, the resolution history became the real signal: a memory where orphans mostly end in `relinked` is a memory that's healing; one where they mostly end in `deleted` was accumulating genuine junk.

The broader lesson generalises past agent memory. When you build observability for an autonomous system, the temptation is to close the loop with automatic remediation. But the most useful thing a detector can do is often to *hold* a finding long enough to see how it gets resolved — and let the resolution pattern, not the detection, tell you whether anything is actually wrong.
