# Audit: 2026-06-22-browse-before-install-skill-catalog

**Verdict: PASS**
**Score: 22/25**
**Claims verified: 21/23 | Fixed: 1 | Flagged weak: 2**

---

## Claim Inventory

| # | Claim | Evidence | Verdict |
|---|-------|----------|---------|
| 1 | AI Core ships skills for finance analysts, DevOps engineers, Copilot Studio developers, security reviewers | `manifest.json` contains `@skills/bistec-devops`, `@skills/bistec-security-reviewer`; git log shows `8d69f94 feat: add 6 skills for Head of Operations and Head of Finance` | ✅ |
| 2 | Each skill declared in `manifest.json` mapping component IDs to names, descriptions, tags, versions | Direct read of `public/manifest.json` — structure confirmed | ✅ |
| 3 | Manifest served by Cloudflare Pages without auth | Footer in `catalog.html` line 179: "Powered by Cloudflare Pages"; `fetch('/manifest.json')` with no auth header | ✅ |
| 4 | Catalog is single static HTML file `public/catalog.html`, 294 lines, commit `d74ca76` | `git show d74ca76 --stat` confirms: `public/catalog.html | 294 ++++...` | ✅ |
| 5 | `fetch('/manifest.json')` call on page load | `catalog.html` line 186 confirmed | ✅ |
| 6 | `load()` code snippet | Copied verbatim from `catalog.html` lines 185–199 | ✅ |
| 7 | Manifest doesn't carry `category` field on most components | `grep "\"category\""` in `manifest.json` = 3 hits; total components = 45; code uses `c.category \|\| detectCategory()` | ✅ |
| 8 | `detectCategory()` infers categories from tags | Code copied verbatim from `catalog.html` lines 203–223 | ✅ |
| 9 | Categories auto-populated from distinct values in loaded data | `buildCategories()` at line 226: `[...new Set(allSkills.map(s => s.category))].sort()` | ✅ |
| 10 | New skill with new tags auto-extends filter UI | Follows from claim 9 by code design | ✅ |
| 11 | Skill card renders install command in dark code block | CSS `.install-row { background: #1a1f2e }` at line 109 | ✅ |
| 12 | Install command format described as `bistec install @skills/<id>` | **❌ INACCURATE** — actual code: `installCmd: 'bistec install ' + id` where id is full manifest key (e.g., `@skills/bistec-developer`, `@plugins/bistec-agent-stack`). The `<id>` placeholder implied user-supplied short name; format should be `bistec install <component-id>` | Fixed |
| 13 | Copy feedback resets after 2 seconds | `setTimeout(..., 2000)` at `catalog.html` line 280 | ✅ |
| 14 | Manifest public because CLI must fetch without credentials | Inferred from code design — CLI install script at `/install.sh` is public; reasonable but not explicitly documented | ⚠️ weak |
| 15 | CLI install: `curl -sL https://cli.bistecglobal.com/install.sh \| bash` | `catalog.html` line 178 confirmed exact string | ✅ |
| 16 | `esc()` uses throwaway `<div>` to escape HTML | `catalog.html` lines 284–287 confirmed | ✅ |
| 17 | No build step, no node_modules, deploys as-is to Cloudflare Pages | Verified from file structure and commit — only static files in `public/` | ✅ |
| 18 | Only 1 line changed in `index.html` | `git show d74ca76 -- public/index.html` shows 1-line diff | ✅ |
| 19 | `index.html` diff | Copied verbatim from git diff output | ✅ |
| 20 | Category filter makes skill library "immediately scannable" | Qualitative UX claim — not measured, but describes observable feature behavior | ⚠️ weak |
| 21 | Catalog stays in sync automatically at runtime | `fetch('/manifest.json')` on every page load — correct by code design | ✅ |
| 22 | Catalog is 294 lines | Confirmed by `git show d74ca76 --stat` | ✅ |
| 23 | Catalog live at `bistecglobal/ai-core` under `public/catalog.html` | File confirmed present | ✅ |

---

## Forward-Looking Scan

Searched for: will, plan to, coming soon, in the future, next step, roadmap, soon.

**0 matches.** Article is clean.

---

## Rubric Scorecard

| Dimension | Score | Notes |
|---|---|---|
| Evidence quality (claims cited) | 4/5 | 21/23 verified; 1 fixed; 2 weak but acceptable |
| Technical depth | 4/5 | Code snippets, design rationale, XSS-escaping note all present |
| Clarity for target audience | 5/5 | Well-structured, developer-appropriate |
| BistecGlobal voice | 4/5 | Professional, practitioner-focused throughout |
| Title specificity | 5/5 | Descriptive and precise |

**Total: 22/25 — PASS (threshold 20)**

---

## Fixes Applied

1. **Claim 12 — install command format**: Changed `bistec install @skills/<id>` to `bistec install <component-id>` with concrete example `bistec install @skills/bistec-developer`. The original wording implied a user-supplied short name; the actual id is the full manifest key including namespace prefix.
