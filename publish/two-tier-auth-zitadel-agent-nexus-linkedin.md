JWT tells you who someone is. It doesn't tell you which secrets vault they're allowed to write to.

Agent Nexus serves two user types on one API: enterprise customers and our own developers. Both authenticate with Zitadel, hit the same endpoints — but must write credentials to completely separate Infisical vaults with separate machine identities that can't cross over.

How we solved it with a three-value enum and bilateral enforcement:

**CredentialTier from JWT**
Zitadel's project-roles claim keys resolve to `Platform` (developer), `Tenant` (customer), or `None`. Both roles simultaneously → `None` + 403. Fail closed on ambiguity, always.

**Secret path routing by tier**
Tenant → `/{tenantId}/{platform}/{connectionId}` in the tenants Infisical project (nexus-gateway identity). Platform → `/{platform}/{connectionId}` in the platform project (nexus-platform identity). No single identity touches both vaults.

**Bilateral enforcement**
Our code routes by tier. Infisical enforces at the identity level — the wrong machine identity gets rejected regardless of what our code sent.

**Frontend mirror**
React's `resolveTier()` reads the same Zitadel claim; `AuthGate` shows "Access denied" for `tier === 'none'` before any API call. Backend is authority, frontend is UX.

The lesson: model user type as a first-class value in request context, not endpoint-level permission checks.

Full article → [link placeholder — operator fills in after Medium publish]

#AI #MultiTenant #Authentication #Zitadel #EnterpriseEngineering
