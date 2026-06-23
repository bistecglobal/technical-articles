Your AI agents start every session knowing nothing — and you have no idea how much their memory has changed since you deployed them.

We built two features in our MCD fleet platform to address this:

**1. Scheduled Inject Templates** — push time-aware context into agent sessions on a cron schedule, separate from task prompts. Templates resolve `{{slug}}`, `{{date}}`, and `{{time}}` at fire time and arrive as ordinary session messages. Agents that previously referenced two-week-old sprint goals stopped doing so after a daily 09:00 orient inject.

**2. Memory Drift Timeline** — run `git log -p MEMORY.md` per project, compute a drift score (cumulative line churn / current file size), and surface a visual diff timeline in the dashboard. Score > 50% triggers an amber alert. One surprise finding: a project whose memory showed high drift turned out to have been edited manually during debugging — the timeline made it obvious in seconds.

Key design decisions:
- Inject schedules route through the same `deliver()` code path as real Discord messages — agent-agnostic by construction
- Two schedule types: `prompt` (expects agent to reply) vs `inject` (silent context delivery, no task footer)
- Drift score caps at 100; projects sorted by score so highest-drift agents surface first
- 1-hour server-side cache; date range and project filters for drill-down

If you run autonomous agents, treat scheduled context injection as infrastructure and memory git history as your audit trail.

Full article → [link placeholder — operator fills in after Medium publish]

#AIAgents #AutonomousAI #DevOps #Observability #EngineeringBlog
