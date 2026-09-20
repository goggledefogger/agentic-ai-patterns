---
type: pattern
date: "2026-08-26"
source: The household-agent repo — the hub build-context provenance gate (PR #271), which passed a seven-row verdict matrix against the live Pi and then refused every deploy forever, fixed the same evening in PR #273
tags:
  - testing
  - verification
  - anti-pattern
  - guards
---

# A Verdict Matrix Proves the Function, Not the Feed

A decision function was reused unmodified for a new guard, and every state it can return was exercised against the real production host before a line of the guard was written:

```
pi 82b6ffeb ≠ marker 532f3be0 ≠ repo 78ffd886
skip · deploy-advance · deploy · probe-failed · deploy · skip · deploy-forced
```

Seven rows, all correct, run against real hashes read over SSH minutes earlier. The guard shipped. It then returned `skip` on every deploy that would ever run, and the drift report it fed said DRIFTED permanently.

The matrix was not wrong. Each row was produced by **substituting** one hash for another to reach that state: pass the marker in as the Pi hash and you get `deploy-advance`, pass the repo hash in and you get `deploy`. Substitution is precisely the thing production cannot do. The one question the matrix never asked was whether the real feed could ever *deliver* those combinations, and it could not: the marker held a **repo** hash while the guard measured the **Pi**, and the two could never be equal on that host, because the Pi's `static/` carries files the repo has never had and the deploy copies without deleting.

## The Pattern

Exercising every branch of a pure function tells you the function is correct. It tells you nothing about which branches its callers can actually reach. Those are separate claims and the first one feels so much like proof that nobody goes looking for the second.

Three checks, cheap next to the cost of shipping a guard that cannot pass:

1. **For every state in the matrix, name the production event that produces it.** Not the substitution you used, the event. "The Pi is byte-identical to the last deploy" is an event. "I passed `$MARKER` as the first argument" is not. A row with no nameable event is unreachable, and an unreachable pass-row means the guard is welded shut.
2. **Ask what would have to be true for the good state to occur, and go measure that.** Here: does `marker == pi_hash` ever hold? One command answers it, and it answers `no, structurally`, which is a design bug rather than a test failure.
3. **Run the guard once against untouched production and read the verdict.** Not a fixture, not a substitution. The gate refused a deploy that should have been ordinary, and that single run is what surfaced it.

## The sibling defect it exposed

The root cause is worth naming separately, because it is what made the matrix unreachable: **one stored artifact was answering two different questions.** A single hash file was read by an existing rebuild-skip optimization asking *did my build context change since I last deployed?* and by the new guard asking *has anyone written to that host since I did?* The first wants a repo-side hash, the second a host-side one. They were never the same value, and conflating them made one consumer permanently wrong while the other kept working, which is exactly the shape that survives review. The fix was two markers, each answering one question, not a cleverer comparison.

## Why it stays invisible

- **A matrix is the most convincing artifact in a review.** Seven rows of measured verdicts read as thoroughness, and the reviewer's attention lands on whether the verdicts are right, never on whether the inputs can co-occur
- **The failure is fail-safe, so it does not look like a failure.** A guard stuck at `refuse` protects everything, breaks nothing loudly, and reads as a strict guard rather than a broken one until someone needs it to pass
- **Reuse hides it.** The decision function was already trusted and already tested, so the new caller inherited that trust, and the untested part was the only part that was new: the wiring feeding it

## When to Use

Any time a guard, router, or state machine is assembled from a tested decision function plus new plumbing. The tested half is not the risky half. Applies with particular force when you reuse an existing verdict function for a second purpose, because reuse is what smuggles the old function's assumptions about its inputs into a caller that does not satisfy them.

## Related

- `verification-needs-a-negative-control.md` — its third-order case (a control leg whose negative case cannot occur) is this defect seen from the other side: there, the control could not fail, here the subject could not pass
- `a-source-assertion-pins-your-belief-not-the-behavior.md` — the sibling for source-level guards: that one pins spelling instead of runtime behavior, this one pins logic instead of reachability
- `half-gate-whole-verdict.md` — the one-artifact-two-claims defect above, in its original form

## Source

The household-agent repo, 2026-08-25. A hub deploy phase had shipped its whole build context with no provenance gate of any kind, silently destroying host-side edits. The gate added to fix it reused `agents_md_deploy_decision` (already shared by two other guards) and the marker file the deploy already wrote. All seven verdicts were measured against the live host before implementation. The guard then refused every deploy, because the marker recorded a repo hash and the guard measured the host. Split into two markers in PR #273 the same evening, and the spec's change log now records that the matrix proved the function while the feed went unexamined.
