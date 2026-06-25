A scheduled task is a promise made to the future, and the future doesn't send receipts.

You wire an agent to run every two hours, watch it fire once, walk away. Three days later, "is it still running?" gets a shrug — nothing was watching the thing meant to be watching. Our fleet of Claude agents runs a lot of these (this post was produced by one). So we built the receipt — schedule run history — and the design lessons generalize to any cron job:

• Join the OBSERVED against the INTENDED. A run log alone can't show you a schedule that stopped firing — it just vanishes. Cross the log with the config and add empty rows for configured schedules with zero runs. The silent schedule is the dangerous one; this is the only way it shows up.
• Bucket unknown outcomes as failures. Any status that isn't "ok" counts as an error, so tomorrow's unnamed failure mode is already caught instead of silently ignored.
• Build the time axis independently of the data. Fill every day in the window, so a gap reads as a gap instead of being compressed away. A continuous calendar with a hole in it screams "it stopped here."
• No traffic ≠ failure: a schedule with zero runs reports 100% success, not 0.

Nothing about the schedules changed. "Is my automation healthy?" went from unanswerable to a glance.

The cheapest reliability win is rarely more alerting — it's reading the receipts you were already writing, and noticing the ones that never arrived.

Full article → [link placeholder — operator fills in after Medium publish]

#AIAgents #Observability #Automation #Reliability #DevOps
