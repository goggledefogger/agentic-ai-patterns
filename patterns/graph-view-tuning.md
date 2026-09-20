---
type: pattern
date: "2026-06-04"
source: A course vault — graph-view audit + Tier-1 config pass, 2026-06-04. 162-note vault with consistent `type:`/`tags:` frontmatter and heavy wikilinks rendering through an unconfigured graph (`colorGroups: []`). Native color groups keyed on the universal `type:` property; no plugin.
tags:
  - obsidian
  - graph
  - schema
  - frontmatter
  - no-plugin
---

# Graph View Tuning

A well-typed vault — every file carrying `type:` frontmatter, every entity wikilinked — already has all the graph edges it will ever need. The graph still looks like an undifferentiated hairball because the *data* is right and the *render* is unconfigured. Improving the graph is almost always a config problem, not a re-tagging problem, and it is solvable with zero community plugins. This pattern is the render-side companion to **wikilink-graph-hygiene** (which gets the data right); here we make Obsidian draw what's already there.

## The Problem

The reflexive move when someone wants "a better graph" is to install a graph plugin (Extended Graph, Juggl, a 3D renderer). That's backwards. Two things are usually true at once:

1. **The data is already good.** A vault that follows the non-negotiable rules — frontmatter on every file, wikilinks for every linkable entity — has a dense, correct edge set. People link to the projects they own (`owner: "[[nico-kitson]]"`), meetings link to attendees, daily notes link to sessions. Those edges render today.
2. **The render is at defaults.** Obsidian ships the graph with `"colorGroups": []`. Every node is the same color. There is no visual distinction between a person, a project, a decision, and a throwaway draft. The graph *is* a hairball — not because it's poorly connected, but because nothing tells the eye what it's looking at.

The default `.obsidian/graph.json` from a real 162-note vault:

```json
{
  "search": "",
  "colorGroups": [],
  "showTags": false,
  ...
}
```

Empty color groups, empty filter. The vault was doing everything right at the data layer and getting no payoff at the view layer. The fix touches one file and re-tags nothing.

## The Pattern

### 1. Color groups keyed on the universal `type:` property, not tags

Every content file has a `type:` (`daily | meeting | project | person | reference | decision | ...`). That single property is the cleanest axis to color by, because it partitions the vault into the categories you actually navigate. Obsidian's color-group search box accepts a **property bracket syntax** that reads frontmatter directly:

```json
"colorGroups": [
  { "query": "[type:person]",    "color": { "a": 1, "rgb": 14723118 } },
  { "query": "[type:project]",   "color": { "a": 1, "rgb": 5025616  } },
  { "query": "[type:meeting]",   "color": { "a": 1, "rgb": 3900150  } },
  { "query": "[type:decision]",  "color": { "a": 1, "rgb": 11032055 } },
  { "query": "[type:daily]",     "color": { "a": 1, "rgb": 9079434  } },
  { "query": "[type:reference]", "color": { "a": 1, "rgb": 1357990  } },
  { "query": "path:Inbox/",      "color": { "a": 1, "rgb": 15680580 } }
]
```

The `rgb` value is a 24-bit integer, not a hex string. The mapping above is gold person `#E0A82E`, green project `#4CAF50`, blue meeting `#3B82F6`, purple decision `#A855F7`, gray daily `#8A8A8A`, teal reference `#14B8A6`, red Inbox `#EF4444`. Convert with `int("E0A82E", 16)`.

**Key the groups on `type:`, not `tag:#type`.** The two look interchangeable — most `person` files also carry a `person` tag — but tags have gaps. A schedule note typed `reference` may be tagged `cohort-2, schedule` with no `reference` tag at all, so `tag:#reference` silently misses it. The `type:` property is on *every* content file by the frontmatter rule, so `[type:reference]` is exhaustive. Color by the property that's guaranteed present, not the tag that's merely usual.

The last group is keyed by `path:` instead — `Inbox/` is a *location* (work-in-flight), not a `type`, so coloring stray drafts red surfaces "things that need to leave the vault" regardless of their type. Mix property-, path-, and tag-keyed groups freely; they're all just search queries.

### 2. Filter noise by path — but keep the load-bearing folders

The graph filter (`"search"`) takes the same query syntax with negation. Strip the folders that are pure scaffolding:

```json
"search": "-path:Templates/ -path:_archive/"
```

The tempting generic advice is to also `-path:Daily/` — "daily notes are journal noise." **Resist that when daily notes are load-bearing.** In a course/work vault, `Daily/` is session notes: each one links to the people, projects, and decisions that moved that day. They're some of the most *connective* nodes in the graph. Excluding them would delete the connective tissue to cut the noise. Strip only what's genuinely inert — templates, archives, attachments (`showAttachments: false`). Judge each folder by whether its notes carry real outbound links, not by its name.

### 3. Make it additive — never overwrite hand-tuned physics

`graph.json` mixes two kinds of state: **config** you author (color groups, filter) and **spatial preference** the user dragged in (`centerStrength`, `repelStrength`, `linkDistance`, `scale`). The second kind is the product of someone hand-zooming and dragging the graph to where they like it. When you tune the graph programmatically, touch only `colorGroups` and `search` (and maybe set `"collapse-color-groups": false` so the Groups panel opens expanded). Leave the force and scale values exactly as found. Rewriting a `scale: 0.279...` that the user set by zooming is the kind of silent, annoying regression that makes people distrust automated edits.

