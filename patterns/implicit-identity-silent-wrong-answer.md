---
type: pattern
date: "2026-07-31"
source: Live walk-and-talk field session, 2026-07-31 — an agent asked to read work email silently read a personal mailbox instead, one Gmail connector shared by two identities with nothing in the request or the response naming which
tags:
  - agents
  - credentials
  - multi-account
  - silent-failure
  - anti-pattern
---

# An Implicit Identity Is a Silent-Wrong-Answer Generator

An agent was asked to read work email and silently read a personal mailbox instead: one Gmail connector, two identities, and nothing in the request or the response named which one answered. The same operator independently runs two GitHub accounts (see `two-github-credential-paths.md`) where using the wrong one routes a PR or issue to the wrong place, a silent failure that's hard to undo. Same shape, different capability: whenever one tool or connector can answer for more than one identity, the identity is a hidden parameter, defaulted by whichever credential happens to be active, not by what was actually asked.

## The Pattern

Make the identity an explicit, required parameter, then prove it's the one that actually answered:

1. **Require the identity in the call.** `--account personal|work`, not an optional flag with a fallback. If the caller doesn't know which identity they want, that's the bug to surface, not paper over with a default.
2. **Give each identity its own credential.** No shared token that both identities can silently fall back to. A shared token means a request for one identity can be silently satisfied by the other's session, which is exactly the failure this pattern names.
3. **Verify at authorization time, not just at call time.** Signing into the wrong account is indistinguishable from the right one from the tool's side, the OAuth flow succeeds either way. Check which identity actually consented, at the moment of consent, not just which identity was requested.
4. **Print the resolved identity on every read.** An answer can never be silently attributed to the wrong source if the source is in the output. This is the cheap, permanent backstop: even if steps 1-3 have a gap somewhere, a wrong-account answer that names itself gets caught on the first read instead of propagating.

## Why It Works

- An implicit identity fails the same way every time: whichever credential is ambient wins, and ambient credentials don't know what the user meant. Naming the identity moves the decision from "whatever's active" to "what was asked."
- Verification at authorization time closes the one gap that naming the parameter alone doesn't: a correctly-named request can still get consent from the wrong account if nothing checks who actually signed in.
- Printing the resolved identity is nearly free, one flag and one line of output, and it's the layer that survives every other layer failing. It converts a silent wrong answer into a visibly wrong answer, which is the difference between a bug that self-reports and one that doesn't.

## When to Use

- Any connector, credential, or capability that can authenticate as more than one identity: email, calendar, GitHub, cloud storage, a shared service account.
- Building or reviewing a tool wrapper around a multi-account API, where the wrapper's default behavior is "use whatever's currently signed in."
- An agent session that might run unattended, since a human would likely notice an unexpected mailbox or handle; an agent reading its own output won't unless the identity is printed where it can see it.

## Adjacent Patterns

- `two-github-credential-paths.md` — the git/gh instance of this same operator's multi-account problem: two accounts, two separate routing mechanisms, both need pinning and verification independently.
- `secret-handling-off-transcript.md` — a different concern (keeping a secret's value out of the transcript) that shares the same fix shape: don't let one credential path stand in for a decision that should be explicit.
- `sensitivity-tiered-access-control.md` — controls what an identity is allowed to read once resolved; this pattern is upstream of that, making sure the right identity was resolved at all.

## Source

Live walk-and-talk field session, 2026-07-31.

## Also seen: 2026-09-09, a mount point is not an identity

Cryptomator mounts a vault at `/Volumes/<obfuscated>` and makes no promise it picks the same path twice. After a mount death the old path survived as an **empty stub on the boot disk**, so a job pointed at the hardcoded constant would have written 88 GiB into a directory that merely looked right, with nothing raising. And two crypts exist on that hardware, so *being mounted* does not say which vault answered. Exactly this pattern's prescription applied to a filesystem: resolve the handle at run time from the mount table, and prove the resource by its **contents** before writing — with distinct refusals for nothing-mounted, several-matching, in-the-table-but-unreadable, and readable-but-wrong-vault, because those are four different fixes.
