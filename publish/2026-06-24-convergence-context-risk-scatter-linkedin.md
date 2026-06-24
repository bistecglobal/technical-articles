One metric is a number. Two metrics are a diagnosis.

Our AI agent fleet tracked context usage (how full the model's window is) and convergence (how close it is to its goal) in separate columns. Each is ambiguous alone — high context usage on an agent that's nearly done is fine; on one that's nowhere near done it's a slow-motion stall. The danger lives in the *combination*, which a table hides and a scatter plot reveals.

So we plotted every agent as a dot: context % on x, convergence on y. The bottom-right quadrant — high context, low convergence — is tinted red as the at-risk zone, and the count of agents in it shows as one number in the header.

What it caught that the table didn't:

• An agent at 68% context / 0.45 convergence — "fine" in two columns, drifting into danger on the scatter
• A stalled dot already deep in the red quadrant — the failure was predictable
• An agent at 95% context but 0.92 convergence — safe; a single-metric alarm would have falsely paged you

The whole feature is two thresholds and a filter, reusing the existing fleet API — no new backend, no model training. Interpretable, tunable, and an operator can argue with it.

A list tells you what's running. A scatter tells you which agent is quietly running out of road.

Full article → [link placeholder — operator fills in after Medium publish]

#AI #DevOps #Observability #AIAgents #DataVisualization
