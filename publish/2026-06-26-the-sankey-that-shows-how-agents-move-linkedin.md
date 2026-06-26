Your AI agents' states — busy, idle, stuck, crashed — are never written to the log. They only exist in the gaps between events. So we reconstructed them after the fact.

Mission Control now renders a Sankey diagram of how a whole agent fleet moves between states, rebuilt entirely from logs we were already keeping — zero new instrumentation.

How it works:

• Four states (idle / active / stuck / circuit-open) reconstructed per project over 30 days
• Fuses two sources — genuine user messages from the transcript + the circuit-breaker event log — into one timeline
• A 30-minute message gap counts as idle; circuit-open overrides activity
• Output: a from→to transition matrix with counts, average dwell time, and per-state time-share — drawn as an SVG Sankey
• The same transcript powers a companion per-tool error-rate view (worst-flakiest tool on top)

The lesson: the log already knew. Most "we need to add metrics for X" instincts are really "we need to read our logs differently." And derived signals must be honest about their assumptions — a Sankey that's subtly miscounting is worse than no Sankey at all.

Full article → [link placeholder — operator fills in after Medium publish]

#AIAgents #Observability #DevOps #LLM #SoftwareEngineering
