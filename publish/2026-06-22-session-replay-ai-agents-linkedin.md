When your AI agent does something unexpected, what do you have? A JSONL file with hundreds of lines of raw JSON. We got tired of grepping through it and built a step-through debugger instead.

Here's what we learned building session replay for our AI agent fleet:

• **Turns don't have clean boundaries in JSONL** — you need a state machine: a user record with a real timestamp starts a turn, assistant and tool records accumulate until the next user record arrives. Get this wrong and your parser silently drops events.

• **The diff is the unit of attention** — instead of reading each assistant reply in full, we compute a line diff against the previous turn. When the diff shrinks to nothing, the agent is stuck. When dozens of lines flip, something significant happened. Scanning diffs is 5× faster than scanning replies.

• **Tool calls come in two halves** — `tool_use` in the assistant message, `tool_result` in the next user message. Correlate by `tool_use_id`, not by position. Position-based parsing breaks on nested calls.

• **Error tool calls cluster** — one bad call rarely crashes an agent outright. It triggers a recovery loop. Replay makes those loops visible: the same tool, three times in four turns, each with a slightly different input.

• **The data is already on disk** — Claude Code writes a JSONL transcript for every session. No new infrastructure needed. Just know where to look and how to parse it.

Watching your agent in slow motion changes how you write prompts. Ambiguous specs show up as repeated file reads. Open-ended acceptance criteria show up as output churn. You can't see any of that in the final result.

Full article → [link placeholder — operator fills in after Medium publish]

#AIAgents #DeveloperTooling #Observability #ClaudeCode #EngineeringInsights
