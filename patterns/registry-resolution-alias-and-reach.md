---
type: pattern
date: "2026-07-11"
source: The personal agent session (2026-07-11), a thin-router resolver that hard-failed on a human nickname and dead-ended on a host with no local path, when both misses had an answer already on file
tags:
  - orchestration
  - routing
  - architecture
  - registry
---

# Registry Resolution: Alias-First, Dead-Ends Return the Next Hop

Two refinements to a thin router's resource resolver. Resolve the names a human actually uses through an alias index before inferring where a thing lives, and when a local path cannot be resolved on this host, return the next way to reach the thing instead of a bare failure. Both keep the router thin, they add pointers, not contents.

## The Problem

A thin router (`thin-router-orchestrator.md`) resolves a canonical resource name to its tier and a logical `where` key, then a per-host location map turns `where` into a local path. Two frictions show up the first time a real person uses it.

First, people do not say the canonical name. They say "the guitar trainer" or "the tax stuff," a nickname or a project handle. The resolver does not match it, so the router infers, guesses, or asks the human where the thing lives, even when the registry already holds the answer under a different name.

Second, on a host where the location map has no entry for that `where` key, for example a fresh cloud box that has no mount for a vault, resolution returns a bare "unresolved." The caller dead-ends, even though the thing is reachable another way (clone the repo, hit an API, pull it into scope on demand). The resolver knew the reach path existed and said nothing.

## The Pattern

Add two behaviors to the resolver, both pointer-only.

### Alias index, consulted before inference

Keep a small handle-to-resource table separate from the registry. A nickname, phrase, or project handle maps to a canonical resource name, which then resolves the normal way through the registry and the location map. Consult it first, before any inference or asking. When the router learns a new handle for something it already tracks, it appends a row. Keep people out of it, a person handle resolves through the who-is path so there is one source of truth per kind.

The alias lookup is the cheapest, most authoritative probe for "what does this name mean," so it belongs before vault recon or a code search, not after.

### Reach vector on a local miss

When the `where` key has no entry on this host, do not stop at "unresolved." Return the next reach vector, the concrete next step to get to the thing on this host, for example "not mounted here, reachable as repo X, pull it in on demand." Carry that reach pointer in the registry or alias note so the resolver can surface it. A dead-end becomes an action.

### Same handle, two entities: disambiguate in the index, by domain context

The alias table's third job appears the first time two entities share a handle — a second "Nico" arrives in a new domain when an "Nico" already lives in another. Bare-name resolution now routes wrong *silently*, in whichever direction the older row points. The fix stays in the index: one row per entity, each carrying the domain context that selects it ("Nico (a client project context) → Ridgeway"; "Kit → Kitson, a course student"), plus an explicit note on the ambiguous bare handle saying *context decides, and when it doesn't, ask*. The alias index is the right home because it is consulted before inference — a disambiguation living anywhere later (a person file, a vault note) fires only after the wrong route is already taken. Detect collisions at registration time: when a new resource brings people files, run the who-is probe on each first name before wiring, not after the first mis-route.

## Why It Works

- The alias probe is nearly free and authoritative, so ordering it first turns a class of "where does this live" misses into a lookup instead of a guess or a question to the human
- Returning the reach vector means the caller acts instead of stalling, the resolver stops hiding what it already knew
- Both stay pointer-only, so the router keeps the thinness the whole architecture depends on

## When to Use

- A thin-router registry where humans name resources by nickname, phrase, or project handle
- Resources that resolve differently per host, so a location miss on one machine is normal, not an error
- Any resolver whose current failure output is a bare "unresolved" with no next step

## When NOT to Use

- A single-host, single-name registry with no aliases and no cross-host reach variance. There is nothing to alias and nothing to fall back to
- A resolver whose only correct answer to a miss is a hard failure by policy, where surfacing an alternate path would be a security bypass, not a help

## Watch-outs

- Aliases are pointers, never contents. A wrong alias routes wrong silently, so leave a handle out when you are unsure what it maps to
- Keep person and agent handles in the who-is or people path, not the resource alias index, one source of truth per kind
- The reach vector is a pointer, not a gate bypass. Surfacing "reachable as repo X" still routes through the same tier and send gates as any other access

## Adjacent Patterns

- `thin-router-orchestrator.md`, the registry and per-host location map this refines
- `registry-based-monitoring.md`, the JSON-registry-plus-auto-discovery shape the resource registry borrows
- `self-reporting-staleness-check.md`, the drift check that keeps alias and registry pointers from rotting

## Source

The personal agent session, 2026-07-11. The router had the tier for "the guitar trainer" instantly (it classifies by tier first) but had no alias mapping the handle to the resource, so it asked the human which repo, when the alias index already recorded it. Separately, the resolver returned a bare "unresolved" on a cloud host with no local mount, hiding that the code was reachable as a GitHub repo. Fixing the resolver to consult an alias index first and to surface the reach note on a local miss closed both.
