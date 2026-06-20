---
description: Scan active Bistec projects for recently shipped features and generate article ideas grounded in production code. Appends new ideas to ARTICLE_IDEAS.md. Use as /bistec-articles:ideate [--project <slug>] to scan one project or all projects.
---

# bistec-articles ideate

Generate article ideas from what's actually shipped in production.

## Step 1 — Identify projects to scan

If `--project <slug>` passed: scan only that project.
Otherwise scan all projects listed in ARTICLE_IDEAS.md (read their local paths).

## Step 2 — For each project: mine recent work

```bash
git -C <project-path> log --oneline --since="90 days ago" --merges
git -C <project-path> log --oneline --since="90 days ago" --no-merges | head -40
```

Also check:
- Merged PRs: `gh -R <repo> pr list --state merged --limit 20 --json title,body,mergedAt`
- CLAUDE.md or README for project context

## Step 3 — Identify article candidates

For each significant commit cluster or merged feature, ask:
1. **Is this novel?** Does it solve a non-obvious problem or use an interesting approach?
2. **Is it generalisable?** Would other engineers find the pattern useful?
3. **Is it evidenced?** Can I point to specific files/commits?
4. **Is it production?** Merged to main, not a branch or BACKLOG item?

Only keep candidates that pass all 4 checks.

## Step 4 — Draft article ideas

For each candidate, write a short entry:
```
- *<Punchy title>* — <one sentence: what problem, what solution, why interesting> [commit: <SHA>]
```

## Step 5 — Deduplicate

Check ARTICLE_IDEAS.md for existing entries covering the same topic. Skip duplicates. Flag if an existing idea now has more evidence (update it).

## Step 6 — Append to ARTICLE_IDEAS.md

Add new ideas under the relevant project section. Add a `## New Ideas — <YYYY-MM-DD>` subsection if adding cross-project ideas.

```bash
git add ARTICLE_IDEAS.md
git commit -m "ideate: add <N> new article candidates from <project(s)>"
git push origin main
```

Reply with: count of new ideas added, top 3 most interesting candidates with one-line summaries.
