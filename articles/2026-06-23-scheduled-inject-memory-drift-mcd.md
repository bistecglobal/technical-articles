---
title: "Keeping Autonomous AI Agents on Track: Scheduled Context Injection and Memory Drift Monitoring"
project: claude-mcd
tags: [AI, DevOps, Autonomous Agents, Observability, Engineering]
status: audited
date: 2026-06-23
---

# Keeping Autonomous AI Agents on Track: Scheduled Context Injection and Memory Drift Monitoring

An autonomous AI agent that runs 24/7 has a dirty secret: it starts every session knowing nothing about what happened before. You can give it persistent memory files, inject goals at startup, and wire it into a fleet of other agents — but unless something actively pushes context into the session, the agent's worldview will silently drift. It forgets what day it is, which tasks are in flight, what its own priorities are. And you, the operator, have no visibility into how far it has drifted until something goes wrong.

We ran into this problem at Bistec while operating a fleet of Claude Code agents across a dozen active projects. Each agent had a `MEMORY.md` file that accumulated observations over time — but we had no way to answer two questions: *what should the agent be thinking about right now?* and *how much has its memory diverged from what we originally intended?*

These aren't abstract concerns. A backlog agent that hasn't been told today's date will reference stale sprint goals. An agent whose memory has been rewritten by a month of autonomous operation may have quietly abandoned its original mandate. We shipped two features to address this directly.

## Two Failure Modes, One Solution Space

The problem splits cleanly into push and observe:

**Context starvation** happens because Claude Code sessions are ephemeral. Session start injects a `GOAL.md`, but between scheduled task prompts, nothing reminds the agent of transient context: the current date, which features are in flight, how many turns it has consumed today. The agent isn't broken — it's just operating with stale information.

**Memory drift** happens because agents that write their own `MEMORY.md` over long periods will naturally evolve their internal model. This is mostly good. But it means the operator loses the ability to audit *how* the agent's understanding has changed. A 60% rewrite of `MEMORY.md` over two weeks is either healthy learning or a silent goal substitution — you can't tell without a diff timeline.

## Scheduled Inject Templates: Push Context, Not Commands

The scheduler in MCD already supported daily `at: HH:MM` and interval-based `every Xm` schedules that fire synthetic Discord messages into project sessions. These were task prompts — they expected the agent to complete work and reply on Discord.

We added a second schedule type: `inject`. An inject-type schedule delivers context directly into the session without the task-completion footer that prompts expect. It's the difference between asking the agent to *do* something and simply *telling* it something.

```typescript
function resolveInjectVars(template: string, slug: string): string {
  const now = new Date()
  return template
    .replace(/\{\{slug\}\}/g, slug)
    .replace(/\{\{date\}\}/g, now.toLocaleDateString('en-CA'))
    .replace(/\{\{time\}\}/g, now.toLocaleTimeString())
}
```

Templates support `{{slug}}`, `{{date}}`, and `{{time}}` substitutions resolved at fire time. A typical morning inject reads:

```
Good morning, {{slug}}. Today is {{date}}. You are an autonomous agent
for the {{slug}} project. Review your MEMORY.md and GOAL.md before
starting any new work. Check BACKLOG.md for pending items.
```

Registering one looks like:

```
!project schedule inject --slug keyflow 09:00 "Good morning, {{slug}}. Today is {{date}}."
```

The key design decision was routing inject schedules through the same `deliver()` code path as real Discord messages. The agent doesn't know the message is scheduled — it arrives as an ordinary inbound envelope. This means inject templates work regardless of the underlying agent runner — Claude Code, openclaw, or any other CLI that speaks the same envelope format.

The Mission Control dashboard gained a **Scheduled** tab on the Inject Templates page, listing active inject schedules grouped by project — each showing the raw template body, schedule time, run count, and last fire time.

## Memory Diff Timeline: Observe How Agents Change

The second feature addresses observability. Every project's `MEMORY.md` lives in a git repository — which means its full change history is already there. We just weren't reading it.

The `/api/memory-diff` endpoint runs `git log --follow -p -- MEMORY.md` for each project, parses the output into a structured diff timeline, and caches the result for one hour. Each entry records the commit SHA, timestamp, lines added, lines removed, and the raw diff.

The drift score is the heart of the feature:

```typescript
function computeDriftScore(entries: MemoryDiffEntry[], totalLines: number): number {
  if (totalLines <= 0) return 0
  const changed = entries.reduce((sum, e) => sum + e.added + e.removed, 0)
  return Math.min(100, Math.round((changed / totalLines) * 100))
}
```

It computes cumulative line churn as a percentage of the memory file's current size. A score above 50% triggers an amber warning in the dashboard — meaning more than half the file's worth of content has been rewritten since the file was created. Projects are sorted descending by drift score so the most-changed agents surface first.

The `/memory-diff` page renders a per-project accordion. Each project shows its drift score chip (green → blue → amber), and each commit is a collapsible row displaying timestamp, SHA prefix, added/removed counts, and the full colored diff on expand. Date range and project filters let operators narrow to a specific agent or window.

## What Surprised Us

**The drift scores were immediately revealing.** On our first run across the active fleet, several projects surfaced with drift scores above 60%. Most were expected — agents that had been running for weeks under heavy autonomous operation. One was a surprise: a project whose memory had been edited manually during debugging, pushing churn artificially high. The timeline made the cause obvious in seconds.

**Inject templates changed agent behavior noticeably.** Before the feature, agents on long-running projects would sometimes open sessions referencing sprint goals that were two weeks stale. After deploying a daily 09:00 inject with `{{date}}` and a reminder to check `BACKLOG.md`, those references disappeared. The agents were simply better oriented.

**The separation between `inject` and `prompt` schedule types mattered.** Early prototypes just appended the context to a normal task prompt. That created confused agents that would try to *complete* the orientation message as a task and post to Discord asking if they'd done it correctly. The explicit `inject` type — which strips the task footer — produced clean, silent context delivery.

## Lessons for Your Own Agent Fleet

If you run autonomous AI agents across multiple projects, both of these patterns are straightforward to implement regardless of your stack:

**Treat context injection as infrastructure.** Scheduled cron jobs that push a time-stamped, project-aware message into every agent session cost almost nothing and prevent a class of orientation failures that are otherwise invisible until they compound.

**Memory is observable through git.** If your agent's persistent memory lives in version-controlled files, you already have a full diff history. Adding a `git log -p` parse and a simple churn metric costs a few hours of implementation — and gives you an audit trail that third-party observability tools don't provide for locally-hosted agents.

**Drift is not inherently bad.** A high drift score means the agent is actively updating its model of the world. What matters is whether that change is intentional. The timeline makes it easy to distinguish between healthy learning and unexpected goal substitution — look at the commit timestamps and what changed, not just the score.

Both features ship in the MCD Mission Control dashboard — accessible at the `/inject-templates` and `/memory-diff` routes. The inject template library supports four categories (standup, review, report, custom) and live variable preview before scheduling.

The broader lesson: autonomous agents need the same operational discipline as any long-running service. Scheduled context injection is a keep-alive for agent orientation. Memory drift monitoring is a health check for agent integrity. Neither replaces good goal-setting upfront — but both make it far easier to catch and correct drift before it becomes a production problem.
