---
type: pattern
date: "2026-07-31"
source: A skill recorded two collaborators' handles as a symmetric fact, "A tags B as @x, B tags A as @y". A draft written minutes later, by the agent that had just written the table, opened by tagging the wrong person
tags:
  - skills
  - documentation
  - anti-pattern
  - context-engineering
---

# A Symmetric Reference Reads Correctly and Gets Used Backwards

Some facts are naturally symmetric: who tags whom, source and target, from and to, ours and theirs. Written down as the symmetric fact, they are accurate, complete, and reversible at the point of use, because the reader has to work out which side they are standing on before the table means anything. That derivation happens under load, gets it wrong, and the output looks confident.

## The Problem

A reference recorded two collaborators' handles:

```
- A is <person-1>. B tags them as @x
- B is <person-2>. A tags them as @y
```

Both lines are true. Minutes after writing it, the same agent drafted a message for A to send and opened it by tagging @x, which is A tagging themself. The information was present, correct, and freshly written. It was still used backwards.

The failure is structural. A symmetric table stores a *relation*. Using it requires a second step the table does not supply: establish which participant you are acting as, then pick the matching row. That step is invisible, unprompted, and every reader performs it silently. Getting it wrong produces output that is well-formed and confidently addressed to the wrong party.

The same shape shows up in migration docs (source and target databases, one line each), diff tooling (ours and theirs), sync rules (which side wins), and access matrices (who may read whose). In every case the table is right and the reader is one inversion away from the wrong action.

## The Pattern

**Write the rule from the acting party's point of view, and name the wrong option explicitly.**

```
- Anything drafted here is A speaking, so the tag is always @y
- @x is B's tag for A. Never open a draft with it, that is A tagging themself
```

Same two facts. No derivation left to perform, and the failure mode is named rather than merely excluded.

Three parts:

1. **Fix the actor in the rule.** Not "who tags whom" but "when writing here, you are A." If the artifact has one dominant direction of use, encode it and let the rare reverse case be the one that costs a lookup
2. **Name the wrong choice and say why it is wrong.** "Never @x, that is tagging yourself" is a self-checking sentence. A reader who has just written @x recognizes the error without re-deriving anything. A table that only lists correct values gives nothing to check against
3. **Say that it was inverted in practice.** One clause of incident, and the rule stops looking like pedantry, which is what gets it trimmed later

## Why It Works

The symmetric form is optimized for the writer, who holds both sides in mind and wants the complete relation on the page. The instruction form is optimized for the reader, who holds one side and needs the next action. Those are different jobs, and reference material defaults to the first without anyone choosing it.

Naming the wrong option converts recall into recognition. Recall under load is where inversions happen. Recognition survives it.

## When to Use

- Any reference where a mistake is a swap rather than a gap: handles, accounts, source and target, from and to, ours and theirs, primary and replica
- Documentation an agent will act from rather than merely read
- Anywhere the asymmetry of *use* is strong (nearly always drafting as one party) even though the underlying fact is symmetric

## When NOT to Use

- Genuine lookup tables with many entries and no dominant direction of use, where a per-actor rewrite means N rules instead of one table
- Reference for humans who already know which side they are on and just need the value

## Watch-outs

- **Do not keep both forms.** Leaving the symmetric table beside the instruction gives the next reader two things to reconcile and reintroduces the derivation. Rewrite in place
- **A correct table is not evidence the shape works.** The failure needs no error in the data, so reviewing the table for accuracy will always pass and will never surface this
- **Watch for it in generated summaries.** A model asked to condense an instruction back into a table will happily restore the symmetric form, because it is shorter

## Adjacent Patterns

- `appending-is-not-learning.md`, the same instinct at the file level, where a correction becomes a new line instead of a change to the wrong line
- `pointer-names-the-file-not-the-policy.md`, related failure of writing down where something lives rather than what to do about it

## Source

A skill documenting a two-person collaboration board recorded the handles symmetrically, with one line per direction. The agent that wrote those lines drafted a reply for one collaborator to post, minutes later, in the same session, and addressed it to the wrong one. Caught by the human with "in trello i'd tag his username not mine, that should be obvious in our skill."

The fix was to invert the rule rather than add a warning next to the table, on the grounds that a file holding both "here is the relation" and "here is what to do" hands the next reader the same derivation that failed the first time.
