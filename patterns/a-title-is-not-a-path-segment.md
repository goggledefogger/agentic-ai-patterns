---
type: pattern
date: "2026-10-10"
source: The personal agent 2026-10-10 session, a credential monitor reporting dead tokens that were alive
tags:
  - monitoring
  - secrets
  - identifiers
  - anti-pattern
---

# A Title Is Not a Path Segment: Build References From Ids, Not From Human-Written Names

A credential monitor built its secret references from item titles in a vault. The titles are written by people. 2 new items had titles carrying a comma and a slash. The reference grammar rejected both strings, and both rows read DEAD. The far end accepted the tokens the whole time.

The monitor reported a failure that did not exist. A monitor that invents failures gets muted, and a muted monitor misses the real one.

## The Pattern

1. **Reference things by id.** The lister returns the id beside the title. The reference is built from the id. The title stays as a label for the human
2. **Treat any human-written string as hostile to a grammar.** Commas, slashes, spaces and quotes all appear in names that people choose
3. **Check that a DEAD row can only mean dead.** A parse error and a failed use are different results. Print them differently
4. **Prove a false alarm false at the far end.** Use the credential for real before declaring the monitor wrong or right

## When It Shows Up

Any tool that joins a name into a path, a URL or a reference string. The tell is a new item that breaks a monitor that was green yesterday.

## Adjacent Patterns

- `a-ci-secret-without-a-vault-row-dies-unseen.md` for the opposite failure, a real credential the monitor never saw
