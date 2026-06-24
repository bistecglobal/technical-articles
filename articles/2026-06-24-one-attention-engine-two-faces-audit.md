# Audit — One Attention Engine, Two Faces

**Verdict: PASS** · **Score: 24/25** · Claims fixed: 0 · Claims flagged: 0

## Claim inventory

| # | Claim | Verdict | Evidence |
|---|-------|---------|----------|
| 1 | Two surfaces: Fleet Advisor (action cards) + Fleet Brief (narrative digest), same underlying data | ✅ | commit `b211279`; `/api/advisor` + `/api/brief` both consume `computeFindings()` |
| 2 | Old design had duplicated rule logic + readers per route | ✅ | diff: `advisor/route.ts` −282, `brief.ts` −214; commit msg "duplicated transcript/budget readers are gone" |
| 3 | Stall thresholds had drifted: advisor 30m–4h, brief 4–48h | ✅ | commit msg "Stall windows merged (advisor 30m-4h + brief 4-48h → one rule)" |
| 4 | After unify: one stall rule 30m–48h with escalating severity | ✅ | stall rule `if (s.ageMs < 30*60_000 \|\| s.ageMs >= 48*3_600_000) return null`; severity info→warn |
| 5 | Advisor gained thrashing/declining; brief gained circuit/context/memory/budget | ✅ | commit msg explicit list |
| 6 | `Finding` type carries severity, signal, title, explanation, message, href, optional action | ✅ | `attention-findings.ts` interface Finding |
| 7 | Rules are pure fns in a single exported `RULES` array; "edit HERE to add a signal" | ✅ | `export const RULES: Rule[]` + AC5 comment |
| 8 | Circuit rule = critical, command action `!project stop` | ✅ | RULES[0] |
| 9 | Context rule: ≥87 warn/critical inject, ≥80 info inject | ✅ | context rule branches (≥95 critical, ≥87 warn, 80–87 info) |
| 10 | Signals gathered once into `FleetSignals`, all rules run against it | ✅ | `computeFindings` per-entry loop builds `s` then `for (rule of RULES)` |
| 11 | "Healthy" is derived (emitted only when no issue rule fired), not a registry rule | ✅ | `if (out.length === before) { healthyFinding(s) }` |
| 12 | Findings sorted severity-first then slug | ✅ | `out.sort((a,b)=> sevOrder...|| a.slug.localeCompare(b.slug))` |
| 13 | `toAdvisorCards` keeps only action-bearing findings, top N by severity; `toBriefResult` maps message + all-nominal collapse | ✅ | adapter source + commit msg |
| 14 | `better-sqlite3` dynamically imported inside `computeFindings` so pure core loads under Bun test runner | ✅ | `const { getMemoryConvergenceXY, getAlertFlow } = await import('../src/db')` + top-of-file NOTE |
| 15 | Tests run rules against fixtures, no DB/fs; assert actionable finding in BOTH surfaces | ✅ | `attention-findings.test.ts` `signals()` fixture; "surfaces in BOTH advisor and brief" test |
| 16 | Net ~500 lines duplicated logic removed, replaced by ~450-line module + tests; interim `lib/brief.ts` deleted | ✅ | diff: `attention-findings.ts` +452, `brief.ts` −214, advisor route −282; commit msg "lib/brief.ts removed — superseded" |

All 16 claims ✅ verified against merged code on `main` (commit `b211279` P208, `0182518` P205).

## Forward-looking scan
No roadmap/aspirational prose. Sole "will" occurs inside a quoted code string ("It will not accept new messages until it auto-resets") — describes present system behavior, not a future feature. PASS.

## Rubric

| Dimension | Score |
|---|---|
| Evidence quality (no raw SHAs in prose) | 5 |
| Technical depth | 5 |
| Clarity for audience | 5 |
| BistecGlobal voice | 5 |
| Title specificity | 4 |

**Total: 24/25** — exceeds 20 threshold. No fixes required.
