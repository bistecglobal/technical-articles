# Audit — No Slug Required: Per-Channel Native Slash Commands

**Verdict: PASS** · **Score: 24/25** · Claims fixed: 0 · Claims flagged: 0

## Claim inventory

| # | Claim | Verdict | Evidence |
|---|---|---|---|
| 1 | One Claude subprocess per project, bound to its own channel | ✅ | MCD core architecture (prior shipped articles + ProjectPool) |
| 2 | Old path: master channel `!project model <slug>` text verb | ✅ | `handleModel` in `master-commands.ts` |
| 3 | New `/model` + `/provider` native slash commands operate on channel-bound project, no slug | ✅ | `model-command.ts` / `provider-command.ts` use `interaction.channelId` as `chatId` |
| 4 | Modules patterned on `/voice` command | ✅ | module docstrings reference `voice-commands.ts`; registration includes `voiceSlashCommands` |
| 5 | Each module exports a command schema + interaction handler | ✅ | `modelSlashCommands` + `handleModelInteraction` exports |
| 6 | Schema: 3 subcommands, `set` requires string `name` | ✅ | exact code (type 1 sub-commands, type 3 required string) |
| 7 | Handler adapts interaction → `handleModel` arg shape; `chatId` is the project selector | ✅ | `rest = [chatId, '--set', name]` etc. |
| 8 | `handleModel`/`handleProvider` exported from master-commands for reuse | ✅ | commit `5237b72` "export handleModel for reuse"; `import { handleModel } from './master-commands.ts'` |
| 9 | Mutations gated: `show` open, `set`/`clear` check `loadAccess().allowFrom.includes(userId)`, ephemeral deny | ✅ | handler auth guard + server dispatch `isAllowed` |
| 10 | `set`/`clear` kill subprocess → respawn with new env; provider routes to alias from `defaults.providers` or back to Claude auth | ✅ | module docstrings + commit `afb9b69` body |
| 11 | Registration `guild.commands.set([...voice, ...model, ...provider])`, single dispatcher | ✅ | `afb9b69` server.ts diff line 56 |

## Forward-looking scan
No matches for `will`/`plan to`/`coming soon`/`in the future`/`next step`/`roadmap`/`soon`. Clean.

## Rubric

| Dimension | Score | Notes |
|---|---|---|
| Evidence quality | 5 | All claims trace to source; no SHAs in prose |
| Technical depth | 5 | Schema contract, adapter pattern, auth split, respawn mechanics |
| Clarity for audience | 5 | Clear arc; "let the surface carry context" framing lands |
| BistecGlobal voice | 4 | Practitioner, evidence-grounded; one small feature stretched well without padding |
| Title specificity | 5 | Concrete, names the key insight (no slug) and scope |
| **Total** | **24/25** | |

Above 20 threshold. No fixes required.
