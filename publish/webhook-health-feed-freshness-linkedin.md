Who tells you when your observability stack itself breaks?

A broken alert webhook doesn't page you — it IS the thing that pages you. A stale data feed doesn't error — it quietly serves yesterday's truth on a green dashboard. Both are silent failures. We shipped two Mission Control views over our Claude agent fleet to fix them:

• Webhook Delivery Health — aggregates a 7-day delivery log into per-webhook success rates, response-code distribution, and last error, sorted worst-first. Zero traffic reports 100% (no traffic ≠ failure).
• Feed Freshness Wall — probes 8 producer tables for newest-row age, normalizing unix/date/ISO-hour columns to epoch in SQL.
• Per-feed cadence, not a global timeout — event-driven feeds (alerts) stay quiet without flipping red; daily jobs get a 36h grace.
• Most-stale-first sort — the broken thing is always at the top.
• The lesson: a monitoring system is a system, and deserves the same scrutiny it gives everything else.

Full article → [link placeholder — operator fills in after Medium publish]

#Observability #AIAgents #DevOps #Reliability #SoftwareEngineering
