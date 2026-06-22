# Adding a Third Cloud: How We Extended Agent Nexus to Govern Vertex AI Agents

*How a narrow connector interface, tenant-isolated GCP credentials, and a one-line cost bug fix brought Google Vertex AI under the same governance plane as Azure and AWS.*

---

You can't govern what you can't see. That was the problem staring us down when a customer asked why their Google Vertex AI agents weren't showing up on the Agent Nexus dashboard alongside their Azure Copilot Studio deployments.

We had built Agent Nexus to give engineering teams a single control plane for AI agents across cloud platforms — score them, track their cost, enforce governance policies. We had Copilot Studio connected and agents syncing. But Vertex AI, Google's production runtime for agents built with the Agent Development Kit, was a gap. Customers building on Google Cloud couldn't see their agents' latency, couldn't track token spend by Gemini model, and couldn't get reliability scores alongside their Azure counterparts.

Closing that gap required three things: discovery via the Vertex AI Reasoning Engines API, cost tracking with real Gemini pricing, and — most importantly — getting credentials right.

---

## The Multi-Cloud Connector Model

Agent Nexus manages platform connections through a thin abstraction: every cloud platform implements `IPlatformConnector`, and a factory routes sync requests to the right implementation by platform name.

```csharp
public sealed class PlatformConnectorFactory : IPlatformConnectorFactory
{
    private readonly Dictionary<string, IPlatformConnector> _connectors;

    public PlatformConnectorFactory(CopilotStudioConnector copilotStudio, VertexAiConnector vertexAi)
    {
        _connectors = new Dictionary<string, IPlatformConnector>(StringComparer.OrdinalIgnoreCase)
        {
            ["copilot_studio"] = copilotStudio,
            ["vertex_ai"]      = vertexAi,
        };
    }
}
```

Adding Vertex AI to this dictionary was the easy part. Registering `VertexAiConnector` and wiring it correctly was where the interesting decisions lived.

---

## Getting Credentials Right: Tenant ADC, Not Host ADC

The first design question was how to authenticate to GCP. The obvious path — `GoogleCredential.GetApplicationDefaultAsync()` — would have authenticated using Application Default Credentials from the host environment. That might work in a demo. In production it's a security hole: every tenant's Vertex discovery would run as the developer identity and could surface the developer GCP project's agents under a tenant's connection.

Agent Nexus already uses a two-tier credential model for all platforms: developer-tier secrets for platform infrastructure, tenant-tier secrets for each customer's cloud resources. Vertex credentials needed to follow the same pattern. Each `PlatformConnection` carries a `SecretRef` pointing to a path in Infisical. For Vertex, that path resolves to `GCP_CREDENTIALS_JSON` — a service account key scoped to that tenant's GCP project.

```csharp
private async Task<string> ResolveCredentialsJsonAsync(string secretRef, CancellationToken ct)
{
    var colonIdx = secretRef.IndexOf(':');
    var environment = secretRef[..colonIdx];
    var secretPath  = secretRef[(colonIdx + 1)..];

    var secrets = await _infisical.GetSecretsAsync(environment, secretPath, ct);
    if (!secrets.TryGetValue("GCP_CREDENTIALS_JSON", out var credsJson) || string.IsNullOrWhiteSpace(credsJson))
        throw new InvalidOperationException("GCP_CREDENTIALS_JSON missing at SecretRef path");

    return credsJson;
}
```

From there, the connector parses the JSON, reads the credential `type` field, and builds a scoped credential using `CredentialFactory.FromJson` — the non-deprecated replacement for `GoogleCredential.FromJson`. The credential is scoped to `https://www.googleapis.com/auth/cloud-platform` before requesting an access token.

This means a tenant connecting their GCP project uses only their own service account — not the host identity, and not another tenant's credentials.

---

## Discovering Reasoning Engines

Agents deployed to the Google Agent Platform appear as Reasoning Engines under the Vertex AI API. The `EnvironmentUrl` on the connection must follow the format `projects/{projectId}/locations/{location}`, which the connector parses before making any API calls.

Discovery hits the `reasoningEngines.list` endpoint for the tenant's project and location, iterates paginated results, and normalises each engine into a `DiscoveredAgentInfo`:

