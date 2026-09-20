---
type: pattern
date: "2026-09-09"
source: The personal agent self-improve session, 2026-09-09 — a week of permission-prompt fatigue turned out to be entirely the harness classifier's; the agent's own carefully built "ask" gates had never once reached the human
tags:
  - agents
  - permissions
  - safety
  - anti-pattern
  - claude-code
  - hooks
---

# An Ask No Mode Shows Is Not a Gate

A hook, plugin, or middleware that returns "ask the human" is only a control in
the harness modes that render an ask as a prompt. In every other mode the same
verdict is answered by something else — a classifier, a default, a skip — and
the human never sees it. The gate's author believes a person is in the loop.
The runtime has quietly substituted a different decider. Nothing errors, every
test passes (the hook does emit `ask`), and the safety story is fiction in
exactly the mode people actually run.

## The failure that named it

The personal agent's three "stop and ask the author" gates — a change-gate on his own safety
files, a per-write ask on a private vault at the desk, an "is this yours to
send?" ask on shared-IP egress — had 296 `ask` verdicts in the audit log for
one week. The author had answered zero of them. He ran Claude Code in auto mode, and
the docs are explicit: in auto mode a PreToolUse hook's `permissionDecision:
"ask"` goes to the classifier, not the user; in bypass mode it is skipped. The
roadmap had even recorded one instance two days earlier ("returned ask, no
prompt appeared, the edit landed") and parked it.

Meanwhile the author was answering ~700 prompts a week and saying yes to all of
them — every one raised by the classifier on plain shell, not one by the personal agent
gate. The fatigue was real and the gates were irrelevant to it. Two wrong
beliefs held each other up: "the prompts are our gates working" and "our gates
are protecting the sensitive paths."

## The Pattern

**Sort every verdict your gate can return by what binds it in every mode the
system runs in, and keep only verdicts that bind.**

1. **Enumerate the modes** the gate will actually run under (interactive
   default, classifier/auto, bypass, headless `-p`, a foreign harness that
   runs no hooks at all). Read the harness docs for what each mode does with
   each verdict; do not infer from the interactive one.
2. **An `ask` binds only where a human sees prompts.** Everywhere else it is
   `allow` with extra steps. If a stop must be the human's, make it `deny` and
   put the yes on a channel that exists in every mode — an approval card, a
   staged file, a PR to merge, a `!` one-liner the human types. Name that route
   in the denial (`a-gate-can-block-but-cannot-speak`).
3. **Where the human is present by construction**, say so with a discriminator
   the unattended path cannot inherit (an interactive-only env var, a shell
   alias) and let the verdict be `allow` — a real lock lower down (a pre-commit
   hook, a server-side check) is the boundary, not the prompt.
4. **Measure the prompt stream before touching the gates.** Count which
   prompts the human answers and which layer raised each. If the answer is
   "none of ours," the fix is not more gates.

## Why It Works

- **The verdict vocabulary is the harness's, not yours.** `ask` means what the
  current mode says it means; a deny means deny everywhere. Building on the
  invariant verdict removes a whole class of silent mode-dependence
- **Reflexive yes is not a control.** The harness vendor's own study: humans
  caught 13.6% of dangerous commands when prompted, the classifier 89%. A gate
  whose only enforcement is a tired click was never protecting anything, so
  converting it to deny-with-route or allow-at-desk loses nothing measurable
- **Fewer prompts make the remaining ones legible.** When every prompt is the
  classifier's, the human can learn what it flags; when they are a mix of
  layers he cannot tell apart, he learns to click yes

## When NOT to apply

A gate that runs only under a single, known interactive mode (a CI approval
step, a chatops bot with one UI) can keep `ask` — but write down which mode it
assumes, so the day the harness grows a classifier the assumption is findable.

Related: `decorative-gate` (this is its mode-dependent cousin: the check runs
and even refuses in tests, but the runtime answers it), `eyes-free-approval-channel`
(the same lesson from the other side: the prompt existed but the human was not
where it rendered), `an-unattended-path-must-not-share-fate-with-an-interactive-one`
(the discriminator that makes "present by construction" true).

## Also seen: 2026-09-11, the two halves of the discriminator split

The personal agent built point 3 as two halves: an env var (`ROY_TRUST=desk`, set by `.envrc` in interactive shells) that opens the gates, and a shell alias that launches the session in bypass mode. They are not the same discriminator. Child processes inherit the env var but not the alias, so a session spawned by an IDE extension or another process carried desk trust under auto mode (measured 2026-09-09). A session started from Antigravity's integrated terminal did load the alias and ran in bypass (measured 2026-09-11), so "it's the IDE" does not predict the mode. Only the `claude` process's own launch args do. The second cost is easy to miss: bypass drops the harness classifier, the one layer that reads intent instead of a path string. With `allow` at the verdict and no classifier, a Bash command that builds a protected path in code (`python3 -c "open(...)"`) meets nothing at all. Apply point 3 knowing which layer it removes. Only an OS read sandbox closes that gap, and it was blocked on the Mac.
