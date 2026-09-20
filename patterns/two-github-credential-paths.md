---
type: pattern
date: "2026-07-31"
source: A work/personal GitHub account split where fixing git push left gh writes still routing to the wrong account
tags:
  - git
  - github
  - credentials
  - multi-account
  - silent-failure
---

# git push and gh Are Separate Credential Paths

If you hold two GitHub accounts, `git push` and the `gh` CLI authenticate through entirely different mechanisms. Fixing one leaves the other exactly as exposed as before, and the second failure is the quieter of the two.

## The Problem

A vault or work repo pushes as the wrong account. You fix it, verify the push works, and move on. Later a `gh pr create` opens a pull request on an internal company repo under your personal handle.

The two paths never touched each other:

| | `git push`, `git ls-remote` | `gh pr`, `gh issue`, `gh api` |
|---|---|---|
| routed by | credential helper | `GH_TOKEN`, else gh's active account |
| configured in | git config | environment, or a directory `.envrc` |
| scope | per repo or per remote | per shell |

Worse, an exported `GH_TOKEN` beats gh's own stored account. If it is set machine wide, `gh auth switch` does nothing, and every repo without a directory override inherits whichever account that token belongs to.

Both failures are silent in the direction that matters. A push to a private repo your other account cannot see fails loudly with "repository not found", which is survivable. A `gh` write succeeds under the wrong identity, which is not.

## Check the Agent Harness, Not Just the Shell

When you go looking for where an ambient token comes from, shell profiles and `launchctl` are the obvious places and they can all come back clean while the token is still there in every session.

The harness is the third place. A Claude Code `SessionStart` hook writing `export GH_TOKEN=$(gh auth token --user someone)` into the session environment supplies a token to every agent run on the machine, and it lives in `~/.claude/settings.json` rather than anywhere a shell audit would look. Grep the harness config alongside the shell config, or you will conclude a token is machine wide when it is really per session, and fix the wrong thing.

This matters more than it sounds, because agent sessions are exactly where the wrong-account write happens. A human notices the unexpected handle in a prompt. An agent does not.

## The Pattern

Fix both paths, separately, and verify each.

**For git, route by remote URL rather than by directory.** Git 2.36 and later supports conditional includes keyed on the remote, which works no matter where the clone sits on disk:

```
[includeIf "hasconfig:remote.*.url:https://github.com/personal-org/**"]
	path = ~/.gitconfig-personal
[includeIf "hasconfig:remote.*.url:https://github.com/work-org/**"]
	path = ~/.gitconfig-work
```

Order matters. A fork matching more than one condition takes the last match, so list the one that should win last. Pin both sides, not just the one that broke. Leaving the other on the ambient default recreates the same trap pointed the other way.

Note that a leftover per-repo helper in a single clone's config beats the global include, so clearing old local overrides is part of the fix rather than housekeeping after it.

**For gh, give each work repo a directory `.envrc`** that exports the right `GH_TOKEN`. Keep it out of the repo, since it names one person's account and every collaborator would inherit it.

## Verify With a Hostile Environment

Checking that the right account answers under normal conditions proves very little, because the ambient default may be the right one that day. Force the wrong token in and confirm the pinning still holds:

```bash
GH_TOKEN=$(gh auth token --user wrong-account) \
  git credential fill <<< $'protocol=https\nhost=github.com\n\n'
```

Compare hashes rather than printing tokens. Run it in both directions, work repo and personal repo, because a one sided fix looks identical to a working one from the side you tested.

Then separately test what happens when the credential lookup fails, not just when it succeeds. Shadow the CLI with a stub that exits non-zero and check whether the helper returns empty (loud, safe) or falls through to some cached credential (silent, wrong). That distinction decides whether a fallback cache is worth adding, and it is not visible from a passing test.

## When to Use

- Two or more GitHub accounts on one machine, which for most people means work and personal
- An agent with push or `gh` access, since it will not notice an identity it never checks
- Repos intermixed in one directory tree, where a path based rule cannot separate them

## The Part That Does Not Get Fixed

`.envrc` only loads under direnv. A bare shell call in an agent session does not pick it up, so the directory fix helps a human at a terminal and an agent following a documented `direnv exec .` pattern, and does nothing for an agent that simply forgets.

That is why a verification rule stays binding no matter how good the routing gets. Run `gh api user --jq '.login'` before any `gh` write and treat a mismatch as a stop, not a warning. Routing removes the trap. Verification is what catches the day routing is not enough.

## Source

Built 2026-07-31 after a push to a private work repo failed with "repository not found" because the clone had no credential helper and git reached for the personal account. Fixing git push globally left `gh` still defaulting to personal, caught later the same session one command before a `gh pr create` would have opened a pull request on a shared internal repo under the wrong handle.
