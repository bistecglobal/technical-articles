AI subprocesses crash differently than web services — and your orchestration layer needs to be ready for it.

We run Claude Code subprocesses in our MCD platform: one per Discord channel, managed by a central pool. When one crashes in a tight loop, unchecked respawning burns resources and floods operators. So we shipped two things in PR #81:

**Cross-platform installer:**
- `curl | bash` (Linux/macOS) or `npx mcd-setup` (Windows) — full setup in one command
- Installs bun + tmux, configures .env, registers a system service (systemd / launchd / NSSM)
- Idempotent — safe to re-run on existing deployments

**FailureLedger circuit breaker in ProjectPool:**
- Exponential backoff on crash: 5s → 10s → 30s → 2min → 5min
- After 5 failures in a 30-min window: circuit opens, messages drop, operator notified
- Auto-reset after 10 min — no manual intervention required
- circuit-state.json feeds a Mission Control dashboard badge in real time

The design borrows directly from distributed systems — because an AI fleet of isolated subprocesses has the same failure topology as a microservices mesh, just with a different kind of node.

Full article → [link placeholder — operator fills in after Medium publish]

#AI #DevOps #NodeJS #Resilience #ArtificialIntelligence
