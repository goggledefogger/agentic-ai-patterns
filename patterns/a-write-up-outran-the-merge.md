---
type: pattern
date: "2026-09-02"
source: The household-agent repo — the deploy-script session-reset fix (PR #274) sat open and unmerged for seven days while its pattern write-up, its RESOLVED ledger entry, and a CLAUDE.md correction all read as done; the same day a second fix (67c4e4d) sat committed-but-unpushed on a peer clone for 24h and was nearly re-authored worse
tags:
  - process
  - anti-pattern
  - operations
  - drift
  - agents
---

# A Write-Up Outran the Merge: A Documented Fix Is Not a Landed Fix

On 2026-08-26 a deploy script was found resetting the assistant's live
conversation on every run. It was diagnosed properly, fixed on a branch,
covered by tests, proven by process identity, written up as a pattern
(`an-unconditional-act-wearing-a-conditional-name`), recorded in the
project's deferred-work ledger as **RESOLVED — PR follows**, and the
`CLAUDE.md` sentences that had mis-described it were corrected.

Every artifact that a reader consults to answer "is this fixed?" said yes.
The PR was never merged. Seven days later the deploy still reset the
session on every run — three wipes in twelve hours with nothing behavioural
changed — and the session that noticed it began, from scratch, to diagnose
a bug whose diagnosis, fix, tests and proof already existed on a branch
thirteen commits behind main.

The same day, the same shape one hop over: a concurrency fix for a media
encoder had been committed on a peer machine's clone and never pushed. The
production box was already running it (deployed from that clone), the drift
check reported the production copy as "newer than the repo", and the
reviewing session read that as a hand-edit and drafted a replacement — worse
than the one that already existed, because it had not seen it.

## The Pattern

A fix exists in several places before it exists where it runs. Each of
those places can answer the question "has this been fixed?" — and each
answers *yes* as soon as the fix exists **there**:

| where the fix exists | what a reader sees | what production sees |
|---|---|---|
| a pattern write-up | "root-caused and fixed, here's the proof" | nothing |
| a ledger row marked RESOLVED | "done; PR follows" | nothing |
| a commit on a branch | `git log` shows the subject | nothing |
| a commit on a peer's clone | the deployed box runs it | the *repo* does not |
| a merged commit on main | — | the fix |

The write-up is the most dangerous of these, because it is the one written
*to be found*. A reader who searches for the symptom lands on a confident,
well-evidenced document and stops. The document is not wrong — every claim
in it was true on the branch — it is simply about a different tree than the
one that is running.

Three checks, mechanical:

1. **A "fixed" claim cites a ref reachable from the deployed branch.** A
   merge SHA on main, not a PR number, not a branch name, not "PR follows".
   `git branch --contains <sha>` must list the branch you deploy from. A
   ledger status of RESOLVED without a landed SHA is a promise wearing a
   verdict's label.
2. **Distrust the write-up in proportion to how well it explains the
   symptom.** This is the same rule as
   `an-unconditional-act-wearing-a-conditional-name` applied to the
   documentation *of the fix* rather than of the bug: an accurate account
   of the mechanism plus a plausible remedy is indistinguishable from a
   landed remedy, and it retires the investigation just as effectively.
   When the symptom is live, the write-up's date is the first thing to
   check, and `git log main --since=<that date> -- <file>` is the second.
3. **Enumerate the trees, not just the branches.** "Where else might this
   fix exist?" has more answers than `git branch -a`: every clone that can
   deploy is a tree, and a commit there reaches production without ever
   reaching the repo. The drift check that compares repo↔production cannot
   see it — it reads a peer's deploy as a hand-edit on the box. Mechanized
   in the source project: the session-start snapshot now fetches each
   known peer clone and prints commits ahead of `origin/main` plus
   uncommitted files and stashes, fail-closed when a peer is unreachable.

## Why It Persists

- **The author's job ended at the write-up.** Diagnose, fix, test, prove,
  document — the natural stopping point is the document, because that is
  the deliverable a reader will see. Merging is a separate, boring step
  with no artifact of its own.
- **Nothing decays.** An open PR does not get louder with age. A RESOLVED
  ledger row does not revert to OPEN when its promised merge fails to
  appear. The symptom keeps happening, but each new observer meets the
  same reassuring paper trail.
- **The fix's own tests pass — on the branch.** Green tests are evidence
  about a tree; they say nothing about which tree is deployed. See
  `version-bump-is-the-delivery` for the next hop of the same mistake:
  merged is also not shipped when a version pin sits between main and the
  installed copy.

## When to Use

- Before diagnosing any symptom that has a write-up, a ledger entry, or a
  memory file describing it as fixed: find the landed SHA first. If there
  is none, the write-up is the first suspect, not the last.
- When closing a ledger entry: the status flips on the merge, not on the
  PR. "RESOLVED — PR follows" is not a state; leave it OPEN with a pointer
  to the branch.
- When a drift check says production is *newer* than the repo: ask which
  trees can deploy to production before concluding someone edited
  production by hand.

## Adjacent Patterns

- `an-unconditional-act-wearing-a-conditional-name` — the bug this fix was
  for; its write-up is the artifact that outran the merge.
- `version-bump-is-the-delivery` — merged ≠ shipped under a pinned cache;
  this pattern is one hop earlier: documented ≠ merged.
- `a-second-writer-satisfies-your-gate` — a peer clone is a second writer
  to the production tree; the repo↔production drift gate cannot name which
  writer moved it.
- `a-source-assertion-pins-your-belief-not-the-behavior` — a confident
  claim about code, sourced from reading rather than running, is the same
  failure at the sentence level.
