---
type: pattern
date: "2026-09-02"
source: The household-agent repo — the hub deploy's provenance gate hashed every shipped file, refused correctly on a Pi-side edit, and printed a hand-written recovery (`scp src/ + Dockerfile`) that did not cover the file that had changed; following it would have recovered nothing and the sanctioned --force would have erased the agent's work
tags:
  - code
  - safety
  - anti-pattern
  - operations
  - agents
---

# A Recovery Hint Must Derive From Its Detector

A deploy gate compared a content hash of the production tree against the
hash stamped at the last deploy, and refused when they differed — the
right call, proven three times on real edits. Under the refusal it
printed what to do about it:

```
Recover it into the repo FIRST — there is no --pull for the hub:
  scp -r pi:~/apps/hub/src/. ./apps/hub/src/
  scp    pi:~/apps/hub/Dockerfile ./apps/hub/
Then re-deploy. Or --force to overwrite.
```

The detector hashed `*.py`, the Dockerfile, the package manifests,
`static/**` and `src/**`. The hint named two of those. On 2026-09-02 the
agent's edit was `house_calendar.py` — a root-level Python file, hashed by
the detector, absent from the hint. An operator following the printed
recovery verbatim would have copied two unchanged things, re-deployed, hit
the same refusal, and reached for the sanctioned `--force`, which would
have overwritten the one file that mattered. The gate would have blocked
correctly and then guided the operator into the exact loss it existed to
prevent.

What found the file was a whole-tree `diff -rq` run by hand, because the
operator distrusted the hint. Nothing in the tooling would have.

## The Pattern

A control that enforces by *telling a human what to do* is a detector plus
a message (`a-gate-can-block-but-cannot-speak`). That pattern is about the
message being **lost**. This one is about the message being **wrong** —
and wrong in the specific way hand-written remediation text always drifts:
it describes the surface the author had in mind when the gate was written,
while the detector describes the surface as it actually is.

The two were written from different sources. The detector's file list was
code — one function, tested, shipped to both sides of the comparison via
`declare -f` precisely so "the gate hashes a different set than the deploy
ships" could not happen. The hint was prose, typed from memory of the case
that motivated the gate (a JavaScript component and a Dockerfile), and
never derived from anything.

Three checks:

1. **Generate remediation from the detector's own data.** If the gate
   computed a hash, it can compute — or store beside the marker — the
   manifest the hash was built from. The refusal then diffs two manifests
   and prints the paths that moved, and the recovery is one `scp` per
   printed path. There is no second list to drift. When no stored
   manifest exists yet, fall back to a diff against the repo *and say so*,
   because that view also shows repo-side edits the operator did not make.
2. **Positive-control the message, not just the verdict.** The gate's
   test suite covered every row of the decision matrix and none of the
   recovery text. Adding one file to the production tree and reading the
   hint back is a thirty-second control; it would have shown the hint
   naming nothing while the verdict said "edited". A message that cannot
   be exercised is a message that has not been tested.
3. **A hint that names a subset invites the override.** Every gate ships
   with an escape hatch, and the operator reaches for it when the printed
   recovery has visibly failed. A partial hint therefore does not merely
   under-help — it manufactures the failure the override is dangerous
   for. When a recovery cannot be derived, print *that* ("could not name
   the files; diff the tree by hand before any --force") rather than a
   confident partial list.

## Why It Persists

- **The verdict gets the attention.** The interesting engineering is in
  the decision — three-way hash comparison, fail-closed on an unreadable
  probe, backup on forced overwrite. The remediation string is the last
  line typed, after the hard part is done and the motivating case is
  still the only case anyone has seen.
- **The hint is only read under duress.** Nobody reviews remediation text
  in the calm; it is read by someone whose deploy just failed, who wants
  the shortest path out. That reader is the least likely to notice the
  text describes a different situation than theirs.
- **The first few real firings match the hint** because they are the
  motivating case. The hint's drift is invisible until the surface widens
  — a new file type, a new writer — which is exactly when it matters.

## When to Use

- Any gate, lint, or check that prints "to fix this, do X": ask where X
  came from. If it did not come from the same data the check used, it
  will eventually name the wrong X.
- Reviewing a refusal's text: add the file the check is about, read the
  message, see whether it names the file. If the check can detect a
  change it cannot name, that gap is the finding.
- Designing the escape hatch: the override's danger is proportional to
  how confidently the hint points the wrong way. Make the hint honest
  about its own limits before making the override convenient.

## Adjacent Patterns

- `a-gate-can-block-but-cannot-speak` — the message dropped in transit;
  here it arrived intact and was wrong.
- `a-partial-read-proves-presence-not-absence` — the hint's two paths were
  a partial read of the shipped surface, offered as the whole.
- `a-second-writer-satisfies-your-gate` — the second writer here was the
  agent on the production box; the gate detected it and could not describe
  what it had written.
- `verification-needs-a-negative-control` — the manifest fallback's first
  live run listed forty Pi-only static assets as "added" and buried the
  one real edit; the control that showed it was the same probe file, read
  back through the message.
