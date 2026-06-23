A count tells you how much an AI agent remembers. It can't tell you how it got there — and the question that actually matters about memory is a shape, not an integer.

Our Claude agents each keep a persistent memory: a directory of one-fact Markdown files that survive across sessions. We built a silhouette streamgraph in Mission Control — cumulative memory entries per project over 30 days, each agent a flowing coloured band. What the build taught us:

🗓️ **Reach for the signal that already exists.** Memory files have no `created_at`. Rather than retrofit a schema, we date entries by file mtime — not perfect, but free and precise enough for a trend. Match data precision to the question.

📊 **Baselines keep cumulative charts honest.** Entries predating the window fold into a starting height, so a band starts at its true height instead of faking a dramatic day-one ramp.

🎨 **You don't always need a charting library.** A silhouette streamgraph is a few lines of stacking math + a hand-rolled SVG path. No D3, lean bundle, full control.

🔍 **Absence is a signal.** A band that goes flat = an agent that stopped recording anything — often the earliest visible sign of trouble, before a stall alert fires. Visualising growth gives you a stall detector for free.

Full article → [link placeholder — operator fills in after Medium publish]

#AI #Observability #DataVisualization #AIAgents #SoftwareEngineering
