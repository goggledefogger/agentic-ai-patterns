---
type: pattern
date: "2026-07-22"
source: The agent's own repo, two scripts, credential-precedence incident, 2026-07-22
tags:
  - testing
  - verification
  - security
  - anti-pattern
---

# Verification Needs a Negative Control

A test proving "credential X works" ran green, and the green was a lie. `git ls-remote` succeeded with a scoped PAT stuffed into `GIT_ASKPASS`, but the machine also had `credential.https://github.com.helper = !gh auth git-credential` configured, and git consults the credential helper before it ever falls back to `GIT_ASKPASS`. The `gh` CLI's active-account token answered every call. The PAT under test was never read. The same misconfiguration then produced a confusing false failure: on a cross-account repo, the helper handed over the wrong account's token, git reported "Repository not found," and the correct credential sitting right there in the env was never suspected, because nothing pointed at it.

## The Pattern

A test with no way to fail proves nothing. Trustworthy verification needs two legs, not one:

- **Control**: run the identical check with an empty or garbage credential. It must fail.
- **Subject**: run the identical check with the credential under test. It must pass.

Only the pair is evidence. A single green check on the subject leg is equally consistent with "it works" and "something else answered, and you never tested the thing you meant to." If the control leg passes too, you've found an ambient fallback, not a working feature.

The instance: `git -c credential.helper= -c credential.https://<host>.helper= ls-remote <url>` clears the helper list so `GIT_ASKPASS` is actually the thing being exercised. Without those two `-c` overrides, `credential.helper` answers first and the test measures the wrong credential.

This generalizes past git. Any auth path with an ambient fallback can satisfy a test using a credential other than the one you're verifying: cloud SDK default-credential chains, `ssh-agent`, kubeconfig contexts, `aws`/`gcloud` profile inheritance, browser cookie jars. Wherever a "try this, then fall back to ambient" resolution order exists, prove the ambient path is off before you trust a pass.

## Second-Order Failure: Self-Checks That Assert on Setup, Not Behavior

The two scripts with this bug both shipped self-tests. The self-tests asserted that the `env` dict handed to `subprocess` contained the token and the askpass path. Those assertions pass forever and can never catch this bug, because the thing that's wrong is that git ignores that env. A check that exercises only the setup you control, and never the dependency you're depending on, is a comfort blanket, not a test. Name it when you see it: correct-looking assertions on inputs, silent about whether the input was ever consumed.

## Third-Order Failure: A Control Leg Whose Negative Case Cannot Occur

The two failures above are missing controls. This one is worse, because the control is *present*, deliberate, and void — so it launders a false pass through the very ritual meant to prevent one.

The instance: a launcher's rows were renamed from a LAN address to a Tailscale hostname, on the belief they only worked at home. To prove it, the operator handed over two URLs to open from a phone on cellular — the new hostname, which must load, and **the old LAN IP, which must fail**. Textbook control/subject. Both loaded. The control could never have failed: the box advertised its subnet as a Tailscale route, so the private IP was reachable from anywhere on the tailnet. The control tested a proposition that was false before the test began.

Note what the shape got right anyway. The pair still produced the finding — not by the control failing as designed, but by it *refusing to*. An impossible negative case is loud when you actually run it and read the result honestly; it is silent only if you assume the pass. That is the argument for running the control leg even when you are sure of it.

Ask of every control leg, before trusting the pair:

- **What single fact makes this negative case fail?** Name it out loud. "The IP is only routable on the LAN" — is that a fact you measured, or the assumption under test wearing a disguise?
- **Does the control share the subject's assumption?** Here both legs assumed one reach path. A control that inherits the subject's blind spot is a copy of the subject, not a check on it.
- **Would the control have failed a week ago, or on another device?** Reach, DNS, and auth are environmental and change under you. A control that passes because of ambient configuration is the same defect this pattern opened with, one level up.

The tell is a control you wrote from a mental model rather than from an observation. If you can't say how you'd *watch* the negative case fail, you have written a second subject leg.

## Dev/Prod Inversion

This defect is invisible exactly where you'd debug it, and absent exactly where it ships. On a developer laptop with `gh` installed and authenticated, the credential helper hijacks the credential and the false pass looks clean. On a keyless cloud or headless host, there's no `gh`, no helper, so `GIT_ASKPASS` is genuinely exercised and the code works as designed. It "works in prod, lies in dev," the reverse of the usual works-on-my-machine direction, which is exactly why it survives review: local verification of the credential path is systematically meaningless on the machine you're most likely to run it on.

## Why It Works

- A control leg forces you to state, in code, what "not working" looks like. If you can't write a failing case, you don't yet know what the test is checking.
- Ambient fallbacks are invisible by construction. They're built to make things "just work," which is exactly why they need an explicit off-switch during verification (`-c credential.helper=`, an unset env var, a `--no-default-credentials` flag) rather than an implicit assumption that the subject path is the only path.
- Naming the "asserts on setup, not behavior" failure mode makes it greppable in review: does this self-test call the real dependency, or only inspect what we were about to hand it?

## When to Use

- Any time you're proving a specific credential, key, or config value is the thing actually taking effect, not just present.
- Writing a self-test for a function that shells out, calls an SDK, or hands data to something outside your process. Ask whether the assertion can be satisfied by the setup alone, without the dependency ever running.
- Debugging a "works here, fails there" report where "here" is your own dev machine. Check what ambient auth or config exists on your machine and not on the target.

## When NOT to Use

- Pure in-process unit tests with no ambient resolution order (no credential helper, no fallback chain) don't need a control leg. There's nothing else that could be answering.

## Adjacent Patterns

- `decorative-gate.md`, the sibling anti-pattern where a control that literally cannot refuse ships as if it were real. This pattern is the diagnostic: add a case that must fail, and a decorative gate can't pass it.
- `verify-adoption-against-installed-source.md`, the same control/subject shape in a different domain: grep a key you know exists (control) before trusting zero hits on the key you're checking (subject).
- `tests-pin-substance-not-identifiers.md`, another member of the "test that can't fail" family: a check keyed on the wrong signal rather than missing a control entirely.
- `reach-surface-resolution.md`, where the third-order instance above came from: a resource's reach is a surface, and a control leg that assumes a single reach path inherits exactly that blind spot.

## Source

The agent's own repo, `scripts/skill_fetch.py` and `scripts/clone_obsidian_claude.py`, 2026-07-22. Verifying a scoped GitHub PAT via `GIT_ASKPASS` initially returned a false pass because `credential.https://github.com.helper = !gh auth git-credential` answered first. The same misconfiguration produced a false failure on a cross-account repo. Fixed by clearing the helper (`git -c credential.helper= -c credential.https://<host>.helper= ls-remote`) and adding an empty-credential control leg that must fail.
