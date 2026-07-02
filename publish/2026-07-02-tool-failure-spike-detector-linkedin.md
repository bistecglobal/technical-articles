A single AI agent throwing a tool error tells you nothing. Three agents failing on the same tool in the same five minutes tells you the tool layer is down.

Autonomous agents generate recoverable errors constantly — alert on each one and you train yourself to ignore alerts. So we built a fleet-wide spike detector for our agent platform that flips the noise into signal:

• A spike is defined by distinct affected projects, not raw error count — ≥3 agents failing on one tool in a 5-minute window
• Severity scales with breadth: the more projects hit, the bigger the fire
• Zero new telemetry — it reads the tool_result error events every session already logs, joining tool_use_id back to the tool name
• A full-fleet scan every 30s stays cheap via mtime skips + a 2-minute cache
• The heatmap shows sub-threshold errors too, so you watch a failure spread from one project to three and catch it crossing the line

The lesson generalizes past AI: in any fleet, the count of distinct affected instances is a far better alarm than raw error count.

Full article → [link placeholder — operator fills in after Medium publish]

#AIAgents #Observability #Reliability #DevOps #SRE
