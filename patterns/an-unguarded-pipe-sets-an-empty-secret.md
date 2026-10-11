---
type: pattern
date: "2026-10-10"
source: The personal agent 2026-10-10 session, rotating 2 CI secrets with a one-line pipe
tags:
  - secrets
  - shell
  - verification
  - anti-pattern
---

# An Unguarded Pipe Sets an Empty Secret: Prove the Credential Before You Write It

A rotation used 1 line: read the secret from the vault, pipe it into the CI secret setter. The read failed. The pipe carried nothing. The setter wrote an empty value into 2 repo secrets and reported success. The consuming workflows were then rerun, against nothing.

A pipe passes the exit status of its last command only. The setter saw valid input of length 0 and had no reason to complain.

## The Pattern

1. **Prove the credential by use.** Call the real API with it first. Proceed only on a 200
2. **Write only on success.** Hold the value in a variable, check it is non-empty and proven, then write it
3. **Confirm by rerunning the consumer.** A set command that exits 0 proves the write, not the value. The rerun proves the value
4. **Fail the pipe.** In shell, set `pipefail`, or avoid the pipe and use 2 steps

## Why It Works

- **Use is the only real test.** Presence, length and exit codes all pass on a wrong value
- **The consumer is the check that matters.** It is the thing that needed the secret

## When It Shows Up

Any one-liner that moves a secret between stores. The tell is a success message and a downstream 401.

## Adjacent Patterns

- `a-ci-secret-without-a-vault-row-dies-unseen.md` for how a dead CI secret stays unnoticed
