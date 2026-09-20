---
type: pattern
date: "2026-07-31"
source: Live walk-and-talk field session, 2026-07-31 — a mobile page's mic-unavailable flag silently blocked the stream-reconnect handler from ever clearing a stale CONNECTION error
tags:
  - state-management
  - anti-pattern
  - ui
  - frontend
---

# An Error Flag Owned by One Subsystem Must Never Gate Another Subsystem's Status Report

A mobile page set `micBlocked = true` when the browser reported the mic unavailable. The stream-reconnect handler refreshed its own status label only `if (!micBlocked)`. So when the stream later reconnected perfectly, the label could not be cleared — the page sat forever displaying a CONNECTION error while the connection was the one thing working.

The two subsystems (mic availability, stream connection) are unrelated. The flag belonged entirely to the mic. But it sat as a silent precondition on the code path that reports the *other* subsystem's health, so a permanent fault in one froze the displayed status of both.

## The Pattern

**An error flag owned by subsystem A must never appear as a condition on subsystem B's status reporting.** If B's report can't run because A is unhappy, the user is told about a fault in the healthy component.

1. **Each subsystem reports its own status unconditionally.** The reconnect handler should update its label whenever the stream state changes, full stop — no gate on anything outside the stream.
2. **Combine statuses at render time, not at write time.** If the UI wants to show "worst of both," that's a pure function computed from two independent, always-current values (`micStatus`, `streamStatus`) — never a write-time guard that can leave one of them stale.
3. **Ask of any `if (!otherSubsystemFlag)` guard: does this gate the write, or the read?** Gating the write is where this breaks — the write silently never happens again, and the last-written value goes stale forever. Gating the read (compute both, choose which to display) is safe, because it re-evaluates every time.

## Why It Bites Specifically

- **It fails toward a permanent, misleading state, not a transient one.** Once the guard skips a write, nothing re-triggers it — the stream can reconnect a hundred times and the label never updates, because the condition that blocks it (`micBlocked`) never becomes false. A one-off missed update would read as a timing quirk; a permanently stuck one reads as a real, ongoing outage.
- **It points the user at the wrong component.** The displayed error names the connection. The actual fault is the mic. Anyone debugging from the UI alone will chase the healthy subsystem.
- **It's invisible in code review of the mic path.** The `micBlocked` flag does exactly what its own subsystem needs. The bug lives in a different file, in a guard clause that looks like unrelated defensive coding.

## Watch-outs

- **This is different from a genuine shared precondition.** If B's status truly cannot be known while A is broken (B depends on A for data), gating is correct — the bug here is that mic availability and stream connectivity don't actually depend on each other; the flag was reused because it happened to be in scope, not because it was the right precondition.
- **The tell is a boolean named after one subsystem, read inside another's update path.** Grep for flags that cross a file/component boundary they weren't defined for use one time as a status-write guard.

## Adjacent Patterns

- `undefined-blank-is-a-decision.md` — a different shape of a flag silently governing more than intended, there through undefined semantics rather than a stray guard.
- `atomic-state-writes.md` — related discipline on keeping state writes from landing in a corrupted or stale intermediate state, though the failure mode there is a crash mid-write rather than a write that never fires.

## Source

Live walk-and-talk field session, 2026-07-31.
