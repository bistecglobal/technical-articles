Your cost data is wrong — and you probably don't know it.

We discovered this while extending Agent Nexus to govern Google Vertex AI agents. Our OTLP cost computation was keying on the *platform name* instead of the *model name*. Every agent on every cloud had been silently falling back to the same hardcoded GPT-4o price. One line fix — but it exposed a deeper lesson about multi-cloud observability.

Here's what we built and what we learned:

🔐 **Tenant credentials, not host ADC.** Using `GoogleCredential.GetApplicationDefaultAsync()` would authenticate every tenant's Vertex discovery as the developer identity. We resolved GCP credentials from each tenant's own Infisical secret — the same two-tier pattern we use for Azure.

🔍 **Discovery via Reasoning Engines API.** Vertex AI agents (built with ADK) appear as Reasoning Engines. The connector hits `reasoningEngines.list` with the tenant's service account token, handles pagination, and normalises results into our shared `DiscoveredAgentInfo` model.

💰 **Model-level pricing, not platform-level.** Gemini 2.5 Flash costs $0.0003/1k input tokens. GPT-4o costs $0.005/1k. Using the platform name as the lookup key makes both look the same. We added Gemini price rows and fixed the lookup to use the model identifier from telemetry.

🔌 **Connectors keep integration predictable.** Plugging in Vertex AI required registering one class in the factory. The scheduler, consumer, and message bus handled it without platform-specific branching — though end-to-end the feature also touched migrations, ingest logic, and frontend hooks.

Full article → [link placeholder — operator fills in after Medium publish]

#AIAgents #MultiCloud #GCP #VertexAI #SoftwareEngineering
