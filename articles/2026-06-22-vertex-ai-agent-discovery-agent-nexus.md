---
title: "Adding a Third Cloud to Our AI Agent Platform: What Vertex AI Taught Us About Our Own Abstraction"
project: agent-nexus
tags: [AI, Google Cloud, Vertex AI, Multi-Cloud, Observability, .NET, Enterprise]
status: draft
date: 2026-06-22
---

# Adding a Third Cloud to Our AI Agent Platform: What Vertex AI Taught Us About Our Own Abstraction

We assumed adding a third cloud would be straightforward. Agent Nexus already governed agents on Microsoft Copilot Studio and Amazon Bedrock behind a common `IPlatformConnector` interface. Adding Vertex AI was, in theory, just another implementation. In practice, three cracks appeared in our design that we hadn't noticed with just two platforms. Here's what we found — and what the third integration taught us about building multi-cloud agent governance.

## The Setup: One Interface, Many Clouds

Agent Nexus is a multi-tenant control plane for governing AI agents across clouds. The connector layer is simple by design: every platform must implement `IPlatformConnector`, which includes a `ListAgentsAsync` method that returns discovered agents for a given connection. A background scheduler fires `SyncAgentsRequested` messages on a configurable interval, and a consumer processes each one — discovering agents, persisting them, and publishing the results over Kafka.

Adding Vertex AI meant writing one class: `VertexAiConnector`. Everything else — scheduling, sync state, failure tracking, observability ingest — already existed. But three things didn't work as expected, and each one pointed to a gap in the abstraction.

## Crack 1: Whose Credentials Are These?

Our first Copilot Studio connector used a single Azure service principal stored as a platform-level secret. When we added Bedrock, we used AWS IAM roles, similarly scoped to the host environment. When we got to Vertex AI, the same pattern would have meant storing a GCP service account key that all tenants shared — which is obviously wrong for a multi-tenant platform.

The Vertex AI connector had to use *each tenant's own GCP credentials*, not host ADC. The implementation resolves them from Infisical using the connection's `SecretRef`:

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

This forced us to revisit our earlier connectors. Copilot Studio was fine — it already used per-tenant credentials via Azure. Bedrock was using ambient IAM, which worked only because we happened to be single-tenant there. Multi-cloud governance means *each cloud, each tenant, separate credentials, always*.

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

This distinction matters in production: retry logic that can't tell a transient network error from a permanent auth failure will either flood the API or silently stall. The three-status model (`connected` / `error` / `needs_reauth`) is now something we'll apply to all future platform connectors.

## Crack 3: Cost Numbers Were Wrong for Gemini

When Vertex AI telemetry started flowing into the observability service, cost figures were wrong for every Gemini event. The `ComputeCost` function looked up pricing by model name, but was being called with `record.Platform` ("vertex_ai") instead of `record.Model` ("gemini-2.5-flash"). Every event hit the fallback price and was silently overcharged.

```csharp
// Before — every Vertex AI event hit the fallback price
CostUsd = (decimal)ComputeCost(record.Platform, input, output),

// After — use the actual model name for the correct price row
CostUsd = (decimal)ComputeCost(record.Model, input, output),
```

This bug existed from day one but went undetected because our first two platforms (Copilot Studio, Bedrock) used model names that happened to match nothing in the price table either — the fallback GPT-4o price was used for everything. Adding Vertex AI with explicit Gemini model names exposed it.

The cost summary endpoint also needed fixing. It was reading from a `CostRecords` ledger that was populated manually (and often not at all). We replaced it with a live aggregation over `OperationalEvents.CostUsd` grouped by platform and model — actual spend derived from actual token usage, not a parallel ledger that drifted from reality.

## What the Third Cloud Actually Validates

Adding a second cloud to an abstraction tells you if the interface holds. Adding a third tells you if the *assumptions behind* the interface hold.

With Copilot Studio and Bedrock, we had implicitly assumed:
- Credentials could be scoped at the platform level
- Failed connections didn't need periodic retry
- Model-based pricing was stable enough that an early bug didn't matter

None of those assumptions survived Vertex AI. The credentials assumption was wrong by design (multi-tenancy requires per-tenant credential isolation). The retry assumption was wrong by omission (the scheduler query simply hadn't considered error recovery). The pricing assumption was a latent bug that only surfaced when real model-named events arrived.

The Vertex AI integration is in production and discovering Vertex AI Reasoning Engine agents alongside Copilot Studio and Bedrock agents in a single normalized view. But the more durable output is the hardened abstraction: a credential model that enforces tenant isolation at the secret-resolution layer, a connection lifecycle that handles both transient failures and permanent auth expiry, and a cost engine that reads from actual telemetry rather than a secondary ledger.

A third integration stress-tests your architecture in ways the first two never could. If you're building multi-cloud infrastructure, plan for it.

---

*Agent Nexus is BistecGlobal's multi-tenant control plane for AI agent governance across Copilot Studio, Bedrock, and Vertex AI.*
