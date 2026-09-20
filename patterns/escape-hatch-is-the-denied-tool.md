---
type: pattern
date: "2026-08-05"
source: The personal agent session 2026-08-05 — a router agent had been unable to write one folder of a personal-private vault for over a week. Three prior sessions logged it as three different bugs (an idle worker, a worker that wrote nothing, an owner-gate on an unrelated folder) before a two-line controlled test showed the real shape: two correct gates whose intersection is empty.
tags:
  - agent-safety
  - gates
  - permissions
  - router-worker
  - diagnosis
---

# The escape hatch is the tool the caller was denied

Two safety gates, designed independently, each correct on its own terms:

- A **router** withholds a tool from its workers to contain exfiltration. No shell, so a worker holding sensitive bytes has no way to move them.
- A **resource** protects its most sensitive folder by denying the structured file tools, leaving exactly one way in: a human at a shell.

Both decisions are defensible, and each has a written rationale. Composed, they leave **no authorized path at all** for the router to write that folder. The one door the resource leaves open is precisely the one the router removed.

Nothing reports this. The router's logs say "Bash denied," which is true and intended. The resource's hooks say "restricted," which is also true and intended. Neither component can enumerate the other's rules, so neither can say *there is no route*. What surfaces instead is a worker that fails in an unremarkable way, which reads as flakiness.

## The incident

A thin-router agent staged content for a `Finance/` folder in a `personal-private` vault. That folder defaults to `sensitivity: restricted`; the vault's own hooks deny `Read`, `Write`, and `Edit` there, and its documented escape for restricted content is `cat` via Bash. The router's writable worker is defined with exactly `["Read", "Write", "Edit"]` and no Bash — the structural half of its exfil sandbox, with a git-only carve-out proposed and deliberately rejected five weeks earlier.

Every write into that folder failed for eight days. It was diagnosed three times:

- **as an idle worker** — the dispatch returned with no report, so "the write probably landed"
- **as a silent no-op** — a later run written up as "wrote zero files," falsifying the first
- **as a folder-specific owner-gate** — a refusal in a *different* restricted folder, generalized to the wrong rule

Each was locally plausible and each was wrong. Worse, a paper-trail note left for another session to apply sat unread for a week while later sessions cited it as done.

The test that ended it took one dispatch: **write to a non-restricted path in the same vault, with the same primitive and the same approval.** It landed. The identical write to the restricted folder did not. One variable, two outcomes, and the three theories collapsed.

A second detail matters for why it took so long: the informative error — the one naming both hooks — only appeared when the **human ran the dispatch by hand**. The agent's own attempts were being refused further upstream by a permission classifier, so the message that explained everything never reached any log the agent could read.

## The Pattern

1. **Before theorizing, run the one-variable test.** Same primitive, same approval, unprotected path. If it succeeds there and fails on the protected path, the problem is the protection, not the plumbing. This is cheaper than any amount of reasoning about the worker.

2. **Ask what the protected resource's own escape hatch is, and whether the caller can hold it.** Resources that gate sensitive content usually leave exactly one door. Read that policy directly. If the door is "a human at a shell" and the caller is an agent forbidden a shell, no agent route exists *by construction* — that is a fact about the architecture, not a bug to be retried.

3. **Route through the lawful surface; do not weaken either gate.** Write to the resource's unprotected queue directory and let something that legitimately holds the escape perform the last hop. Both boundaries stay intact and the content still arrives.

4. **Write the impossibility down where the next session will hit it.** An undiagnosed structural deadlock is indistinguishable from flakiness, and flakiness invites retries. Retrying a deadlock is how eight days pass.

5. **Fix one direction, then ask whether the other has the same shape.** The deadlock is a property of the two tool lists, not of the verb. If reads and writes are both denied and both escape through the same door, one shipped fix leaves the other half of the class open — and looking handled.

## The second arrow

