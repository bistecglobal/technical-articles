# Audit: two-tier-auth-zitadel-agent-nexus

**Date:** 2026-06-22  
**Verdict: PASS** (with two minor inline fixes applied)  
**Score: 22/25** → **23/25** post-fix

---

## Claim Inventory

| # | Claim | Status | Evidence |
|---|-------|--------|----------|
| 1 | Agent Nexus is multi-tenant control plane for AI agent governance | ✅ | Project description, README |
| 2 | Tenants use Copilot Studio and Vertex AI | ✅ | `PlatformConnectorFactory` registers `copilot_studio` + `vertex_ai` |
| 3 | Two Infisical projects: tenants (nexus-gateway) and platform (nexus-platform) | ✅ | `InfisicalOptions` has `WorkspaceId` + `PlatformWorkspaceId`; `InfisicalClient` has `_tenantCache` + `_platformCache`; commit T06 documents "nexus-gateway" and "nexus-platform" identity names |
| 4 | Tenant secret path: `/{tenantId}/{platform}/{connectionId}` | ✅ | `ConnectionEndpoints.cs` exact match |
| 5 | Platform secret path: `/{platform}/{connectionId}` | ✅ | `ConnectionEndpoints.cs` exact match |
| 6 | Zitadel JWT includes `urn:zitadel:iam:org:project:roles` claim | ✅ | `TenantMiddleware.cs`: `FindFirst("urn:zitadel:iam:org:project:roles")` |
| 7 | Two Zitadel roles: `developer` and `tenant` | ✅ | `CredentialTierResolver.Resolve()` checks these exact strings |
| 8 | CredentialTier enum: None, Platform, Tenant | ✅ | `CredentialTier.cs` exact match |
| 9 | Both roles → None (ambiguous-deny) | ✅ | `CredentialTier.cs` line 18-19 |
| 10 | CredentialTierResolver code snippet | ✅ | Exact match to `CredentialTier.cs` |
| 11 | "a misconfiguration we've seen in staging" | ⚠️ | Not verifiable from code; test `resolveTier({ developer, tenant })` confirms the scenario is tested, not that it was seen in staging. Soften. |
| 12 | Role claim is JSON object with role names as keys | ✅ | `TenantMiddleware.cs: ParseProjectRoleKeys` — `EnumerateObject().Select(p => p.Name)` |
| 13 | JsonDocument extraction in middleware | ✅ | `TenantMiddleware.cs` exact match |
| 14 | Snowflake ID → Guid via BinaryPrimitives | ✅ | `TenantMiddleware.cs:79-84` — `BinaryPrimitives.WriteInt64BigEndian(bytes, snowflake)` |
| 15 | `ConnectionEndpoints.MapPost` uses `ITieredInfisicalClient` | ✅ | `ConnectionEndpoints.cs:42` |
| 16 | ConnectionEndpoints secret routing code snippet | ✅ | Exact match |
| 17 | `GetTokenAndWorkspaceAsync` switch snippet | ✅ | Exact match to `InfisicalClient.cs:131-143` |
| 18 | Separate cached tokens per tier (`_tenantCache`, `_platformCache`) | ✅ | `InfisicalClient.cs:43-44` |
| 19 | Frontend `resolveTier` mirrors backend resolver | ✅ | `AuthProvider.tsx:13` — `export function resolveTier(user)` |
| 20 | `AuthGate` renders "Access denied" for `tier === 'none'` | ✅ | `App.tsx:59-61` |
| 21 | "test scenario we cover in integration tests" | ✅ | `tierRouting.test.tsx` covers both-roles-none, single-role-platform, single-role-tenant, no-roles-none |
| 22 | "EF Core tenant filters can branch on it [CredentialTier]" | ❌ | EF Core query filters use `TenantId` from `TenantContext`, NOT `CredentialTier`. `ConnectorDbContext.cs:36`: `HasQueryFilter(e => e.TenantId == _tenantContext.TenantId)`. The claim that CredentialTier enables EF Core filter branching is imprecise — fix to say "TenantId" flows from the same TenantContext. |

---

## Forward-Looking Scan

No matches for "will", "plan to", "coming soon", "in the future", "next step", "roadmap", "soon". ✅ Clean.

---

## Rubric Scorecard

| Dimension | Pre-fix | Post-fix | Notes |
|-----------|---------|----------|-------|
| Evidence quality (verifiable, no SHAs in prose) | 4 | 5 | Two minor issues: anecdote + EF Core mischaracterisation |
| Technical depth | 5 | 5 | Covers enum design, JWT parsing, Zitadel claim format, snowflake IDs, secret routing, frontend mirror, bilateral enforcement |
| Clarity for target audience | 4 | 4 | Good narrative arc, three clear sections |
| BistecGlobal voice | 4 | 4 | Professional, practitioner-focused |
| Title specificity | 5 | 5 | Specific and human |
| **Total** | **22/25** | **23/25** | |

---

## Rewrite Instructions

### Fix claim 11 — remove unverifiable staging anecdote

**Before:** "a misconfiguration we've seen in staging"  
**After:** "a misconfiguration that is easy to make during Zitadel project role setup"

### Fix claim 22 — correct EF Core filter description

**Before:** "every downstream operation — secret paths, Infisical identities, EF Core tenant filters — can branch on it [CredentialTier]"  
**After:** "every downstream operation — secret paths, Infisical identities, and EF Core tenant queries filtered by `TenantId` — flows from the same `TenantContext` without each service re-deriving who the caller is"
