---
title: "The Hot-Reload We Shipped Turned Off"
project: agent-nexus
tags: [AI, DevOps, dotnet, developer-experience, architecture]
status: draft
date: 2026-07-02
---

Every editor save used to cost us a container restart.

The wizard at the heart of agent-nexus — the guided flow that walks a user through designing and building an agent — is defined not in code but in a folder of Markdown files. Each `step-*.md` file describes one step: its prompt, its ordering, its branching. At boot, a loader reads the folder, builds an in-memory graph, and hands it to every service that drives a build session. Clean, declarative, easy to reason about.

It also meant that changing a single word in a step description required tearing the process down and bringing it back up. For an engineer iterating on wizard copy, that was a restart per keystroke-worth-of-thought. And a restart doesn't just cost the ten seconds of downtime — it kills every in-flight build session on that instance. You fix a typo, three people lose their place.

So we built hot-reload. Then we shipped it turned off. Here's why both halves of that sentence are the right call.

## The problem with a graph loaded once

The original wiring was the obvious thing: register the step graph as a singleton, load it at startup, inject it everywhere.

```csharp
builder.Services.AddSingleton<StepGraph>(sp =>
{
    var steps = StepLoader.LoadAll(pluginRoot);
    return StepLoader.BuildGraph(steps);
});
```

A singleton loaded once is immutable for the life of the process — which is exactly the property you want for correctness and exactly the property that forces a restart to change anything. The graph and the process had the same lifetime, and there was no seam between them.

The fix is to put a seam there: a holder that consumers read through, whose contents can be swapped underneath them.

## An atomic swap behind an interface

We introduced `IStepGraphProvider` — a thin indirection between the loaded graph and everyone who consumes it.

```csharp
public interface IStepGraphProvider
{
    StepGraph Current { get; }   // atomic read; always a complete graph
    void Swap(StepGraph next);   // atomically publish a new graph
}

public sealed class StepGraphProvider : IStepGraphProvider
{
    private StepGraph _current;
    public StepGraph Current => Volatile.Read(ref _current);
    public void Swap(StepGraph next) => Interlocked.Exchange(ref _current, next);
}
```

The two primitives matter. `Interlocked.Exchange` publishes a fully-built graph in a single step, and `Volatile.Read` ensures a reader never sees a half-constructed one. Because the graph itself is an immutable record, a consumer that captures `Current` holds a snapshot that stays valid for its entire turn — even if a swap lands a millisecond later. No locks on the read path, no torn reads, no coordination between readers and the reloader.

The DI registration then flips: the provider becomes the singleton, and `StepGraph` becomes a *scoped* factory that resolves `provider.Current`.

```csharp
builder.Services.AddScoped<StepGraph>(sp =>
    sp.GetRequiredService<IStepGraphProvider>().Current);
```

Scoped, deliberately — not transient. If anything resolves the graph twice within one request or one dispatched message, it gets the identical instance. There is no window in which two different graphs coexist inside a single turn. Every existing consumer keeps its `StepGraph` constructor parameter untouched; they transparently pick up the latest graph at the *start* of each scope and never mid-flight.

## Watching files that don't announce themselves

Something has to notice the files changed and call `Swap`. That's a `BackgroundService` that watches the steps directory.

The naive approach — a `FileSystemWatcher` — works on a developer's laptop and fails in the exact environment we run in. Docker bind-mount volumes don't reliably emit inotify events, so a watcher alone would silently miss edits. The reload service runs both: a `FileSystemWatcher` for native events *and* a polling fallback that scans a cheap content fingerprint of the directory on an interval. The poll is the real trigger under Docker; the watcher is the fast path when the OS cooperates.

```
 step-*.md edits ──▶  StepGraphReloadService  (BackgroundService)
 (bind mount)         - FileSystemWatcher (fast path)
                      - polling fallback (real trigger under Docker)
                      - debounce (collapse multi-event saves)
                      - keep-last-good on parse failure
                             │  Swap(newGraph)   Interlocked.Exchange
                             ▼
                      StepGraphProvider ──▶ Current (Volatile.Read)
                             │
              WizardOrchestrator · endpoints · adapters  (scoped snapshot)
```

Two details earned their place through review. First, **debounce**: editors save in bursts (write, truncate, rename), so a linked cancellation token collapses a flurry of events into one reload after a quiet window. Second, **keep-last-good**: if the reloaded files fail to parse or build, the service logs the error and *retains the current graph*. A bad edit degrades to "your change didn't take," never to "the wizard is now broken." The provider is only swapped on a successful rebuild.

The fingerprint itself is deliberately cheap — each file's name, size, and last-write time, concatenated. A change to any of them flips the signature and triggers a reload; an unchanged directory produces a stable string and no spurious work.

## The decision that makes this an engineering story

Here is the part that separates a demo from a production feature: **the watcher is off by default.**

```csharp
public sealed class StepGraphWatchOptions
{
    // When false (default), the service loads once at startup and never watches.
    public bool Enabled { get; set; }
    public TimeSpan Debounce { get; set; } = TimeSpan.FromMilliseconds(500);
    public TimeSpan PollInterval { get; set; } = TimeSpan.FromSeconds(2);
}
```

Turn-boundary correctness guarantees that a reload never corrupts an *in-flight* turn — the snapshot is captured at the start of the scope. But it makes no promise about the turn *after*. A build session persists its position as a step id and index. If a structural edit — adding, removing, or reordering a step — lands between two turns of a live session, that persisted pointer can suddenly reference the wrong step, or one that no longer exists. Content edits are safe. Structural edits mid-session are not.

That risk is acceptable on a developer's machine, where the person editing the file is the same person driving the session and can just start over. It is not acceptable for a real user halfway through building an agent. So the safe default is *off*: in production the graph loads once at boot and never moves, exactly as before. The dev environment opts in explicitly through an environment variable in the dev compose file:

```yaml
# StepGraph hot-reload is opt-in (default off, prod-safe).
# Enable it for the dev loop so editing a step-*.md reloads without a restart.
- Wizard__StepGraphWatch__Enabled=true
```

The feature is fully shipped and fully wired in every environment. What differs is a single flag — and the reasoning behind where that flag points.

## What we took away

**Put the seam where the lifetime mismatch is.** The whole capability came from one insight: the graph and the process had been forced to share a lifetime. An interface with an atomic swap gave the graph its own, and every downstream service came along for free with zero changes to their signatures.

**Immutable snapshots make concurrency boring.** No read locks, no reader-writer coordination, no torn reads — because the thing being swapped can't be mutated and readers capture it by reference. The hard part of hot-reload is usually the concurrency; an immutable record erases most of it.

**A safe default is a design decision, not a cop-out.** It would have been easy to ship this on and call it a feature. Shipping it *off* — and being able to say precisely which class of edit is unsafe and why — is the difference between a convenience and a liability. The most useful thing we built here might be the flag that keeps it disabled.

The best production features sometimes look, from the outside, like nothing changed at all. In production, nothing did. In the dev loop, the restart-per-typo tax is gone — and that's where it was costing us.
