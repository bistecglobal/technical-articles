# Audit — The Hot-Reload We Shipped Turned Off (hot-reload-we-shipped-turned-off)

**Verdict: PASS**
**Score: 25/25**
**Claims: 12 verified, 0 weak, 0 unverified · 0 forward-looking · 0 fixed**

Source: agent-nexus PR 244 (`ade2161`, on main), `src/Nexus.Forge/Wizard/*`, `src/Nexus.Orchestrator/Program.cs`, `docker-compose.dev.yml`, `.specclaw/changes/plugin-step-graph-hot-reload/design.md`, `tests/Nexus.Forge.Tests/Wizard/`.

## Claim inventory

| # | Claim | Evidence | Verdict |
|---|---|---|---|
| 1 | Wizard steps defined in `step-*.md` files; loader builds an in-memory graph at boot | `StepLoader.LoadAll(pluginRoot)`/`BuildGraph`, steps dir `.claude/skills/build-studio/references/steps` | ✅ |
| 2 | Original wiring: `StepGraph` registered as a singleton loaded once | pre-diff `AddSingleton<StepGraph>` in Forge Program.cs | ✅ |
| 3 | `IStepGraphProvider` interface with `Current` (read) + `Swap` | `IStepGraphProvider.cs` verbatim | ✅ |
| 4 | Atomic swap via `Interlocked.Exchange`; lock-free read via `Volatile.Read` | `StepGraphProvider` impl verbatim | ✅ |
| 5 | `StepGraph` is an immutable record → captured snapshot stays valid | `StepGraph.cs:7 public sealed record StepGraph(...)` | ✅ |
| 6 | DI flips: provider singleton, `StepGraph` becomes scoped factory → `provider.Current` | Forge Program.cs diff `AddScoped<StepGraph>` | ✅ |
| 7 | Scoped (not transient) so one scope resolves an identical instance | design.md "Turn-boundary correctness"; `AddScoped` | ✅ |
| 8 | `StepGraphReloadService` is a `BackgroundService` w/ FileSystemWatcher + polling fallback | `StepGraphReloadService.cs` class + `ExecuteAsync` poll loop + `TrySetUpWatcher` | ✅ |
| 9 | Polling exists because Docker bind mounts don't reliably emit inotify | class doc comment + WatchOptions.PollInterval doc | ✅ |
| 10 | Debounce collapses multi-event saves; keep-last-good retains graph on parse failure | `Trigger` linked-CTS debounce; `ReloadAsync` catch → return without Swap | ✅ |
| 11 | Cheap fingerprint = name+size+last-write per `step-*.md` | `ComputeSignature` verbatim | ✅ |
| 12 | `Enabled` defaults off (prod-safe); dev opts in via `Wizard__StepGraphWatch__Enabled=true`; structural mid-session edit corrupts persisted step index | `StepGraphWatchOptions` default false + doc; `docker-compose.dev.yml` both services set flag | ✅ |

Orchestrator mirrors the same wiring (Wave 35/T04). Tests present: `StepGraphProviderTests.cs`, `StepGraphReloadServiceTests.cs` incl. `Service_MalformedReload_KeepsLastGoodGraph`.

## Forward-looking scan

Clean. No "will / plan to / coming soon / roadmap / next step / soon" in prose. Article framed in past/present tense throughout.

## Rubric

| Dimension | Score | Notes |
|---|---|---|
| Evidence quality | 5 | Every claim maps to merged source; no SHAs/PR numbers in prose |
| Technical depth | 5 | Atomic-swap concurrency, scoped snapshot correctness, watcher/poll duality |
| Clarity | 5 | Strong hook, clean arc, code snippets under 30 lines |
| BistecGlobal voice | 5 | Practitioner-focused, honest "shipped it off" narrative |
| Title specificity | 5 | Specific, human, non-generic |

**Total: 25/25.**

## Notes

Honest-framing check passed: the article does NOT claim the feature is enabled for production users. It explicitly states the watcher is off by default in prod and enabled only in the dev compose — matching `StepGraphWatchOptions.Enabled=false` default and the dev-only env flag. The code ships in all environments; only the flag differs. Compliant with editorial rule against claiming aspirational/unshipped behavior.
