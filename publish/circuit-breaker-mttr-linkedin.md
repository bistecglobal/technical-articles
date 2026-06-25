A circuit breaker that protects you silently has a blind spot the size of your fleet.

Ours stopped respawning crashing Claude agents — good protection, but it forgot every trip the moment it happened. It couldn't answer the morning-after questions: which agent is flakiest, how long does it stay down, is it getting worse? Those are reliability questions with a standard language — MTTR, incident frequency, longest outage. We just weren't speaking it about our own agents.

The fix was almost free:

• Append every circuit open/close transition to a per-project JSONL log (4 lines in the event dispatcher).
• Stamp durationMs at close time, when both timestamps are trivially available — so the hard MTTR math is done at write time, not reconstructed later.
• Read it back as a sortable table: total opens, mean recovery, longest outage, opens/week, 30-day sparkline.
• Surface unclosed opens (opens − closes) — the agents arguably still down — instead of averaging them away.
• Wrap the write in a try/catch: an observability side-effect must never take down what it observes.

Nothing got more reliable that day. Crashing just became legible — a vibe became a ranked work queue. The most useful metrics came from writing down what the system was already deciding.

Full article → [link placeholder — operator fills in after Medium publish]

#SRE #Reliability #AIAgents #Observability #SoftwareEngineering
