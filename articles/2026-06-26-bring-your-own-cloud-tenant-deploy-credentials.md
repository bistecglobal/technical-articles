---
title: "Whose Cloud Does It Deploy To? Tenant-Scoped Deploy Credentials Done Carefully"
project: agent-nexus
tags: [Cloud, MultiTenancy, Security, AI, Azure]
status: audited
date: 2026-06-26
---

Here is a question that should keep anyone building a multi-tenant platform up at night: when a tenant clicks "deploy," whose cloud account does the thing actually land in?

In Agent Nexus — our control plane for building, deploying, and governing AI agents across Copilot Studio, Azure AI Foundry, Bedrock, and Vertex — the honest answer used to be "the operator's." The Foundry deploy endpoint read every Azure credential it needed from process environment variables: the service-principal trio, the project endpoint, the API key. Those env vars belonged to us, the operator. So every tenant's agent deployed into one single, global, operator-owned Foundry project. A tenant who wanted to deploy into *their own* Foundry project simply couldn't — not without an operator editing process config on their behalf.

That breaks the whole self-service promise of a multi-tenant platform, and it violates an invariant we hold everywhere else in the system: vendor credentials are tenant-scoped and live in a secrets manager, never read from ambient process configuration. Deploy was the one place that hadn't caught up. This is the story of closing that gap — and of the surprisingly delicate fallback logic required to do it without opening a security hole.

## The model: Platform Connectors

Agent Nexus already had the right machinery for tenant-owned credentials: Platform Connectors. A tenant saves credentials through the UI, the platform live-verifies them, and only then stores them — per tenant, in Infisical (our secrets store) — keyed at a path like `/{tenantId}/azure_ai_foundry/{connectionId}`. The database holds nothing but a `SecretRef` pointer. The same pipeline already backed telemetry ingest and CLI provider keys.

The change extended that pipeline to cover deploy credentials, declared as additional fields on the existing `azure_ai_foundry` connection. A single descriptor became the source of truth for the field set, consumed identically by the save-time verifier, the internal resolve endpoint, and the frontend's auth-method selector — so the three layers can never drift.

## Exactly one method, or none

Foundry deploy supports two mutually-exclusive auth methods: a **Service Principal** (tenant id, client id, client secret, project endpoint) or an **api-key** (OpenAI endpoint plus key). "Mutually exclusive" is easy to say and easy to get wrong. The descriptor classifies a submitted credential bag into one of four outcomes, and the interesting cases are the ones that *aren't* a clean single method:

```csharp
var spPresent  = HasAny(ServicePrincipalFields);
var keyPresent = HasAny(ApiKeyFields);

if (!spPresent && !keyPresent)
    return DeployCredentialKind.None;          // telemetry-only save — not an error

if (spPresent && keyPresent)
{
    error = "Provide exactly one Foundry deploy auth method — Service Principal or api-key, not both.";
    return DeployCredentialKind.Incomplete;
}
```

Two design decisions are baked in here. A save with *no* deploy fields at all is `None`, not an error — the connection might be telemetry-only, and we don't punish that. A save with fields from *both* methods is rejected outright, because a half-and-half credential set has no well-defined meaning. The classifier counts a field as present only when its value is non-blank, so a stray empty string can't be mistaken for a real credential.

## The dangerous part: fallback

Now the subtle bit. At deploy time the endpoint resolves credentials from the tenant's connection, and for backward compatibility it falls back to the old environment variables when a tenant has no connection. That fallback is exactly where a careless implementation leaks one tenant's deploy into the operator's account. We drew three sharp lines.

**No per-field mixing.** When a tenant connection resolves, the deploy uses *only* its credentials — env vars never backfill individual fields:

```csharp
var hasResolvedConnection = resolvedFields.Count > 0;
string? Cred(string key) => hasResolvedConnection
    ? (resolvedFields.TryGetValue(key, out var v) && !string.IsNullOrWhiteSpace(v) ? v : null)
    : cfg[key];
```

Why all-or-nothing? Because per-field mixing would let an ambient service-principal env var silently shadow a tenant's verified api-key (or vice versa), quietly defeating the "exactly one method" guarantee we just enforced at save time. Credentials resolve wholesale from one source or the other — never spliced.

**Fall back only when nothing resolved.** A 404 (the tenant has no connection) or an unreachable Connectors service both mean "no tenant credentials exist" — safe to use the operator's env vars, preserving the old behaviour for operator-global setups.

**But fail closed when a connection exists and can't be read.** This is the line that matters most. If a tenant *does* have a deploy connection but its credentials can't be resolved — say Infisical is having a bad day — the endpoint returns a 503 and refuses to deploy:

```csharp
if (resolution.Unresolvable)
    return Results.Problem(
        "Deploy credentials are temporarily unavailable (credential service error). Please retry shortly.",
        statusCode: 503);
```

Falling back to env vars here would be catastrophic: the tenant intended to deploy into *their* Foundry project, the platform couldn't reach their credentials, so it would deploy into the *operator's* global project instead. A transient outage would silently misroute a tenant's agent into someone else's cloud. Failing closed turns a security incident into a retry.

## Beyond credentials: a deploy that's safe to call twice

The same endpoint hardens the deploy path in two more ways worth noting, because tenant-facing deploy buttons get clicked twice. Concurrent deploys for one build session are serialized with a Postgres advisory lock, so two near-simultaneous requests can't both create an Azure agent. And the deploy is idempotent: it computes a content hash of the build artifacts and consults a stored deployment record — same hash means return the existing agent untouched, a changed hash means update the existing agent by id rather than spawn a duplicate. There's also an evaluation gate (deploy only when the latest persisted verdict is `DEPLOY`) and a stale-pass guard that blocks deploy if the agent was regenerated after it last passed evaluation.

## Outcome and takeaways

The feature shipped with 33 passing unit tests — 28 on the connectors side, 5 on the deploy resolver — covering the classifier's edge cases, the verifier, and the resolve-and-fallback matrix specifically. That test count isn't vanity: nearly every test exists because some fallback combination *could* have leaked or mismatched credentials, and we wanted each one pinned.

Two lessons generalise. First, **the fallback is the feature.** Adding tenant-scoped credentials was the easy 80%; the hard, security-critical 20% was deciding precisely when *not* to fall back to operator defaults. "Fail closed when a credential might be wrong" should be the default instinct, not an afterthought. Second, **reusing the existing pipeline beat inventing a new one.** Deploy credentials piggybacked on the same connector, verifier, and secrets path already proven for telemetry and provider keys — so there was one credential model to reason about, not two. In multi-tenant systems, a single well-worn path for "whose credentials are these?" is worth more than any individual feature built on top of it.
