---
type: pattern
date: "2026-07-10"
source: The personal agent job-applications session, 2026-07-10 — a cloud session declared a personal-private resource "unreachable" because its local path resolved to a deny-safe sentinel, missing that the same resource was also a private GitHub repo reachable via a discovery/clone channel the path-keyed deny hooks never see. The reach-surface companion to `router-worker-exfil-containment.md` and `gate-propagation-into-headless-workers.md`, both of which assume reach == the paths and tools the hooks watch.
tags:
  - routing
  - security
  - layered-defense
  - resolution
  - hooks
---

# Reach-Surface Resolution

A resource's reachability is a *surface*, not a single path. It can have several backing stores — a local vault, a git remote, a synced folder, an API — and a deterministic path-keyed gate governs only the ones it can resolve to a watched path. Resolve the whole surface, flag which reaches the gate covers, and treat any reach through a channel the gate can't see as a named, human-authorized exception, never a routine door.

## The Problem

`sensitivity-tiered-access-control.md` and its exfil companions all key on **filesystem paths**: a deny hook resolves a resource's sensitive path through a host location map and blocks tools that touch it. That holds when the only way to reach the resource is that path. It breaks the moment the resource has more than one backing store:

- A vault that is also a **private GitHub repo**.
- A folder that is also a **Syncthing / Drive mirror**.
- A dataset reached by a **local path on one host and an API on another**.

The alternate store is reached by a **discovery/clone channel** — `add_repo`, `list_repos`, `git clone` to an arbitrary path, an API pull — that produces real reach *without ever creating the path the gate watches*. Two failures follow, and they compound:

1. **False negative → phantom wall.** On a host where the local path resolves to a deny-safe sentinel, the resource looks *unreachable*. The operator reasons from that ("it only lives on the other machine"), states it as fact, and even designs around the phantom constraint — when a second, wide-open door was one discovery call away. The inferred "unreachable" is a bluff: the resource was reachable the whole time, just through a store the resolver never mentioned.
2. **Ungated pull.** Because the discovery channel never touches a watched path, pulling the resource that way lands its contents in the agent's context behind **no gate at all**. The tier deny hook, the exfil containment, the send gate — none fire on a repo the operator `add_repo`'d and cloned to an unregistered path. The safety story the path-keyed gate tells has a hole exactly the width of the alternate backing store.

## The Pattern

**Make resolution return the full reach-surface, flag each reach for gate coverage, and make any gate-blind reach a named exception rather than a default.**

Three moves:

1. **Resolve the surface, not a path.** The resolver answers "where and how is this reachable" with *every* known backing store, not the first one that fails. Its output is a set of reaches — `{local_path (or None + why), repo, mirror, api}` — plus the tier. `no local path here` then means "this one door is closed on this host," never "the resource is unreachable." The operator can't infer a phantom wall, because the resolver already handed back the other doors.

2. **Flag each reach for gate coverage.** For every reach, the resolver marks whether the deterministic gate can see it: a local path registered in the host location map is **in-gate**; a repo reachable only via `add_repo` / clone-to-unregistered-path is **out-of-gate**. The safety fact ("this door bypasses the gate") is carried by the tool, not by the operator's memory of a rule.

3. **Gate-blind reach is a named exception.** Reaching a resource through an out-of-gate channel is allowed only as a deliberate, human-authorized exception with the reason named — never the routine reach. The gated route (resolve to a registered local path, dispatch a worker behind the gate bundle) stays the default. "It was convenient" or "the sanctioned path was down" is a reason to *ask*, not to quietly walk the ungated door.

## Why It Works

- **No phantom walls.** "Unreachable" becomes a computed conclusion over the whole surface, not an inference from the first closed door. The false negative that hides an ungated channel can't form.
- **The blind spot is made visible.** Labeling each reach in-gate / out-of-gate turns an invisible hole into a marked one. You can't accidentally route through a channel you can see is outside the gate.
- **The safety fact outlives the operator's memory.** Encoding gate-coverage in the resolver's output means a fresh context, a new agent, or a tired operator is still told "this reach is ungated" — the same reason `sensitivity-tiered-access-control.md` puts enforcement in a hook, not a CLAUDE.md rule.
- **Default-gated, exception-ungated.** The routine path stays behind the gate bundle; the ungated channel costs a human "yes" each time, so its use stays rare, deliberate, and auditable.

## When to Use

- Any `thin-router-orchestrator.md` whose resources can have more than one backing store (local + git remote is the common case), especially across hosts where a path resolves on one machine and sentinels on another.
- Any environment with a discovery channel (`add_repo`, repo search, an API pull) that can reach a resource by a route the deterministic gate doesn't cover.

