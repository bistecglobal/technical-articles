---
description: Research a specific project feature or implementation and write a polished BistecGlobal technical article. Verifies the feature is merged and in production before writing. Outputs a draft in articles/YYYY-MM-DD-title.md. Use as /bistec-articles:write <project> <topic>.
---

# bistec-articles write

Write a technical article about an implemented, production feature.

## Step 1 — Verify it's real

Before writing a single word:

1. Identify the project path (see ARTICLE_IDEAS.md for paths).
2. Run `git -C <project-path> log --oneline --since="6 months ago" | grep -i "<topic>"` to confirm merged commits exist.
3. Check the feature is on the main/production branch — not just a PR or BACKLOG item.
4. If no evidence of implementation found: **stop**. Reply "Cannot write — feature not confirmed in production. Check ARTICLE_IDEAS.md for verified topics."

## Step 2 — Research

Deep-read the implementation:
- Relevant source files (read actual code, not just commit messages)
- Merged PRs: `gh -R <repo> pr list --state merged --search "<topic>" --limit 5`
- Commit history: `git -C <project-path> log --oneline -20`
- Any CLAUDE.md or README for context

Extract: the problem it solved, the approach taken, key design decisions, measurable outcomes.

## Step 3 — Outline

Draft a tight outline (H2 sections only):
- Problem / context
- Solution approach
- Architecture or implementation detail (with code snippet or diagram)
- Outcomes / lessons learned
- Conclusion

Review outline against the editorial rules in CLAUDE.md. Remove any forward-looking claims.

## Step 4 — Write

Write the full article to `articles/YYYY-MM-DD-<slug>.md`:
- Frontmatter: `title`, `project`, `tags`, `status: draft`, `date`
- Length: 800–1500 words
- Voice: BistecGlobal — professional, practitioner-focused, evidence-grounded
- Every factual claim must cite a commit SHA, file path, or PR number inline

## Step 5 — Self-review

Before committing, verify:
- [ ] No unimplemented features mentioned
- [ ] No forward-looking statements
- [ ] Every claim has a code/commit reference
- [ ] Title is specific and descriptive (not generic)
- [ ] Word count 800–1500

## Step 6 — Commit

```bash
git pull --ff-only origin main
git checkout -b article/<slug>
git add articles/YYYY-MM-DD-<slug>.md
git commit -m "article: <title>"
git push -u origin article/<slug>
```

Reply with the article title and a 2-sentence summary.
