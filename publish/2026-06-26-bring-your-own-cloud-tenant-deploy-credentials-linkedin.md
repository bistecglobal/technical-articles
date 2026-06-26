When a tenant clicks "deploy" on your multi-tenant platform — whose cloud account does the thing actually land in?

In Agent Nexus, the deploy endpoint used to read every Azure credential from operator-owned environment variables, so every tenant's AI agent deployed into one global, operator-owned Foundry project. We fixed it by letting tenants bring their own deploy credentials — and the hard part wasn't adding them.

The hard part was the fallback:

• Tenant credentials resolve wholesale from one source — env vars never backfill individual fields (per-field mixing would shadow a verified auth method)
• No connection / connectors unreachable → fall back to operator env vars (back-compat, safe)
• Connection exists but credentials can't be read (secrets-store outage) → FAIL CLOSED with a 503, never fall back

That last line is the whole ballgame. Falling back to env vars there would silently deploy a tenant's agent into the operator's cloud. Failing closed turns a security incident into a retry.

Shipped with 33 unit tests — nearly every one exists because some fallback combination *could* have leaked or mismatched credentials.

Two lessons: the fallback is the feature. And reusing your existing credential pipeline beats inventing a second one.

Full article → [link placeholder — operator fills in after Medium publish]

#MultiTenancy #CloudSecurity #AIAgents #Azure #SoftwareArchitecture
