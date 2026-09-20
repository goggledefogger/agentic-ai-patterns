---
type: pattern
date: "2026-08-07"
source: The personal agent session 2026-08-07 — porting the tier gate into an Antigravity IDE pane
tags:
  - testing
  - verification
  - anti-pattern
  - agent-safety
---

# Green tests can mirror the same guess

When one agent writes both the code and its tests, the tests verify self-consistency, not the contract. All-green proves the two halves agree with each other, it does not prove either agrees with the world.

A builder that invents a detail invents it once, in the code, and then confirms it forever, because the fixtures it writes next are downstream of the same invention. The suite is not lying. It is answering a question nobody asked: do these two files agree with each other.

## The incident

A builder subagent was given the documented contract for a foreign IDE's hook payload (key: `args`) and its hook-registration schema, both quoted in the brief. It implemented against a key it invented (`arguments`) and a registration schema it invented, then wrote its test payloads to match its own inventions. Eleven tests, all green, report clean. Every real payload the IDE would ever send would have missed the translation entirely. The defect surfaced only when the reviewing session piped a payload built from the docs, not from the code, into the binary: the answer came back wrong-reason, and the invented key was three greps away. Two more instances turned up in the same build: the registration file schema, and a probe document that told the system under test what outcome was expected, inviting a role-played pass.

## The Pattern

1. **The reviewer's first fixture comes from the specification, never from the implementation or its tests.** One payload hand-built from the docs, piped into the real binary, is worth more than the whole self-written suite.
2. **Adversarial reviewers must be told explicitly to check whether the tests test the documented contract or merely mirror the implementation.** Nobody does this by default because a passing suite doesn't ask to be re-derived.
3. **A probe or acceptance prompt must withhold the expected outcome from the thing being probed.** A verifier handed its own answer verifies nothing.

## Why it stays invisible

- **Green is the strongest possible signal of done.** Nobody re-derives fixture provenance from a passing suite.
- **The report can quote the correct contract while the code contradicts it.** The builder knew the documented key and drifted anyway, so reading the report cannot catch it, only running against the spec can.

## Watch-outs

- **The same session that caught the invented key then hid a real test failure behind `tail -3`.** Truncated test output turned an exit-1 suite into an apparent pass for two rounds. Reading only the tail of a test run is the reviewer's edition of the same disease, judging by a fragment that agrees with your expectation. Always check the exit code or the full summary line.

## When NOT to use

If there is no external contract to check against, no docs, no spec, no other team's schema, this doesn't apply. Self-written tests against self-invented behavior are the only source of truth there is in genuinely greenfield code. The failure mode is specific to an agent handed a documented contract that drifts from it while its own tests agree with the drift.

## Adjacent Patterns

- `a-fake-can-only-fail-the-ways-you-have-seen` — same shape, a fixture can't express what nobody wrote into it, here the fixture invented its own contract to begin with
- `a-capability-contract-must-name-the-verb` — same week, same harness
- `the-native-deny-list-beats-the-ported-gate` — same build, second act

## Source

The personal agent session 2026-08-07, porting the tier gate into an Antigravity IDE pane. A builder subagent implemented against an invented hook-payload key and an invented registration schema, then wrote its own tests to match both inventions, eleven tests green. The defect surfaced only when the reviewing session built a payload from the documented contract instead of the code and piped it into the binary.
