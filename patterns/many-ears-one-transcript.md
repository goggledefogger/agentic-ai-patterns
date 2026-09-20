---
type: pattern
date: "2026-09-09"
source: The personal agent voice pipeline, one recording transcribed by two engines on two hosts, and arriving through two doors under two source ids, measured 2026-09-09
tags:
  - capture
  - transcripts
  - dedup
  - concurrency
  - local-models
---

# Many Ears, One Transcript

The same recording from `capture-before-you-answer` produced two separate transcripts of the same audio: a laptop running Parakeet finished in 73 seconds, a cloud-VM leader running whisper base.en finished in 51 seconds, and the second one landed as a Syncthing conflict file next to the first because both hosts were watching the same folder. Separately, the recording also arrived through two different doors under two different source ids, with byte-identical audio in both.

Nothing compared the two transcripts against each other, and dedup keyed only on source id: exactly the thing that differs across doors and never repeats within one. So one spoken utterance turned into two rows in the triage queue, and whichever transcript happened to finish first, or land wherever something read first, became the version. The better transcript didn't win by being better. It won by being early.

## The pattern, in five rules

**Racing engines is a feature, not a conflict to resolve.** Two transcription passes on the same audio are two independent witnesses, not competitors for one slot. Keep every version. Never discard one because another arrived first: arrival order carries no information about which is more correct.

**Identity has two layers.** Within one door, a source id (a chat id, a file path) is a fine dedup key. Across doors, it's useless: two doors invent their own ids independently, and the only fact both doors agree on is the audio itself. Dedup on the source id *and* the content hash of the audio, or a recording that crosses doors always looks like two recordings.

**Reconciliation writes a new file. It never rewrites an old one.** When multiple versions of one recording are found, the merge lands in a file beside them (`<name>.reconciled.md`) carrying a `versions:` list (engine, host, duration, per version) and each original gets stamped `superseded-by:` pointing at it. No original transcript body is ever edited. This is the same discipline [[meeting-transcript-processing]] already keeps for a single transcript's raw text, extended to multiple raw versions instead of one: the source stays untouched, and what changes is the record about it, never the thing itself.

**The merger keeps every sentence and marks disagreement instead of guessing.** Two engines rarely disagree on everything: usually it's one word or a name in an otherwise identical sentence. The merge takes the more coherent reading where the two agree, and where they genuinely diverge it writes both inline (`[alt: whisper heard "polk county", parakeet heard "polk quarry"]`) rather than silently picking one. A merge that silently picks is a third, unwitnessed transcription happening one layer up.

**Local-first, and unmerged is a state, not a failure.** The merge itself runs on a local model. When no local model is up, the note doesn't get force-merged on whatever's reachable. It gets stamped pending, and the person gets a one-tap choice: allow a cloud merge, or wait for local. Unclassified audio is maximally sensitive by default, and merging two guesses about someone's private recording is exactly the judgment call that shouldn't happen on a stranger's API because the local box was asleep.

## Why this needed two engines to become visible

A single-engine pipeline never sees this bug class, because it never has two versions to compare: one wrong transcript alone just looks like a bad transcript. It took a second host racing the first, on a folder both were watching, to turn a routine transcription error into a legible dedup problem. Running two engines is worth it for correctness, but it's also the only thing that will ever surface where dedup is actually keyed.

## When to use

Any pipeline where more than one process can transcribe, OCR, or otherwise interpret the same raw input, whether by design (racing engines for coverage) or by accident (two watchers on one folder, two ingestion doors for one source).

## Related

- [[a-model-never-picks-the-destination]], same local-first, cloud-by-explicit-choice discipline one step earlier: routing never gets a model, and neither does deciding to send someone's unclassified audio to a cloud merge without asking first
- [[count-the-source-not-the-survivors]], same shape at the counting layer: what gets measured has to track the thing itself, not whichever copy of it happened to survive first
- [[meeting-transcript-processing]], the raw-stays-out rule this pattern's reconciliation-file rule extends to multiple raw versions
- [[capture-before-you-answer]], the incident this pattern shares its recording with; that one is about doors losing the input, this one is about what happens once two doors both keep it

## Source

Measured 2026-09-09 in the personal agent's voice pipeline. The capture-bot copy of one recording was transcribed independently by a laptop (Parakeet, 73s) and a cloud-VM leader (whisper base.en, 51s), the second landing as a Syncthing conflict file. The same recording separately arrived through two ingestion doors under two different source ids with byte-identical audio. Dedup keyed only on source id, so the recording produced two triage items with no comparison between them, and the surviving transcript was whichever host's pass got read first.

## Addendum, same evening: one filename per engine

The cloud-VM leader's whisper transcript in this incident arrived as a Syncthing conflict copy of the laptop's Parakeet file, because both watchers write `<name>.md` for whatever they transcribe. That collision is exactly what the identity-layers rule above predicts, but it has a consequence the rule doesn't say out loud: a conflict copy looks like clutter, not evidence.

The author cleaned "21 conflicts" off his phone that evening, the ordinary way anyone clears sync noise, and the whisper witnesses went with them. A second whisper pass had to be regenerated by hand afterward.

The fix is one filename convention: each engine writes its own file, `<name>.<engine>.md`, never a bare `<name>.md` a second watcher can collide with. Reconciliation then groups candidate versions by the `source:` field in frontmatter, not by filename, so naming stays collision-proof without the grouping logic needing to know engine names in advance.
