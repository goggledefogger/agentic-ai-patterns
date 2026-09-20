---
type: pattern
date: "2026-07-22"
source: A shared tooling stack — a doc + render-time join producing per-artifact attribution panels
tags:
  - documentation
  - attribution
  - architecture
---

# Curated Manifest + Runtime Join: Docs That Describe What's Actually Running

To show "the technology behind this artifact," the obvious moves are both wrong. Dependency introspection (walk package manifests) surfaces noise — build tooling, transitive deps — and misses the story: a wire format or an HTTP trick isn't a dependency. Hardcoding the story into templates rots instantly and can't serve multiple surfaces.

The shape that works: **one curated markdown manifest** describing the system as typed components (kind, one-line what, role, integration, links — closed enums validated by a ~30-line parser that fails loudly), **joined at render time with the artifact's real parameters**. The generic TiTiler entry plus this render's actual collection id and tiler host produces "serving `sentinel-2-l2a` via `titiler.xyz`" — the panel describes what is on screen, never a generic diagram. The same layer renders differently on different artifacts because it is telling the truth about each one.

## The Pattern

- **Markdown is the contract** so non-implementers can edit it. `## Name` blocks with `- key: value` lines; the parser is small enough to rewrite in an afternoon.
- **Closed enums are the policy layer.** Constraints you care about become field validation, not review vigilance — e.g. an attribution field restricted to role values (`created / maintains / contributes / uses`) makes "projects, never people" structurally impossible to violate, rather than a rule someone has to remember.
- **The join is deterministic code**, not model output: manifest entry + runtime facts → instance line. No LLM in the render path, nothing to hallucinate an attribution.
- **Curation is the feature, not a compromise.** The file says what *matters*, which no scanner can know. It's a small editorial artifact worth keeping good — that's intended, the same way a good changelog is.
- **Degrade, don't die:** manifest missing at runtime → the artifact renders without the panel and says so in its return value, not an exception in the happy path.

Generalizes to any "explain yourself" surface: about pages, credits screens, SBOM-adjacent attribution panels, `/.well-known/` self-descriptions — anywhere the honest answer is a join between what a human curated and what the process actually did.
