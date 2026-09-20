---
type: pattern
date: "2026-08-07"
source: The personal agent session 2026-08-07 — gating an Antigravity IDE pane, second act
tags:
  - security
  - verification
  - anti-pattern
  - agent-safety
---

# The native deny list beats the ported gate

When enforcement has to reach a foreign platform, the platform's own declarative controls are the first rung, not the last resort. A ported adapter is the fallback, and without an in-situ probe, a wired adapter and a dormant one look identical.

## The incident

A tier gate needed to enforce inside a third-party IDE. The obvious build, a translator adapter registered in the IDE's hook config, invoking the existing authoritative gate, was designed, adversarially reviewed, tested green against docs-derived fixtures, and committed. The first live probe sailed straight through: the IDE never invoked the hook at all, hooks turned out to be a CLI-only feature there, and every working example in the wild was CLI. The fix that actually enforced was nine rows of the IDE's own native deny list, added by hand in its settings UI in two minutes, "Permission denied. Matches user-configured deny rule." on the very next probe. The native layer had existed the whole time, nobody had checked for it before building.

A follow-up probe asked the pane to try the shell instead. It read the same protected file with `head`, then `grep`, zero friction, the native file rules do not see the terminal, and the IDE's auto-execute setting was on. The human chose to keep auto-execute for flow and accept the harness as untrusted for private tiers. That is a legitimate outcome, the honest posture ("this harness stays open-tier") beat a half-closed hole that would have been cited as protection.

## The Pattern

1. **Before porting enforcement, inventory the target's native declarative controls**, permission lists, sandbox modes, path rules. Native wins on every axis: no subprocess, no schema drift, no dependence on which folder is open.
2. **Keep the ported adapter only as fallback** for platforms with no native layer, and label it VERIFIED or DORMANT per platform, never "installed".
3. **The probe is the only classifier.** Run the forbidden action in the real platform and watch what happens. A probe must also test each tool class separately, file tools and shell are different doors, and rules that close one say nothing about the other.
4. **When a hole stays open by choice, write the choice down and keep the ceiling.** A documented open door is safer than an undocumented half-closed one.

## Why it stays invisible

- **Every proxy signal said protected.** The build was reviewed, tested, and committed.
- **Platform docs blur product lines.** IDE features and CLI features sit under one name, so the assumption that hooks run everywhere read as fact. Only the live deny message, or its absence, told the truth.

## Watch-outs

- **The pane, denied by the file rules, immediately and helpfully suggested three bypasses**, including the shell one that worked. An agent facing a deny will enumerate the other doors, closing one tool class can actively route traffic to the open ones.

## When NOT to use

If the target platform genuinely has no native declarative control, no permission list, no sandbox mode, no path rule, a ported adapter is the only option and this doesn't call it wrong. The failure mode is specific to skipping the native-control inventory and probe before building the port, not to building a port at all.

## Adjacent Patterns

- `green-tests-can-mirror-the-same-guess` — same build, first act
- `a-capability-contract-must-name-the-verb` — same harness, the week's arc
- `escape-hatch-is-the-denied-tool` — a denied tool class routing traffic to the one left open

## Source

The personal agent session 2026-08-07, gating an Antigravity IDE pane, second act. A hook-based adapter designed, reviewed, and tested green never fired in the live IDE because hooks were CLI-only there; the IDE's own native deny list, added by hand, enforced on the first real probe. A second probe found the shell untouched by the file rules and the human chose to leave it open rather than half-close it.