```csharp
var url = $"https://{location}-aiplatform.googleapis.com/v1/projects/{projectId}/locations/{location}/reasoningEngines";

do
{
    using var request = new HttpRequestMessage(HttpMethod.Get, url);
    request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", token);

    var response = await http.SendAsync(request, ct);
    response.EnsureSuccessStatusCode();

    using var doc = await JsonDocument.ParseAsync(
        await response.Content.ReadAsStreamAsync(ct), cancellationToken: ct);

    if (doc.RootElement.TryGetProperty("reasoningEngines", out var engines))
    {
        foreach (var engine in engines.EnumerateArray())
        {
            var name        = engine.TryGetProperty("name",        out var n) ? n.GetString() ?? string.Empty : string.Empty;
            var displayName = engine.TryGetProperty("displayName", out var d) ? d.GetString() ?? name         : name;
            var externalId  = name.Contains('/') ? name.Split('/').Last() : name;

            agents.Add(new DiscoveredAgentInfo(externalId, displayName, null, "active", updateTime));
        }
    }

    url = doc.RootElement.TryGetProperty("nextPageToken", out var token2) && token2.GetString() is { } t && !string.IsNullOrEmpty(t)
        ? $"...?pageToken={Uri.EscapeDataString(t)}"
        : string.Empty;

} while (!string.IsNullOrEmpty(url));
```

The `SyncAgentsConsumer` picks up `SyncAgentsRequested` events from the message bus and calls this on a schedule via `DiscoverySyncScheduler`, the same background service that already drives Copilot Studio syncs. Vertex connections are treated identically — the platform name `vertex_ai` is the only dispatch key.

---

## Getting Cost Right: Model vs. Platform

Once agents are discovered, OTLP telemetry from their runtime flows into the observability pipeline. Here we hit a subtle bug that had been silent across all platforms: `ComputeCost` was computing cost using `record.Platform` as the model lookup key instead of `record.Model`. The Prices dictionary is keyed on model identifiers like `gpt-4o` and `gemini-2.5-flash` — not platform names like `copilot_studio` or `vertex_ai`. Without a match, every event fell back to the same hardcoded default price, making the cost data uniformly wrong regardless of which model actually ran.

The fix was a one-line change to use `record.Model`, combined with adding the Gemini pricing rows to the lookup table:

```csharp
// USD per 1k tokens — Vertex AI (Gemini) models
["gemini-2.5-pro"]       = (0.00125, 0.01),
["gemini-2.5-flash"]     = (0.0003,  0.0025),
["gemini-2.0-flash"]     = (0.0001,  0.0004),
["gemini-1.5-pro"]       = (0.00125, 0.005),
["gemini-1.5-flash"]     = (0.000075, 0.0003),
```

The `/api/costs/summary` endpoint was also updated to derive the cost summary from actual `OperationalEvents` token usage — grouping by platform and by model — rather than a separate cost ledger. This means the Optimize tab in the dashboard now reflects real spend for Vertex-originated events, broken down by the Gemini model that generated each token.

---

## What We Learned

**Pluggable connectors keep integration points minimal.** Because `IPlatformConnector` is narrow — just `ListAgentsAsync`, `GetStatusAsync`, `ReadConfigAsync`, and lifecycle stubs — the connector itself is the only place that touches GCP-specific logic. The factory registration, scheduler, and message consumer all treat Vertex AI identically to any other platform. That said, adding a new cloud end-to-end also required database migrations to carry the agent name through the observability pipeline, OTLP ingest changes to recognise Vertex event types, and frontend hook updates — the connector interface kept those integration points predictable, not zero.

**Host ADC is a trap in multi-tenant systems.** Relying on ambient GCP credentials would have silently mixed tenant data in local dev and caused opaque failures in production when the host identity lacked access to a tenant's project. Resolving credentials from the tenant's own Infisical secret at discovery time is the only safe model.

**Price tables need model-level keys, not platform-level keys.** When you're ingesting telemetry from multiple cloud platforms running multiple foundation models, the cost computation has to key on the model identifier that arrives in the telemetry, not on the platform that sent it. A platform might run several models with very different prices, and a bug at this level produces silently wrong cost data across every dashboard and report.

Adding Vertex AI to Agent Nexus brings the platform to three clouds. The connector model is now proven across two fundamentally different authentication systems — Microsoft MSAL for Copilot Studio, GCP service account JSON for Vertex AI — and the observability pipeline handles events from all three without platform-specific branching.

---

*Originally published at bistecglobal.com*

**BistecGlobal** builds AI-native enterprise software. Follow us for engineering insights.
