---
type: pattern
date: "2026-07-21"
source: The personal agent PR #40 (day-walk image harvester), caught in the 2026-07-21 consolidation review
tags:
  - code
  - safety
  - anti-pattern
---

# Decorative Gate (Anti-Pattern): A Control That Returns "Passed" Without Checking

A "privacy gate" shipped as `{"privacy_gate": {"status": "passed_privacy_gate"}}` — a hardcoded string in a function that returned mock data. No tier check, no file inspected, nothing refused. Every caller, every log line, every reviewer skimming the API response saw a control that had run and passed. There was no control.

This is worse than having no gate. An absent control is a visible hole someone eventually fills; a decorative one *retires the pressure to build the real thing* while providing none of its protection. It is the security twin of a test with no assertions.

## The Pattern

A gate is only real if all three hold:

1. **It can refuse.** There is a reachable code path that returns deny/blocks the action, and a test exercises it. If no input can make the gate say no, it isn't a gate.
2. **It fails closed.** The refusal is the default for anything unrecognized — unknown tier, missing manifest entry, unresolvable path. "Passed" is the *earned* branch, never the fallback.
3. **Its verdict is derived, not declared.** The status field is computed from the check that just ran, never a literal in the source. `grep` for the passing status string: if it appears verbatim in code, the gate is decorative.

Review heuristic that caught this one: for any claimed control, ask "show me the input that makes it refuse, and the test that proves it." The personal agent fix was a selftest asserting a gated-tier item *cannot exit* the exporter.

## When It Shows Up

Agent-generated code is especially prone: the spec says "with a privacy gate," and the generator satisfies the *shape* of the requirement (a field named `privacy_gate`) without the substance. Same failure family as claimed-verified-but-unexercised work — the artifact exists, the behavior doesn't.
