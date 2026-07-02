Give an AI agent a persistent memory and it starts writing things down. Give it a whole fleet of memories and a quieter problem shows up: notes that no longer connect to anything — still on disk, still loaded into context, pointing to nothing and cited by nothing.

We built a mission-control page that treats agent memory as a graph and hunts for the disconnected islands.

Here's what made it work:

🔗 **Orphan = structurally isolated, not old.** A note with zero inbound and zero outbound `[[links]]`. A brand-new stranded fact is more suspicious than a two-month-old one that half the memory points at.

🧭 **The graph is rebuilt on every scan.** Walk each project's memory dir, extract wiki-links, count in/out degree. Cheap enough to run on demand — no new database, no background job.

⏸️ **Detection never deletes.** It tracks each orphan across a lifecycle: flagged → relinked, deleted, or ignored. The system points at a problem; humans and agents resolve it.

📊 **The resolution pattern is the real signal.** Mostly "relinked"? The memory is healing. Mostly "deleted"? It was accumulating junk.

The broader lesson: when you build observability for an autonomous system, the temptation is to auto-remediate. But often the most useful thing a detector can do is *hold* a finding long enough to see how it actually gets resolved.

Full article → [link placeholder — operator fills in after Medium publish]

#AIAgents #SoftwareEngineering #Observability #KnowledgeManagement #DevOps
