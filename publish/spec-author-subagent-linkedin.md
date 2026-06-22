Single-shot AI spec generation produces thin specs. Here's how we fixed it.

Our Specclaw plugin was generating `spec.md` from proposals in one pass — vague acceptance criteria, missing edge cases, requirements that were really design decisions. Engineers rewrote them by hand. We shipped a `spec-author` subagent to fix this at the source.

Key design decisions:

- **Section-by-section dialogue** — the agent walks through Overview → FRs → NFRs → ACs → Edge Cases one at a time, waiting for explicit confirmation before moving on
- **Named brainstorming techniques** — 5 Whys for the overview, Jobs-to-be-Done for FRs, Pre-mortem for edge cases. Named aloud, not applied robotically
- **Mandatory challenge mode** — vague terms ("fast", "scalable") are rejected until the user gives a measurable threshold. Untestable ACs don't get written to the file
- **Opt-in, not default** — `/specclaw:plan --author-spec` triggers the dialogue; without the flag, plan runs as before so automation stays non-interactive
- **No partial writes** — if the user abandons mid-dialogue, no file is written

The quality gate moved from "during build" to "before spec is approved." A gap caught in a 10-minute dialogue is cheaper than a revert after implementation.

Full article → [link placeholder — operator fills in after Medium publish]

#AI #SoftwareEngineering #DeveloperTools #ClaudeCode #EngineeringExcellence
