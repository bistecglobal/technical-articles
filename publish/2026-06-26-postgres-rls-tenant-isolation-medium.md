# One Forgotten WHERE Clause: Making the Database Enforce Tenant Isolation

*How we moved multi-tenant isolation out of application code and into Postgres Row-Level Security — and the sharp edges of every decision.*


Every multi-tenant bug report you never want to read starts the same way: "Customer A can see Customer B's data." And almost every one traces back to a single line of code that *should* have had `WHERE tenant_id = @me` and didn't. A new raw SQL query. A reporting endpoint someone bolted on. An `IgnoreQueryFilters()` call that looked harmless. The application was one forgotten filter away from a breach the whole time, and nobody knew until it leaked.

agent-nexus — our multi-tenant platform for building and running AI agents — lived with exactly that exposure. Tenant isolation rested on the application *remembering* to scope every query, plus a "Platform" super-account that could see across all tenants and was, by its own admission, an untested production path. We closed both holes by moving the boundary out of the application code and into the database itself: pure single-tenant model, plus Postgres Row-Level Security as a third, independently-enforced layer. Here's how, and why each decision had a sharp edge.

## Defense in depth, with the last layer non-negotiable

Isolation in agent-nexus now sits at three independent layers, so a miss at one is caught by the next:

1. **TenantMiddleware** resolves the caller's tenant — a Zitadel `org_id`/`tenant_id` claim in production, `X-Tenant-Id` in dev — into a scoped `ITenantContext`.
2. **EF Core query filter** applies `HasQueryFilter(e => e.TenantId == ...)` on every `DbContext`.
3. **Postgres RLS** enables `FORCE ROW LEVEL SECURITY` and a `tenant_isolation` policy on every tenant table.

The first two layers are application code, which means they're exactly the layers a developer can forget. The third can't be forgotten, because the database refuses to return rows that don't match — even to a raw query, even to a reporting tool, even to code that never heard of `ITenantContext`. The `FORCE` keyword is the crux: a plain RLS policy doesn't apply to the table owner, and `FORCE` makes it apply to everyone.

## The policy keys off the connection, not a row the app picks

The natural mistake is to let the application pass its tenant id into each query — which just re-creates the "remember to filter" problem one level down. Instead, the policy reads a per-connection session variable (a Postgres GUC) that the app sets *once* when the connection opens:

```sql
CREATE OR REPLACE FUNCTION current_tenant_id() RETURNS uuid AS $$
  SELECT NULLIF(current_setting('app.current_tenant', true), '')::uuid;
$$ LANGUAGE sql STABLE;

CREATE POLICY tenant_isolation ON <schema>.<table>
    USING      ("TenantId" = current_tenant_id() OR current_setting('app.bypass_rls', true) = 'on')
    WITH CHECK ("TenantId" = current_tenant_id() OR current_setting('app.bypass_rls', true) = 'on');
```

A `TenantGucConnectionInterceptor` — an EF `DbConnectionInterceptor` — sets `app.current_tenant` from the scoped `ITenantContext` on **every connection open**. The behavior when no tenant is resolved is the whole point: if `TenantId` is `Guid.Empty` and the caller isn't a trusted cross-tenant component, *nothing is set*. `current_tenant_id()` returns `NULL`, the policy matches no rows, and the query returns **zero rows — fail closed**. A bug doesn't leak data; it returns nothing, which is loud and safe instead of quiet and catastrophic. And because Npgsql resets session state when a connection returns to the pool, a tenant's GUC can never bleed into the next checkout.

## Why the app can't connect as the schema owner

This is the trap that quietly defeats most first attempts at RLS. The role that owns the schema, `nexus`, is `SUPERUSER` with `BYPASSRLS` — RLS is *silently ignored* for it. Connect your app as the owner and you'll write all the policies, run all the tests against a permissive connection, and ship something that enforces nothing.

So the apps connect as a separate, deliberately weaker role:

```sql
CREATE ROLE nexus_app LOGIN NOSUPERUSER NOBYPASSRLS INHERIT;
GRANT nexus TO nexus_app;   -- member of the owner → can run migrations/DDL on nexus-owned tables
```

`nexus_app` inherits the owner's table privileges, so migrations and CRUD still work. But `BYPASSRLS` is never inherited, and `nexus_app` isn't the table owner — so `FORCE RLS` applies to it in full. Migrations and DDL still run as `nexus`; every application connection string points at `nexus_app`. The split was verified empirically: as `nexus_app`, a set tenant GUC returns only that tenant's rows, no GUC returns zero rows, and owner-level DDL still runs.

## The escape hatch, and keeping it small

Some server-side work legitimately crosses tenants and has no request tenant in scope — a retention sweeper deleting across all tenants, a discovery scheduler walking every tenant's connections. They opt in explicitly by setting `ITenantContext.CrossTenant = true`, which makes the interceptor set `app.bypass_rls = 'on'`. The discipline around that flag is what keeps it from becoming the new hole:

- **Opt-in** — it defaults to `false`; HTTP request paths never set it.
- **Fail-closed** — an unset GUC is still zero rows, not all rows.
- **Server-only and audited** — only our code on `nexus_app` connections can set it, at a handful of reviewed call sites.

The design even prefers a stricter option where possible: consumers whose message already carries a `TenantId` *seed that tenant* rather than bypassing RLS at all. Bypass is reserved for genuine all-tenant sweeps and trusted by-id loads where the tenant simply isn't available.

## What we deliberately left out — and said so

Not every table belongs under RLS, and pretending otherwise creates breakage that looks like security. MassTransit saga state (`deploy_sagas`, `build_sagas`) is orchestration infrastructure accessed by `CorrelationId` — an unguessable GUID — in a consumer scope with no tenant GUC. Force RLS there and every saga read fails closed, breaking correlation. The `TenantId` on those rows is a routing/logging field, not a query boundary, and there's no endpoint that lists sagas across tenants — so leaving them out opens no hole. We wrote that reasoning into the docs rather than leaving the next engineer to discover the omission and assume it was a mistake.

We were equally explicit about the limit that *remains*: RLS scopes to whatever tenant the GUC carries, and the GUC derives from the gateway-stripped `X-Tenant-Id` header. An internal-network foothold that bypasses the gateway could still impersonate a single tenant. RLS shrank the blast radius from "any forgotten filter leaks everything" to "a network breach impersonates one tenant" — a real improvement, not a cure, and we documented it as such.

## Takeaways

- **Put the boundary where it can't be forgotten.** Application-layer filters are necessary but never sufficient; the layer that enforces isolation should be the one a future developer can't accidentally skip.
- **Fail closed.** An unresolved tenant returning zero rows turns a whole class of bugs from silent data leaks into obvious, safe failures.
- **Your app must not connect as the schema owner.** `BYPASSRLS` and `FORCE` are the difference between RLS that enforces and RLS that decorates.
- **Keep the escape hatch opt-in, server-only, and audited** — and prefer seeding the real tenant over bypassing whenever the tenant is known.
- **Document what you intentionally exclude.** An undocumented gap reads as a bug; a documented one reads as a decision.

The change shipped with the full backend and frontend suites green, five new interceptor tests, and a fresh-database docker stack proving all three layers live. The cross-tenant super-account is gone. The database now says no on its own.

---

*Originally published at bistecglobal.com*

**BistecGlobal** builds AI-native enterprise software. Follow us for engineering insights.
