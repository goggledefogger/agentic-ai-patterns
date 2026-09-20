---
type: pattern
date: "2026-07-27"
source: The agent's own repo, a config and a script, two fail-fast CI outages, 2026-07-26 and 2026-07-27
tags:
  - testing
  - ci
  - monitoring
  - anti-pattern
---

# A Check That Never Ran Reads As Passing

A suite of twenty checks lived in one CI step:

```yaml
- name: hook tests + hermetic selftests
  run: |
    python3 hooks/test_hooks.py
    python3 scripts/dispatch.py --selftest
    ...eighteen more...
```

A `run: |` block is `bash -e`. The first failing line ends the block, and every line after it never executes. The run reports one thing: *this step failed*. It says nothing about the eighteen checks below the failure, and nothing is what everyone hears as *fine*.

Twice in two days that cost real coverage. On 2026-07-26 the block died at check 25 of 36, so eleven checks — including the guard on a fix that had shipped that same morning — had not executed on a runner for over a day. On 2026-07-27 it died at check 9 of 21 and buried the other twelve. In both cases the checks were healthy; nobody knew, because "did not run" and "ran and passed" are the same shape from outside: absence of a failure message.

## The Pattern

Two halves, and the structural one matters more.

**1. One check, one independently-reported unit.** Split the block so the platform reports each check's own conclusion, and mark each so a sibling's failure does not suppress it:

```yaml
- name: dispatch.py
  if: ${{ !cancelled() }}
  run: python3 scripts/dispatch.py --selftest
- name: scheduler.py
  if: ${{ !cancelled() }}
  run: python3 scripts/scheduler.py --selftest
```

`!cancelled()` rather than `always()`: a cancelled run should stop, a failed one should keep going. The job still goes red on any failure — nothing is weakened. What changes is that a red run now names *every* check that failed and proves the rest actually ran.

This is a platform affordance, not a framework: GitHub Actions steps, `pytest` without `-x`, `make -k`, a shell loop that records `|| fail=1` and exits at the end. Reach for the native one before writing a harness.

**2. Any watcher counts executed-vs-total, never the exit code alone.** A single conclusion cannot distinguish "everything ran and passed" from "a fraction ran." So the check that reports on the suite reports two *different* facts, because they need different reactions:

- **Red** — a check failed. Name which ones.
- **Ran only N of M** — the run published a conclusion covering checks that never executed. An unexecuted check is **UNKNOWN**, never green.

Collapsing those into one line loses the second, which is the one that hides.

## Why the second half is not redundant

Splitting the steps makes today's runs honest; the counter is what notices when they stop being honest. A step added later without the guard, a runner that dies mid-job, a `paths:` filter someone introduces — each silently shrinks what a green tick covers, and each is invisible to a reader who only sees the tick.

There is a denominator trap here worth naming. Counting the run's *own reported* steps works while the run exists, and platforms often omit steps that never started, so the reported list shrinks with the coverage. The count is honest about a green run and needs a second source — the workflow definition, or the commit the run claims to cover — before it can speak about a run that ended early or never happened at all.

## Not all absence is equal, and blanket UNKNOWN is how the rule dies

"An unexecuted check is UNKNOWN, never green" is right, and applied without a second distinction it destroys itself.

A session-start digest in `the household-agent repo` — a block that replays every flagged line from a ~350-line health run, built precisely so flags stop being missed — printed `✓ nothing flagged this run` on a run where one of two hosts sat behind an expired auth prompt and its whole section was skipped: no messages, no health, no drift, no cron. A green verdict over a host nobody looked at. The same shape as the CI case one layer up: the digest counted flags and never asked what it had covered.

The obvious fix is to raise UNKNOWN on any uninspected host. That would have been wrong, because the *other* host in that fleet is offline permanently and by design. Flagging it would have fired a warning on every run forever — and a warning that fires every run is read as decoration within a week. The rule would have been technically satisfied and practically dead, inside the very digest built to stop things being missed.

So coverage loss splits in two, and they get different renders:

- **Expected absence** — a decommissioned host, a suite skipped by a deliberate `paths:` filter, a probe that doesn't apply to this platform. Report as *coverage information*: unmissable, not an alarm. `— coverage: 1 host not inspected: hopper (OFFLINE)`
- **Unexpected absence** — auth expired, a runner died, a step vanished. A flag, with the action attached.

Both must break the all-clear. Only the second may cry wolf. The load-bearing change is that `✓ nothing flagged this run` became **unreachable** unless coverage was complete — the clean string now asserts what readers always took it to mean, and the two kinds of gap differ in loudness, not in whether they appear at all.

The tell that you need this split: someone proposes suppressing the new UNKNOWN for a specific known-benign case. That impulse is right and the suppression is wrong — it wants a quieter render, not a missing one.

## The failure this rhymes with

The same shape appears wherever a partial result is published under a whole-population label:

- A migration that stops at row 400 of 10,000 and logs "completed."
- A sweep with a silent `top-N` cap, reported as coverage.
- A test suite whose collection errors are warnings, so a file that fails to import is a file that always passes.
- A monitor with a filter that quietly matches nothing.

The tell is always the same: **the absence of a complaint is being read as evidence of health, by a mechanism that could not have complained.**

## Verification

Test the silence, not just the alarm. A watcher's assertions must include a healthy suite producing **no** line — without that control, every other assertion also passes on a nudge that fires unconditionally, and a line that always fires is one the reader learns to skip. Then exercise it against a genuinely red run and read the output, rather than trusting that the code that prints the alarm would have.

## Adjacent Patterns

- **Verification needs a negative control** (`verification-needs-a-negative-control.md`) — the sibling failure. There a green cannot fail; here a green speaks for work it never did. Both are passes that carry no information.
- **Registry-based monitoring** (`registry-based-monitoring.md`) — where the watcher belongs. A red CI run is the same shape of signal as a scheduled task that failed, and wants the same channel.
- **Wire into existing flows** (`wire-into-existing-flows.md`) — a CI watcher nobody runs is another standalone artifact. It earns its keep by riding a surface that already fires, such as session start.
- **A plausible cause ends the search** (`plausible-cause-ends-the-search.md`) — the reason the first outage ran twelve deep: one named symptom looked like the whole story.
