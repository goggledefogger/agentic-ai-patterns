---
type: pattern
date: "2026-10-07"
source: A personal agent chained repo setup and a public create-and-push into one shell line, the setup failed on a wrong branch name, and the push ran anyway with the full private history
tags:
  - agents
  - git
  - publishing
  - safety
  - anti-pattern
---

# A Publish Step Gets Its Own Command Line

Anything that leaves the machine (a push, a repo create, a send) runs alone, on its own command line, after a separate step whose output you actually read. Never join setup and a send with `;`. And when a gate watches certain verbs, put each gated verb on a line by itself.

## The Problem

A personal agent was publishing a catalogue as a new public repo. The plan was sound: back up the history, start an orphan branch, commit the current tree once, then create the remote and push. It wrote that plan as one line:

```
git branch backup main && git checkout --orphan public && git commit -m "first" ; gh repo create NAME --public --source . --push
```

The branch was called `master`, not `main`. The first command failed, so the `&&` chain stopped right there. No orphan branch, no clean commit. Then the `;` did what a `;` does and ran the next command regardless. The create-and-push published the original branch, full history included, with the private raw files that the orphan commit was meant to leave behind. It was public for about a minute before anyone noticed.

Every part of the plan was right. The punctuation decided the outcome.

The same week showed the other half. A deterministic pre-tool gate denies force pushes. The agent wrote a line that mixed safe steps with one `git push -f`. The gate denied the whole line, as it should, because a gate that parses a shell string can only allow or deny it as a unit. The safe steps never ran either, and the agent had to work out which part of the line had been the problem.

## The Pattern

1. **Setup and verify are one step. Sending is another.** Do the local work, then run a check whose output you read: `git branch --show-current`, `git log --oneline | head`, `git ls-files | grep` for anything that should not be there. Only then type the push
2. **Never put `;` between setup and a send.** `&&` at least stops on failure. `;` turns "the setup failed" into "publish whatever is there". If a send has to be in a script, the script checks its preconditions and exits before the send
3. **A send runs alone.** One verb that leaves the machine per command. A success is then unambiguous, and a failure points at one thing
4. **A gated verb runs alone too.** If a hook or policy watches `push -f`, `rm -rf` or a send, give it its own line. A denial then names exactly what was denied, and nothing safe gets thrown out with it

## Why It Works

Shell chaining is written for the happy path. When something early goes wrong, the operators decide what still runs, and `;` decides "everything". An agent writing long one-liners to save a turn is optimizing the cheapest thing in the session at the risk of the most expensive one. Splitting the line costs one extra call. It buys a moment where the state is visible before anything irreversible happens.

## Watch-outs

- The branch name is the classic trap. `main` and `master` both still exist in the wild. Read it, do not assume it
- A verification step that was run but not read is not a verification step. The output has to come back to you before the send is written
- `--push` flags on create commands hide a send inside a setup verb. Treat them as sends

## Adjacent Patterns

- [[going-public-starts-from-an-orphan-commit]] is the full route this one-liner was trying to take, and what to do once history has leaked
- [[a-gate-can-block-but-cannot-speak]] is a gate that acts correctly and still changes nothing useful. A whole-line denial is a smaller version of the same thing
- [[verification-needs-a-negative-control]] covers making the pre-send check able to fail

## Source

A personal agent publishing a catalogue as a new public repo, 2026-10-07. A wrong branch name stopped the setup chain, a `;` let the create-and-push run anyway, and the private history was public for about a minute.
