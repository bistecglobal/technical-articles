# The Code-Reviewer Subagent: Wiring Automated Code Review into a Spec-Driven Pipeline

*How Specclaw adds a second AI agent to the verify step — one that checks code quality independently from acceptance criteria, with BLOCK/WARN/NOTE severity tagging and an optional PR gate.*

---

When a spec-driven pipeline verifies a change, it checks whether the implementation satisfies its acceptance criteria. That is a meaningful gate — but it is not the same as a code review. Acceptance criteria tell you *what* the code does. A code reviewer asks *how well* it does it: are there correctness bugs, security holes, YAGNI violations, dead code? These are orthogonal concerns, and treating them as one causes the familiar situation where a feature passes every AC and still ships with a subtle off-by-one or a speculative abstraction that nobody asked for.

In Specclaw, the verify step (`/specclaw:verify`) already spawns a verification subagent that evaluates each acceptance criterion. Shipping the code-reviewer subagent adds a second, independent agent to that same step — one that reviews the changed files across 10 quality dimensions and produces a structured verdict. The two agents run sequentially: the verifier answers "did we build the right thing?"; the reviewer answers "did we build it well?"

## The Problem With Combining AC Checks and Code Quality

Acceptance criteria are deliberately coarse. They describe observable behaviour, not internal structure. "Given `workflow.code_review: false`, verify behaviour is identical to pre-change" is a valid AC. It says nothing about whether the implementation is 80 lines when 8 would do, or whether it imports a module it never uses.

The alternative — embedding code quality checks into the verification agent — is tempting but wrong. A verification agent that tries to evaluate both ACs and code quality produces noisy, unfocused reports. Worse, the two concerns pull the agent's attention in opposite directions: the verifier needs to reason about behaviour contracts; the reviewer needs to look at implementation structure. Conflating them makes each weaker.

The cleaner design is a separate agent with a narrow job.

## Implementation

