---
type: pattern
date: "2026-09-09"
source: The personal agent voice pipeline, one private voice note sent through three doors, one of which answered fluently and kept nothing, measured 2026-09-09
tags:
  - capture
  - reliability
  - voice
  - data-loss
  - architecture
---

# Capture Before You Answer

The author sent one private voice note, four minutes long, through three doors at once. A phone recorder dropped it into a Syncthing folder, and it was captured and transcribed locally in about a minute. A Telegram voice message went to the capture bot, and it was captured and transcribed by two separate hosts. A Telegram voice message went to a voice bridge, takopi with a local whisper shim behind it, and that door gave the best performance of the three: it transcribed the note, answered with a fluent ten-minute reply, and threw the audio away. Not stored anywhere. Answered, and gone.

Nothing about that door looked broken from the inside. The capture gateway reported a healthy "0 captured", which is exactly what it reports when nothing arrived. The inbox looked fresh. The one system with a visible failure state said everything was fine, because from its point of view nothing had happened. The bridge never told it a recording existed in the first place.

The bridge's own diagnosis, once this got noticed, was that it shared a bot token with the capture gateway and the two were racing. That diagnosis was wrong on the facts (they're different bots) and wrong on the shape (a shared token produces a duplicate, not an absence). The real defect was structural: the other two doors write the raw payload to disk before anything else touches it, and this door never had that step. It went straight from audio in, to a transcription call, to a generated reply, and the only thing ever holding the recording was memory that got freed when the request ended.

## The pattern

Any door that accepts audio or text from a person writes the raw payload to the inbox, atomically, before it transcribes, answers, or routes it. The answer is a courtesy. The capture is the contract.

This isn't distrust of the transcription step: it's asymmetry of consequence. A late or wrong answer is annoying and correctable next turn. A never-captured recording is gone, and the sender has no way to know, because from where they're standing the door worked: it replied, fluently, at length. The better the answer, the more convincing the illusion that nothing was lost.

The fix that falls out of this is architectural, not a patch to one bridge: put the capture step somewhere every door shares, rather than trusting each door's author to remember it. The voice stack's Ear Door is an OpenAI-compatible transcription endpoint that saves the upload verbatim before it transcribes anything. Any bridge that already knows how to point at an OpenAI-shaped transcription URL (which is most of them, since that's the API shape the ecosystem converged on) gets capture for free, without its author ever writing a capture step of their own.

## Why it's easy to miss

Two doors on the same recording had capture built in, and the one that didn't was the newest and most capable: it could hold a ten-minute conversation. Sophistication on the answering side isn't evidence of care on the capture side; they're unrelated axes, and the more effort a door's author puts into the reply, the easier it is to never notice the raw audio was never saved.

The health signal made it worse. This pipeline's instinct is to alarm on capture failures, and it does, for doors that have a capture step to fail. A door with no capture step has nothing to fail, so it has nothing to alarm on. "0 captured, no errors" is indistinguishable from "nothing arrived" and from "everything that arrived was thrown away," and only one of those is fine.

## When to use

Any place a person's input passes through a step that can produce a plausible response (a chat reply, a summary, a transcription) before that input is durably stored anywhere. Voice bridges, chat relays, form-to-model pipelines, anything where "answer sent" can happen without "input kept."

## Related

- [[ack-outran-the-write]], an ack is a delete on someone else's server; this is the same shape one step earlier, an answer that outran the write, except the input was never on your own server to begin with, so there's no server left to recover it from
- [[no-delivery-without-arrival-accounting]], same family: a channel that can produce output with no accounting for whether the input it was fed ever landed durably

## Source

Measured 2026-09-09 in the personal agent's voice pipeline. One 4-minute voice note sent through three doors: a phone recorder into a Syncthing folder (captured, transcribed locally, ~1 minute), a Telegram voice message to the capture bot (captured, transcribed by two hosts), and a Telegram voice message to a voice bridge running takopi with a local whisper shim (transcribed, answered with a ten-minute reply, audio discarded, never written anywhere). The bridge attributed the loss to sharing a bot token with the capture gateway; the two bots were in fact different, and the actual defect was the absence of a capture-before-process step in that one door. Fix: The voice stack's Ear Door, an OpenAI-compatible transcription endpoint that persists the upload verbatim before transcribing, so any bridge pointed at it captures without its own author implementing anything.
