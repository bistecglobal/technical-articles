# Audit — When Three Agents Fail at Once (tool-failure-spike-detector)

**Verdict: PASS**
**Score: 24/25**
**Claims: 13 verified, 0 weak, 0 unverified · 0 flagged · 0 fixed**

Source: claude-mcd P291+P292 (`c442538`, on origin/main), `app/api/tool-spike-detector/route.ts`.

## Claim inventory

| # | Claim | Evidence | Verdict |
|---|---|---|---|
| 1 | Spike = ≥3 distinct projects erroring on same tool in same 5-min bucket | `SPIKE_MIN_PROJECTS=3`, `BUCKET_MS`, `affectedSlugs.size < 3` continue | ✅ |
| 2 | Severity scales with affected project count (med 5, high 8) | `severity()` + constants verbatim | ✅ |
| 3 | Counts distinct projects, not raw error count, for spike trigger | Set of slugs per bucket, size check | ✅ |
| 4 | Reads JSONL `tool_result` blocks with `is_error: true` | `obj.type==='tool_result' && obj.is_error` | ✅ |
| 5 | Resolves tool name via `tool_use_id` → name map from tool_use blocks | `toolNames.set(b.id,b.name)` then `.get(tool_use_id)` | ✅ |
| 6 | Tolerates malformed final line | try/catch skip in parse loop | ✅ |
| 7 | Skips files with mtime older than 24h window | `st.mtimeMs < cutoff24h) continue` | ✅ |
| 8 | 2-minute response cache | `CACHE_TTL_MS = 2*60*1000` | ✅ |
| 9 | Page auto-refreshes every 30s | commit msg "Auto-refreshes every 30s" | ✅ |
| 10 | currentSpikes = 2 most recent buckets, sorted by error count | `windowStart >= prevBucketStart` + sort | ✅ |
| 11 | history capped 200; heatmapBuckets = all tool×bucket capped 500, hover shows slugs | `.slice(0,200)`, `.slice(0,500)`, commit msg hover | ✅ |
| 12 | Feature = ~210-line route + page + sibling alert-delivery audit log same PR | `git show --stat`: route 210, page 256, alert-delivery route/page present | ✅ |
| 13 | Spike detector adds no schema change / no agent instrumentation | route reads transcripts only; no migration in P292 (P291 columns are the sibling, scoped separately) | ✅ |

## Forward-looking scan
No matches for will / plan to / coming soon / in the future / next step / roadmap / soon. Clean.

## Rubric

| Dimension | Score |
|---|---|
| Evidence quality | 5 |
| Technical depth | 5 |
| Clarity for audience | 5 |
| BistecGlobal voice | 4 |
| Title specificity | 5 |
| **Total** | **24/25** |

No fixes required. Status → audited.
