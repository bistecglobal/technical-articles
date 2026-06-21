# Audit: spec-author-subagent

**Date:** 2026-06-21
**Verdict:** PASS
**Score:** 23/25

---

## Claim Inventory

| # | Claim | Evidence | Status |
|---|-------|----------|--------|
| 1 | Single-shot plan produced thin/vague specs, users rewrote by hand | `proposal.md`: "The result is often thin…Users end up rewriting the spec by hand" | ✅ |
| 2 | `spec-author` subagent shipped in commit `1179910` | `git log`: `1179910 feat: spec-author subagent + /specclaw:author-spec skill (#21)` | ✅ |
| 3 | Specclaw workflow: propose → plan → build → verify → publish | `plan/SKILL.md`, README | ✅ |
| 4 | `/specclaw:plan` generates `spec.md`, `design.md`, `tasks.md` | `plan/SKILL.md` step 4 | ✅ |
| 5 | Subagent defined in `plugins/specclaw/agents/spec-author.md` | File exists, `1179910` | ✅ |
| 6 | Dialogue order: Overview → FR → NFR → AC → Edge Cases → Deps → Notes | `spec-author.md` lines 22–31 (Dialogue Protocol) | ✅ |
| 7 | Agent doesn't advance until user confirms or says "skip" | `spec-author.md`: "Move to the next section only after the user explicitly confirms" | ✅ |
| 8 | Final summary + approval before writing file | `spec-author.md` Completion section lines 73–75 | ✅ |
| 9 | Mid-dialogue abandon = no partial spec.md written | `spec-author.md`: "If the dialogue is abandoned, no file is written." | ✅ |
| 10 | Technique table (5 Whys/JTBD/Inversion/Concrete-probe/Pre-mortem/MoSCoW) | `spec-author.md` Technique Catalog table | ✅ |
| 11 | "Challenge Mode" is mandatory, not optional | `spec-author.md` section header: "Challenge Mode (mandatory)" | ✅ |
| 12 | Vague terms ("fast", "easy", "secure", "scalable") rejected until measurable | `spec-author.md` Challenge Mode bullet 1 | ✅ |
| 13 | ACs must be observable: command output, file path, UI state, log line | `spec-author.md`: "observable from outside the system (a command, a file existing, a UI state, a log line)" | ✅ |
| 14 | After one round of pushback, accepts vague phrasing but logs to Notes | `spec-author.md`: "accept it but record the conversation in the section's Notes" | ✅ |
| 15 | Standalone skill at `plugins/specclaw/skills/author-spec/SKILL.md` | File exists, `1179910` | ✅ |
| 16 | Validates proposal.md, prompts to overwrite spec.md (default no) | `author-spec/SKILL.md` steps 1–2 | ✅ |
| 17 | Syncs status and GitHub after agent runs | `author-spec/SKILL.md` steps 4–5 | ✅ |
| 18 | `--author-spec` flag added to `plan/SKILL.md` | `plan/SKILL.md` Flags section | ✅ |
| 19 | With flag: delegates to spec-author, pauses, requires approval | `plan/SKILL.md`: "STOP and require explicit user approval ('approved', 'yes', 'go')" | ✅ |
| 20 | Rejection → no design.md / tasks.md created | `plan/SKILL.md`: "Do not generate the remaining files until the user approves" | ✅ |
| 21 | Without flag: single-shot, no dialogue (NFR3) | `spec.md` NFR3 + `plan/SKILL.md` FR7 | ✅ |
| 22 | `/specclaw:auto` unaffected (non-interactive) | `spec.md` NFR3 | ✅ |
| 23 | Agent runs Claude Opus — planning model in `config.yaml` | `spec-author.md` frontmatter `model: opus`; `.specclaw/config.yaml` line 12: `planning: "anthropic/claude-opus-4-6"` | ✅ |
| 24 | Self-test: AC3 invokes `/specclaw:author-spec spec-author-agent` | `spec.md` AC3 | ✅ |
| 25 | "Engineers would then spend more time rewriting the spec than it would have taken to write it by hand" | `proposal.md` says "users end up rewriting" but does not quantify relative time | ⚠️ weak — overstated |
| 26 | "The resulting spec (the file you're reading the sourced data from) was authored through this dialogue" | Unverifiable from git history alone | ⚠️ weak — remove |

---

## Forward-Looking Scan

No instances of "will", "plan to", "coming soon", "in the future", "next step", "roadmap", "soon" found. ✅

---

## Rubric

| Dimension | Score | Notes |
|-----------|-------|-------|
| Evidence quality (claims cited) | 5/5 | 24/26 claims fully verified with file/commit refs |
| Technical depth | 4/5 | Good technique catalog + dual-invocation design; could show agent dialogue excerpt |
| Clarity for target audience | 5/5 | Clean progression, table formatting effective |
| BistecGlobal voice | 4/5 | Professional, practitioner-focused throughout |
| Title specificity | 5/5 | Specific and descriptive |
| **Total** | **23/25** | |

---

## Fixes Required

1. **Claim 25** — "spend more time rewriting…than it would have taken to write by hand": soften to match `proposal.md` wording.
2. **Claim 26** — Remove "the file you're reading the sourced data from" (unverifiable); keep the self-test fact without the provenance claim.
