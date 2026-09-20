---
type: pattern
date: "2026-08-02"
source: The household-agent repo — catchup.sh printed a bare `2026-08-02 06:36` header from a UTC controller above hundreds of Pacific timestamps from the Pi. Three separate TZ-handling comments already existed in that file; the same subtraction was still gotten wrong three times, most recently turning a sub-hour billing blip into a reported 8-hour outage with no alerting (both halves false).
tags:
  - documentation
  - verification
  - monitoring
  - anti-pattern
  - timezone
---

# Print the Answer, Not a Warning About the Question

If readers keep deriving the same wrong value from your output, put the derived value *in the output*. A comment in the source cannot reach someone reading the render, and a warning is a request that the reader do arithmetic correctly — which is the thing they have already demonstrated they will not do.

`catchup.sh` opens with its own clock and then prints hundreds of timestamps from a remote host. The header read:

```
WORKSHOP CATCHUP — 2026-08-02 06:36
```

No zone. Run from a UTC controller against a Pacific host, every timestamp below that line is seven hours off it. Subtracting across the two produced three wrong conclusions in one week: a 7h offset reported as "the 6h cron is not landing"; the identical bug inside the script's own coverage checker; and a `<1h` service interruption reported as an 8-hour outage during which "nothing alerted" — when in fact zero user requests had failed and the alerter had fired correctly three minutes in.

The file already carried TZ-handling comments at three separate call sites. All three were present, and had been read, while the third failure happened.

## Why the warnings did not work

**The warning and the mistake live in different media.** The comments were in the source; the mistake was made by someone reading the output. Those are different people, or the same person in a different posture, and nothing carries between them. A reader scanning 350 lines of diagnostic output is not holding the script's internals in mind — that's the entire reason the output exists.

A warning also has the wrong shape. It says *be careful here*, which delegates the work back to the reader at exactly the moment they are least equipped to do it: mid-scan, pattern-matching, looking for something else. Every warning is a small tax on attention that pays out only if the reader both notices it and correctly performs the thing it warns about.

The third instance is the proof. This is not "we forgot to warn." It is "warning does not work," established three times.

## The Pattern

**Compute the ambiguous thing and render it, next to the data it applies to.**

```
WORKSHOP CATCHUP — 2026-08-02 06:36 UTC

  ...

  Clock: 2026-08-02 00:11:19 PDT  ←  timestamps below are THIS clock
         catchup header is 07:11 UTC, 7h ahead — do not subtract across them
```

Three properties make it work where the comments didn't:

1. **Both values appear together.** Not "note the timezone difference" but both clocks, rendered, adjacent. Nothing is left to derive.
2. **The delta is computed, signed, and labelled.** `7h ahead` is the number that was being gotten wrong. It is now a fact in the output rather than an exercise.
3. **It sits where the affected data is**, not in a preamble. The line precedes the timestamps it governs.

**Stay quiet when there is nothing to say.** Same-zone runs collapse to a one-line `(same as this box — no skew)`. A line that renders identically whether or not the hazard exists is a line readers stop seeing — the failure mode being escaped, reintroduced.

**Fail loud when the value can't be computed.** An unreachable clock prints `⚠ TZ skew UNKNOWN`, never nothing. A missing line reads as "no skew," which is a claim, and the wrong one.

## Generalising past clocks

The shape is: *output that requires a derivation the reader will get wrong.* Timezones are the common case; the family is larger.

- **Units.** Print `1.4 GB (1,503,238,553 bytes)`, not bytes with a comment explaining they're bytes.
- **Currency and rates.** Print the converted figure and the rate used, not a footnote that amounts are in USD.
- **Relative vs absolute time.** Print `3 days ago (2026-07-30)`. "Last updated 3 days ago" in a cached page is wrong the moment it's cached.
- **Percentages.** Print the numerator and denominator. `90% (1,335/1,375 chars)` cannot be misread; `90%` invites a guess about the base.
- **Counts under a cap.** Print `showing 50 of 1,284`, never `50 results`.

The test: if you are about to write a comment, a footnote, or a doc paragraph explaining how to interpret a value correctly — that explanation is a computation, and computations belong in the output.

## When a warning IS the right tool

Warnings earn their place when there is no value to compute: a genuine judgement call, a known-unknown, a hazard whose resolution depends on context the program doesn't have. `doc-warning-preamble.md` is the correct pattern for a living doc that will drift, because "which claims are stale" is not derivable at render time.

The distinguishing question: **could the program have computed the thing it is warning about?** If yes, the warning is a bug report against the renderer. If no, warn — and give a verification recipe rather than just a caution.

## Verification

Render both branches and read them. The hazard case must show the computed value; the no-hazard case must be quiet; the can't-compute case must be loud. Then pin the derivation itself with tests — a wrong delta printed confidently is worse than no delta, because it launders a bad number through the fix. Test the boundaries the derivation will actually meet: for clocks, both daylight and standard offsets (a delta hardcoded from a summer observation is an hour wrong for a third of the year), a negative skew, a half-hour zone, and the leading-zero parse (`"0800"` is 8h, not an octal error).

## Adjacent Patterns

- `doc-warning-preamble.md` — the complement, and the boundary. Warn when the thing is genuinely underivable; compute when it isn't. Reaching for a preamble where a computation would do is this anti-pattern.
- `declared-presence-beats-host-clock.md` — the other clock failure. That one is about *whose* clock (a user's body vs a machine's location); this one is about two machine clocks both being correct and a reader subtracting across them.
- `unrun-checks-read-as-passing.md` — same root: a reader drawing a confident conclusion from output that could not support it.
- `verdicts-from-structured-signals.md` — the machine-facing sibling. There a program reads the wrong layer; here a human does.
- `framework-gotcha-comments.md` — where a source comment IS the right home, because the audience is the next editor of that line, not a reader of its output.

## Source

The household-agent repo — `workshop/catchup.sh` header and per-host clock line, `scripts/tests/test-catchup-tz-delta.sh`, the household-agent repo PR #190, 2026-08-02.
