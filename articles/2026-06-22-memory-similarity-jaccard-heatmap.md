---
title: "What Your AI Agents Know in Common: Mapping Fleet Memory with a Jaccard Heatmap"
project: claude-mcd
tags: [AI, Agents, Observability, DevOps, Engineering]
status: audited
date: 2026-06-22
---

# What Your AI Agents Know in Common: Mapping Fleet Memory with a Jaccard Heatmap

Six months into running a fleet of AI agents across a dozen projects, we noticed a peculiar problem. Each agent was building up its own long-term memory — preferences, patterns, codebase conventions, recurring problems — stored as structured markdown files that persisted across sessions. But we had no way to see what these agents *shared*. Were our keyflow and agent-nexus agents learning the same lessons about authentication patterns? Were projects that seemed unrelated actually accumulating knowledge in the same domains? We couldn't tell, and that invisibility was quietly costing us.

This is the story of how we built a cross-project memory similarity heatmap using Jaccard similarity to answer one deceptively simple question: *which of our AI agents think alike?*

## The Problem with Siloed Agent Memory

Agent memory in our fleet is stored in per-project directories as a collection of markdown files — user preferences, feedback patterns, project-specific conventions, architectural decisions. Each agent writes and reads its own memory independently, growing a richer context over weeks of operation.

The benefit is obvious: agents get smarter about their specific domain. The hidden cost is coordination opacity. If keyflow's agent has learned that "JWT tokens should always be short-lived" and agent-nexus independently learned the same lesson, we're duplicating effort and missing the signal that this is a cross-cutting concern worth centralizing. Conversely, if two agents share heavy vocabulary overlap around "authentication" but approach it differently, that's a divergence worth investigating.

You can't discover this by reading individual memory files. With 19 projects each accumulating their own memory files, manual comparison is intractable. We needed an automated lens.

## Jaccard Similarity as the Right Tool

The core question — how much do two sets of knowledge overlap? — maps naturally to Jaccard similarity: the ratio of the intersection to the union of two sets.

```
J(A, B) = |A ∩ B| / |A ∪ B|
```

We extract keyword sets from each project's memory files: lowercase the text, strip punctuation, remove stop words and short tokens (under 4 characters), then count word frequencies. From these frequency maps we compute Jaccard similarity between every project pair, keeping track of the top-5 shared keywords by combined frequency.

```typescript
function jaccardSimilarity(
  a: Map<string, number>,
  b: Map<string, number>
): { score: number; shared: string[] } {
  const aSet = new Set(a.keys())
  const bSet = new Set(b.keys())

  const intersection: string[] = []
  for (const word of aSet) {
    if (bSet.has(word)) intersection.push(word)
  }

  const unionSize = aSet.size + bSet.size - intersection.length
  if (unionSize === 0) return { score: 0, shared: [] }

  const score = intersection.length / unionSize

  const topShared = intersection
    .sort((x, y) => (b.get(y)! + a.get(y)!) - (b.get(x)! + a.get(x)!))
    .slice(0, 5)

  return { score: Math.round(score * 1000) / 1000, shared: topShared }
}
```

The result is an N×N score matrix — every project pair gets a similarity percentage between 0 and 100%.

## Building the Heatmap

The matrix is exposed through a Next.js API route that reads each project's memory directory. The path resolution is the trickiest part: project directories may be symlinks, and Claude Code encodes the real path into the memory directory name, so we have to resolve symlinks before constructing the memory path.

```typescript
function readMemoryFiles(slug: string, mcdDir: string): string {
  const projectPath = path.join(mcdDir, 'projects', slug)
  let realPath = projectPath
  try { realPath = fs.realpathSync(projectPath) } catch { /* no symlink */ }
  const encoded = encodeProjectCwd(realPath)
  const memoryDir = path.join(os.homedir(), '.claude', 'projects', encoded, 'memory')

  let combined = ''
  // Read per-session memory files
  const files = fs.readdirSync(memoryDir).filter((f) => f.endsWith('.md'))
  for (const file of files) {
    combined += ' ' + fs.readFileSync(path.join(memoryDir, file), 'utf-8')
  }
  // Also include the project's MEMORY.md index if present
  combined += ' ' + fs.readFileSync(path.join(mcdDir, 'projects', slug, 'MEMORY.md'), 'utf-8')
  return combined
}
```

