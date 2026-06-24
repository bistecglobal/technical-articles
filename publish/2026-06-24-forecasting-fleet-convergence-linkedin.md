A health score of 62 tells you where an autonomous agent is. It tells you nothing about whether it's getting there.

Our Mission Control dashboard tracks a 0–100 convergence score per Claude agent — how close it is to its goal. We had the number over time, but couldn't answer the operator's real question: is this one going to make it, and when? A 62 could be 3 days from done or flatlined a week ago. The snapshot can't tell them apart. The slope can.

So we fit a least-squares trend line to each agent's convergence history and project the day it crosses "healthy." The interesting part was keeping the math honest:

🔹 **Don't forecast from noise.** Require ≥3 data points before projecting anything.

🔹 **Use a deadband.** A stalled agent still posts a +0.01/day slope from noise — divide distance-to-target by that and you get an ETA of thousands of days. We refuse to forecast inside ±0.05/day and show "stalled" instead. That one rule is the difference between a forecast and a fabrication.

🔹 **Rank by what to look at first.** Soonest ETA on top, dead weight (reached, declining) at the bottom.

🔹 **Draw the projection, not just the number.** A dashed line from "now" to the target line lets operators sanity-check the ETA — a bare "7 days" invites blind trust; a drawn trajectory invites judgement.

No new data, no new table — just reading an existing series forward instead of backward. A score says where you are; a slope says where you're going.

Full article → [link placeholder — operator fills in after Medium publish]

#AIAgents #Forecasting #Observability #DataScience #SoftwareEngineering
