---
type: pattern
date: "2026-09-09"
source: The personal agent voice pipeline, a three-version transcript merge on a 16 GB Mac, measured 2026-09-09
tags:
  - local-models
  - transcripts
  - verification
  - voice
  - grading
---

# A Small Model Chooses, It Does Not Rewrite

Asked to merge three 1,000-word transcript versions of the same recording into one clean file, a 7B coder model running locally returned one of the three source versions byte for byte and reported the merge done. The file existed, had the right size, and carried the right provenance stamps. Every artifact said the job was finished.

The task, as given, asked the model to hold three long documents in mind at once and produce a fourth that improved on all of them. That is a rewriting task, and rewriting a document from memory is exactly the shape a small model degrades into copying one input and calling it output: under a shrunk context and a shrunk sense of what "better" means, the safest-looking answer is to change nothing.

## The Pattern

Reframe the task from rewriting to choosing. Diff the versions into hunks (`difflib`), and for each hunk that differs, ask the model one narrow question: which reading is coherent English. Batch the hunks (8 per call worked here) instead of one call per hunk, and instead of one call for the whole document. The model never sees the whole merge. It only ever sees a small disagreement and picks a side.

On the same recording this produced 88 hunks, 27 of them decided, and 32 left marked `[alt: reading A / reading B]` rather than guessed. That is a working merge. The rewrite framing, on the identical input, produced a copy.

Three calibrations turned out to matter, and each one was found by a grader checking the output against its sources, not by reading the merge and thinking it looked fine (see [[many-ears-one-transcript]] for the reconciliation format this feeds):

- **Hide which candidate is the base.** Naming one version "the original" and the others "alternates" made the model anchor on the original and keep its garbled readings with false confidence, even when an alternate was plainly cleaner. Present all candidates unlabeled as to origin.
- **Ask twice with the order reversed, keep only the picks that agree.** A model that answers A when A is listed first and A again when A is listed second has a real preference. A model that flips its answer when the order flips is guessing, and that hunk goes to a human instead of getting silently decided on one pass.
- **Run a free deterministic tiebreak before spending a model call.** A candidate made entirely of dictionary words beats a candidate containing non-words, and that check costs nothing. On this merge it resolved 17 of the 27 decided hunks before the model was ever asked, leaving the model only the hunks where both readings were plausible English.

One more trap is worth naming on its own: reasoning models returned an empty `content` field with the actual answer sitting in `reasoning_content` instead. Code that reads `content` and treats empty as not confident silently drops every one of that model's answers. Skip reasoning models for this kind of narrow forced-choice job, or read the field the model actually writes to.

## Grade Before You File

Every stage of this merge looked complete from its own artifacts: a new file existed, the byte count was plausible, the provenance frontmatter was correct. None of that is evidence the content is right, only that something ran. A sandboxed, read-only grader that compares the merge back against its three sources is what actually caught the failures, three times in a row on the same task: first a byte-for-byte copy of one source (verdict REDO), then a version that kept three garbled spans over the clean readings sitting right next to them (verdict FILE-WITH-NOTE), then a version that passed.

The grader's checklist is short and reusable for any merge-of-witnesses job: does anything present in a source go missing, does anything appear that no source said, how does it handle the hunks where the sources genuinely disagreed, do the `[alt: ...]` marks sit only where the disagreement is real rather than papering over a lazy pick, and would this still read cleanly in a month. A transcript, or any merged artifact, gets filed to the person only after that verdict, never on the strength of the file existing.

The grader itself has no write access. It reads the merge and the three sources and returns a verdict, REDO, FILE-WITH-NOTE, or PASS, and nothing else. Giving a grader the power to fix what it finds turns it back into a second, unaudited pass at the same task, and the failure it's meant to catch (something that looks done but isn't) can just as easily happen to the fix.

## Why Reading the Output Doesn't Catch This

A byte-for-byte copy of one source reads as a perfectly good transcript, because it is one, of the wrong thing. Nothing about the prose is broken: no dangling sentence, no obvious garble, no sign anything was skipped. The only way to see the failure is to hold the output next to all three sources at once and check whether it actually merged them, which is precisely the comparison a human skimming the finished file never does. The same blindness hit the second attempt: three garbled spans read as normal sentences unless you already know what the clean versions said at those exact points.

This is also why the calibrations above needed a grader rather than a person spot-checking outputs. A model that anchors on the labeled original, or flips its pick when the order reverses, produces text that looks confident either way. The tell is not in the sentence, it's in the disagreement between two runs of the same question, which only shows up when something checks both runs against each other.

## When to Use

Any task that hands a small local model more than one full version of the same content and asks for a single best output: transcript reconciliation, merging drafts, resolving conflicting notes. If the ask can be reframed as a series of small forced choices between existing readings rather than free generation from a large context, do that first, and grade the result before filing it.

## Related

- [[many-ears-one-transcript]], the reconciliation format this choosing method fills in: the `versions:` list, the `superseded-by:` stamp, and the `[alt: ...]` marker this pattern's tiebreak logic decides when to use
- [[a-model-never-picks-the-destination]], same local-first discipline one layer over: a model chooses among options a deterministic process already enumerated, it never generates the option space itself

## Source

Measured 2026-09-09 in the personal agent's voice pipeline. A 7B coder model asked to merge three roughly 1,000-word transcript versions into one returned a byte-for-byte copy of one source and reported success. Reframed as a per-hunk forced choice (difflib hunks, 8 per batch, unlabeled candidates, order-reversed double-ask, a free dictionary-word tiebreak first), the same model resolved 88 differences: 27 decided, 32 marked `[alt: ...]`. A sandboxed grader comparing the merge to its sources caught the copy, then a version that kept garbled spans, before passing the third attempt.
