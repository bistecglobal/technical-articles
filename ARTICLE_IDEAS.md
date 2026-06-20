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
- *From BACKLOG.md to Production: Autonomous Software Delivery* — the scheduling, heartbeat, and inject pattern powering zero-touch delivery
- *Domain-First MCP Design: Building OKR Tools That AI Agents Actually Use* — how we systematically exposed recognition, feedback, analytics, and template tools via MCP, including a service-layer extraction to eliminate API/MCP duplication [commits: `051cdfc`, `f7161bc`, `5149b32`]

---

## specclaw
**Path:** `/home/openclaw/.claude/channels/discord-multi/projects/specclaw`
**Repo:** https://github.com/chan4lk/specclaw

AI-powered spec and proposal generation tool.

**Article ideas:**
- *Spec-Driven Development with AI: How Specclaw Turns Ideas into Actionable Proposals* — the proposal format, acceptance criteria generation, and backlog integration
- *Designing for AI-First UX: Lessons from Specclaw* — reducing friction for users interacting primarily through Claude
- *Packaging a Workflow as a Claude Code Plugin: From Private Tool to Marketplace* — how we took the specclaw spec workflow and published it as a distributable Claude Code plugin, including auto-init, per-repo learning, and Azure Boards integration [commits: `2c80c88` v0.1.0, `2d11406` per-repo knowledge, `aec77ba` Azure Boards]
- *The Spec-Author Subagent: Automating Proposal Writing Within a Planning Workflow* — how `/specclaw:author-spec` spawns a focused subagent to draft the spec document, and how it integrates with the `/specclaw:plan` command via `--author-spec` flag [commit: `1179910`]

---

## claude-mcd (Multi-Channel Discord Bot)
**Path:** `/home/openclaw/.claude/channels/discord-multi/projects/claude-mcd`
**Repo:** https://github.com/chan4lk/claude-multi-channel-discord

Multi-project AI agent orchestration platform over Discord and Teams.

**Article ideas:**
- *One Bot, Many Agents: Architecting a Multi-Channel AI Fleet* — how MCD isolates one Claude subprocess per Discord channel
- *Building a Mission Control for AI Agents* — neon graph UI, stall detection, autonomous heartbeat inject
- *Heartbeat Watchdog Pattern for Autonomous AI Agents* — detecting and recovering stalled agents at scale ✅ Published ([bistecglobal/blog#2](https://github.com/bistecglobal/blog/issues/2))
- *Cross-Platform AI: Wiring Claude to Discord, Teams, and WhatsApp* — adapters, credential routing, and session isolation
- *Cross-Channel Memory: Embedding-Based Shared Context for a Multi-Project AI Fleet* — SQLite + embedding store (`src/memory-store.ts`) that lets agents read and write shared memory across isolated channels, with MCP tools and backup [commit: `02b3a83` PR #47]

---

## ai-core
**Path:** `/home/openclaw/.claude/channels/discord-multi/projects/ai-core`
**Repo:** https://github.com/bistecglobal/ai-core

Core AI infrastructure and shared services for Bistec.

**Article ideas:**
- *Provider Routing at the Edge: Switching Between Anthropic and MiniMax Without Redeployment* — hot-swap provider pattern with env-level routing
- *Building a Multi-Provider AI Gateway for Enterprise* — abstracting model APIs behind a unified interface
- *DRY Principle for AI Skills: Extracting Shared Discipline Skills Across an Enterprise Agent Stack* — how we eliminated coding-standard duplication across four role skills into one canonical shared skill (`bistec-coding-standards`), cutting 166 lines from `bistec-developer` alone [commit: `ca03393` TEC-41] ✅ Published ([bistecglobal/blog#3](https://github.com/bistecglobal/blog/issues/3))

---

## agent-nexus
**Path:** `/home/openclaw/.claude/channels/discord-multi/projects/agent-nexus`

Multi-tenant cloud control plane for building, deploying, and governing AI agents across Copilot Studio, Azure AI Foundry, Bedrock, and Vertex.

**Article ideas:**
- *Multi-Tenant AI Agent Governance: What We Learned Building Agent Nexus* — policy enforcement, quota management, and audit trails across platforms
- *Unifying AI Agents Across Azure, AWS, and GCP: Architecture Patterns* — adapters, normalisation, and cross-cloud observability
- *AI Agent Observability with OTLP: Real-Time AgentScore, Cost Tracking, and Multi-Tenant Ingest* — how Agent Nexus ingests OpenTelemetry traces from Copilot Studio / Bedrock / Vertex agents, assigns per-agent scores, tracks live cost dimensions, and gates on tenant ID [commits: `46e27a5`, `4041469`, `6ad734a`, `d40a15e`]

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
- *Voice Cloning at Scale: MiniMax TTS Backend for AI-Generated Training Content* — how we replaced manual narration with a MiniMax voice-clone backend integrated into the academy video pipeline, covering model selection, voice consistency across tracks, and Higgsfield generative video [commits: `6e2ea3a`, `83f5eb9`]

---

## Cross-Project / Platform Articles

These cut across multiple projects and capture the broader Bistec engineering story:

- *How We Run 19 AI Agents Simultaneously (and Keep Them From Going Rogue)* — MCD fleet overview, autonomous mode, supervised heartbeat
- *The Specclaw-to-Production Pipeline: From Proposal to Merged PR Without a Human* — end-to-end autonomous delivery
- *WhatsApp as a Dev Interface: Routing Claude Agents Over Unofficial APIs* — Baileys integration, ToS trade-offs, session management
- *Memory That Persists: Building Cross-Session AI Context for a Multi-Project Fleet*
- *Provider Agnostic AI: How We Switch Between Anthropic and MiniMax Mid-Sprint*

---

*Updated: 2026-06-20 (ideate scan) | Maintained by bistec-articles agent*
