# Audit — Whose Cloud Does It Deploy To? Tenant-Scoped Deploy Credentials

**Verdict: PASS**
**Score: 24/25**

Evidence: PR 241 (commit `72e52be`, merged to `main`). Files: `src/Nexus.Shared/Platforms/FoundryDeployDescriptor.cs`, `src/Nexus.Forge/Endpoints/DeployEndpoints.cs`, `src/Nexus.Connectors/Endpoints/{ConnectionEndpoints,DeployCredentialEndpoints}.cs`, `Foundry/FoundryDeployVerifier.cs`, test files.

## Claim inventory

| # | Claim | Verdict | Evidence |
|---|---|---|---|
| 1 | Deploy endpoint previously read all Azure creds from IConfiguration env vars | ✅ | PR problem statement; `cfg[key]` fallback path in DeployEndpoints |
| 2 | Endpoint is `POST /api/build-sessions/{id}/deploy` | ✅ | DeployEndpoints.MapDeployEndpoints |
| 3 | Old behaviour = single global operator-owned deploy target | ✅ | PR problem statement |
| 4 | Invariant: vendor creds tenant-scoped in Infisical, not ambient config | ✅ | PR problem statement |
| 5 | Platform Connectors: save → live-verify → store per-tenant in Infisical | ✅ | ConnectionEndpoints + FoundryDeployVerifier |
| 6 | Infisical path `/{tenantId}/azure_ai_foundry/{connectionId}`, DB holds only SecretRef | ✅ | PR solution step 2 |
| 7 | Deploy fields added to existing azure_ai_foundry connection | ✅ | FoundryDeployDescriptor (shared platform id) |
| 8 | Single descriptor consumed by verifier, resolve endpoint, frontend | ✅ | FoundryDeployDescriptor doc comment |
| 9 | Two mutually-exclusive methods: Service Principal / api-key | ✅ | descriptor methods + fields |
| 10 | Classify → None on no fields (not error); reject both-methods | ✅ | Classify (descriptor) — quoted code matches |
| 11 | Field "present" only when non-blank | ✅ | HasAll/HasAny IsNullOrWhiteSpace |
| 12 | All-or-nothing: resolved connection used wholesale, no per-field env backfill | ✅ | Cred() helper — quoted code matches |
| 13 | Rationale: per-field mixing would shadow verified method | ✅ | code comment at Cred() |
| 14 | 404 / connectors unreachable → fall back to env vars (back-compat) | ✅ | comment + resolution logic |
| 15 | Fail closed (503) when connection exists but unresolvable | ✅ | `resolution.Unresolvable` → Results.Problem 503 — quoted |
| 16 | Concurrent deploys serialized via Postgres advisory lock | ✅ | SessionDeployLock.TryAcquireAsync |
| 17 | Idempotent deploy via artifact content hash + deployment record | ✅ | ComputeArtifactContentHash + ReadDeploymentRecordAsync |
| 18 | Eval gate (DEPLOY verdict) + stale-pass guard | ✅ | eval report check + latestArtifactAt guard |
| 19 | 33 passing tests (28 connectors + 5 forge) | ✅ | PR testing section + test files present |

19/19 verified. No ⚠️/❌.

## Forward-looking scan
No will / plan / coming soon / roadmap / next step describing unbuilt work. ("Will be used for OpenAI path" appears only inside a quoted code comment describing existing behaviour — not article prose.) Clean.

## Rubric

| Dimension | Score |
|---|---|
| Evidence quality | 5 |
| Technical depth | 5 |
| Clarity for audience | 5 |
| BistecGlobal voice | 5 |
| Title specificity | 4 |

**Total: 24/25** — exceeds 20 threshold. No fixes required.