The API computes the full matrix on first request, then caches the result in-process for 5 minutes. This is the right trade-off: memory files change at session boundaries (every few hours), so sub-minute freshness provides no value.

Once projects are scored, we sort them by average similarity in descending order — a greedy cluster sort that puts the most connected projects together along both axes, making clusters visually obvious without requiring explicit clustering algorithms.

## The Visualization

The frontend renders the matrix as a CSS grid where each cell's color maps the score from dark (near zero) to bright cyan (near 100%). We apply a power curve to the color ramp so low-similarity differences are still visible and not washed out at the dark end.

A threshold slider (0–50%) grays out cells below the chosen cutoff, letting operators focus on meaningful overlaps. At a 15% threshold you see the clearly related projects; at 5% you see even faint connections. Cell size adapts automatically based on fleet size — 52px cells for 8 or fewer projects, down to 26px for 16+.

Hovering any non-diagonal cell opens a tooltip with the two project slugs, the similarity percentage, and the top-5 shared keywords ordered by combined frequency. That last piece is what makes the heatmap actionable rather than just interesting. Knowing that keyflow and agent-nexus share 22% similarity is data; knowing that their shared keywords are `authentication`, `budget`, `session`, `token`, and `agent` is insight.

## What We Discovered

When we first ran the heatmap across our fleet, the results validated some expectations and surfaced genuine surprises.

Expected: the two most similar projects were our internal developer tooling agents — they'd both accumulated conventions about TypeScript style, testing patterns, and git hygiene. Their shared keywords reflected a shared codebase culture.

Surprising: two product-facing projects from completely different business domains showed meaningful overlap — around the 15–20% range. The shared keywords were almost entirely operational: terms around deployment, token management, timeouts, and retries. Both agents had independently learned the same hard lessons about distributed system reliability — not from each other, but from hitting the same failures. That's a signal worth acting on: these operational patterns deserve a shared memory layer, not parallel reinvention.

We also found a cluster of projects with near-zero similarity across the board — agents that had accumulated mostly domain-specific vocabulary with minimal overlap. These are probably fine; they're genuinely different problem spaces. But it's useful to confirm that rather than assume it.

## Design Decisions Worth Noting

**Keyword-based Jaccard over embedding similarity.** We considered using vector embeddings and cosine similarity for richer semantic matching. We chose Jaccard on keywords for two reasons: it's transparent (the shared keywords explain the score directly), and it runs without any external model call. An embedding-based approach would require either an API call per project pair or a local model, adding latency and cost to what should be a lightweight diagnostic. Jaccard's interpretability wins for an operational tool.

**No global state in the keyword computation.** Each project's keyword map is built independently before any pair-wise comparison. This means you can add a new project and only need to compute N new pairs rather than recomputing the full matrix from scratch — the design keeps per-project state separate by construction.

**5-minute cache at the API layer.** Memory files change at session boundaries, not continuously. A 5-minute cache eliminates redundant recomputation without serving stale data that matters. We cache in the Next.js process rather than in Redis because the dashboard is single-tenant; a shared cache would add complexity without benefit.

## What the Heatmap Teaches You to Ask

The heatmap is a diagnostic, not a control plane. Its value is in the questions it forces: Why do these two agents share vocabulary? Is that overlap from shared infrastructure, shared problems, or just coincidence? Is the divergence between these others a sign of healthy specialization or of agents that failed to learn from each other?

For any team running more than a handful of AI agents long-term, the question of what your agents collectively know — and where they diverge — turns out to be as important as what any individual agent knows. A Jaccard heatmap over memory files is a surprisingly affordable way to start answering it. The core algorithm — keyword extraction, pairwise Jaccard, greedy cluster sort — fits in a single API route file alongside the React visualization layer.
