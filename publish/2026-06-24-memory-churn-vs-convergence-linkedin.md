An AI agent rewriting its memory furiously looks identical, from the outside, whether it's having a breakthrough or spinning in circles. Both produce a wall of diffs.

In our Mission Control fleet dashboard we log every memory change an autonomous Claude agent makes, and separately track a convergence score (how close it is to its goal). Heavy churn was easy to see — what it *meant* was not. So we put both on one chart and asked the question that disambiguates them: when this agent churns, does its convergence go up or down?

🔹 **Two axes beat one.** Churn on x, convergence delta on y. The plane splits into quadrants: heavy churn + rising = productive; heavy churn + falling = *thrashing*. The dangerous quadrant is invisible if you only watch churn — both look the same on the x-axis alone.

🔹 **Count the thrashers.** Bottom-right population goes in the header in red — a direct prompt to intervene on agents that are busy and going backwards.

🔹 **A correlation with an honesty deadzone.** Fleet-wide Pearson r of churn vs progress — but null below 3 points, null on zero variance, and a ±0.3 dead zone so a 0.12 reads as "none," not "weakly positive." The statistical honesty lives in the code, not the reader's head.

The lesson: a single metric is often ambiguous until you pair it with an outcome. Churn alone can't tell thinking from thrashing — progress on the other axis can.

Full article → [link placeholder — operator fills in after Medium publish]

#AIAgents #Observability #DataScience #Statistics #SoftwareEngineering
