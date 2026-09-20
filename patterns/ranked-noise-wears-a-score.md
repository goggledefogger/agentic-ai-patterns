---
type: pattern
date: "2026-08-28"
source: The course — a search bridge on a real vault; probed live before a working session, fixed in a search-bridge fork PR #1
tags:
  - semantic-search
  - verification
  - honesty
  - retrieval
---

# Ranked Noise Wears a Score

Semantic search never returns empty for an absent topic — ranking always ranks *something* — so every honesty guard keyed on the empty result has a blind side. Measured: pure gibberish (`zqxjkbrtplm`) against a real vault returned five confident semantic matches at 0.62–0.64, over a 0.4 similarity threshold that the docs called the default floor. The floor existed and filtered nothing, because absolute cosine floors sit below the embedding model's baseline for *unrelated* text. An agent asking about something the vault does not hold gets a list that looks exactly like findings, and no warning fires, because the warning machinery was waiting for `results: []`.

This is `verification-needs-a-negative-control.md` applied to retrieval: the system had probes ("can I find a note I hold a vector for?") and coverage accounting ("is every note embedded from current text?"), and both passed while gibberish scored like a finding — because nobody had ever asked it a question with a known-absent answer.

## The Pattern

- **Measure the noise baseline per corpus, never assume it per model.** Embed a small fixed set of gibberish anchors (fixed, so two runs measure the same thing; varied in length, because similarity drifts with token count) and score them against every vector the corpus holds. Cache per corpus generation.
- **A max over few samples undershoots the tail — allow for it with the measured scatter, not an invented margin.** A ninth gibberish string scored 0.007 over the eight-anchor max. Ceiling = max of per-anchor bests + one standard deviation of those bests. Both numbers come from the measurement.
- **Report the fact; warn on the verdict.** The envelope carries `noiseCeiling` always (the fact, for agents to compare against). The loud warning fires only when even the *best* semantic hit sits at or below it: "these results are unrelated text wearing scores." A literal keyword hit is never explained away as noise — a term match stands on its own.
- **Unknown is not zero.** When the measurement cannot run (no model, empty corpus, dimension mismatch), the field is absent — never `0`, which is a similarity a result could legitimately have.
- **Validate with discriminating controls, both directions.** Gibberish queries must warn; relevant queries must not. On the live check: 3/3 gibberish warned, 3/3 relevant clean (tops 0.79–0.84 vs ceiling 0.55). One direction alone is memorizable; both together pin the discrimination.

## Why it works

The empty-result guard and the noise ceiling cover complementary halves of the same lie. `negativeResultsTrustworthy` answers "can I believe *nothing came back*"; the noise ceiling answers "can I believe *what came back*." A retrieval system needs both, and the second cannot be a constant because the noise baseline is a property of the corpus × model pair — the same model scored gibberish at 0.51 on a 3-note vault and 0.64 on a 587-note one.

## When to Use

- Any semantic/vector search surface an agent consumes — the agent will confidently narrate top-k results whether or not they mean anything.
- Any similarity threshold chosen as a constant: check it against the measured gibberish baseline before trusting it to filter anything.
- Ground-truth the probe itself: when testing "search finds X / doesn't find Y," establish X and Y from the filesystem (grep), never from the index under test — a health number computed from the thing it checks is the disease (`half-gate-whole-verdict.md`).
