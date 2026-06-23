# Audit — The Silent Agent: Catching Usage Limits and Auto-Switching Models Mid-Run

**Verdict: PASS**
**Score: 24/25**
**Date:** 2026-06-23
**Source:** claude-mcd, feature `usage-limit-switch-offer` (PR #164, merged to main; commits `058b686`..`0b6f679`)

## Claim inventory

| # | Claim | Verdict | Evidence |
|---|---|---|---|
| 1 | 429 limit-hit written as synthetic assistant transcript line with `isApiErrorMessage:true`, `apiErrorStatus:429`, `model:"<synthetic>"` | ✅ | proposal.md (confirmed live transcripts); T4 detection gates on these fields |
| 2 | `!project model --set` and `!project provider --set` exist and respawn the subprocess | ✅ | referenced in proposal; `computeLimitOffer` emits these verbs (`limit-offer.ts`) |
| 3 | Transcript poll loop runs every ~2s, tails `.jsonl`, parses assistant + tool-progress lines | ✅ | `claude-process.ts` `startTranscriptWatcher`; proposal "runs every 2s" |
| 4 | Detection gates on structured fields first, regex second | ✅ | T4 diff: `if (rec.isApiErrorMessage === true && rec.apiErrorStatus === 429)` then `parseLimitMessage` |
| 5 | `lastLimitRaw` dedupes per episode | ✅ | T4 diff `if (text && text !== this.lastLimitRaw)` |
| 6 | `parseLimitMessage` keeps reset string verbatim, no Date parse | ✅ | `limit-offer.ts`; test AC1 `resetsAt === 'Jun 24, 7am (UTC)'` |
| 7 | `computeLimitOffer` hides provider offer when `apiKeyEnv` unset | ✅ | `limit-offer.ts` `if (env[def.apiKeyEnv])`; test `noKey.autoSwitch === null` |
| 8 | `limitFallback: z.array(z.string()).optional()` per-project schema field; absent → offer-only | ✅ | T1 `channels-config.ts` diff |
| 9 | Auto-switch writes config + respawns via existing set plumbing (`saveConfig` + `killChat`) | ✅ | T6 `server.ts` diff |
| 10 | Limit-hit flows through ProjectPool as `limit-hit` pool event via `onLimitHit` | ✅ | T5 `project-pool.ts` diff (new event kind, `offLimitHit`) |
| 11 | Default offer-only; auto-switch opt-in | ✅ | proposal approved decision #1; T6 branches on `autoSwitch` null |
| 12 | Unit tests cover model extraction, reset capture, opus-over-sonnet fallback, hide-when-no-key | ✅ | `limit-offer.test.ts` AC1/AC6 + autoSwitch opus + noKey checks |
| 13 | Code snippets match implementation | ✅ | all snippets transcribed from actual source files |

## Forward-looking scan

No matches for will / plan to / coming soon / roadmap / soon / next step. The closing "won't always announce themselves" is general engineering advice, not a roadmap claim. **Clean.**

## Raw SHA / PR scan

No commit SHAs or PR numbers in prose. **Clean.**

## Rubric

| Dimension | Score | Notes |
|---|---|---|
| Evidence quality | 5 | Every claim maps to a read source file; no SHAs in prose |
| Technical depth | 5 | Detection gating, pure-module testability, event wiring, opt-in mutation |
| Clarity | 5 | "thinking vs throttled" framing carries the piece; snippets under 30 lines |
| BistecGlobal voice | 5 | Practitioner, evidence-grounded, consistent with series |
| Title specificity | 4 | "The Silent Agent" is evocative + concrete subtitle; slightly metaphor-led |
| **Total** | **24/25** | |

## Fixes applied

None required — no flagged claims. Frontmatter `status` → `audited`.
