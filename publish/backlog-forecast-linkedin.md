We run AI agents that pull their own work — each project has a BACKLOG.md the agent implements, merges, and checks off, mostly unattended. Great, until someone asks: when will it be done?

For a human team you'd open a burndown chart. For an autonomous agent we had a markdown file with some boxes ticked. So we built the forecast — and the interesting part is how to keep it honest:

• The backlog IS the dataset. Parse BACKLOG.md, count done vs pending. Source of truth and metrics source are the same file.
• Velocity two ways: a 4-week average (what it's actually doing — plan against this) and a p75 "good week" rate (optimistic). Show a range, not a falsely confident point.
• Burn down: subtract velocity/week from pending until zero → a finish date and a sloping line per scenario.
• Refuse to forecast when you can't: cap the horizon at 52 weeks and emit a "stalled" flag instead of a year-2090 finish date. A flat line is an early warning the agent quietly stopped pulling work.
• Be explicit about proxy data: we don't have a true completion timestamp, so we say so and use created-date as a stand-in — not dress a proxy up as a measurement.

It didn't make any agent faster. It made autonomous progress legible — and honest.

Full article → [link placeholder — operator fills in after Medium publish]

#AIAgents #Forecasting #AutonomousSystems #Observability #SoftwareEngineering
