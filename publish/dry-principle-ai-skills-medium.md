# DRY Principle for AI Skills: Extracting Shared Discipline Skills Across an Enterprise Agent Stack

*How Bistec eliminated duplicated coding standards across four Claude agent roles — and what every enterprise AI team can learn from the fix.*

When you run ten AI agents across an engineering organisation, you quickly discover a problem that every software architect recognises from the non-AI world: duplicated rules drift apart. At Bistec, four of our role-based Claude agents — developer, architect, QA engineer, and security reviewer — each carried their own copy of our .NET, TypeScript, and Node.js coding standards. By May 2026 those four copies had grown enough that keeping them in sync was a manual, error-prone chore. We applied the same fix software engineers have used for decades: we extracted the shared content into a single canonical source, and made every role skill point to it.

## The Problem: Four Copies, Zero Single Source of Truth

Our agent stack lives in the [ai-core repository](https://github.com/bistecglobal/ai-core). Each role skill is a Markdown file that Claude loads when it takes on that persona. The `bistec-developer` skill, for example, tells the agent how to branch, commit, write tests, and review code. But it also contained a full specification of our technology stack: Clean Architecture layout for .NET projects, naming conventions for C#, TypeScript strict-mode rules, React component patterns, Node.js module conventions, and a multi-section code review checklist.

Before commit `ca03393`, `skills/bistec-developer/SKILL.md` was 553 lines long. `bistec-architect` carried its own `.NET Defaults` and `Node.js Defaults` lists. `bistec-qa` had its own coding-convention section. `bistec-security-reviewer` restated the secure-coding rules that overlap with day-to-day engineering.

None of those four copies was wrong — they had been written intentionally, because each role genuinely needed those rules. But any update to a standard (say, adopting Pino instead of Winston for Node.js logging, or tightening the 30-line method cap) had to be applied in four places. Miss one, and the developer agent and the security reviewer would give contradictory guidance on the same pull request.

## The Solution: A Shared Discipline Skill

The fix, tracked as **TEC-41** and merged on 2026-05-02 (PR #3, branch `feature/TEC-41-bistec-coding-standards`), introduced a new layer in the agent stack: *shared discipline skills*.

A discipline skill is a standalone Claude skill that lives under `plugins/bistec-agent-stack/skills/` rather than inside any single role. It captures cross-cutting engineering knowledge that every role needs equally. The first discipline skill is `bistec-coding-standards` (`plugins/bistec-agent-stack/skills/bistec-coding-standards/SKILL.md`, v1.0.0, 296 lines).

The skill's own frontmatter states its intent directly:

```markdown
---
name: bistec-coding-standards
description: Canonical Bistec coding standards for .NET, TypeScript/React, and Node.js.
  Loaded by every Bistec role skill (developer, architect, qa, security-reviewer)
  so engineers, architects, and reviewers all measure code against the same rules.
version: 1.0.0
tags: [bistec, coding-standards, dotnet, typescript, react, node, shared]
---
```

## What the Shared Skill Contains

`bistec-coding-standards` is structured in three parts.

**Cross-cutting principles** apply regardless of language or framework: no secrets in code (all credentials go to Azure Key Vault), structured logging only (Serilog for .NET, Pino for Node.js, never `console.log` in production), fail loudly rather than swallowing exceptions, intent-revealing names (no `Manager`, `Helper`, `Util` unless genuinely cross-cutting), and no dead code. These eight rules had previously been restated implicitly across all four role skills; now they are stated once, explicitly.

**Per-language standards** cover the three stacks Bistec ships on:

- **.NET 8+** — Clean Architecture project layout (Domain, Application, Infrastructure, Api layers), C# naming conventions (PascalCase, `_camelCase` for private fields, `Async` suffix), code style (file-scoped namespaces, primary constructors, nullable reference types, 30-line method cap, 300-line class cap), error handling (Result pattern, global exception middleware), DI registration patterns, API conventions (Problem Details, FluentValidation, API versioning), and secure coding (parameterised queries, no `BinaryFormatter`, CSP headers, no PII in logs).

- **TypeScript / React** — Next.js App Router structure, functional components with named exports, TypeScript strict mode, no `any`, Zod schemas as the single source of truth for runtime and compile-time types, TanStack Query for server state, Zustand for client state, Tailwind + shadcn/ui for styling, and a prohibition on scattered `fetch` calls (all API requests go through a centralised client).

- **Node.js** — Same TypeScript strict mode, Zod at every external boundary, Pino for structured logging, Prisma or Drizzle for ORM access, ESM modules, and no floating promises.

**Cross-cutting code review checklist** closes the skill with five categories — correctness, security, performance, maintainability, and Bistec-specific items (structured logging, health-check endpoints, OpenAPI spec updates, migration inclusion). Every reviewer runs these same checks regardless of their role.

## How Role Skills Reference It

Each role skill that previously restated these conventions now carries a one-paragraph pointer instead. The top of `skills/bistec-developer/SKILL.md` v2.0.0 reads:

```markdown
## Coding standards

Load `@plugins/bistec-agent-stack/skills/bistec-coding-standards` for the
canonical Bistec coding standards (.NET, TypeScript/React, Node.js project
structure, naming, code style, error handling, DI, API conventions, and
secure coding rules). The rules apply unchanged in this role; everything
below adds developer-specific workflow, PR, testing, and review guidance on
top.
```

The `bistec-developer` skill then continues with content that is genuinely developer-specific: Conventional Commits format, PR templates, branching strategy, testing requirements (xUnit, Vitest, Playwright, k6), code review workflow, and handoff triggers to `bistec-qa` and `bistec-security-reviewer`. That is the content that belongs to the developer role and nowhere else.

`bistec-architect` v2.0.0 trimmed its per-language coding-convention sections down to a single paragraph describing *which* framework to choose and pointing downstream for the coding rules. `bistec-qa` v2.0.0 added the pointer and kept its testing-specific patterns (Arrange-Act-Assert, page objects, k6 scripts). `bistec-security-reviewer` v2.0.0 added the pointer and kept its OWASP/Azure review checklists.

## The Outcome

The extraction produced a direct, measurable result: `skills/bistec-developer/SKILL.md` dropped from **553 → 387 lines** — a 30% reduction, with 166 lines moved rather than deleted (commit `ca03393`). The developer skill was the primary beneficiary: it had carried the most inlined content. The other three role skills added pointer paragraphs in place of their smaller per-language sections, with modest line-count shifts reflecting how much duplication each had held. The content did not change; it moved to a place where a single edit propagates everywhere.

The skills/README.md migration log records the v1 → v2 transition explicitly: consumers pinned to `v1.x` of any role skill keep working until v1 is retired, because the rule surface area did not change. Upgrading to v2 means installing `bistec-coding-standards` alongside the updated role skill; together they contain exactly the same content as v1 alone.

The plugin manifest at `v0.2.0` now registers `bistec-coding-standards` as a first-class shared skill, making it discoverable and installable independently of any role skill.

## What This Pattern Gets Right

The discipline-skill layer solves a problem that is specific to AI agent stacks but structurally identical to the one solved by shared libraries in traditional software: *knowledge has a natural home, and that home should be unique*.

In conventional code, the DRY principle tells you to avoid copy-pasting implementation. In an AI skill stack, the analogous risk is copy-pasting *specification* — the rules an agent uses to evaluate and produce work. When a rule is stated in four skill files, the agent stack has four independent sources of truth for that rule. Updates require four coordinated edits. Any inconsistency between them results in agents that give contradictory guidance to the same team.

The shared discipline skill pattern addresses this by treating extracted cross-cutting knowledge the same way a well-designed library treats a shared utility: as a versioned, named artefact with a stable interface, loaded by consumers rather than restated by them.

The result is an agent stack where a coding standard, once updated in `bistec-coding-standards`, is immediately reflected in the developer, architect, QA, and security reviewer personas — without any of them needing to change.

---

*Implemented in [bistecglobal/ai-core](https://github.com/bistecglobal/ai-core), commit `ca03393`, merged 2026-05-02 (PR #3). Source: `plugins/bistec-agent-stack/skills/bistec-coding-standards/SKILL.md` v1.0.0; `skills/bistec-developer/SKILL.md` v2.0.0.*

---

*Originally published at bistecglobal.com*

---

**BistecGlobal** builds AI-native enterprise software. Follow us for engineering insights.
