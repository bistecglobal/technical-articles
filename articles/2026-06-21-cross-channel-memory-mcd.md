---
title: "Cross-Channel Memory: Embedding-Based Shared Context for a Multi-Project AI Fleet"
project: claude-mcd
tags: [AI, MCP, SQLite, Embeddings, DevOps, Multi-Agent]
status: draft
date: 2026-06-21
---

# Cross-Channel Memory: Embedding-Based Shared Context for a Multi-Project AI Fleet

Running a fleet of AI agents across isolated Discord channels creates a clean separation of concerns — each project agent stays focused on its own codebase, its own sprint, its own problems. But that isolation comes at a cost: the orchestrator that coordinates all of them woke up every session with no memory of what it had done before.

We solved this by shipping a SQLite-backed, embedding-indexed memory store directly into the multi-channel-discord (MCD) bot, wired to the master orchestrator via four MCP tools. This is what that looks like in production.

## The Problem: Stateless Orchestration at Scale

MCD runs AI agents for over a dozen Bistec projects simultaneously — keyflow, specclaw, agent-nexus, and more — each in its own isolated Claude Code subprocess, mapped to a Discord channel. The master channel is the control plane: it runs heartbeat scans, injects continuation prompts into stalled agents, and coordinates cross-project state.

The problem became apparent as the fleet grew. Every time the master agent restarted — after a deployment, after a crash, at the start of a new session — it had no institutional memory. It couldn't answer: *Which channel stalled last Tuesday and why?* *Have we seen this pattern before?* *What did I inject into keyflow three heartbeat cycles ago?*

The heartbeat watchdog could detect stalls, but it couldn't accumulate intelligence about them. Each scan started cold.

## The Architecture: SQLite + Local Embeddings + MCP

