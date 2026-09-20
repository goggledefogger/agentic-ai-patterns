---
type: pattern
date: "2026-07-29"
source: The work vault transcript-processor skill, generalized while standing up a second private business vault
tags:
  - transcripts
  - meetings
  - vault
  - attribution
---

# Meeting Transcript Processing

Turn loose meeting transcripts (Granola, MacWhisper, Gemini, any tool) into a durable, attributed outline in the vault, without ever storing the raw transcript.

## The Problem

Transcripts are the richest capture of business context most people have, and the least trustworthy. Names get mangled, speakers get merged, half-sentences read as decisions. Two failure modes compete: dump the raw transcript into the vault (unreadable, duplicates the transcription tool, privacy surface) or summarize it casually (attribution errors and invented facts propagate into notes, frontmatter, and drafts, where they rot silently).

## The Pattern

1. **Raw stays out.** The original lives in the transcription tool. The vault gets a processed outline in a dated meeting note (`coordination/meetings/YYYY-MM-DD-topic.md` or equivalent) with frontmatter naming the source tool and a "raw transcript not saved" line
2. **Clarification round before processing.** Cross-check every name and term against the vault (and any connected sources like a task board) first. Batch what's still unclear into numbered questions with your best guess attached, and wait for answers. Don't file anything load-bearing on a guess
3. **Attribute carefully, never assume.** Attribute only what the format supports: a two-person call with Me/Them labels is high confidence, a group call with merged speaker turns is not. Write "likely X" or "speaker unknown" over forced attribution. For load-bearing claims, check three independent signals before propagating: surrounding pronouns, continuity with prior threads in the vault, and whether the claim fits the person's role
4. **Flag garbled spots verbatim.** A mangled phrase gets quoted and marked "meaning unrecovered," not silently interpreted. Mishearings resolve with context ("Claude asset viewer" is Cloud Asset Viewer), but the resolution should be checkable
5. **Route, then outline.** Facts go to the area notes they belong in (people, product, decisions), linked not copied. The meeting note itself holds a topic-by-topic narrative outline with per-speaker attribution, detailed enough to be the long-term record of who said what and why
6. **Action items with real owners.** The person who said "I will" owns it. A suggestion someone made for you is routed as "X suggested" until you actually commit

## Why It Works

The clarification round converts silent errors into cheap questions. Attribution discipline keeps the outline quotable months later, when nobody remembers the call and the note is the only witness. Keeping raw text out means the vault holds interpretation (which you verified) rather than data (which the transcription tool already holds better).

## When to Use

Any vault that receives meeting transcripts. Worth wiring in at vault creation: a short "how transcripts are processed" section in the vault CLAUDE.md, or a thin vault-local skill adapting this pattern, so the first transcript that arrives has a path.

## Source

The author's the employer work vault grew a full transcript-processor skill (clarification rounds, three-signal name verification, attendance confirmation for group meetings, per-tool blind-spot notes). This pattern is the portable core, proven again the day a second business vault processed its first two-person partner call.
