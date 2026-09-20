---
type: pattern
date: "2026-08-05"
source: The household-agent repo deploy-to-pi.sh — a cron whose description contained an apostrophe failed to install for months behind a guard written for the *other* non-zero exit
tags:
  - code
  - safety
  - anti-pattern
  - verification
  - operations
---

# Tolerance Named the Failure, Not the Benign Set (Anti-Pattern)

Some steps return non-zero on a perfectly normal path. A registry refuses a
duplicate. A linter finds nothing to fix. A sync reports "already up to date."
The honest response is a tolerance: *this code is fine, keep going.* The
tolerance is where the bug gets planted — because it is almost always written
as **"only code X counts as a failure"** rather than **"only codes A and B
count as success."** Every failure mode nobody had met yet inherits the
exemption.

## The Problem

A deploy registered ~31 cron jobs by SSHing an `add-cron` call per job. That
helper returns **1** on its normal idempotent path: the job already exists, so
it refuses the duplicate. Every re-deploy of an unchanged file therefore exits
1 on every line. A plain `rc != 0` check would have marked routine deploys
broken, so the code grew a tolerance, and the tolerance was written as:

```bash
cron_step_failed() { [[ "${1:-0}" -eq 255 ]]; }   # 255 = ssh transport death
```

Reasonable, documented, and load-bearing — it names the one failure the author
had actually seen.

Then one job's *description* grew apostrophes: "the Pi's durable log dir",
"the script's default". The remote command was built as `ssh host "... add-cron
'$sched' '$cmd' '$desc'"`, so the apostrophe closed the quote early and the
remote shell died parsing the rest — exit **2**.

Two is not 255. The guard said "not a failure." The deploy printed one
`syntax error` line amid ~5,000 lines of output and **exited 0**. That cron —
a six-hourly media job — could not be installed on a rebuilt machine, and
nothing anywhere said so. It survived because the box already had it from
before the description was edited.

The tolerance built for the benign non-zero swallowed a real one with the same
gesture, and only the author's imagination separated them.

## The Pattern

**Enumerate the codes that mean success. Everything else is a failure, including
the ones you have not met.**

```bash
# The helper has exactly two exits: 0 (added) and 1 (already exists).
cron_step_failed() { local rc=${1:-0}; [[ "$rc" -ne 0 && "$rc" -ne 1 ]]; }
```

Now 2 (remote parse error), 127 (script missing) and 255 (transport) all
surface, and so does whatever the next unknown is.

Three parts:

1. **Read the callee's exits, don't guess them.** Grep the thing you are
   tolerating for every `exit`/`return` it can produce. Here it was two codes
   and the whole allowlist was three characters wide. If a callee's exit set
   is genuinely open-ended, that is a finding about the callee.
2. **Report the code you actually got.** The old skip note asserted "ssh
   transport failure (255)" — the number was hardcoded into the *message*, so
   even after widening, a reader would have been told the wrong cause. Print
   the observed rc.
3. **Make the benign path cheap to distinguish.** If "already exists" and
   "catastrophe" share an exit code, the tolerance cannot be written safely at
   all, and the fix belongs in the callee.

## When It Shows Up

Anywhere a wrapper absorbs an expected non-zero: `|| true` on an idempotent
step, `grep -q ... || true`, `set +e` around a retry, `diff` used as a
predicate (1 = differs), `git diff --quiet`, package installs that return
non-zero for "nothing to do", HTTP handlers treating 404 as normal and
inheriting silence on 500.

The tell is a comparison against a **specific failure value** — `-eq 255`,
`== "ENOENT"`, `if err.code == 404` — inside a helper whose job is deciding
whether something worked. Invert it and see whether the inverted form is one
you could defend.

Sibling to [[half-gate-whole-verdict]]: there the evidence covered one axis
while the verdict named two; here the exemption covers one code while the
verdict covers all of them. Both are sound checks whose *scope* is narrower
than the claim they license. Related to [[unrun-checks-read-as-passing]] — in
that one a check never ran; in this one it ran, returned honestly, and was
graded against the wrong rubric.

## Related

- [[half-gate-whole-verdict]] — evidence for one axis, a claim covering both
- [[unrun-checks-read-as-passing]] — silence from a step that never executed
- [[unenforced-red-becomes-the-baseline]] — the other way a failure signal stops
  meaning anything: not exempted, just endured
- [[decorative-gate]] — a check whose pass does not mean what it claims
- [[guard-evidence-outlives-the-failure]] — keeping the proof a guard fired
- [[quote-survives-the-re-parse]] — the bug this tolerance was hiding

## The Rule of Thumb

Write the allowlist, never the denylist. If you find yourself typing "the
failure is code X," you have just decided that every other failure is success.
