---
type: pattern
date: "2026-06-13"
source: A portable vault-setup script and the decision record written beside it, on where the baseline config should live
tags:
  - obsidian
  - tooling
  - scaffolding
  - distribution
---

# Portable Tool Baseline

You want a public repo to set up a good working environment for anyone who uses it, plugins, config, and tools, without redistributing large third-party binaries you do not own.

## The Pattern

- Keep the declarative config in the repo, version-controlled. Settings, hotkeys, small JSON
- Vendor only your own small tools into the repo. A plugin you wrote, a helper script
- Copy the large third-party binaries at setup time from a reference install on the user's machine
- If the reference install lacks a piece, report what to install by hand rather than failing

## Why It Works

- The repo stays light and shareable, with no licensing problem from redistributing other people's binaries
- Users get their real configured tools by copy, settings included
- Readers without a reference install still get the config and the parts you own, plus a clear manual list for the rest
- The split tracks ownership. Yours travels with the repo, theirs comes from where it already lives

## When to Use

- Shipping an environment baseline from a public repo. Obsidian vault setup, editor config, dotfiles
- When some pieces are yours and small, and others are large and third-party
- Skip it when every dependency has a clean scriptable release, then just download them

## Source

- `skills/new-vault/setup-obsidian.sh` copies the baseline config plus the vendored `hotkey-passthrough` plugin, then pulls `terminal` and `obsidian-git` from a reference vault
- `docs/decisions/2026-06-13-obsidian-baseline-source-split.md` records why the custom plugin is vendored and the third-party binaries are not
