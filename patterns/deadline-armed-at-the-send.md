---
type: pattern
date: "2026-10-09"
source: A dashboard's bridge to an ACP-speaking coding agent. The agent took a prompt and sent nothing for 72 minutes, and the "went quiet after the first words" guard never armed because no words ever came
tags:
  - reliability
  - watchdogs
  - timeouts
  - streaming
---

# A Deadline Armed by Output Cannot See the Failure That Produces None

A bridge streamed a coding agent's turn to a chat page. It had a settle guard: if the agent streamed words, then went quiet for 90 seconds without ending its turn, close the turn. The guard armed on the first words.

Then the agent took a prompt and sent nothing at all for 72 minutes. No words, no thought, no tool call. The guard never armed, because the thing that arms it never happened. The member had to press Stop, or type "status?", to find out. The protocol has no timeout of its own, and the upstream bug report was still open.

## The Pattern

**Arm the deadline at the send. Let only real work disarm it.**

A guard armed by the first sign of output covers the stall in the middle of a turn. It cannot cover a stall before the first sign, because that failure produces no output and so never starts the clock. Keep both:

- the settle guard, armed by the first words (silence after progress)
- a first-word deadline, armed when the prompt is sent (3 minutes in this case), cleared by the first real work signal

Three rules that keep the second one honest:

1. **Count only real work as life.** Words, thought, a tool call, a permission ask. Bookkeeping events (usage counters, command lists) do not count, because a wedged engine can still emit them and would disarm the guard while doing nothing
2. **One close per turn, never an automatic resend.** The prompt may have been received and be half-acted on. Resending can run it twice ([[at-least-once-needs-a-poison-ledger]]). Close, say why, and let the member decide
3. **Write the verdict, not the content.** Each guard that fires records which guard, which engine, which session, how long it waited, to a file that survives restarts. The server's stderr went to /dev/null that day, so a second watchdog's failure to fire could not be diagnosed afterward ([[guard-evidence-outlives-the-failure]], [[log-the-verdict-not-the-volume]])

## The Test

For each deadline in the system, ask what event starts its clock. If the answer is something the failing component produces, the guard cannot see that component failing to produce it. Move the start to something the caller does, like the send.

## Related

- [[liveness-is-measured-at-the-ear]]: a watchdog fed by the wrong signal certifies the wrong thing. This one is fed by a signal that only exists when things are going well
- [[a-stream-that-ends-its-unit-may-still-owe-a-turn]]: the other end of a turn's lifetime, closing too early instead of too late
- [[at-least-once-needs-a-poison-ledger]]: why the close does not retry
- [[guard-evidence-outlives-the-failure]]: the verdict file is the guard's evidence
- [[log-the-verdict-not-the-volume]]: what the verdict line holds
