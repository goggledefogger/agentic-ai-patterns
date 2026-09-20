---
type: pattern
date: "2026-08-01"
source: The personal agent — walk-and-talk: three separate retry loops (each locally reasonable) compounded into an infinite chime storm on a real phone; fixed by one governor, not better loops
tags:
  - resilience
  - rate-limiting
  - anti-pattern
  - agents
  - audio
---

# One Authority for Repeating Behaviors

A behavior that can repeat — a retry, a reconnect, a heal, a restart — always
starts as one reasonable line: *try again in 250ms*. Then a second author adds a
watchdog that heals every second. A third adds a fast-retry cap. Each policy is
defensible alone; together they compound, because **none of them can see the
others' attempts**. The harness measured it: a mic that died instantly drew 173
restart attempts in 30 seconds from code where every individual loop was
"bounded."

Patching loop-by-loop is whack-a-mole with a tell: if you have fixed the same
runaway three times in one day with three different local counters, the design is
missing a layer, not a constant.

## The Pattern

**Route every instance of the repeating behavior through ONE gate with ONE
budget, and give exhaustion a terminal, announced state.**

```js
const BUDGET = 6;                      // starts per rolling minute, ALL sources
let starts = [], resting = false;
function governorAllows() {
  if (resting) return false;
  starts = starts.filter(t => Date.now() - t < 60000);
  if (starts.length >= BUDGET) { enterRest(); return false; }
  starts.push(Date.now());
  return true;
}
```

Five properties do the work:

1. **One choke point.** The constructor/starter itself calls the governor, so a
   new caller added next month is governed automatically. Policies attached at
   call sites rot; a policy inside the only door cannot be bypassed.
2. **A rolling budget over all sources combined** — not per-loop counters. The
   storm was never one loop; it was the sum.
3. **Exhaustion is terminal and announced, once.** When the budget is spent the
   system says so (one line, one time) and goes quiet. Silent retrying and
   silent giving-up are both failures; the announcement is what makes autonomous
   quitting acceptable.
4. **Only a human gesture refills the budget** (plus genuine success earning it
   back). Time-based refill just schedules the next storm.
5. **Schedulers get generation tokens.** A repeating timer chain carries the
   generation it was born with and dies when stale — a second live chain becomes
   structurally impossible, instead of improbable. (The sibling bug: start/stop
   churn left old `setTimeout` chains alive, stacking audio gain ramps into a
   loud crackle.)

## The Refinement the Healthy Case Taught

**Budget churn, not uptime.** Hours after shipping, the governor caused its own
outage: on the platform in question a *healthy* session gets cycled by the OS
every ~6 seconds, each cycle spent budget, and a long quiet stretch exhausted it
mid-conversation — the safety mechanism became the failure. The fix: an instance
that *held* for a meaningful time (here, 3s+) earns its successor a free start;
only fast deaths — the actual storm signature — pay. A rate limit that cannot
tell breathing from thrashing will eventually rate-limit breathing.

## The Test That Finds It

Simulate the hostile dependency — the one that fails *instantly*, forever — and
count total attempts across a window. Healthy-path tests can never catch this
class: every loop behaves when the thing it retries succeeds. And when the
repeat has a real-world side effect the harness cannot perceive (Android chimes
on every recognition start; a desk test is deaf), the count IS the sound —
log every attempt as an event, then assert on the log.

## Related

- [[convergent-standup-sloppy-quits]] — the start-side sibling
- [[health-check-that-never-exercises]] — the dependency that lies about holding
- [[log-the-verdict-not-the-volume]] — attempts as first-class logged events

## The Rule of Thumb

The second time you add a backoff to the same behavior, stop: you are writing
the governor's job into another call site. Count the places that can initiate
the behavior; if the answer is more than one, the budget belongs in the door,
not in the callers.
