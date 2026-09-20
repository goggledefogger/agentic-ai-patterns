---
type: pattern
date: "2026-08-20"
source: The dashboard — a message sent while the brain was already writing its answer ran as the CLI's NEXT turn; the server ended the page's stream at the first result, so that turn ran with nobody reading and the page looked idle until history re-read
tags:
  - code
  - reliability
  - event-driven
  - streaming
  - anti-pattern
---

# A Stream That Ends Its Unit May Still Owe a Turn

A long-lived stream carries units of work — a turn, a response, a job. The
obvious close point is the end-of-unit marker (`result`, `done`, `finish`).
But a producer can owe *more than one* unit on a single stream: a message
the member sent mid-answer that the CLI defers to its own next turn, a queued
item about to be delivered, a retry the server will run itself. End the
stream at the first marker and the next unit runs into a closed pipe — it
executes, nobody reads it, and the consumer looks **idle**, not broken.

The failure is invisible wherever a turn owes nothing after it, which is the
common case and the one you test. It shows only when a second unit is already
in flight at close time: you send a message while the answer is still
streaming, the answer finishes, and then — nothing, until something else
(a history re-read, a poll) happens to repaint. "It's not initiating the next
command" is the report; the next command initiated fine, off-screen.

## The Pattern

At the end-of-unit marker, ask whether the producer still owes work on this
stream. If it does, keep the stream open, write a boundary event, and let the
next unit ride the same stream:

```js
if (pending && (owesDeferred > 0 || queue.length)) {
  pending.write(resultLine + '\n');
  pending.write(JSON.stringify({ type: 'turn_break' }) + '\n'); // boundary, not close
  deliverNext();                     // the next unit streams onto the SAME pipe
  if (uncertainWhetherMoreComes) armGrace(() => pending.end()); // close honestly if not
  return;
}
pending.write(resultLine + '\n'); pending.end();  // owes nothing → close
```

The consumer treats the boundary as a unit restart, not an end: reset the
in-progress render, show the working indicator again, keep reading. When the
producer *might* owe nothing (a mid-turn message the CLI folded into the
current answer instead of deferring — you can't always tell which), arm a
short grace timer that closes the stream if no next unit arrives, so an
uncertain owe never hangs the consumer on an indicator that leads nowhere.

## Why it bites here specifically

Interactive CLIs distinguish *fold* (the message joins the running turn) from
*defer* (it becomes the next turn), and don't always announce which. A bridge
that streams the CLI 1:1 inherits that ambiguity at exactly the close point.
The grace timer is how you stay correct without the CLI telling you.

## Also

Strip inherited env that silently changes child behavior. A dashboard started
*from* a Claude Code session passed `CLAUDE_CODE_CHILD_SESSION` down to the
`claude` children it spawned; that marker turns transcript saving **off**, so
history read empty and a resumed/remote session had nothing to resume. It
never reproduced from a normally-launched server. Add such variables to the
child-env strip list beside the credentials you already remove — see
[[a-twin-inherits-the-conversation-not-the-channels]].

## Related

- [[a-refused-reconnect-ends-the-stream]] — the other way an event stream goes
  quiet while looking merely unchanged; both fail invisibly on the dev machine
- [[never-trade-the-screen-for-a-promise]] — the consumer-side discipline for
  the repaint that a turn_break triggers
- [[a-resumed-session-answers-from-yesterday]] — why transcript saving has to
  actually happen for resume/remote to mean anything
