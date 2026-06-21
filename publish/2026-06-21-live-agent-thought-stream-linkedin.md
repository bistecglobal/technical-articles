When you run 10+ AI agents simultaneously, "busy" tells you nothing. We needed to see *what kind* of busy.

We built the Live Agent Thought Stream for MCD's mission-control dashboard: it tails Claude Code JSONL transcripts in real time and renders each tool call as a color-coded particle floating up from that project's node on the fleet graph.

**How it works:**
- SSE backend tails `.jsonl` transcripts per project, tracking line position to catch only new events
- `tool_use` blocks broadcast as `tool-event` SSE — internal Discord tools suppressed to cut noise
- FleetContext holds a 200-event ring buffer with monotonic IDs to prevent duplicate renders
- Particles are color-coded: cyan = file ops, amber = web, purple = sub-agent spawns
- Max 3 particles per node, 3-second lifetime — readable at fleet scale

**Key design insight:** Transcript files are append-only JSONL. Tailing by line count (not byte offset) handles file rotation cleanly — no inotify, no watchers, no infra.

**What operators actually see:** A cyan burst of Read/Grep = agent deep in research. Amber Fetch = web lookup. Purple Agent = complex multi-step delegation. Silence from a "busy" node = early stall signal.

Toggle with `T` on the fleet graph. Preference persists in localStorage.

Full article → [link placeholder — operator fills in after Medium publish]

#AIEngineering #Observability #React #TypeScript #DevOps
