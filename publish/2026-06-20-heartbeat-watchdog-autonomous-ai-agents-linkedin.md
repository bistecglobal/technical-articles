When an AI agent stalls at 2am, nobody notices — unless you built a watchdog that reads its own transcripts.

We run 19 Claude Code agents simultaneously at BistecGlobal. Here's what we learned about keeping them alive without human babysitting:

**The problem:** Agents fail in 3 ways — a tool call that never resolves, an unanswered question, or a crashed tmux session. Each one looks like silence from the outside.

**The key insight:** Claude writes a detailed `.jsonl` conversation transcript after every turn. Reading that file tells you *what* is wrong, not just *that* something is wrong. Process health checks give you a heartbeat. Transcript analysis gives you a diagnosis.

**What we built:**
- `classifyChannel()` parses the last 200 transcript entries, flags unmatched `tool_use`/`tool_result` pairs and unanswered questions
- An adaptive watchdog threshold: `min(30min, max(5min, ceil(longest_recent_turn × 1.5)))` — scales with the channel's actual workload, not a fixed cutoff
- Recovery via synthetic Discord messages routed through the same code path as real messages — so dedup, pool eviction, and reply routing all work identically

**The lesson that cost us the most:** We killed aggressively first. A flat 5-minute stuck threshold fired on channels running 15-minute parallel-subagent workflows. Measuring observed turn history before setting the kill window fixed it.

Full article → [link placeholder — operator fills in after Medium publish]

#AI #DevOps #Claude #TypeScript #AutonomousAgents