The solution shipped in [PR #47](https://github.com/chan4lk/claude-multi-channel-discord/pull/47) (commit `02b3a83`, merged 2026-06-20) centers on three new files:

**`src/memory-store.ts`** — the core abstraction. A `MemoryStore` class wraps a Bun SQLite database at `$MCD_CHANNELS_DIR/memory.db` with this schema:

```sql
CREATE TABLE memories (
  id TEXT PRIMARY KEY,
  channel_slug TEXT,          -- null = global memory
  type TEXT NOT NULL,          -- channel_summary | decision | pattern | coordination | general
  content TEXT NOT NULL,
  embedding BLOB,              -- Float32Array serialized as BLOB
  created_at TEXT NOT NULL,
  last_accessed_at TEXT NOT NULL,
  access_count INTEGER NOT NULL DEFAULT 0
);
```

The embedding model is `Xenova/all-MiniLM-L6-v2` loaded via `@xenova/transformers`. Critically, the pipeline initialises asynchronously in the background at startup — bot startup time is not affected, and recall degrades gracefully to keyword LIKE search if the model hasn't finished loading yet.

**`src/memory-backup.ts`** — R2 durability. After each backup trigger (scheduled interval or `!project memory backup`), the store runs a WAL checkpoint (`PRAGMA wal_checkpoint(FULL)`) to flush in-flight writes before reading the raw `.db` file for upload. Two S3 `PutObjectCommand` calls go to R2: a timestamped copy (`memory-backups/memory-2026-06-20T12-04-10Z.db`) and an always-current `memory-backups/latest.db`.

**`src/master-mcp-server.ts`** — exposure as MCP tools. Four tools are registered, but only exposed to the master channel session — any non-master session calling them receives an authorization error. This keeps memory write access strictly in the hands of the orchestrator:

| Tool | Purpose |
|------|---------|
| `mcp__mcd__remember` | Save a memory with type and optional channel slug |
| `mcp__mcd__recall` | Semantic + keyword search across stored memories |
| `mcp__mcd__forget` | Delete a memory by ID |
| `mcp__mcd__memory_stats` | Count memories by type and channel |

## The Recall Strategy: Semantic First, Keyword Fallback

The `recall()` method implements a two-pass approach. The first pass runs a SQL `LIKE` query to fetch candidate rows:

```typescript
const rows = this.db.prepare(`
  SELECT id, channel_slug, type, content, embedding, ...
  FROM memories
  WHERE content LIKE '%' || ? || '%'
  ORDER BY last_accessed_at DESC
  LIMIT ?
`).all(query, limit)
```

If the embedding pipeline is ready and at least one candidate row has a stored embedding, a second pass re-ranks results using cosine similarity against the query embedding. The cosine computation is an inline loop over the Float32Array:

```typescript
private _cosine(a: Float32Array, b: Float32Array): number {
  let dot = 0, na = 0, nb = 0
  for (let i = 0; i < a.length; i++) {
    dot += a[i]*b[i]; na += a[i]*a[i]; nb += b[i]*b[i]
  }
  return na && nb ? dot / (Math.sqrt(na) * Math.sqrt(nb)) : 0
}
```

Memories without embeddings (stored before the pipeline was ready) fall into a secondary group, sorted by recency. Every retrieved memory updates `last_accessed_at` and increments `access_count`, creating a natural LRU signal without a separate frequency table.

## Five Memory Types for an Orchestrator's Needs

Rather than a flat key-value store, MCD memory models five distinct concepts — a taxonomy shaped by what the orchestrator actually needs to remember:

- **`channel_summary`** — current state of a project channel (what's in flight, any blockers)
- **`decision`** — engineering decisions that should not be re-derived (e.g. "keyflow uses `bun test`, not `jest`")
- **`pattern`** — cross-channel regularities observed over time (e.g. "channels stall on PR reviews on Fridays")
- **`coordination`** — records of autonomous actions taken (what was injected into which channel and why)
- **`general`** — catch-all for context that doesn't fit a category

The type system feeds the heartbeat template. The memory-aware heartbeat schedule injected into `templates/master.CLAUDE.md` instructs the agent to:

1. Recall prior context (`mcp__mcd__recall` with query `channel summary stall coordination`, limit 20) *before* running the scan
2. After each inject, save a `coordination` memory describing what was done and why
3. After the scan, save a `channel_summary` memory per channel

This creates a feedback loop: every heartbeat cycle deposits memories that make the next cycle more informed.

## Operator Controls

Memory operations are also available as operator commands from the master Discord channel (`src/master-commands.ts`):

```
!project memory stats                        — counts by type and channel
!project memory backup                       — immediate R2 upload
!project memory clear [--slug S] [--type T] --yes
```

`memory clear` uses `recall('')` with the LIKE wildcard path to list all entries, then calls `forget()` for each matching record.

## Testing and Verification

The implementation ships with 13 unit tests in `src/memory-store.test.ts` covering remember/recall, forget, stats, slug filtering, null-slug (global) entries, and first-run behavior (no db file). All tests passed against the acceptance criteria documented in the specclaw spec (8/8 ACs verified in `.specclaw/changes/mcd-memory-integration/verify-report.md`).

One known non-blocking gap documented in the PR: LIKE wildcards (`%`, `_`) in recall queries are not escaped. For the current operator-only usage this is not a security concern, and a follow-up ticket captures it.

## Why This Matters for Multi-Agent Systems

The pattern here — SQLite for durability, local embeddings for semantic retrieval, MCP for agent-facing access, type taxonomy for query filtering — is broadly applicable to any multi-agent system that needs shared persistent context without a heavyweight external service.

Every component uses runtime-available primitives: Bun's bundled SQLite driver, a Xenova model that runs fully local without GPU requirements, the MCP protocol that MCD already relies on for tool delivery. There are no new infrastructure dependencies. The entire feature is a 233-line TypeScript class plus wiring.

For a fleet that will keep growing, the investment compounds: each heartbeat scan now leaves behind evidence that the next scan can use. The orchestrator is no longer an amnesiac.

---

*Cross-channel memory shipped in [PR #47](https://github.com/chan4lk/claude-multi-channel-discord/pull/47), merged 2026-06-20. Source: [`src/memory-store.ts`](https://github.com/chan4lk/claude-multi-channel-discord/blob/main/src/memory-store.ts), [`src/memory-backup.ts`](https://github.com/chan4lk/claude-multi-channel-discord/blob/main/src/memory-backup.ts), [`src/master-mcp-server.ts`](https://github.com/chan4lk/claude-multi-channel-discord/blob/main/src/master-mcp-server.ts).*
