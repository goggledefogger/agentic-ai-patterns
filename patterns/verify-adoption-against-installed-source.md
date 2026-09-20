---
type: pattern
date: "2026-06-30"
source: The household-agent repo (a household agent on a Raspberry Pi), Hermes 2026.6.x adoption-key audit
tags:
  - upgrades
  - verification
  - configuration
  - anti-hallucination
---

# Verify Adoptions Against the Installed Source

When an upgrade plan says "adopt feature X, set config key Y," grep the actually-installed, pinned artifact for Y before you adopt. A plan researched against upstream's main branch or its docs can name keys that do not exist in the version you pinned. Writing those keys is a silent no-op.

## The Pattern

The setup is routine. You research an upgrade against the latest upstream, write a plan that lists the new config keys and feature flags to flip, then pin an older tag for safety because you do not want bleeding-edge. The plan and the pin now disagree, and the gap between them is exactly the features the plan is excited about, because the newest work is what the research found.

You apply the keys. The config loads. Nothing errors. The behavior you wanted never turns on, because the build you pinned never reads those keys. A soak "passes" while doing nothing different.

Concrete instance: a Hermes upgrade proposal listed `contextInjection: continuation-skip`, `compaction.midTurnPrecheck`, and `localModelLean` as "primary-source verified." A grep of the pinned v0.17.0 tree found zero hits for all three across every file type. A positive control (`protect_last_n`, a key known to exist) was found correctly, so the grep itself worked. The three keys live in upstream main, hundreds of commits past the pin. Applying them would have been three no-ops inside a soak that reported success.

## Why It Works

The installed source is the primary source, above the proposal, above the published docs, above the changelog. All three describe a moving target. The install is the one thing that determines behavior.

- Grep the pinned tree for each key, including snake_case and camelCase variants, across all file types not just code
- Run a positive control. Grep a key you know exists, to prove the grep would have found a hit. Zero hits only means something if a control returns a hit
- A key with zero hits is not read by that build. Reconcile before adopting: find its real name, confirm it shipped in your version, or accept it only exists in a newer commit than your pin, which reopens the version decision

This is cheap. It is one grep per key against a tree you already have on disk, and it converts "the plan says this works" into "the build I am running reads this."

## When to Use

Every upgrade where the plan lists config keys or feature flags to flip, especially when you deliberately pin behind HEAD. Do it before the adoption step, not after a soak quietly proves nothing. The same check applies to any "set this flag to get behavior X" instruction sourced from docs or a blog rather than from the code you installed.

## Adjacent Patterns

- `system-understanding-protocol.md` is the general discipline, atomic citations and a source pass over the aggregate docs. This is the sharp instance for config adoption: cite the installed tree, not the proposal
- `grill-the-plan.md` should ask this question during the adversarial pass: does each named key exist in the version we are actually shipping

## Source

The household-agent repo Hermes upgrade, where the Story 7-16 section 3.1 adoption keys were absent from the pinned v0.17.0 install. Found by grepping `~/.hermes/hermes-agent` with a `protect_last_n` positive control on 2026-06-30, which flipped the adoption step from ready to blocked pending per-key reconciliation.
