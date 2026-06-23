---
title: "The Silent Agent: Catching Usage Limits and Auto-Switching Models Mid-Run"
project: claude-mcd
tags: [AI, DevOps, LLM, Reliability, Agents]
status: draft
date: 2026-06-23
---

An autonomous agent that goes quiet looks exactly like an agent that's thinking. There's no error banner, no stack trace, no crash to page someone about. It just… stops replying. For a fleet of long-running Claude agents working across Discord channels, that ambiguity is the whole problem — and one of the most common causes turned out to be the most invisible: the agent hit a plan usage limit.

This is how MCD, our multi-channel agent platform, learned to tell the difference between *thinking* and *throttled* — and to do something about it without a human in the loop.

## A failure mode that looks like success

When a Claude subprocess exhausts a model's quota, the CLI doesn't throw. It writes a synthetic assistant message into the session transcript:

```json
{"type":"assistant","message":{"role":"assistant","model":"<synthetic>",
"content":[{"type":"text","text":"You've hit your Sonnet limit · resets Jun 24, 7am (UTC)"}]},
"isApiErrorMessage":true,"apiErrorStatus":429}
```

From the operator's seat in Discord, nothing happens. The agent had been streaming tool calls a moment ago; now it's silent. Was it mid-task? Stuck? Dead? You can't tell from the outside.

The frustrating part: recovery was already one command away. MCD has long supported `!project model <slug> --set opus` and `!project provider <slug> --set minimax` — both respawn the subprocess on a different model or provider. A Sonnet limit frequently hits while **Opus is still available**, or while a configured MiniMax provider is sitting idle. The capability to recover existed. The *signal* that recovery was needed did not.

## Detect where the agent already watches

MCD already runs a transcript poll loop — every two seconds it tails each project's `.jsonl` and parses new lines for assistant messages and tool-call progress. That loop was the natural place to catch the limit-hit, because the limit-hit *is* a transcript line.

The detection is deliberately conservative about what it trusts. It gates on the **structured fields first**, and only then reaches for the text:

```ts
if (rec.isApiErrorMessage === true && rec.apiErrorStatus === 429) {
  const msg = rec.message
  let text = ''
  if (typeof msg?.content === 'string') text = msg.content
  else if (Array.isArray(msg?.content)) text = msg.content.map(b => b?.text ?? '').join(' ').trim()
  if (text && text !== this.lastLimitRaw) {
    this.lastLimitRaw = text
    this.fireLimitHit(parseLimitMessage(text))
  }
}
```

Two design decisions are buried in those eight lines. First, the regex that pulls out the model name and reset time never decides *whether* this is a limit event — `isApiErrorMessage` and `apiErrorStatus: 429` do. If a future CLI release rewords "You've hit your Sonnet limit," the worst case is a generic alert with a null model, not a missed event. Second, `lastLimitRaw` dedupes per episode: the same 429 line gets re-read on every 2-second poll, but the operator is alerted exactly once.

## A pure module for everything that's testable

The interesting logic — *what* to offer and *whether* to auto-switch — lives in a standalone module (`limit-offer.ts`) with no I/O and no Claude dependency. That's what makes it unit-testable without spawning a subprocess.

`parseLimitMessage` extracts the limited model and reset time, and keeps the reset string **verbatim**:

```ts
const modelMatch = text.match(/hit your\s+(.+?)\s+limit\b/i)
const resetMatch = text.match(/resets?\s+(.+?)\s*$/i)
return {
  limitedModel: modelMatch ? modelMatch[1].trim() : null,
  resetsAt:     resetMatch ? resetMatch[1].trim() : null,
  raw: text,
}
```

`"Jun 24, 7am (UTC)"` stays a string. We never parse it into a `Date`. Reset times come from the CLI in whatever timezone and format it chooses, and the only thing the operator needs is to read it — parsing buys nothing and risks a class of timezone bugs we'd rather not own.

`computeLimitOffer` then builds two things: the human-facing list of switch commands, and an optional auto-switch decision. Crucially, it only offers a provider switch when that provider's API key is actually present in the environment:

```ts
for (const [alias, def] of Object.entries(providers)) {
  if (env[def.apiKeyEnv]) {
    offerLines.push(`!project provider ${slug} --set ${alias}`)
  }
}
```

No dead-end commands. If `MINIMAX_API_KEY` isn't set, the bot never suggests routing to MiniMax.

## Offer first, automate only when asked

The recovery path has two modes, and the default is the cautious one.

By default the bot **offers** — it posts a Discord alert naming the limited model, the reset time, and a ready-to-paste set of switch commands. The operator stays in control.

Auto-switch is **opt-in**, gated on a new per-project config field:

```ts
limitFallback: z.array(z.string()).optional(),
// e.g. ["opus", "minimax"] — absent/empty → offer-only
```

When `limitFallback` is set, `computeLimitOffer` walks the ordered list, skips the model that's currently limited, skips any provider whose key is missing, and returns the first usable entry. The server applies it by writing the new model or provider into config, respawning the subprocess, and posting a one-line "auto-switched" notice:

```ts
if (autoSwitch.kind === 'model') updated.model = autoSwitch.value
else updated.provider = autoSwitch.value
saveConfig({ ...cfg, projects: { ...cfg.projects, [evt.chatId]: updated } })
void projectPool?.killChat(evt.chatId).catch(() => {})
```

No new respawn machinery — it reuses the exact `set`-and-restart plumbing the manual commands already used. The limit-hit event itself flows through the same `ProjectPool` event bus as tool-progress and budget alerts: the process fires `onLimitHit`, the pool re-emits it as a `limit-hit` pool event, and the server's existing event handler does the routing. One new event kind, zero new transport.

## What it changed

The win isn't dramatic code — it's the elimination of a guessing game. A silent agent now announces *why* it's silent, names the model that's blocked, says when it resets, and either hands the operator a one-paste fix or quietly switches itself onto an available model and asks to be poked. The work that used to start with "is it stuck or dead?" now starts with a clear answer.

A few things were worth holding the line on:

- **Trust structure over prose.** Gating on `isApiErrorMessage` + `429` before touching the regex means a CLI wording change degrades to a generic alert instead of a silent miss.
- **Don't parse what you only need to display.** Keeping the reset time as an opaque string sidestepped a timezone-bug surface for zero loss of function.
- **Make the risky thing opt-in.** Detection and alerting are read-only and additive — safe to ship for everyone. Auto-switching mutates config and respawns a process, so it stays behind an explicit per-project flag.
- **Push the logic somewhere you can test it.** The detection lives in a noisy poll loop, but the *decisions* live in a pure module with unit tests covering model extraction, reset capture, the opus-over-sonnet fallback, and the hide-when-no-key rule.

The broader lesson for anyone running autonomous agents at scale: your fleet's worst failures won't always announce themselves. Some of them look identical to the agent working perfectly. The job is to make the invisible legible — and, where it's safe, to let the system recover before anyone has to ask why it went quiet.
