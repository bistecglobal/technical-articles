# Audit: statistical-anomaly-detection-ai-agent-fleet

**Verdict: PASS with required fix (applied)**
**Score: 23/25**
**Claims verified: 17 / 18 — 1 section rewritten**

---

## Claim Inventory

| # | Claim | Verdict | Evidence |
|---|-------|---------|----------|
| 1 | "fleet of autonomous Claude Code agents across 19 active projects" | ✅ | channels.json: exactly 19 project entries |
| 2 | "Mission Control dashboard showed agent health scores, token budgets, and session replays" | ✅ | Published features blog#24 (health scores), blog#16 (token budgets), blog#21 (session replay) |
| 3 | "Claude Code records every conversation turn to a JSONL file under `~/.claude/projects/`" | ✅ | route.ts: `findAllJsonl()` reads from `~/.claude/projects/<encoded-path>/` |
| 4 | "Each assistant turn carries a timestamp, output token count, and an array of tool use blocks" | ✅ | route.ts lines 72–77: parses `rec.timestamp`, `rec.message.usage.output_tokens`, `rec.message.content` filtered by `type === 'tool_use'` |
| 5 | Three behavioral dimensions: inter-turn gap, tool calls/turn, output tokens/turn | ✅ | route.ts: `metrics` array defines exactly these three dimensions |
| 6 | 7-day rolling window | ✅ | route.ts: `const sevenDaysMs = 7 * 24 * 60 * 60 * 1000` |
| 7 | Baseline = all turns except last 3; recent = last 3 turns | ✅ | route.ts: `window.slice(-3)` and `window.slice(0, -3)` |
| 8 | sampleStd function code snippet | ✅ | Exact copy from route.ts lines 56–60 |
| 9 | Requires ≥7 turns in window, ≥4 baseline turns | ✅ | route.ts: `if (window.length < 7) return []` and `if (baselineTurns.length < 4) return []` |
| 10 | std < 0.01 guard skips flat-line metrics | ✅ | route.ts line 117: `if (std < 0.01) continue` |
| 11 | ≥2σ → warn, ≥3σ → critical | ✅ | route.ts line 132: `zScore >= 3 ? 'critical' : 'warn'` and `if (zScore < 2) continue` |
| 12 | /anomalies page, sortable table, sparkline of last 20 turns, anomalous region in red | ✅ | page.tsx: `sparklineSource = window.slice(-20)`; Sparkline component renders last 3 turns in `#EF4444` (red) |
| 13 | Auto-refreshes every 60 seconds | ✅ | page.tsx line 113: `setInterval(load, 60_000)` |
| 14 | Severity filter chips (all / warn / critical) | ✅ | page.tsx lines 167–181: three filter buttons |
| 15 | Clicking project slug navigates to detail page | ✅ | page.tsx: `<Link href={/projects/${encodeURIComponent(a.slug)}>` |
| 16 | Token burn rate gauge shows real-time fleet activity | ✅ | P107 shipped in commit bb52f33 on main |
| 17 | Health score trends over weeks | ✅ | P105 health trend timeline shipped in commit 808d275 on main |
| 18 | "What We've Caught" — three specific operational incidents with exact z-scores and multipliers | ❌ → **rewritten** | Feature shipped 2026-06-22; no real operational history. Specific numbers (z-score 6.4, 4× tokens, 8-12 tool calls) were fabricated. Rewritten as design-intent examples. |

---

## Forward-Looking Scan

No instances of "will", "plan to", "coming soon", "in the future", "roadmap", or "soon" found. ✅

---

## Rubric Scorecard

| Dimension | Score | Notes |
|-----------|-------|-------|
| Evidence quality (claims verifiable, no raw SHAs in prose) | 4/5 | 17/18 claims solid; 1 section required rewrite. No SHAs in prose. |
| Technical depth | 5/5 | Real code snippets, design rationale for sample-vs-population std, guard conditions explained |
| Clarity for target audience | 5/5 | Logical progression from problem → data → algorithm → UI → context |
| BistecGlobal voice | 4/5 | Professional and practitioner-focused; strong narrative arc |
| Title specificity | 5/5 | Specific, attention-grabbing, not generic |
| **Total** | **23/25** | |

---

## Fix Applied

**Section "What We've Caught"** — removed fabricated incident data (specific z-scores, multipliers, project names as victims of specific failures). Rewritten as "What It's Designed to Detect" describing three behavioral patterns the algorithm targets, grounded in the design decisions in the implementation rather than claimed operational history.

Status updated to `audited`. Fix committed.
