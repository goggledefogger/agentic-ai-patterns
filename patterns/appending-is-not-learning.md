---
type: pattern
date: "2026-07-28"
source: The work vault, voice system v2 (a doc), confirmed on a live support-email correction set 2026-07-28
tags:
  - skills
  - maintenance
  - context-engineering
  - lint
  - anti-pattern
---

# Appending Is Not Learning: Classify Each Correction, and Budget What It Lands In

A voice skill had absorbed a year of corrections the only way anything absorbs corrections by default, which is to append them. One medium reference reached 7,545 words across roughly 85 prohibition bullets. The core mistakes list was numbered 10, 10c, 10b. About 85% of the rules were prohibitions or "X over Y" pairs, with no positive description of what the voice actually was. The owner named the failure himself: "we make longer and longer files giving more and more examples, but that doesn't make it better." The proof it had stopped working was a draft that violated 8 rules read minutes earlier. Reading is not compliance, and a file grown by accretion is optimized for the author's sense of coverage rather than the reader's ability to comply.

The rebuild changed one thing about how a correction gets absorbed: it is routed by its *nature* before anyone opens a file, and every artifact it could land in carries a hard ceiling. References went from ~20k words to ~4.5k. On 2026-07-28 the loop ran live on a support-email draft. The deterministic lint caught an AI-tell ("landed") in a draft that had already been hand-checked minutes earlier, the human's own edits classified cleanly into all three rule-bearing bins, and one edit ("pushing this through" to "moving this through approval") was correctly left as a commit-message note rather than becoming a rule. The system got better without getting longer.

## The Pattern

1. **Classify before you open the file.** The decision is which bin, and it happens while the correction is still in hand. Once you are already inside the rules file, the only move the file offers is append:

| The correction is... | Where it lands |
|---|---|
| A mechanical swap a regex could catch: word, phrase, punctuation | One row in the deterministic checker. No prose |
| Another instance of a rule that already exists | Strengthen or generalize that rule in place. Swap in the new example if it teaches better |
| A genuinely new recurring judgment | One checklist item, 20 words or fewer, plus a short why |
| Specific to this one situation | No rule. The commit message, and nothing else |

2. **The first bin is the only one that makes the system shorter and stronger at once.** Anything a checker can decide leaves the prose entirely, which shrinks what a human reads and converts a rule enforced by vibes into one that cannot be missed. Route aggressively here. Mechanical checks are cheap, and a hit that turns out to be a false positive costs one judgment call at the point of use.

3. **The fourth bin gets skipped, and it does the most work.** Most corrections are situational. A system with no discard bin reads every correction as evidence of a missing rule, which is how 85 bullets happen. Naming "this is not a rule" as a legitimate outcome, with the observation preserved in the commit message, is what keeps the count flat.

4. **Budget every artifact, enforced one-in-one-out.** Give each file a number: portrait 200 words, skill core 120 lines, each reference 120 lines, 15 checklist items, 6 examples. When a file is at budget, nothing is added until something merges or dies. This is the mechanism. Classification on its own still grows the file, because three of the four bins add something, and the ceiling is what turns every addition into a trade and forces the merge that classification never demands by itself.

5. **Provenance lives in the commit message, never inline.** No "Source: June 18 2026" lines in the rules. Git already holds the story, and the inline copy is weight on every future read.

6. **Run a distillation pass when a budget trips, or quarterly.** Re-derive each checklist from its examples, cut any rule not evidenced by an example or a repeated correction, merge near-duplicates, and move anything a regex can decide into the checker.

## When It Shows Up

Any rule system fed by corrections: lint pattern tables, review checklists, registry rows, coding-standard docs, agent skill files, a `CLAUDE.md` that grows a line after every bad session. The tell is a file whose newest rules are its most specific, where two rules say the same thing at different altitudes, and where nobody can name a rule that would ever be removed. What separates this from its neighbors is which failure is happening: [`living-doc-refresh-ritual.md`](living-doc-refresh-ritual.md) fixes a doc that became untrue, [`living-doc-archive-split.md`](living-doc-archive-split.md) fixes bloat from content that aged out, and here nothing is untrue and nothing ages out, so neither refresh nor archival touches it. [`always-loaded-context-budget.md`](always-loaded-context-budget.md) is the same budget idea where a harness enforces the ceiling by silently truncating, so that one is about surviving a cap someone else set and this one is about holding a cap nobody enforces. [`tdd-for-skills.md`](tdd-for-skills.md) covers the artifact's birth, this covers its middle age.
