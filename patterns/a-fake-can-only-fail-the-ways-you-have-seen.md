---
type: pattern
date: "2026-08-02"
source: The personal agent — walk-and-talk phone bridge, a 23-test suite stayed green through a bug the user found by ear on the first walk, 2026-08-02
tags:
  - testing
  - verification
  - anti-pattern
  - observability
---

# A Fake Can Only Fail the Ways You Have Already Seen

Every mock, stub, and fixture is written after something went wrong. That is its origin and its ceiling. The failure it imitates is a failure someone already understood well enough to describe — so the suite built on it is, by construction, a regression test for the past. It will hold the line beautifully. It will also stay green through the next thing, and its greenness is what makes that dangerous.

The trap is not that the fake is inaccurate. It is that the fake is **narrower than reality in a direction nobody wrote down**.

## The Problem

`walk-and-talk`'s phone bridge had a 23-test sound-discipline suite, built over several days of live walks. It was a good suite. Every test simulated a hostile condition — a mic that dies instantly, rapid state churn — and asserted the page stayed quiet and bounded. It caught real regressions.

Its speech-recognizer mock:

```js
FAILING_MIC = """
  window.webkitSpeechRecognition = window.SpeechRecognition = class {
    constructor(){ window.__srMade++; }
    start(){ const s=this; setTimeout(()=>{ s.onend && s.onend(); }, 50); }
    abort(){}
  };
"""
```

A recognizer that dies 50 milliseconds after starting. That was written to reproduce the chime-storm bug: a mic dying instantly, over and over, restarting each time. It reproduced it perfectly and the fix held.

Then on a live walk the user's sentences started arriving cut in half. The cause was a recognizer that died **1.8 seconds in, with words already captured** — at which point the page committed the half-sentence as a finished turn. All 23 tests were green throughout, and stayed green for an entire afternoon of development on that exact file.

They could not have failed. The mock dies at 50ms, before any result is ever delivered, so **the state that carried the bug — a dying session with pending words — is not expressible in the harness.** Not untested. *Unrepresentable.* No amount of additional tests written against that mock would have found it, because the shape of the fake had already decided which bugs were sayable.

The follow-up made the same point twice. A new test was added to bound the retry count:

```python
ok1l2 = (r["retriesAttempted"] <= 3 and ...)
# observed: retriesAttempted == 0
```

It passed. It would have passed with the entire retry mechanism deleted, because the mock's session had auto-died before the test drove a result, so the code path under test never ran. A test written *specifically* for the new behaviour, against the same fake, measuring nothing.

## Why It Persists

The fake is normally the most trusted object in the suite, for three reasons that all sound like virtues:

- **It was validated once.** It genuinely reproduced a real bug, so it carries the authority of having worked.
- **It is shared.** Twenty-three tests depend on it, so changing it feels risky and re-examining it feels gratuitous.
- **Its narrowness is invisible.** A missing test leaves a gap someone can notice. A too-narrow fake leaves no trace at all — the bug it cannot express simply never occurs to anyone, because there is no place to write it down.

That last one is the whole pattern. The harness is not just where tests run; it is the vocabulary in which failures can be *stated*. Anything outside the vocabulary is not merely untested, it is unthinkable.

## The Move

**Build the fake as a control surface, not a simulation.** A simulation encodes one remembered behaviour. A control surface lets a test declare any behaviour, including tomorrow's:

```js
// not: dies at 50ms, always
window.__sr = { die(), result(text, isFinal), speechStart(), speechEnd(), autoDie: true }
```

Keep the old behaviour as the default so the existing suite does not move — the upgrade is additive, and additive is what makes it affordable.

**Assert on the system's own event stream, not on proxies.** The bridge already emitted a semantic log — `utt-hold`, `utt-commit`, `utt-retry`, `speak-deferred` — which the production diagnostic tool read, and which named the bug in one command once someone ran it. The tests, meanwhile, counted constructor calls. Two languages for one system, and the translation between them is where `retriesAttempted <= 3` lost its meaning. When tests assert on the same events the production tool renders, **a real-world failure can be replayed as a fixture instead of hand-translated into counter arithmetic.** That is what closes the loop between what users hit and what the suite can say.

**Treat an assertion with slack as a smell.** `<= 3` over a number nobody printed will pass at zero. Prefer an exact expectation, and print the observed value next to it.

**And know what green means.** A green suite says "none of the failures we have already imagined are present." That is worth a great deal and it is not the same sentence as "this works." For anything with a human on the other end, the real exercise — the actual walk, the actual device — is not ceremony after the tests pass. It is the only part that can surprise you.

## Related

- [[health-check-that-never-exercises]] — the same blindness in monitoring: a check that measures everything except the work.
- [[verification-needs-a-negative-control]] — a test that cannot fail proves nothing; make it fail first.
- [[an-event-is-not-a-cause]] — the bug this suite could not see, and why the event stream is the honest place to assert.
- [[green-tests-can-mirror-the-same-guess]] — same failure shape from the other side: not a fake too narrow to say it, tests derived from the same invention as the code.
