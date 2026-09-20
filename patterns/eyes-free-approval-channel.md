---
type: pattern
date: "2026-07-31"
source: Live walk-and-talk field session, 2026-07-31 — a voice-driven agent session hit a permission dialog while the user was away from the screen that showed it
tags:
  - agents
  - permissions
  - voice
  - safety
  - anti-pattern
  - claude-code
---

# An Approval Crossing a Channel the Approver Can't See Needs Its Own Channel

A voice-driven agent session hit a permission dialog. The session stopped dead. Nothing surfaced on the user's phone. The only way through was the machine they'd walked away from — but they couldn't see the screen, so they couldn't clear it.

The tempting fix is to say "yes" into the same mic that drives the rest of the session. That's a trap, for two independent reasons:

1. **The voice/text input path injects text with a trailing Enter.** Dictating "yes" is a blind keypress at a menu you cannot see. Option 2 in these dialogs is routinely "yes, and don't ask again" — a standing approval nobody granted, landed by cursor position, not by decision.
2. **Speech recognition has separately been observed producing a clean, plausible sentence the user never said.** A fabricated "yes," fed straight into a safety-critical approval, would land with the same confidence as a real one and no way to tell them apart after the fact.

Both failures are silent. Nothing in the transcript flags a misfire — the dialog was cleared, a choice was made, and the log reads exactly like informed consent.

## The Pattern

The approval channel must be structurally separate from the content channel, and it must carry the verbatim choices — never a synthesized Yes/No.

1. **Publish the dialog's real question and real option labels out-of-band.** Never collapse a multi-option menu into a binary the voice layer can answer with one word. If the dialog offers "Allow once / Allow and don't ask again / Deny," all three travel, not a summary.
2. **Speak that a decision is waiting**, not what the decision should be. The voice output's job is to raise the flag, not to pre-chew the choice.
3. **Render one button per option** on the out-of-band surface (phone, push notification, whatever reaches the user away from the machine).
4. **Accept a choice only from an explicit tap on a dedicated endpoint that validates against the currently-pending options.** A tap that doesn't match a live option is rejected, not coerced into the nearest one.
5. **Refuse the text/voice channel entirely while a dialog is up.** Don't let "yes" typed or spoken into the normal conversation resolve a pending approval, even if it would be the right answer. The channel that can't see the options doesn't get to answer for them.

## Why the Content Channel Can't Carry This

The content channel (voice, chat) is built for open-ended intent, which is exactly what makes it unsafe here: it accepts a free-form utterance and turns it into an action, with no way to check that utterance against a fixed, correct set of choices. An approval is a closed-form decision — one of N known options — and closed-form decisions want a channel that can only express those N options and nothing else. Mixing the two lets an open-ended channel resolve a closed-form question, which is where both the blind-keypress and the hallucinated-consent failures come from.

## Watch-outs

- **A synthesized "yes/no" is already a loss of information**, even before the input pipeline mangles it. If the real dialog has more than two options, reducing it to a binary throws away the option the user would have picked.
- **"It cleared the dialog" is not evidence it cleared it correctly.** Verify against the option the dialog actually recorded, not against the fact that the dialog is gone.
- **This applies to any hands-off input mode**, not just voice — a background agent's stdin, a webhook replaying a stored command, anything that can inject a keystroke without a human reading the current screen state.

## Adjacent Patterns

- `subagent-cannot-consent.md` — the sibling failure one layer down: a subagent has no channel to reach the human at all. This pattern is about a human who has a channel, but the wrong one.
- `approval-scope-invisible-to-gates.md` — a gate can't tell an approved action from an unapproved one when scope isn't carried into the brief; here the gate can't tell a real answer from a fabricated one when the input channel isn't trustworthy.
- `decorative-gate.md` — a dialog that accepts any injected "yes" without validating it against the live pending options is a gate that looks real and refuses nothing.
- `escape-hatch-is-the-denied-tool.md` (2026-08-18 addendum) — the same channel-separation applied to an agent's *own* staged work: when the lawful last hop was "a human types the command", the paste itself became the friction, and the fix was a loopback-only approve/deny card rather than a standing grant.

## Source

Live walk-and-talk field session, 2026-07-31.
