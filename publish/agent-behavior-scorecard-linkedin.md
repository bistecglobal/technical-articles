You can see what your AI agent *costs*. Can you see what it's actually *doing*?

We run a fleet of autonomous AI agents across multiple projects. Cost and latency metrics were easy. Behavioral insight wasn't — until we added a tool-call efficiency score to our Mission Control dashboard.

The insight: output tokens per tool call is a better signal than total calls or total tokens alone. An agent making many tool calls with little output is probably spinning. One making fewer calls with rich output is working methodically.

Key takeaways from shipping this:

• **Your transcripts already have the data.** Claude Code JSONL files log every tool call and token count. No new pipeline needed — we were already reading them for cost metrics.

• **Ratio metrics beat raw counts.** Total tool calls correlates with how long the agent ran, not how well. Output tokens per call filters out duration.

• **Context makes scores actionable.** A score of 45 during a research phase is expected. The same score during an implementation phase is a flag worth investigating.

• **Color thresholds give instant fleet health.** Green ≥70, amber 40–69, red <40 — you see behavioral quality across the whole fleet at a glance.

• **Behavioral observability is different from infra observability.** Agents make decisions. The interesting signal is in *how* they use their tools, not just how much they cost.

Full article → [link placeholder — operator fills in after Medium publish]

#AIEngineering #AgentOps #Observability #DevOps #ArtificialIntelligence
