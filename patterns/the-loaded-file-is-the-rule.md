---
type: pattern
date: "2026-08-11"
source: The personal agent — rule 80 of the ops playbook shipped as 51 lines of rationale and zero lines of rule, discovered hours later by unrelated work that went looking for it
tags:
  - documentation
  - agent-instructions
  - context-engineering
  - anti-pattern
  - drift
---

# The Loaded File Is the Rule; Everything Else Is Commentary

A mature instruction set usually splits in two: a short imperative that loads into every session, and a long provenance file that does not. The split is right — it is the only way a rule set grows evidence without growing the context budget. But it creates a failure with no error message: **a change can land in the half nobody reads.**

The provenance file is where the thinking happens, so it is where the author's attention is. It is also the satisfying half to write. The imperative is four lines of compression done last, when the interesting work feels finished. Skip it and every artifact of diligence survives — a real incident, a correct analysis, a clean commit — while the rule itself was never written anywhere a future session will look.

## The incident

An agent playbook keeps `SKILL.md` (loaded at every session start) beside `rationale.md` (loaded never, read on demand). The convention is explicit in the file: *"Core carries the imperative; the war-story goes to `rationale.md`."*

A session hit a genuine architectural trap — a note it had authored under a sensitivity tier that its own tooling then refused to reopen — diagnosed it correctly, declined two tempting non-fixes, and wrote it up as rule 80. Fifty-one lines of rationale, a careful commit message, real insight.

`git show --stat` on that commit lists exactly one file: `rationale.md`.

Hours later a different session hit the same wall from the opposite direction, went looking for prior art, and found `## Rule 80` in the provenance file with no rule 80 in the playbook. Between those two moments, every session that loaded the playbook loaded 79 rules and had no idea the eightieth existed. The trap it described stayed fully armed.

Nothing failed. No test covers "the rule exists in both halves," the rationale file rendered fine, and the commit message described a rule that was, in the only file that matters, never added.

## The Pattern

1. **Treat the pair as the unit of change, not the file.** "Add a rule" means both files or neither. A commit touching one is incomplete by definition, whatever its message says.

2. **The check is mechanical, so write it.** Every provenance section has an identifier — a number, a slug, an anchor. Assert the two sets match, both directions: a rationale entry with no imperative is a rule nobody will read, and an imperative with no rationale is a rule nobody can question. This is a five-line script and it never needs judgment.

3. **Write the compressed half first.** It is the deliverable; the essay is the support. Authoring in that order also surfaces rules that cannot be compressed, which are usually two rules wearing one number.

4. **Suspect the diff, not the message.** A commit that says "adds rule N" and touches one file is the whole signature. It is visible in `--stat` and invisible in every other view, which is why it survives review.

## Why it stays invisible

- **Both files are individually valid.** Neither is malformed; nothing is untrue. The gap exists only in the join, and no reader holds both halves at once.
- **The author has the rule in working memory.** They finish the session behaving as though it exists, because for them it does.
- **The provenance file is the one people grep.** Someone searching for the topic *finds* it, concludes it is documented, and never checks whether it loads.
- **Discovery requires unrelated work.** Only a session that goes looking for the rule without knowing its number will notice — which is exactly the session that most needed it to be loaded.

## Watch-outs

- **This is not solved by "remember to update both."** It is a completeness invariant, and completeness invariants are what checkers are for. The rule that failed here was written by a careful session that knew the convention.
- **The same shape appears wherever a summary and a detail file are maintained together**: an index and its entries, a changelog and its release notes, a schema and its migration, an `AGENTS.md` and the docs it points at. Ask which half the consumer actually loads, and treat that one as the artifact.
- **A budget makes it more likely, not less.** Ceilings on the loaded file create real pressure to park content in the unbudgeted half, and parking the *imperative* there is the degenerate case of a healthy instinct.
- **Restoring a missing half is not the same as authoring it.** Write it from the provenance entry, not from memory of the incident, or the two halves will disagree on what the rule says.

## When NOT to use

If the provenance file is a genuine archive — closed items, superseded decisions, history nobody acts on — then entries there legitimately have no live counterpart, and pairing checks will fight the archive. See `living-doc-archive-split`. The invariant applies to the *active* set only.

## Adjacent Patterns

- `appending-is-not-learning` — how corrections get routed into a rule set without inflating it; this is the failure one step earlier, where the routing drops the payload
- `always-loaded-context-budget` — the budget pressure that makes the unloaded half attractive
- `living-doc-archive-split` — the legitimate version of content living outside the loaded file
- `pointer-names-the-file-not-the-policy` — the sibling failure, where the loaded half is present but too abstract to act on
- `churn-free-generated-artifacts` — the same "assert the join" instinct applied to derived files

## Source

The personal agent, `skills/the agent's ops playbook/` — rule 80 committed to `rationale.md` alone on 2026-08-11 and restored to `SKILL.md` the same day by a session investigating the mirror-image problem. The restoring commit added the pairing note to the rule's own rationale entry, on the theory that the next person to make this mistake will be reading that file when they do.
