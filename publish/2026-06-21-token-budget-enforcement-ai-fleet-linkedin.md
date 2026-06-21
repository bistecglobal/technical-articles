Running 19 AI agents autonomously means one runaway agent can drain a month's API budget overnight — unless you build enforcement into the platform itself.

Here's how we wired token budget controls into BistecGlobal's Multi-Channel Discord (MCD) fleet:

🟡 **Graduated alerts** — `budget-alert` events fire at 50%, 80%, and 100% usage. Each threshold fires once per month via an in-memory dedup map, so operators aren't spammed.

🚫 **Hard enforcement with queueing** — at 100%, inbound messages queue in memory rather than reaching the Claude subprocess. They don't drop — they wait for the month to reset.

🔄 **Automatic month rollover** — on every message delivery, the pool checks if the UTC year-month changed. On rollover: alert state clears, queued messages drain automatically, and a `budget-restored` Discord notification fires.

📊 **Dashboard visibility** — `budgetStatus` (`ok`/`warning`/`critical`/`exhausted`) flows from transcript JSONL files through `/api/fleet` to color-coded slug chips in the Mission Control grid. No extra database needed — the transcripts carry everything.

One data source. Four observability layers. Zero manual month-reset work.

Full article → [link placeholder — operator fills in after Medium publish]

#AI #MultiAgent #DevOps #CloudEngineering #Observability
