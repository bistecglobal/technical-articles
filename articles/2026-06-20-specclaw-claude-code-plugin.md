---
title: "Packaging a Workflow as a Claude Code Plugin: How Specclaw Went from Private Tool to Marketplace"
project: specclaw
tags: [AI, Claude, DevOps, Developer Tools, Plugin Architecture]
status: audited
date: 2026-06-20
---

# Packaging a Workflow as a Claude Code Plugin: How Specclaw Went from Private Tool to Marketplace

Software teams that work closely with AI agents often develop internal workflow tools — scripts, prompt files, and conventions that accumulate organically in a single repository. Eventually those tools become valuable enough to share, but extracting them into something distributable is non-trivial. This is the story of how Specclaw, a spec-driven development framework built at Bistec, made that transition — and what we learned about Claude Code's plugin system in the process.

## The Starting Point: A Monolithic Skill File

Specclaw began as a single `skill/SKILL.md` file containing 623 lines of instructions covering the entire `propose → plan → build → verify → pr` lifecycle. Every command — proposing a new change, planning its implementation, executing tasks, running verification, opening a pull request — lived in one document, loaded wholesale whenever any specclaw command was invoked.

This had a predictable problem: Claude received the entire workflow specification on every invocation, regardless of which verb was actually being used. The `/specclaw:build` command loaded the same context as `/specclaw:propose`, even though they are entirely different operations. The monolith also had no install story — teams had to clone the repo and configure it themselves.

## The Verb-per-Skill Decomposition (v0.1.0)

The v0.1.0 release restructured Specclaw as a proper Claude Code plugin. The monolithic `skill/SKILL.md` was decomposed into 15 individual skill files under `plugins/specclaw/skills/<verb>/SKILL.md` (the plugin has since grown to 17 as Azure Boards and spec-author skills were added in later releases):

```
archive/    auth-azdo/   auth-jira/   auto/    build/
init/       issue/       learn/       patterns/ plan/
pr/         propose/     status/      verify/
```

At v0.1.0, every skill had `disable-model-invocation: true` in its frontmatter so that specclaw verbs would only fire on explicit user invocation. That changed in v0.2.0: the flag was removed from 13 of 15 skills to allow Claude to route conversational utterances to the right verb automatically — "I have a proposal" fires `/specclaw:propose`, "let's plan the feature" fires `/specclaw:plan`. Only `auth-azdo` and `auth-jira` retain the flag, since credential commands should never trigger accidentally. The build skill's 151 lines of wave-loop orchestration are still not loaded during a propose, but that isolation is now achieved by the plugin system's per-skill loading rather than an invocation lock.

The 18 shell scripts that power the workflow were moved to `plugins/specclaw/bin/` and renamed from `specclaw-init.sh` to `specclaw-init` — proper PATH-resident executables. Scripts that need to locate plugin-internal resources use `$CLAUDE_PLUGIN_ROOT` with a `BASH_SOURCE`-based fallback: `PLUGIN_ROOT="${CLAUDE_PLUGIN_ROOT:-$(cd "$SCRIPT_DIR/.." && pwd)}"`. This means the plugin works whether Claude resolves the path via its marketplace install or a developer runs the script directly from a checkout. This matters: a script needs to find its sibling templates and references whether it was installed via the marketplace or run directly from a checkout.

The install experience collapsed to a single command:

```
/plugin marketplace add chan4lk/specclaw
```

## Host Repo vs. Plugin Dir: The Isolation Constraint

The central design constraint for any Claude Code plugin is that the plugin installation directory must remain clean — it is versioned and shared. All per-project state has to live in the consuming repository, not in the plugin itself.

Specclaw addresses this by writing all change artifacts to `.specclaw/` in the host repository's working directory. The `specclaw-init` script creates this directory structure on first use; every subsequent command reads and writes there. The plugin directory stays generic; the repo accumulates its own history.

This constraint shaped the auto-init feature in v0.2.1: rather than requiring explicit initialization before any other command, `specclaw-ensure-init` runs idempotently as the first step of every skill. If `.specclaw/` does not exist yet, it is created automatically using the current directory's basename as the project name. From the user's perspective, `/specclaw:propose` just works on a fresh repo.

## Per-Repo Learning: Knowledge That Grows in Place

The per-repo isolation constraint turned out to enable a meaningful feature. Because `.specclaw/` belongs to the host repo, it can accumulate knowledge that is specific to that codebase — patterns, lessons from past builds, and spec guidelines derived from real failures.

