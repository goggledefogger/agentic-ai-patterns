---
type: pattern
date: "2026-10-10"
source: The personal agent 2026-10-10 session, 2 expired CI tokens found by their failures
tags:
  - secrets
  - monitoring
  - rotation
  - anti-pattern
---

# A CI Secret Without a Vault Row Dies Unseen: Register a Credential When You Mint It

2 personal access tokens were minted on 2026-06-25. They lived only as repo secrets. The credential monitor reads the vault, so it never knew they existed. About 90 days later they expired.

A site build had not succeeded since 2026-09-18, with 52 of 60 runs failing. A dispatch from 1 source repo returned 401 from 2026-09-24. Nobody was paged. A sibling consumer's token had been rotated on 2026-09-18, and this one was missed.

## The Pattern

1. **Add the credential's row in the same change that mints it.** If the monitor cannot see it, it does not exist
2. **Rotate every consumer in one sitting.** List all places a credential is used before rotating, and update each one before closing the task
3. **Keep expiry dates where the monitor reads them.** A date in someone's head is not a date
4. **Alert on the consumer too.** A failing run rate is a signal. 52 of 60 should have paged someone

## When It Shows Up

Any token with a built-in expiry, copied into a place only a build system reads. The tell is a rotation that fixed 1 consumer and was declared done.

## Adjacent Patterns

- `a-title-is-not-a-path-segment.md` for a monitor that invents failures
- `an-unguarded-pipe-sets-an-empty-secret.md` for a rotation that wrote nothing
