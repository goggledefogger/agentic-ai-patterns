---
type: pattern
date: "2026-08-22"
source: Agent-to-household-agent property lane (the agent board #18–#55) — 24 tasks over three weeks that the receiver could not execute, none refused at the sender
tags:
  - agents
  - agent-to-agent
  - capability-contracts
  - permissions
  - tool-design
---

# The sender enforces the receiver's ceiling

When one agent hands another agent work through a shared surface, the sender
knows the task and the receiver knows its own limits — and nothing forces those
two facts to meet. A task that the receiver can never execute posts fine, gets
claimed fine, and fails in a way that reads as the receiver's fault. Posting
the same impossible shape twenty-four times is not twenty-four bugs; it is one
missing check on the side that writes.

## The incident

The personal agent posts the household agent bounded tasks as issues on a shared board. Her poller wakes a
Hermes session by cron to work each one. Hermes denies a cron-woken session
**every** shell command (`approvals.cron_mode: deny`). So every task whose
completion was a command — 21 "pull the repo", 3 "send the link" — was
structurally impossible from the moment it was posted, and the personal agent posted them for
three weeks. The receiver's refusals read as policy ("prohibited under safety
rules"), and a human finishing the step by hand after each one kept the lane
looking ~90% alive (`a-refusal-names-a-rule-not-a-wall`). The sender had a
500-char cap, a fence guard, an idempotency key and a never-contents bound on
the envelope — and no idea what the receiver could *do*.

## The Pattern

1. **Put the receiver's execution ceiling in the sender's contract, where the
   verb is named.** Not in the receiver's docs, which the sender never reads at
   post time. `dispatch_sheila.py`'s usage now says: *a task is a judgment or a
   pointer, never a command — her board session cannot run one.* Same move as
   `a-capability-contract-must-name-the-verb`, from the other direction: that
   one names what the foreign agent *may* do; this names what the peer
   *cannot*.
2. **Refuse the unambiguous shapes at the sender, and name the route.** An
   interpreter prefix, a git verb, a remote shell in the task text is refused
   with "make it a judgment or a pointer, or route the mechanical step to her
   poller / a cron." Be honest that this is a speed bump: "Send the author the link"
   contains no command and was exactly as impossible. Do not parse prose for
   intent; catch the honest mistake and say why.
3. **Move the mechanical step to the receiver's deterministic layer, not its
   turn.** The structural fix is on the receiving side: the poller now performs
   `deliver-*` itself (resolve name → chat id, check the page is 200, send,
   close quoting the log line), zero tokens, no approval surface in the way.
   What the sender's check can't catch, the receiver's design makes moot.
4. **Carry what the deterministic layer needs in the envelope.** A name in the
   task *sentence* is for a reader; the poller resolves a `requester:` *field*.
   The first re-post after the fix failed for exactly this — the field was
   missing, the poller correctly refused to guess. A field the consumer reads,
   not a sentence it would have to parse.

## Why it stays invisible

- The sender's own validation is thorough on **form** (length, fences, keys)
  and silent on **feasibility**. Every envelope that fails the ceiling passes
  the schema.
- A peer's refusal arrives in the register of policy, so the sender tunes
  wording, consent, lookup — everything except the one question "could it have
  run that at all?"
- The sender's tests prove its envelopes against its own reader. They never
  prove them against what the receiver's runtime permits.

## Watch-outs

- **A path is a pointer, not a command.** The first version of the check
  refused `~/.openclaw/...` paths and broke the one task type that is *correct*
  (append a line to the household-asks file). The receiver's file tools are not
  its terminal tool. Refuse interpreters and verbs; keep pointers.
- **The ceiling can move.** A Hermes upgrade or a config reset changes what the
  cron session may do; the sender's check encodes today's ceiling. Pair it with
  a live contract probe where one exists (`--check-contract` already reads the
  receiver's caps; the ceiling is the next thing it should read).
- **Do not loosen the receiver to fit the task.** The tempting fix was a
  Hermes-level allow for the send script. The right fix removed the LLM from a
  step that had no judgment in it.

## Adjacent Patterns

- `a-refusal-names-a-rule-not-a-wall` — the receiver's side of this incident:
  why the refusals misled and how a hand-finish masked the rate
- `a-capability-contract-must-name-the-verb` — the sender's contract names the
  verb; this one names the ceiling beside it
- `subagent-cannot-consent` — a delegated agent cannot satisfy the gate it
  hits; here the delegated agent could not reach the gate at all
- `coordinate-agents-through-shared-state` — the board is the right surface;
  this is what the writer to that surface still owes
- `a-model-never-picks-the-destination` — the same instinct (`requester:` is a
  field the poller resolves, never a name the model infers)
