---
type: pattern
date: "2026-10-10"
source: The personal agent 2026-10-10 session, a safety gate on its own hook directory denying unrelated paths
tags:
  - gates
  - safety
  - subagents
  - anti-pattern
---

# A Path-Pattern Gate Fires Outside Its Repo: Scope a Gate to What It Guards

A safety gate protects an agent's own hook directory. It matched the segment anywhere in a command or path. It denied a plugin's hook config in a scratch clone under the temp directory. It denied a docs URL that ended in the word. It denied a settings JSON key. Subagents inherited the denial.

None of these touched the guarded repo. The gate guarded a word, not a place.

## The Pattern

1. **When a gate blocks, do everything else.** The agent finished all other work and handed the human the 1 command to create the file at the desk
2. **Do not rename a path to dodge a gate.** Evading a lock teaches the system nothing and weakens the lock
3. **The durable fix is scope.** Match the guarded segment only under the repo it guards, not as a substring of any string
4. **Make the trusted desk the exception path.** A session with desk trust set in its environment passes. Sessions started elsewhere lack it, so they hit the gate, and that is by design

## Measured

On 2026-10-10 the denial fired for 3 unrelated cases. Subagents saw it too.

## When It Shows Up

Any gate that matches on a name. The tell is a denial whose path is nowhere near the thing the gate protects.

## Adjacent Patterns

- `policy-on-every-door.md` for the mirror failure, a rule that skips doors it should cover
