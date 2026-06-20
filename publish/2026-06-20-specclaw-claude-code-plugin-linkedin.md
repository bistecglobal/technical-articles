A 623-line monolithic skill file is not a plugin. Here's how we turned Specclaw's internal workflow into a distributable Claude Code plugin — and what the plugin architecture forced us to get right.

Key lessons from shipping chan4lk/specclaw (v0.1.0 → v0.4.2):

• **Decompose by invocation surface.** One skill file per verb (`/specclaw:propose`, `/specclaw:build`, etc.) means Claude loads only what the current command needs. The wave-loop build orchestration is never in context during a proposal.

• **Plugin dir is read-only. Repo dir is yours.** All per-project state lives in `.specclaw/` in the consuming repo. The plugin stays generic and versioned. Auto-init (`specclaw-ensure-init`) makes `/specclaw:propose` work on a fresh repo with zero setup.

• **Let knowledge accumulate in the repo.** `.specclaw/knowledge/agent-hints.md` grows as agents encounter and log patterns. Promoted learnings are injected into every future coding agent's context — knowledge earned in that codebase, staying in that codebase.

• **Proactive routing vs. explicit-only is a spectrum.** v0.1.0 locked all verbs behind `disable-model-invocation: true`. v0.2.0 opened 13 of 15 — only credential commands stay explicit-only.

• **Bash + pipefail = silent exits.** Our Azure Boards integration had a grep pipeline that silently exited on "no match" under `set -euo pipefail`. Fix: `|| true`. Logged as a build learning, promoted to the knowledge base.

Install: `/plugin marketplace add chan4lk/specclaw`

Full article → [link placeholder — operator fills in after Medium publish]

#ClaudeCode #AIAgents #DeveloperTools #PluginArchitecture #BistecGlobal
