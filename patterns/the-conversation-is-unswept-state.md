---
type: pattern
date: "2026-08-18"
source: The dashboard member-zero day two (chat residue, SAVE.md sweep, Kept provenance)
tags:
  - memory
  - honesty
  - capture
  - ux
---

# The Conversation Is Unswept State

A save indicator computed only from files silently claims the conversations
are clean. They aren't: the sneakiest unsaved state in an agent system is a
decision made in talk that no file reflects yet. A member who chats all
evening and sees "all saved" is being lied to by omission — walk away, and
the decision is gone.

## The Pattern

Treat conversations as a second species of residue, alongside dirty files —
under the same honesty discipline:

1. **Detect by fact, never by inference.** "The member last spoke in this
   thread after the last save" is a timestamp fact, exactly like a file
   mtime. It claims the thread is *unswept*, never that it is *important* —
   the same modest claim file residue makes. A server guessing which chats
   hold meaning would be a lie dressed as a feature; if you want meaning, a
   deeper layer exists where the **session itself reports** save-worthy
   moments as they happen — reported, never inferred.
2. **The sweeping conversation clears itself for free.** Mark threads by
   *last member prompt*, not last activity: the "save this" prompt that
   triggers a sweep predates the save mark it causes, so the sweeping
   thread drops off the list with no special case — and any new talk
   re-marks it honestly.
3. **A sweep writes notes, not transcripts.** Swept content lands as durable
   notes in their proper homes (update-don't-duplicate, absolute dates),
   through each store's own write discipline — never as a chat-export dump.
   The conversation was the *source*; the brain's organization is the
   *destination*.
4. **Provenance is half of retrieval.** Every capture swept from a thread
   names its source conversation in its ledger entry ("…from the chat
   'planning the garden'"). Future-you finds a fact by remembering the
   conversation, not the filename.
5. **"Looked, nothing to keep" is a verb, not silence.** A dead-end thread
   (an error, a question that fizzled) still needs an explicit way off the
   list — a reported dismissal, distinct from deletion. New talk brings it
   back; the member's judgment call is a fact worth recording, and letting
   the list silently rot teaches the member to ignore it.

## Why It Matters

The failure mode is invisible: everything *looks* saved, so nobody checks.
The fix costs almost nothing — the timestamps already exist — and the
boundary it draws (facts now, reported meaning later, inferred meaning
never) is the same line that keeps every other honesty surface honest.
