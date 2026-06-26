# Audit — One Forgotten WHERE Clause (postgres-rls-tenant-isolation)

**Verdict: PASS**
**Score: 25/25**
**Claims: 20 verified, 0 weak, 0 unverified · 0 flagged · 0 fixed**

Source of truth: agent-nexus PR 237 (`1596b7a`, on origin/main) + `docs/tenant-isolation-rls.md`.

## Claim inventory

| # | Claim | Evidence | Verdict |
|---|---|---|---|
| 1 | Three layers: TenantMiddleware, EF query filter, Postgres RLS | doc "The three layers" | ✅ |
| 2 | Tenant from Zitadel org_id/tenant_id (prod) / X-Tenant-Id (dev) | doc layer 1 | ✅ |
| 3 | EF `HasQueryFilter(e => e.TenantId == ...)` | doc layer 2 | ✅ |
| 4 | ENABLE + FORCE ROW LEVEL SECURITY on every tenant table | doc + commit msg | ✅ |
| 5 | `current_tenant_id()` function SQL | verbatim from doc | ✅ |
| 6 | `tenant_isolation` policy USING/WITH CHECK | verbatim from doc | ✅ |
| 7 | TenantGucConnectionInterceptor (DbConnectionInterceptor) sets app.current_tenant on every connection open | doc "How RLS resolves the tenant" | ✅ |
| 8 | Unresolved tenant → NULL → 0 rows (fail closed) | doc | ✅ |
| 9 | Npgsql resets session state on pool return | doc | ✅ |
| 10 | Owner `nexus` is SUPERUSER+BYPASSRLS → RLS silently ignored | doc "why owner can't be app user" | ✅ |
| 11 | `nexus_app` role SQL (LOGIN NOSUPERUSER NOBYPASSRLS INHERIT; GRANT nexus) | verbatim from doc | ✅ |
| 12 | BYPASSRLS never inherited; not owner so FORCE applies | doc | ✅ |
| 13 | Migrations run as nexus; app connstrings point at nexus_app | doc + commit msg | ✅ |
| 14 | Empirically verified: GUC→own rows, no GUC→0, owner DDL still runs | doc + commit "Verification" | ✅ |
| 15 | CrossTenant opt-in, defaults false, HTTP never sets, server-only/audited | doc "escape hatch" | ✅ |
| 16 | Consumers carrying TenantId seed it (strict) vs bypass for sweeps | doc table | ✅ |
| 17 | MassTransit sagas excluded; CorrelationId GUID; TenantId routing field; no cross-tenant listing | doc "NOT under RLS" | ✅ |
| 18 | Residual internal-network header-trust → impersonate one tenant | doc "Gotchas" + bug doc ref | ✅ |
| 19 | Removed Platform tier / DeveloperTenant / super-account | commit msg "Changes" | ✅ |
| 20 | Shipped: suites green, +5 interceptor tests, fresh-DB docker stack proving 3 layers | commit msg "Verification" | ✅ |

## Forward-looking scan
No matches for will / plan to / coming soon / in the future / next step / roadmap / soon. All outcomes stated past/present. Clean.

## Rubric

| Dimension | Score |
|---|---|
| Evidence quality (no raw SHAs in prose) | 5 |
| Technical depth | 5 |
| Clarity for audience | 5 |
| BistecGlobal voice | 5 |
| Title specificity | 5 |
| **Total** | **25/25** |

No fixes required. Status → audited.
