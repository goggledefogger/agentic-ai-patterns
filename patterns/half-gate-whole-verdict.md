---
type: pattern
date: "2026-08-01"
source: Walk-and-talk voice-loop-check.sh — the readiness gate certified a voice loop off a typed reply (2026-07-31 live session)
tags:
  - code
  - safety
  - anti-pattern
  - verification
---

# Half Gate, Whole Verdict (Anti-Pattern): Evidence for One Axis, a Claim Covering Both

A walk-and-talk session must prove two independent things before it briefs: the assistant can be **heard** (output), and the user can **reply** (input). The gate enforced the first with real evidence — `speak.sh` drops a `handshake-spoken` marker, and `--confirm` refused without it. The second it took on faith. Its own text read:

> "The moment ANY reply arrives, that IS the proof submit works."

So when the author **typed** "ready", the gate printed **"Two-way loop CONFIRMED"** while nothing on the Mac was listening — Voice Control off, no helper running, the built-in dictation being push-to-talk. The gate built to prevent a one-way brief waved one through, and every downstream decision inherited its confidence.

This passes all three tests in [[decorative-gate]]. It *can* refuse (and did, on the output axis). It *fails closed*. Its verdict *is* derived, not a literal. It is still unsound — because the derivation covers one axis and the verdict names two.

## The Pattern

**A verdict may be no broader than its narrowest piece of evidence.** When a control certifies a compound claim ("two-way", "end-to-end", "synced", "authenticated and authorized"), each conjunct needs its own evidence, and the strong half will hide the weak one — a confident sentence reads as covering everything in it.

Three checks:

1. **Decompose the verdict string.** For every claim in it, name the artifact that proves that claim. A conjunct with no artifact is prose wearing a gate's uniform.
2. **The evidence must distinguish, not merely exist.** "A reply arrived" is real evidence — of *a* channel. It cannot tell voice from keyboard, so it cannot certify voice. Ask: what *other* world produces this same signal? If the answer is "the failing one", the signal is not evidence.
3. **Make the caller state which channel it saw.** The fix was `--confirm --via voice|typed`. Typed is legitimate and still confirms — it just records `via=typed` and says out loud that voice is unproven. A bare `--confirm` is refused. Forcing the evidence to be *named* is what stops the silent widening, and it costs one flag.

## Corollary: the witness must be the thing that holds the resource

The gate could not answer "is anything listening?" by inspecting the machine. All three probes were dead ends: `ioreg` returns nothing on Apple Silicon, CoreAudio's `kAudioDevicePropertyDeviceIsRunningSomewhere` is documented blind on Bluetooth mics (the earbud case), and macOS Voice Control's state is unreadable from the CLI.

The working design inverts it: **the component holding the microphone reports its own state**, to a `listener-heartbeat` file, re-stamped every ~10s so a dead reporter goes *stale* rather than lying. Prior art in the author's own `work-assistant`, which binds its "🎤 Listening…" indicator to `isConnected` after `getUserMedia` resolves rather than to an assumption.

Two properties fall out, both free:
- **Portable by construction.** No OS APIs, so macOS, Linux and Windows behave identically, and the reporter can be a browser tab on a different machine entirely.
- **The reason travels with the state.** `no-recognizer`, `mic-denied`, `not-started`, `muted` each carry a different one-line fix. "Voice isn't working" is not actionable; "that browser can't do speech, open it in Chrome" is.

Same family as [[opaque-write-needs-a-read-back]] (don't infer success, read it back) and [[self-reporting-staleness-check]] (a reading must carry its own age).

## Why It's Hard to See

The asymmetry is invisible from inside. The half that *is* enforced feels like rigor, and rigor in one place reads as rigor overall — the more machinery around the strong axis, the more the weak axis is assumed to be covered too. This gate had a marker file, four exit codes, and a paragraph in the skill doc about why prose controls fail. All of it was about output.

Related: an assertion written once, unqualified, becomes load-bearing for everything downstream — the personal agent's `CLAUDE.md` carried "the hook is the binding control" for weeks before measurement showed it was a speed bump on Bash. See [[stale-pointer-asserts-confidently]] and [[unrun-checks-read-as-passing]].

The author's framing, which is the shortest version of the whole pattern:

> "We need the full loop test to actually work because it didn't."

A test that cannot fail is worse than no test — it manufactures the confidence that stops anyone looking.
