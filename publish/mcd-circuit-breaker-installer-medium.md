# Circuit Breaker for AI Subprocesses: Cross-Platform Installer and Crash Recovery in MCD

*How we applied distributed-systems resilience patterns to an AI fleet orchestration platform — exponential backoff, circuit breakers, and a one-liner cross-platform installer.*

Running a fleet of AI agents simultaneously sounds resilient until one of them crashes in a tight loop at 2 AM, burning through resources and spamming your master channel with failure notifications. This is the problem we solved in [PR #81](https://github.com/chan4lk/claude-multi-channel-discord/pull/81) for MCD (Multi-Channel Discord), our AI fleet orchestration platform.

## The Problem: Fragile Subprocesses at Scale

MCD's architecture is straightforward: one Claude Code subprocess per Discord channel, managed by a central `ProjectPool` in `server.ts`. A Discord message arrives, the pool routes it to the right Claude process, Claude replies. When it works, it's seamless. When it doesn't, the failure modes are ugly.

Before this work, there were two compounding problems:

**Setup friction.** Every new machine required manually installing Bun, tmux, cloning the repo, configuring `.env`, and registering a system service. There was no one-liner install path, meaning every deployment was a multi-step artisanal process prone to variation.

**Crash loops.** If a Claude subprocess crashed repeatedly — due to a bad environment variable, a malformed prompt, an upstream API timeout, or any of a dozen other transient causes — MCD would respawn it immediately and unconditionally. No backoff, no limit. A pathologically failing project could thrash indefinitely, consuming system resources and flooding the operator with crash notifications.

## Solution Part 1: The Cross-Platform Installer

The installer lives in two files: `bin/install.sh` (273 lines, Linux/macOS) and `bin/install.ps1` (167 lines, Windows PowerShell 7+), with a 14-line `bin/mcd-setup.js` shim that lets operators run `npx mcd-setup` from anywhere.

The shell script handles the full lifecycle:

```bash
# Non-interactive mode — pass env vars to skip prompts
MCD_BOT_TOKEN=xxx MCD_USER_ID=yyy MCD_MASTER_CHANNEL=zzz bash install.sh
```

It detects the OS (`uname -s`), installs missing dependencies idempotently (bun via `bun.sh/install`, tmux via `apt-get` or `brew`), clones the repo, runs the interactive wizard if `.env` doesn't already exist, and registers a system service. The service registration adapts to platform:

- **Linux**: user-space `systemd` unit at `~/.config/systemd/user/mcd.service`, falling back to `/etc/systemd/system/mcd.service` with `sudo`
- **macOS**: `launchd` plist at `~/Library/LaunchAgents/com.github.chan4lk.mcd.plist`
- **Windows**: NSSM service (falling back to `sc.exe` if NSSM is absent)

The idempotency guarantee is deliberate. Re-running the installer on a machine that already has MCD installed is a no-op for each component already present — no double-registration, no config overwrites. This matters for scripted deployments and CI-driven updates.

## Solution Part 2: The FailureLedger Circuit Breaker

The more architecturally interesting change is in `src/project-pool.ts`. We added a `FailureLedger` per project:

```typescript
interface FailureLedger {
  count: number
  windowStart: number
  backoffMs: number
  circuitOpen: boolean
  circuitOpenAt?: number
}
```

Four constants govern the behavior (from `src/project-pool.ts`, commit `2a86715`):

```typescript
static readonly FAILURE_WINDOW_MS = 30 * 60_000       // 30 min
static readonly MAX_FAILURES_BEFORE_CIRCUIT = 5
static readonly CIRCUIT_RESET_MS = 10 * 60_000         // 10 min
static readonly BACKOFF_SEQUENCE_MS = [5_000, 10_000, 30_000, 120_000, 300_000]
```

When a subprocess exits with a non-zero code, `onExit` calls `recordFailureAndMaybeRespawn`. The logic:

1. Reset the count if the 30-minute window has elapsed (stale failures don't count).
2. Increment the count and look up the exponential backoff from the sequence: 5s → 10s → 30s → 2min → 5min.
3. Schedule a `setTimeout` for the respawn at the backoff delay.
4. If count reaches 5 within the window, open the circuit instead of scheduling another respawn.

```typescript
if (ledger.count >= ProjectPool.MAX_FAILURES_BEFORE_CIRCUIT) {
  ledger.circuitOpen = true
  ledger.circuitOpenAt = now
  this.fireEvent({ kind: 'circuit-open', chatId, slug: project.slug, failureCount: ledger.count })
  return
}
```

Once open, the `deliver` method checks the circuit before accepting any new messages:

```typescript
if (ledger?.circuitOpen) {
  const sinceOpen = this.now() - (ledger.circuitOpenAt ?? 0)
  if (sinceOpen < ProjectPool.CIRCUIT_RESET_MS) {
    process.stderr.write(`pool: circuit open for ${project.slug}, dropping message\n`)
    return
  }
  // Auto-reset after 10-minute clean window
  ledger.circuitOpen = false
  ledger.count = 0
  this.fireEvent({ kind: 'circuit-reset', chatId, slug: project.slug })
}
```

The circuit auto-resets after 10 minutes without manual intervention. This matters in production: an operator asleep at 2 AM doesn't need to manually clear a tripped circuit to restore a project once the underlying issue resolves.

## Dashboard Integration

Circuit state is visible in Mission Control (the MCD web UI). When the circuit opens, `server.ts` writes `circuit-state.json` in the state directory. `apps/mission-control/src/fleet-compute.ts` reads this file during fleet computation and injects `circuitOpen: true` into the per-project status. The `/api/fleet` endpoint (added in this PR) exposes `circuitOpen` per project, which the dashboard renders as a circuit-open badge next to the affected agent.

The state file approach was intentional: it decouples the server process from the dashboard without requiring a shared in-memory state or database. The file is a simple JSON map of `chatId → { circuitOpen, slug, ts }`, and `fleet-compute.ts` auto-expires entries older than 10 minutes to prevent stale badges when the server restarts.

## Test Coverage

The PR added 2 new test blocks (13–14) to `src/project-pool.test.ts`, with 7 individual assertions covering the full circuit lifecycle:

- **Test 13**: 5 consecutive crashes → 4 `respawn-scheduled` events with correct backoff delays, then 1 `circuit-open` event with `failureCount: 5`.
- **Test 14**: Message delivery while circuit is open → message dropped, no new spawn. Time advance past 10-minute reset window → `circuit-reset` event fires, next message spawns a fresh process.

The tests inject a mock `now()` function into `ProjectPool` to simulate time without actual sleeps, making the full circuit lifecycle testable in milliseconds.

## What Changed in Practice

Before this work, a misconfigured project could crash-loop indefinitely. Now it fails fast, backs off exponentially, trips the circuit after 5 strikes, and self-heals after a 10-minute quiet period. Operators see a circuit-open badge in Mission Control and can investigate without the system continuing to thrash.

The installer reduces a multi-step manual setup to a single `curl | bash` or `npx mcd-setup` invocation. Idempotency means it can be safely re-run — useful when updating an existing deployment without a full teardown-reinstall cycle.

Both changes reflect the same underlying principle: AI subprocesses are more failure-prone than typical web services, and the orchestration layer has to compensate with production-grade resilience patterns borrowed from distributed systems — exponential backoff, circuit breakers, and observable state.

---

*Originally published at bistecglobal.com*

---

**BistecGlobal** builds AI-native enterprise software. Follow us for engineering insights.
