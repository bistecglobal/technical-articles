Your AI agents are doing work every day — but do you know what it costs, where they're stalling, or which project shipped the most last week?

We added autonomous weekly fleet reporting to MCD (our multi-agent Discord orchestration platform) by reading the JSONL transcript files Claude Code already writes to disk. No new instrumentation. No external service.

What the report extracts per project:

- **Turns, tokens, tool calls** — direct from assistant message records in `~/.claude/projects/<path>/*.jsonl`
- **Stalls** — turns where `stop_reason` is `max_tokens` or `end_turn` with no tool use (agent stopped without acting)
- **PRs merged** — `git log --merges` on the project directory
- **Memories written** — `.md` files modified in `.claude/memory/` this week
- **Impact score** — `turns×2 + prCount×20 + toolCalls + memoriesWritten×5` — weighted toward durable outputs

Fleet view adds model-aware cost estimation (haiku/sonnet/opus rates per million tokens), `topByActivity` (highest impact score), and `topByEfficiency` (lowest cost per turn).

The `/reports` page renders neon stat cards, a sortable per-project table, 7-day turn sparklines, and an Export HTML button for offline sharing.

Zero new dependencies. Zero new infrastructure. The data was already there.

Full article → [link placeholder — operator fills in after Medium publish]

#AI #AgentOps #Observability #DevTools #EngineeringExcellence
