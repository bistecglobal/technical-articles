We discovered our AI agent cost scores were permanently maxed at 100 — because the scoring engine read from a ledger that the ingest pipeline never wrote to.

Here's how we fixed that, along with two other silent failures in our OTLP observability pipeline for Agent Nexus:

🔍 **The problem**: Running AI agents across Copilot Studio, Azure AI Foundry, and Vertex AI gave us a fragmented view — different telemetry formats, no runtime cost data, and tenant identity being read from an HTTP header (which any in-cluster caller could spoof).

⚙️ **What we built**: A dedicated Nexus.Observability service that ingests OTLP log records, computes cost at ingest time (per-model USD pricing for GPT-4o through Gemini 2.5), and calculates a live AgentScore across four dimensions: reliability (35%), latency (25%), cost (20%), throughput (20%).

🐛 **Bug 1 — CostScore always 100**: The scorer summed CostRecords; the ingest path wrote to OperationalEvents.CostUsd. Table stayed empty → score permanently maxed. Fix: source cost from OperationalEvents, matching the cost summary endpoint.

⚡ **Bug 2 — Unbounded table growth under burst**: Every telemetry event triggered a full 24h aggregate scan and appended a new row. Fix: 30-second debounce window + upsert with unique index on (TenantId, AgentId) + DbUpdateException conflict handling for concurrent inserts.

🔒 **Tenant isolation**: tenant.id travels inside each OTLP record attribute — never from an HTTP header. The collector stamps it at the source; the ingest endpoint never trusts ambient headers.

Full article → [link placeholder — operator fills in after Medium publish]

#AIEngineering #OpenTelemetry #MultiTenant #Observability #BistecGlobal