Six days after the write fix shipped, the same router hit the same folder in the **read** direction and spent three dispatches rediscovering the identical deadlock. Nothing had regressed. The staging-plus-promotion script was working exactly as designed, and its existence is what made the gap invisible: a directory containing a script named for this problem reads as *this problem is solved*, so nobody asks which arrow it solved.

The read half needs one thing the write half does not. A promotion moves bytes the router already composed, so a plain `cat` in the last hop leaks nothing new. A read moves bytes **toward** the router, so the same shape would satisfy the resource's gate and break the router's own hold-no-contents rule in the same motion. The lawful read surface therefore ends in a summary, not a file: the human-run script reads the protected file, hands the text to a worker with an **empty tool list**, and returns only the answer. Containment by tool list, not by instruction.

Watch for the asymmetry whenever the two directions have different blast radii. "Can the caller hold the escape?" is the first question; "does holding it violate something on the caller's side?" is the second, and only the read direction usually has to answer it.

## Why it stays invisible

- **Both sides' logs are locally correct.** Neither is an error message; both are policies working.
- **No component sees both halves.** The router cannot read the resource's hooks; the resource cannot read the router's tool list. The empty intersection exists only in the union, which nothing computes.
- **Unprotected paths keep succeeding**, so the route looks healthy in every test that does not happen to touch the restricted subset.
- **It is 100% reproducible but presents as intermittent**, because it reproduces only on the protected paths — which are the minority of writes and the ones least likely to be retried casually.

## Watch-outs

- **"The worker is flaky" is the tell.** Structural impossibility masquerades as nondeterminism. Treat a repeated unexplained failure on a *specific class of path* as evidence of a rule, not of noise.
- **A shipped fix for one direction reads as coverage for the class.** The tell is a later session re-running the diagnosis from scratch beside a script written for the same deadlock. Name the arrow in the script's own docstring so the gap is legible from the filename outward.
- **A handoff note is a request, not a write.** Leaving a note for another session to apply is a queue with no acknowledgement. Probe the target's mtime before believing it landed; ours was cited as done for a week while unapplied.
- **The manual last hop will generate pressure to grant a standing exception.** That pressure is the erosion vector described in `approval-scope-invisible-to-gates` and `decorative-gate`. The correct resolution is a *deterministic mechanism with a human-scoped trigger* — a small script that does the mechanical work, run by the human per batch — not a standing allow rule that authorizes every future crossing.
- **Do not re-roll a failed dispatch unchanged.** If the shape was refused once, the second identical attempt spends a worker to learn nothing.

## Sighting: the fixture found it in one turn (2026-08-31)

A minimal repro built for a different bug produced this one unprompted. A PreToolUse hook returned `ask` for every `Write`; the harness never answered, so the Write failed. The model's next move, same turn, no prompting: `printf 'hi\n' > note.md` — the same file, written through Bash, which auto mode allowed without a question. One gate on the structured tool, one open shell beside it, and the denied write completed by the route the gate never covered. The gate's own target list was the map to the hatch.

## When NOT to use

If a caller genuinely *should* be able to write the protected resource unattended, this pattern is the wrong frame — the answer there is a deliberate, narrowly-scoped authorization for that path, not a staging queue. The pattern applies when the protection is correct and the caller's containment is also correct, and the goal is to get work done without dissolving either.

## Adjacent Patterns

- `router-worker-exfil-containment` — why the shell is withheld in the first place
- `sensitivity-tiered-access-control` — the folder-level protection on the other side, whose own history records losing a boundary to convenience
- `lawful-write-surface` — give the agent somewhere legal to put the proposal
- `filesystem-queue` — the staging directory as producer/consumer handoff
- `subagent-cannot-consent` — the same failure signature (a correct gate silently killing a worker) from a different cause
- `approval-scope-invisible-to-gates`, `decorative-gate` — why the fix is not a standing grant
- `opaque-write-needs-a-read-back` — why the probe, not the worker's word, is the evidence
- `eyes-free-approval-channel` — where the human-scoped trigger belongs once "a human types the command" is itself the friction being routed around

