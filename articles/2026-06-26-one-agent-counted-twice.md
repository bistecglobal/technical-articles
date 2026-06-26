---
title: "One Agent Counted Twice, Every Total Counted Too Little: Reconciling Cross-Cloud Telemetry"
project: agent-nexus
tags: [AI, Cloud, Observability, Multi-Tenant, DotNet, OpenTelemetry]
status: draft
date: 2026-06-26
---

A customer opened the Agent Nexus dashboard and saw the same Vertex agent listed twice. One copy showed traffic. The other showed nothing — no requests, no tokens, no cost. Same name, same platform, two rows. Then they noticed something worse: the totals at the top of the page were quietly wrong too. Request counts, token usage, and spend were all under-reported.

Neither bug was a crash. Nothing threw. The numbers were just *off* — the most dangerous kind of failure in an observability product, because the whole point of the product is that you trust the numbers. This is the story of three fixes that landed together to make the totals add up again.

## How one agent becomes two

Agent Nexus discovers agents running across Copilot Studio, Bedrock, and Vertex by reading them through *connections* — credentialed links to a tenant's cloud. The wrinkle: the same external engine can be reachable through more than one connection. A platform-tier connection and a tenant-tier connection can both point at the same GCP project, so discovery finds the same agent twice and writes two `DiscoveredAgents` rows.

That was harmless until someone promoted them. Promotion turns a discovered agent into a first-class registry `Agent`. The old code minted a fresh `Agent` with a new GUID for *every* discovered row it was handed:

```csharp
var agent = new Agent { Name = discovered.DisplayName, Platform = discovered.Platform, ... };
db.Agents.Add(agent);
discovered.PromotedAgentId = agent.Id;
await db.SaveChangesAsync();
```

Two rows, two promotes, two registry agents — for one real agent. And because telemetry binds to whichever agent identity the traces resolve to, only one of the twins carried any data. The other was a permanent ghost.

## Dedupe at the identity, not the row

The fix reframes identity. An agent is not "a discovered row" — it's the tuple `(TenantId, Platform, ExternalAgentId)`. Promotion now checks whether a *sibling* discovered row for the same tuple has already been promoted, and if so, reuses that agent instead of minting a new one:

```csharp
private static Task<Guid?> SiblingPromotedAgentAsync(
    RegistryDbContext db, DiscoveredAgent discovered, CancellationToken ct) =>
    db.DiscoveredAgents
        .Where(d => d.Id != discovered.Id
                 && d.TenantId == discovered.TenantId
                 && d.Platform == discovered.Platform
                 && d.ExternalAgentId == discovered.ExternalAgentId
                 && d.PromotedAgentId != null)
        .Select(d => d.PromotedAgentId)
        .FirstOrDefaultAsync(ct);
```

If a sibling was already promoted, the new row just points at the same agent and returns it. No duplicate, no orphan.

A read-then-act check has an obvious hole: two promotes racing on the two sibling rows can both read "no sibling yet" and both insert. So the dedupe isn't trusted to be the only line of defence. A new partial unique index makes the database the referee:

```csharp
migrationBuilder.CreateIndex(
    name: "IX_Agents_TenantId_Platform_ExternalAgentId",
    schema: "registry", table: "Agents",
    columns: new[] { "TenantId", "Platform", "ExternalAgentId" },
    unique: true,
    filter: "\"ExternalAgentId\" IS NOT NULL");
```

The filter matters — agents created by hand (not discovered) have a null `ExternalAgentId` and are exempt from the constraint, so the rule only governs discovered identities.

With the index in place, the loser of a race gets a `DbUpdateException` on insert. Rather than surfacing that as a 500, the endpoint treats it as information — *someone else just won, go find their agent*:

```csharp
catch (DbUpdateException)
{
    db.Entry(agent).State = EntityState.Detached;            // discard our duplicate
    if (await SiblingPromotedAgentAsync(db, discovered, ct) is not Guid winnerId) throw;
    var winner = await db.Agents.FindAsync([winnerId], ct);
    if (winner is null) return Results.NotFound();
    discovered.PromotedAgentId = winnerId;                    // point at the winner
    await db.SaveChangesAsync(ct);
    return Results.Ok(ToAgentResponse(winner));
}
```

