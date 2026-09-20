---
type: pattern
date: "2026-07-31"
source: The voice stack (a personal skills repo) — SpeakOptions.rate was words-per-minute at the port; macOS `say` consumed it as wpm and Kokoro consumed the same number as a speed multiplier, so a default SpeakOptions asked Kokoro for 140x speed. Found by reading the adapters side by side, not by a failing test.
tags:
  - architecture
  - interfaces
  - adapters
  - anti-pattern
  - verification
---

# Units Belong to the Adapter, Not the Port

A port that carries a number in one backend's native units is not an
abstraction — it is that backend's API with extra steps. Every other adapter
then either converts silently, ignores the units, or reads the number as its
own, and the third case produces no error at all.

## The failure

The voice stack's engine port passed `rate: float = 140.0`, documented as words per
minute. Two adapters implemented it:

- **macOS `say`** takes absolute wpm. `say -r 140` is correct.
- **Kokoro** takes a *speed multiplier*, where 1.0 is natural pace.

Both read `opts.rate` and used it directly. So the default options asked Kokoro
to speak at **140× speed**. Nothing raised: the types matched, the field name
matched, the tests passed, and each adapter was individually correct against its
own backend's documentation.

It survived review twice because every artifact agreed with itself. The
dataclass said wpm. The `say` adapter honored wpm. The Kokoro adapter used the
field it was given. The bug lived *between* three correct files.

## The pattern

**The port carries a dimensionless, engine-neutral quantity. Every adapter
converts at its own boundary, from a baseline it names.**

```python
# port: a multiplier of natural pace, 1.0 = natural. No backend's units.
rate: float = 1.25

# say adapter: converts, and names the baseline it converts from
SAY_BASE_WPM = 140.0
cmd += ["-r", str(int(SAY_BASE_WPM * clamp_rate(opts.rate)))]

# kokoro adapter: already a multiplier, still clamped at the boundary
speed = clamp_rate(opts.rate)
```

Three properties make this hold rather than merely look tidy:

**1. The neutral quantity has a fixed anchor.** "A multiplier of natural pace"
is meaningless unless 1.0 is defined per adapter. Each names its own baseline in
code, so `rate=1.0` produces the same *perceived* result everywhere even though
the numbers differ. Without the anchor you have swapped one ambiguity for a
vaguer one.

**2. Clamping happens at the adapter, not the caller.** Bounds are the system's
judgment, not the model's — Kokoro documents no formal speed range at all. A
clamp at the port would be a lie about the backend; a clamp per adapter is an
honest statement about what the product will emit.

**3. Capabilities advertise the same shape.** The mismatch hid partly because
one adapter reported `rates=(80.0, 300.0)` and another `(0.5, 2.0)` and nothing
compared them. A cross-adapter test that asserts every range *brackets 1.0 and
is not a wpm scale* catches the reintroduction — and it must be shape-based, not
equality-based, because a backend may legitimately support a wider range (OpenAI
TTS does 0.25–4.0).

## Verification

The test that matters is the cross-adapter one, and it must fail against the old
code. A per-adapter test cannot find this class of bug: each adapter was already
passing its own.

Two controls worth writing explicitly:

- Feed the *old default* through the new clamp and assert it does not pass
  through unchanged (`clamp_rate(140.0)` must not be 140.0). This is the
  regression, named.
- Assert the natural-pace case is preserved end to end (`rate=1.0` still yields
  the spec'd 140 wpm on `say`). Changing units must not quietly change behavior,
  or the fix ships a second bug wearing the first one's clothes.

When updating the tests that encoded the old contract, say so out loud. Those
tests were not wrong when written; the contract changed deliberately. Editing a
test to match new behavior is legitimate exactly when the behavior change is the
point and illegitimate when it is a workaround — and the difference must be
visible in the diff, not in someone's memory of the conversation.

## Where else this shape appears

The tell is a shared interface carrying a **bare number with a unit in its
docstring** rather than in its type or its name:

- Timeouts: seconds at one call site, milliseconds at another. The classic.
- Temperature/top-p passed straight through to models whose ranges differ.
- Money as a float, dollars in one service and cents in the next.
- Sizes: bytes vs KiB vs "human" strings across a storage abstraction.
- Coordinates: degrees vs radians across a geometry boundary.

In each, the port should carry the neutral quantity (or a typed unit), and each
adapter should convert from a baseline it names in code.

## Adjacent Patterns

- **`framework-gotcha-comments.md`** — the converted line earns a two-line
  comment saying why the naive pass-through is wrong, or the next reader
  "simplifies" the conversion away.
- **`count-the-source-not-the-survivors.md`** — sibling failure mode: there, a
  number is correct about the wrong population; here, a number is correct in the
  wrong units. Both produce confident, well-formed, wrong output.
- **`verification-needs-a-negative-control.md`** — the cross-adapter test only
  counts if it fails against the pre-fix code.
