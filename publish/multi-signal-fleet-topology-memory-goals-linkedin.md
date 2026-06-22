Your AI agent topology is only showing you half the picture.

When we run 15+ agents in parallel, edges only appeared when agents explicitly mentioned each other's slugs. But agents sharing the same memory patterns and working toward the same goals? Invisible — until we added two new signals.

We extended MCD's fleet topology with:

• **Memory similarity** — Jaccard overlap of keywords from each agent's MEMORY.md (what it's learned across sessions)
• **Goal similarity** — Jaccard overlap from each agent's GOAL.md (current mission)
• **Shared git remote** — hard signal; always renders as a solid red edge
• **Transcript refs** — the original signal, still weighted highest at 0.5 in the combined formula

Inferred edges (memory/goal only) render dashed. Hover to see the shared keywords that triggered the connection — "tenant, token, auth, rotation" — a fast explanation of why two agents you never wired together are actually working in the same cognitive space.

We also shipped Fleet Digest: one page, five health dimensions per project (context pressure, convergence, goal advancement, alerts, turns today), auto-flagged at defined thresholds. No more clicking through 15 project pages to find the one that's stuck.

Transcript edges are trailing indicators. Memory and goal overlap are earlier ones.

Full article → [link placeholder — operator fills in after Medium publish]

#AIAgents #Observability #DevOps #MultiAgent #EngineeringLeadership
