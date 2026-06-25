# Audit — One Tenant, Two IDs: The Identity Bug Hiding in Your Multi-Tenant Guard

**Verdict: PASS**
**Score: 24/25**
**Date:** 2026-06-25
**Evidence base:** agent-nexus commit `d40a15e`. Files: `src/Nexus.Forge/Endpoints/BuildSessionCreateEndpoint.cs`, `src/Nexus.Gateway/Program.cs`.

## Claim inventory

| # | Claim | Verdict |
|---|-------|---------|
| 1 | Build-session create endpoint refuses unless body tenantId matches JWT `tenant_id` claim (cross-tenant guard) | ✅ verified (endpoint logic + comment) |
| 2 | Original guard used `string.Equals(..., OrdinalIgnoreCase)` | ✅ verified (removed line in diff) |
| 3 | Zitadel represents tenant id as either GUID or int64 snowflake | ✅ verified (commit msg + added comment) |
| 4 | Same tenant in two forms → raw string compare yields false `tenant_mismatch` 400 | ✅ verified (logical consequence of old code; commit "Fix forge 400") |
| 5 | `TryParseToGuid`: GUID parses directly; snowflake → `WriteInt64BigEndian` into 16 bytes → `new Guid(bytes)` | ✅ verified (verbatim) |
| 6 | Missing/unparseable claim → `missing_tenant_claim` | ✅ verified |
| 7 | Unparseable body or GUID mismatch → `tenant_mismatch` | ✅ verified |
| 8 | Guard now compares GUIDs, not strings | ✅ verified |
| 9 | Parser mirrors `TenantMiddleware.TryParseTenantId` (same rule both places) | ✅ verified (code comment) |
| 10 | Gateway validates Bearer / resolves tenant at edge, injects `X-Tenant-Id` downstream | ✅ verified (Program.cs diff + comment) |
| 11 | Internal services use dev-auth and trust the header without re-validating JWT | ✅ verified (diff comment) |
| 12 | Injection only when `TenantId != Guid.Empty` | ✅ verified |
| 13 | Tenant middleware (and injection) applies only on `/api` routes | ✅ verified (`StartsWithSegments("/api")`) |
| 14 | AllowedOrigins made configurable (was hardcoded localhost) | ✅ verified (`GetSection("AllowedOrigins")`) |

14/14 verified. Zero ⚠️ / ❌.

## Forward-looking scan

No matches for will (roadmap sense) / plan to / coming soon / in the future / next step / roadmap / soon. "introduced later" and "will eventually let the wrong request in" are instructional/cautionary, not product roadmap. PASS.

## Rubric

| Dimension | Score | Notes |
|---|---|---|
| Evidence quality | 5 | Every claim maps to diff; no SHAs/PR# in prose |
| Technical depth | 5 | Snowflake→GUID widening + single-trust-boundary architecture both grounded |
| Clarity for audience | 5 | "Two spellings of one identity" frames it cleanly; tight arc |
| BistecGlobal voice | 5 | Practitioner, security-minded, transferable principle |
| Title specificity | 4 | Specific, intriguing; slightly headline-y but accurate |

**Total: 24/25** — exceeds threshold.

## Fixes applied

None required. Frontmatter status promoted draft → audited.
