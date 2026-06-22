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

## Step 2 — Research the implementation

Deep-read the implementation:
- Relevant source files (read actual code, not just commit messages)
- Merged PRs: `gh -R <repo> pr list --state merged --search "<topic>" --limit 5`
- Commit history: `git -C <project-path> log --oneline -20`
- Any CLAUDE.md or README for context

Extract: the problem it solved, the approach taken, key design decisions, measurable outcomes.

Keep an internal record of supporting commits and file paths — these are for verification only and must NOT appear in the article text.

## Step 3 — Research trending article styles

Before outlining, search for 3–5 high-performing recent articles on the same general topic (AI agents, DevOps, developer tooling) on Medium and dev.to:

```
WebSearch: "trending medium articles AI agents engineering 2025"
WebSearch: "top dev.to articles developer tooling autonomous software 2025"
```

Fetch 2–3 of the top results and extract:
- **Hook pattern**: how does the opening paragraph grab attention?
- **Structure**: what H2 rhythm do high-performing articles use?
- **Voice**: conversational vs formal? First-person? Analogies?
- **Specificity style**: do they lead with numbers, drama, or a relatable problem?
- **Code/diagram balance**: how much code vs explanation?

Synthesise a style brief: 3–5 bullet notes on the conventions to adopt for this article.

## Step 4 — Outline

Draft a tight outline (H2 sections only), shaped by the style brief from Step 3:
- Hook / problem (opens with tension or a striking fact, not background)
- Solution approach
- Architecture or implementation detail (code snippet or ASCII diagram)
- Outcomes / what surprised us
- Lessons / takeaways readers can apply

Review outline against the editorial rules in CLAUDE.md. Remove any forward-looking claims.

## Step 5 — Write

Write the full article to `articles/YYYY-MM-DD-<slug>.md`:
- Frontmatter: `title`, `project`, `tags`, `status: draft`, `date`
- Length: 800–1500 words
- Voice: BistecGlobal — professional, practitioner-focused, evidence-grounded
- Style: apply the style brief from Step 3 — narrative arc, strong hook, human voice
- Every factual claim must be verifiable (use file names, module names, behaviour descriptions — NOT raw commit SHAs or PR numbers in body text)
- Code blocks: show real code from the implementation; keep snippets under 30 lines
- Do NOT embed commit SHAs or PR numbers in article prose; they belong only in the audit trail

## Step 6 — Self-review

Before committing, verify:
- [ ] No unimplemented features mentioned
- [ ] No forward-looking statements
- [ ] Every claim is verifiable via implementation research (no commit SHAs in prose)
- [ ] Opening paragraph hooks the reader within 2 sentences
- [ ] Title is specific and human (not generic, not "Building X with Y")
- [ ] Word count 800–1500

## Step 7 — Commit

```bash
git pull --ff-only origin main
git checkout -b article/<slug>
git add articles/YYYY-MM-DD-<slug>.md
git commit -m "article: <title>"
git push -u origin article/<slug>
```

Reply with the article title and a 2-sentence summary.
