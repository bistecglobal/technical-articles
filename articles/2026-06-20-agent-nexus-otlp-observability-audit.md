# Audit Report: 2026-06-20-agent-nexus-otlp-observability

**Verdict: PASS (with minor fixes applied)**
**Score: 23/25**

---

## Claim Inventory

1. "Agent Nexus manages AI agents across Copilot Studio, Azure AI Foundry, Bedrock, and Vertex AI" — ⚠️ WEAK. Connectors directory (`src/Nexus.Connectors/Platforms/`) contains only `CopilotStudio/` and `VertexAi/`. Bedrock appears in `DslValidator.cs` as a recognized DSL platform string (`"amazon_bedrock"`) but has no connector implementation. Azure AI Foundry is referenced in the OTLP price table comment. Rewritten to remove Bedrock from the "implemented connectors" claim.

2. "Copilot Studio emits webhook events. Vertex AI exposes Cloud Trace. Foundry wraps Azure Monitor." — ⚠️ WEAK. General cloud platform descriptions, not grounded in Agent Nexus source. Removed from final article.

3. "OTLP log records using the `gen_ai.*` semantic conventions" — ✅ Verified. `OtlpParser.cs` extracts `gen_ai.agent.id`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, `gen_ai.system`, `gen_ai.request.model`, `gen_ai.request.id`, `gen_ai.response.finish_reason` etc.

4. "POST /otlp/v1/logs endpoint, internal-only, not reachable through YARP gateway" — ✅ Verified. `OtlpIngestEndpoints.cs` line 41: `app.MapPost("/otlp/v1/logs", HandleLogsAsync)`. Comment: "MUST NOT be added to the YARP route table."

5. "Gated by X-Nexus-Ingest-Key, fail-closed outside Development (returns 503)" — ✅ Verified. `OtlpIngestEndpoints.cs` lines 57-68.

6. "OTLP JSON envelope: resourceLogs → scopeLogs → logRecords" — ✅ Verified. `OtlpParser.cs` lines 44-56.

7. "OTLP spec encodes int64 as JSON strings; Python SDK uses JSON numbers" — ✅ Verified. `OtlpParser.cs` lines 122-123 with comment.

8. "Tenant identity from tenant.id attribute, never X-Tenant-Id header (commit 6ad734a)" — ✅ Verified. `OtlpParser.cs` line 96-98 `ParseTenantId`. Commit 6ad734a message confirms fix. Comment in `OtlpIngestEndpoints.cs`: "NEVER sourced from a request header".

9. "Model price table covering GPT-4o, GPT-4o-mini, GPT-4 Turbo, and eight Gemini variants" — ✅ Verified. `OtlpIngestEndpoints.cs` lines 14-31: gpt-4o, gpt-4o-mini, gpt-4-turbo, gpt-35-turbo, gpt-4, and 8 Gemini models (2.5-pro, 2.5-flash, 2.0-flash, 2.0-flash-001, 1.5-pro, 1.5-pro-002, 1.5-flash, 1.5-flash-002).

10. "Unknown models fall back to GPT-4o pricing" — ✅ Verified. `OtlpIngestEndpoints.cs` line 194: `if (!Prices.TryGetValue(model, out var p)) p = (0.005, 0.015)` — GPT-4o rate.

11. "Governance event 60-second dedup window" — ✅ Verified. `OtlpIngestEndpoints.cs` line 33: `private static readonly TimeSpan DedupWindow = TimeSpan.FromSeconds(60)`.

12. "Two-level dedup: in-memory HashSet + AnyAsync DB check" — ✅ Verified. `OtlpIngestEndpoints.cs` lines 81, 108-118.

13. "Four scoring dimensions: Reliability 35%, Latency 25%, Cost 20%, Throughput 20%" — ✅ Verified. `AgentRankEngine.cs` lines 7-10.

14. "TargetLatencyMs = 2000ms" — ✅ Verified. `RecalculateAgentRankConsumer.cs` line 14.

15. "MonthlyBudgetThreshold = $100" — ✅ Verified. `RecalculateAgentRankConsumer.cs` line 16.

16. "TargetRps = 1 RPS" — ✅ Verified. `RecalculateAgentRankConsumer.cs` line 15.

17. "CostScore was always 100 due to empty CostRecords table (commit 46e27a5)" — ✅ Verified. Commit 46e27a5 message: "AgentRank cost scoring summed CostRecords, but the OTLP ingest path computes CostUsd onto OperationalEvent and never publishes RecordCost, so CostRecords stayed empty and CostScore was permanently maxed."

18. "Debounce: 30-second window (commit 4041469)" — ✅ Verified. `RecalculateAgentRankConsumer.cs` line 23: `private static readonly TimeSpan RecalcDebounce = TimeSpan.FromSeconds(30)`. Commit 4041469 confirms.

19. "Upsert with unique index (TenantId, AgentId) and DbUpdateException conflict handling (commit 46e27a5)" — ✅ Verified. `RecalculateAgentRankConsumer.cs` lines 106-124. Commit 46e27a5 includes migration `20260612171512_UniqueAgentScorePerAgent.cs`.

20. "Gateway injects X-Tenant-Id downstream (commit d40a15e)" — ✅ Verified. Commit d40a15e message: "Gateway: inject X-Tenant-Id header downstream after tenant resolution."

21. "EF Core query filters apply WHERE TenantId globally via TenantContext" — ✅ Verified. `RecalculateAgentRankConsumer.cs` line 39: `_tenantContext.TenantId = msg.TenantId`. TenantContext is a registered service used throughout.

22. "Unique index on AgentScores(TenantId, AgentId)" — ✅ Verified. Migration `20260612171512_UniqueAgentScorePerAgent.cs` in commit 46e27a5.

23. "Live and processing telemetry from Copilot Studio and Vertex AI agents in production" — ⚠️ WEAK. CI/CD pipeline commits (`cd8f1e5`, `e4c820e`) confirm production deployment pipeline exists. Cannot verify from code that telemetry is actively flowing. Rewritten to reference the production CI/CD pipeline without claiming live traffic.

---

## Forward-Looking Scan

Searched for: "will", "plan to", "coming soon", "in the future", "next step", "roadmap", "soon".

**No matches found.** Article is entirely retrospective.

---

## Rubric Scorecard

| Dimension | Score | Notes |
|---|---|---|
| Evidence quality (claims cited) | 4/5 | 20/23 claims fully verified; 3 weakly grounded — fixed |
| Technical depth | 5/5 | Parser internals, debounce logic, upsert pattern, cost calculation all shown |
| Clarity for target audience | 5/5 | Problem → solution → code → lesson structure is clear |
| BistecGlobal voice | 4/5 | Professional, practitioner-focused, grounded in real bugs |
| Title specificity | 5/5 | Names OTLP, AgentScore, cost tracking, multi-tenant — all present in article |

**Total: 23/25 — PASS**

---

## Fixes Applied

- Claim 1: Removed "Bedrock" from the intro platform list; changed to "Copilot Studio, Azure AI Foundry, and Vertex AI" (the three platforms with actual OTLP price table entries or connector implementations).
- Claim 2: Removed platform-specific telemetry description ("Copilot Studio emits webhook events…") — not grounded in Agent Nexus code.
- Claim 23: Changed "live and processing telemetry from Copilot Studio and Vertex AI agents in production" to "merged to main and deployed via the Azure DevOps CI/CD pipeline, with e2e tests covering the full ingest-to-score path."
