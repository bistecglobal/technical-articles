A slow CI pipeline isn't a speed problem — it's an attention problem. If your E2E suite takes twenty minutes, nobody waits for it. They push and move on, and the check that was meant to catch regressions becomes a notification they skim.

We reworked keyflow's Playwright E2E pipeline (Next.js + Prisma + Postgres) around one idea: stop doing the same expensive work on every run. What carried the weight:

🗄️ **Three content-keyed caches.** node_modules, the Playwright browser, and the Next.js `.next/cache` — each keyed on a lockfile/source hash so a hit is always safe and a real change always misses.

⏭️ **Skip, don't speed up.** On a warm cache, `npm ci` isn't faster — it doesn't run at all, guarded by a cache-hit conditional. Look for whole steps to remove, not just optimise.

🔑 **`restore-keys` is the unsung hero.** A perfectly precise key rarely hits mid-development; the prefix fallback gives the incremental webpack builder a warm starting point anyway.

🖥️ **Self-hosting is a tradeoff.** Local cache and no VM cold-start — but a missing Docker binary turned into a buildx→podman migration. You own the environment now.

🎯 **The cheapest test is the one you correctly skip.** PRs map branch name → feature spec; the full suite is the merge gate.

Full article → [link placeholder — operator fills in after Medium publish]

#DevOps #CICD #GitHubActions #Playwright #NextJS
