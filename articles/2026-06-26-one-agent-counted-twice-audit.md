# Audit — One Agent Counted Twice, Every Total Counted Too Little

**Verdict: PASS**
**Score: 23/25**
**Source:** agent-nexus, PR 238 / commit `4c744f4` (merged to `main`)

## Claim inventory & verdicts

| # | Claim | Verdict | Evidence |
|---|---|---|---|
| 1 | Same external engine reachable via >1 connection produces multiple `DiscoveredAgents` rows | ✅ | commit msg + dedupe code comment |
| 2 | Old promote minted a fresh `Agent` (new GUID) per discovered row | ✅ | diff `DiscoveredAgentEndpoints.cs` (pre-change path) |
| 3 | Only one twin carried telemetry; duplicate was a ghost | ✅ | commit msg ("duplicate carried no telemetry") |
| 4 | `SiblingPromotedAgentAsync` reuses existing agent for same (TenantId, Platform, ExternalAgentId) | ✅ | diff — new private method |
| 5 | Partial unique index, filter `ExternalAgentId IS NOT NULL` | ✅ | migration `20260625055730_AddAgentExternalAgentIdUniqueIndex.cs` |
| 6 | `DbUpdateException` caught, duplicate detached, re-pointed to winner, returns 200 | ✅ | diff catch block |
| 7 | `PromoteRaceRecoveryTests` asserts 200 + winner id + count==1 + re-pointed row | ✅ | test file assertions (lines 104–116) |
| 8 | Obs summary previously computed from latest 200 rows | ✅ | diff `OperationalEventEndpoints.cs` (old `Take(200)`) |
| 9 | Fix aggregates full filtered set server-side (COUNT/SUM) inside tenant scope | ✅ | diff — `CountAsync`/`SumAsync` in `ReadInTenantScopeAsync` |
| 10 | Events list bounded by `?limit=N`, default 1000 | ✅ | diff |
| 11 | p95 via Postgres `percentile_cont`; LINQ linear-interp fallback for in-memory tests | ✅ | diff + `ContinuousPercentile` helper |
| 12 | Vertex token usage never extracted; lives in top-level log labels `gen_ai.usage.*` | ✅ | collector.yaml diff |
| 13 | Engine id mapped to `gemini-2.0-flash` for price table | ✅ | collector.yaml diff |
| 14 | Governance classifier dropped token-bearing inference logs; now skips them | ✅ | collector.yaml diff (added `input_tokens == nil` guards) |
| 15 | `ExternalAgentId` column added to `Agent` entity | ✅ | diff `Entities/Agent.cs` + migration |

15/15 verified. No ⚠️ or ❌.

## Forward-looking scan
NONE. ("worth carrying forward" = lessons framing, not roadmap.) No raw SHAs or PR numbers in prose.

## Rubric

| Dimension | Score | Notes |
|---|---|---|
| Evidence quality | 5 | Every claim maps to merged code; no SHAs in prose |
| Technical depth | 5 | TOCTOU recovery, partial index, in-DB percentile, OTLP label extraction |
| Clarity for audience | 4 | Dense in places but well-scaffolded |
| BistecGlobal voice | 5 | Practitioner, narrative, evidence-grounded |
| Title specificity | 4 | Specific + human; long |
| **Total** | **23/25** | |

## Fixes applied
None required. Frontmatter status → audited.
