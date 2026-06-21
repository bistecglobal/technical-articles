Passing your acceptance criteria is not the same as writing good code.

In Specclaw's spec-driven pipeline, we added a second AI subagent that runs inside /specclaw:verify — not to re-check ACs, but to review *how* the code implements them. Two separate agents, two separate concerns.

What the code-reviewer subagent does:

• Reviews changed files across 10 dimensions: correctness, security, YAGNI, naming, complexity, test quality, design adherence, scope creep, dead code, one-liner opportunities
• Tags each finding 🔴 BLOCK / 🟡 WARN / 🟢 NOTE
• Produces a verdict in a separate review-report.md
• Optionally gates /specclaw:pr — CHANGES_REQUESTED blocks PR creation when code_review_block: true

Key design choices:
— Opt-in (code_review: false default) — existing projects unaffected
— Sonnet, not Opus — pattern-matching against a rubric, cost proportional
— Surgical scope — only changed code, not pre-existing issues in touched files
— Silent skip when disabled — zero latency, zero noise

Two narrow agents with clear contracts outperform one broad agent covering both jobs.

Merged to Specclaw main (commit f972cdc, PR #26). Enable with workflow.code_review: true in .specclaw/config.yaml.

Full article → [link placeholder — operator fills in after Medium publish]

#AI #CodeReview #DevTools #Claude #SpecDrivenDevelopment
