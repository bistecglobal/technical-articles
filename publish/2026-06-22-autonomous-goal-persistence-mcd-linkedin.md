# Autonomous Goal Persistence for AI Agent Fleets

When an AI agent restarts mid-task, it loses its immediate focus — even with memory files and system prompts intact. We solved this with a single file.

**The problem:** Our MCD platform runs 19 autonomous Claude agents, each managing a separate software project. Context resets (window exhaustion, watchdog kills, circuit breaker trips) restart sessions from scratch. Agents re-orient by re-reading the backlog — sometimes re-picking already-completed items.

**The fix — GOAL.md:**
- One plain-text file, max 500 characters, per project
- Set via Discord: `!project set <slug> --goal "..."`
- Injected as `<goal>text</goal>` into the *first* message of every new session — once, not repeated
- Zero schema, zero database, no config change needed

**Key design choices:**
- One-shot injection avoids per-turn token waste
- XML tag (`<goal>`) signals structured context, not a user command
- File-based persistence survives any process restart
- 500-char cap forces operator brevity — prevents it becoming a second system prompt

**Dashboard evolution:** From a static purple chip in the fleet grid (PR #83) → inline editor with active/paused/completed status (PR #89) → full `/goals` kanban board across all 19 projects (PR #93).

The lesson: the simplest mechanism capable of surviving a restart is often good enough. Write a file, read it on the next start, inject it once.

Full article → [link placeholder — operator fills in after Medium publish]

#AI #MultiAgent #Claude #SoftwareEngineering #DevOps
