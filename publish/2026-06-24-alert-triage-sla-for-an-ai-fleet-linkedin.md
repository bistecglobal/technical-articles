An alert nobody can close isn't a signal — it's noise wearing a signal's clothes.

Our Mission Control dashboard watches a fleet of autonomous Claude agents. Every stall, budget breach, and memory-drift spike landed in one append-only table. The heatmaps lit up, but no operator could tell which alerts were already handled. A 3-day-old stall fixed an hour after it fired counted the same as a fresh one. Technically accurate, operationally useless.

We fixed it in two moves:

🔹 **A lifecycle.** Two nullable columns added to the live table — no rebuild. A NULL acknowledge-timestamp *is* the open state, so there's no status field to drift. State transitions are guarded single-row UPDATEs: the `WHERE ack_ts IS NULL` predicate is the lock, so two operators can't double-ack.

🔹 **Backward-compatible reads.** One optional flag (`includeAcked`, default true) means every existing dashboard keeps working; only the "current pressure" views flip to count open signal.

🔹 **An audit trail.** Closing an alert is a human action in an unattended fleet — so it's session-gated and writes an audit-log entry every time.

🔹 **An SLA on top.** The DB returns raw rows; the handler computes median/p90 time-to-ack, ack rate, and open backlog in one pass, sorted worst-first. The triage queue ranks itself by how much it's hurting.

The lesson: collecting events is cheap. The lifecycle and the measurement on top of them is where the value — and the engineering — actually lives.

Full article → [link placeholder — operator fills in after Medium publish]

#AIAgents #Observability #DevOps #SoftwareEngineering #IncidentResponse
