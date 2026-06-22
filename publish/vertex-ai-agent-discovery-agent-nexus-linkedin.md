We shipped a new AI platform connector and broke our own multi-cloud abstraction in three ways.

Adding Vertex AI to Agent Nexus (our multi-tenant AI agent control plane) revealed assumptions we didn't know we were making. The `IPlatformConnector` interface held — but the design behind it didn't.

Three things broke:

**1. Host credentials in a multi-tenant system**
The initial connector used host Application Default Credentials. In production, every tenant's agent discovery ran as the developer identity, surfacing the wrong project's agents. Fix: resolve per-tenant GCP credentials from Infisical at sync time, per connection.

**2. Failed connections never retried**
Our scheduler only dispatched syncs for "connected" status. A connection that failed twice stayed silently broken. One-line fix to include "error" status — plus a dedicated "needs_reauth" status to avoid retrying expired tokens indefinitely.

**3. A latent cost pricing bug**
ComputeCost was called with `record.Platform` ("vertex_ai") instead of `record.Model` ("gemini-2.5-flash"). The price table is keyed by model name. Every Gemini event hit the GPT-4o fallback — a 50× overcharge. The bug existed since day one; Copilot Studio agents on GPT-4o had made it accidentally correct.

The real lesson: the first connector validates your interface. The second validates your assumptions.

Full article → [link placeholder — operator fills in after Medium publish]

#AI #MultiCloud #VertexAI #EnterpriseEngineering #AgentNexus
