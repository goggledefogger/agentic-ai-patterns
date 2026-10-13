---
type: pattern
date: "2026-10-07"
source: Items swept out of a public catalogue moved to a private folder beside the repo, and a local overlay merged them back in so the owner's copy stayed complete
tags:
  - privacy
  - publishing
  - data
  - architecture
---

# Ship the Public Catalogue With a Private Overlay Beside It

When some items in a catalogue cannot be public, do not delete them. Move them to a private file outside the repo and merge that file in only when the catalogue is served on the owner's machine. Privacy becomes a move, the owner's copy stays whole, and promoting an item later is moving it back.

## The Problem

A sweep of a public catalogue found items that could not ship: internal planning, a co-founded product's details, links to private repos. Deleting them would make the public version right and the owner's version poorer. The owner uses the catalogue too, and the private items are often the ones they care about most.

Keeping them in the repo behind a flag would be worse. A flag is one bug away from publishing them, and the file is still in the history.

## The Pattern

Two layers, one baseline.

```
catalogue/                      public repo
  data/projects.json            the baseline, everything public
catalogue-private/              sibling folder, never a git remote
  data/private-projects.json    items that are not public
  raw/                          the original sources
```

1. **The public file is the baseline.** It is complete on its own. A clone of the public repo works without the private folder
2. **The overlay merges in locally.** When the catalogue is served on the owner's machine, the loader looks for the sibling file and merges it over the baseline. If the file is missing, the loader does nothing
3. **Privacy is a move.** An item that cannot be public moves from the baseline to the overlay. Nothing is deleted
4. **Promotion is a move back.** When an item clears review, it moves to the baseline in a commit you can read
5. **New kinds of item start private.** When the owner adds a new category (music recordings, travel, photos), it lands in the overlay by default. Each item is promoted one at a time

The private folder sits outside the repo on purpose. No `.gitignore` line has to hold, and no clean-checkout script can sweep it in.

## Why It Works

The sweep stops being a loss. Without the overlay, every hard call about privacy is also a call about whether the owner keeps the item at all, and that pressure pushes toward leaving borderline items in. With it, the safe answer costs nothing, so the sweep can be strict.

It also fixes the direction of mistakes. New material is private until someone promotes it. Forgetting to review something means it stays at home, not that it ships.

## Watch-outs

- The loader must treat a missing overlay as normal. A public build that errors without the private file is a hint that someone will copy the file in to make it work
- Keep the overlay's item shape identical to the baseline's, so a move is a cut and paste and not a rewrite
- Back up the private folder. It is outside the repo, so the repo's remote is not its backup

## When to Use

Any public artifact derived from a larger private set: a portfolio, a catalogue, a dataset, a site built from notes. The more the owner uses the artifact personally, the more this matters.

## Adjacent Patterns

- [[a-derived-public-dataset-leaks-through-its-provenance]] is the sweep that decides which items move
- [[a-tier-says-what-you-may-touch-not-what-others-may-see]] is the same split at a different level: what the owner can see and what the public can see are separate questions

## Source

A public catalogue, 2026-10-07. Swept items moved to a private sibling folder with their raw sources, a local overlay restored them on the owner's machine, and the owner's next additions (recordings, travel, photos) went into the private layer first.
