When one AI agent spawns another, the delegation structure disappears into a flat transcript — and that structure is the most important thing about a multi-agent run.

We built a Mission Control view that reconstructs it. The `/agent-tree` page reads an agent's raw JSONL transcript and renders the full spawn hierarchy as a collapsible, color-coded tree.

How it works:

• Two-pass parse — first reassemble multi-step turns and index tool results by id, then extract every `Agent`/`Task` spawn into a node (label, prompt snippet, depth, children)
• First-level delegation is exact (straight from the parent's tool-use blocks); deeper nesting is best-effort, inferred from result text with a hard depth cap — and the UI never hides that limit
• Depth is encoded as color (cyan → purple → green → amber → red) so you read nesting level at a glance
• Zero new instrumentation — the transcripts Claude Code already writes contained the whole tree

The lesson: structure is a first-class observability signal, not a number you can aggregate. Shape is a topology — and the richest source for it was data we were already keeping.

Full article → [link placeholder — operator fills in after Medium publish]

#AIAgents #Observability #DeveloperTooling #LLM #SoftwareEngineering
