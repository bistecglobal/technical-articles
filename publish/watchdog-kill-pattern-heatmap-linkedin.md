A stuck AI agent is invisible — right up until a watchdog kills it and the reason evaporates. Run 19 autonomous agents for a few weeks and you're left with a pile of terminated sessions and no memory of why.

We fixed that by giving the watchdog a logbook. Every time it kills a frozen agent, it writes one line — timestamp, project, runtime, and the tool the agent was stuck inside — reconstructed from the pending-tool map at the exact moment of death. Then a mission-control page turns those lines into three plain cross-tabs:

🔴 A 7×24 day/hour heatmap — do kills cluster at 2 AM, or wash evenly across a chronically fragile agent?
🔧 A top-10 chart of the tool each agent was running when it hung — if 8 of 10 kills share one tool, that's a tool problem, not a fleet problem.
📉 A context-pressure scatter — is the agent dying because it's overloaded, or hanging on something external?

Three lessons that generalized:

• Log at the moment of death — the most useful field only exists while the agent is still hung.
• Never let telemetry block the operation it observes — the kill must succeed even if logging fails.
• Record distress, not routine — logging every lifecycle event drowns the signal.

A watchdog that kills silently keeps your fleet alive. One that keeps a logbook teaches you why it had to.

Full article → [link placeholder — operator fills in after Medium publish]

#AI #Observability #DevOps #AutonomousAgents #SoftwareEngineering