## Source

The personal agent session 2026-08-05. Eight days, three wrong diagnoses, one controlled test. The resolution shipped as a staging directory plus a deterministic promotion script that a human triggers per batch — deliberately not allowlisted, with the reason recorded in the script itself so a later pass does not "fix" the tedium.

Read direction added 2026-08-11, same vault, same folder, same worker. Three dispatches, the third returning the hook's denial text verbatim: it names `cat` via Bash as the authorized escape, which is the one tool the worker does not have. Resolution mirrors the write side (`read_restricted.py`, human-run, not allowlisted) with the empty-tool-list worker standing in for the last hop, so the router gets an answer and never the file.

## Addendum (2026-08-18): the human-scoped trigger does not have to be a keystroke

The staging-plus-promotion answer above ended with the human typing the command,
and that was load-bearing for exactly one reason: it was the only per-batch
authorization anyone could point at. Two weeks later it collided with a
standing directive on the same system — *never hand the human a command to
paste; they say it plainly and it lands* — and the two were true at once, so a
session handed over a `promote_staged_block.py` line for a finance entry and
the human named the friction: "I still don't want to have to paste command line
commands." The paste was the friction, and friction on a lawful surface is the
erosion vector this pattern warns about — someone will eventually "fix" it with
a standing grant.

**The reconciliation is a change of trigger, not of gate.** Keep the
deterministic mechanism byte-identical. Move the trigger from a typed command
to a *click on a card served by a separate, closed-form channel*: a loopback-
only daemon that renders each pending item with its dry run and the composed
text, exposes exactly Approve and Deny, checks the (verb, resource) pair
against a change-gated table at click time, claims the item atomically before
running so a second click is a refusal, and logs every render, decision and
refusal. The router files items into that queue and never touches the surface
itself. This is `eyes-free-approval-channel` applied to the agent's own staged
work rather than to a harness permission dialog: the approval channel is
structurally not the content channel, the real options travel, the endpoint
validates against what is pending, and a "yes" typed or spoken into the chat
resolves nothing.

Three things worth stating plainly, because each is a place a later pass could
slip:

- **The click is not a stronger lock than the keystroke; it is the same lock
  reachable without a paste.** No local-process boundary makes the router
  physically unable to POST to a daemon on its own machine, just as none made it
  unable to run the script. What the design buys is a trigger outside the
  content channel and a log line under every press. Say that in the rule; do
  not dress a `deny` on the daemon's URL as a boundary (`decorative-gate`).
- **The read arrow rides the same click.** A read still ends in a worker
  summary, never file contents — the click authorizes the *run*, not a `cat` to
  the router. The answer lands where the router already reads its own mail.
- **Reach is a property of the bind, not a rule about behavior.** Loopback plus
  a Host check that accepts only this machine's own tailnet name means the
  human's own devices can press the card and nothing else on the internet can.
  The earlier "cannot review a diff by thumb" rule was about diffs; these cards
  are agent-composed, append-only, dry-run shown, so the human chose to allow the
  phone with the desk *encouraged* for anything heavy — a preference the voice
  layer states, not a lock the daemon enforces.
- **The link is a control, not a rule.** The first live session said the link;
  the human asked how he would have known if it hadn't. So a Stop hook refuses
  to end a turn that filed an item without saying the link in reply text —
  built to `a-gate-can-block-but-cannot-speak`: structured block + user-facing
  message, a log line per fire, and verified by ending a turn in the failing
  state on purpose and reading what came back.

Source: The personal agent, 2026-08-18 — `scripts/stage_approval.py`, `run_daemon.py`
`/approvals`, `registry/run-allowlist.md` "Approval verbs", the agent's ops playbook rule 84
(rewritten) and its rationale entry.
