# Audit: 2026-06-20-specclaw-claude-code-plugin

**Verdict: PASS (with minor fixes applied)**
**Score: 23/25**

---

## Claim Inventory

1. ✅ `skill/SKILL.md` containing 623 lines — confirmed by `2c80c88` diff stat (`skill/SKILL.md | 623 ---------------------`)
2. ✅ Commit `2c80c88` (May 15, 2026) restructured as a Claude Code plugin — confirmed by git log
3. ❌ "15 individual skill files" but code block lists 16 — at v0.1.0 the commit message says exactly 15; the code block was showing a mix of v0.1.0 and later additions (azdo-issue added v0.3.0, author-spec added PR #21). **Fixed: noted v0.1.0 had 15; current has 17.**
4. ❌ "Each skill has `disable-model-invocation: true`" — FALSE for current code. v0.1.0 set it on all 15; commit `c045d9d` removed it from 13/15 in v0.2.0. Only `auth-azdo` and `auth-jira` retain it. **Fixed: rewrote paragraph to describe the v0.1.0 → v0.2.0 evolution accurately.**
5. ⚠️ "Each script resolves plugin-internal resources via `$CLAUDE_PLUGIN_ROOT`" — grep shows only 4 scripts use it, with BASH_SOURCE fallback pattern `${CLAUDE_PLUGIN_ROOT:-$(cd "$SCRIPT_DIR/.." && pwd)}`. `specclaw-build-context` uses only BASH_SOURCE. "Each script" is an overstatement. **Fixed: changed to "scripts use `$CLAUDE_PLUGIN_ROOT` (with a BASH_SOURCE fallback)".**
6. ✅ 18 shell scripts at v0.1.0 — commit message confirms "18 executable bin/ scripts"
7. ✅ Scripts renamed from `.sh` to no extension — commit message: "dropped .sh, on PATH"
8. ✅ Auto-init in v0.2.1 commit `d5537a5` — confirmed by git log
9. ✅ `specclaw-ensure-init` first step of every skill — confirmed by build/SKILL.md and propose/SKILL.md
10. ✅ Commit `2d11406` (May 21, 2026) per-repo knowledge base — confirmed by git log
11. ✅ Knowledge files: `agent-hints.md` and `spec-guidelines.md` — confirmed by commit message and templates/
12. ✅ Code snippet from `specclaw-build-context` for knowledge injection — confirmed exact match lines 221-229
13. ✅ Pattern recurrence ≥ 3 triggers alert — confirmed by build/SKILL.md "If any pattern has recurrence ≥ 3, alert the user"
14. ✅ `specclaw-log-learning --promote <id>` — confirmed by commit message for `2d11406`
15. ✅ Specclaw v0.3.0 commit `aec77ba` Azure Boards support — confirmed by git log
16. ✅ 376 lines in `specclaw-azdo-issue` — confirmed by commit diff stat
17. ✅ create/update/comment/close/link-pr subcommands — confirmed by commit message
18. ✅ ADO ArtifactLink relations API — confirmed by commit message
19. ✅ grep/pipefail bug: `|| true` fix — confirmed by commit message with exact code shown
20. ✅ Logged as L2 high-priority — confirmed by commit message
21. ✅ Four behavioral rules from Andrej Karpathy's CLAUDE.md — confirmed by build/SKILL.md
22. ✅ MAX_FILE_LINES=500 — confirmed by `specclaw-build-context` line 13
23. ✅ Last three failures — confirmed by `specclaw-build-context` error history extraction
24. ✅ Current version 0.4.2 — confirmed by plugin.json

---

## Forward-Looking Scan

No instances of "will", "plan to", "coming soon", "in the future", "next step", "roadmap", "soon" found. ✅

---

## Rubric Scorecard

| Dimension | Score | Notes |
|---|---|---|
| Evidence quality (claims cited) | 4/5 | 3 errors found and fixed; remaining claims all have SHA/file refs |
| Technical depth | 5/5 | Code snippets, shell internals, bash pitfalls, context payload structure |
| Clarity for target audience | 5/5 | Developer-appropriate level, clear progression |
| BistecGlobal voice | 4/5 | Professional and practitioner-focused |
| Title specificity | 5/5 | Concrete and specific |

**Total: 23/25 → PASS**

---

## Fixes Applied

3 claims fixed directly in the article:
1. `disable-model-invocation` paragraph rewritten to show v0.1.0 vs v0.2.0 accurately
2. Skill count corrected (15 at v0.1.0, 17 now; code block clarified)
3. `$CLAUDE_PLUGIN_ROOT` claim narrowed to "scripts use... with a BASH_SOURCE fallback"
