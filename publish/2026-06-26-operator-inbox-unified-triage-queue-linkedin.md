Running a fleet of autonomous AI agents, the hard question isn't "what's wrong?" — it's "out of a dozen dashboards, where do I look FIRST?"

We had a sharp page for every failure mode: context pressure, circuit breakers, watchdog kills, health scores. Each one true, none of them answering the question that comes before all of them. So we built an Operator Inbox — one ranked queue that joins them.

What made it work:

→ No new telemetry. It reads signals the fleet already emitted — ~230 lines of API on top of existing logs and transcripts.

→ Severity floats with the reading, not the source. Context at 80% is a warning; at 90% it's critical. The same signal escalates itself.

→ Dedupe per project per signal type. One struggling agent shouldn't fire four alarms for one root cause.

→ Rank so the top row is always the next action — criticals first, newest first. Then deep-link straight to the dashboard that diagnoses it.

The lesson for anyone building control planes over autonomous systems: at scale, the missing piece is rarely another metric. It's a prioritized join across the metrics you already have.

Full article → [link placeholder — operator fills in after Medium publish]

#AIAgents #Observability #DevOps #PlatformEngineering #SoftwareArchitecture
