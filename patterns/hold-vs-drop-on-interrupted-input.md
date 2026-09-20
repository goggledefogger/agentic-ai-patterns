---
type: pattern
date: "2026-08-02"
source: The personal agent — walk-and-talk phone bridge, deciding what happens to a walker's half-spoken sentence after the system interrupts him, 2026-08-02 (ux-debt 57)
tags:
  - architecture
  - voice
  - input
  - reliability
---

# Holding Is the Default, Staleness Is the Bound

Something interrupts a user mid-input. A reply starts playing and takes the microphone; a route change resets the audio session; the platform times out. Whatever the reason, you are now holding half a sentence and you have to decide what it is.

There are three options and only one of them is right, which is easy to see once they are side by side and surprisingly easy to get wrong one at a time.

## The Problem

**Send it.** This is what the bridge did, and it is the worst of the three. A truncated clause goes downstream as a completed turn — *"okay so the other thing is I want"* — and the assistant answers it. The user is not merely ignored; they are **misquoted**, and then responded to on the basis of the misquote. Every downstream system now believes something about them that is false.

**Drop it.** Safer and still bad. The user finishes a sentence into a dead channel, and nothing tells them so. They cannot see that the words are gone. Re-forming a thought you already completed is the most expensive thing a voice interface can ask for, and it is asked silently, so the interface merely feels unreliable in a way nobody can point at.

**Hold it.** Keep the words uncommitted and visible, let the next capture session continue the same sentence. The interruption becomes a pause instead of an ending.

Holding wins because of an asymmetry in what the two directions cost. The system's own output is renewable — a lost reply can be asked for again. The user's speech is not: no transcript, no scrollback, no way to know it vanished. So the defaults must differ, and holding is the one that respects which side can recover.

## The Move

**Never send what you interrupted.** If capture stopped for the system's own reasons, that path may not commit — full stop. The decisive question for any capture-ending path is *who ended it*; if the answer is "we did", commit is off the table. See [[an-event-is-not-a-cause]].

**Rejoin, don't restart.** When capture resumes, the buffered prefix is still there and new results append. The user experiences one sentence, not two fragments they have to mentally staple together.

**Bound the hold with staleness, not with hope.** A fragment abandoned forty seconds ago must not graft itself onto an unrelated new turn. Expire it, and log the expiry — an abandoned fragment is a distinct outcome from a completed turn, and a system that conflates them cannot tell you how often it is failing.

**Make held-ness perceptible.** This is the part that is easy to skip and shouldn't be. "Still listening" and "kept what you said, ear closed" demand opposite actions from the user — keep talking, or tap — and a screen-free user cannot distinguish them by default. Holding words the user believes are lost is better than dropping them, but it is not the same as them knowing.

**Log the lifecycle with causes.** `hold`, `commit`, `stale`, each with why. Without it you cannot answer the only question that matters after a bad session: did their words arrive, and if not, which of the three things happened?

## Beyond Voice

Any interruptible input has this decision, and the send-it failure is the one that keeps getting shipped because it looks like progress. A draft auto-saving mid-keystroke on a connection drop. A form submitting on blur when focus was stolen by a modal. A partial upload finalised because the socket closed. In each, the system had incomplete input, an event arrived, and it treated the event as the user's intent.

The user did not stop. Something stopped them. Those are different facts and only one of them is theirs.

## Related

- [[an-event-is-not-a-cause]] — why the interruption got read as completion in the first place.
- [[half-duplex-turn-taking]] — not interrupting, which is the better fix where it is available.
- [[latched-state-needs-a-reconciler]] — a hold whose release event is lost becomes a permanent one.
