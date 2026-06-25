# BistecGlobal Technical Article Ideas

A living index of article candidates derived from active projects. Each entry links to the local project path, describes what's being built, and proposes article angles around novel methods, tools, or problems solved.

---

## keyflow
**Path:** `/home/openclaw/.claude/channels/discord-multi/projects/keyflow`
**Repo:** https://github.com/chan4lk/keyflow

OKR and performance management platform used by Bistec.

**Article ideas:**
- *Building an AI-Autonomous Backlog Loop* — how we set up Claude to continuously implement, merge, and regenerate backlog items without human intervention
- *MCP-First Feature Development* — shipping features with Claude MCP tools as the primary API surface for AI agents
- *From BACKLOG.md to Production: Autonomous Software Delivery* — the scheduling, heartbeat, and inject pattern powering zero-touch delivery ✅ Published ([bistecglobal/blog#20](https://github.com/bistecglobal/blog/issues/20))
- *Domain-First MCP Design: Building OKR Tools That AI Agents Actually Use* — how we systematically exposed recognition, feedback, analytics, and template tools via MCP, including a service-layer extraction to eliminate API/MCP duplication [commits: `051cdfc`, `f7161bc`, `5149b32`] ✅ Published ([bistecglobal/blog#6](https://github.com/bistecglobal/blog/issues/6))

---

## specclaw
**Path:** `/home/openclaw/.claude/channels/discord-multi/projects/specclaw`
**Repo:** https://github.com/chan4lk/specclaw

AI-powered spec and proposal generation tool.

**Article ideas:**
- *Spec-Driven Development with AI: How Specclaw Turns Ideas into Actionable Proposals* — the proposal format, acceptance criteria generation, and backlog integration
- *Designing for AI-First UX: Lessons from Specclaw* — reducing friction for users interacting primarily through Claude
- *Packaging a Workflow as a Claude Code Plugin: From Private Tool to Marketplace* — how we took the specclaw spec workflow and published it as a distributable Claude Code plugin, including auto-init, per-repo learning, and Azure Boards integration [commits: `2c80c88` v0.1.0, `2d11406` per-repo knowledge, `aec77ba` Azure Boards] ✅ Published ([bistecglobal/blog#5](https://github.com/bistecglobal/blog/issues/5))
- *The Spec-Author Subagent: Automating Proposal Writing Within a Planning Workflow* — how `/specclaw:author-spec` spawns a focused subagent to draft the spec document, and how it integrates with the `/specclaw:plan` command via `--author-spec` flag [commit: `1179910`] ✅ Published ([bistecglobal/blog#8](https://github.com/bistecglobal/blog/issues/8))
- *The Code-Reviewer Subagent: Wiring Automated Code Review into a Spec-Driven Pipeline* — how Specclaw's `/specclaw:verify` now optionally invokes a `code-reviewer` agent that evaluates 10 dimensions (correctness, security, YAGNI, naming, complexity, test quality, design adherence, scope, dead code) with BLOCK/WARN/NOTE severity tagging, opt-in via `workflow.code_review: true` in `config.yaml` [commit: `f972cdc`] ✅ Published ([bistecglobal/blog#9](https://github.com/bistecglobal/blog/issues/9))

---

## claude-mcd (Multi-Channel Discord Bot)
**Path:** `/home/openclaw/.claude/channels/discord-multi/projects/claude-mcd`
**Repo:** https://github.com/chan4lk/claude-multi-channel-discord

Multi-project AI agent orchestration platform over Discord and Teams.

**Article ideas:**
- *Multi-Signal Fleet Topology: Connecting AI Agents by Memory and Goals* — how MCD's topology extended beyond transcript cross-references to Jaccard similarity of MEMORY.md/GOAL.md keywords and shared git remotes, plus Fleet Digest for per-project health aggregation [commits: `ddd8004`, `76755c8`] ✅ Published ([bistecglobal/blog#29](https://github.com/bistecglobal/blog/issues/29))
- *Scheduled Context Injection and Memory Drift Monitoring for AI Agent Fleets* — how MCD's `inject`-type schedules push time-aware context into agent sessions (vs task `prompt` schedules), and how the `/memory-diff` page tracks MEMORY.md churn with a drift score and git diff timeline [commits: `8246bb4`, `1389d61`] ✅ Published ([bistecglobal/blog#30](https://github.com/bistecglobal/blog/issues/30))
- *One Bot, Many Agents: Architecting a Multi-Channel AI Fleet* — how MCD isolates one Claude subprocess per Discord channel
- *Building a Mission Control for AI Agents* — neon graph UI, stall detection, autonomous heartbeat inject
- *One Number Per Agent: Composite Health Scorecard for an AI Fleet* — weighted 5-dimension health score (turns 30% + memory 30% + goals 20% + context 10% + anomalies 10%), worst-first sort, drill-down accordion [commit: `1757044`] ✅ Published ([bistecglobal/blog#31](https://github.com/bistecglobal/blog/issues/31))
- *Heartbeat Watchdog Pattern for Autonomous AI Agents* — detecting and recovering stalled agents at scale ✅ Published ([bistecglobal/blog#2](https://github.com/bistecglobal/blog/issues/2))
- *Cross-Platform AI: Wiring Claude to Discord, Teams, and WhatsApp* — adapters, credential routing, and session isolation
- *Cross-Channel Memory: Embedding-Based Shared Context for a Multi-Project AI Fleet* — SQLite + embedding store (`src/memory-store.ts`) that lets agents read and write shared memory across isolated channels, with MCP tools and backup [commit: `02b3a83` PR #47] ✅ Published ([bistecglobal/blog#7](https://github.com/bistecglobal/blog/issues/7))
- *From Polling to Push: Migrating an AI Fleet Dashboard to Server-Sent Events* — how MCD replaced N×30s REST polls with a single SSE `EventSource` per tab via React context, eliminating redundant requests with 5s server-pushed fleet diffs and exponential backoff reconnect (1s→30s) [commit: `d84461d`] ✅ Published ([bistecglobal/blog#10](https://github.com/bistecglobal/blog/issues/10))
- *Circuit Breaker for AI Subprocesses: Cross-Platform Installer and Crash Recovery in MCD* — how MCD ships a platform-native installer (Linux systemd / macOS launchd / Windows Task Scheduler) and a `FailureLedger` circuit breaker that exponentially backs off crashing Claude subprocesses before opening the circuit [commit: `2a86715`] ✅ Published ([bistecglobal/blog#11](https://github.com/bistecglobal/blog/issues/11))
- *Autonomous Weekly Fleet Report: Turning Claude Transcripts into Observability Data* — how MCD's `/api/reports/weekly` reads Claude Code JSONL transcripts to produce per-project turns, tokens, cost, stalls, PRs, and memories with a composite impact score and neon dashboard [PR #96, commit `3e3f957`] ✅ Published ([bistecglobal/blog#14](https://github.com/bistecglobal/blog/issues/14))
- *Live Agent Thought Stream: Visualizing Tool Calls as Floating Particles on the Fleet Graph* — how MCD tails project transcripts via SSE for new `tool_use` events and renders color-coded floating particles (cyan=file ops, amber=web, purple=agent, MCP suppressed) drifting up from project nodes with 3s lifetime [commit `f291a54` PR #95] ✅ Published ([bistecglobal/blog#15](https://github.com/bistecglobal/blog/issues/15))
- *Cross-Session Memory Distillation: Auto-Condensing Agent History on Session End* — how MCD's `distillation.ts` runs `claude -p` with 90s timeout + one retry after each session kill (`--distill-on-stop` flag), wires audit trail emission on completion, and surfaces distillation status in `!project show` [commit `feb0ed6` PR #83] ✅ Published ([bistecglobal/blog#17](https://github.com/bistecglobal/blog/issues/17))
- *Autonomous Goal Persistence: Injecting Project Goals into Every Claude Session* — how MCD writes a `GOAL.md` per project (max 500 chars, set via `!project set --goal`) and injects it as the opening message of every new session through `formatPrompt()`, keeping long-running agents on task across context resets [commit `feb0ed6` PR #83] ✅ Published ([bistecglobal/blog#19](https://github.com/bistecglobal/blog/issues/19))
- *Token Budget Enforcement for AI Agent Fleets: SSE Alerts, Queue Drain, and Monthly Rollover* — how MCD enforces per-project monthly token budgets: SSE `budget-alert` events at 50/80/100%, Discord master-channel notifications, inbound message queue when exhausted, UTC month-rollover drain with `budget-restored` event [commit `17a0f12` PR #84] ✅ Published ([bistecglobal/blog#16](https://github.com/bistecglobal/blog/issues/16))
- *Session Replay for AI Agents: Step-Through JSONL Debugging in MCD* — how MCD's `/replay` page parses Claude Code JSONL transcripts into `ReplayTurn` objects and renders them with turn counter, collapsible tool call rows, line-level diff vs the prior turn, timeline scrubber, and 3s/turn auto-play — making agent session debugging a visual, interactive experience [commit `eb98aa9`] ✅ Published ([bistecglobal/blog#21](https://github.com/bistecglobal/blog/issues/21))
- *Cross-Project Memory Similarity: A Jaccard Heatmap for Your AI Agent Fleet* — how MCD's `/similarity` page computes an N×N overlap matrix from per-project memory markdown files using Jaccard similarity on extracted keyword sets, with threshold slider, greedy cluster sort, and a 5-min cached `/api/similarity` endpoint [commit `0b5ffb3`] ✅ Published ([bistecglobal/blog#22](https://github.com/bistecglobal/blog/issues/22))
- *Fleet State Snapshots and Alert History: Point-in-Time Observability for AI Agent Fleets* — how MCD stores labeled fleet snapshots in SQLite and renders an A/B diff view (added/removed/changed projects with state and token deltas), alongside a unified alert event log with type/slug filtering and 30-day auto-purge [commit `5cd1c48`] ✅ Published ([bistecglobal/blog#23](https://github.com/bistecglobal/blog/issues/23))
- *Agent Behavior Scorecard: Measuring Tool-Call Efficiency Across an AI Fleet* — how MCD extends its metrics API with per-agent toolStats (top tools by call count, avg calls/turn, avg output tokens/turn, efficiency score 0-100) and renders a collapsible ScoreCard with horizontal bar chart in the fleet dashboard [commit `d9b4814`] ✅ Published ([bistecglobal/blog#24](https://github.com/bistecglobal/blog/issues/24))
- *When Your AI Agents Start Behaving Strangely: Statistical Anomaly Detection for Claude Fleets* — how MCD's `/anomalies` page applies z-score anomaly detection to Claude Code JSONL transcripts, computing 7-day baselines (mean + sample stddev) for inter-turn gap, tool calls/turn, and output tokens/turn, then flagging projects whose last-3-turn mean deviates ≥2σ (warn) or ≥3σ (critical) [commit `808d275`] ✅ Published ([bistecglobal/blog#25](https://github.com/bistecglobal/blog/issues/25))
- *Don't Wait for the Watchdog: Forecasting AI Agent Stalls Before They Fire* — how MCD's `/api/stall-risk` (P148) computes a forward-looking 0–100 stall risk per project from four weighted heuristics — context pressure (30%), turn-quality trend (30%), recency vs watchdog threshold (25%), kill history (15%) — omitting (not renormalizing) unknown signals to avoid false alarms, reusing `/api/fleet` as single source of truth; `/stall-risk` page with risk bars + factor chips + pre-inject, top-3 widget in FleetAdvisorPanel [commit `a67e551` PR #166] ✅ Published ([bistecglobal/blog#32](https://github.com/bistecglobal/blog/issues/32))
- *Forecasting Fleet Exhaustion: Token Burn-Rate and Backlog Velocity Projections in Mission Control* — how MCD's `/api/metrics/burn-rate` computes month-to-date tokens, a trailing 7-day daily rate, projected month-end usage, and days-until-exhausted vs each project's `monthlyTokenBudget` (sparklines + exhaust badges on `/burn-rate`), and how `/api/metrics/forecast` turns the burndown series into a 3-scenario fan chart — pessimistic (slowest 7d), expected (trailing 14d mean), optimistic (fastest 7d) — with a shaded band on `/forecast` [commits `6a4394f` P150, `80901d1` P160] ✅ Published ([bistecglobal/blog#34](https://github.com/bistecglobal/blog/issues/34))
- *Per-Channel Native Slash Commands for a Multi-Project Discord Fleet* — how MCD adds Discord-native `/model` and `/provider` commands (show/set/clear) that operate on the project bound to the invoking channel — no slug needed, unlike master-channel `!project` verbs — killing the Claude subprocess so it respawns with the new model/provider env, with mutations gated on `access.allowFrom`; pure `model-command.ts`/`provider-command.ts` modules reused by master commands [commits `5237b72` PR #165, `afb9b69` PR #167] ✅ Published ([bistecglobal/blog#35](https://github.com/bistecglobal/blog/issues/35))
- *Streamgraphs for Agent Memory: Visualizing Cumulative Knowledge Growth Across a Fleet* — how MCD's `/memory-stream` renders a stacked silhouette streamgraph of cumulative memory entries per project over 30 days, dating entries by file mtime, daily-bucketing, and seeding the baseline from pre-window entries via `/api/memory/growth` [commit `80901d1` P159] ✅ Published ([bistecglobal/blog#37](https://github.com/bistecglobal/blog/issues/37))
- *Does a Bigger Memory Make a Smarter Agent? A Diagnostic Quadrant for an AI Fleet* — how MCD's `/memory-convergence` page plots each agent as a bubble (x = log10 memory `sizeBytes`, y = `convergenceScore`, color = goal status), splits the plane into four quadrants at the memory midpoint / 50% convergence, flags the heavy-memory-stalled quadrant as prune candidates (red count in header), and reports a bucketed Pearson correlation hint (±0.3 dead zone, `n/a` below 3 points) — all client-side off `/api/fleet` [commit `cf699d5` P176] ✅ Published ([bistecglobal/blog#38](https://github.com/bistecglobal/blog/issues/38))
- *The Oldest Chart in Statistics, Rebuilt for an AI Agent Fleet* — how MCD's `/platform-matrix` renders a contingency-table cross-tab (rows = platform discord/teams/whatsapp, cols = state idle/active/stalled/autonomous) with per-cell count, opacity ∝ count (`0.18 + 0.82*(n/max)`, max clamped ≥1), row/col/grand-total margins, defensive bucketing of unknown platform→discord & state→idle, pre-seeded zero grid — all client-side off `/api/fleet`, ~125-line page + 1 nav line, 60s refresh [commit `beeeddb` P179] ✅ Published ([bistecglobal/blog#39](https://github.com/bistecglobal/blog/issues/39))
- *The Silent Agent: Catching Usage Limits and Auto-Switching Models Mid-Run* — how MCD detects a `claude` CLI usage-limit hit (`isApiErrorMessage`+`apiErrorStatus:429`) in the existing 2s transcript poll loop, parses the limited model + verbatim reset time, dedupes per episode, and either posts ready-to-paste `!project model/provider --set` offers (hiding providers whose `apiKeyEnv` is unset) or auto-switches via an opt-in per-project `limitFallback` chain; pure `limit-offer.ts` module + `limit-hit` ProjectPool event [commits `058b686`..`0b6f679` PR #164] ✅ Published ([bistecglobal/blog#33](https://github.com/bistecglobal/blog/issues/33))
- *When Do Your AI Agents Cry for Help? A Day-and-Hour Heatmap of Fleet Alerts* — how MCD's `/alert-calendar` builds a 7×24 GitHub-style punchcard of alert frequency, grouping `alert_events` by `strftime('%w')` (day-of-week) and `strftime('%H')` (hour) over the trailing 30 days in `/api/alert-calendar`, finding the peak cell and rendering a green-intensity grid so operators can spot *when* their fleet most often degrades [commit `f557d09` P190 PR #194] ✅ Published ([bistecglobal/blog#40](https://github.com/bistecglobal/blog/issues/40))
- *Scrubbing Through Time: A Snapshot Slider for Replaying Fleet State* — how MCD's `/snapshot-scrubber` parses stored fleet snapshots, sorts them by timestamp, and lets an operator drag a slider through history to replay active/autonomous project counts as a sparkline — reusing the existing `/api/snapshots` store, no new endpoint [commit `f557d09` P187 PR #194] ✅ Published ([bistecglobal/blog#42](https://github.com/bistecglobal/blog/issues/42))
- *Two Numbers, One Verdict: A Convergence × Context-Risk Scatter for Triaging Agents* — how MCD's `/convergence-risk` page plots every agent on a scatter (x = context usage %, y = convergence score) and flags the danger quadrant — `contextUsagePct ≥ 70 AND convergenceScore < 50` — agents burning context without converging on their goal, all client-side off `/api/fleet` [commit `8bfc36b` P186 PR #193] ✅ Published ([bistecglobal/blog#41](https://github.com/bistecglobal/blog/issues/41))
- *Who's Improving, Who's Slipping? Day-over-Day Convergence Movers for an AI Fleet* — how MCD's `/convergence-movers` + `/api/convergence-movers` compute per-project day-over-day convergence deltas from the latest two `convergence_history` rows, surfacing the biggest gainers and losers so operators triage regressions first [commit `a5e32e4` P185 PR #191] ✅ Published ([bistecglobal/blog#44](https://github.com/bistecglobal/blog/issues/44) — paired with P183)
- *Is the Whole Fleet Getting Smarter? A 14-Day Convergence Trend Line* — how MCD's `/convergence-trend` + `/api/convergence-trend` aggregate `convergence_history` per calendar day (AVG score, count of converged agents ≥90) over 14 days to show whether fleet-wide goal convergence is trending up or down [commit `a5e32e4` P183 PR #191] ✅ Published ([bistecglobal/blog#44](https://github.com/bistecglobal/blog/issues/44) — paired with P185)
- *An EKG for an AI Agent Fleet* — how MCD's `/ekg` page renders a 48-hour, 5-lane activity waveform (alerts, injects, memory writes, digests, broadcasts) pulled as bare timestamp arrays from five tables via `getEkgTimestamps` (splitting alerts/injects by `alert_type`, normalizing broadcasts' ISO `ts` to epoch with `strftime('%s')`), hourly-bucketed with `now` snapped to the hour, each lane scaled to its own busiest hour so rare events stay visible, busiest-hour headline, `/api/ekg` 60s refresh [commit `8918ad7` P199 PR #204] ✅ Published ([bistecglobal/blog#45](https://github.com/bistecglobal/blog/issues/45))
- *Histogram of Minds: Visualizing Convergence Distribution Across an AI Fleet* — how MCD's `/convergence-dist` bins agent convergence scores into deciles (`Math.floor(score/10)`) and renders a red→green gradient histogram off `/api/fleet`, exposing whether the fleet is bimodal (some converged, some stuck) at a glance [commit `5293573` P182 PR #189] ✅ Published ([bistecglobal/blog#43](https://github.com/bistecglobal/blog/issues/43))
- *Your AI Fleet's Alerts Are Lying to You — Until You Give Them a Lifecycle* — how MCD adds an ack/re-open lifecycle to `alert_events` via two nullable columns (`ack_ts`/`ack_by`, additive migration), guarded single-row UPDATEs (`WHERE ack_ts IS NULL`) as the concurrency lock, a session-gated POST/DELETE `/api/alerts/[id]/ack` that writes `alert.ack`/`alert.unack` audit entries, and a backward-compatible `includeAcked` read flag; then layers `/api/alert-sla` (P198) computing median/p90 time-to-ack, ack rate, and open-backlog stats from raw `(type, ts, ack_ts)` rows with interpolated percentiles, sorted worst-first [commits `9a88fd4` P196, `e6fa35e` P198] ✅ Published ([bistecglobal/blog#46](https://github.com/bistecglobal/blog/issues/46))

### New candidates — ideate scan 2026-06-24 (MCD P200–P211)
- *One Attention Engine, Two Faces: Unifying a Fleet Advisor and a Natural-Language Brief* — how MCD extracted a shared `attention-findings.ts` (452 lines + unit tests) that both the `/api/advisor` and `/api/brief` routes consume, collapsing ~500 lines of duplicated heuristics into one tested module [commits `0182518` P205, `b211279` P208] ✅ Published ([bistecglobal/blog#47](https://github.com/bistecglobal/blog/issues/47))
- *Forecasting Fleet Convergence: Projecting Whether Your Agents Will Actually Finish* — MCD's P202 Fleet Convergence Forecast extrapolates `convergence_history` trend to project fleet-wide goal attainment [commit `f276ed6` P202] ✅ Published ([bistecglobal/blog#48](https://github.com/bistecglobal/blog/issues/48))
- *Does Bigger Memory Correlate With Convergence? Measuring It Per-Fleet* — MCD's P201 Memory × Convergence Correlation quantifies the relationship between agent memory churn and convergence delta (Pearson r + thrashing quadrant) [commit `6865f7f` P201] ✅ Published ([bistecglobal/blog#49](https://github.com/bistecglobal/blog/issues/49))
- *Signal Co-Occurrence: A Force Graph of Which Fleet Alerts Fire Together* — MCD's P211 builds a force-directed graph linking alert/attention signals that co-occur, surfacing correlated failure modes [commit `5fb93ca` P211] ✅ Published ([bistecglobal/blog#51](https://github.com/bistecglobal/blog/issues/51))
- *Webhook Delivery Health and Feed Freshness: Watching the Watchers* — MCD's P195/P197 track outbound webhook delivery success and per-feed staleness so the observability pipeline itself is observable [commit `ad41787` P195+P197]

### New candidates — ideate scan 2026-06-24 (keyflow)
- *A Cycle Health Context Row: Showing Elapsed % vs Average Score Inline* — keyflow's my-okrs page adds a context row comparing how far through the cycle you are against the average KR score, flagging at-risk cycles at a glance [commit `9862fc1` #353]
- *Global API Error Toast With Retry: One Resilience Pattern for a Whole Next.js App* — keyflow's app-wide error toast intercepts failed API calls and offers a one-click retry, replacing scattered per-component error handling [commit `18371c3` #354] ✅ Published ([bistecglobal/blog#50](https://github.com/bistecglobal/blog/issues/50))

---

## ai-core
**Path:** `/home/openclaw/.claude/channels/discord-multi/projects/ai-core`
**Repo:** https://github.com/bistecglobal/ai-core

Core AI infrastructure and shared services for Bistec.

**Article ideas:**
- *Provider Routing at the Edge: Switching Between Anthropic and MiniMax Without Redeployment* — hot-swap provider pattern with env-level routing
- *Building a Multi-Provider AI Gateway for Enterprise* — abstracting model APIs behind a unified interface
- *DRY Principle for AI Skills: Extracting Shared Discipline Skills Across an Enterprise Agent Stack* — how we eliminated coding-standard duplication across four role skills into one canonical shared skill (`bistec-coding-standards`), cutting 166 lines from `bistec-developer` alone [commit: `ca03393` TEC-41] ✅ Published ([bistecglobal/blog#3](https://github.com/bistecglobal/blog/issues/3))
- *AI-Generated Load Tests: Turning an OpenAPI Spec into a k6 Performance Script* — how the `k6-test-generator` skill interprets an API contract, URL, or plain description and produces a ready-to-run k6 test with realistic scenario, thresholds, and parameterized load stages [commit: `83da7fe`] ✅ Published ([bistecglobal/blog#12](https://github.com/bistecglobal/blog/issues/12))
- *Browse-Before-Install: Building a Public Skill Catalog for a Claude Code Plugin* — how ai-core serves a client-side `/catalog` page that reads the public `manifest.json`, supports tag/category filtering and name/description search, and renders one-click copy-to-install commands with no auth required [commit `d74ca76`] ✅ Published ([bistecglobal/blog#18](https://github.com/bistecglobal/blog/issues/18))

---

## agent-nexus
**Path:** `/home/openclaw/.claude/channels/discord-multi/projects/agent-nexus`

Multi-tenant cloud control plane for building, deploying, and governing AI agents across Copilot Studio, Azure AI Foundry, Bedrock, and Vertex.

**Article ideas:**
- *Multi-Tenant AI Agent Governance: What We Learned Building Agent Nexus* — policy enforcement, quota management, and audit trails across platforms
- *Unifying AI Agents Across Azure, AWS, and GCP: Architecture Patterns* — adapters, normalisation, and cross-cloud observability
- *AI Agent Observability with OTLP: Real-Time AgentScore, Cost Tracking, and Multi-Tenant Ingest* — how Agent Nexus ingests OpenTelemetry traces from Copilot Studio / Bedrock / Vertex agents, assigns per-agent scores, tracks live cost dimensions, and gates on tenant ID [commits: `46e27a5`, `4041469`, `6ad734a`, `d40a15e`] ✅ Published ([bistecglobal/blog#4](https://github.com/bistecglobal/blog/issues/4))
- *Two-Tier Auth for Multi-Platform AI Agent Governance: Developer vs Tenant Credentials in Agent Nexus* — how Agent Nexus implements a `CredentialTier` enum (None / Developer / Tenant) parsed from Zitadel project-role JWT claims, gates `ConnectionEndpoints` per tier returning 403 on underqualified requests, and wires Infisical two-tier config and Zitadel Cloud role assertion across 14 specclaw tasks [commits: `d41ba41` through `d40a15e`] ✅ Published ([bistecglobal/blog#27](https://github.com/bistecglobal/blog/issues/27))
- *Adding a Third Cloud: Vertex AI Agent Discovery and Token-Based Cost in Agent Nexus* — how Agent Nexus extended its multi-cloud observability to Vertex AI: discover agents using tenant Vertex credentials (not host ADC), ingest telemetry, and compute token-based cost alongside Copilot Studio [commit `16ffdeb`] ✅ Published ([bistecglobal/blog#28](https://github.com/bistecglobal/blog/issues/28))

---

## btg-devops
**Path:** `/home/openclaw/.claude/channels/discord-multi/projects/btg-devops`
**Repo:** https://github.com/bistecglobal/btg-devops

DevOps tooling and infrastructure for Bistec on Hetzner + Power Platform.

**Article ideas:**
- *Cost-Effective AI Infrastructure on Hetzner: Our Setup and Lessons* — self-hosted Claude runners, SSH credential management, tmux-based agent sessions
- *Power Platform Meets Open-Source DevOps: Bridging Microsoft and Linux Stacks*

---

## gasflow
**Path:** `/home/openclaw/.claude/channels/discord-multi/projects/gasflow`
**Repo:** `git@ssh.dev.azure.com:v3/BistecGlobal/gasflow/Gasflow` (Azure DevOps)

Gas industry workflow platform.

**Article ideas:**
- *AI-Assisted Workflow Automation in Industrial Domains* — applying Claude Code to domain-specific enterprise software
- *Azure DevOps + Claude Code: Integrating AI Agents into Enterprise Git Pipelines*

---

## keyflow — CI/CD

- *Cutting E2E CI from 20 Minutes to 5: Three-Layer Caching and a Self-Hosted Runner Migration* — how keyflow layered GitHub Actions caching on `node_modules` (keyed on `package-lock.json`), Next.js webpack build artifacts (`.next/cache`), and the Playwright Chromium browser so warm runs skip `npm ci` entirely, alongside migrating all workflows off GitHub-hosted runners to a self-hosted runner and swapping buildx for podman build/push [commits `21d07d0`, `8c78d0a`, `00cd3ca`] ✅ Published ([bistecglobal/blog#36](https://github.com/bistecglobal/blog/issues/36))

---

## qualitylife
**Path:** `/home/openclaw/.claude/channels/discord-multi/projects/qualitylife`
**Repo:** https://github.com/chan4lk/fitness-ai

Fitness and wellness AI platform.

**Article ideas:**
- *Personalised Fitness with AI: Building a Coaching Agent That Learns Over Time*
- *Applying LLM Memory Patterns to Health and Wellness Apps*

---

## mindbrige
**Path:** `/home/openclaw/.claude/channels/discord-multi/projects/mindbrige`
**Repo:** https://github.com/chan4lk/mindbrige

**Article ideas:**
- *MindBrige: AI-Powered Knowledge Bridging — Design and Architecture*

---

## academy-videos
**Path:** `/home/openclaw/.claude/channels/discord-multi/projects/academy-videos`
**Repo:** https://github.com/bistecglobal/process-docs

Video and learning content pipeline for Bistec Academy.

**Article ideas:**
- *Automating Learning Content Pipelines with AI Agents* — from raw footage to structured course material
- *Building a Process Documentation Bot: How We Keep Docs in Sync with Reality*
- *Voice Cloning at Scale: MiniMax TTS Backend for AI-Generated Training Content* — how we replaced manual narration with a MiniMax voice-clone backend integrated into the academy video pipeline, covering model selection, voice consistency across tracks, and Higgsfield generative video [commits: `6e2ea3a`, `83963a9`] ✅ Published ([bistecglobal/blog#13](https://github.com/bistecglobal/blog/issues/13))

---

## Cross-Project / Platform Articles

These cut across multiple projects and capture the broader Bistec engineering story:

- *How We Run 19 AI Agents Simultaneously (and Keep Them From Going Rogue)* — MCD fleet overview, autonomous mode, supervised heartbeat
- *The Specclaw-to-Production Pipeline: From Proposal to Merged PR Without a Human* — end-to-end autonomous delivery
- *WhatsApp as a Dev Interface: Routing Claude Agents Over Unofficial APIs* — Baileys integration, ToS trade-offs, session management
- *Memory That Persists: Building Cross-Session AI Context for a Multi-Project Fleet*
- *Provider Agnostic AI: How We Switch Between Anthropic and MiniMax Mid-Sprint*

---

*Updated: 2026-06-24 (published alert-triage-sla #46; ideate scan — added 7 new candidates: MCD P201/P202/P205/P208/P211/P195+P197, keyflow #353/#354) | Maintained by bistec-articles agent*
