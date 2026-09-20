---
type: pattern
date: "2026-07-30"
source: The personal agent session (2026-07-30), registering a new vault whose people/ folder was lowercase where every prior vault used People/ — the router's cross-vault person lookup had the convention hardcoded and macOS hid the mismatch
tags:
  - orchestration
  - multi-vault
  - conventions
  - portability
---

# Shared Tooling Matches Per-Vault Conventions Leniently

Tooling that spans N independent vaults must treat each vault's structural conventions (folder names, casing, layout) as observations to match, not a contract to assume. Each vault is its own repo with its own author and its own `CLAUDE.md`; nothing enforces that vault seven spells `People/` the way vaults one through six did.

## The Problem

A cross-vault lookup (find a person, find an inbox, find meeting notes) hardcodes the convention the first vaults happened to share: `Path(vault) / "People"`. A new vault arrives with `people/`. On a case-insensitive filesystem (macOS APFS, the usual dev machine) the hardcoded path still resolves, so the mismatch is invisible — every test passes. The same lookup on a Linux host (a server, a CI box, a cloud clone) finds nothing and reports "no such person" — a confident empty result, not an error. The failure is silent twice over: hidden where developed, wrong-but-plausible where deployed.

## The Pattern

- In shared tooling, discover the folder by name, not by literal: iterate the vault root and match `d.name.lower() in ("people", "agents")` rather than joining a hardcoded path
- The vault's own `CLAUDE.md` wins on its conventions; the router records the quirk as a pointer note ("folders are lowercase here") rather than asking the vault to conform
- At registration time, exercise the shared tooling against the new vault once (the who-is probe, the inbox route) — a convention mismatch surfaces in one command instead of at first real use on the wrong host

## Why It Works

- Lenient matching makes the tooling correct on every filesystem, so the case-insensitive dev machine stops being able to hide the bug
- Leaving each vault's conventions alone keeps ownership honest: the vault documents itself, the router holds a pointer, nothing has to be renamed to join the federation
- One exercised probe at registration is cheaper than the debugging session on the host where the empty result finally matters

## When to Use

- Any tool that walks more than one vault/repo it doesn't own, keyed on structural names (folders, filenames, frontmatter keys)
- Federations that grow one member at a time, where each new member was scaffolded by a different hand or era

## When NOT to Use

- Inside a single vault's own tooling, where its `CLAUDE.md` *is* the contract — there, enforce the convention rather than tolerating drift
- Where lenient matching would merge things that are genuinely distinct (`Notes/` vs `notes-archive/`); leniency is for spelling variance of one concept, not for guessing semantics

## Watch-outs

- A confident empty result reads as truth (`stale-pointer-asserts-confidently.md`) — "no such person" and "wrong folder spelling" are indistinguishable to the caller, which is why the match must be lenient at the source
- Case-insensitivity of the dev filesystem is the exact class `unrun-checks-read-as-passing.md` warns about: the check ran, passed, and verified nothing

## Adjacent Patterns

- `registry-resolution-alias-and-reach.md` — the resolver this lookup rides on; same spirit of matching what humans/vaults actually wrote
- `wire-into-existing-flows.md` — registering a new member includes exercising the flows that now cross it

## Source

The personal agent session, 2026-07-30. A new a client project planning vault used `people/` where every registered vault used `People/`. The personal agent's cross-vault `who_is` lookup hardcoded the capitalized form; macOS resolved it anyway, so the lookup "worked" on the dev machine and would have silently found nobody on the Linux leader. Caught because registration included running the probe against the new vault; fixed by matching the folder name case-insensitively.
