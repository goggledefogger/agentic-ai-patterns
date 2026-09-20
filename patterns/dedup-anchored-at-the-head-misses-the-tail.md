---
type: pattern
date: "2026-08-07"
source: The personal agent — walk-and-talk phone bridge glued one spoken sentence into a twenty-deep staircase after a recognizer restart, 2026-08-07 (ux-debt 81)
tags:
  - correctness
  - streaming
  - voice
  - deduplication
  - diagnosis
---

# Dedup Anchored at the Head Misses the Tail

A stream that can be resumed will re-deliver. The obvious defense is to compare what arrives against what you already hold and drop the repeat — and the obvious way to compare is from the beginning of both. That works right up until something precedes the repeating part, at which point every head-anchored test fails at the first word and the duplicate sails through as if it were new.

## The Problem

A phone's speech recognizer delivers a sentence as a growing prefix: *"I want"*, *"I want you"*, *"I want you to ask"*. The dedup for that is easy and it shipped: if the new text extends what we hold, replace; if it is a shorter echo, drop. It held for months on continuous speech.

Then a mic restart mid-utterance produced this, verbatim from the log:

> thanks but actually we **should should have should have a should have a way should have a way where**…

Look at where it starts. The head — *"thanks but actually we"* — appears exactly once. The staircase is a growing prefix **of the tail**. The recognizer resumed and began re-delivering from where the walker's new words started, not from the beginning of the held sentence.

Three separate dedup rules were in place by then, added over three incidents: raw prefix, reverse prefix, and a fuzzy near-twin check for reworded re-finalizations. All three compared from word one. So all three failed at position zero, every delivery took the append branch, and the sentence grew about twenty segments deep before it was sent.

The diagnostic trap is the interesting part. Each earlier fix was correct, tested, and aimed at a real observed failure — and each one made the next incident *look* like a regression of the previous fix, so the instinct was to make the existing comparison smarter. No amount of smarter helps: the rules were not too weak, they were pointed at the wrong end of the string. **A heuristic that cannot see the failure is not improved by tuning it.**

The staircase also opened on a **single** shared word. Any minimum-overlap threshold set to be conservative would have left a visible duplicate in the very first merge.

## The Move

**Overlap the held tail against the arriving head.** Find the longest suffix of what you hold that matches a prefix of what arrived, and keep only what is new after it. This is the one comparison that sees re-delivery-from-the-middle, and it degenerates gracefully into plain prefix growth when the overlap happens to be the whole held string.

**Scope it to the resume boundary, and let that scoping be the safety.** Tail overlap is aggressive: applied to ordinary continuous input, a shared *"the"* merges *"I went to the store"* with *"the weather is nice"* and silently eats a real sentence. So arm it only in the window where a resumed stream is actually re-delivering — from the restart until the next commit — and disarm it outside. With the window as the guard, a one-word overlap inside it is safe, because inside it the arriving text is by definition a re-delivery. **Prefer a narrow window with a permissive rule over a wide window with a strict one:** the strict-and-wide version leaves exactly the residue it was tuned to avoid, and it is harder to explain.

**Compare on normalized tokens, not raw strings.** Capitalization and punctuation are the producer's editorial opinion and they drift across a resume; a raw `startsWith` sees two different strings where a human sees one sentence.

**Test the interrupted path explicitly.** Every one of the earlier fixes was tested on smooth continuous input and passed. The bug lives at the seam, so the test has to force the seam — kill the stream mid-utterance and resume it. Replaying the real logged sequence verbatim is worth more than a synthesized one, because the synthesized one embodies the theory you are trying to check.

## Beyond Voice

Nothing here is about speech. It applies to any consumer that holds partial state across a producer restart: a resumed file upload re-sending the last chunk, a WebSocket reconnect replaying from an approximate cursor, a log tailer that reopens after rotation, an SSE stream resuming from a last-event-id the server rounds down. In every case the producer resumes from *its* notion of where you were, which is behind where you actually are — so the re-delivery arrives with your recent tail on the front of it, and a head-anchored comparison never fires.

The failure signature is the same everywhere and worth recognizing on sight: **the beginning of the record appears once, and everything after it stutters.** When you see that shape, stop tuning the comparison and go look at where it is anchored.

## Related

- [[half-duplex-turn-taking]] — the other half of the same walk; both bugs came from an interruption, and the deference fix makes this one rarer without touching it
- [[a-fake-can-only-fail-the-ways-you-have-seen]] — why the existing tests could not have caught this: the mock only died the way the previous incident died
- [[an-event-is-not-a-cause]] — the restart is the event; the anchor is the cause
- [[a-heuristic-where-an-exact-key-exists]] — the near-twin rule was reached for when the structural fact (a resume happened) was available and unused
