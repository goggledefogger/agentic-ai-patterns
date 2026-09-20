---
type: pattern
date: "2026-06-30"
source: The household-agent repo (a household agent on a Raspberry Pi), Hermes 2026.6.x upgrade deploy-gate wedge
tags:
  - upgrades
  - testing
  - ci
  - patches
---

# Soak-Aware Patch Tests

If you carry local patches over a fast-moving upstream and you guard them with a test that asserts "the patch is still applied," that test goes red the moment a sanctioned upgrade wipes the patch. If the test sits in your deploy gate, it wedges every deploy, not just the patched component.

## The Pattern

You patch an upstream you do not control (source edits, monkeypatches, vendored tweaks, config overrides). A regression test asserts the patch is present so an `update` cannot silently revert it. Good instinct. But the test cannot tell two states apart:

- **Cleanly drifted, decision deferred** — you just ran a sanctioned upgrade, it reinstalled the tree, the patch is gone, and the re-apply-or-delete call is intentionally parked until later evidence comes in. Expected. Acceptable for now
- **Partially applied or corrupt** — the patch is half-there, an anchor matched the wrong region, someone reverted one of four sites. A real regression

Both read as "assertion failed." So a normal upgrade trips the same alarm as genuine breakage, and a gate that hard-aborts on any red refuses to ship anything until someone untangles it.

Concrete instance: a Hermes upgrade reinstalled the source tree and wiped 7 truncation-UX patches. The regression test asserting them applied went red. That test was one file in a pre-deploy unit-test gate, and the gate hard-aborts on any failure. `--force` did not bypass it. No deploy succeeded for two days, and the first deploy to hit the wall was unrelated to the patches. The patch drift was expected and even documented, the wedge was not.

## Why It Works

The fix is to teach the patch test the difference:

- When the patch checker reports the patch **cleanly** absent (all anchors gone, zero sites applied) AND an auditable deferral marker says the drift is sanctioned, **skip** with a reason that names the pending decision
- Still **fail** on a partial or mixed result, that is the corruption the test exists to catch. Treat any lone surviving site as suspect, a dumb substring check can match a collision in rewritten upstream code
- Keep the offline checks (does the patch script itself still carry the intended wording) running unconditionally, they validate your source regardless of what is deployed

The deferral marker is the load-bearing piece. "Skip whenever the patch is missing" would mask a future non-upgrade revert. A keyed marker (patch id, reason, `deferred_until`, `as_of`) makes the skip auditable and forces its own removal when the decision lands, so it cannot rot into a permanent silent hole.

## When to Use

Any repo that carries local patches against an upstream you update on a cadence, with a test gate in front of deploys. The day you write a "patch stays applied" assertion is the day to decide what it does when a sanctioned upgrade legitimately removes the patch. The answer is skip-with-marker, not delete-the-test, since you may still re-derive the patch once the evidence is in.

## Adjacent Patterns

- `decision-doc-adr.md` is where the deferral reason lives if the marker needs more than a line. The marker is a tiny machine-readable ADR
- `unattended-run-discipline.md` says fail loud and stop. The refinement here is that "loud" for an expected upgrade must mean "skip with a reason," not "wedge every deploy"

## Source

The household-agent repo `test-hermes-truncation-ux.py`, which asserted the Story 3-20 patches applied against the live install and hard-blocked the deploy gate after the 2026-06-28 Hermes upgrade wiped them. Planned fix: the Pi-applied assertions skip when the patch checker reports a clean drift backed by a deferral marker, and still fail on partial application.
