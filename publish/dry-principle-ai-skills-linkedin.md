Four AI agents, four copies of the same coding standards — and every rule update had to be applied four times or your agents would contradict each other.

We fixed it the same way you'd fix it in code: extract once, reference everywhere.

**What we did:**
- Pulled all .NET, TypeScript/React, and Node.js standards out of four role skills (developer, architect, QA, security-reviewer) into one canonical `bistec-coding-standards` skill
- Each role skill now loads the shared skill instead of restating the rules
- The developer skill dropped 166 lines (553 → 387) — a 30% reduction. Content didn't change; it just has one home now.

**Why it matters for AI agent teams:**
- DRY applies to *specification*, not just *implementation*. When a rule lives in four skill files, you have four sources of truth.
- A single edit to `bistec-coding-standards` now propagates instantly to every agent persona that loads it.
- Versioned migration path: consumers on v1 keep working until v1 is retired — same pattern as a shared library release.

**The lesson:** AI skill stacks have the same maintenance problems as code. Treat agent knowledge like a dependency, not a copy-paste.

Full article → [link placeholder — operator fills in after Medium publish]

#AIEngineering #ClaudeAI #EnterpriseAI #SoftwareEngineering #DeveloperProductivity
