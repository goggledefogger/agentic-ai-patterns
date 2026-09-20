---
type: pattern
date: "2026-07-11"
source: The personal agent session on Claude Code on the web (2026-07-11), an ephemeral cloud agent whose commits landed Unverified with no signing key and no secrets store available
tags:
  - git
  - workflow
  - security
  - agents
  - ci
---

# Platform-Signed Commits from a Keyless Agent

An ephemeral cloud agent can get platform-Verified commits without holding any signing key, by creating the commits through the hosting platform's own API with an app or installation token. The platform signs them with its key. Nothing secret enters the container.

## The Problem

A cloud coding session runs in a fresh, throwaway container. Its git is often pre-wired to sign commits (for example `commit.gpgsign=true`, `gpg.format=ssh`, a `user.signingkey` path), but the private key was never provisioned, so every commit lands Unverified. The container also has no `ssh-keygen` and no secrets store, so it cannot even perform the signature locally.

The obvious fix is to inject a private signing key as an environment variable and materialize it at startup. That works, but it puts long-lived key material into a place that is readable by anyone who can edit the environment and that transits the host on every use. In a container that is supposed to hold nothing, a persistent private key is the one thing you least want to leave lying around.

## The Pattern

Do not sign in the container. Let the platform sign.

Create the commit through the platform's write API using its app or installation token, instead of `git commit` plus `git push`. The platform signs the resulting commit with its own key and attributes it to the identity the token already authenticates. On GitHub this is the `push_files` or create-commit API with a GitHub App installation token, which yields a commit that shows Verified with no key anywhere in the session.

The move in one line: route commit creation through the API that already trusts your identity, rather than proving that identity again with a key you had to smuggle in.

## Why It Works

- The signing key never enters the ephemeral container, so there is nothing to leak, rotate, or scrub
- The platform already authenticated the token, so it can vouch for the commit without a second credential
- Keyless and ephemeral by construction, which is exactly the shape of a cloud agent session
- No dependency on `ssh-keygen`, `gpg`, or a local keystore that the base image may not ship

## When to Use

- An ephemeral or cloud CI agent with no persistent keystore that still needs Verified commits
- Any context where injecting a private key would mean a long-lived secret in plaintext, readable config
- Work that already flows through the platform's API and token

## When NOT to Use

- You need the commit authored as a specific human identity the platform will not sign as. API commits are attributed to the token's identity, which may be an app or bot, not you
- A local, offline, or self-hosted workflow with no signing platform in the loop
- You must sign existing local commits in place. The API creates new commits with new SHAs, it cannot retroactively sign history

## Watch-outs

- A plain `git push` from the container stays unsigned. The signing only happens when the platform creates the commit through its API, so route creation through the API, not git
- API commits carry the token's author and committer identity and get new SHAs, so a local branch of the same work will diverge. Reset the local branch to the pushed result if you want them to match
- The API path only reaches repos inside the token's scope. Out-of-scope repos will refuse the call
- Verify the badge out of band. The API response may not expose the signature field, so confirm Verified in the UI rather than assuming it

## Adjacent Patterns

- `pre-push-pr-discipline-hook.md`, the other end of git hygiene, mechanically blocking direct pushes to main
- `router-worker-exfil-containment.md`, the same instinct of keeping secrets and broad credentials out of an agent that does not need to hold them
- `sensitivity-tiered-access-control.md`, why a readable plaintext secret in shared config is a real exposure, not a convenience

## Source

The personal agent session on Claude Code on the web, 2026-07-11. Commits landed Unverified because the container had a signing config but no private key, no `ssh-keygen`, and no secrets store. Recreating the same changes through the GitHub `push_files` API with the session's app token produced commits signed by GitHub, attributed to the account identity, with no key ever materialized in the container. Cross-checked against GitHub's docs on commit signature verification for API and bot-created commits.
