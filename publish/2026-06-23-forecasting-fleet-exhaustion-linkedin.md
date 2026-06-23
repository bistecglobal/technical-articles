A budget alert at 100% is a smoke detector that goes off after the house has burned down. By then the agent is queued and the sprint has stalled. The question that matters isn't "are we over budget?" — it's "*when* will we be?"

We added two forecasts to Mission Control, the dashboard for our multi-channel Claude fleet — one for token spend, one for backlog throughput. What we learned building them:

📈 **Forecast rates, not levels.** A gauge says where you are. A recent rate × time remaining says where you'll be — the only number that lets you act *before* the wall.

🪟 **Pick the smoothing window on purpose.** A 7-day rate is responsive enough to catch a real shift in pace, stable enough to ignore one heavy afternoon. One day is noise; 30 days is too slow.

❓ **Render uncertainty honestly.** No budget set → no projected date (not `Infinity`, not a fake "you're fine"). The backlog forecast shows a fan chart whose band *widens* the further out you look. The widening is the feature — a single confident line into the future is a lie.

🔗 **One source of truth.** Both forecasts layer on existing endpoints, so they can never contradict the charts beside them.

Full article → [link placeholder — operator fills in after Medium publish]

#AI #DevOps #Observability #SoftwareEngineering #AIAgents
