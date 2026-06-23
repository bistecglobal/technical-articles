When you run close to twenty AI agents simultaneously, "are they working?" isn't the right question. The right question is: which ones are quietly degrading while you're not looking?

We built a Composite Health Scorecard for our AI fleet — a single weighted score per agent, default-sorted worst-first, so the most troubled projects are always at the top of the page.

**Five dimensions, each 0–100:**
- Turn Quality (30%) — are agents producing meaningful output in the last 24h?
- Memory Health (30%) — is agent memory recent, dense, stable, and fresh?
- Goal Progress (20%) — are agents actually pursuing their defined goals?
- Context Pressure (10%) — are agents approaching their 200k-token context limit?
- Anomaly Score (10%) — z-score deviations in turn gap, tool calls, output tokens?

**Three design lessons that weren't obvious at first:**

1. Anomalies should be an early-warning signal, not a primary metric — a busy agent doing real work looks anomalous. Weighting it at 10% instead of higher kept the score honest.

2. "Worst first" sort is a policy decision. It feels wrong if you think of scores as grades. It's exactly right if you're triaging a live fleet every morning.

3. Null ≠ zero. A new project without a GOAL.md should not look critically unhealthy. Redistributing the null dimension's weight across the remaining four preserved trust in the score.

The scorecard doesn't replace five detail dashboards — it tells you *which one* to open. Triage in one second, diagnose from there.

Full article → [link placeholder — operator fills in after Medium publish]

#AIEngineering #DevOps #Observability #AgentFleet #BistecGlobal
