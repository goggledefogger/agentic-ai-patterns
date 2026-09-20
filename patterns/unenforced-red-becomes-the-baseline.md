---
type: pattern
date: "2026-08-04"
source: A personal skills repo — PR #47 merged with validate failing (2026-08-02), main red until #51/#52 (2026-08-04)
tags:
  - ci
  - workflow
  - safety
  - anti-pattern
---

# An Unenforced Red Becomes the Baseline

A repo's `validate` job caught exactly what it was written to catch: a new skill directory with no registry line. It printed the failure, named the directory, and exited 1. The PR was merged anyway.

`main` went red on 2026-08-02 and stayed red for two days. The two PRs that followed each showed a red `validate` — neither had caused it, both inherited it, and both were merged. The check was healthy the entire time. Nothing was **bound** to its refusal.

## Not the same as its three siblings

This one is easy to file under an existing anti-pattern and it belongs under none of them:

- [[decorative-gate]] — a control that *cannot* refuse. This one refused, correctly, every single run.
- [[half-gate-whole-verdict]] — evidence for one axis, a verdict naming two. This verdict was exactly as broad as its evidence.
- [[unrun-checks-read-as-passing]] — a check that never executed, whose silence read as health. This one executed and *complained*, on every push to main and on every PR.

The check was sound. The **binding** was missing. A verdict with no consequence attached is a log line, and the distance between a log line and a control is the thing nobody audits, because the check itself looks fine in isolation — and it *is* fine.

## The second failure, which costs more than the first

The direct cost of the first merge was one unregistered directory: trivial, a five-line fix.

The real cost is what a persistent red does to everything downstream. Once `main` is red, no later run carries information. Every subsequent PR arrives already-red, so the only question a reviewer can ask is *"is this the same red or a new one?"* — which requires reading logs, which nobody does, which means the honest reading of a red check becomes **no reading at all**. A single unenforced refusal converts the check from a signal into a background condition, and it stays converted until someone does the archaeology.

So the damage is not proportional to the bug that went in. It is proportional to how many checks ran afterward and told you nothing.

## The Pattern

**1. For every check, name what is bound to its refusal.** Not what it detects — what *stops* when it says no. The answers are a short list, and they rank:

| Binding | Strength | Sees |
|---|---|---|
| required status check (server-side) | boundary | every merge path, including the web UI |
| pre-push hook (client-side) | speed bump | terminal pushes; `--no-verify` walks past it |
| a rule in CONTRIBUTING.md | memory | whoever read it recently |
| nothing | none | — |

A check whose row is "nothing" is not a control yet, however good its logic. Write the row down; the exercise is the point, because "we have CI for that" is what gets said in place of naming the row.

**2. Say which rung you actually bought.** When the boundary is unavailable — required checks need GitHub Pro on a private repo, and making the repo public was not an option here — install the speed bump and *call it a speed bump*. The failure mode is not the weak control; it is the weak control described as a strong one, after which nobody revisits it. Same honesty this library already demands of a gate's evidence.

**3. Treat a red default branch as an incident, not a backlog item.** It is the one bug whose cost compounds per subsequent CI run. Fix it before merging the next thing, even when the next thing is unrelated and the red is not yours — *especially* then, because "not mine" is the sentence that keeps it red.

**4. The tell.** Someone says *"that failure was already there"* to justify a merge. That sentence is diagnostic: it means the check has already stopped being a control and become weather. It is also, usefully, said out loud — so it is catchable in review long before an audit would find it.

## Verification

Exercising the check is not enough; that only proves it can refuse, which [[decorative-gate]] already demands. Exercise the **binding**: perform the real action the control is supposed to stop and confirm it does not complete.

For the pre-push hook that came out of this, that meant pushing a genuinely violating commit to the real remote and then asserting on the remote's state, not on the hook's output:

```
push exit=1        refs on origin: 0
```

The first number says the hook ran. The second says the push did not happen. Only the second is the control. An earlier run of that same test passed the hook's own output check while the branch reached origin anyway — the test had stashed the untracked hook it was testing, so nothing was installed and nothing blocked. Asserting on the guarded resource rather than the guard's log is what caught it.

## Adjacent Patterns

- [[decorative-gate]] — a control that cannot refuse. Necessary condition; this pattern is the one after it.
- [[unrun-checks-read-as-passing]] — the mirror image: there, absence of complaint read as health; here, presence of complaint produced no effect. Both are verdicts carrying no information.
- [[pre-push-pr-discipline-hook]] — the usual answer when the server-side boundary is unavailable, with the caveats that make it survive contact.
- [[guard-evidence-outlives-the-failure]] — what a refusal should leave behind so the next reader can tell a new red from an old one.
