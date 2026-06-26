Your AI agent fleet's average response time is lying to you.

We watched a dashboard report "average: 41 seconds" while an operator swore one agent felt sluggish. The mean said all was well. The mean was smothering the tail — and the tail is where pain lives.

So we built per-project percentile latency into our Mission Control dashboard, reading the signal that was already on disk:

• Latency = user message → first assistant reply, parsed straight from Claude Code JSONL transcripts. No new instrumentation.
• Six percentiles (p10–p99) rendered as a box-and-whisker row per project, sorted worst-first by p90.
• Median colour-codes each agent: green <30s, amber <120s, red beyond.
• A 7-day-over-7-day p90 trend flags agents drifting the wrong way — tracked on the tail, not the mean.

The averages hid two failure modes: the snappy agent with an ugly p99, and the agent whose whole distribution quietly drifted right over two weeks. Both obvious in a box plot you read in two seconds.

The percentile is 40-year-old SRE wisdom. The only novelty is that the thing being measured is a language model in a loop. Audit your dashboards for averages — each one is a place a problem hides in plain sight.

Full article → [link placeholder — operator fills in after Medium publish]

#AIAgents #Observability #SRE #DevOps #LLMOps