## Why It Works

- **Zero re-tagging, zero plugins.** Color groups and path filters are native graph features reading frontmatter you already wrote. The entire payoff comes from data that was already there — it was just rendering monochrome.
- **`type:` is the right axis** because it's both exhaustive (frontmatter rule guarantees it) and navigational (it maps to how you actually think about the vault — "show me the people," "show me the decisions").
- **Property-keyed beats tag-keyed** on coverage, and the difference only shows up on the notes you'd most want not to miss (the ones with sloppy tags).
- **It degrades gracefully.** A new note gets colored the instant it has a `type:` — no per-note graph maintenance, no index to keep in sync.

## Watch-outs

**Verify the bracket syntax renders in your Obsidian version.** `[property:value]` is rock-solid in *search*; in *color groups* it's underdocumented and was flaky in older builds. It works in current Obsidian (verified 1.8.x, 2026-06). If you apply the config and every node stays one color, that's the cause — and the fallback is trivial because your `type` values double as your primary tags: swap `[type:person]` → `tag:#person`. You lose the exhaustiveness argument from §1 but get a working graph. Test by opening the graph once after applying.

**Untyped files render in the default color.** The same audit that confirmed 112 cleanly-typed nodes also found 17 files with no `type:` at all — mostly transcripts and sources. They'll show up uncolored. That's not a graph bug; it's the graph *surfacing* a frontmatter gap. It shares a root cause with orphan nodes (notes with no wikilinks): both are discipline gaps the colored graph now makes visible. Treat a sea of gray nodes as a backlog signal, not a config failure.

**`showTags` is a real tradeoff, not a default.** Turning tags on adds a node per tag — more cross-cutting connections, but also more hairball. With color-by-`type` already giving you category structure, you often don't need tag nodes too. Leave `showTags: false` unless you specifically want to see tag-based clustering.

**Plugins are the escape hatch, not the entry point.** If, after native color groups, you still want per-node *arcs* (a ring segment per tag/property on each node), statistical node sizing (hubs drawn bigger), or saved per-view configs, **Extended Graph** (ElsaTam, the major 2025 graph plugin) does all of it and consumes your existing `type:`/`tags:` with no new metadata. But it reverse-engineers the core graph with no official API, so Obsidian updates can break it — don't build a workflow dependency on it. Reach for it only when you've hit a real wall the native config can't clear. Juggl and the 3D renderers are effectively unmaintained (last meaningful updates 2+ years old); skip them.

## How to Adopt

1. Read your current `.obsidian/graph.json`. If `colorGroups` is `[]`, you have the whole payoff ahead of you.
2. List your vault's `type:` values: `grep -rh "^type:" --include="*.md" . | sort | uniq -c`. Those are your color groups.
3. Write one group per type with the `[type:X]` bracket syntax, plus a `path:Inbox/` (or equivalent work-in-flight) group. Set `collapse-color-groups: false`.
4. Set `search` to strip only inert folders (`-path:Templates/ -path:_archive/`). Do **not** strip folders whose notes carry real links.
5. Leave every force/scale value untouched.
6. Open the graph and verify colors render. If everything's one color, swap bracket queries for `tag:#X` and reopen.
7. Note the uncolored stragglers — they're your untyped-frontmatter backlog, fixable later.

## Adjacent Patterns

- **wikilink-graph-hygiene** — the data-side companion. That pattern gets the edges and node categories right; this one renders them. Read both: a perfectly tuned color scheme over a poorly linked vault still shows a hairball, and a well-linked vault with default colors shows the same hairball. You need both halves.
- **dataviewjs-custom-ui** — when the graph isn't the right view at all (you want a sortable/filterable table of the same nodes), Bases or a Dataview dashboard beats graph tuning. The graph is for spatial/relational intuition, not lookup.
- **framework-gotcha-comments** — the `rgb`-is-an-integer-not-hex detail and the `[type:]`-vs-`tag:#`-coverage gap are exactly the kind of non-obvious facts worth a one-line note next to the config, so the next editor doesn't "fix" the integer into a hex string.

## Source

- A course vault, graph audit + Tier-1 config pass, 2026-06-04. 162 markdown notes; pre-change `graph.json` had `colorGroups: []` and `search: ""`. Node counts by type at config time: project 28, daily 25, person 22, reference 17, meeting 11, decision 9 (112 typed), plus 17 files with no `type:` and a ~42% wikilink-orphan rate driven by unlinked transcripts/drafts.
- Property bracket syntax `[type:value]` confirmed rendering in graph color groups (not just search) in current Obsidian, 2026-06. The same syntax was historically flaky in color groups, hence the `tag:#value` fallback.
- Extended Graph (ElsaTam, released March 2025, ~54K downloads) noted as the native-config escape hatch for arcs/statistical-sizing/saved-views; flagged as core-graph-reverse-engineered and update-fragile. Juggl and 3D-graph plugins assessed as effectively unmaintained as of 2026-06.
