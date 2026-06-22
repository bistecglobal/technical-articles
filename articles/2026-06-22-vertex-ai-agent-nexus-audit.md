# Audit: vertex-ai-agent-nexus

**Date:** 2026-06-22  
**Verdict:** PASS (fixes applied inline)  
**Score:** 23/25 → 25/25 after fixes

---

## Claim Inventory

| # | Claim | Verdict | Evidence |
|---|---|---|---|
| 1 | Agent Nexus is a single control plane for AI agents — score, cost, governance | ✅ | Established from codebase context and prior articles |
| 2 | Agent Nexus had Copilot Studio support | ✅ | `PlatformConnectorFactory.cs` registers `copilot_studio` → `CopilotStudioConnector` |
| 3 | "We were ingesting Bedrock telemetry" | ❌ | Bedrock appears only in wizard UI (`StepFormDefaults.cs`), not in `PlatformConnectorFactory` or price tables. No `BedrockConnector` exists. **Fixed.** |
| 4 | Vertex AI is Google's runtime for agents built with Agent Development Kit | ✅ | `VertexAiConnector.cs` doc comment: "Lists agents deployed to the Google Agent Platform (Vertex AI Reasoning Engine / ADK)" |
| 5 | Every cloud platform implements `IPlatformConnector` | ✅ | `IPlatformConnector.cs` — both `CopilotStudioConnector` and `VertexAiConnector` implement it |
| 6 | Factory routes sync requests by platform name | ✅ | `PlatformConnectorFactory.GetConnector(string platform)` |
| 7 | Factory maps `copilot_studio` → `CopilotStudioConnector`, `vertex_ai` → `VertexAiConnector` | ✅ | `PlatformConnectorFactory.cs` exact code |
| 8 | `PlatformConnection` carries a `SecretRef` pointing to Infisical | ✅ | `VertexAiConnector.ResolveCredentialsJsonAsync` reads `SecretRef` |
| 9 | SecretRef for Vertex resolves to `GCP_CREDENTIALS_JSON` | ✅ | `ResolveCredentialsJsonAsync` looks up key `GCP_CREDENTIALS_JSON` |
| 10 | Uses `CredentialFactory.FromJson`, non-deprecated replacement for `GoogleCredential.FromJson` | ✅ | `VertexAiConnector.cs` line 61; commit message states "Uses the non-deprecated CredentialFactory.FromJson (GoogleCredential.FromJson is obsolete)" |
| 11 | Credential scoped to `cloud-platform` | ✅ | `VertexAiConnector.cs` line 62: `.CreateScoped("https://www.googleapis.com/auth/cloud-platform")` |
| 12 | Agents appear as Reasoning Engines under Vertex AI API | ✅ | API URL in `FetchReasoningEnginesAsync` targets `reasoningEngines` endpoint |
| 13 | `EnvironmentUrl` format `projects/{projectId}/locations/{location}` | ✅ | `TryParseEnvironmentUrl` validates this format |
| 14 | Discovery hits `reasoningEngines.list` endpoint | ✅ | `VertexAiConnector.cs` `FetchReasoningEnginesAsync` |
| 15 | Results are paginated via `nextPageToken` | ✅ | `VertexAiConnector.cs` lines 150-153 |
| 16 | `SyncAgentsConsumer` picks up `SyncAgentsRequested` from message bus | ✅ | `SyncAgentsConsumer.cs`: `IConsumer<SyncAgentsRequested>` |
| 17 | `DiscoverySyncScheduler` is background service driving syncs | ✅ | `DiscoverySyncScheduler : BackgroundService` |
| 18 | `ComputeCost` was using `record.Platform` instead of `record.Model` | ✅ | Commit message: "fix ComputeCost to use record.Model (was record.Platform)" |
| 19 | "For Copilot Studio events this happened to work because platform name matched a model alias" | ❌ | `copilot_studio` is not in the Prices dict. Bug was silent for all platforms — everything fell back to the GPT-4o price. **Fixed.** |
| 20 | Vertex events with `gemini-2.5-flash` would miss the price lookup | ✅ | Prices dict keyed on model names; `vertex_ai` not in dict |
| 21 | Events fell back to GPT-4o price | ✅ | `OtlpIngestEndpoints.cs` line 193-194: fallback `p = (0.005, 0.015)` = GPT-4o |
| 22 | Fix was adding Gemini pricing rows and using `record.Model` | ✅ | `OtlpIngestEndpoints.cs` Gemini rows present; commit confirms model fix |
| 23 | Gemini pricing values correct (e.g. gemini-2.5-flash = 0.0003/0.0025) | ✅ | `OtlpIngestEndpoints.cs` exact values match article |
| 24 | `/api/costs/summary` groups by platform and by model from `OperationalEvents` | ✅ | `CostEndpoints.cs` lines 62-71 |
| 25 | Previously used a separate cost ledger | ✅ | Comment in `CostEndpoints.cs` line 49: "without a separate cost ledger" |
| 26 | `IPlatformConnector` has `ListAgentsAsync`, `GetStatusAsync`, `ReadConfigAsync`, lifecycle stubs | ✅ | `IPlatformConnector.cs`: all four present plus `DeployAsync`/`DeleteAsync` as stubs |
| 27 | "Touching fewer than a dozen lines outside the connector itself" | ❌ | Commit: 24 files, 836 insertions — includes migrations, frontend, OTLP, factory. **Fixed to: connector plugged into factory with minimal integration code.** |
| 28 | "Scheduler, consumer, and database layer required no changes" | ❌ | `SyncAgentsConsumer.cs` +1 line, migrations added `AgentName` column, `DiscoverySyncScheduler.cs` +2 lines. **Fixed.** |
| 29 | Connector model proven across MSAL (Copilot Studio) and GCP service account (Vertex AI) | ✅ | Two distinct auth systems in factory |

---

## Forward-Looking Scan

No instances of "will", "plan to", "coming soon", "in the future", "next step", "roadmap", "soon" found. **Clean.**

---

## Rubric Scorecard

| Dimension | Score | Notes |
|---|---|---|
| Evidence quality (pre-fix) | 3/5 | 3 ❌ claims; code snippets accurate throughout |
| Evidence quality (post-fix) | 5/5 | All claims verified or removed |
| Technical depth | 5/5 | Connector model, tenant ADC, pagination, cost pipeline all covered |
| Clarity for target audience | 5/5 | Clear narrative, good hook, readable code snippets |
| BistecGlobal voice | 5/5 | Professional, practitioner-focused, first-person plural |
| Title specificity | 5/5 | Specific, human, describes exactly what was built |

**Total (post-fix): 25/25**

---

## Fixes Applied

1. **Claim 3** — Removed "We were ingesting Bedrock telemetry." Replaced with accurate statement: Copilot Studio was connected; Vertex AI was the gap.
2. **Claim 19** — Removed "For Copilot Studio events this happened to work because the platform name sometimes matched a model alias." Replaced with: the bug was silent across all platforms — without model-matched pricing, every event fell back to the same hardcoded default price.
3. **Claims 27–28** — Removed "fewer than a dozen lines outside the connector itself" and "scheduler, consumer, and database layer required no changes." Replaced with accurate description: the connector interface kept integration points minimal, but the end-to-end feature also required database migrations, OTLP ingest updates, and cost endpoint revisions.
