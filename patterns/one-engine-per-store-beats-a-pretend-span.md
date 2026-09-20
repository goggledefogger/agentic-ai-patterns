---
type: pattern
date: "2026-08-28"
source: The course — the dashboard PR #38; a multi-store brain whose search bridge served exactly one vault
tags:
  - semantic-search
  - multi-store
  - mcp
  - architecture
  - migration
---

# One Engine Per Store Beats a Pretend Span

A brain whose memory spans several stores (a home plus N vaults) had a search engine that took exactly one vault path. The organs (map, backup, projects) had already learned the multi-store contract; search silently hadn't. Measured: 5,074 of 5,661 notes were unreachable, and a query about a person with 22 files in one store returned only home-vault noise — with nothing in the result to signal the miss. `an-unnamed-blind-spot-reads-as-an-empty-source.md`, at vault scale.

The tempting fix is to teach the engine multiple roots. The shipped fix was to not touch the engine at all: **wire one server *entry* per store**, all sharing one installed engine — `search` for the home, `search-<store-slug>` for each additional store, each with its own single-vault env. MCP namespaces the tools per entry for free.

## The Pattern

- **N config entries × one unchanged single-vault engine.** No fork of the engine, no upstream feature negotiation, reversible per store, and it does not pre-decide the cross-store-index question a real multi-root engine would force.
- **The approval list is computed from the same function that wires the entries.** Project-scoped MCP servers sit behind an approval gate; a store wired but not approved is a server that installs and never loads — the classic silent half. One function produces both the entries and the allow-list, so they cannot drift.
- **Prune only what you wrote, proven by substance.** Ownership of an entry is established by its args pointing into the clone this wiring manages — never by name shape alone. A hand-wired entry under a similar name survives; a stale entry for a store that left `MEMORY_ROOTS` is removed.
- **Tell the agent the shape.** The session brief states: search is per-store; a real search asks every store; "nothing" requires every store's health-check license (`negativeResultsTrustworthy` per store), or you name the stores you could not trust.
- **Caches that land inside member stores must ignore themselves.** The per-store index writes megabytes of vectors *inside* each vault, and no vault's own `.gitignore` can be assumed to cover it (found: an 8MB untracked file in a live repo, one `git add -A` from history). The cache directory carries its own `.gitignore` holding `*`, written by whoever creates it — fixed at the engine (root cause) *and* at the wiring (installs pinned to older engine tags).
- **Say the ceiling.** N stores = N processes per session and N× the tool surface in context. That cost is stated where the wiring is documented; collapsing it into one cross-store index is named as the open question it is, owned by a real decision, not smuggled in by a config change.

## The migration corollary

Per-store wiring makes "import my old vault" a spectrum instead of a cliff: a vault can join as a store (no passes, no frontmatter rewrites, no risk — it stays where it lives, still opened by its own tools) and *optionally* move into the home later, store by store, reversibly. Never present migration as all-or-nothing; adopting-in-place is the common case for anyone with a living system.

## When to Use

- A single-target engine (search, indexer, watcher, backup) meets a multi-root world and the multi-root feature would be a fork, a big upstream ask, or a decision someone else owns.
- Any config writer that manages entries in a file members also hand-edit: prove ownership by substance before pruning, compute the allow-list and the entries from one source.
