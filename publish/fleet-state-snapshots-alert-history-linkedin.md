Your live AI agent dashboard only shows you *right now*. It tells you nothing about what happened while you slept.

We've been running 19+ autonomous Claude Code agents through Mission Control. Real-time metrics are great for triage — but when an agent stalled overnight and the watchdog rescued it by morning, we had no record it ever happened.

So we shipped two things: fleet state snapshots and a persistent alert history log.

**What we built:**
- 📸 **Snapshots** — freeze the full fleet state (project count, idle/active/stalled/autonomous, token burn per project) as a labeled SQLite blob, any time
- 🔀 **A/B diff view** — select two snapshots, get an instant client-side diff showing added/removed/changed projects and token deltas
- 🚨 **Alert history** — every stall, budget threshold, and inject event written to a separate table the moment it fires; 30-day retention, type/slug filtering, cursor pagination
- 🗄️ **JSON blob design** — stored as `TEXT` in SQLite so schema changes don't break old snapshots

**What we immediately caught:**
Within day one, we spotted two projects that consistently stalled overnight and were always the first to need intervention. That pattern was invisible on the live dashboard. Budget alerts clustering at the same hour on the same weekday also became obvious — sprint-end pushes running in sync.

The lesson: live dashboards are for triage. Historical records are for understanding. If you're running autonomous agents at scale, instrument the history before you need it.

Full article → [link placeholder — operator fills in after Medium publish]

#AIAgents #Observability #DevOps #AutonomousSoftware #EngineeringLeadership
