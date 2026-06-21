30-second polling intervals in a monitoring dashboard mean a stalled AI agent can burn compute silently for half a minute before an operator notices. Here's how we fixed it.

Our Mission Control dashboard for claude-mcd (our 19-agent AI fleet) had two 30-second polling loops: one for fleet health, one for stall detection. With multiple operator tabs open, that's 4+ round-trips/minute — each re-reading every project's state from disk.

We replaced them with a single Server-Sent Events stream. Here's the architecture in three layers:

**Server broadcaster (`src/sse.ts`):** A single `setInterval` at 5s pushes fleet diffs to all connected clients. It starts on the first SSE connection and stops when the last client disconnects — zero overhead when nobody's watching.

**`globalThis` singleton pattern:** We store the interval on `globalThis` instead of module scope so it survives Next.js hot module replacement. Module scope resets on every HMR cycle; `globalThis` doesn't.

**`FleetContext` (React):** One `EventSource` per browser tab distributes `fleet-update` and `stall-alert` events to all components via `useContext`. Exponential backoff (1s → 30s) on reconnect. REST polling kept as fallback when SSE disconnects.

Results: update latency dropped from ≤30s to 5s. Fleet/stall polling eliminated. Budget alerts (50%/80%/100% token spend thresholds) piggyback the same SSE pipe at no extra cost.

The core — broadcaster, stream route, React context — is ~243 lines.

Full article → [link placeholder — operator fills in after Medium publish]

#SSE #React #NextJS #AIInfrastructure #WebDevelopment
