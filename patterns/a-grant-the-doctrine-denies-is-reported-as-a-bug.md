---
type: pattern
date: "2026-08-30"
source: The personal agent — desk trust (the agent's own repo, issue #110) shipped a permission grant and left the sentence that denies it standing in CLAUDE.md; within an hour the agent both refused to use the grant and reported its own correct behaviour as a security failure (the personal agent, issue #111)
tags:
  - agents
  - prompting
  - permissions
  - anti-pattern
  - security
---

# A Grant the Doctrine Denies Is Reported as a Bug

Shipping a new capability means changing two things: the mechanism, and every
sentence in the agent's own context that says the mechanism does not exist.
Change only the first and the capability is not merely unused — the agent will
**report its own correct behaviour as a defect**, and send someone to fix
working code.

## The incident

A permission change gave one agent direct read access to a previously-denied
vault when running on one specific machine. It shipped with the mechanism
complete: a policy block, a hook honouring it, and a new numbered rule
documenting the lane. 131 tests passed.

What it did not touch was the always-loaded `CLAUDE.md`, which still asserted:

> the deny hook is **exact on the structured tools** (`Read`, `Grep`, `Edit`,
> `Write` — it inspects a real path field)

framed as *"Measured, not theorized."* Top of the file, restated in a table one
line below.

Both statements were in context on every session. They were not in different
layers; nothing resolved one before the other. They simply disagreed, and one
was louder — top of the first-read file, emphatic, repeated — while the new
rule was a single paragraph at position 91 of 91 in a long list.

The agent went with the louder one, twice in one sitting:

1. Asked a question the new grant was built for, it **delegated to a gated
   worker** instead of reading directly — because per the doctrine it was
   reading, delegation was still the only legal path.
2. Forced onto the direct path by an explicit instruction, it read the file
   **exactly as the new rule allows** — and then told its human the gate had
   failed, that a private file had been opened with nothing stopping it, and
   that they should check whether the deny hook was still wired. It quoted the
   stale sentence as its evidence.

Nothing was broken. The agent filed a false security report against itself, and
its recommended fix would have removed the feature.

## The Pattern

**A capability lives in two places: what the system permits, and what the agent
believes the system permits.** The second is prose, it is rarely diffed
alongside the mechanism, and when the two disagree the agent does not detect a
contradiction — it acts on whichever statement is more prominent, and reasons
confidently from it.

The second symptom is the one worth naming, because it is counter-intuitive and
expensive: **an agent that does not know its own permissions will report correct
behaviour as a breach.** A false negative (the grant goes unused) merely wastes
the feature. A false alarm burns the human's trust in the mechanism itself,
arrives with a plausible citation, and points maintenance at the wrong file.

Three properties make it hard to catch:

- **Prominence beats recency.** The new rule was newer, more specific, and
  correct. It lost to position and emphasis. Writing it *harder* would not have
  helped; the old sentence would still be first.
- **The test suite cannot see it.** Every test asserted the mechanism's verdict
  for a synthetic call. None asserted that the agent *attempts* the call, and
  none asserted what it *believes* about the result. A gate that correctly
  returns `allow` to a request nobody makes passes everything.
  (`a-source-assertion-pins-your-belief-not-the-behavior.md` is the same blind
  spot one layer down.)
- **The false alarm is well-argued.** It cited a real file, quoted it
  accurately, and drew a sound conclusion from a stale premise. It reads like
  diligence, which is exactly why it gets believed.

## The discipline

- **A permission change is a documentation change.** Before shipping, grep for
  every sentence asserting the old behaviour and reconcile each one. The list is
  usually short and always longer than one.
- **Prefer generated prose over hand-written prose for anything derived from
  policy.** If the deny rules are already generated from a policy file, the
  sentence describing them should be generated from the same source. Prose that
  cannot go stale does not need reconciling. This is the durable fix; the greps
  above are the version you can do today.
- **State the carve-out where the absolute claim lives**, not only in the new
  rule. The reader who needs it is reading the old sentence.
- **Test the belief, not just the gate.** At minimum, one check that the agent
  reaches for the newly-permitted path unprompted. A human driving a session
  once is a valid version of this and is better than nothing.

## When It Shows Up

Any agent whose permissions are configuration but whose understanding of them is
prose: permission grants, tier or trust changes, a new environment or host, a
capability enabled per-machine or per-session. The risk scales with how emphatic
the original prohibition was — a rule written with hard-won conviction
(*"measured, not theorized"*) is exactly the one a later grant will lose to.

The tell in review: a diff that adds a capability and touches no documentation
outside its own new section.

## Related

- `a-rule-must-live-where-matching-happens-first.md` — the layered-ordering
  cousin. There a rule never fires because something above it already decided;
  here both rules are loaded at the same layer and the more prominent one wins.
- `a-source-assertion-pins-your-belief-not-the-behavior.md` — why the suite
  stayed green: the assertions guarded the mechanism, not the conduct.
- `contradiction-surfacing.md` — the audit pass built to find exactly this class
  of disagreement, and the flag-don't-fix discipline for handing it back.
- `a-summary-can-invert-the-policy.md` — the adjacent failure where a compressed
  restatement of a rule reverses its meaning.
