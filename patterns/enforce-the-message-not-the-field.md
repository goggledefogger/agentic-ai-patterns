---
type: pattern
date: "2026-08-02"
source: The personal agent — the walk-and-talk Stop hook guaranteed the phone link reached the walker, asked for "this bare URL", and got a walk that opened with a one-word status token and a naked link
tags:
  - controls
  - hooks
  - ux
  - tone
  - anti-pattern
---

# Enforce the Message, Not the Field

A rule that keeps getting walked past becomes a control — a hook, a gate, a
template that refuses to let the work continue until the required thing is
present. That is the right move. But the moment a control prescribes *what to
say*, its own wording becomes the artifact the user reads. It is no longer
checking output; it is authoring it. A control that asks for the minimum
required field will get exactly that field, and nothing around it.

## The Problem

A Stop hook was built to fix a real failure: setup finished on the Mac and the
walker, on a phone, saw nothing, because every step had run in tool output that
the UI collapses ([[announce-the-move-in-the-old-room]]). The hook blocks the
end of a turn until the bridge URL appears in an assistant *text* block. It
works — first walk after it shipped, the URL landed.

What the walker actually saw was the word `staged` twice and a bare link. His
words: *"all I see is the word staged twice and some weird output."* The hook's
message had said **"Put this bare URL in your REPLY TEXT"**, so a bare URL is
what it got, at the one moment in the whole session where warmth is the product
rather than a garnish on it.

Nothing failed. The gate passed, the delivery was real, and the first contact
was cold — because the gate had already decided the wording and had asked for a
datum.

## The Pattern

**When a control dictates user-facing output, make it carry the finished
message, not the required field.**

Three parts:

1. **Emit the whole line.** The hook now pulls the canonical welcome from the
   script that already owns it (`voice-loop-check.sh --handshake-text` — the
   same line spoken at the handshake) and hands over welcome-plus-link as one
   block to paste. One source of truth for the words, reused rather than
   retyped ([[one-authority-for-repeating-behaviors]]).
2. **Name the exclusions.** "No status tokens, no recap of the setup, no
   plumbing." A control that only says what must be present leaves everything
   else to whatever the assistant was mid-thought about — which is usually
   scaffolding.
3. **Keep the fallback warm.** If the canonical source cannot be reached, fall
   back to a fixed human line, never to the bare field. A degraded path is
   still a path the user reads.

## When It Shows Up

Anywhere a machine prescribes words a human will read: Stop/PreToolUse hooks
that tell an agent what to say, commit-message templates, error-copy
constants, PR-description scaffolds, the refusal text in a policy gate. The
tell is a control whose message names a *variable* ("put the URL", "include the
ticket ID") rather than a *sentence*.

It is the tone-side twin of [[decorative-gate]]: there, the passing branch was
reachable without the thing being true; here, the passing branch is reachable
while the thing is technically true and useless. Both come from specifying the
check instead of the outcome.

## Related

- [[announce-the-move-in-the-old-room]] — the failure this control was built for
- [[decorative-gate]] — a check whose pass does not mean what it claims
- [[one-authority-for-repeating-behaviors]] — why the welcome has one home
- [[personality-as-discipline]] — register is a constraint, not decoration

## The Rule of Thumb

If a control tells an agent what to say, read its message as if you were the
user. That text *is* the product.
