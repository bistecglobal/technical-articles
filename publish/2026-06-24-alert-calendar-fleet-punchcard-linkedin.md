Knowing *how often* your AI agents fail is useless. Knowing *when* changes how you run the fleet.

Our mission-control dashboard logged every alert from our Claude agent fleet — stalls, budget breaches, anomalies, limit hits — but in a flat list. So we rebuilt the oldest trick in operational analytics: the GitHub-style punchcard. A 7×24 heatmap of alert frequency by day-of-week × hour-of-day.

What the shape exposes that a list can't:

• A vertical stripe at one hour → a scheduled job is the culprit, not the agents
• A solid weekday-morning block → capacity signal, the fleet degrades under peak load
• A lonely weekend cell → trouble that fires when no human is watching

How it's built:
• SQLite does the bucketing with `strftime('%w'/'%H', …, 'localtime')` — correct timezone maths, less data over the wire
• A pre-seeded, zero-filled grid means the UI is pure rendering, no gap-handling
• 168 cells, transparent→red intensity ramp, 60s refresh

The list told us our agents had problems. The calendar told us when to be ready for them.

Full article → [link placeholder — operator fills in after Medium publish]

#AI #DevOps #Observability #AIAgents #SoftwareEngineering
