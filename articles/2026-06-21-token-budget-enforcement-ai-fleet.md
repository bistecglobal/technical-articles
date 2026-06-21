---
title: "Token Budget Enforcement for AI Agent Fleets: SSE Alerts, Queue Drain, and Monthly Rollover"
project: claude-mcd
tags: [AI, DevOps, Cloud, Multi-Agent, Observability]
status: draft
date: 2026-06-21
---

# Token Budget Enforcement for AI Agent Fleets: SSE Alerts, Queue Drain, and Monthly Rollover

When you run more than a handful of autonomous AI agents, token cost stops being an afterthought. At BistecGlobal we operate a fleet of 19+ Claude Code agents through our Multi-Channel Discord (MCD) platform — one agent per Discord channel, each capable of autonomously merging PRs, writing code, and sending notifications around the clock. Without spending controls, a single misbehaving agent can silently exhaust a month's API budget in hours.

This post covers the token budget enforcement system we shipped in PR #84 (commit `17a0f12`): a layered mechanism spanning the project pool, SSE broadcast, Discord notifications, and the Mission Control dashboard.

## The Problem: Invisible Spending at Scale

Before this feature, `monthlyTokenBudget` was already stored in `channels-config.ts` — but it was purely decorative. The fleet dashboard showed raw token counts from JSONL transcripts, but nothing blocked an agent that went over, and operators only found out by checking the Anthropic console manually.

What we needed:

1. Graduated visibility — warn before the budget runs out, not after
2. Hard enforcement — stop new messages when fully exhausted
3. Queue, don't drop — messages sent while a project is over budget should still go through once the month resets
4. Zero-config escape hatch — `monthlyTokenBudget: null` must mean unlimited

## Architecture: Four Layers, One Data Source

The entire system reads from a single source of truth: Claude Code's JSONL transcript files, the same files the [Weekly Fleet Report](https://github.com/bistecglobal/blog/issues/14) uses. Token counts come from `assistant`-type records in `~/.claude/projects/<encoded-path>/*.jsonl`, filtered to the current UTC year-month:

```typescript
// src/project-pool.ts — computeMonthlyTokensUsed()
for (const line of raw.trim().split('\n').filter(Boolean)) {
  let rec: Record<string, unknown>
  try { rec = JSON.parse(line) } catch { continue }
  if (rec.type !== 'assistant') continue
  const ts = typeof rec.timestamp === 'string' ? rec.timestamp : null
  if (!ts) continue
  const recYearMonth = ts.slice(0, 7)
  if (recYearMonth !== currentYearMonth) continue
  const usage = rec.message?.usage
  total += (usage?.input_tokens ?? 0) + (usage?.output_tokens ?? 0)
}
```

No extra API calls. No separate accounting database. The transcripts already carry everything needed.

### Layer 1: Project Pool Enforcement

`ProjectPool.deliver()` now checks budget before routing any inbound Discord message to a Claude subprocess. If `monthlyTokenBudget` is set and `used >= budget`, the envelope goes into a per-chat in-memory queue rather than to the process:

```typescript
// src/project-pool.ts
if (project.monthlyTokenBudget != null) {
  const used = this.computeMonthlyTokensUsed(project.slug, config)
  const budget = project.monthlyTokenBudget
  this.checkBudgetThresholds(chatId, project.slug, used, budget)
  if (used >= budget) {
    const queue = this.budgetQueue.get(chatId) ?? []
    queue.push(envelope)
    this.budgetQueue.set(chatId, queue)
    this.fireEvent({ kind: 'budget-exhausted', chatId, slug: project.slug,
                     used, budget, queuedCount: queue.length })
    return
  }
}
```

The pool also fires graduated `budget-alert` events at 50%, 80%, and 100% thresholds — but only once per threshold per month. A `budgetAlertFired: Map<chatId, Set<50|80|100>>` tracks which thresholds have fired, preventing repeat alerts from spamming operators:

```typescript
private checkBudgetThresholds(chatId: string, slug: string,
                               used: number, budget: number): void {
  const pct = (used / budget) * 100
  let fired = this.budgetAlertFired.get(chatId)
  if (!fired) { fired = new Set(); this.budgetAlertFired.set(chatId, fired) }
  for (const threshold of [50, 80, 100] as const) {
    if (pct >= threshold && !fired.has(threshold)) {
      fired.add(threshold)
      this.fireEvent({ kind: 'budget-alert', chatId, slug, threshold, used, budget })
    }
  }
}
```

