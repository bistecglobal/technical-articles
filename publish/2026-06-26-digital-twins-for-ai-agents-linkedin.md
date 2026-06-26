Run twenty autonomous AI agents and they stop being individuals — they become a population. And populations have a useful property: members resemble each other. The agent burning context at 2am almost certainly has a twin three projects over that hit the same wall last week.

Mission Control now finds those twins automatically — a nearest-neighbour lookup for an entire agent fleet, built from logs we already keep.

How it works:

• Each agent is reduced to a 5-number behavioural fingerprint over 7 days: turns/day, tool-call rate, memory size, context pressure, avg tokens/turn — all reconstructed from JSONL transcripts
• Features are min-max normalised across the fleet first, so no single scale (tokens!) drowns out the rest
• Cosine similarity compares the *shape* of behaviour, not its magnitude; pairs ≥ 0.80 surface as twins
• Each match carries its reasoning — the shared dimensions — so a "0.94" reads as an explanation, not an oracle

Two lessons: the hard part of "find similar things" is feature design and normalisation, not the math. And a similarity score must carry its reasoning — a black box helps no one triage.

Full article → [link placeholder — operator fills in after Medium publish]

#AIAgents #Observability #MachineLearning #LLM #SoftwareEngineering
