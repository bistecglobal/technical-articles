Almost every "Customer A can see Customer B's data" breach traces back to one query that forgot its tenant filter.

Our AI agent platform relied on the application *remembering* to scope every query by tenant — plus a super-account that could see all tenants. We moved the boundary out of the app and into the database with Postgres Row-Level Security:

• FORCE ROW LEVEL SECURITY on every tenant table — a raw query or reporting tool can't escape it, even one that never heard of our tenant context
• The policy reads a per-connection session variable set once on connection open — not a tenant id the app passes per query
• Unresolved tenant → zero rows. Fail closed: a bug returns nothing instead of leaking everything
• The app must NOT connect as the schema owner — owners have BYPASSRLS and RLS is silently ignored. A weaker nexus_app role makes FORCE actually enforce
• The cross-tenant escape hatch is opt-in, server-only, and audited — and we documented exactly which tables we left out and why

RLS shrank the blast radius from "any forgotten filter leaks everything" to "a network breach impersonates one tenant." A real improvement, documented honestly — not a magic cure.

Full article → [link placeholder — operator fills in after Medium publish]

#Security #MultiTenancy #PostgreSQL #SoftwareArchitecture #AI
