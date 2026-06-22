Your AI agents are misbehaving right now — and you won't know until a human checks in.

We run 19 autonomous Claude Code agents at BistecGlobal. The hardest part isn't deployment — it's detecting when one quietly goes off the rails: a context runaway, a silent stall after a tool timeout, an exploration spiral that burns tokens without making progress.

Our solution: z-score anomaly detection on Claude Code JSONL transcripts. No extra instrumentation. The data was already there.

How it works:
• Read every assistant turn in the last 7 days per project
• Compute baseline mean + std for 3 metrics: inter-turn gap, tool calls/turn, output tokens/turn
• Flag projects where the last 3-turn average deviates ≥2σ (warn) or ≥3σ (critical) from their own baseline
• Render a sortable table with sparklines — anomalous region highlighted in red

The key insight: use sample standard deviation (n-1), skip metrics with near-zero variance (flat baselines produce false positives), and require a minimum history window before scoring. Each project calibrates to itself — not a fleet-wide average.

The result is an /anomalies dashboard that tells operators which agents are behaving strangely *right now*, not just how they've performed over weeks.

Full article → [link placeholder — operator fills in after Medium publish]

#AIAgents #Observability #DevOps #Claude #EngineeringInsights
