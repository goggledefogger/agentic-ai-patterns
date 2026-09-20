---
type: pattern
date: "2026-08-06"
source: The personal agent session 2026-08-06 — job rejection routed from an Antigravity IDE pane
tags:
  - agent-safety
  - sandboxing
  - tool-permissions
  - discovery
  - worker-dispatch
---

# A worker that cannot list cannot find

Discovery and content are separate capabilities. A sandboxed worker allowed to read a file but not to enumerate its directory can only guess-and-check.

Locking down a worker usually means trimming the allowed-tool list until only the necessary tools remain. Read/Write/Edit looks complete — the worker can consume and produce files inside its authorized root. But Read takes a path you already know. Without Glob or an equivalent listing tool, a worker facing a directory it is fully authorized to read cannot learn what is in it. It falls back to constructing plausible filenames and reading them one at a time until something hits, and reports failure when nothing does.

## The incident

A worker authorized to write a whole vault burned three dispatches unable to locate one file in a directory it could already read. It knew the naming convention. It tried six variants of the filename. The real name differed by a single date field — filed under a sourcing date, not the submitted date the task described. One Glob call resolved it immediately.

## The Pattern

1. **Ask the discovery question and the content question separately.** When trimming a worker allowlist, ask whether it can (a) reach the bytes and (b) learn what exists — they are different capabilities and get cut independently.
2. **Be strict on egress, not on discovery.** Egress (shell, send tools) is the axis to guard. A listing tool inside a root the worker is already authorized to read costs nothing on that axis.

## Why it stays invisible

- **The containment reasoning is sound and stays sound.** Glob returns names, not contents; every path-level deny still applies; a worker with no shell and no send tools still cannot exfiltrate.
- **It was omitted for tidiness, not for safety.** The cost only appears the first time a task needs a filename nobody hardcoded.

## Watch-outs

- **This failure reports as a task failure, not a permissions failure.** The worker says "I could not find the file," which reads as a bad path in the brief. The gate is invisible in the symptom. If a worker can't find something you're confident exists, check its tool list before rewriting the brief.

## When NOT to use

If the worker needs to discover things outside its authorized root, that's not a missing-Glob problem — it's a scope boundary, and the fix is deliberately widening what the worker can reach, not handing it a listing tool that reaches past its own containment.

## Adjacent Patterns

- `a-capability-contract-must-name-the-verb` — same session, the sibling capability gap: the executable route withheld instead of discovery
- `escape-hatch-is-the-denied-tool` — the same shape of invisible gate: correct containment on both sides, empty intersection in the middle
- `lawful-write-surface` — the write-side counterpart: giving a worker somewhere legal to act, not just something to read

## Source

The personal agent session 2026-08-06. An job rejection, routed from an Antigravity IDE pane, surfaced a worker that could read a file it couldn't find, because the fix required a discovery tool the allowlist never granted.
