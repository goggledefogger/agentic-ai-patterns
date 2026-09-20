---
type: pattern
date: "2026-08-10"
source: The personal agent — walk-and-talk player suite, tests 8 and 9 after the quiet-announce moved from speakPhraseNow to speakLocal, 2026-08-10
tags:
  - testing
  - verification
  - anti-pattern
  - refactoring
---

# A Stub Outlives the Seam It Watches

A test that stubs a function is making a claim about the code's routing: *this
is the door the behavior walks through.* The claim is true the day the test is
written and is re-checked never. When the mechanism later moves to a different
door — usually for a good reason, in a change that touches neither the test
file nor anything it visibly depends on — the stub keeps watching the old
seam, and the test starts lying.

The dangerous part is that it lies **in both directions at once**:

- A **positive** assertion ("the nudge fires") goes red on healthy code — the
  behavior fires through the new seam, the stub sees nothing, and a working
  feature reads as broken. Cost: a false regression hunt.
- A **negative** assertion ("the veto keeps it silent") goes vacuously green —
  a broken veto would fire through the new seam, the stub would still see
  nothing, and `spoke == []` passes forever. Cost: a guard that can no longer
  fail, which is worse than no test.

One moved seam, two polarities of lie, zero failing assertions pointing at the
actual cause.

## The incident

The walk page's "still on it" nudge was deliberately rerouted from the phrase
path (`speakPhraseNow`) to `speakLocal`, so that profile and sound toggles
could never mute essential narration. Correct change, shipped with its own
rationale. Two tests stubbed the old path:

- Test 8 ("the nudge cannot be starved") stubbed `speakPhraseNow`, recorded
  nothing, and failed — on code where the nudge fired correctly through
  `speakLocal`. It sat red at baseline and was nearly attributed to an
  unrelated presence feature landing the same day; only a worktree run at the
  pre-change commit proved it predated everything.
- Test 9 ("a genuinely playing voice vetoes the nudge") asserted `spoke == []`
  with the same stub — so it was passing *because the stub was blind*, not
  because the veto worked. Its green proved nothing and had been proving
  nothing since the reroute.

The same repository had hit the same shape one day earlier in a different
suite (a permission-dialog fixture built before a desk-only filter existed),
which is the tell that this is a class, not an accident.

## The rule

**When you move a mechanism, grep the tests for the old seam's name before you
ship.** Every stub of the old door is now wrong twice: fix the positive tests
so they watch the new door, and re-derive the negative tests so their silence
is once again earned rather than structural. A negative assertion is only
evidence while the thing it forbids *could* happen through a path the test can
see.

The inverse discipline for test authors: stub the narrowest seam the contract
names, not the deepest function you happened to find, and leave a one-line
comment saying *which* routing fact the stub encodes — that comment is what
turns a silent double-lie into a one-line diff review catch when the routing
changes.

## Related

- `a-fake-can-only-fail-the-ways-you-have-seen` — the fake's ceiling is the
  failures already understood; this pattern is the time-axis of the same
  problem: the fake's *placement* also rots.
- `green-tests-can-mirror-the-same-guess` — self-consistency masquerading as
  verification; here the consistency is between a stub and a routing fact
  that stopped being true.
