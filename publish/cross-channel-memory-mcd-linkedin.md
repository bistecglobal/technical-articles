We gave our fleet of 19 AI agents a memory that survives restarts — using 233 lines of TypeScript, SQLite, and a local embedding model with no new infrastructure.

Here's what we built and why it matters:

**The problem:** Our multi-channel-discord bot runs Claude Code agents across 19+ projects simultaneously. The master orchestrator ran heartbeat scans and injected prompts into stalled agents — but woke up each session with zero context. No record of past stalls, past decisions, past injections.

**What we shipped (PR #47):**
- SQLite-backed `MemoryStore` with 5 typed memory categories: `channel_summary`, `decision`, `pattern`, `coordination`, `general`
- Local embeddings via `Xenova/all-MiniLM-L6-v2` — no GPU, no API, runs on the same box
- Two-pass recall: LIKE keyword filter → cosine similarity re-ranking on the candidate set
- R2 backup with WAL checkpoint for durability (timestamped + `latest.db` keys)
- 4 MCP tools exposed exclusively to the master session: `remember`, `recall`, `forget`, `memory_stats`

**The key insight:** The embedding pipeline loads async in the background at startup — the bot doesn't block. If the model isn't ready, recall falls back to keyword search. Graceful degradation by design.

**Result:** Every heartbeat scan now deposits `coordination` and `channel_summary` memories. The next scan reads them before acting. The orchestrator is no longer an amnesiac.

Full article → [link placeholder — operator fills in after Medium publish]

#AI #MultiAgent #MCP #SQLite #EnterpriseAI
