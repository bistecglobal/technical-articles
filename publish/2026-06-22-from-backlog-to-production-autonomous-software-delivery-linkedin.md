We shipped 83 features in 3 months on keyflow — almost entirely without a human driving each one.

Here's the loop we built:

• **BACKLOG.md as the queue** — 86 numbered proposals, each with a spec in `.specclaw/changes/<slug>/proposal.md`. The spec is the agent's contract: problem, scope, acceptance criteria. Vague proposals produce vague code.

• **Specclaw as the build engine** — a gated pipeline: `propose → plan → build → verify → pr`. `workflow.strict: true` means no skipping. A code-reviewer agent blocks the PR if it returns `CHANGES_REQUESTED`.

• **MCD Scheduler as the heartbeat** — ~170 lines of TypeScript that tick every 60s, pick up due schedules, and inject a synthetic Discord message into the project channel. The agent receives it like any other message and acts. The scheduler knows nothing about Specclaw or keyflow — it just fires a string on a timer.

• **Self-regenerating backlog** — every implementation session ends by appending new proposals. The agent ships a feature, spots the gaps it exposed, and writes the next round of work. The backlog evolves instead of draining to zero.

83 PRs merged. Build gate (`bun run build`) enforced by pre-push hook on every branch. One operator role: review PRs and set direction.

Full article → [link placeholder — operator fills in after Medium publish]

#AI #SoftwareEngineering #AIAgents #DevOps #ClaudeCode