### Layer 2: UTC Month Rollover and Queue Drain

Budget resets at calendar month boundaries. Every call to `deliver()` first checks whether the UTC year-month has changed. If it has, two things happen:

1. `budgetAlertFired` is cleared — thresholds can fire again for the new month
2. All queued messages are re-delivered in order via `drainBudgetQueues()`

```typescript
private checkMonthRollover(): void {
  const currentYM = ProjectPool.currentYearMonth()
  if (currentYM === this.lastYearMonth) return
  this.lastYearMonth = currentYM
  this.budgetAlertFired.clear()
  void this.drainBudgetQueues()
}
```

Draining works by calling `deliver()` recursively on each queued envelope. Since the new month's usage starts at zero, the budget check passes and the messages flow through normally. A `budget-restored` pool event fires with the count of drained messages.

### Layer 3: Discord Master-Channel Notifications

`server.ts` listens to the pool event stream and posts to the configured master Discord channel whenever a budget threshold is crossed or a queue is drained:

```typescript
if (evt.kind === 'budget-alert') {
  const emoji = evt.threshold === 100 ? '🚫'
    : evt.threshold === 80 ? '🔶' : '🟡'
  const notice = {
    text: `${emoji} \`${evt.slug}\`: monthly token budget at ${evt.threshold}% ` +
      `(${evt.used.toLocaleString()} / ${evt.budget.toLocaleString()} tokens).` +
      (evt.threshold === 100 ? ' New messages queued until next month.' : ''),
  }
  routeNotification(loadChannelsConfig(), notice, `budget-alert-${evt.threshold}`)
}
```

Operators see a `🟡 keyflow: 50%` notification in their main Discord channel — no dashboard required.

### Layer 4: SSE Dashboard Alerts and Visual Status

Mission Control's SSE broadcaster (`sse.ts`) independently checks budget status on each 5-second fleet tick and emits `budget-alert` SSE events to connected browser clients. It maintains its own dedup state (`budgetAlertState: Map<string, string>`) keyed by `slug:threshold:YYYY-MM` to prevent repeated client-side toasts.

`fleet-compute.ts` computes a `budgetStatus` field for every project:

```typescript
function computeBudgetStatus(used: number, budget: number): BudgetStatus {
  const pct = used / budget
  if (pct >= 1) return 'exhausted'
  if (pct >= 0.8) return 'critical'
  if (pct >= 0.5) return 'warning'
  return 'ok'
}
```

`InstanceGrid.tsx` uses this to color-code each project's slug chip: amber for warning (≥50%), red for critical (≥80%), grey for exhausted, with a tooltip explaining the state. Operators glancing at the fleet graph see budget health at a glance without clicking into any individual project.

## Design Decisions Worth Noting

**Queue over drop.** When exhausted, messages are queued in memory rather than discarded. An operator sent a message for a reason — dropping it silently would be worse than a short delay. The month-rollover drain handles the eventual delivery.

**Transcript-based accounting.** Reading JSONL files avoids adding a new accounting database. The cost is re-reading transcript files on every `deliver()` call for budgeted projects — acceptable for fleet sizes in the tens. If the fleet grew to hundreds, a cached counter with a short TTL would be the natural next step.

**Two separate dedup states.** The pool (`budgetAlertFired`) and the SSE layer (`budgetAlertState`) both track fired thresholds independently. This is intentional: the pool fires pool events for Discord notifications; the SSE layer fires browser events for dashboard toasts. Neither knows about the other's consumers, keeping the layers decoupled.

**`null` means unlimited.** Any project without `monthlyTokenBudget` set skips enforcement entirely. Operators adopt budget controls project by project rather than needing a global policy at rollout.

## Outcomes

Budget state is now a first-class property of every fleet project. Operators know a project is approaching its limit before it exhausts — at 50% and 80% — with graduated visual and Discord signals. At 100%, messages queue rather than fail, and the system recovers automatically at month start without operator intervention.

The integration test suite added in #85 (commit `7a9b31b`) covers the full lifecycle: threshold crossing, queue accumulation, month rollover drain, and `budget-restored` event emission — all without live API calls by wiring `MockProjectProcess` into the pool.

For teams running multi-agent AI fleets, predictable cost is as important as reliability. This feature treats token budgets as a first-class operational concern: observable, enforceable, and self-healing.
