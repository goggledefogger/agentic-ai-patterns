---
type: pattern
date: "2026-08-09"
source: The personal agent — walk-and-talk stand-up debugging; `head -30` of a config concluded a key was absent, the key was on a later line, and the false absence became the premise of a diagnosis, a code change, and a written debt entry
tags:
  - debugging
  - agents
  - anti-pattern
  - evidence
---

# A Partial Read Proves Presence, Never Absence

A cheap read — `head`, `tail`, the first screen of a file, the first page of
results — is perfectly good evidence that something **is** there. It is no
evidence at all that something **isn't**. The asymmetry is total, and it is easy
to forget precisely because the read felt like looking.

## The Problem

Diagnosing why a walk's phone bridge and watchdog never started, a session read
the first thirty lines of `config.yaml`, saw no `runtime:` line, and concluded
the key was missing. The consuming code gates on `case "$_rt" in phone|both)`,
so "key absent" cleanly explained the silence. That explanation was written into
a debt entry, used to justify a code change, and repeated in a commit message.

`runtime: both` was in the file the whole time, below line thirty.

The correction is not the interesting part. This is: when the entry was revised,
the revision **made the same class of error again** — it substituted a second
partial-read conclusion for the first — and it was written while the author was
literally documenting confident partial reads as a failure mode. Being mid-way
through describing the trap conferred no immunity to it.

## The Pattern

Absence is a strong claim and needs strong evidence, because absence is almost
never the end of a chain — it is the *premise* of "and therefore the code took
the other branch."

1. **Match the read to the claim.** Proving presence: any read that finds it.
   Proving absence: the whole file, or an exhaustive search over it (`grep -c`
   across the entire file, not `head | grep`). Cheap either way; the cost is
   remembering which claim you are making.
2. **Notice when absence became load-bearing.** A conclusion that begins "there
   is no X, so Y never ran" has promoted a negative observation to a causal
   premise. That promotion is the moment to go back and read properly.
3. **Prefer the direct observation over the inferred one.** "The watchdog is not
   running" (`pgrep`) is a fact. "The config lacks the key that would have
   started it" is a theory about that fact. When both are available, the theory
   must be checked against the file, not assumed from a glance.
4. **A tidy single cause is a warning, not a reward.** The false absence
   explained everything at once, which is exactly why it survived a full
   write-up. Elegance is a property of stories, not of systems.

## When It Shows Up

Any agent reading files with bounded tools: `head`/`tail` limits, line-capped
readers, truncated tool output, paginated APIs, a `grep` whose pattern was
almost right. It is endemic to long-context agents specifically, because the
economical habit — read a slice, not the file — is *correct* for most questions
and silently wrong for this one.

The tell in review: a conclusion of the form "X is not present" whose supporting
command had a limit in it.

## The harder case: a limit you did not write

The tell above — "a conclusion whose supporting command had a limit in it" —
assumes you can see the limit. Three incidents in one session, 2026-08-30,
where nobody wrote a limit and the scan was narrowed anyway:

- **A shimmed tool.** `grep -r` on that machine was a shell function routing to
  `ugrep --ignore-files`, so it honoured `.gitignore` and silently skipped
  `Inbox/`. It reported **1 match where there were 126**, which looked exactly
  like 125 failed writes. Nothing in the command said "and skip some
  directories," and `grep` is the tool you reach for precisely because you
  believe it looks everywhere.
- **A fixed window inside a helper.** A retagging script read only the first
  20 lines of each file to find `region:`. Notes with deep frontmatter (brand,
  material, sku, diameter, weight) carried it below the window, so **23 of 81
  notes were reported done and were not**. The script's limit was real and
  invisible from its output, which said only "retagged 58."
- **A count whose scope moved.** The same quantity — untagged notes — was
  reported as 243, then 1,954, then 7,895 within a few hours. Every figure was
  arithmetically correct. Each scanned a different set, and each was presented
  as the total.

The unifying move is the same in all three: **a count is only as wide as its
scan, so state the scan with the count.** "7,895 untagged across 8 stores,
`.md` only, frontmatter anywhere in the file" survives being wrong in a way
"7,895 untagged" does not, because the next reader can see which assumption
to attack.

Two habits fall out of it. **Verify a write by re-counting after it, never by
trusting its own report** — that is what caught the 58/81 gap. And when a
number is load-bearing, **get it twice by different means**; the disagreement
is the finding, and two tools agreeing is the only cheap evidence that neither
was silently narrowed.

## Related

- `a-fast-answer-is-a-suspect-answer.md` — the speed of the answer as the signal.
  This is its narrower, sharper case: the fast answer was a *negative* one.
- `unrun-checks-read-as-passing.md` — absence of a signal misread as a verdict.
  Same asymmetry, one layer out: there, nothing ran; here, nothing was seen.
- `stale-pointer-asserts-confidently.md` — evidence that is wrong rather than
  incomplete, asserted with the same confidence.
