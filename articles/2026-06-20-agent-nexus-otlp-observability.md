---
title: "AI Agent Observability with OTLP: Real-Time AgentScore, Cost Tracking, and Multi-Tenant Ingest"
project: agent-nexus
tags: [AI, Observability, OpenTelemetry, Multi-Tenant, DevOps, Azure]
status: audited
date: 2026-06-20
---

# AI Agent Observability with OTLP: Real-Time AgentScore, Cost Tracking, and Multi-Tenant Ingest

When you run AI agents across Azure AI Foundry, Copilot Studio, and Google Vertex AI simultaneously, you quickly discover that each platform has its own telemetry format, its own token-reporting shape, and its own idea of what "observable" means. None of them tell you what you really need to know: which agents are performing well, what they cost per month, and whether any are leaking data across tenant boundaries.

This is the problem Agent Nexus set out to solve with a dedicated observability service — one that ingests OpenTelemetry Protocol (OTLP) log records from every platform, computes a real-time score for each agent, tracks cost at the point of ingest, and enforces strict multi-tenant isolation. Here is how we built it.

## The Problem: Platform Telemetry Without a Common Denominator

Agent Nexus manages AI agents across Copilot Studio, Azure AI Foundry, and Vertex AI. Each platform is instrumented with an OpenTelemetry collector that transforms platform-specific events into OTLP log records using the `gen_ai.*` semantic conventions — `gen_ai.agent.id`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, `gen_ai.system`, `gen_ai.request.model`, and so on.

This gave us a common wire format, but three problems remained:

1. **Cost was invisible at runtime.** Platforms report token counts, not dollars. You had to correlate with billing exports hours or days later.
2. **AgentRank accumulated garbage.** Every telemetry event triggered a full recalculation over 24 hours of metric history, and the result was appended as a new row — the table grew unboundedly and scores were wrong because the cost dimension sourced from an empty ledger.
3. **Multi-tenant ingest was broken.** Tenant identity was being read from an HTTP header (`X-Tenant-Id`) rather than from the OTLP record attributes themselves, so a tenant-tier worker could accidentally stamp all of its events under the dev tenant.

Fixing all three required coordinated changes across the ingest path, the scoring engine, and the database schema.

## OTLP Ingest: Parsing, Costing, and Routing in a Single Pass

The ingest endpoint lives at `POST /otlp/v1/logs` in `Nexus.Observability` (`src/Nexus.Observability/Endpoints/OtlpIngestEndpoints.cs`). It is internal-only — not reachable through the YARP gateway — and gated by a shared secret (`X-Nexus-Ingest-Key`) checked fail-closed: a missing key in production returns 503, not an open door.

The parser (`src/Nexus.Observability/OtlpParser.cs`) walks the OTLP JSON envelope (`resourceLogs → scopeLogs → logRecords`) and extracts a typed record for each log entry. One detail that tripped us up early: the OTLP spec encodes `int64` values as JSON strings, but the Python SDK emits them as JSON numbers. The parser handles both:

```csharp
else if (valueEl.TryGetProperty("intValue", out var iv))
    val = iv.ValueKind == JsonValueKind.String ? iv.GetString() : iv.GetRawText();
```

Tenant identity is resolved entirely from each record's `tenant.id` attribute. The parser validates it as a well-formed GUID and falls back to the dev tenant when absent or malformed — but it never reads `X-Tenant-Id` from the request. This enforces a key invariant: no in-cluster caller can attribute telemetry to an arbitrary tenant by spoofing a header.

Cost is computed at ingest time, not asynchronously. The endpoint maintains a model price table covering GPT-4o, GPT-4o-mini, GPT-4 Turbo, and eight Gemini variants (2.5 Pro through 1.5 Flash). Each operational event gets a `CostUsd` field computed from token counts before the record is persisted:

```csharp
CostUsd = (decimal)ComputeCost(record.Model, input, output),
```

where `ComputeCost` applies the per-1k-token rates. Unknown models fall back to GPT-4o pricing. This means every `OperationalEvent` row carries its cost at write time — no join against an external billing API needed.

Governance events (guardrail violations, prompt injection detections, content filter hits) go through a two-level dedup before persisting: an in-memory `HashSet` collapses duplicates within a single batch (before `SaveChangesAsync` runs), and an `AnyAsync` DB check drops events with the same `(TenantId, SessionId, PolicyName)` seen within a 60-second window.

## AgentScore: A Four-Dimension Composite Score

After each operational or governance event, the ingest path publishes a `RecalculateAgentRank` message to a MassTransit in-memory bus. The consumer (`src/Nexus.Observability/Consumers/RecalculateAgentRankConsumer.cs`) computes a composite score from four dimensions over a rolling 24-hour window:

