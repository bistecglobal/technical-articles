# Audit: vertex-ai-agent-discovery-agent-nexus

**Date:** 2026-06-22  
**Verdict: NEEDS REVISION** (pre-fix) → **PASS** after applying inline fixes  
**Score: 19/25** → **22/25** post-fix

---

## Claim Inventory

| # | Claim | Status | Evidence |
|---|-------|--------|----------|
| 1 | Agent Nexus governed agents on Copilot Studio | ✅ | `PlatformConnectorFactory` registers `copilot_studio` |
| 2 | Agent Nexus governed agents on Amazon Bedrock | ❌ | No Bedrock connector in `src/Nexus.Connectors/Platforms/`. Factory only has `copilot_studio` + `vertex_ai` |
| 3 | `IPlatformConnector` includes `ListAgentsAsync` | ✅ | `IPlatformConnector.cs`, commit `0f7f679` |
| 4 | Background scheduler fires `SyncAgentsRequested` on configurable interval | ✅ | `DiscoverySyncScheduler.cs` with `DiscoveryOptions.SyncIntervalMinutes` |
| 5 | Consumer discovers agents, persists, publishes over Kafka | ✅ | `SyncAgentsConsumer.cs` + `IKafkaSyncProducer` |
| 6 | Adding Vertex AI required one class: `VertexAiConnector` | ✅ | `VertexAiConnector.cs`, `PlatformConnectorFactory.cs` |
| 7 | "Copilot Studio used a single Azure service principal stored as a platform-level secret" | ❌ | `CopilotStudioSecretResolver.cs` resolves per-tenant credentials (COPILOT_TENANT_ID/CLIENT_ID/SECRET/REFRESH_TOKEN) from Infisical via SecretRef — not a platform-level shared credential |
| 8 | "Bedrock was using ambient IAM" | ❌ | No Bedrock connector |
| 9 | Initial Vertex AI connector used host ADC (not tenant creds) | ✅ | Commit `9ce38b3` message: "ListAgentsAsync authenticated with GoogleCredential.GetApplicationDefaultAsync() (host/dev ADC) regardless of which tenant connection was syncing" |
| 10 | `ResolveCredentialsJsonAsync` code snippet | ✅ | Matches `VertexAiConnector.cs` exactly |
| 11 | `Google.Apis.Auth` for OAuth token scoped to `cloud-platform` | ✅ | Code: `credential.UnderlyingCredential.GetAccessTokenForRequestAsync` after `CredentialFactory.FromJson(...).CreateScoped("https://www.googleapis.com/auth/cloud-platform")` |
| 12 | Hits Vertex AI `reasoningEngines` API | ✅ | `FetchReasoningEnginesAsync` in `VertexAiConnector.cs` |
| 13 | "Copilot Studio was fine — already used per-tenant credentials" | ✅ | `CopilotStudioSecretResolver.cs` uses per-tenant Infisical secrets |
| 14 | ConsecutiveFailures → error after 2 strikes | ✅ | `SyncAgentsConsumer.cs` |
| 15 | Scheduler only dispatched "connected" status before fix | ✅ | Commit `16ffdeb` note: "DiscoverySyncScheduler retries 'error' connections" implies it didn't before |
| 16 | Scheduler fix code snippet | ✅ | Matches `DiscoverySyncScheduler.cs` exactly |
| 17 | `needs_reauth` status for token expiry | ✅ | `SyncAgentsConsumer.cs` |
| 18 | `needs_reauth` code snippet | ✅ | Matches exactly |
| 19 | ComputeCost used `record.Platform` before fix | ✅ | `git show 16ffdeb~1` confirms `ComputeCost(record.Platform, input, output)` |
| 20 | ComputeCost before/after snippet | ✅ | Matches code history |
| 21 | "Copilot Studio and Bedrock model names matched nothing in price table — fallback GPT-4o used for everything" | ⚠️ | Bedrock never existed. CopilotStudio events used GPT-4o model names which were in the price table — the fallback happened to be GPT-4o which was close. The real reason it went unnoticed: CopilotStudio cost was accidentally correct |
| 22 | Cost summary moved from CostRecords to OperationalEvents aggregation | ✅ | `CostEndpoints.cs` `/summary` endpoint confirmed |
| 23 | Vertex AI in production discovering Reasoning Engine agents | ✅ | `VertexAiConnector.FetchReasoningEnginesAsync` |

---

## Forward-Looking Scan

- "The three-status model is now something **we'll apply** to all future platform connectors." — ⚠️ Forward-looking. Reframe as a design principle rather than a future commitment.

---

## Rubric Scorecard

| Dimension | Pre-fix | Post-fix | Notes |
|-----------|---------|----------|-------|
| Evidence quality (verifiable, no SHAs in prose) | 3 | 5 | Two false claims (Bedrock) drag score |
| Technical depth | 4 | 4 | Three distinct cracks with code evidence |
| Clarity for target audience | 4 | 4 | Good flow, three-section structure |
| BistecGlobal voice | 4 | 4 | Professional, practitioner-focused |
| Title specificity | 4 | 5 | Title references "third cloud" which was wrong; fix to "second connector" |
| **Total** | **19/25** | **22/25** | |

---

## Rewrite Instructions

### Remove Bedrock throughout

1. **Line 11** (intro): "Agent Nexus already governed agents on Microsoft Copilot Studio and Amazon Bedrock" → "Agent Nexus already governed agents on Microsoft Copilot Studio"
2. **Line 15** (setup): "everything else...already existed" — no change needed
3. **Line 21** (Crack 1 setup): Remove "When we added Bedrock, we used AWS IAM roles, similarly scoped to the host environment."
4. **Line 42** (Crack 1 conclusion): "Bedrock was using ambient IAM..." → remove sentence
5. **Line 97–100** (conclusion bullets): Remove "Credentials could be scoped at the platform level" (implied by Bedrock story — now moot)
6. **Line 89** (Crack 3): Rewrite false claim about Copilot Studio/Bedrock model names

### Rewrite Crack 1 narrative for Copilot Studio claim

Replace "Our first Copilot Studio connector used a single Azure service principal stored as a platform-level secret" with accurate framing:

> The initial `VertexAiConnector` took the quick path: it called `GoogleCredential.GetApplicationDefaultAsync()` — the host Application Default Credentials. That worked in local dev, where the developer's ADC had access to a test GCP project. It broke in multi-tenant production: every tenant's discovery ran as the host identity, surfacing the dev project's agents under whichever tenant connection was syncing. Copilot Studio had handled this correctly from the start, resolving per-tenant credentials from Infisical via each connection's `SecretRef`. The Vertex AI connector needed the same treatment.

### Rewrite ComputeCost bug explanation

Replace "This bug existed from day one but went undetected because our first two platforms (Copilot Studio, Bedrock) used model names that matched nothing in the price table either" with:

> This bug existed from day one but went undetected because Copilot Studio agents typically ran on GPT-4o — which happened to be the fallback price too. The erroneous lookup key ("copilot_studio") fell back to GPT-4o pricing, which was accidentally correct. Vertex AI events, priced at 10–50× less per token than GPT-4o (Gemini 2.0 Flash at $0.0001/$0.0004 vs GPT-4o at $0.005/$0.015), made the error undeniable.

### Fix title and "third cloud" framing

Article is about adding the **second** connector (Vertex AI after Copilot Studio), not the third. Update title and all "third cloud" references to "second connector" or "a second cloud."

### Fix forward-looking statement

"we'll apply" → "a pattern worth applying"
