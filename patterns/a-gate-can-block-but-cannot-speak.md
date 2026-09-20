---
type: pattern
date: "2026-08-03"
source: The personal agent — walk-and-talk's link-delivery Stop hook blocked the turn correctly, and its reason was dropped in transit; the agent got a bare "Continue from where you left off" and ended the turn again
tags:
  - agents
  - hooks
  - reliability
  - delivery
  - anti-pattern
---

# A Gate Can Block, But It Cannot Speak

A control that enforces by **telling another actor what to do** is two mechanisms
wearing one name: a *detector*, and a *message*. Tests cover the detector — it is
local, deterministic, and the part you wrote. The message rides a channel you did
not write and did not test, and when that channel drops it, the control fires
perfectly, blocks exactly as designed, and changes nothing.

## The Problem

A Stop hook existed to guarantee one thing: while a walk bridge is live and its
loop unconfirmed, the assistant may not end its turn until the phone URL has
appeared in its own reply text. It was written *because* prose had failed twice at
the same job. Its detection is sound — it parses the transcript, rejects tool
output, and proves absence cheaply before confirming presence by parse.

On a live run it detected correctly and returned exit 2, blocking the stop. What
reached the agent was:

```
Continue from where you left off.        (isMeta: true)
```

The hook's stderr — the URL, the warm welcome line, the explicit instruction to
say it in reply text — appears nowhere in the transcript. The agent, given a bare
continuation and holding no pending intent of its own, produced a null turn ("no
response requested"). On that second stop, `stop_hook_active` was true, so the
hook returned 0 by design — **never loop** — and the turn ended. The walker got
silence and asked for the link by hand, which is precisely the failure the hook
was built to prevent.

Nothing malfunctioned. The detector was right, the block landed, the guard behaved
as specified. The *payload evaporated between two processes*, and no layer was
watching for that, because every layer's tests end at its own boundary.

The symmetry is worth sitting with: the original bug was a URL printed into Bash
output, which the walker's phone UI collapses. The fix's own reason was printed to
stderr, which the agent's harness collapsed. **Same failure, one level up — a
message delivered into a surface that discards it, by a sender with no way to
tell.**

## The Pattern

1. **At a choke point, perform the side effect — do not request it.** The hook
   knew the URL. Anything the hook can *ask* the agent to do that the hook could
   instead *do*, it should do. An instruction is a hope with a schema.
2. **Deliver to the destination, not through the conversation.** The link's job
   was to be opened on a phone. A push to that phone is both more reliable and
   far shorter than any quantity of correct instruction to an agent about what to
   type into a UI whose rendering you cannot see. Prefer a channel already proven
   in production for something else (here: an existing Telegram nudge bot with
   credentials in a vault) over a new one.
3. **Treat the enforcement message as untested infrastructure until you have seen
   it arrive.** Assert it the way you assert the detector: end a turn in the
   failing state on purpose, and read the transcript for your own words. If your
   reason cannot be found there, your control is mute and you did not know.
4. **A "never loop" guard caps a mute control at exactly zero.** `stop_hook_active`
   is correct — a hook that can wedge a session is worse than the silence it
   prevents. But combined with a droppable message it converts *one dropped
   message* into *permanent silence on this run*. If the control's only power is
   a message, the guard is the ceiling on its power.
5. **Prefer the documented structured field over an ad-hoc stream, and still do
   not trust it.** For Claude Code Stop hooks, `{"decision":"block","reason":…}`
   on stdout is the specified path for feedback the model reads, and
   `systemMessage` the specified path for text the *user* reads. Both are better
   bets than exit-2 stderr. Neither is a guarantee: `systemMessage` has a known
   Stop-hook rendering regression, closed as not-planned. Structured beats
   improvised; measured beats structured.

## Corollary

The strength of a control is the *weakest* of {detect, decide, act}. Reviews and
tests concentrate on detect, because it is the part with interesting logic. Ask
of any gate: **when this fires, what physically changes?** If the honest answer is
"a string is emitted and someone else is expected to read it," the gate is
advisory wearing a deterministic costume — and its costume is the danger, because
everyone downstream stops carrying the rule themselves.

## Related

- [[decorative-gate]] — a control that checks nothing. This one checks perfectly
  and cannot make anything happen; the failure moved from the predicate to the
  effect.
- [[a-second-writer-satisfies-your-gate]] — the same hook, the day before, failing
  the other way: predicate true, delivery absent.
- [[enforce-the-message-not-the-field]] — when the message *does* arrive, its
  wording becomes the artifact; this pattern is the case where it never arrives.
- [[announce-the-move-in-the-old-room]] — the human-facing twin: a surface bound
  to the old location that never learns it was left behind.
- [[silence-is-not-confirmation]] — the sender's view: no error came back, and no
  error was ever going to.
- [[verify-by-exercising]] — the only way this was found: run the failing path and
  read what actually landed.
