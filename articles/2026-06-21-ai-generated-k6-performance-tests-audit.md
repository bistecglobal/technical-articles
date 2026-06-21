---
slug: ai-generated-k6-performance-tests
audited: 2026-06-21
verdict: PASS
score: 23/25
---

# Audit Report — AI-Generated Load Tests: Turning an API Contract into a Running k6 Script

## Summary Verdict

**PASS — 23/25**. Two weak anecdotal framing claims rewritten with design-intent grounding. All technical claims verified against `83da7fe`, `SKILL.md`, and related commits. No forward-looking statements. Minor fixes applied inline.

---

## Claim Inventory

| # | Claim | Evidence | Status |
|---|-------|----------|--------|
| 1 | BistecGlobal shipped a Claude Code skill — `k6-test-generator` | commit `83da7fe`, `skills/k6-test-generator/SKILL.md` | ✅ |
| 2 | Accepts 3 input modes: OpenAPI spec, URL patterns, plain-English description | `SKILL.md` "How It Works" — explicitly lists all three | ✅ |
| 3 | Output: script + test data file + run command + results guide | `SKILL.md` "Output Format" section — all four parts listed | ✅ |
| 4 | Commit `83da7fe` introduced it to `skills/k6-test-generator/SKILL.md`, registered in `public/manifest.json` | `git show 83da7fe --stat` confirms both files changed | ✅ |
| 5 | OpenAPI parsing: groups by resource, generates bodies from schema, CRUD flow, auth from `securityDefinitions`, skips deprecated | `SKILL.md` "OpenAPI Spec Parsing" lines 129–140 | ✅ |
| 6 | URL pattern parsing: identifies path params, infers methods, POST/PUT bodies, logical user flows | `SKILL.md` "URL Pattern Parsing" lines 143–150 | ✅ |
| 7 | Five standard load profiles: Smoke, Load, Stress, Spike, Soak | `SKILL.md` "Load Profiles" table | ✅ |
| 8 | Smoke: 1–5 VUs, 1 min | `SKILL.md` load profiles table | ✅ |
| 9 | Best practices as generation rules: SharedArray, check(), group(), sleep(1–3), correlation, `__ENV.BASE_URL` | `SKILL.md` "Best Practices to Apply" section | ✅ |
| 10 | Default thresholds: p(95)<500, p(99)<1500, error rate<0.01, throughput>100 | `SKILL.md` "Thresholds" section | ✅ |
| 11 | `bistec-qa` (commit `395e9d4`): xUnit, Playwright, k6, four-tier quality gates, DoD | `git show 395e9d4` commit message explicitly lists all | ✅ |
| 12 | `perf-tests` (commit `25cf302`): Playwright + k6 hybrid, 24 files, create/edit/validate modes | `git show 25cf302` — "self-contained bistec skill with 24 files across create/edit/validate modes" | ✅ |
| 13 | "QA engineers and backend developers frequently told us performance testing was perpetually deprioritized" | No commit/PR evidence — anecdotal; rewritten as design motivation | ⚠️ fixed |
| 14 | "Teams ran smoke tests manually, shipped to staging, and hoped nothing fell over under load" | Anecdotal generalization; rewritten as problem framing grounded in SKILL.md goal | ⚠️ fixed |
| 15 | Skill available across all Bistec projects that install ai-core plugin | `public/manifest.json` registration in `83da7fe` | ✅ |
| 16 | Thresholds gate test — exits non-zero code on failure, fails CI | Standard k6 behavior, factually correct | ✅ |

---

## Forward-Looking Scan

Searched for: "will", "plan to", "coming soon", "in the future", "next step", "roadmap", "soon"

**None found.** All statements are present-tense descriptions of shipped functionality.

---

## Rubric Scorecard

| Dimension | Score | Notes |
|---|---|---|
| Evidence quality (claims cited) | 4/5 | 14/16 claims fully verified; 2 anecdotal framing claims fixed |
| Technical depth | 5/5 | Covers input modes, load profiles, best practices, thresholds, ecosystem context |
| Clarity for target audience | 5/5 | Well-structured; practical examples; invocation patterns included |
| BistecGlobal voice | 4/5 | Professional, practitioner-focused; one passage slightly abstract ("adoption friction") |
| Title specificity | 5/5 | "Turning an API Contract into a Running k6 Script" — concrete and specific |
| **Total** | **23/25** | **PASS** |

---

## Fixes Applied

### Claim 13 — Anecdotal team feedback → design intent
**Before:** "Across Bistec's project portfolio — keyflow, specclaw, agent-nexus, and others — QA engineers and backend developers frequently told us the same thing: performance testing was perpetually deprioritized."

**After:** "The `k6-test-generator` skill (`skills/k6-test-generator/SKILL.md`) was designed around a documented problem: QA engineers, developers, and DevOps engineers need performance tests but lack the k6 depth to write them efficiently from scratch."

*Grounds the motivation in the skill's stated design goal rather than uncited team feedback.*

### Claim 14 — Anecdotal outcome → design problem statement
**Before:** "The result: teams ran smoke tests manually, shipped to staging, and hoped nothing fell over under load."

**After:** "The result, as framed in the skill's design intent: teams without k6 expertise either wrote ad-hoc scripts that missed k6 best practices, or skipped performance testing altogether."

*Removes unverifiable behavioral claim; keeps problem framing tied to the documented design rationale.*
