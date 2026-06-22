# Audit: 2026-06-22-memory-similarity-jaccard-heatmap

**Verdict: PASS (minor fixes applied)**
**Score: 21/25**
**Claims: 19 total — 14 ✅ verified, 4 ⚠️ weak, 1 ❌ factually wrong (fixed)**

---

## Claim Inventory

| # | Claim | Status | Evidence |
|---|-------|--------|----------|
| 1 | Memory stored in per-project directories as markdown files | ✅ | `readMemoryFiles()` reads `~/.claude/projects/<encoded>/memory/*.md` |
| 2 | Each agent writes/reads its own memory independently | ✅ | Separate per-project memory dirs, no shared write path |
| 3 | "12+ projects each carrying 8–15 memory documents" | ❌ | Real fleet: 19 projects; checked 5 projects — 0–7 files each, not 8–15 |
| 4 | Jaccard formula J(A,B) = \|A∩B\| / \|A∪B\| | ✅ | Mathematically correct; matches `intersection.length / unionSize` |
| 5 | Keywords: lowercase, strip punctuation, remove stop words, min 4 chars | ✅ | `extractKeywords()`: `.toLowerCase()`, `.replace(/[^a-z0-9\s]/g,' ')`, `.filter(w => w.length >= 4 && !STOP_WORDS.has(w))` |
| 6 | Top-5 shared keywords by combined frequency | ✅ | `.sort((x,y) => (b.get(y)!+a.get(y)!) - (b.get(x)!+a.get(x)!)).slice(0,5)` |
| 7 | Result is N×N score matrix, 0–100% | ✅ | `scores: Record<string, Record<string, number>>`, `Math.round(score * 1000)/1000` × 100 in UI |
| 8 | Symlink resolution before memory path construction | ✅ | `fs.realpathSync(projectPath)` + `encodeProjectCwd(realPath)` |
| 9 | 5-minute in-process cache | ✅ | `CACHE_TTL_MS = 5 * 60 * 1000`, module-level `let cache` |
| 10 | "computation is linear in the number of project pairs" | ⚠️ | Technically true (O(N²) pairs × O(k) per pair) but phrasing implies O(N) scaling; misleading |
| 11 | Greedy cluster sort by average similarity | ✅ | `avgSim()` + `sort((a,b) => avgSim(b)-avgSim(a))` |
| 12 | Color ramp dark→cyan with power curve | ✅ | `Math.pow(score, 0.6)`, rgb(0,20,30)→rgb(0,220,240) |
| 13 | Threshold slider 0–50% | ✅ | `min={0} max={50}` range input |
| 14 | Cell size: 52px for ≤8 projects, 26px for 16+ | ✅ | `n <= 8 ? 52 : n <= 12 ? 40 : n <= 16 ? 32 : 26` |
| 15 | Tooltip: slugs, score%, top-5 shared keywords | ✅ | `setTooltip({a, b, score, shared})` + tooltip rendering confirmed |
| 16 | "keyflow and agent-nexus share 22%" (illustrative) | ⚠️ | Example value not verifiable from code; framed as hypothetical but could mislead |
| 17 | "18% similarity" and specific keywords in What We Discovered | ⚠️ | Observational claim from running the tool, not code-verifiable; 18% is precise without citation |
| 18 | Keyword maps built independently (no global state) | ✅ | Each slug processed in separate `for` loop before pairwise comparison |
| 19 | "The implementation is a few hundred lines" | ⚠️ | route.ts=178 lines, page.tsx=352 lines → 530 total; "few hundred" understates by ~2× |

---

## Forward-Looking Scan

Searched for: will, plan to, coming soon, in the future, next step, roadmap, soon.

**One borderline instance found:**

> "We haven't implemented incremental updates yet, but the data structure supports it."

This doesn't use the flagged keywords but implies future implementation. Rewording to keep the factual observation (not yet implemented) without implying roadmap intent.

---

## Rubric Scorecard

| Dimension | Score | Notes |
|---|---|---|
| Evidence quality (claims verifiable, no raw SHAs in prose) | 3/5 | Claim #3 (doc counts) factually wrong; Claims #17, #19 weak |
| Technical depth | 5/5 | Real code with algorithm explanation, cache design, path resolution detail |
| Clarity for target audience | 4/5 | Clear progression; "linear in pairs" phrase could confuse |
| BistecGlobal voice | 4/5 | Professional, practitioner-first, good hook |
| Title specificity | 5/5 | Names the technique (Jaccard) and artifact (heatmap) |

**Total: 21/25 → PASS**

---

## Fixes Applied

1. **Claim #3**: "12+ projects each carrying 8–15 memory documents" → "19 projects accumulating their own memory files" (verified count from live fleet; doc counts vary widely per project)
2. **Claim #10**: Removed "the computation is linear in the number of project pairs" — the efficiency rationale stands without the misleading complexity claim
3. **Claim #17**: Softened "18% similarity" to make clear these are example patterns observed when first using the tool, not verified measurements
4. **Claim #19**: "a few hundred lines" → removed (total is 530; the claim was imprecise)
5. **Forward-looking**: Rewrote "We haven't implemented incremental updates yet, but the data structure supports it" to state only the design fact, not the implication

All fixes are wording-level. Article remains structurally intact.
