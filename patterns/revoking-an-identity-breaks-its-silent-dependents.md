---
type: pattern
date: "2026-07-31"
source: An IAM cleanup removed a personal account from one cloud project as part of a hardening pass. A CLI still held a stored login token for that identity, so deploys to that project broke immediately and nobody noticed for two days, because only a deploy attempt surfaces it
tags:
  - credentials
  - identity
  - security
  - silent-failure
  - anti-pattern
---

# Revoking an Identity Breaks Everything Still Authenticating As It, Quietly

Removing a human account from a system is a deliberate, well-understood security action. What it also does is break every stored credential that authenticates as that human, and those live somewhere nobody is looking: a CLI's config directory, an application-default credential file, a CI secret, a laptop keychain. The revocation is instant and correct. The breakage is invisible until somebody happens to exercise the path.

## The Problem

Access removal is designed to be silent. That is the point of it. But the identity's *consumers* are distributed and unenumerated, and they fail in three ways that all defeat detection:

**The error names the resource, not the identity.** Losing access does not produce "your credential no longer works." It produces the resource looking absent: *"Invalid project selection, please verify the project exists and you have access."* That reads like a typo or a deleted resource. The natural next move is to check the resource, find it healthy, and get more confused.

**The path is used rarely.** Deploys, restores, and admin scripts run weekly or monthly, not hourly. The gap between the revocation and the discovery is however long until someone next needs it, which is exactly when they are least able to absorb a detour.

**Adjacent systems keep working, which reads as evidence.** If the same identity retains access to a second project, that project's path is fine. "It worked on the other environment" gets taken as proof the credential is good and the problem is local.

Worst case: the same revocation is scheduled for the production environment next, and it will produce the identical failure there.

## The Pattern

**1. Enumerate the consumers before the revocation, not after.** For the identity being removed, list everything that authenticates as it. The list is short and almost always includes more than the person expects:

- CLI login tokens in tool config directories, which survive independently of the cloud-side grant
- application-default credentials, a separate file with a separate lifecycle from the interactive login
- CI/CD secrets and deploy keys
- anything impersonating the identity or inheriting from it

**2. Re-credential before revoking, in that order.** Point the consumers at the replacement identity first, verify one real operation, then remove the old grant. Reversed, you get a window of unknown length where a rarely-used path is broken and nothing says so.

**3. When the error blames the resource, run a differential check.** Three reads separate "the resource is gone" from "my identity lost access", and they take seconds:

```
<tool> auth whoami            # which identity is the tool actually using
<platform> describe <resource>  # does the resource exist, as an identity that CAN see it
<platform> get-policy <resource> | grep <old-identity>   # is the old identity still bound
```

A healthy resource plus a zero-binding grep is the whole diagnosis. Run the same grep against a working environment for contrast, a `1` there explains why that one still works and confirms the mechanism.

**4. Do not fix it with a long-lived key.** The pressure at this moment is to make the pain stop by minting a service-account key or a CI token. That reintroduces exactly the long-lived credential the hardening pass was removing, usually with broader scope than the human had. Re-login interactively, or use short-lived impersonation.

## Why It Works

Enumeration converts an unbounded question ("what might break?") into a finite list you can walk, and the list is genuinely short. Ordering re-credentialing before revocation removes the broken window entirely instead of shortening it. The differential check works because the two hypotheses predict different observations, where the error message alone is consistent with both.

## When to Use

- Any IAM cleanup that removes a human account from a project, org, or repo
- Rotating away from a shared or legacy identity
- Offboarding, where the same shape applies and the person is not around to say what authenticated as them
- Immediately before repeating a revocation on a second environment, since the first one already demonstrated the failure

## When NOT to Use

- Adding access, which fails loudly and immediately at the point of use
- An identity created minutes ago for one purpose, where the consumer list is known by construction

## Watch-outs

- **The interactive login and the application-default credential are different files with different lifecycles.** Refreshing one does not refresh the other, and a stale ADC can be months older than anyone assumes
- **Re-login can inherit the wrong default.** A fresh application-default login may adopt whatever project the CLI currently points at and announce it in a line that reads like routine confirmation. Set the project explicitly afterwards and re-read it
- **Verify which identity actually answered, do not trust the consent screen.** It may carry a login hint for the account being retired. Resolve the token and print the email
- **A monitor will not catch this.** There is no failing request to alert on until someone makes one. The control is the pre-revocation enumeration, not detection

## Adjacent Patterns

- `implicit-identity-silent-wrong-answer.md`, the sibling failure where two identities are both live and the wrong one silently answers. Here only one is live and the dead one is still configured
- `two-github-credential-paths.md`, the same lesson at smaller scale, one tool's credential fixed while another path still routes by the old identity
- `verification-needs-a-negative-control.md`, why the contrasting environment matters, a check that passes everywhere proves nothing

## Source

A cloud hardening pass removed personal accounts from one project so that access flowed only through org-level admin identities. Correct and intended. The CLI used for deploys held a stored login token for a removed account, so `use <project>` began failing with a message about the project not existing or being inaccessible. The project was healthy the whole time. Two days passed before anyone attempted a deploy.

Confirmed by the differential: the CLI reported the old identity, the project described as ACTIVE under an admin identity, and the old identity had zero bindings on that project against one on the not-yet-cleaned environment, which is the only reason the second environment still worked. That second environment was scheduled for the same cleanup, so the same break was queued up for the more important system.

The team's own runbook had anticipated it, in a parenthetical: *"re-login once migration lands."* The migration landed. The re-login did not happen. A note that records a dependency without a forcing function only reads as prescient afterwards.
