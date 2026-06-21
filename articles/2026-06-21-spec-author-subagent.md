---
title: "The Spec-Author Subagent: How Specclaw Turns a Proposal into a Testable Spec Through Socratic Dialogue"
project: specclaw
tags: [AI, Claude, Developer Tools, Software Engineering, Specification]
status: draft
date: 2026-06-21
---

# The Spec-Author Subagent: How Specclaw Turns a Proposal into a Testable Spec Through Socratic Dialogue

When AI agents write software, the weakest link is usually the specification. Not the code — the spec that drives it. At Bistec, we ran into this problem repeatedly with our Specclaw plugin: the AI would generate requirements in one shot from a short proposal, producing specs that were thin, vague, and riddled with untestable acceptance criteria. Engineers would then spend more time rewriting the spec than it would have taken to write it by hand.

We solved this by shipping a dedicated `spec-author` subagent (commit `1179910`) — a conversational AI agent that doesn't just fill in a template, but actively challenges requirements, names brainstorming techniques aloud, and refuses to write vague acceptance criteria into the final file.

## The Problem with Single-Shot Spec Generation

Specclaw is a Claude Code plugin that implements a structured software delivery workflow: propose → plan → build → verify → publish. The `/specclaw:plan` command is the engine of the plan phase — it reads a `proposal.md`, analyzes the codebase, and generates three files: `spec.md`, `design.md`, and `tasks.md`.

The single-shot approach works well for straightforward changes where the proposal is already detailed. But when a proposal is short — a common situation when an engineer is exploring a vague idea — the AI is left inferring requirements without dialogue. The outputs reflected that: acceptance criteria like "the system should work correctly," functional requirements that were really design decisions in disguise, and missing edge cases that only surfaced during implementation.

The files `spec.md`, `design.md`, and `tasks.md` chain together — the design derives from the spec, the tasks from the design. A weak spec propagates forward through every downstream artifact.

## The Spec-Author Subagent

The fix is a focused subagent defined in `plugins/specclaw/agents/spec-author.md` (`1179910`). Where `/specclaw:plan` is a batch process, `spec-author` is a blocking, conversational agent that walks the user through `templates/spec.md` one section at a time.

The dialogue follows a fixed order — Overview, Functional Requirements, Non-Functional Requirements, Acceptance Criteria, Edge Cases, Dependencies, Notes — and the agent does not advance to the next section until the user explicitly confirms the current one or says "skip." At the end, after all sections are confirmed, the agent presents a full summary and asks for final approval before writing the file. If the user abandons mid-dialogue, no partial `spec.md` is created.

### Named Techniques, Not Generic Questions

The agent doesn't just ask "does this look right?" It applies and names specific brainstorming and challenge techniques, matched to each section:

| Section | Technique |
|---------|-----------|
| Overview | **5 Whys** — drill from stated solution to root user/business need |
| Functional Requirements | **Jobs-to-be-Done** — "When [situation], I want to [motivation], so I can [outcome]" |
| Non-Functional Requirements | **Inversion** — "What would make this spec useless or actively bad?" |
| Acceptance Criteria | **Concrete-example probe** — if you can't give a worked example, the AC is too vague |
| Edge Cases | **Pre-mortem** — "Imagine we shipped this and it broke a week later. What broke?" |
| Scope additions | **MoSCoW** — Must / Should / Could / Won't categorization |

Naming the technique is not cosmetic — it lets the user know why they're being asked a seemingly strange question, and it makes the dialogue feel like a structured review session rather than an interrogation.

### Challenge Mode

The agent's system prompt (`plugins/specclaw/agents/spec-author.md`, `1179910`) explicitly defines a "challenge mode" that is mandatory, not optional. The agent must push back on:

- **Vague terms** — "fast", "easy", "secure", "scalable" are rejected until the user gives a measurable threshold (e.g., "p95 < 200ms").
- **Untestable acceptance criteria** — every AC must be observable from outside the system: a command output, a file path, a UI state, a log line. "Code is clean" is not an AC.
- **Solution-disguised-as-requirements** — if an FR prescribes implementation ("use Redis"), the agent asks whether that's a true requirement or a design choice that belongs in `design.md`.

If the user insists on vague phrasing after one round of pushback, the agent accepts it but records the ambiguity in the Notes section so the gap remains visible in the artifact.

## Two Invocation Paths

The feature ships with two ways to reach the agent.

**Standalone: `/specclaw:author-spec <change>`** — defined in `plugins/specclaw/skills/author-spec/SKILL.md` (`1179910`). It validates that `proposal.md` exists, checks whether `spec.md` already exists (prompting to overwrite if so, defaulting to no), invokes the agent, then syncs status and GitHub if configured.

**Integrated: `/specclaw:plan <change> --author-spec`** — the `--author-spec` flag was added to `plugins/specclaw/skills/plan/SKILL.md` (`1179910`). When present, `plan` delegates the spec step to `spec-author`, **pauses** after the agent writes `spec.md`, and requires explicit user approval ("approved", "yes", "go") before proceeding to generate `design.md` and `tasks.md`. If the user rejects the spec, neither downstream file is created.

The key design constraint: **without the flag, `/specclaw:plan` behaves exactly as before** — single-shot, no dialogue. This means `/specclaw:auto` (Specclaw's fully autonomous delivery mode) is unaffected. Interactive spec authoring is strictly opt-in; the automation path stays non-interactive.

## Model Choice

The agent runs on Claude Opus — the planning model configured in `config.yaml`. Spec authoring is a reasoning-heavy, dialogue-driven task where surface-level plausibility matters less than catching subtle logical gaps. Using the same model class as human-facing planning work is deliberate.

## The Self-Test

One of the acceptance criteria (`spec.md` AC3, `1179910`) is self-referential: `/specclaw:author-spec spec-author-agent` — running the new agent to author the spec for the change that introduced the agent itself. This served as both a smoke test and a demonstration that the agent could handle the edge case of being invoked on a change with an existing `proposal.md` but no prior `spec.md`. The resulting spec (the file you're reading the sourced data from) was authored through this dialogue.

## What Changed in Practice

The single-shot generation path is still there and still useful for well-specified proposals. What the `spec-author` subagent adds is an escape valve: when a proposal is exploratory, when requirements are genuinely unclear, or when the domain is new to the team, engineers can opt into a structured dialogue that surfaces gaps before code is written — not after.

Thin specs don't just produce bad software. They produce software that satisfies the spec while missing the point. By encoding the challenge techniques into the agent's system prompt and making vague acceptance criteria a hard stop, we moved the quality gate earlier in the delivery pipeline, where fixing a gap costs a conversation rather than a revert.

---

*Specclaw is an open-source Claude Code plugin. Source: [github.com/chan4lk/specclaw](https://github.com/chan4lk/specclaw). The `spec-author` subagent shipped in commit `1179910` (PR #21).*
