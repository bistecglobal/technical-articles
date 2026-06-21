AI agents forget everything when a session ends. We fixed that by making memory the platform's job, not the agent's.

At BistecGlobal, we run autonomous Claude agents across 19+ projects. Each session is isolated — context window resets, agent starts cold. We had a MEMORY.md convention, but agents inconsistently wrote to it while juggling actual work.

Our solution: after every session kill, MCD automatically spawns a background `claude -p` process that distills the session into MEMORY.md — then exits. No blocking. No human intervention.

Key design choices:

• **Merge, not replace** — the distillation prompt explicitly preserves existing MEMORY.md content so early-session foundational notes survive later sessions
• **Fire-and-forget** — session kills complete immediately; distillation runs in background with 90s hard timeout + one retry
• **Opt-in per project** — `!project set <slug> --distill-on-stop` enables it only where persistent memory adds value
• **Audit trail** — `distillation_complete` events surface in the fleet dashboard alongside session starts and heartbeats
• **Dashboard visibility** — a 💭 chip on each project node shows MEMORY.md size; click to read or trigger on-demand distillation

The core implementation is 109 lines (src/distillation.ts). Zero external dependencies.

The lesson: competing priorities inside a session means memory gets skipped. Move memory responsibility to the platform — the agent works, the platform remembers.

Full article → [link placeholder — operator fills in after Medium publish]

#AI #MultiAgent #Claude #EnterpriseAI #SoftwareEngineering
