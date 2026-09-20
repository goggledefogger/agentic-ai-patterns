---
type: pattern
date: "2026-07-28"
source: The personal agent 2026-07-28 session — the agent-aware registry column: zero yes rows, scattered no rows, blanks on the very resources the policy cared about
tags:
  - registry
  - safety
  - anti-pattern
---

# Undefined Blank Is a Decision (Anti-Pattern): An Optional Column Without Blank-Semantics Becomes Decoration

A registry grew a policy column — which downstream resources may know the orchestrator exists. The column had a name, a few explicit "no" rows, and blanks everywhere else, including on the exact resources the owner wanted flagged "yes." Nobody could say what a blank meant: not-decided? default-no? default-yes? The lint treated the column as optional and validated nothing, so any string — a typo, a "maybe" — would have parsed as silently as the blanks did. The column *looked* like governance and governed nothing: a schema-level decorative gate.

The failure has two independent halves. Blanks with no defined meaning make the common case unreadable — and since most rows are blank, the column's dominant value is "shrug." And an unvalidated cell means even the filled rows are untrustworthy: a value that would fail review passes silently, wearing the shape of a decision.

## The Pattern

1. **Define blank explicitly, in the file itself.** A legend line — "blank means NO, and no is the default" — converts every existing blank from ambiguity into policy retroactively, no row edits needed. Pick the fail-closed reading for anything safety-adjacent.
2. **Lint the vocabulary.** Enumerate allowed values and fail on anything else: the typo that would silently read as "not decided" while looking like a decision becomes a build break. (One ternary check; the lint's own selftest feeds it a `maybe`.)
3. **Gate the widening transitions.** If one value grants more than another (aware > not-aware, writable > read-only), the widening flip is an approval event with provenance — same posture as a tier downgrade, and the approval lives in git history, never in the cell.
4. **Populate the known rows the moment the semantics land.** A policy column with defined blanks and zero affirmative rows is still untested doctrine; the first yes rows are what prove the vocabulary works.

## When It Shows Up

Any table that accretes columns faster than semantics: access flags, review states, feature toggles in config files. The tell is a column someone added with clear intent, that appears in the schema and the parser, and that no query or hook ever branches on — its blanks have been "meaning something" different to every reader since the day it shipped. Sibling of `decorative-gate.md` (control-shaped, no control) and `unrun-checks-read-as-passing.md` (absence misread as verdict), one level down: the schema itself is the decoration.
