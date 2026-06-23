An autonomous AI agent that hits its usage limit doesn't crash — it just goes silent. And a silent agent looks exactly like one that's thinking. That ambiguity was one of the most common failure modes in our multi-channel Claude fleet, and the most invisible.

Here's how we made MCD tell *thinking* apart from *throttled* — and recover on its own:

🔎 Detect where it already watches. The transcript poll loop already tails each agent's session log every 2s. A usage limit shows up there as a synthetic message with `isApiErrorMessage:true` and `apiErrorStatus:429` — so we catch it in the loop that's already running.

🧱 Trust structure over prose. Detection gates on the 429 status fields first; the regex that reads the model name and reset time runs second. If the CLI rewords its message, the worst case is a generic alert — never a silent miss.

🕒 Don't parse what you only display. The reset time ("Jun 24, 7am UTC") stays a string. No Date parsing, no timezone bugs.

⚙️ Offer first, automate only when asked. By default the bot posts a ready-to-paste switch command and lets the operator decide. An opt-in per-project fallback chain (e.g. ["opus", "minimax"]) lets it auto-switch and respawn — skipping any provider whose API key is missing.

The broader lesson: your fleet's worst failures won't always announce themselves. Some look identical to the agent working perfectly. The job is to make the invisible legible.

Full article → [link placeholder — operator fills in after Medium publish]

#AI #LLM #DevOps #SoftwareEngineering #Reliability
