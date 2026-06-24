Two features watching the same AI fleet quietly disagreed about its health — a bug you can't catch in any single screenshot.

Our Mission Control dashboard had a Fleet Advisor (actionable cards) and a Fleet Brief (narrative digest). Same reality, two voices. But each had grown its own copy of the detection rules — and the copies had drifted. The Advisor flagged stalls at 30min–4h; the Brief at 4–48h. One knew about thrashing; the other knew about circuit breakers. Two answers to one question.

We collapsed them into a single rule engine. The lesson was about shape, not line count:

🔹 **Name the fact, not the card.** The duplication came from computing *advisor cards* and *brief messages* instead of a neutral `Finding` (severity, signal, both phrasings, optional action). Once the fact had a type, both surfaces became thin adapters — and adapters don't drift.

🔹 **One registry.** Every rule is a pure function in a single array. Add a signal in one place; both surfaces pick it up. The merge forced divergent thresholds to reconcile into one definition.

🔹 **Read once, run many.** Gather each project's signals once; run all rules against the struct. "Healthy" is derived from the absence of red, not asserted.

🔹 **Keep the core pure.** The native SQLite addon is imported *inside* the one impure function, so the rule set unit-tests with zero infrastructure.

Net: ~500 lines of duplicated logic gone, replaced by one tested ~450-line module. The real win is a single, readable definition of "what deserves attention."

Full article → [link placeholder — operator fills in after Medium publish]

#SoftwareEngineering #AIAgents #Refactoring #TypeScript #Observability
