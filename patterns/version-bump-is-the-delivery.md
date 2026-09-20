---
type: pattern
date: "2026-07-22"
source: A shared tooling stack's feature wave — 6 PRs merged, then the realization that no installed copy would ever see them
tags:
  - distribution
  - upgrades
  - plugins
  - first-run
---

# Version Bump Is the Delivery: Merged ≠ Shipped Under a Version-Pinned Cache

A feature wave merged six PRs into a tool distributed as a Claude Code plugin. Every eval passed, main was green — and not one existing install would ever see the new tools. Plugin installs are cached per version (`~/.claude/plugins/cache/<marketplace>/<plugin>/<version>/`), so an installed copy keeps serving exactly the tool set it was installed with until the manifest version changes *and* the user runs the update. The `plugin.json` still said `0.2.2`. Merging was not shipping; the version bump was the shipping.

The failure is invisible from the repo side: nothing is broken, nothing errors, CI is green. It surfaces on the *user's* side as a tool that "doesn't have" a feature the README describes — which reads as a bug in the tool, not as a stale copy.

## The Pattern

Treat the distribution cache as part of the release, with three legs:

1. **The bump rides the wave.** Any PR set that changes user-visible surface includes the manifest version bump (its own PR or the last one in). An eval can hold the line: assert the version is real semver, and assert any prose-stated capability counts ("all N tools") against the code's actual registry — parallel branches each adding a tool is exactly how the prose number silently rots.
2. **The update path is documented as two commands**, not a paragraph. If the platform has `marketplace update` + `plugin update`, the README says those two lines and "then restart."
3. **The doctor names which copy is running.** A first-run preflight script should also be an upgrade preflight: compare the installed cache version against the checkout, and print the exact update command when they differ. The stale-install symptom ("some tools present, one missing") goes in the agent-facing skill too, diagnosed as *stale, not broken* — otherwise the agent debugs a healthy server.

The general form beyond plugins: any version-pinned distribution channel (pinned Docker tags, vendored copies, lockfiles, app-store builds) has a "merged but undelivered" gap, and the fix is always the same three legs — bump rides the change, update path is two commands, something on the user's side can name the gap.

Sibling pattern: `verify-adoption-against-installed-source.md` is the consumer-side mirror (grep the pinned artifact before adopting config keys). This is the producer side: make sure what you shipped can actually arrive.
