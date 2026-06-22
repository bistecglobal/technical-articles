When you run 19 AI agents across different projects, each one quietly builds its own memory. The problem: you can't see what they share — until you map it.

We added a Jaccard similarity heatmap to our fleet dashboard that answers "which agents think alike?" Here's what we learned:

🔹 **The approach**: Extract keywords from each project's memory files, compute Jaccard similarity (intersection / union) across every project pair, render as a dark-to-cyan N×N heatmap with a threshold slider and hover tooltips showing top shared keywords.

🔹 **The surprise**: Two product-facing projects in completely different business domains shared 15–20% overlap — almost entirely around deployment, token management, and retries. Both agents hit the same distributed system failures independently. That's signal, not noise.

🔹 **The design choice**: We used keyword Jaccard over vector embeddings deliberately — it's transparent (the shared keywords explain the score), runs without external model calls, and fits the bill for a lightweight operational diagnostic.

🔹 **The result**: A 5-minute cached API + React grid that surfaces coordination opportunities and confirms healthy specialization — all from reading the markdown files your agents already write.

🔹 **Key insight**: What your AI fleet knows collectively matters as much as what any individual agent knows. A heatmap over memory files is an affordable way to find out.

Full article → [link placeholder — operator fills in after Medium publish]

#AIAgents #Observability #EngineeringBlog #DevOps #BistecGlobal
