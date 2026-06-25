The most dangerous line in a multi-tenant system is the one deciding whether request A can touch tenant B's data. Ours was a string comparison — and it started rejecting legitimate requests with "tenant_mismatch."

The guard wasn't too loose. It was too literal.

Our identity provider (Zitadel) can spell one tenant id two ways: a GUID, or an int64 "snowflake." Same tenant, two representations. The body carried the snowflake; the JWT claim carried the GUID. string.Equals compared characters, saw a difference, and blocked the right request. A security control firing on false positives — which is exactly the pressure that gets someone to "loosen" it later and open a real hole.

The principle, in three parts:

• Identity is not the string you received — it's the canonical value it denotes. Normalize before you compare (we widen the snowflake into a GUID, then compare GUIDs).
• Normalize the SAME way everywhere. Our parser deliberately mirrors the gateway's. Two layers normalizing differently is a new vulnerability in the seam.
• Establish identity once, at the trust boundary. The edge gateway validates the token and injects a canonical X-Tenant-Id inward, so internal services never re-derive (and never re-normalize inconsistently).

Skip any one and your controls eventually let the wrong request in — or turn the right ones away.

Full article → [link placeholder — operator fills in after Medium publish]

#Security #MultiTenancy #SoftwareArchitecture #DotNet #IdentityManagement
