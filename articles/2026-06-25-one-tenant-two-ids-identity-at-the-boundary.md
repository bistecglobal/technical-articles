---
title: "One Tenant, Two IDs: The Identity Bug Hiding in Your Multi-Tenant Guard"
project: agent-nexus
tags: [Security, Multi-Tenancy, Identity, DotNet, Architecture]
status: draft
date: 2026-06-25
---

The most dangerous line in a multi-tenant system is the one that decides whether request A is allowed to touch tenant B's data. In Agent Nexus — our control plane for governing AI agents across clouds — that line lives in the endpoint that creates a build session. A caller posts a request with a `tenantId` in the body, and the endpoint refuses to create the session unless that id matches the `tenant_id` claim baked into the caller's JWT. It's the single check that stops a cross-tenant POST from quietly materializing a session under someone else's umbrella.

The check looked airtight. It was a string comparison:

```csharp
if (!string.Equals(claimTenantId, request.TenantId, StringComparison.OrdinalIgnoreCase))
    return Results.BadRequest(new { error = "tenant_mismatch" });
```

Then legitimate requests started getting rejected with `tenant_mismatch`. Same tenant, same user, same everything — 400. The guard wasn't too loose. It was too literal.

## One identity, two spellings

The culprit was our identity provider. Zitadel can represent a tenant id in two different ways depending on the path it travels: sometimes as a GUID, sometimes as an int64 "snowflake" — a 64-bit integer. Both denote the *same* tenant. They're two spellings of one identity.

So the body might carry the snowflake `523847000000000001` while the JWT claim carried the GUID form of the very same tenant. `string.Equals` does exactly what it's told: it compares characters, sees `5238...` against `a1b2c3...`, and declares a mismatch. The security guard was correctly enforcing a rule against an identity it had failed to recognize as itself.

This is a subtle and common class of bug. It isn't a hole — nothing unauthorized gets through — but it's the equally serious inverse: a security control firing on false positives, blocking the people it's supposed to admit. Left alone, the pressure to "make it work" pushes someone toward loosening the check, which is exactly how a real hole gets introduced later.

## Normalize, then compare — never compare raw

The fix is the rule that should govern every identity comparison: **canonicalize both sides to one representation before you compare them.** You never compare identities in whatever form they happened to arrive; you convert them to a single form and compare that.

We added a small parser that accepts either spelling and produces a GUID. A real GUID parses directly. A snowflake gets deterministically widened into one — written as a big-endian int64 into a 16-byte buffer, then read back as a GUID:

```csharp
// Accepts GUID or Zitadel int64 snowflake; produces a canonical GUID.
private static bool TryParseToGuid(string value, out Guid guid)
{
    if (Guid.TryParse(value, out guid))
        return true;
    if (long.TryParse(value, out var snowflake))
    {
        Span<byte> bytes = stackalloc byte[16];
        BinaryPrimitives.WriteInt64BigEndian(bytes, snowflake);
        guid = new Guid(bytes);
        return true;
    }
    guid = Guid.Empty;
    return false;
}
```

The guard then compares GUIDs, not strings — and the failure modes get sharper in the process:

```csharp
var claimTenantId = http.User.FindFirst("tenant_id")?.Value;
if (string.IsNullOrWhiteSpace(claimTenantId) || !TryParseToGuid(claimTenantId, out var tenantGuid))
    return Results.BadRequest(new { error = "missing_tenant_claim" });

if (!TryParseToGuid(request.TenantId, out var bodyGuid) || tenantGuid != bodyGuid)
    return Results.BadRequest(new { error = "tenant_mismatch" });
```

A claim that's absent or unparseable is now a distinct `missing_tenant_claim`. A body id that's unparseable collapses into the same path as a real mismatch — an unrecognizable tenant id is treated as not-your-tenant, which is the safe default. The comparison is `Guid != Guid`, immune to casing, leading zeros, and representation drift.

One detail matters more than the code: that parser is a deliberate mirror of the one already used in the gateway's tenant middleware. The canonicalization rule — "GUID or snowflake, here's how they map" — now lives in exactly the spots that make tenant decisions, written identically. If two layers normalize identity differently, you've just built a new vulnerability in the seam between them. The fix isn't only "normalize" — it's "normalize the *same way everywhere*."

## Push the trust boundary to the edge

The second half of this work answers a question the first half raises: if identity has to be normalized consistently, who's responsible for establishing it? The answer in Agent Nexus is the gateway, and only the gateway.

The edge gateway is where the Bearer token is validated and the tenant resolved from its claims. Once it has done that, it injects the result downstream as a header so internal services don't each have to re-validate the JWT themselves:

```csharp
var middleware = new TenantMiddleware(async _ =>
{
    var tenantContext = context.RequestServices.GetRequiredService<TenantContext>();
    if (tenantContext.TenantId != Guid.Empty)
        context.Request.Headers["X-Tenant-Id"] = tenantContext.TenantId.ToString();
    await next(context);
});
```

Internal services run with a development-auth scheme and trust `X-Tenant-Id` as the tenant of record. That's a sound pattern precisely *because* the trust boundary is singular: token validation happens once, at the edge, and the resolved tenant flows inward as a simple, already-canonicalized header. The injection only fires for a genuinely resolved tenant — `TenantId != Guid.Empty` — and only on `/api` routes, so an unresolved request can never propagate an empty or forged tenant inward.

The architectural rule this encodes is worth saying plainly: **establish identity once, at the outermost layer, and pass the canonical form inward.** Re-deriving tenant identity at every hop multiplies the number of places that can normalize it inconsistently — which is the exact bug we just spent the first half of this article fixing.

## The takeaway

Nothing here is exotic. A string comparison, an integer, a header. But the two halves rhyme into one principle that holds far beyond this codebase: **identity is not the string you received — it's the canonical value that string denotes.** Normalize before you compare, normalize the same way in every place that compares, and establish the canonical form once at the trust boundary so the rest of the system never has to guess. Skip any of those three and your security controls will eventually either let the wrong request in or — as ours did — turn the right ones away.
