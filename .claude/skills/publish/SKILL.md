---
description: Prepare a fully audited article for publishing. Formats it for Medium and LinkedIn, opens a GitHub issue tagged ready-for-publish with both versions attached, and merges the article branch to main. Use as /bistec-articles:publish <slug>. Requires audit status = PASS first.
---

# bistec-articles publish

Prepare an audited article for publishing on Medium and LinkedIn.

## Step 0 — Pre-flight checks

1. Read `articles/<slug>.md` frontmatter — confirm `status: audited`.
2. Check `articles/<slug>-audit.md` exists and verdict is PASS.
3. If either check fails: stop. Run `/bistec-articles:audit <slug>` first.

## Step 1 — Format for Medium

Create `publish/<slug>-medium.md`:
- Remove frontmatter
- Add Medium-style subtitle (italic, one sentence under the title)
- Convert code blocks to GitHub Gist-style (Medium renders fenced code well)
- Add canonical tags line at bottom: `*Originally published at bistecglobal.com*`
- Add author bio block: `**BistecGlobal** builds AI-native enterprise software. Follow us for engineering insights.`

## Step 2 — Format for LinkedIn

Create `publish/<slug>-linkedin.md`:
- Compress to 1200–1500 characters (LinkedIn optimal engagement length)
- Lead with a hook sentence (the core insight, not the title)
- 3–5 bullet points covering key takeaways
- End with: `Full article → [link placeholder — operator fills in after Medium publish]`
- Add 5 relevant hashtags

## Step 3 — Open GitHub issue

```bash
gh issue create \
  --title "Publish: <article title>" \
  --label "ready-for-publish" \
  --body "$(cat <<'EOF'
## Article ready for publishing

**Slug:** <slug>
**Audit:** PASS (<score>/25)

### Checklist for operator
- [ ] Post Medium version (`publish/<slug>-medium.md`) to BistecGlobal Medium publication
- [ ] Copy Medium URL and paste into LinkedIn version (`publish/<slug>-linkedin.md`) link placeholder
- [ ] Post LinkedIn version to BistecGlobal LinkedIn page
- [ ] Close this issue once both are live

### Files
- Draft: `articles/<slug>.md`
- Medium: `publish/<slug>-medium.md`
- LinkedIn: `publish/<slug>-linkedin.md`
- Audit: `articles/<slug>-audit.md`
EOF
)"
```

## Step 4 — Merge branch

```bash
git checkout main
git pull --ff-only origin main
git merge --no-ff article/<slug> -m "publish: merge article <slug>"
git push origin main
gh pr merge article/<slug> --merge --delete-branch 2>/dev/null || true
```

## Step 5 — Update ARTICLE_IDEAS.md

Find the entry for this article topic in `ARTICLE_IDEAS.md` and mark it published:
- Add `✅ Published` tag to the bullet
- Add the GitHub issue number as a reference

Commit: `git commit -m "tracking: mark <slug> as published"`

Reply with the GitHub issue URL and a one-line summary of what to post where.
