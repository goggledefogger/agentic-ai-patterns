---
type: pattern
date: "2026-09-09"
source: The personal agent session 2026-09-09 — adding a liveness check to a backup job's precondition made five existing selftests start passing for the wrong reason. The suite stayed green the whole time; the drift was only visible because one unrelated control case failed and forced a read of the others.
tags:
  - testing
  - verification
  - refactoring
  - anti-pattern
  - agent-safety
---

# A New Guard Relocates the Failure Mode of Every Test That Crosses It

A backup job's precondition checked `ismount(root)`. A dropped userspace mount passes `ismount` — it stays in the mount table and fails only on read — so a liveness probe was added: actually `listdir` the root, and skip if it raises.

The new case passed. So did the whole suite. But five *existing* cases were now passing for a reason they never claimed: each had faked a mounted root that did not exist on disk, so each one now tripped the **new** check and returned the expected skip code without ever reaching the condition it was written to prove. The test named "mounted but nothing under it must skip" no longer tested that at all.

It surfaced only by accident. One control case — "source present, tool missing, must fail loudly" — expected a *failure* code, so the new guard's skip broke it visibly. Had that control not existed, five tests would have gone on reporting green while proving nothing.

## The Pattern

Adding a check upstream of existing assertions silently changes what those assertions test. The suite does not go red; it goes green for new reasons.

1. **After adding any guard, re-read every test whose path crosses it.** Not the tests you wrote — the ones that were already there. Ask of each: does it still reach the condition in its own name? A test's name is a claim about which branch it exercises, and a new early return invalidates that claim without touching the test file.
2. **Give each case an explicit stub for the new condition**, so it still isolates the one thing it names. If the new guard reads the filesystem, the cases that are not about the filesystem must inject a passing filesystem — otherwise they are all now testing the guard.
3. **Mutation-prove in both directions.** Delete the new guard and confirm *exactly* the new assertion fails. If old assertions fail too, they were depending on it. If nothing fails, the new test proves nothing.
4. **Keep at least one case that expects a non-skip outcome.** A suite whose cases all assert the same defensive result cannot distinguish "correctly refused" from "refused for the wrong reason, or for every reason." The control that expects success or loud failure is what makes the refusals meaningful — here it was the only reason the drift was ever seen.

## Why It Bites Specifically

- **Green is the failure mode.** Every other refactoring hazard announces itself with a red test. This one removes coverage while improving the appearance of the suite, and the commit that does it reads like hardening.
- **It scales with how defensive the code is.** The more preconditions a function accumulates, the more of its tests short-circuit at the first one, until a suite of ten cases is really one case run ten times. Defensive code and test decay grow together.
- **Skip codes make it invisible.** When several distinct conditions all return the same "skipped" value, an assertion on that value cannot tell you *which* condition produced it. Distinct reasons need distinct return values, or at minimum distinct messages the test can assert on.
- **The agent version is worse.** An agent adding a guard sees the suite pass and moves on, having just deleted four-fifths of the coverage it was relying on to move safely. Nothing in the transcript looks wrong.

## Related

- `a-fake-can-only-fail-the-ways-you-have-seen.md` — the fidelity of the stub; this pattern is about the *reachability* of the case behind it.
- `a-verdict-matrix-proves-the-function-not-the-feed.md` — the same instinct applied to which inputs a verdict is proven over.
