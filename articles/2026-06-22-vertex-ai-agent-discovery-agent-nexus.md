---
title: "Adding a Second Connector to Our AI Agent Platform: What Vertex AI Taught Us About Our Own Abstraction"
project: agent-nexus
tags: [AI, Google Cloud, Vertex AI, Multi-Cloud, Observability, .NET, Enterprise]
status: audited
date: 2026-06-22
---

# Adding a Second Connector to Our AI Agent Platform: What Vertex AI Taught Us About Our Own Abstraction

We assumed adding Vertex AI would be straightforward. Agent Nexus already governed agents on Microsoft Copilot Studio behind a common `IPlatformConnector` interface. Adding Google's platform was, in theory, just another implementation. In practice, three cracks appeared in our design that the first connector never surfaced. Here's what we found — and what the second integration taught us about building multi-cloud agent governance.

## The Setup: One Interface, Many Clouds

Agent Nexus is a multi-tenant control plane for governing AI agents across clouds. The connector layer is simple by design: every platform must implement `IPlatformConnector`, which includes a `ListAgentsAsync` method that returns discovered agents for a given connection. A background scheduler fires `SyncAgentsRequested` messages on a configurable interval, and a consumer processes each one — discovering agents, persisting them, and publishing the results over Kafka.

Adding Vertex AI meant writing one class: `VertexAiConnector`. Everything else — scheduling, sync state, failure tracking, observability ingest — already existed. But three things didn't work as expected, and each one pointed to a gap in the abstraction.

## Crack 1: Whose Credentials Are These?

The initial `VertexAiConnector` took the quick path: it called `GoogleCredential.GetApplicationDefaultAsync()` — the host Application Default Credentials. That worked in local development, where the developer's ADC had access to a test GCP project. It broke in multi-tenant production: every tenant's discovery ran as the host identity, surfacing the dev project's agents under whichever tenant connection happened to be syncing.

Copilot Studio had handled this correctly from the start, resolving per-tenant Microsoft credentials from Infisical via each connection's `SecretRef`. The Vertex AI connector needed the same treatment: *each tenant's own GCP credentials*, loaded at sync time, never shared. The implementation:

```csharp
private async Task<string> ResolveCredentialsJsonAsync(string secretRef, CancellationToken ct)
{
    var colonIdx = secretRef.IndexOf(':');
    var environment = secretRef[..colonIdx];
    var secretPath  = secretRef[(colonIdx + 1)..];

    var secrets = await _infisical.GetSecretsAsync(environment, secretPath, ct);
    if (!secrets.TryGetValue("GCP_CREDENTIALS_JSON", out var credsJson))
        throw new InvalidOperationException("GCP_CREDENTIALS_JSON missing at SecretRef path");

    return credsJson;
}
```

The credential JSON — a GCP service account key or authorized user file — is loaded per-connection at sync time. `Google.Apis.Auth` exchanges it for a short-lived OAuth token scoped to `https://www.googleapis.com/auth/cloud-platform`, then we hit the Vertex AI `reasoningEngines` API. No shared ADC, no cross-tenant bleed.

The lesson generalised: any connector that relies on host-level credentials is a latent multi-tenancy bug. Multi-cloud agent governance means *each cloud, each tenant, separate credentials, always* — and that invariant must be enforced at the connector layer, not assumed.

## Crack 2: Self-Healing or Just Masking Failures?

Vertex AI's `reasoningEngines` API is occasionally flaky on cold discovery runs. Our `SyncAgentsConsumer` already tracked `ConsecutiveFailures` and set a connection's status to `"error"` after two strikes. What it didn't do was *retry error connections*.

The scheduler was only dispatching syncs for connections in `"connected"` status. A Vertex AI connection that failed twice would stay silent forever.

The fix was a one-line change to the scheduler query:

```csharp
var connections = await db.PlatformConnections
    .IgnoreQueryFilters()
    .Where(c => c.Status == "connected" || c.Status == "error")
    .ToListAsync(ct);
```

But there was a subtlety: token expiry is not a transient failure. If a GCP credential expires, re-auth is needed before retrying — endlessly retrying a bad token wastes API quota and logs. We added a separate `"needs_reauth"` status for that case, which the scheduler explicitly skips:

```csharp
if (errorCode.StartsWith("token_expired"))
{
    connection.Status = "needs_reauth";
    // Not a transient failure — don't count toward ConsecutiveFailures
}
else
{
    connection.ConsecutiveFailures++;
    if (connection.ConsecutiveFailures >= 2)
        connection.Status = "error";
}
```

This distinction matters in production: retry logic that can't tell a transient network error from a permanent auth failure will either flood the API or silently stall. The three-status model (`connected` / `error` / `needs_reauth`) is a pattern worth applying to every platform connector that authenticates with external credentials.

## Crack 3: Cost Numbers Were Wrong for Gemini

When Vertex AI telemetry started flowing into the observability service, cost figures were wrong for every Gemini event. The `ComputeCost` function looked up pricing by model name, but was being called with `record.Platform` ("vertex_ai") instead of `record.Model` ("gemini-2.5-flash"). Every event hit the fallback price and was silently overcharged.

```csharp
// Before — every Vertex AI event hit the fallback price
CostUsd = (decimal)ComputeCost(record.Platform, input, output),

// After — use the actual model name for the correct price row
CostUsd = (decimal)ComputeCost(record.Model, input, output),
```

This bug existed from day one but went undetected because Copilot Studio agents typically ran on GPT-4o — which happened to be the fallback price too. The erroneous lookup key ("copilot_studio") fell back to GPT-4o pricing, accidentally correct. Vertex AI events made the error undeniable: Gemini 2.0 Flash is priced at $0.0001/$0.0004 per 1k tokens versus GPT-4o's $0.005/$0.015 — a 50× difference that showed up immediately in cost dashboards.

The cost summary endpoint also needed fixing. It was reading from a `CostRecords` ledger that was populated manually (and often not at all). We replaced it with a live aggregation over `OperationalEvents.CostUsd` grouped by platform and model — actual spend derived from actual token usage, not a parallel ledger that drifted from reality.

## What a Second Connector Actually Validates

Adding a first platform to an abstraction tells you if the interface compiles. Adding a second tells you if the *assumptions behind* the interface hold.

With only Copilot Studio, we had implicitly assumed:
- Host-level credentials were an acceptable shortcut for new connectors
- Failed connections didn't need periodic retry — they'd surface as user complaints
- Model-based pricing was consistent enough that an early implementation detail didn't matter

None of those assumptions survived Vertex AI. The credentials assumption was a security flaw (host ADC exposed one tenant's agents to another's discovery). The retry assumption was wrong by omission (the scheduler query simply hadn't considered error recovery). The pricing assumption was a latent bug that only surfaced when Gemini model names diverged significantly in cost from the GPT-4o fallback.

The Vertex AI integration is in production, discovering Vertex AI Reasoning Engine agents alongside Copilot Studio agents in a single normalized view. But the more durable output is the hardened abstraction: a credential model that enforces tenant isolation at the secret-resolution layer, a connection lifecycle that handles both transient failures and permanent auth expiry, and a cost engine that reads from actual telemetry rather than a secondary ledger.

A second integration stress-tests your architecture in ways the first never could. If you're building multi-cloud infrastructure, plan for it.

---

*Agent Nexus is BistecGlobal's multi-tenant control plane for AI agent governance across Copilot Studio and Vertex AI.*
