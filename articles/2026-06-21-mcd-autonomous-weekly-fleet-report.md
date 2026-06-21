---
title: "Autonomous Weekly Fleet Report: Turning Claude Transcripts into Observability Data"
project: claude-mcd
tags: [AI, Observability, AgentOps, DevTools, FleetManagement]
status: draft
date: 2026-06-21
---

# Autonomous Weekly Fleet Report: Turning Claude Transcripts into Observability Data

When you're running a fleet of AI agents across a dozen projects, the natural question at the end of the week is: what did they actually do? Not in a qualitative sense — you can read the Discord threads — but quantitatively. Which project consumed the most tokens? Which agent stalled the most? How many PRs did the fleet merge? What did it cost?

MCD (Multi-Channel Discord) is Bistec's AI fleet orchestration platform — one Claude Code subprocess per Discord channel, all managed by a central `ProjectPool`. As the fleet grew, visibility degraded. This post covers how we built autonomous weekly fleet reporting into the Mission Control dashboard.

## The Data Source: JSONL Transcripts

Claude Code writes every conversation to disk as JSONL files at `~/.claude/projects/<encoded-path>/<session-id>.jsonl`. Each line is a message record — `user`, `assistant`, or system — with a timestamp, token usage, model name, and message content.

MCD maps each Discord channel to a project directory. The weekly report endpoint (`GET /api/reports/weekly`) walks every project slug in `MCD_CHANNELS_DIR/projects/`, resolves symlinks to real paths, encodes the path to find the right transcript directory, then scans all JSONL files modified in the past seven days.

```typescript
const encoded = encodeProjectCwd(realDir)
const transcriptDir = path.join(os.homedir(), '.claude', 'projects', encoded)

let jsonlFiles: string[] = []
try {
  jsonlFiles = fs.readdirSync(transcriptDir)
    .filter((f) => f.endsWith('.jsonl'))
    .map((f) => path.join(transcriptDir, f))
    .filter((f) => {
      const stat = fs.statSync(f)
      return stat.mtimeMs >= weekStartMs
    })
} catch {}
```

The file-mtime pre-filter is a performance guard — no point parsing a transcript that hasn't been touched this week.

## What Gets Extracted Per Project

For each project, `analyzeJsonl()` reads every `assistant` message within the week window and accumulates:

- **Turns**: assistant message count — the clearest proxy for agent activity
- **Token counts**: input and output, from the `usage` field on each record
- **Tool calls**: count of `tool_use` blocks in the assistant message content
- **Stalls**: assistant messages where `stop_reason` is `max_tokens` or `end_turn` with no tool calls and at least one prior turn — these indicate the agent stopped without taking action
- **Average turn latency**: time between consecutive assistant messages
- **Daily turn distribution**: a 7-element array bucketing turns into days 0–6 since week start

```typescript
const stopReason = (rec.message as { stop_reason?: string } | undefined)?.stop_reason
if (stopReason === 'max_tokens' || stopReason === 'end_turn') {
  const hasTools = (content as Array<{ type?: string }>).some((b) => b.type === 'tool_use')
  if (!hasTools && turns > 1) stalls++
}
```

Two additional metrics come from outside the transcript:

- **PR count**: `git log --oneline --after=<weekStart> --merges` on the project's real directory
- **Memories written**: count of `.md` files modified this week inside `.claude/memory/` or `memory/`

## Cost Estimation and the Impact Score

Token counts become dollar figures via a model-aware pricing table:

```typescript
const MODEL_PRICING: Record<string, [number, number]> = {
  haiku: [0.8, 4],
  sonnet: [3, 15],
  opus: [15, 75],
}
```

Input and output rates are in dollars per million tokens. The model is detected from the `model` field on assistant records — the last seen model per project wins. Cost is then `(inputTokens / 1M) * inputRate + (outputTokens / 1M) * outputRate`.

Each project also receives a composite impact score:

```
impactScore = turns * 2 + prCount * 20 + toolCalls + memoriesWritten * 5
```

The weights reflect the cost of meaningful work: merged PRs are weighed heavily (a PR requires coordination across multiple turns), memories represent durable knowledge persisted for future sessions, and raw turns and tool calls are cheaper signals of activity. Projects sort by impact score descending.

Fleet-wide aggregates add one more derived metric: `topByEfficiency` — the project with the lowest cost per turn. This surfaces projects where the agent is doing meaningful work cheaply, as opposed to high-turn, low-output sessions.

## Generating and Persisting the Report

`POST /api/reports/weekly/generate` fetches the weekly data from the GET endpoint and persists it as JSON:

```typescript
const weekLabel = `${reportData.weekStart.slice(0, 7)}-W${getISOWeek(reportData.weekStart)}`
const filePath = path.join(mcdDir, 'reports', `${weekLabel}.json`)
fs.writeFileSync(filePath, JSON.stringify(report, null, 2), 'utf-8')
```

Files land at `MCD_CHANNELS_DIR/reports/2026-06-W25.json`. The week label uses ISO week numbering. This gives operators a rolling archive of fleet performance without any external storage dependency — just the local filesystem where MCD already runs.

## The /reports Page

The Mission Control `/reports` page fetches the weekly data on load and renders it as a neon stat card grid for fleet totals, then a sortable per-project table with per-project sparklines.

Each project row shows a 7-day sparkline of daily turns, colored green for upward trend and red for downward. The top three projects by impact score get gold/silver/bronze left-border accents. An "Export HTML" button generates a self-contained offline report for sharing without requiring dashboard access.

The implementation uses the same `NeonStat` component and `TurnsSparkline` SVG renderer as the rest of Mission Control — no new dependencies.

## What It Tells You

At a glance, the weekly report answers:

- **Which project is most active?** (`topByActivity` — highest impact score)
- **Which is most cost-efficient?** (`topByEfficiency` — lowest cost per turn)
- **Where are agents stalling?** (stall count per project, sortable column)
- **How many PRs did the fleet merge this week?** (fleet total + per project)
- **What did the fleet cost?** (estimated USD per project and total)

Before this, answering any of these required digging through Discord threads or parsing JSONL files manually. The report endpoint reads the same data Claude Code already writes — no instrumentation changes required, no external observability service to run.

The feature shipped in MCD PR #3e3f957, alongside the Proposal-to-Impact Traceability Matrix (P53), which connects pipeline changes to their git commit counts and line diffs for a complementary view of what agents shipped vs. what they consumed.

---

*Implementation: `apps/mission-control/app/api/reports/weekly/route.ts`, `apps/mission-control/app/reports/page.tsx` — MCD repo, commit `3e3f957`*