The code-reviewer agent is defined at `plugins/specclaw/agents/code-reviewer.md` (commit `f972cdc`, PR [#26](https://github.com/chan4lk/specclaw/pull/26)). Its frontmatter declares it as a Sonnet-class agent with access to Read, Write, and Bash:

```yaml
---
name: code-reviewer
description: Reviews changed files for a specclaw change across 10 quality dimensions.
tools: [Read, Write, Bash]
model: sonnet
---
```

The agent receives four inputs when invoked: the content of every changed file, the change's `spec.md`, `design.md` (if it exists), and `tasks.md`. It reviews across exactly 10 dimensions:

| # | Dimension | Focus |
|---|-----------|-------|
| 1 | Correctness | Logic errors, off-by-one, null dereference |
| 2 | Security | Injection, hardcoded secrets, missing auth checks |
| 3 | YAGNI / Simplicity | Speculative abstractions, unused parameters |
| 4 | One-liner opportunities | Multi-line blocks with idiomatic one-liner equivalents |
| 5 | Naming | Misleading names, inconsistency with existing conventions |
| 6 | Complexity | Functions > 30 lines, nesting depth > 3 |
| 7 | Test quality | Missing tests for changed paths, mirror-implementation assertions |
| 8 | Design adherence | Divergence from `design.md` without justification |
| 9 | Scope creep | Files modified outside `files:` lists in `tasks.md` |
| 10 | Dead code | Functions, variables, or imports added but never referenced |

Every finding is tagged with one of three severity levels: `🔴 BLOCK` (must fix before PR), `🟡 WARN` (should fix), or `🟢 NOTE` (optional improvement). The agent assigns a verdict after all 10 dimensions: **APPROVED** if no BLOCKs, **CHANGES_REQUESTED** if any BLOCKs, **APPROVED_WITH_NOTES** if no BLOCKs but WARNs or NOTEs exist.

## Wiring It Into Verify

The integration point is a new Step 3.5 in `plugins/specclaw/skills/verify/SKILL.md`. It sits between the existing verification agent (Step 3) and report saving (Step 4):

```
/specclaw:verify
  │
  ├── Step 1–3: collect evidence → verify agent → write verify-report.md
  │
  ├── Step 3.5 (conditional):
  │   ├── read workflow.code_review from config.yaml
  │   ├── if false → skip entirely (silent, no output, no error)
  │   └── if true:
  │       ├── read design.md (empty string if absent)
  │       ├── read tasks.md (empty string if absent)
  │       ├── spawn code-reviewer agent
  │       ├── write output → review-report.md
  │       └── append one-line verdict to verify-report.md
  │
  └── Steps 4–7: save report, update status, sync, notify
```

The conditional skip is intentional: the default value of `workflow.code_review` is `false` in `plugins/specclaw/templates/config.yaml`. Projects that have not opted in see no behavioural change — no extra latency, no additional output.

When enabled, the reviewer's findings land in a separate file, `.specclaw/changes/<change>/review-report.md`. The verify-report gets one appended line:

```
**Code Review:** APPROVED_WITH_NOTES — 3 findings: 0 BLOCK, 1 WARN, 2 NOTE
```

This keeps the two reports separated — the verifier's AC evaluation and the reviewer's code quality assessment stay in their own files — while making the review verdict visible in the primary summary.

## The PR Gate

A second flag, `workflow.code_review_block`, adds a hard gate to `/specclaw:pr`. When set to `true`, the PR skill reads `review-report.md` before creating the PR. If the verdict is `CHANGES_REQUESTED`, it aborts and prints all BLOCK findings:

```
PR blocked: code review verdict is CHANGES_REQUESTED.
Fix BLOCK findings in review-report.md then re-run /specclaw:verify.

[BLOCK] src/auth-middleware.ts:17 — Correctness
Problem: token expiry check uses `<` instead of `<=`, allowing tokens that
         expired exactly at the boundary to pass validation.
Fix: Change `expiresAt < now` to `expiresAt <= now`.
```

Both flags default to `false`. Teams can enable the reviewer without the PR block, treating it as advisory feedback before they decide it's ready to gate merges.

## Design Decisions Worth Noting

**Model choice.** The agent is configured for `models.review` (Sonnet), not Opus. Code review is pattern-matching against a fixed rubric, not open-ended synthesis. Sonnet handles it well, and the cost stays proportional to the task. (AC from spec NFR2: "keeping cost proportional to a pattern-matching task.")

**Surgical review, not sweep.** Guardrail Rule 3 ("Surgical Changes") applies to the reviewer's findings: it addresses only the changed code, not pre-existing issues in files it reads. This prevents the report from becoming a global code audit for every verify run, which would bury actionable findings in noise.

**Graceful dimension skipping.** When `design.md` is absent (common for small fixes that skipped the design phase), Dimension 8 (Design adherence) is silently skipped with a note. When `tasks.md` has no `files:` declarations, Dimension 9 (Scope creep) is skipped. The report still produces a verdict on the 8–9 applicable dimensions.

**Separate report files.** An earlier design option merged reviewer output into `verify-report.md`. The resolved design keeps them separate: the verify-report is the record of AC satisfaction; the review-report is the record of code quality. They answer different questions and are consumed by different downstream steps (the PR body draws from verify-report; the `code_review_block` check reads review-report directly).

## What It Looks Like in Practice

A typical flow for a project with `workflow.code_review: true`:

1. `/specclaw:build` implements the change across one or more tasks.
2. `/specclaw:verify` runs the test/lint/build commands, evaluates each AC, and — via Step 3.5 — spawns the code-reviewer subagent on the changed files.
3. The developer reads `review-report.md` and sees, for example, that a helper function introduced in the change is never called (dead code, WARN) and that a conditional has an off-by-one on the boundary case (correctness, BLOCK).
4. They address the BLOCK, re-run `/specclaw:verify`, and proceed to `/specclaw:pr`.

If `code_review_block: true`, the PR would not have been creatable until the BLOCK was resolved. With the default `false`, it is advisory — the developer sees the finding and decides how to handle it.

## Lessons from the Integration

**Composing agents sequentially is simpler than combining them.** The temptation to add code quality checks to the existing verify agent would have saved one agent invocation. The resulting complexity — mixed concerns, diluted attention, harder-to-parse reports — was not worth it. Two narrow agents with clear contracts outperform one broad agent trying to cover both jobs.

**The opt-in default matters.** Releasing the reviewer as opt-in (`code_review: false`) meant existing projects were unaffected on upgrade. Teams can adopt it incrementally: reviewer on, block off first; reviewer on, block on once they trust the rubric.

**Silently skipping is the right default for conditional steps.** Step 3.5 produces no output and no error when disabled. This means the verify skill's observable behaviour is identical to before the change for any project that has not set the flag — an important property when modifying a skill that runs in every automated pipeline.

The code-reviewer subagent is merged to the Specclaw main branch (commit `f972cdc`, PR [#26](https://github.com/chan4lk/specclaw/pull/26)) and available in the current plugin release. Enable it with `workflow.code_review: true` in your `.specclaw/config.yaml`.

---

*Originally published at bistecglobal.com*

**BistecGlobal** builds AI-native enterprise software. Follow us for engineering insights.