| Dimension | Weight | Source |
|-----------|--------|--------|
| Reliability | 35% | success count / (success + failure) |
| Latency | 25% | avg latency vs 2000ms target |
| Cost | 20% | monthly spend vs $100 budget |
| Throughput | 20% | actual RPS vs 1 RPS target |

The `AgentRankEngine` (`src/Nexus.Observability/Engines/AgentRankEngine.cs`) maps each dimension to a 0–100 score and combines them with the weights above:

```csharp
private const decimal ReliabilityWeight = 0.35m;
private const decimal LatencyWeight     = 0.25m;
private const decimal CostWeight        = 0.20m;
private const decimal ThroughputWeight  = 0.20m;
```

Two bugs in the original implementation had to be fixed before scores were meaningful.

**Bug 1: CostScore was always 100.** The consumer summed `CostRecords` for the cost dimension, but the ingest path computes cost onto `OperationalEvents.CostUsd` and never writes to the `CostRecords` ledger. The table stayed empty, so `totalCost` was always zero, and `CostEngine.CalculateCostEfficiencyScore(0, threshold)` returns 100. The fix: source cost directly from `OperationalEvents` aggregated by month — matching the cost summary endpoint.

**Bug 2: Unbounded table growth and duplicate rows under concurrency.** Every telemetry event triggered a recalculation that appended a new `AgentScores` row, so the table grew one row per event. Concurrent recalculations for the same agent could each find `existing == null` and both insert, breaking the one-row-per-agent invariant. Two fixes landed together:

- **Debounce**: skip the full aggregate scan when the agent's score was updated within the last 30 seconds. Bursts collapse to at most one scan per agent per 30-second window.
- **Upsert with conflict handling**: a unique index on `(TenantId, AgentId)` enforces one row per agent at the database level. If a concurrent insert wins the race, the loser catches `DbUpdateException`, reloads the winner's row, and applies its computation as an update.

```csharp
catch (DbUpdateException)
{
    _db.Entry(score).State = EntityState.Detached;
    score = await _db.AgentScores
        .FirstAsync(s => s.AgentId == msg.AgentId, cancellationToken);
    ApplyScore(score, computed, agentName);
    await _db.SaveChangesAsync(cancellationToken);
}
```

## Multi-Tenant Isolation End-to-End

The tenant isolation story runs across three layers:

**Ingest layer**: `OtlpParser` extracts `tenant.id` from the OTLP record attributes and falls back to the dev tenant — never from an HTTP header. This means the collector is responsible for stamping the correct tenant, and the ingest endpoint cannot be fooled by a header injection.

**Gateway layer**: The Nexus Gateway resolves the authenticated tenant from the JWT and injects `X-Tenant-Id` downstream to backend services. Backend services read this header rather than re-validating the token, which keeps auth logic in one place.

**Database layer**: Every entity in `Nexus.Observability` carries a `TenantId` column, and EF Core query filters apply `WHERE TenantId = @tenantId` globally via `TenantContext`. The unique index on `AgentScores(TenantId, AgentId)` ensures one score row per agent per tenant.

## What This Enables

With this pipeline in place, the Agent Nexus dashboard can display a live AgentScore for every connected agent, broken down by reliability, latency, cost efficiency, and throughput — updated within 30 seconds of any telemetry event, regardless of whether the agent runs on Copilot Studio, Vertex, or Foundry.

Cost is no longer a billing-cycle concern. Because `CostUsd` is computed at ingest against a model price table, the cost summary endpoint can answer "what did this agent cost this month?" from a simple aggregate query over `OperationalEvents` — no external billing API, no overnight ETL job.

The multi-tenant design means a single deployment serves all tenants without data leakage: telemetry flows in stamped with tenant identity at the source, persists under that identity, and query filters prevent any cross-tenant reads.

## Lessons Learned

**Source cost at the point of ingest, not from a separate ledger.** Maintaining two cost representations (OperationalEvents.CostUsd and CostRecords) and keeping them in sync is harder than it looks. If your scoring engine reads from the ledger but your ingest path writes to the events table, you get silently wrong scores. Pick one source of truth and use it everywhere.

**Debounce before you optimize the query.** The aggregate scan over 24 hours of metric history is not inherently expensive for a single agent. What made it expensive was running one scan per telemetry event under burst conditions. A 30-second debounce window reduced that to one scan per agent per burst — no query restructuring required.

**Put tenant identity in the payload, not the header.** Headers are easy to spoof inside a cluster. Stamping `tenant.id` onto each OTLP record at the collector level means tenant identity travels with the data through every processing hop, and the ingest endpoint never needs to trust an ambient header.

The observability pipeline in Agent Nexus is merged to main and deployed via the Azure DevOps CI/CD pipeline, with Playwright e2e tests covering the full ingest-to-score path. The source lives at [github.com/chan4lk/agent-nexus](https://github.com/chan4lk/agent-nexus).
