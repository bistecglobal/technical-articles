A hospital monitor doesn't show one number — it shows a waveform, and a clinician reads the rhythm and the flatlines before any digit. We built the same for a fleet of AI agents.

Our agent fleet's activity landed in five separate logs: alerts, context injects, memory writes, digests, broadcasts. Each had its own page. None told you whether the fleet *as a whole* was busy, idle, or quietly flatlined. So we built a Fleet Activity EKG — a 48-hour, multi-lane waveform putting all five pulses on one screen.

The engineering choices that made it readable:

• Pull only what you plot — one query returns bare timestamp arrays from five tables, reconciling schema quirks (alerts vs injects split by type; broadcasts' ISO timestamps normalized to epoch) at the edge
• Scale each lane independently — memory fires dozens/hour, broadcasts twice a day; a shared y-axis would bury the rare ones. Per-lane normalization lets you read rhythm, not just volume
• Pre-seed empty hourly buckets — the silences are data; zero-filling makes a flatline actually look like a flatline

What the shape reveals that logs can't: a fleet-wide flatline (everything stops at once — a red flag on an autonomous fleet), a spike crossing lanes (alerts + injects align = something broke and got answered), a healthy diurnal rhythm whose arrhythmia you learn to spot.

A pile of logs tells you what happened. A waveform lets you feel the skipped beat before you go counting.

Full article → [link placeholder — operator fills in after Medium publish]

#AI #DevOps #Observability #AIAgents #DataVisualization