A `WebApplicationFactory` integration test (`PromoteRaceRecoveryTests`) drives the exact race: a concurrent sibling wins, this promote loses the index, recovers, and the assertions confirm the caller gets `200 OK` with the winner's id, exactly one agent exists for the tuple, and the losing row is re-pointed at the winner. Application logic does the fast path; the unique index is the backstop that makes the fast path safe to be approximate.

## The totals that capped themselves

The second bug lived in the observability service. The operational-events endpoint computed its headline summary — total requests, total tokens, total cost, p95 latency — *from the rows it returned*. And it only ever returned the latest 200:

```csharp
var events = await db.ReadInTenantScopeAsync(
    () => query.OrderByDescending(e => e.EventTimestamp).Take(200).ToListAsync(ct), ct);
var totalCost = events.Sum(e => e.CostUsd);   // sums 200 rows, calls it "total"
```

For any tenant with more than 200 events, every total was capped at 200 events' worth of activity. The summary and the list were the same query, so the page that was supposed to *measure* usage was structurally incapable of measuring past its own display limit.

The fix separates the two concerns. The summary now aggregates over the **full filtered set** server-side, and the events list becomes an explicitly bounded display page:

```csharp
var (totalRequests, totalCost, totalToks, p95, events) = await db.ReadInTenantScopeAsync(
    async () =>
    {
        var count = await query.CountAsync(ct);
        var cost  = await query.SumAsync(e => e.CostUsd, ct);
        var toks  = await query.SumAsync(e => e.TotalTokens, ct);
        // ... p95 ...
        var list  = await query.OrderByDescending(e => e.EventTimestamp).Take(limit).ToListAsync(ct);
        return (count, cost, toks, p95Latency, list);
    }, ct);
```

`COUNT` and `SUM` run in the database, inside the same transaction-local tenant scope that row-level security uses, so the aggregates never leak across tenants. The list keeps an optional `?limit=N` (default 1000) — the page stays bounded on a hot telemetry table, but the totals no longer inherit the page size.

p95 latency got the same treatment. The old code pulled the entire `LatencyMs` column into memory and sorted it — unbounded work on a high-volume table. Now it's a `percentile_cont(0.95)` computed in Postgres. Because the test suite runs on the in-memory provider (which isn't relational), there's a LINQ fallback using a linear-interpolation percentile that matches Postgres's `percentile_cont` for identical data — so tests and production return the same p95.

## The tokens that were never read

The third gap explained those zero-traffic Vertex agents. The collector's Vertex transform never read token usage at all. Vertex's ADK inference logs carry per-call usage in the log entry's top-level labels — `gen_ai.usage.input_tokens` and `output_tokens` — and the pipeline simply wasn't extracting them. No tokens meant no cost, because cost is derived from tokens via a price table.

The collector config now lifts those labels into the standard `gen_ai.usage.*` attributes, and maps the engine id to its model (`gemini-2.0-flash`) so the price table can compute spend. The subtle part was a governance interaction: the inference logs also got swept into a denial-phrase classifier, and any log routed to `GovernanceEvents` had its token usage dropped on the floor. The classifier now skips token-bearing inference logs entirely — their body is raw conversation, so a phrase like "denied" there is *content*, not a governance decision. Denials still surface, but only from the audit and stderr logs where they actually mean something.

## What it adds up to

Three fixes, one theme: in a multi-tenant, multi-cloud control plane, the hard bugs aren't exceptions — they're arithmetic. An agent counted twice. Totals counted too little. Tokens counted as zero. None of them threw; all of them eroded trust in the dashboard.

A few things worth carrying forward:

- **Model identity on the thing it identifies, not on the row that found it.** The duplicate-agent bug existed because identity was implicitly "a discovered row." Naming the real key — `(TenantId, Platform, ExternalAgentId)` — and enforcing it with a partial unique index made the duplicate structurally impossible.
- **Let the database referee races, and treat the constraint violation as data.** The dedupe check is the fast path; the unique index is the truth. Catching `DbUpdateException` and recovering to the winner turns a 500 into a correct 200.
- **Never compute a "total" from a display page.** Aggregations belong in the database, over the full filtered set, inside the same tenant scope as the rest of the read. The list is a view; the summary is a measurement; they should not share a `Take(200)`.
- **A telemetry field that's never extracted reads as a real zero.** The missing Vertex tokens looked exactly like an idle agent. When a metric is suspiciously empty, check whether anything is populating it before concluding the activity isn't there.

When your product *is* the numbers, getting them to add up is the feature.
