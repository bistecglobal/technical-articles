Tool usage is the closest thing an AI agent has to a behavioral fingerprint — and most teams never look at it.

We run a fleet of autonomous coding agents. We measured *that* they worked (turns, tokens, stalls) but never *how*. So one agent could quietly rack up 4,000 Edit calls and almost no Reads, and we'd never know whether it was refactoring or editing blind.

We built a single cross-project heatmap to fix that — projects × tools, color = call count:

• Built entirely from transcript logs we already had — no new pipeline, no DB, no agent instrumentation
• Gamma-corrected color (pow 0.55) so skewed, power-law usage stays readable instead of one bright corner on black
• Top 20 projects × top 30 tools, with the truncation reported in the header — no silent lies
• Hand-drawn SVG with marginal totals — a contingency table, the oldest chart in statistics
• 4 files, zero storage cost

The outlier you can't spot across twenty separate dashboards becomes obvious on one canvas.

Full article → [link placeholder — operator fills in after Medium publish]

#AIAgents #Observability #DevOps #DataVisualization #SoftwareEngineering
