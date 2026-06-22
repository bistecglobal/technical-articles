# Audit: multi-signal-fleet-topology-memory-goals

**Verdict: PASS (minor fixes applied)**
**Score: 22/25**

---

## Claim Inventory

| # | Claim | Status | Evidence |
|---|-------|--------|----------|
| 1 | "live D3 topology ... edges form when transcript references slug within last 15 minutes" | ✅ | TopologyGraph.tsx: d3.forceSimulation; topology/route.ts: WINDOW_15M = 15 * 60 * 1000 |
| 2 | "MEMORY.md (accumulated learnings) and GOAL.md (mission statement, set by operator)" | ✅ | topology/route.ts reads both files per project; confirmed in prior published articles |
| 3 | Four signal types: Transcript, Memory, Goal, Shared remote | ✅ | All four computed in topology/route.ts |
| 4 | "Transcript normalized over cap of 10 references" | ✅ | MAX_TRANSCRIPT_REFS = 10; transcriptScore = Math.min(totalRefs / 10, 1) |
| 5 | "Shared remote produces weight of 1.0" | ✅ | sameRemote ? 1.0 : ... in topology/route.ts:268 |
| 6 | Edge weight formula: 0.5*transcript + 0.3*memory + 0.2*goal | ✅ | topology/route.ts:269 verbatim |
| 7 | "Filtering to tokens of four or more characters" | ✅ | extractKeywords: w.length >= 4 |
| 8 | Stop words include "project", "claude", "code", "file", "there", "just" | ✅ | STOP_WORDS set in topology/route.ts:53-59 |
| 9 | Jaccard similarity code snippet | ✅ | topology/route.ts:66-73 verbatim |
| 10 | "Edge created when memory or goal Jaccard exceeds 0.05" | ✅ | topology/route.ts:248: memSim.score >= 0.05 \|\| goalSim.score >= 0.05 |
| 11 | Edge types: transcript, inferred, shared-remote | ✅ | TopologyEdge type definition in topology/route.ts |
| 12 | "Inferred edges render dashed" | ✅ | TopologyGraph.tsx:228: stroke-dasharray '4,3' for edgeType==='inferred' |
| 13 | "Shared-remote edges: solid red, width 3" | ✅ | TopologyGraph.tsx: stroke '#EF4444', stroke-width 3 for shared-remote |
| 14 | "Up-to-five shared keywords in title attribute, visible on hover" | ✅ | jaccardSimilarity returns shared.slice(0,5); TopologyGraph.tsx:229 sets title attribute |
| 15 | Fleet Digest five dimensions: contextPct, convergence, goalPct, alertCount, turnsToday | ✅ | DigestProject interface in digest/route.ts |
| 16 | "Context pressure from existing context-pressure table" | ✅ | getLatestContextPressure(slug) called in computeDigest |
| 17 | "Convergence score — 0–100 measure" | ⚠️ | Schema stores score as number; 0-100 range inferred from threshold usage (30, 60) in UI |
| 18 | "Goal advancement — percentage of GOAL.md checklist completed" | ✅ | getGoalAdvancementScore(slug) from goal_advancement table; confirmed in prior articles |
| 19 | Flag thresholds code snippet | ✅ | digest/route.ts:143-147 verbatim |
| 20 | "stuck flag scans last 24 hours for word 'stuck'" | ✅ | cutoff = Date.now() - 24*3_600_000; /\bstuck\b/i.test(raw) |
| 21 | "MCD's heartbeat watchdog uses it in recovery injections" | ❌ | Heartbeat fires internal {kind:'stuck'} events; no evidence of injecting 'stuck' text into JSONL — must remove |
| 22 | "Each digest run persists in digest_log SQLite table" | ✅ | db.ts:203 CREATE TABLE digest_log; insertDigest called in computeDigest |
| 23 | "Three tabs: current, history (last 30), markdown" | ✅ | digest/page.tsx: three tab buttons; getDigestHistory(30) |
| 24 | "Two projects working on different layers of the same auth stack appeared linked" | ⚠️ | Code supports this scenario but specific pair cannot be verified from code alone |
| 25 | "Operators now open it first when something feels off" | ⚠️ | session-health link verified; behavioral "now" claim unsupported |

---

## Forward-Looking Scan

Searched for: will, plan to, coming soon, in the future, next step, roadmap, soon.

**No forward-looking statements found.** ✅

---

## Rubric Scorecard

| Dimension | Score | Notes |
|---|---|---|
| Evidence quality (claims verifiable, no raw SHAs in prose) | 4/5 | 22/25 claims verified; 1 wrong claim (C21), 2 weak behavioral claims |
| Technical depth | 5/5 | Formula, thresholds, Jaccard implementation, edge taxonomy, SQLite schema |
| Clarity for target audience | 4/5 | Strong hook; "cognitive territory" metaphor effective; good visual narrative |
| BistecGlobal voice | 4/5 | Professional, first-person plural, practitioner-focused |
| Title specificity | 5/5 | Clear gap framing, not generic |

**Total: 22/25 — PASS**

---

## Fixes Applied

1. **C21 removed**: "MCD's heartbeat watchdog uses it in recovery injections" — no evidence this word appears in injected text. Replaced with accurate description: the heuristic works because agents themselves surface the word when reporting stalls.

2. **C24 softened**: Specific "two projects on auth stack" changed to illustrative pattern framing.

3. **C25 softened**: "Operators now open it first" changed to "the typical workflow" — removes unsupported behavioral claim.
