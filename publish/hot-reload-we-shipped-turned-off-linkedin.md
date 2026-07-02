We built hot-reload for our AI agent wizard — then shipped it turned off in production. That wasn't a mistake. It was the design.

Our wizard's flow lives in a folder of Markdown step files, loaded once at boot into an in-memory graph. Editing one word meant restarting the container — which killed every in-flight build session on that instance. Fix a typo, three users lose their place.

How we fixed the dev loop without risking production:

• Put a seam where the lifetime mismatch was — an `IStepGraphProvider` with an atomic `Interlocked.Exchange` swap and lock-free `Volatile.Read`.
• Made the graph an immutable record, so a live turn keeps the snapshot it captured even if a reload lands mid-flight. Concurrency becomes boring.
• Ran a FileSystemWatcher plus a polling fallback — Docker bind mounts don't reliably fire inotify events, so polling is the real trigger.
• Kept-last-good on parse failure: a bad edit means "your change didn't take," never "the wizard is broken."
• Shipped it off by default. A structural edit mid-session can corrupt a user's saved position — fine on a dev laptop, not for a real user. Dev opts in with one env flag.

The most useful thing we built might be the flag that keeps it disabled. A safe default is a design decision, not a cop-out.

Full article → [link placeholder — operator fills in after Medium publish]

#SoftwareEngineering #DotNet #AIAgents #DeveloperExperience #SystemDesign
