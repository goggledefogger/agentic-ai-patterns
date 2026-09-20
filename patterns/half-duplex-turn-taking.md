---
type: pattern
date: "2026-08-02"
source: The personal agent — walk-and-talk phone bridge cut the author off mid-sentence and killed three of its own six replies in one walk, 2026-08-02 (ux-debt 57)
tags:
  - architecture
  - voice
  - turn-taking
  - reliability
---

# One Pipe, Two Speakers

A phone on a walk has one audio channel and two things that want it. Android ducks media playback while a microphone is live, so the assistant genuinely cannot speak over a listening ear without both sides coming out broken. Something has to wait.

The obvious answer — the assistant always waits — silences it whenever a room is noisy. The shipped answer — the assistant always wins — is what produced the complaint that started this: *"it was listening then cutting me off."*

## The Problem

Two failures on one walk, which looked unrelated and were the same thing.

**The assistant talked over the walker.** A reply arriving mid-sentence began playing, which killed the mic, which submitted his half-finished sentence. He experienced being interrupted; the system experienced a routine playback start.

**The assistant talked over itself.** Every reply seized the channel from whatever was already playing. Three of six lines that walk started and never finished — the walker heard sentences trailing off partway and read it as the assistant losing its train of thought. Nothing in the code considered this a fault; superseding was the default and had no name.

That second one is the tell. A system with no turn-taking model does not merely mishandle the human's turn — it has no concept of a turn at all, so it collides with itself too, and nobody notices because the victim never complains.

## The Move

**Defer behind audible speech, not idle state.** When a reply is ready, ask whether the human is *audibly mid-sentence* — speech detected now, or words landed within the silence window. Idle flags are not the question; the question is whether interrupting them would take something away.

**Bound the wait, and mean it.** Someone who never pauses, or a noisy street holding the detector open, must not be able to mute the assistant forever. Cap the deference (2.2s here) and speak when it expires. An unbounded courtesy is an outage with good manners.

**Queue in order; never supersede.** Replies that arrive during a wait play FIFO. A new arrival cancelling an unplayed predecessor is data loss dressed as freshness, and it is invisible in testing because the thing destroyed is your own output.

**Drain before reopening the ear.** Speak everything queued, *then* re-arm the microphone. Opening the ear and immediately talking into it produces exactly the collision the queue was built to prevent.

**Every bounded wait needs its stuck-flag guard, written in the same change.** The deference test reads a "speech in progress" flag, and platforms drop the corresponding end event routinely. A latched flag turns "wait if they're talking" into "wait 2.2 seconds, always" — the feature still works, so nothing looks broken; it just feels sluggish forever. Force-clear the flag when capture dies. See [[latched-state-needs-a-reconciler]].

## Choosing Who Waits

The asymmetry is the design, and it comes from what each side costs to lose. A delayed or dropped reply costs the user nothing they cannot ask for again. A sentence they spoke while walking is unrecoverable: they cannot see that it was lost, and re-forming a finished thought is the most expensive thing you can ask of them.

So: **output is cheap and interruptible, input is precious and held.** Every specific rule above follows from that one line, and a system that gets the asymmetry backwards will feel rude no matter how well it is tuned.

## Related

- [[an-event-is-not-a-cause]] — why the interruption also *submitted* the interrupted sentence.
- [[hold-vs-drop-on-interrupted-input]] — what happens to the words when a turn collides anyway.
- [[latched-state-needs-a-reconciler]] — the stuck flag that turns a conditional wait into an unconditional one.
- [[self-report-must-not-end-what-it-reports-on]] — budget on silence, not on the subject's state.
