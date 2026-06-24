"It was fine an hour ago." A live dashboard can't prove that — it only shows what *is*, never what *was*.

Our AI agent fleet's mission control had this gap. We could see the current breakdown — idle, active, stalled, autonomous — but couldn't drag backward to the moment three agents went stalled at once. The data existed: we were already writing labeled fleet snapshots to a table. We just had no way to play them back.

So we built a Snapshot Scrubber — a VCR transport for fleet history. One client page, no new backend.

What makes it a tool, not a demo:

• Drag the slider to re-render the fleet as it was at any instant; Play auto-advances one snapshot/sec
• Live data arriving underneath you? The selection snaps to newest on every count change — no silently jumping to the wrong frame
• Play stops at the present instead of looping — it ends where reality is, like an instrument not a screensaver
• A malformed snapshot degrades to zeros instead of crashing the timeline — because the corrupt one is often right next to the incident
• Built by reusing the snapshot data already captured for a state-diff feature — recognising an existing source answered a new question

A live dashboard answers "what is happening." A scrubber answers "what happened, and when did it turn."

Full article → [link placeholder — operator fills in after Medium publish]

#AI #DevOps #Observability #AIAgents #SoftwareEngineering
