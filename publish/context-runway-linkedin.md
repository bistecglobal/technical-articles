A long-running AI agent doesn't crash when it runs out of context. It does something worse — it silently hits the wall, the conversation gets reset under it, and it keeps going having lost the thread. On a fleet, you find out three confused turns later.

We wanted to see the wall coming, not as "80% full" but as "about 4 turns left." A position isn't a forecast; a runway is. Here's how we compute one per agent from its transcript:

• Measure REAL context — input_tokens + cache_read + cache_creation. Cache reads still occupy the window; counting only uncached input wildly underestimates.
• Take the derivative — average context growth per turn over a short rolling window, counting only positive deltas (so compaction drops don't smear the trend).
• turnsRemaining = headroom ÷ growth/turn. That's the headline.
• Convert to hours via recent inter-turn pace, but throw out idle gaps > 1h or the estimate is meaningless.
• Report "unknown," never a confident wrong number, when data is thin.
• Triage the fleet: critical < 5 turns, warning < 10, soonest-to-exhaust on top.

It didn't make windows bigger. It bought lead time — rotate the session deliberately instead of letting the harness do it abruptly mid-task.

Full article → [link placeholder — operator fills in after Medium publish]

#AIAgents #LLM #Observability #MLOps #SoftwareEngineering
