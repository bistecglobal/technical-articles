---
title: "No Slug Required: Per-Channel Native Slash Commands for a Multi-Project AI Fleet"
project: claude-mcd
tags: [AI, DevOps, Discord, Developer Tooling, Bots]
status: audited
date: 2026-06-23
---

The fastest way to make an operator type the wrong thing is to ask them which project they're in. Our multi-channel Discord bot runs one Claude subprocess per project, each bound to its own channel. To switch a project's model, you used to drop into a master control channel and type `!project model <slug>` — naming the slug explicitly, from somewhere else, hoping you got it right. It works, but it asks the human to carry context the system already has: *the channel you're typing in already is the project.*

So we made the channel do the talking. The bot now ships native Discord slash commands — `/model` and `/provider` — that operate on whatever project is bound to the channel you invoke them from. No slug. Discord autocompletes the command, validates the arguments, and the change applies to *this* project because there is no other one it could mean.

## Why native commands, not another prefix verb

The bot already had a perfectly good text interface: `!project ...` verbs parsed out of message content. Adding `/model` wasn't about replacing that — it was about removing a class of mistake.

A text prefix verb is invisible until you remember it exists, accepts free-form arguments Discord can't check, and — in the master-channel design — forces you to name the target project even when you're already looking at it. A native slash command flips all three: Discord surfaces it in the command picker, enforces the argument types and required fields before the bot sees them, and inherits the channel as implicit context. The operator's mental model shrinks from "which project, which verb, which syntax" to "set the model here."

## The shape of a command

Each command is a small, self-contained module. `/model` lives in `model-command.ts`, `/provider` in `provider-command.ts`, and they are deliberate mirror images of each other — and of the older `/voice` command they were patterned on. Each exports two things: a command *schema* Discord registers, and an *interaction handler* the bot dispatches to.

The schema is the contract Discord enforces on the client side. Three subcommands, with `set` requiring a string:

```ts
export const modelSlashCommands: RESTPostAPIApplicationGuildCommandsJSONBody[] = [
  {
    name: 'model',
    description: "View or change Claude's model for this channel's project",
    options: [
      { type: 1, name: 'show',  description: 'Show the current model for this channel' },
      { type: 1, name: 'set',   description: 'Set the model for this channel (respawns the subprocess)',
        options: [ { type: 3, name: 'name', description: 'Model name, e.g. opus, sonnet, haiku', required: true } ] },
      { type: 1, name: 'clear', description: 'Clear the override and fall back to the default model' },
    ],
  },
]
```

By the time the handler runs, Discord has already guaranteed `name` is present on a `set`. The bot never has to parse or defend against a malformed command — the platform did it.

## Reuse without duplication

The interesting decision is what the handler *doesn't* do. It doesn't reimplement model-switching logic. The existing master-channel path already had a `handleModel` function that reloads config, validates, writes the override, and kills the subprocess. The slash handler just adapts an interaction into the argument shape that function already expects:

```ts
const ctx = { mutator: deps.mutator } as unknown as MasterContext
let rest: string[]
if (sub === 'set') {
  const name = interaction.options.getString('name', true)
  rest = [chatId, '--set', name]
} else if (sub === 'clear') {
  rest = [chatId, '--clear']
} else {
  rest = [chatId]
}
const text = await handleModel(rest, ctx)
await interaction.editReply(text)
```

The channel id *is* the project selector — `chatId` slots into the exact position the master verb expected a slug. To make this work, `handleModel` and `handleProvider` were exported from the master-commands module for reuse. One implementation, two front doors: the master-channel text verb and the per-channel slash command both call the same code. Behaviour can't drift between them because there is only one behaviour.

## Authorization and the respawn

Two details make these commands safe and effective.

First, **mutations are gated.** `show` is open to anyone, but `set` and `clear` check the caller against the bot's access list — `loadAccess().allowFrom.includes(userId)` — mirroring the master-channel mutation policy exactly. An unauthorized user gets an ephemeral "Not authorized" reply that nobody else in the channel sees. Read is free; change is privileged.

Second, **a change actually takes effect.** Switching a model or provider isn't a config note for next time — `set` and `clear` kill the running Claude subprocess so the next message in the channel respawns it with the new environment. For `/provider`, that means routing the channel to a provider alias from the bot's defaults (or back to Claude subscription auth) and relaunching against it. The operator types one command and the very next turn runs on the new model. No restart, no redeploy.

Registration ties it together: at startup the bot calls `guild.commands.set([...voiceSlashCommands, ...modelSlashCommands, ...providerSlashCommands])`, and a single interaction dispatcher routes each invocation to the matching handler.

## Lessons

This is a small feature, but it encodes a few principles worth stealing:

- **Let the surface carry the context.** The channel already identified the project; making the operator restate it as a slug was pure friction and a source of error. Bind commands to where they're typed.
- **Push validation to the platform.** Native slash commands let Discord enforce required arguments and types before your code runs. That's a whole category of input-handling bugs you simply don't write.
- **One implementation, many front doors.** The slash handlers are thin adapters over the same `handleModel`/`handleProvider` functions the text verbs use. New entry points shouldn't mean new logic to keep in sync — they should mean a new way to call the logic you already trust.
- **Gate the verb, not the noun.** Splitting authorization at `show` vs `set`/`clear` keeps inspection frictionless while protecting every state change behind the existing access list.

The result is unglamorous and exactly right: an operator sitting in a project's channel types `/model set name:opus`, gets an ephemeral confirmation, and the next message runs on Opus — without ever naming the project, because they were already in it.
