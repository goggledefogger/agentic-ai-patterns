---
type: pattern
date: "2026-11-03"
source: The personal agent's dashboard (2026-11-03): an automated security review after merge of a change that made the secret resolver read through a service account, found the human prompt had been the gate for unexpected references
tags:
  - approvals
  - secrets
  - confused-deputy
  - review
---

# Removing a Prompt Moves the Gate

A dashboard asked its owner for a fingerprint on every restart, because its secret resolver read references to a password manager through the owner's desktop session. To stop that, the resolver was changed to read through a service account token whenever one was configured. The prompts stopped. A security review run after the merge found 2 problems the change had created.

The first was a confused deputy. The resolver now served every reference silently, including references saved into the dashboard's key store from outside. Before, an unexpected reference met a human prompt. Now anything that could write to the store could name a secret and have the resolver fetch it with no one asked. The second was the token itself. Set in the server's environment, it was inherited by every chat agent the server spawned, handing each one the silent identity.

Both were fixed within the hour. The cost was that the review came after the merge.

## The Pattern

A prompt is also a gate for every input that reaches it. When you replace it with a silent identity, the gate does not disappear, it moves to whatever now decides what the silent identity may do.

1. **List the inputs that flow through.** Everything that used to hit the prompt, including inputs from stores, configs, or other writers, now arrives with no human in front of it
2. **Scope the silent path to what the operator declared.** Serve only the references the operator's own launcher put in the environment. Anything that comes from a store still meets the normal prompt
3. **Keep the credential out of child processes.** Strip the token variables from the environment of anything the server spawns, so the silent identity is not inherited by code that never needed it
4. **Review a prompt-removing change before merge.** The question for the review is "what did this prompt stand in front of", and it is cheap to ask before the merge and awkward after

## Why It Works

- **Silence widens the audience.** A prompt limited the callers to those a human would approve. A silent path is limited only by who can reach it
- **Declared beats discovered.** An allowlist the operator set at launch is intent. A reference found in a store is just data
- **Environments are inherited by default.** A secret placed there reaches every descendant unless something removes it

## When to Use

Any change that swaps a human approval for a service identity, a cached grant, or an always-on token. Also worth a pass on older silent paths that were once prompted.

## Source

The personal agent's dashboard, 2026-11-03. See `an-approval-that-guards-nothing-is-only-friction` (deciding which prompts to remove in the first place), `decorative-gate`, and `an-agent-can-wire-a-secret-it-never-holds`.
