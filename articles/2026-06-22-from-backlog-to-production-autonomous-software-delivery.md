---
title: "From BACKLOG.md to Production: Autonomous Software Delivery at Bistec"
project: keyflow
tags: [AI, autonomous-agents, software-delivery, claude-code, specclaw, devops]
status: audited
date: 2026-06-22
---

# From BACKLOG.md to Production: Autonomous Software Delivery at Bistec

In three months of active development, keyflow — Bistec's OKR and performance management platform — shipped 83 features almost entirely through autonomous cycles. Each feature went from a backlog entry to a merged pull request through an unattended loop: a scheduler fired, an agent picked up the task, implemented it wave by wave, opened a PR, and added new proposals before signing off. Here is how that loop works.

## The Problem: Backlogs Stall Without Humans to Drive Them

Most engineering teams maintain a backlog that grows faster than it shrinks. Prioritisation meetings, sprint planning, and developer availability create constant friction between "this should be built" and "this got built." For keyflow, which is actively used inside Bistec and evolving rapidly based on user feedback, waiting on a human-driven sprint cycle was too slow.

The goal was to keep the backlog continuously draining — autonomously, without sacrificing code quality or introducing hidden complexity.

## The Solution: Three Cooperating Pieces

The autonomous loop is assembled from three separate systems: a work queue (`BACKLOG.md`), a spec-driven build engine (Specclaw), and a heartbeat scheduler (MCD).

### 1. BACKLOG.md as the Work Queue

`BACKLOG.md` is a structured Markdown table with numbered proposals, each linked to a slug, impact rating, effort estimate, and a `Done` or `PR #N` status. As of commit `0086df7`, it carries 86 proposals, 83 of which are marked complete.

Each proposal has a corresponding spec in `.specclaw/changes/<slug>/proposal.md`. The proposal captures the problem, the solution, the scope, and the measurable outcome. This is not aspirational writing — it is the contract that the implementation agent reads.

A representative example (`cycle-status-visual-cues`):

> *Cycle status (DRAFT / ACTIVE / CLOSED) is only clearly visible on the admin cycles page. Throughout the rest of the app — objective lists, dashboards, cycle switcher — users can't tell at a glance whether they're viewing an active cycle or a closed archive.*

That proposal drove four tasks, four commits, and one merged PR — all without a developer touching it.

### 2. Specclaw: Spec-Driven Implementation

Specclaw is Bistec's spec-driven development workflow, configured in `.specclaw/config.yaml`. The workflow gates each phase:

```
propose → plan → build → verify → pr
```

`workflow.strict: true` ensures no phase can be skipped. `code_review: true` and `code_review_block: true` mean a code-reviewer agent evaluates 10 quality dimensions during verify, and blocks the PR step on a `CHANGES_REQUESTED` verdict.

The build phase decomposes the spec into waves. Each wave is a set of tasks that can be executed in parallel (up to `parallel_tasks: 3`). For `cycle-status-visual-cues`, wave 1 created the shared `CycleStatusBadge` component; wave 2 wired it into two pages. The implementation commits follow a consistent `specclaw(<slug>): T<N> — <description>` pattern (e.g., commit `0859719` through `904b663` in the keyflow repo).

### 3. MCD Scheduler: The Heartbeat

The glue is MCD's `Scheduler` class (`src/scheduler.ts`, introduced in commit `e99a5cd`). It ticks every 60 seconds, walks a `schedules.json` table, and fires any entry that is due. Firing means delivering a synthetic Discord-shaped envelope directly into the keyflow project channel — the same code path a real operator message takes.

```typescript
private envelopeFor(s: Schedule, now: Date): InboundEnvelope {
  return {
    messageId: `sched-${s.id}-${now.getTime()}`,
    userId: '__mcd_scheduler__',
    username: 'scheduler',
    content: s.prompt + footer,
    ts,
  }
}
```

The agent inside the keyflow channel receives this as an ordinary inbound message and acts on it. The scheduler is agent-agnostic: it dispatches through `ProjectPool.deliver()`, which does not care whether the underlying process is Claude, a MiniMax-backed runner, or something else. Swapping the implementation model is a config change, not a code change.

Schedules support daily `at: HH:MM` entries and interval-based `every Xm` / `every Xh` entries. A `maxRuns` cap prevents runaway jobs; `lastRunAt` ensures a missed tick — if the host was asleep at 09:00 and restarts at 09:30 — does not double-fire.

## The Self-Regenerating Loop

The most striking property of this system is that it feeds itself. Each implementation session ends not just with a merged feature, but with new proposals appended to `BACKLOG.md`. The git history makes this visible:

- `7775083` — *feat(ux): empty-cycle quick-start banner + 4 new UX backlog proposals*
- `b17bba0` — *feat(ux): KR score context panel + 5 new UX backlog proposals*
- `8c7f47e` — *feat(ux): first-KR inline prompt + error humanizer + 5 new backlog proposals*

After implementing a feature, the agent audits the PR, identifies gaps exposed by the new code, and writes proposals for them. Those proposals enter the queue and become the next round of work. The backlog does not drain to zero — it evolves.

## What This Looks Like in Practice

The `.specclaw/STATUS.md` dashboard shows 83 completed changes. Pull requests are numbered from `#104` to `#318`, each on a `claude/<slug>` branch. A feature that would take a developer half a day — speccing, implementing, writing tests, opening a PR — completes in one unattended scheduler cycle.

The keyflow CLAUDE.md enforces one safety gate that keeps this from going off the rails: `bun run build` must pass before any push. The pre-push hook (`.githooks/`) enforces this. A build failure stops the branch before it reaches review.

## What We Learned

**Proposals need to be self-contained.** The spec must capture the problem, the scope, and the acceptance criteria with enough precision that an agent starting cold can implement it correctly. Vague proposals — "improve the UX" — produce vague implementations.

**Wave decomposition controls parallelism, not just sequencing.** The wave structure in `tasks.md` is the engineer's primary design artefact. Getting wave boundaries right determines whether tasks can run in parallel and whether dependencies are respected.

**The scheduler is the simplest possible orchestrator.** MCD's `Scheduler` is ~170 lines of TypeScript. It does not need to know about Specclaw, about keyflow, or about Claude. It fires a string into a channel at a time. Everything else is handled by the agent that receives it. This separation is what makes the system composable across different projects with different workflows.

**Self-regeneration keeps the backlog meaningful.** Ending each session by proposing new work shifts the backlog from a static TODO list to a live signal. The proposals that get generated after a feature lands tend to be more targeted than those written up front, because they reflect what the implementation revealed.

## Conclusion

Autonomous software delivery at Bistec is not magic — it is three components with a clear contract between them: a structured work queue, a gated implementation workflow, and a heartbeat that keeps the loop running. Over 83 shipped features, the pattern has proven that an AI agent, given precise proposals and a quality gate at the push boundary, can reliably drain a backlog and keep itself supplied with work. The role of the human operator shifts from "drive each feature" to "review PRs and set direction."

The full implementation is visible in the keyflow repository. The scheduler is in `claude-mcd/src/scheduler.ts`. The Specclaw configuration is in `keyflow/.specclaw/config.yaml`.
