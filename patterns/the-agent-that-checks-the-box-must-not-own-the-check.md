---
type: pattern
date: "2026-09-29"
source: A personal agent's own repository, where the rules that gate the agent's self-edits are files the agent can edit, measured 2026-07-25 and again 2026-08-06
tags:
  - safety
  - verification
  - self-modification
  - long-running-agents
  - anti-pattern
---

# The Agent That Checks the Box Must Not Own the Check

A personal agent edits its own repository: its instructions, its rules, its hooks. Three of the files it edited on 2026-07-25 were the hook that gates safety-critical edits, that hook's tests, and the instructions file that ranks the gates. A pre-tool hook returned "ask" on every one of those edits, correctly, verified afterwards by piping the exact payloads into it by hand. The session's permission mode auto-accepted the ask. Nothing paused. The edits went through.

What stopped the change was a pre-commit hook: blocked, safety-critical change, with the staged paths listed, escapable only by an environment variable the agent is not permitted to set. The ask that nobody sees is a log line. The commit hook is the one the agent cannot answer on its own behalf.

The same shape shows up in a rulebook. When the agent's operating rules were prose the agent itself maintained, the count went from 41 to 67 in one month, every incident becoming a rule. The ceiling that held was a pre-commit check and a CI job counting words and rules, so the bar is measured at commit time rather than argued in the file.

## The Pattern

Separate two write permissions that look like one:

- **The box.** The agent marks the item done, checks the box, writes the progress line, sets the status. It owns this and should.
- **The check.** The definition of done, the failing test, the lint row, the gate, the ceiling. The agent that is trying to close the item does not delete, weaken or rewrite any of these to get there.

When it cannot pass honestly, the outcome is "stuck", one line on what was tried, and a stop. Assume the agent will want the list green, because at the end of a long session it will, and make sure the only way to green is the real one.

Then put the check where the agent's own approval does not reach:

1. A pre-tool "ask" is only a control in a mode that shows asks. In any mode that auto-accepts, it is a log line. Verify which mode you are in by exercising it, not by reading the hook.
2. A commit hook, a protected branch, a required review on the test folder, a CI job: these refuse regardless of what the agent decided. Use one of them for anything the agent could otherwise talk itself past.
3. The escape hatch is not the agent's. On 2026-08-06, with the person's explicit approval in hand, the prefixed commit was still refused by a harness-level classifier that reads the command string and never the transcript. The right move was to split the commit, land the ungated files, and hand the person the one command for the rest.

## Why It Works

A checklist the agent can edit is not evidence of anything. The value of a red test is that it stays red until the code changes, and the agent closing the item is the one actor with a motive to change the test instead. Moving the check one layer below the agent's approval turns "I decided this passes" into "this passed", which is the only sentence a person can act on without re-reading the diff.

## When to Use

Any agent that maintains its own rules, its own tests, or its own tracker across sessions. The tell is a green list with nobody able to say which check would have gone red, or a rule count that only ever goes up.

Near neighbours: [`unenforced-red-becomes-the-baseline.md`](unenforced-red-becomes-the-baseline.md) is a check that refused correctly and was ignored by people. [`a-second-writer-satisfies-your-gate.md`](a-second-writer-satisfies-your-gate.md) is a gate satisfied by evidence a different actor wrote. This one is the check's own author being the actor it checks. [`green-tests-can-mirror-the-same-guess.md`](green-tests-can-mirror-the-same-guess.md) is a test that was a copy of the judgment it certified, which is what happens one step before the agent edits it.

## Source

A personal agent's repository: the change-gate hook, its pre-commit twin, and the rulebook ceiling. The outside corroboration is the published practice for long-running coding agents where the agent may mark a feature passing but is not permitted to delete a test that says it does not, cited in a 2026-09 talk on agent setups.