A subsequent release introduced a dedicated knowledge base at `.specclaw/knowledge/`:

- `agent-hints.md` — behavioral rules for coding agents, promoted from recurring patterns and build learnings
- `spec-guidelines.md` — specification standards derived from past spec gaps and design gaps

These files are injected into every coding agent's context payload by `specclaw-build-context`:

```bash
# From plugins/specclaw/bin/specclaw-build-context
KNOWLEDGE_HINTS_FILE="$SPECCLAW_DIR/knowledge/agent-hints.md"
KNOWLEDGE_SECTION=""
if [[ -f "$KNOWLEDGE_HINTS_FILE" ]]; then
  KNOWLEDGE_SECTION="## Repo Knowledge Base
$(cat "$KNOWLEDGE_HINTS_FILE")
"
fi
```

The learning pipeline works like this: when a coding agent encounters a problem, it logs it via `specclaw-log-learning`. If the same pattern recurs three or more times (tracked by `specclaw-detect-patterns`), the operator can promote it to the knowledge base with `specclaw-log-learning --promote <id>`. From that point, every future coding agent in that repository starts with that hard-won lesson already in context.

This is knowledge that belongs in the repo — not in the plugin, not in the agent's general training, but in the codebase where it was earned.

## Real Tracker Integrations: Azure Boards, GitHub, Jira

A workflow tool without issue tracker integration is an island. Specclaw v0.3.0 added symmetric Azure Boards support alongside the existing GitHub Issues and Jira integrations.

The implementation — 376 lines in `plugins/specclaw/bin/specclaw-azdo-issue` — targets the ADO REST Work Items API with `create`, `update`, `comment`, `close`, and `link-pr` subcommands. It reuses credentials from `/specclaw:auth-azdo` and is idempotent: re-running `create` for a change that already has a Work Item exits cleanly.

The integration hooks into every lifecycle phase. With `azdo.boards.sync: true` in `.specclaw/config.yaml`, the propose skill creates a Work Item, the build skill comments on task completion and failure, and the verify skill updates the Work Item on completion. The PR is automatically linked via ADO's ArtifactLink relations API.

One bug worth noting from the live smoke test against `BistecGlobal/Agent-Accelerator`: the `existing_wi_id()` function used a grep pipeline that, under `set -euo pipefail`, would silently exit when grep found no match (the expected case on first propose). The fix was appending `|| true` to the grep pipeline — a subtle bash idiom with `pipefail` that the agent logged as a high-priority build learning (L2 in `.specclaw/learnings.md`). That learning has since been promoted to the specclaw repo's own knowledge base, so future builds won't repeat it.

## The Coding Agent Context Payload

The architecture of `specclaw-build-context` illustrates how all these pieces come together. For each task during `/specclaw:build`, it assembles a complete prompt for a coding agent:

1. **Agent guardrails** — four behavioral rules vendored from Andrej Karpathy's CLAUDE.md, always injected
2. **Repo knowledge base** — accumulated hints from `.specclaw/knowledge/agent-hints.md`
3. **Specification** — the full `spec.md` for this change
4. **Design** — the full `design.md`
5. **Task** — the specific task (title, files, notes, wave, dependencies)
6. **Existing code** — current file contents, up to 500 lines each
7. **Error history** — last three failures for this task (if retrying)

Each agent gets exactly what it needs and nothing more. The guardrails and knowledge base are the only cross-task state; everything else is scoped to the specific change and task being implemented.

## What We Learned

**Decompose by invocation surface, not by logical module.** The right decomposition for a Claude Code plugin is not "group related commands together" — it is "give Claude only what it needs for the command being invoked." Verb-per-skill achieves this directly.

**Separate plugin state from repo state explicitly.** The plugin directory is versioned and shared; the repo directory is per-project and owned. Mixing them forces users to fork or patch the plugin to make project-specific changes. `$CLAUDE_PLUGIN_ROOT` for plugin resources, `.specclaw/` for project state, is a clean boundary.

**Let knowledge accumulate where it was earned.** The per-repo knowledge base is not a clever feature — it is a consequence of the isolation architecture. Once you accept that repo state lives in the repo, it follows that the best place for repo-specific agent hints is also the repo.

Specclaw is available at `chan4lk/specclaw` on the Claude Code marketplace. The current version is 0.4.2.
