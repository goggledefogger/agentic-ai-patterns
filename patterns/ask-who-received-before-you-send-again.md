---
type: pattern
date: "2026-08-03"
source: The personal agent — walk-and-talk sent five duplicate Telegram voice notes for lines the phone page was already playing, and ran a whole walk as one never-ending turn, 2026-08-03 (ux-debt 64)
tags:
  - architecture
  - voice
  - delivery
  - turn-taking
  - reliability
---

# Ask Who Received Before You Send Again

A system that can reach a person more than one way will eventually reach them every way at once. The fix is not fewer channels — it is asking, before each send, whether anyone already has it.

## The Problem

A walking assistant had three ways to reach its user: a phone web page, a Telegram voice note, and the laptop's own speakers. Every reply fired the page *and* Telegram. Five duplicate voice notes in one walk. The user named it himself, and named the fix in the same breath:

> *"aren't we duplicating work… ideally it would send through telegram if it knew for sure that i didn't hear it through the web."*

The damning detail is not that the check was missing. **The check already existed.** Ten lines above the Telegram send, the same function computed whether a page was connected and listening — and used that answer to decide *not* to speak through the laptop speakers. Then it sent to Telegram without consulting the value it was still holding.

That is the shape worth recognising: not an unknown, but a **known that no one asked**. Each channel had been added at a different time to close a different silence, and each was written as *"also send here"* rather than *"send here if no one else did."* Nobody wrote a bug; the composition was the bug.

The second failure the same day rhymes with the first. The assistant ran an entire walk as one long turn, producing replies while the user's messages arrived mid-flight — so it answered message N while N+1 was already in the air, and pre-produced speech that depended on an answer the user had not given yet. The user heard back-to-back monologue and guessed at a cause:

> *"maybe it's having parallel threads?"*

It was not threads. It was one turn that never ended. **Sends without arrival accounting and turns without boundaries are the same defect at two scales**: both dispatch without establishing that the previous dispatch landed with someone.

## The Move

**Choose a channel per message, don't fan out.** Redundant channels are alternatives, not a set. The decision belongs at send time, for that one message.

**A second channel is a fallback conditioned on non-arrival — never a parallel pipe.** Send on the primary; send on the secondary only after a grace window in which the primary reported nothing. Log the fallback *as* a fallback, so the duplicate you do send is legible rather than mysterious.

**A surface cannot be primary unless it reports arrival.** This is the constraint that makes the rest buildable, and it is a design requirement on the surface, not a nicety. "Did they hear it" must be answerable from a beacon the receiver emits — a playback event, a read receipt, a rendered-frame ping. A channel that cannot say whether it delivered may only ever be the *fallback*, because nothing downstream can reason about it.

**Prefer the receiver's report over the sender's inference.** "The connection is open" is not "a human received this." Those diverge, and they diverge exactly when it matters — a backgrounded page, a locked phone, a socket the OS has not yet reaped. See [[opaque-write-needs-a-read-back]].

**Fail toward the redundant send, not the silent one.** When the arrival signal is unavailable or ambiguous, send again. A message the user did not need costs a moment of annoyance; a message that reached nobody costs the interaction. Ranking those two wrongly is how a de-duplication fix becomes an outage.

**End the turn to prove you are waiting.** A question whose answer changes what you do next must be the *last* thing produced in that turn. Anything generated after it — however well-intentioned — is a second dispatch that presumes an answer, and the person on the other end reads it as not being heard.

## Why Both Halves Fail Invisibly

Over-delivery and never-ending turns share an alibi: **from the sender's side, both look like working harder.** Every duplicate was a successful send. Every extra utterance was a real answer to a real question. Neither raises an error, appears in a failure count, or shows up in a test that asserts *"the message was delivered."* The only instrument that detects them is a person saying *this feels wrong* — usually in the vocabulary of manners ("too much", "it talked over itself") rather than the vocabulary of defects.

So both are caught late, and both are caught by the least scalable detector available. That argues for deciding repeat behaviour **in the same change that adds the channel**, when the question is still cheap: *how many times may this fire, and how will I know it landed?* Retrofitting it means first convincing yourself that something which never errored is broken. See [[count-the-source-not-the-survivors]] for the same asymmetry in counting, and [[decorative-gate]] for a check that reports success without checking.

## The Trade to Watch

Every fix in this pattern pushes toward *fewer* sends, and the prior generation of fixes in the same system all pushed toward *more* — each one added because a walker got silence and could not start. Those fixes were right. Pushing harder to close a silence and asking who received before pushing again are not opposites; they are the two halves of one property, and shipping only the first half lands you somewhere worse than where you began. An under-delivered message leaves a person who cannot act. An over-delivered one leaves a person who has learned to ignore the channel — the same place, reached more slowly, and much harder to reverse.

## Related

- [[one-pipe-two-speakers]] — the same system's turn-taking model; this pattern is what happens when a message escapes it into a second channel.
- [[opaque-write-needs-a-read-back]] — a success receipt from the transport is not confirmation the payload landed.
- [[a-second-writer-satisfies-your-gate]] — "it's in the store" is not "it was delivered," and someone else's write can vouch for yours.
- [[announce-the-move-in-the-old-room]] — the converse failure: a conversation that relocates and strands the channel the person is still watching.
- [[an-event-is-not-a-cause]] — a lifecycle event mistaken for "the turn ended."
- [[convergent-standup-sloppy-quits]] — put lifecycle intelligence in start, because quits are where attention already left.

## Rule of Thumb

Adding a way to reach someone is half a feature. The other half is how you will know they were already reached — and if that half has no answer, the new channel is a fallback, not a peer.
