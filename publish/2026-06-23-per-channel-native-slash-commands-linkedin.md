The fastest way to make an operator type the wrong thing is to ask them which project they're in.

Our multi-channel Discord bot runs one Claude subprocess per project, each bound to a channel. Switching a project's model used to mean a master control channel and `!project model <slug>` — naming the slug from somewhere else, hoping you got it right. We replaced that with native `/model` and `/provider` slash commands that act on the channel you're already in. No slug.

What the redesign taught us:

🎯 **Let the surface carry the context.** The channel already identifies the project. Making the operator restate it as a slug was pure friction — and a source of error.

🛡️ **Push validation to the platform.** Native slash commands let Discord enforce required arguments and types before your code runs — a whole category of input bugs you never write.

🚪 **One implementation, many front doors.** The slash handlers are thin adapters over the same functions the text verbs already call. New entry points, not new logic to keep in sync.

🔑 **Gate the verb, not the noun.** `show` is open; `set`/`clear` check the access list and respawn the subprocess so the change takes effect on the very next message.

Type `/model set name:opus` in a project's channel — next turn runs on Opus, no restart, no redeploy.

Full article → [link placeholder — operator fills in after Medium publish]

#AI #DevOps #Discord #DeveloperTooling #SoftwareEngineering