## When NOT to Use

- A single-backing-store world: one vault, one path, no git remote, no API. There is one door and the path-keyed gate already covers it.
- A resource whose every reach is in-gate by construction — nothing to flag, no exception to name.

## Watch-outs

- **Discovery tools are reach, not just listing.** `list_repos` / `add_repo` feel like read-only inventory, but the instant they let you clone, they are a reach channel. Gate the ability to *pull the resource in*, not the ability to *see it exists*.
- **A clone to a registered path is in-gate; a clone anywhere else is not.** The gate keys on the location map, so cloning a sensitive repo to a path the map lists keeps it gated — cloning it to `/tmp/whatever` does not. If you must clone a sensitive resource locally, clone it where the gate can see it (see `gate-propagation-into-headless-workers.md`); if you can't, that's the named exception.
- **Open-tier resources still deserve the honest label.** Reaching an open-tier repo via the ungated channel is fine — but record *why* it was fine (it's open tier), so the move doesn't silently become the template for the next resource, which might not be. That is exactly the trap that turns one justified exception into a standing hole.
- **Resolve-don't-infer is the human half.** The resolver returning the full surface only helps if the operator *runs* it before asserting where something lives or whether it's reachable. A confident "it only lives on X" that was never resolved is the same bluff this pattern exists to kill.
- **An address's shape is not a reachability fact.** The inverse bluff, and it ships as code rather than as a sentence. A launcher classified links by pattern-matching the URL — a private range (`192.168.*`, `10.*`) meant "home-network service, works from home" — and rendered that on the card as a promise to the user. It was wrong for four rows: the box hosting them advertised its whole subnet as a Tailscale **subnet route**, so every tailnet device reached those private IPs from anywhere, including a phone on cellular. Mobile clients pick up approved routes automatically, so nothing on the reading device had to opt in for the "LAN-only" label to be false. The private-range check answers *what kind of address is this*, which merely resembles *can I reach this from here*. Same trap as the discovery channel above, arriving from the opposite direction: there, an unmodelled channel created reach the gate could not see; here, an unmodelled channel created reach the **classifier** could not see, and the UI confidently understated it. VPN meshes, subnet routers, exit nodes, split-horizon DNS, and corporate zero-trust proxies all do this. If a label claims reach, derive it from a probe, or word it as the guess it is.

## The scar keeps re-opening, and the newest one is the cheapest to avoid (2026-08-17)

The 2026-07-23 instance above was an agent reasoning its way to a wrong conclusion about a repo. The 2026-08-17 instance took **one denied tool call**, and it is worth recording because the failure needed no reasoning at all.

Asked to read a user's email from a phone-driven session, the agent called the connector tool, received *"you haven't granted permission"*, and answered **"nothing I can do from here."** A working headless route had existed for two weeks — a script with its own OAuth token, wired specifically so that mail could be read from exactly this host — and the resource's own registry row listed it in the reach column, one line below the connector that had just failed.

Three things generalise:

- **A permission denial is the loudest possible signal about ONE door, and says nothing about any other.** It arrives with an error message, which makes it feel like a verdict on the resource. It is a verdict on the tool.
- **The reach column must be consulted on failure, not only when planning.** Enumerating the surface up front is the discipline this pattern already prescribes; the miss happens at the moment of the first refusal, when the natural move is to report the refusal rather than resolve the surface. Make the failure path re-enter the resolution step.
- **The bluff is more convincing when it quotes a real error.** "It only lives on X" is a claim someone might challenge. "Permission denied" is evidence — of the wrong proposition.

There is a second-order trap in the recovery, too. The retry ran the working script through an interpreter (`python3 script.py`) rather than by its path, which stripped the dependency declarations in its shebang and produced a `ModuleNotFoundError` for **every** account. That reads exactly like a broken host, and very nearly bought a second false "unreachable" verdict within the same minute. **When a reach you just discovered appears broken, suspect your invocation before you suspect the route.**

## Adjacent Patterns

- **`router-worker-exfil-containment.md`** — closes the exfil exits for reaches the gate *can* see; this pattern makes sure you've enumerated the reaches it *can't*. Read together: contain the known doors, surface the unknown ones.
- **`gate-propagation-into-headless-workers.md`** — how a gated reach travels into a worker (clone to a registered path, ship the gate bundle). The in-gate route this pattern points the default at.
- **`sensitivity-tiered-access-control.md`** — the path-keyed gate whose blind spot (alternate backing stores) this pattern names. "Encode the rule as code, not memory," applied to reach-surface rather than read-gating.
