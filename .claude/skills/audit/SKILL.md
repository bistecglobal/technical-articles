---
description: Editorial audit of a draft article. Checks every factual claim is grounded in production code, removes speculation and forward-looking statements, flags missing citations, and scores the article on the BistecGlobal editorial rubric. Use as /bistec-articles:audit <slug>.
---

# bistec-articles audit

Audit a draft article for editorial compliance before publishing.

## Step 1 — Load the draft

Read `articles/<slug>.md`. If not found, list `articles/` and ask the operator to confirm the filename.

## Step 2 — Claim extraction

Parse every factual claim in the article — anything stated as true about the system, performance, outcomes, or design decisions. List them as a numbered inventory.

## Step 3 — Verify each claim

For each claim:
1. Identify the project and feature it refers to.
2. Find confirming evidence by reading the source: actual code files, merged PR details, commit history, or measured metric. Evidence is internal (for the audit record) — it does NOT need to appear in the article prose as commit SHAs or PR numbers.
3. Mark: ✅ verified | ⚠️ weak (indirect evidence) | ❌ unverified / speculative

If a claim is ❌: flag it for removal or rewrite using verifiable but reader-friendly language (e.g. "the scheduler module", "our metrics endpoint") — not raw SHAs.

## Step 4 — Forward-looking scan

Search the article for: "will", "plan to", "coming soon", "in the future", "next step", "roadmap", "soon". Flag every match — these must be removed or reframed as historical ("we later discovered…").

## Step 5 — Rubric score

Score the article on five dimensions (1–5 each):

| Dimension | Score | Notes |
|---|---|---|
| Evidence quality (claims verifiable, no raw SHAs in prose) | | |
| Technical depth | | |
| Clarity for target audience | | |
| BistecGlobal voice | | |
| Title specificity | | |

Total /25. Threshold to proceed to publish: **20/25**.

## Step 6 — Produce audit report

Write `articles/<slug>-audit.md` with:
- Summary verdict: PASS / NEEDS REVISION / FAIL
- Full claim inventory with verdicts
- Forward-looking flags
- Rubric scorecard
- Specific rewrite suggestions for each flagged item

## Step 7 — Apply fixes (if PASS or minor NEEDS REVISION)

If score ≥ 20 and fixes are minor (wording, citation adds):
- Apply them directly to `articles/<slug>.md`
- Update frontmatter `status: audited`
- Commit: `git commit -m "audit: fix claims and citations in <slug>"`

If score < 20: do not publish. Report to operator with the audit file.

Reply with verdict, score, and count of claims fixed vs flagged.
