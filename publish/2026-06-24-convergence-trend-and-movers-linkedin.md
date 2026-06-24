A trend line tells you the fleet's average slipped. It's useless at telling you *who* slipped it. A leaderboard names names but can't tell you if today was normal or a crisis. You need both — and they have to agree.

Our AI agent fleet scores each agent daily on convergence (how much its work advanced its goal), stored in one history table. From that single table we built two views at two altitudes:

• Trend — a 14-day line of fleet mean convergence, with faint bars underneath counting how many agents actually finished (score ≥ 90). A mean that holds while the top-bin count drains = a fleet coasting on a few strong performers, erosion a single line hides.
• Movers — a day-over-day leaderboard: Climbers, Fallers, and a separate "New" bucket for agents with no prior day. Net delta in the header reconciles the two: it's the single-day analogue of the trend's slope.

Three lessons that generalize:

• Pair the aggregate with the attribution — a dip becomes a worklist, not a worry
• Never fake a baseline — treating "no prior data" as "zero" invents phantom +60 heroes and kills trust
• Encode one metric at two altitudes, not two metrics — read the same table, they can never disagree

The line tells you the fleet moved. The leaderboard tells you who moved it.

Full article → [link placeholder — operator fills in after Medium publish]

#AI #DevOps #Observability #AIAgents #DataVisualization
