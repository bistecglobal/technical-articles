---
title: "Two Kinds of Users, One Platform: How We Built Credential Tiers into Agent Nexus"
project: agent-nexus
tags: [AI, Authentication, Multi-Tenant, Zitadel, Infisical, .NET, Enterprise]
status: audited
date: 2026-06-22
---

# Two Kinds of Users, One Platform: How We Built Credential Tiers into Agent Nexus

Agent Nexus is a multi-tenant control plane for governing AI agents. Our tenants — enterprise customers — run agents on Copilot Studio and Vertex AI and use Agent Nexus to discover, monitor, and govern them. Our developers — the Bistec team — connect platforms, configure integrations, and set up shared infrastructure. Both groups log in. Both use the same API. But they should never be able to touch each other's secrets.

That's a harder problem than it sounds. A single JWT authentication layer tells you *who* someone is. It doesn't tell you *what kind* of user they are and which secrets vault they're allowed to write to. We built a credential tier system into Agent Nexus to solve exactly this — and the design decisions along the way were more interesting than expected.

## The Problem: Two Users, One API, Two Secret Vaults

When a user creates a platform connection in Agent Nexus, we store their credentials (API keys, service account JSON, OAuth tokens) in Infisical. But we have two different Infisical projects with two different machine identities:

- **The tenants project** — accessed by `nexus-gateway`, our primary service identity. This is where tenant-owned secrets live: `/tenantId/platform/connectionId`. Tenant users write here.
- **The platform project** — accessed by `nexus-platform`, a separate identity with no access to the tenants project. This is where Bistec developer credentials live: `/platform/connectionId`. Developer users write here.

No single identity can access both. If you get the routing wrong — if a developer's request lands in the tenants project, or a tenant request gets the platform identity — you either leak cross-tenant data or allow a customer to overwrite shared infrastructure credentials.

The JWT doesn't solve this by itself. We needed a way to derive *which tier a user belongs to* from their token, and route every downstream operation accordingly.

## From JWT Claim to CredentialTier

We use Zitadel as our identity provider. Zitadel supports project-role assertions: when a user authenticates, their JWT includes a `urn:zitadel:iam:org:project:roles` claim containing a JSON object whose keys are the roles they hold in our project. We defined two roles: `developer` (Bistec staff) and `tenant` (customer users).

The middleware extracts these roles and resolves a `CredentialTier` enum:

```csharp
public enum CredentialTier
{
    None = 0,
    Platform,
    Tenant,
}

public static class CredentialTierResolver
{
    public static CredentialTier Resolve(IReadOnlyList<string> roles)
    {
        var hasDeveloper = roles.Contains("developer", StringComparer.OrdinalIgnoreCase);
        var hasTenant    = roles.Contains("tenant",    StringComparer.OrdinalIgnoreCase);

        if (hasDeveloper && hasTenant) return CredentialTier.None; // ambiguous — deny
        if (hasDeveloper)              return CredentialTier.Platform;
        if (hasTenant)                 return CredentialTier.Tenant;
        return CredentialTier.None;
    }
}
```

Two decisions worth calling out:

**Ambiguity is denied, not resolved.** If a token carries both `developer` and `tenant` roles — a misconfiguration that is easy to make during Zitadel project role setup — the resolver returns `None`, which gates the request with a 403. We considered defaulting to the lower-privilege tier, but decided that a confused token should never succeed quietly. Fail closed.

**Zitadel's role claim needs parsing.** The `urn:zitadel:iam:org:project:roles` claim is a JSON object like `{"developer": {"orgId": "..."}}` — the role names are the keys. The middleware uses `JsonDocument` to extract those keys, merging them with any flat `role` claims for backwards compatibility:

```csharp
var projectRolesClaim = context.User.FindFirst("urn:zitadel:iam:org:project:roles")?.Value;
var projectRoles = ParseProjectRoleKeys(projectRolesClaim);
var allRoles = flatRoles.Concat(projectRoles).Distinct(...).ToList();
tenantContext.Tier = CredentialTierResolver.Resolve(allRoles);
```

One more parsing edge: Zitadel org IDs are 64-bit snowflake integers, not GUIDs. Our EF Core models use `Guid` for tenant IDs. We convert deterministically by encoding the snowflake into the first 8 bytes of a `Guid` — the same org ID always produces the same GUID, with no database schema changes needed.

## Routing Secrets by Tier

With the tier resolved and stamped on `TenantContext`, every endpoint that touches Infisical routes its operations accordingly. The `ConnectionEndpoints.MapPost` handler uses `ITieredInfisicalClient` rather than the plain `IInfisicalClient`:

```csharp
if (tenantContext.Tier == CredentialTier.None)
    return Results.Forbid();

string path = tenantContext.Tier == CredentialTier.Platform
    ? $"/{conn.Platform}/{conn.Id}"            // platform project, no tenant segment
    : $"/{conn.TenantId}/{conn.Platform}/{conn.Id}";  // tenants project, under tenant

conn.SecretRef = $"{env}:{path}";
await infisical.SetSecretAsync(tenantContext.Tier, env, path, name, value, ct);
```

`ITieredInfisicalClient.SetSecretAsync` routes to the correct Infisical identity based on the tier:

```csharp
private async Task<(string token, string workspaceId)> GetTokenAndWorkspaceAsync(
    CredentialTier tier, CancellationToken ct)
{
    return tier switch
    {
        CredentialTier.Platform => (
            await GetAccessTokenAsync(_opts.PlatformClientId, _opts.PlatformClientSecret, _platformCache, ct),
            _opts.PlatformWorkspaceId),
        _ => (
            await GetAccessTokenAsync(_opts.ClientId, _opts.ClientSecret, _tenantCache, ct),
            _opts.WorkspaceId),
    };
}
```

Each tier uses a separate cached access token. The `nexus-platform` machine identity has no `WorkspaceId` in the tenants project — if the tier routing bugs out and tries to use it there, Infisical rejects the token. The enforcement is bilateral: code logic on our side, identity scope on Infisical's side.

## The Frontend Mirror

The tier isn't only a backend concept. The React frontend mirrors `CredentialTierResolver` in TypeScript, reading the same Zitadel claim from the OIDC user object and exposing `tier` on `AuthContext`. If the resolved tier is `'none'`, an `AuthGate` component renders an "Access denied" screen immediately after authentication — no server round-trip needed to discover the user lacks any valid role.

This duplication is intentional. The backend is the authority; the frontend is UX. Both read from the same JWT source of truth, so they should agree — and when they don't (a test scenario we cover in integration tests), the backend wins.

## What We Got Out of It

The credential tier system is a two-layer enforcement: Zitadel role assertion ensures only the right users carry the right claims, and `ITieredInfisicalClient` ensures those claims route to the right vault identity. No single surface can be compromised to cross the boundary — a misconfigured Zitadel role produces `None` and a 403, a misconfigured Infisical identity produces an auth failure from Infisical itself.

Multi-tenant platforms often treat "who are you" and "what can you touch" as one question, answering both with permission checks on individual resources. That works for data. It's insufficient for infrastructure credentials, where the wrong write doesn't just expose a row — it puts one tenant's secrets under another tenant's identity, or gives a customer write access to shared platform configuration.

The design that helped most: model the user type as a first-class value in the request context, not as a permission check at the endpoint level. Once `CredentialTier` is stamped on every request, every downstream operation — secret paths, Infisical identities, and EF Core tenant queries scoped by `TenantId` — flows from the same `TenantContext` without each service needing to re-derive who the caller is.

---

*Agent Nexus is BistecGlobal's multi-tenant control plane for AI agent governance across Copilot Studio and Vertex AI.*
