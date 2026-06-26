# Audit — Digital Twins for AI Agents

**Verdict: PASS**
**Score: 24/25**

Evidence: `apps/mission-control/app/api/project-twins/route.ts`, `app/project-twins/page.tsx` (commit `0600993`, on `main`).

## Claim inventory

| # | Claim | Verdict | Evidence |
|---|---|---|---|
| 1 | 5-feature vector per project | ✅ | FEATURE_NAMES (route:9-15) |
| 2 | Features: turns_per_day, tool_call_rate, memory_file_count, context_pressure_pct, avg_tokens_per_turn | ✅ | route:173-178 |
| 3 | 7-day rolling window | ✅ | WINDOW_MS = 7*24h (route:61) |
| 4 | Features reconstructed from JSONL transcripts | ✅ | parseStats (route:64-123) |
| 5 | Token features read usage block off assistant messages | ✅ | route:90-92, 103-104 |
| 6 | Genuine-user-message filter excludes tool_result | ✅ | route:98 |
| 7 | tool_call_rate = toolCalls/userMessages; turns_per_day = userMessages/7 | ✅ | route:174-175 |
| 8 | memory_file_count = .md files in projects/slug/memory | ✅ | memoryFileCount (route:125-128) |
| 9 | context_pressure from most-recently-modified file vs 200k window, capped 100 | ✅ | route:110-113, 177; MODEL_CONTEXT=200_000 |
| 10 | avg_tokens_per_turn = totalTokens/assistantTurns | ✅ | route:178 |
| 11 | Min-max normalise per feature across fleet (0–1) | ✅ | normalize (route:141-146), applied per feature (184-187) |
| 12 | Pairwise cosine similarity on normalized vectors | ✅ | cosine (route:130-139), loop 199-203 |
| 13 | Only pairs ≥ 0.80 kept | ✅ | route:204 |
| 14 | sharedFeatures = dims with normalized diff ≤ 0.2 | ✅ | route:206-209 |
| 15 | Sorted strongest-first | ✅ | route:220 |
| 16 | Needs ≥2 projects else empty | ✅ | route:161-163 |
| 17 | Page: similarity matrix, ranked pair list, side-by-side feature bars | ✅ | commit msg + project-twins/page.tsx |
| 18 | Built from existing logs, no agent changes | ✅ | route read-only over transcripts + memory dir |

18/18 verified. No ⚠️/❌.

## Forward-looking scan
No will / plan / coming soon / roadmap / next step describing unbuilt work. Clean.

## Rubric

| Dimension | Score |
|---|---|
| Evidence quality | 5 |
| Technical depth | 5 |
| Clarity for audience | 5 |
| BistecGlobal voice | 5 |
| Title specificity | 4 |

**Total: 24/25** — exceeds 20 threshold. No fixes required.
